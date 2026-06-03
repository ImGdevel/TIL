# @Transactional 실무 함정

## 1. self-invocation에서는 @Transactional이 적용되지 않는다

Spring의 선언적 트랜잭션은 기본적으로 프록시 기반 AOP로 동작한다. 같은 클래스 내부에서 자기 자신의 메서드를 호출하면 프록시를 거치지 않는다.

```java
@Service
public class OrderService {

    public void outer() {
        inner(); // 프록시를 거치지 않는다.
    }

    @Transactional
    public void inner() {
        // 트랜잭션이 기대처럼 시작되지 않을 수 있다.
    }
}
```

가장 단순한 해결은 트랜잭션 경계가 필요한 메서드를 다른 bean으로 분리하는 것이다.

```java
@Service
public class OrderFacade {

    private final OrderCommandService orderCommandService;

    public void outer() {
        orderCommandService.inner();
    }
}

@Service
public class OrderCommandService {

    @Transactional
    public void inner() {
        // 프록시를 통해 호출된다.
    }
}
```

## 2. private 메서드에 붙인 @Transactional은 의미가 없다

프록시 기반 AOP는 외부에서 호출되는 public 메서드를 중심으로 동작한다.

```java
@Transactional
private void saveInternal() {
    // 기대한 방식으로 트랜잭션 경계가 잡히지 않는다.
}
```

트랜잭션 경계는 외부에서 호출되는 서비스 public 메서드에 둔다. 내부 helper 메서드는 트랜잭션이 이미 열린 상태에서 실행되는 일반 메서드로 두는 편이 명확하다.

## 3. RuntimeException은 롤백, checked exception은 기본적으로 롤백되지 않는다

Spring의 기본 롤백 규칙은 다음과 같다.

- `RuntimeException`, `Error`: 롤백
- checked exception: 기본적으로 커밋

checked exception에서도 롤백해야 한다면 명시한다.

```java
@Transactional(rollbackFor = IOException.class)
public void importFile(File file) throws IOException {
    importer.importFile(file);
}
```

반대로 특정 런타임 예외에서 커밋해야 한다면 `noRollbackFor`를 사용할 수 있다.

```java
@Transactional(noRollbackFor = AlreadyProcessedException.class)
public void process(Message message) {
    ...
}
```

다만 롤백 규칙이 복잡해지면 예외 설계와 트랜잭션 경계가 적절한지 다시 점검해야 한다.

## 4. 예외를 잡아도 rollback-only는 남아 있을 수 있다

`REQUIRED` 전파에서는 내부 논리 트랜잭션이 같은 물리 트랜잭션을 공유한다. 내부 메서드가 트랜잭션을 rollback-only로 표시하면, 외부에서 예외를 잡아도 커밋 시점에 `UnexpectedRollbackException`이 발생할 수 있다.

```java
@Transactional
public void outer() {
    try {
        innerService.inner();
    } catch (Exception ignored) {
        // 예외를 잡았지만 물리 트랜잭션은 rollback-only일 수 있다.
    }

    // commit 시점에 UnexpectedRollbackException 가능
}
```

해결 방향:

- 실패를 유스케이스 실패로 보고 예외를 삼키지 않는다.
- 정말 독립적으로 실패를 기록해야 한다면 `REQUIRES_NEW`로 분리한다.
- 내부 작업의 실패가 전체 트랜잭션에 어떤 의미인지 비즈니스 규칙으로 먼저 정의한다.

## 5. @Transactional은 final 클래스/메서드와 궁합이 나쁠 수 있다

프록시 방식에 따라 final 클래스나 final 메서드는 AOP 적용이 제한될 수 있다. Kotlin을 사용할 때도 클래스와 메서드가 기본적으로 final이므로 Spring 플러그인 설정이나 open class 전략을 확인해야 한다.

Java에서도 다음을 피하는 편이 안전하다.

```java
@Service
public final class OrderService {

    @Transactional
    public final void placeOrder() {
        ...
    }
}
```

## 6. 테스트의 @Transactional은 운영 코드와 다르게 보일 수 있다

Spring 테스트에서 `@Transactional`을 붙이면 테스트 메서드가 끝날 때 기본적으로 롤백된다.

```java
@Transactional
@SpringBootTest
class OrderServiceTest {

    @Test
    void placeOrder() {
        orderService.placeOrder(command);
        // 테스트 종료 후 롤백
    }
}
```

이 때문에 다음을 놓칠 수 있다.

- commit 시점에 발생하는 제약 조건 위반
- transaction synchronization afterCommit 동작
- 실제 DB에 남은 데이터 기준의 후속 검증

커밋 이후 동작을 검증해야 한다면 테스트 트랜잭션을 끄거나, 명시적으로 flush/commit 경계를 다르게 잡아야 한다.

## 7. 실무 점검표

- `@Transactional` 메서드가 같은 클래스 내부에서 호출되고 있지 않은가?
- 트랜잭션 경계가 public 서비스 메서드에 있는가?
- checked exception에서도 롤백이 필요한가?
- 예외를 catch한 뒤에도 rollback-only가 남을 가능성이 있는가?
- `REQUIRES_NEW`를 독립 커밋 목적 외에 남용하고 있지 않은가?
- 테스트의 rollback 동작 때문에 실제 commit 문제를 놓치고 있지 않은가?

