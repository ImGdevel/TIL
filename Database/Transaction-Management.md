# 트랜잭션 관리 방법 정리 (JPA/Spring 중심)

> “트랜잭션은 꼭 쓰라면서, 정작 언제·어디에 써야 하는지는 잘 안 알려준다.”  
> 이 문서는 **트랜잭션 개념 → Spring에서 관리하는 방법 → 실무에서 자주 하는 실수와 패턴** 순으로 정리한다.

---

## 1. 트랜잭션이란?

- **하나의 작업 단위(Unit of Work)**  
  - 모두 성공하거나(All or Nothing), 모두 롤백되어야 하는 작업 묶음
- 4가지 특성(ACID)
  - **Atomicity(원자성)**: 전부 성공 또는 전부 실패
  - **Consistency(일관성)**: 비즈니스 규칙을 항상 만족하는 상태 유지
  - **Isolation(고립성)**: 동시에 실행되는 트랜잭션 간 간섭 최소화
  - **Durability(지속성)**: 커밋된 내용은 영구적으로 저장

JPA/Spring에서 트랜잭션 관리는 대부분 **서비스 계층**에서 이루어지며,  
Spring이 프록시/프레임워크 레벨에서 커넥션과 커밋/롤백을 대신 관리한다.

---

## 2. Spring에서 트랜잭션을 관리하는 방법

### 2.1 선언적 트랜잭션 (@Transactional)

가장 많이 사용하는 방식으로, 애노테이션만으로 트랜잭션을 제어한다.

```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder(OrderRequest request) {
        // 1. 주문 생성
        // 2. 재고 차감
        // 3. 결제 처리
        // 중간에 예외가 나면 전체 롤백
    }
}
```

- 트랜잭션 시작/커밋/롤백을 Spring이 대신 처리
- 기본 전략
  - **런타임 예외(RuntimeException, Error)** 발생 시 롤백
  - 체크 예외는 기본적으로 롤백하지 않음

#### 옵션들

- `@Transactional(readOnly = true)`
  - 읽기 전용 트랜잭션
  - JPA에서는 플러시를 줄이고, 일부 DB에서는 최적화 가능
- `rollbackFor`, `noRollbackFor`
  - 특정 예외에 대해 롤백/비롤백 세밀하게 설정
- `timeout`, `isolation`, `propagation`

---

### 2.2 XML/Java Config 기반 AOP 설정 (요즘은 거의 사용 X)

- 과거에는 XML이나 `@EnableTransactionManagement` + 수동 설정으로 AOP를 구성
- 최근엔 대부분 `@Transactional` 기반 + 자동 설정으로 충분

---

### 2.3 프로그래밍 방식 트랜잭션 (TransactionTemplate, PlatformTransactionManager)

코드를 통해 직접 트랜잭션 경계를 제어해야 할 때 사용:

```java
public class TxExample {

    private final TransactionTemplate txTemplate;

    public TxExample(PlatformTransactionManager txManager) {
        this.txTemplate = new TransactionTemplate(txManager);
    }

    public void doWork() {
        txTemplate.execute(status -> {
            // 트랜잭션 안에서 실행될 로직
            return null;
        });
    }
}
```

- 선언적 방식으로 표현하기 어려운 복잡한 흐름(분기, 재시도 등)에서 사용
- 대부분의 일반적인 서비스 로직은 `@Transactional`로 충분하다.

---

## 3. 트랜잭션 경계는 어디에 둘 것인가?

### 3.1 서비스 계층에 둔다

- **Controller (웹/API) → Service (비즈니스) → Repository (DB)** 구조를 기준으로
  - 트랜잭션 경계는 보통 **Service 메서드 단위**에 둔다.

이유:

- 비즈니스 관점에서 “하나의 작업 단위”가 서비스 메서드에 들어가기 쉬움
- 여러 Repository 호출을 하나의 트랜잭션으로 묶기 좋음
- 계층 간 역할 분리가 명확해짐

### 3.2 Controller/Repository 수준에서의 트랜잭션은 지양

- Controller에 `@Transactional`:
  - 웹 계층 + 비즈니스 로직이 섞여 관리가 어려워짐
- Repository에 `@Transactional`:
  - 여러 Repository 호출을 하나의 트랜잭션으로 묶기 어려워짐
  - 재사용성과 조합성이 떨어짐

---

## 4. 트랜잭션 전파(Propagation)와 중첩 호출

### 4.1 기본 전파 규칙 – REQUIRED

