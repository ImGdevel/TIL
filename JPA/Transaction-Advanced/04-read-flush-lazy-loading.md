# 조회 트랜잭션, flush, Lazy Loading

## 1. readOnly 트랜잭션은 조회 의도를 명확히 한다

조회 서비스에는 `readOnly = true`를 붙이는 것이 좋다.

```java
@Transactional(readOnly = true)
public OrderDetailResponse getOrder(Long orderId) {
    Order order = orderRepository.findDetail(orderId)
        .orElseThrow(OrderNotFoundException::new);

    return OrderDetailResponse.from(order);
}
```

장점:

- 이 메서드는 변경 의도가 없다는 사실이 드러난다.
- JPA/Hibernate가 flush, dirty checking 관련 최적화를 적용할 수 있다.
- 일부 DB/드라이버 조합에서는 읽기 전용 힌트로 활용될 수 있다.

주의할 점:

- `readOnly = true`가 항상 DB write를 물리적으로 막아주는 것은 아니다.
- 조회 중 엔티티를 변경하는 코드를 숨겨도 된다는 뜻이 아니다.
- 쓰기 작업은 명확히 별도 command 서비스로 분리하는 편이 낫다.

## 2. flush는 커밋 직전에만 발생하지 않는다

JPA는 보통 트랜잭션 커밋 직전에 변경 내용을 DB에 flush한다. 하지만 다음 상황에서는 커밋 전에도 SQL이 나갈 수 있다.

- JPQL/QueryDSL 쿼리 실행 전
- 명시적 `entityManager.flush()`
- 식별자 생성 전략상 즉시 insert가 필요한 경우
- flush mode 설정에 따른 자동 flush

예를 들어 다음 코드는 조회 쿼리 실행 전에 insert/update가 먼저 flush될 수 있다.

```java
@Transactional
public void register(Member member) {
    memberRepository.save(member);

    boolean duplicated = memberRepository.existsByEmail(member.getEmail());
    if (duplicated) {
        throw new DuplicateEmailException();
    }
}
```

flush 시점은 "SQL이 언제 나가는가"와 "예외가 언제 터지는가"를 결정한다. unique constraint 위반, not null 위반, FK 위반은 save 시점이 아니라 flush/commit 시점에 드러날 수 있다.

## 3. flush와 commit은 다르다

`flush`는 영속성 컨텍스트의 변경 내용을 DB에 SQL로 보내는 작업이다. `commit`은 그 변경을 확정하는 작업이다.

```java
entityManager.flush();
```

flush 이후에도 트랜잭션이 rollback되면 변경은 확정되지 않는다. 이 차이를 헷갈리면 "flush했으니 이미 저장됐다"거나 "save를 호출했으니 DB에 확정됐다"는 잘못된 판단을 하게 된다.

## 4. 명시적 flush는 실패 지점을 앞당길 때 쓴다

```java
@Transactional
public void createOrder(OrderCommand command) {
    Order order = Order.create(command);
    orderRepository.save(order);

    entityManager.flush();

    outboxRepository.save(OrderCreatedEvent.from(order));
}
```

명시적 flush는 DB 제약 조건 위반을 특정 지점에서 확인하고 싶을 때 유용하다. 하지만 남발하면 JPA가 변경 사항을 모아 처리하는 장점을 잃는다.

명시적 flush가 적합한 경우:

- 외부 API 호출 전에 DB 제약 조건 위반을 먼저 확인해야 한다.
- 테스트에서 SQL 발생 시점과 예외 발생 시점을 명확히 검증해야 한다.
- 대량 처리 중 일정 단위로 DB 반영과 영속성 컨텍스트 정리가 필요하다.

## 5. Dirty Checking은 의도치 않은 update를 만들 수 있다

JPA는 트랜잭션 안에서 managed entity의 변경을 감지해 flush 시점에 update한다.

```java
@Transactional
public void readAndNormalize(Long memberId) {
    Member member = memberRepository.getById(memberId);
    member.setName(member.getName().trim());
}
```

