# 동시성 제어와 락 전략

## 1. 동시성 문제는 격리 수준만으로 해결하지 않는다

JPA 애플리케이션에서 자주 만나는 동시성 문제는 다음과 같다.

| 문제 | 예시 | 흔한 해결 |
| --- | --- | --- |
| Lost Update | 두 사용자가 같은 주문 상태를 동시에 변경 | 낙관적 락, 조건부 update |
| Write Skew | 서로 다른 row를 수정했지만 전체 비즈니스 규칙이 깨짐 | 관련 row 전체 잠금, aggregate 단위 잠금, 제약 조건, 격리 수준 조정 |
| 재고 음수 | 동시에 재고를 차감해 0보다 작아짐 | 비관적 락, 조건부 update |
| 중복 처리 | 같은 결제 승인 요청이 여러 번 들어옴 | idempotency key, unique constraint |
| 팬텀 리드 | 조회 조건에 맞는 row가 트랜잭션 중 새로 생김 | 격리 수준 조정, 명시적 잠금 |

격리 수준을 올리면 일부 문제는 줄어든다. 하지만 락 범위가 커지고 처리량이 줄 수 있다. 실무에서는 DB 격리 수준, unique constraint, JPA lock, 조건부 update, 재시도를 조합하는 경우가 많다.

## 1.1 exists 검사만으로 중복을 막을 수 없다

다음 코드는 실무에서 매우 자주 보이지만 동시 요청에 취약하다.

```java
if (!memberRepository.existsByEmail(email)) {
    memberRepository.save(Member.create(email));
}
```

두 트랜잭션이 동시에 `existsByEmail()`에서 false를 보고 둘 다 insert할 수 있다. 중복 가입, 중복 쿠폰 발급, 중복 신청처럼 "절대 한 번만"이어야 하는 규칙은 애플리케이션 검사만으로 막지 않는다.

```java
@Table(
    name = "member",
    uniqueConstraints = @UniqueConstraint(
        name = "uk_member_email",
        columnNames = "email"
    )
)
public class Member {
    ...
}
```

권장 방향은 unique constraint를 최종 방어선으로 두고, 중복 예외를 비즈니스 예외로 변환하는 것이다.

soft delete가 있으면 더 주의해야 한다. `deleted_at`이 있는 테이블에서 "삭제되지 않은 row만 email unique" 같은 요구는 DB별 partial index, functional index, 별도 active key 테이블 같은 설계가 필요할 수 있다.

## 1.2 Write Skew는 단순 row lock으로 놓치기 쉽다

Write Skew는 각 트랜잭션이 서로 다른 row를 수정하지만, 전체 규칙이 깨지는 문제다.

```text
규칙: 최소 한 명의 의사는 당직 상태여야 한다.

현재:
doctor A: on_call = true
doctor B: on_call = true

T1: B가 당직이므로 A를 off
T2: A가 당직이므로 B를 off

결과:
doctor A: off
doctor B: off
```

각 트랜잭션은 자기 입장에서 정당한 판단을 했지만 최종 상태는 비즈니스 규칙 위반이다.

해결은 "수정하는 row만 잠근다"보다 넓게 봐야 한다.

- 규칙 판단에 참여하는 row 전체를 같은 순서로 잠근다.
- aggregate root 또는 parent row를 잠금 대상으로 둔다.
- 가능한 규칙은 DB constraint로 강제한다.
- 정말 필요한 경우 격리 수준을 올린다.

## 2. 낙관적 락은 충돌이 드문 경우 기본 선택이다

낙관적 락은 "대부분 충돌하지 않는다"고 가정한다. 충돌이 발생하면 커밋 또는 flush 시점에 감지한다.

```java
@Entity
public class Product {

    @Id
    private Long id;

    @Version
    private Long version;

    private int stock;

    public void decreaseStock(int quantity) {
        if (stock < quantity) {
            throw new NotEnoughStockException();
        }
        stock -= quantity;
    }
}
```