- `@Transactional`의 기본 전파 속성: `Propagation.REQUIRED`
  - 현재 트랜잭션이 있으면 참여
  - 없으면 새로 시작

예:

```java
@Transactional
public void outer() {
    inner(); // inner도 @Transactional(REQUIRED)인 경우 같은 트랜잭션 공유
}

@Transactional
public void inner() {
    // outer와 같은 트랜잭션
}
```

- 이 경우 `outer`에서 예외가 나면 `inner`까지 포함해 **전체 롤백**

### 4.2 REQUIRES_NEW, NESTED 등

- `REQUIRES_NEW`
  - 기존 트랜잭션을 잠시 보류하고 **새 트랜잭션을 시작**
  - 부모가 롤백되더라도, 자식 트랜잭션을 따로 커밋할 수 있음 (반대 방향도 마찬가지)
- `NESTED`
  - 일부 DB/드라이버에서 지원
  - 부모 안에 **저장점(Savepoint)** 를 두고 부분 롤백

실무에서는 주로,

- **로그 저장, 감사 기록, 알림 발송 등**  
  “메인 트랜잭션과는 독립적인 부가 작업”에 `REQUIRES_NEW`를 쓰기도 한다.

---

## 5. 자주 하는 실수와 주의점

### 5.1 같은 클래스 내부의 메서드 호출 (self-invocation) 문제

```java
@Service
public class MyService {

    @Transactional
    public void outer() {
        inner(); // ❗ 여기서 @Transactional이 적용되지 않을 수 있음
    }

    @Transactional
    public void inner() {
        // ...
    }
}
```

- Spring AOP 프록시는 **외부에서 호출할 때만** 트랜잭션을 적용한다.
- 같은 클래스 내부에서 `this.inner()` 호출은 프록시를 거치지 않기 때문에  
  `inner()`의 `@Transactional`이 무시된다.

해결:

- 트랜잭션 경계를 명확히 다시 설계하거나
- 내부 호출이 필요하면 **별도 빈/클래스로 분리**하여 외부 호출 형태로 만든다.

### 5.2 readOnly 트랜잭션에서 쓰기 작업 수행

- `@Transactional(readOnly = true)` 안에서 `save`, `delete` 같은 쓰기 작업을 하면
  - JPA 플러시가 제한되거나, DB에서 에러가 날 수 있음
  - 의도와 다른 동작이 발생할 수 있음

권장:

- 조회 전용 서비스 메서드는 `readOnly = true`
- 쓰기/수정/삭제가 포함된 메서드는 기본 `@Transactional` 또는 명시적으로 `readOnly = false`

### 5.3 트랜잭션 안에서만 동작하는 JPA 기능을 트랜잭션 밖에서 사용

- Lazy 로딩, 변경 감지(Dirty Checking) 등은 **영속성 컨텍스트와 트랜잭션**에 의존
- 트랜잭션 밖에서 Lazy 로딩 사용 시 `LazyInitializationException` 발생

권장:

- 엔티티 조회/로딩, 변경 감지가 필요한 모든 작업은 **트랜잭션 안에서 수행**
- Controller에서는 DTO만 사용하고, 엔티티 초기화는 Service에서 끝낸다.

### 5.4 예외 처리 시 롤백 규칙을 잘못 이해

- 기본적으로 RuntimeException만 롤백 대상임
- 체크 예외를 던지면서 롤백을 기대하는 경우:

```java
@Transactional(rollbackFor = MyCheckedException.class)
public void doSomething() throws MyCheckedException {
    // ...
    throw new MyCheckedException();
}
```

---

## 6. 트랜잭션 관리 베스트 프랙티스 요약

- 트랜잭션 경계는 **서비스 계층**에 둔다.
- **선언적 트랜잭션(@Transactional)** 을 기본으로 사용하고,  
  특별한 경우에만 프로그래밍 방식 트랜잭션을 고려한다.
- 조회 전용 메서드는 `@Transactional(readOnly = true)`로 최적화한다.
- 같은 클래스 내부에서의 `@Transactional` 메서드 호출(self-invocation)에 주의한다.
- Lazy 로딩, 변경 감지 등은 반드시 **트랜잭션 안에서** 사용한다.
- 전파(Propagation)를 이용해, 메인 트랜잭션과 부가 작업 트랜잭션을 적절히 분리한다.

이 원칙만 지켜도, 대부분의 JPA/Spring 기반 애플리케이션에서  
트랜잭션 관련 버그와 데이터 일관성 문제를 크게 줄일 수 있다.

