# 실무 안티패턴 체크리스트

## 1. 외부 API를 트랜잭션 안에서 호출한다

```java
@Transactional
public void order(OrderCommand command) {
    orderRepository.save(Order.create(command));
    paymentClient.pay(command.paymentKey());
}
```

위험한 이유:

- 외부 API 대기 시간만큼 DB 커넥션과 락을 오래 잡는다.
- 외부 API는 성공했지만 DB commit이 실패할 수 있다.
- timeout이 났지만 외부 시스템에서는 성공했을 수 있다.

대응:

- DB 상태 변경과 외부 호출을 분리한다.
- Outbox, Saga, idempotency key를 사용한다.
- 외부 결과는 상태 전이로 기록하고 reconcile 경로를 둔다.

## 2. exists 검사 후 save로 중복을 막으려 한다

```java
if (!couponRepository.existsByUserIdAndEventId(userId, eventId)) {
    couponRepository.save(Coupon.issue(userId, eventId));
}
```

동시 요청에서는 두 트랜잭션이 모두 `exists = false`를 볼 수 있다.

대응:

- unique constraint를 둔다.
- insert 실패를 비즈니스 예외로 변환한다.
- 선착순 수량은 조건부 update 또는 비관적 락으로 제어한다.
- soft delete가 있다면 active row 기준 unique 설계를 별도로 검토한다.

## 3. 조회 후 조건 검사 후 update한다

```java
Product product = productRepository.getById(productId);

if (product.getStock() > 0) {
    product.decreaseStock(1);
}
```

동시성 환경에서는 같은 stock 값을 여러 트랜잭션이 읽고 각각 차감할 수 있다.

대응:

- `@Version`으로 lost update를 감지한다.
- `PESSIMISTIC_WRITE`로 먼저 잠근다.
- `where stock > 0` 조건부 update를 사용한다.

## 4. 트랜잭션 범위가 너무 넓다

```java
@Transactional
public void complexProcess() {
    // 조회, 검증, 외부 호출, 파일 처리, DB 변경, 이벤트 발행이 모두 섞임
}
```

문제:

- lock 보유 시간이 길어진다.
- dirty checking 대상이 늘어난다.
- 실패 시 어떤 작업까지 되돌려야 하는지 불명확해진다.
- Lazy Loading, N+1, 의도치 않은 update가 섞여도 추적이 어렵다.

대응:

- 하나의 비즈니스 불변식을 지키는 최소 범위로 줄인다.
- 외부 I/O는 트랜잭션 밖으로 뺀다.
- command와 query 흐름을 분리한다.
- 긴 작업은 상태 machine과 후속 작업으로 나눈다.

## 5. REQUIRES_NEW를 "부분 성공" 용도로 쉽게 쓴다

```java
@Transactional
public void placeOrder() {
    orderRepository.save(order);
    auditService.saveAudit(); // REQUIRES_NEW
    throw new RuntimeException();
}
```

`saveAudit()`는 커밋되고 주문은 롤백될 수 있다. 의도한 감사 로그라면 괜찮지만, 비즈니스 데이터에 쓰면 불일치가 생긴다.

대응:

- 독립 커밋이 비즈니스적으로 맞는지 먼저 확인한다.
- 감사 로그, 실패 이력처럼 독립성이 필요한 곳에 제한한다.
- 일반 도메인 상태 변경에는 남용하지 않는다.

## 6. 이벤트 발행을 트랜잭션 내부에서 직접 처리한다

```java
@Transactional
public void placeOrder() {
    orderRepository.save(order);
    kafkaTemplate.send("order-placed", event);
}
```

DB commit과 메시지 발행은 원자적으로 묶이지 않는다.

대응:

- 유실이 치명적이면 Outbox Pattern을 사용한다.
- 소비자는 중복 이벤트를 처리할 수 있어야 한다.
- 커밋 이후 간단한 후속 작업은 `@TransactionalEventListener(AFTER_COMMIT)`을 고려한다.
- 이벤트 핸들러 실패와 재시도 정책을 명시한다.

## 7. Dirty Checking을 의식하지 않고 엔티티를 변경한다

```java
@Transactional
public MemberResponse getMember(Long id) {
    Member member = memberRepository.getById(id);
    member.setLastViewedAt(Instant.now());
    return MemberResponse.from(member);
}
```

조회처럼 보이는 메서드에서도 update가 나갈 수 있다.

대응:

- 조회 API는 `readOnly = true`와 DTO projection을 우선 검토한다.
- 변경 의도가 있는 메서드와 조회 메서드를 분리한다.
- setter를 줄이고 도메인 변경 메서드로 의도를 드러낸다.

## 8. flush와 commit을 같은 것으로 본다

```java
entityManager.flush();
```

flush는 SQL을 DB에 보내는 것이고, commit은 확정이다. flush 이후에도 rollback될 수 있다.

대응:

- 제약 조건 위반을 빨리 보고 싶을 때만 명시적 flush를 쓴다.
- 외부 API 호출 전에 DB 제약 조건을 확인해야 하는 경우 flush 위치를 의도적으로 둔다.
- flush를 성공의 의미로 해석하지 않는다.

## 9. 캐시를 트랜잭션과 같은 원자성으로 생각한다

```java
@Transactional
public void updateProduct(Long id, String name) {
    productRepository.getById(id).changeName(name);
    cache.evict("product:" + id);
}
```

캐시 삭제 후 DB가 rollback될 수 있고, DB commit 후 캐시 삭제가 실패할 수도 있다.

대응:

- commit 이후 캐시 삭제를 고려한다.
- TTL과 재시도를 둔다.
- stale read 허용 범위를 정한다.
- 강한 정합성이 필요한 데이터는 캐시를 최종 판단 근거로 쓰지 않는다.

## 10. 동시성 테스트 없이 재고, 쿠폰, 좌석 예약을 배포한다

재고, 쿠폰, 좌석 예약은 단위 테스트만으로 안전성을 판단하기 어렵다.

대응:

- 동시에 같은 key를 때리는 테스트를 작성한다.
- unique constraint, row count, 최종 수량을 검증한다.
- lock timeout과 deadlock 상황을 운영 로그에서 관찰한다.
- 재시도 정책이 부하를 증폭시키지 않는지 확인한다.

## 빠른 점검 질문

- 트랜잭션 안에서 외부 API, 메시지 발행, 캐시 변경을 직접 수행하는가?
- 중복 방지를 애플리케이션 `exists` 검사에만 의존하는가?
- 동시에 같은 데이터를 수정하면 lost update가 가능한가?
- soft delete와 unique constraint 요구가 충돌하지 않는가?
- rollback 대상 예외가 명확한가?
- 이벤트 소비자는 중복 이벤트를 처리할 수 있는가?
- 조회 메서드에서 dirty checking으로 update가 나갈 가능성이 있는가?
- N+1 때문에 트랜잭션과 커넥션 점유 시간이 길어지지 않는가?