명시적으로 `save()`를 호출하지 않아도 update가 나갈 수 있다. 트랜잭션 범위가 넓고 setter가 열려 있으면 조회용 로직에서 의도치 않은 변경이 발생하기 쉽다.

대응 방향:

- 엔티티 setter를 무분별하게 열어두지 않는다.
- 조회 전용 로직은 DTO projection 또는 `readOnly = true`로 분리한다.
- 변경 메서드는 비즈니스 의도가 드러나는 이름으로 제한한다.
- 트랜잭션 안에서 엔티티를 오래 들고 다니지 않는다.

## 6. Lazy Loading 문제는 트랜잭션을 길게 늘려서 해결하지 않는다

Lazy Loading 예외는 보통 서비스 트랜잭션 밖에서 지연 로딩이 발생할 때 난다.

```java
@GetMapping("/orders/{id}")
public OrderResponse getOrder(@PathVariable Long id) {
    Order order = orderService.getOrder(id);
    return OrderResponse.from(order); // 여기서 orderItems 접근 시 LazyInitializationException 가능
}
```

해결책은 트랜잭션을 컨트롤러까지 늘리는 것이 아니라, 서비스 계층 안에서 필요한 데이터를 명시적으로 조회하는 것이다.

```java
@Query("""
    select o
    from Order o
    join fetch o.orderItems
    where o.id = :orderId
""")
Optional<Order> findDetail(@Param("orderId") Long orderId);
```

또는 DTO projection을 사용한다.

```java
@Query("""
    select new com.example.OrderDetailRow(o.id, i.name, i.quantity)
    from Order o
    join o.orderItems i
    where o.id = :orderId
""")
List<OrderDetailRow> findOrderDetailRows(@Param("orderId") Long orderId);
```

## 7. 트랜잭션 안의 N+1은 커넥션 점유 시간도 늘린다

N+1은 단순 조회 성능 문제로만 보이지만, 트랜잭션 안에서 발생하면 커넥션과 영속성 컨텍스트를 더 오래 붙잡는다.

```java
@Transactional(readOnly = true)
public List<OrderResponse> getOrders() {
    return orderRepository.findAll().stream()
        .map(order -> OrderResponse.of(order, order.getMember().getName()))
        .toList();
}
```

주문 수만큼 member lazy loading이 반복되면 쿼리 수가 늘고, 트랜잭션도 길어진다. 읽기 API라도 대량 목록에서는 DTO projection, fetch join, batch size를 먼저 검토한다.

## 8. OSIV는 편하지만 쿼리 위치를 숨긴다

Open Session In View를 켜면 웹 요청이 끝날 때까지 영속성 컨텍스트가 열려 있어 컨트롤러나 뷰에서도 Lazy Loading이 가능하다.

문제는 다음과 같다.

- 컨트롤러/뷰 계층에서 예상치 못한 쿼리가 발생한다.
- N+1 문제가 뒤늦게 드러난다.
- 커넥션 점유 시간이 길어질 수 있다.
- 서비스 계층의 조회 의도가 흐려진다.

실무에서는 OSIV를 끄고, 필요한 데이터는 서비스 계층에서 fetch join, entity graph, DTO projection으로 명시적으로 가져오는 방향이 더 예측 가능하다.

## 9. 조회 전략 요약

| 상황 | 권장 방식 |
| --- | --- |
| 단건 상세 조회 | fetch join, entity graph |
| 목록 조회 | DTO projection |
| 수정 목적 조회 | 엔티티 조회 + 트랜잭션 안에서 변경 |
| 읽기 전용 API | `@Transactional(readOnly = true)` |
| Lazy Loading 예외 | 필요한 연관 데이터를 서비스 계층에서 명시 조회 |
| 제약 조건 예외를 빨리 확인 | 필요한 지점에서 명시적 flush |
| 조회 중 의도치 않은 update 방지 | DTO projection, readOnly, 제한된 변경 메서드 |
