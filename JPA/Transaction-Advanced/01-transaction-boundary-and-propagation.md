# 트랜잭션 경계와 전파 전략

## 1. 트랜잭션 경계는 서비스 유스케이스에 둔다

JPA/Spring 애플리케이션에서 트랜잭션 경계는 보통 서비스 계층의 public 유스케이스 메서드에 둔다.

```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder(OrderCommand command) {
        Order order = orderFactory.create(command);
        orderRepository.save(order);

        inventoryService.decrease(command.productId(), command.quantity());
        outboxRepository.save(OrderPlacedEvent.from(order));
    }
}
```

핵심은 "하나의 비즈니스 불변식을 지키는 최소 범위"다. 주문 생성과 재고 차감이 함께 성공하거나 함께 실패해야 한다면 같은 트랜잭션에 둔다. 반대로 외부 API 호출, 파일 업로드, 메시지 발행처럼 DB 트랜잭션으로 원자성을 보장할 수 없는 작업은 같은 경계에 오래 붙잡아 두지 않는 편이 낫다.

## 2. 긴 트랜잭션은 커넥션과 락을 오래 점유한다

다음 코드는 위험하다.

```java
@Transactional
public void pay(Long orderId) {
    Order order = orderRepository.getById(orderId);
    paymentClient.approve(order.getPaymentKey());
    order.markPaid();
}
```

외부 결제 API가 느려지면 트랜잭션도 같이 길어진다. 그동안 DB 커넥션, 영속성 컨텍스트, 경우에 따라 row lock까지 오래 잡힌다.

더 나은 방향은 DB 상태 변경과 외부 호출을 분리하는 것이다.

```java
@Transactional
public void requestPayment(Long orderId) {
    Order order = orderRepository.getById(orderId);
    order.markPaymentRequested();
    outboxRepository.save(PaymentRequestedEvent.from(order));
}
```

이후 별도 워커가 outbox 이벤트를 읽어 결제 API를 호출하고, 결과를 다시 트랜잭션으로 반영한다.

## 3. 기본 전파 전략은 REQUIRED다

`@Transactional`의 기본 전파 속성은 `Propagation.REQUIRED`다.

- 현재 트랜잭션이 있으면 참여한다.
- 현재 트랜잭션이 없으면 새로 시작한다.
- 여러 논리 트랜잭션이 하나의 물리 트랜잭션을 공유할 수 있다.

```java
@Transactional
public void outer() {
    innerService.inner();
}

@Transactional
public void inner() {
    // outer와 같은 물리 트랜잭션에 참여한다.
}
```

대부분의 서비스 로직은 `REQUIRED`로 충분하다. 복잡한 전파 전략을 먼저 설계하기보다, 트랜잭션 경계 자체가 적절한지 먼저 봐야 한다.

## 4. REQUIRES_NEW는 독립 커밋이 필요할 때만 쓴다

`REQUIRES_NEW`는 기존 트랜잭션을 잠시 보류하고 새 물리 트랜잭션을 시작한다.

```java
@Transactional
public void placeOrder(OrderCommand command) {
    try {
        orderProcessor.process(command);
    } catch (Exception e) {
        auditService.saveFailureLog(command, e);
        throw e;
    }
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveFailureLog(OrderCommand command, Exception e) {
    auditLogRepository.save(AuditLog.failure(command, e));
}
```

이 패턴은 감사 로그, 실패 이력, 일부 outbox 보정처럼 메인 트랜잭션이 롤백되어도 반드시 남겨야 하는 기록에 쓴다.

주의할 점:

- 커넥션을 추가로 사용할 수 있다.
- 외부 트랜잭션과 독립적으로 커밋된다.
- 남용하면 전체 흐름의 원자성이 깨지고 추적이 어려워진다.

## 5. NESTED는 저장점 기반 부분 롤백이다

`NESTED`는 부모 트랜잭션 안에서 savepoint를 만들고 일부 작업만 롤백하는 전략이다.

```java
@Transactional(propagation = Propagation.NESTED)
public void processStep(Step step) {
    stepProcessor.process(step);
}
```

다만 모든 DB, 드라이버, 트랜잭션 매니저 조합에서 같은 방식으로 지원되는 것은 아니다. JPA 중심 애플리케이션에서는 `REQUIRES_NEW`보다 사용 빈도가 낮고, 부분 롤백 요구가 명확할 때만 검토한다.

## 6. 전파 전략 선택 요약

| 전략 | 특징 | 실무 사용처 |
| --- | --- | --- |
| `REQUIRED` | 기존 트랜잭션 참여, 없으면 생성 | 일반 서비스 유스케이스 |
| `REQUIRES_NEW` | 항상 새 트랜잭션 | 감사 로그, 실패 이력, 독립 outbox 저장 |
| `NESTED` | savepoint 기반 부분 롤백 | 일부 단계만 되돌리는 처리 |
| `SUPPORTS` | 트랜잭션이 있으면 참여, 없어도 실행 | 조회성 보조 로직 |
| `MANDATORY` | 기존 트랜잭션이 없으면 예외 | 반드시 상위 트랜잭션 안에서만 호출되어야 하는 내부 API |
| `NOT_SUPPORTED` | 트랜잭션 없이 실행 | 트랜잭션 밖에서 실행해야 하는 긴 작업 |