동시에 같은 row를 수정하면 version 조건이 맞지 않아 update가 실패한다. 이때 애플리케이션은 재시도하거나 사용자에게 갱신 실패를 알려야 한다.

낙관적 락이 적합한 경우:

- 충돌이 드물다.
- 사용자 입력 기반 수정처럼 재시도 또는 갱신 실패 안내가 가능하다.
- 긴 시간 읽고 나중에 저장하는 흐름에서 lost update를 막고 싶다.

## 3. 비관적 락은 충돌이 잦거나 순서가 중요할 때 쓴다

비관적 락은 row를 먼저 잠그고 다른 트랜잭션의 수정을 기다리게 한다.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select p from Product p where p.id = :id")
Optional<Product> findForUpdate(@Param("id") Long id);
```

Hibernate에서는 DB에 따라 `select ... for update` 계열 SQL이 나간다.

비관적 락이 적합한 경우:

- 선착순 쿠폰, 재고 차감, 정산 마감처럼 충돌이 집중된다.
- 충돌 후 롤백/재시도 비용이 더 크다.
- 동시에 처리되면 안 되는 임계 구역이 명확하다.

주의할 점:

- 락 대기와 timeout이 발생할 수 있다.
- 데드락 가능성이 있다.
- 트랜잭션이 길면 전체 처리량이 급격히 떨어진다.

## 4. 조건부 update는 높은 처리량이 필요할 때 유용하다

재고 차감처럼 단순한 원자 연산은 엔티티를 조회한 뒤 변경하기보다 조건부 update가 더 적합할 수 있다.

```java
@Modifying(clearAutomatically = true)
@Query("""
    update Product p
    set p.stock = p.stock - :quantity
    where p.id = :productId
      and p.stock >= :quantity
""")
int decreaseStock(
    @Param("productId") Long productId,
    @Param("quantity") int quantity
);
```

반환값이 `1`이면 성공, `0`이면 재고 부족 또는 대상 없음으로 판단한다.

장점:

- DB가 한 문장으로 원자성을 보장한다.
- 엔티티 조회와 dirty checking 비용을 줄일 수 있다.
- 높은 경합 상황에서 비관적 락보다 단순하게 동작할 수 있다.

주의할 점:

- 벌크 update는 영속성 컨텍스트의 엔티티 상태와 DB 상태를 어긋나게 만들 수 있다.
- `clearAutomatically`, 명시적 `clear()`, 별도 트랜잭션 경계가 필요할 수 있다.
- 도메인 메서드의 검증 로직을 우회하므로 비즈니스 규칙이 SQL 조건에 반영되어야 한다.

## 5. 재고 차감 전략 비교

| 전략 | 장점 | 단점 | 추천 상황 |
| --- | --- | --- | --- |
| 낙관적 락 | 구현이 자연스럽고 충돌 감지 명확 | 충돌이 많으면 재시도 폭증 | 일반 수정, 낮은 경합 |
| 비관적 락 | 충돌 전 차단, 정합성 직관적 | 락 대기, 데드락, 처리량 저하 | 선착순, 정산, 높은 경합 |
| 조건부 update | 빠르고 원자적 | 영속성 컨텍스트 동기화 주의 | 단순 카운터, 재고, 쿠폰 수량 |
| unique constraint | 중복 방지 강력 | 예외 처리 필요 | idempotency, 중복 신청 방지 |

## 6. 재시도는 무조건 붙이지 않는다

낙관적 락 실패, 데드락 victim, lock timeout은 재시도로 회복 가능한 경우가 있다. 하지만 모든 예외를 자동 재시도하면 장애를 숨기고 부하를 증폭시킨다.

재시도는 다음 조건을 만족할 때만 제한적으로 둔다.

- 작업이 멱등적이다.
- 재시도 횟수와 backoff가 제한되어 있다.
- 사용자에게 중복 효과가 발생하지 않는다.
- 실패 로그와 모니터링이 남는다.
