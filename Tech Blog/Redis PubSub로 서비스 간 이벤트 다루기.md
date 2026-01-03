# Redis Pub/Sub로 서비스 간 이벤트 다루기

> “Kafka를 쓰기엔 무겁고, 그냥 동기 호출만 하기엔 아쉬울 때.”

## 0. 이 글을 쓰게 된 계기

마이크로서비스까지는 아니더라도,  
서비스 레이어가 조금씩 나뉘기 시작하면 자연스럽게 “이벤트” 이야기가 나온다.

- 주문이 생성되면 알림 서비스를 호출해야 하고  
- 유저가 가입하면 추천/포인트/메일링 시스템이 반응해야 한다.

처음에는 대부분 **동기 HTTP 호출**로 시작한다.

> “주문 생성 API 안에서 알림/포인트/메일을 순서대로 호출하면 되지.”

근데 서비스가 커질수록,

- 한 서비스의 장애가 다른 서비스까지 전파되고  
- 요청 응답 시간이 일관되지 않고  
- “이건 사실 비동기로 처리해도 되는 일인데…”라는 생각이 쌓인다.

그 지점에서 가볍게 고려해볼 수 있는 도구 중 하나가 **Redis Pub/Sub**였다.

이 글은,

- Redis Pub/Sub이 어떤 구조로 동작하는지  
- 실제 프로젝트에서 어떻게 써봤는지  
- Kafka 같은 “무거운 메시지 브로커”와 비교했을 때 어디까지를 맡길 수 있는지

를 정리해 보려는 시도다.

---

## 1. 왜 Redis Pub/Sub을 알아둘 필요가 있을까?

솔직하게 말하면, Redis Pub/Sub은 **만능 메시지 브로커가 아니다.**  
그럼에도 불구하고 알아둘 가치가 있다고 느낀 이유는 세 가지 정도였다.

1. **“동기 호출 vs 제대로 된 메시지 브로커” 사이의 간극을 메워준다**
   - Kafka, RabbitMQ, NATS 같은 브로커는 강력하지만, 초기 도입/운영 비용이 있다.  
   - Pub/Sub은 이미 쓰고 있는 Redis 위에서 “가볍게 비동기 이벤트”를 붙이기 좋다.

2. **서비스 간 결합도를 한 단계만이라도 낮출 수 있다**
   - 주문 서비스가 “알림/포인트/메일” 서비스의 존재를 직접 알지 않고,  
   - 단순히 “order.created” 이벤트를 발행하는 식으로 바꿀 수 있다.

3. **Redis를 이미 쓰고 있다면, 추가 인프라 없이 바로 실험해 볼 수 있다**
   - 별도의 클러스터나 운영 도구 없이도,  
   - “이 이벤트는 Pub/Sub으로 한 번 빼보자”는 실험을 빠르게 해볼 수 있다.

물론 그 대가로,

- 메시지 내구성  
- 재처리/재시도  
- 소비자 그룹 관리

같은 부분은 어느 정도 포기해야 한다.  
그래서 어디까지를 Redis Pub/Sub에게 맡길지, 어떤 곳부터는 Kafka 같은 도구로 넘길지 기준을 갖고 있는 게 중요하다.

---

## 2. Redis Pub/Sub 구조 한 번에 훑어보기

먼저 개념부터 정리해 보자.

- **Publisher**
  - 특정 채널(channel)로 메시지를 발행(PUBLISH)하는 쪽
- **Subscriber**
  - 하나 이상의 채널에 구독(SUBSCRIBE)하고, 메시지를 받아 처리하는 쪽
- **Channel**
  - 문자열로 된 이름. 예: `"order.created"`, `"user.signup"`, `"notice.*"` 등  
  - Pub/Sub에서는 채널별로 메시지가 브로드캐스트된다.

흐름을 텍스트로 그려보면 대략 이렇다.

1. Subscriber가 `SUBSCRIBE order.created`로 Redis에 구독을 건다.
2. Publisher가 `PUBLISH order.created "{...payload...}"`를 보낸다.
3. Redis는 해당 채널을 구독 중인 모든 Subscriber에게 메시지를 푸시한다.
4. 메시지는 **즉시 전달되고, 그 뒤에는 저장되지 않는다.**

한 줄로 줄이면,

> **“Redis Pub/Sub은 ‘현재 연결 중인 구독자들’에게만 메시지를 뿌리는 브로드캐스트 채널”**이라고 이해하는 게 제일 편했다.

여기서 중요한 제약이 하나 나온다.

> **구독자가 잠시 끊겨 있던 동안 발행된 메시지는, 나중에 다시 연결해도 받을 수 없다.**

이 제약 때문에, Redis Pub/Sub은  
“로그성 이벤트를 안전하게 쌓고 나중에 재처리해야 하는” 시나리오보다는,  
“실시간 알림/신호”에 더 가깝게 쓰는 게 맞다.

---

## 3. Pub/Sub 명령어 감각 잡기

실제 Redis CLI 기준으로 보면 명령어는 단순하다.

### 3-1. 발행 – PUBLISH

```bash
PUBLISH order.created "{ \"orderId\": 123, \"userId\": 1 }"
```

- 반환값은 이 메시지를 받은 Subscriber의 수다. (예: `integer 2`)

### 3-2. 구독 – SUBSCRIBE / PSUBSCRIBE

```bash
SUBSCRIBE order.created
```

패턴으로 여러 채널을 한 번에 구독할 수도 있다.

```bash
PSUBSCRIBE order.*
```

일단 `SUBSCRIBE`/`PSUBSCRIBE` 모드에 들어가면,  
해당 Redis 연결(connection)은 **블로킹된 상태로 계속 메시지를 기다리게 된다.**

그래서 애플리케이션 코드에서는 보통:

- 별도의 스레드/비동기 작업에서 구독 연결을 유지하고  
- 메시지가 올 때마다 핸들러를 호출하는 구조를 많이 쓴다.

---

## 4. Spring에서 Redis Pub/Sub 사용하기

Spring Data Redis를 쓰면 Pub/Sub도 비교적 손쉽게 연동할 수 있다.

### 4-1. 의존성과 기본 설정

Gradle:

```groovy
implementation "org.springframework.boot:spring-boot-starter-data-redis"
```

`application.yml`에서 Redis 접속 정보를 설정해 두고,  
보통은 `LettuceConnectionFactory` 기반의 `RedisConnectionFactory`가 자동 구성된다.

### 4-2. 메시지 발행 – RedisTemplate / StringRedisTemplate

발행 쪽은 단순하다. 예를 들어:

```java
@Service
public class OrderEventPublisher {

    private final StringRedisTemplate redisTemplate;

    public void publishOrderCreated(OrderCreatedEvent event) {
        String channel = "order.created";
        String payload = objectMapper.writeValueAsString(event);
        redisTemplate.convertAndSend(channel, payload);
    }
}
```

내부적으로는 `PUBLISH` 명령을 사용한다.

### 4-3. 구독 – MessageListener와 Container

구독 쪽은 `MessageListener`와 `RedisMessageListenerContainer`를 사용한다.

```java
@Configuration
public class RedisPubSubConfig {

    @Bean
    public RedisMessageListenerContainer redisContainer(
        RedisConnectionFactory connectionFactory,
        MessageListenerAdapter orderCreatedListener
    ) {
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        container.addMessageListener(orderCreatedListener, new ChannelTopic("order.created"));
        return container;
    }

    @Bean
    public MessageListenerAdapter orderCreatedListener(OrderCreatedSubscriber subscriber) {
        return new MessageListenerAdapter(subscriber, "onMessage");
    }
}
```

핸들러는 이런 식으로 작성할 수 있다.

```java
@Component
public class OrderCreatedSubscriber {

    public void onMessage(String message, String channel) {
        // message: payload, channel: "order.created"
        // 여기서 JSON 파싱 후 실제 비즈니스 로직 처리
    }
}
```

중요한 포인트는:

- `RedisMessageListenerContainer`가 내부적으로 **구독용 연결을 유지**하면서  
- 메시지가 도착할 때마다 `MessageListener`를 콜백으로 호출해 준다는 것.

---

## 5. Redis Pub/Sub의 장점과 한계

실제로 써보면서 느낀 장점과 한계를 정리해 보면 이렇다.

### 5-1. 장점

1. **도입이 매우 쉽다**
   - Redis를 이미 쓰고 있다면 추가 인프라 없이 바로 시작할 수 있다.

2. **지연(latency)이 낮다**
   - Pub/Sub 자체는 메모리 기반이고, 브로커 로직도 단순하다.

3. **구현이 가볍다**
   - Publisher/Subscriber 코드가 비교적 단순하고, 개념이 직관적이다.

4. **서비스 간 결합도를 줄이는 첫 단계로 좋다**
   - 기존 동기 호출에서 “이벤트 발행”으로만 바꿔도, 의존 관계가 한 단계 느슨해진다.

### 5-2. 한계와 주의사항

1. **메시지가 저장되지 않는다 (내구성 없음)**
   - Subscriber가 잠시 죽어 있던 동안 발행된 메시지는 사라진다.  
   - 나중에 다시 켜져도 “놓친 메시지”를 알 수 없다.

2. **재처리/리플레이가 어렵다**
   - Kafka처럼 “특정 시점 이후 메시지를 다시 읽기” 같은 패턴이 없다.  
   - 이런 요구가 생기는 순간, Pub/Sub만으로는 충분하지 않다.

3. **소비자 그룹, 오프셋 개념이 없다**
   - Pub/Sub은 브로드캐스트에 가깝다.  
   - 특정 그룹 단위로 메시지를 분배하는 구조가 아니라,  
     채널에 붙어 있는 모든 Subscriber가 다 받는다.

4. **백프레셔/버퍼링 전략이 거의 없다**
   - Subscriber가 느리게 처리하면, 그 연결에서 메시지가 쌓이거나,  
     결국 처리 지연/타임아웃으로 이어질 수 있다.

5. **보안/격리 수준이 브로커 전문 솔루션보다 약하다**
   - Redis 전체 인스턴스에 대한 접근 제어에 많이 의존한다.  
   - 멀티 테넌트/복잡한 권한 모델이 필요한 환경에서는 부족할 수 있다.

결론적으로, Redis Pub/Sub은  
**“메시지는 잃어도 괜찮지만, 실시간 반응이 중요한 이벤트”**에 어울리는 도구라는 느낌이 강했다.

---

## 6. Redis Pub/Sub 사용할 때 실무 주의점

장단점을 알고 나서 실제로 붙여보면, 특히 아래 포인트들을 신경 쓰게 되었다.

1. **절대 유실되면 안 되는 이벤트에는 쓰지 않는다**
   - 주문/결제/정산/재고 등은 Pub/Sub 대신 Kafka·큐·DB 트랜잭션 로그 등 내구성 있는 경로를 먼저 고려한다.

2. **Subscriber 수명/상태를 모니터링한다**
   - 프로세스가 죽어 있던 동안의 메시지는 놓치기 때문에,  
     “구독자가 살아 있는지”를 별도의 HealthCheck/알람으로 감시하는 게 중요하다.

3. **처리 실패/재시도 전략을 애플리케이션에서 직접 설계한다**
   - Pub/Sub 자체는 재시도 큐가 없다.  
   - 실패 시 별도 DLQ(예: Redis List, DB 테이블)에 적재하거나, 재시도 로직을 애플리케이션 레벨에서 구현해야 한다.

4. **채널 설계를 너무 거칠게 가져가지 않는다**
   - `*` 패턴으로 너무 많은 이벤트를 한 채널에 섞어 버리면,  
     구독자 코드에서 분기/파싱이 복잡해지고 디버깅이 어려워진다.

5. **한 구독자에 너무 많은 일을 몰아 넣지 않는다**
   - 하나의 Subscriber가 여러 종류의 일을 모두 처리하면,  
     특정 이벤트 처리 지연이 다른 이벤트의 지연으로 그대로 전파된다.

6. **Pub/Sub을 “로그 저장소”처럼 쓰지 않는다**
   - 나중에 다시 보고 싶은 이벤트라면, Pub/Sub과는 별도로  
     DB·Kafka·파일 등 영속적인 저장소로도 함께 흘려 보내는 설계가 필요하다.

---

## 7. 언제 Redis Pub/Sub을 쓰고, 언제 Kafka 같은 걸 써야 할까?

내가 기준으로 삼고 싶은 몇 가지 질문은 이렇다.

1. **메시지를 잃어버려도 되는가?**
   - 예: 실시간 알림 배너, 온라인 사용자 수, 메트릭/모니터링 신호 등  
     → 한두 개 놓쳐도 큰 문제가 없다면 Pub/Sub도 충분하다.
   - 반대로 “모든 주문 이벤트를 반드시 기록하고, 나중에 재처리해야 한다”  
     → Kafka 같은 내구성 있는 로그/브로커가 필요하다.

2. **여러 소비자 그룹이 서로 다른 속도로 처리해도 괜찮은가?**
   - Kafka는 consumer group별로 오프셋을 관리해 각자 속도로 읽을 수 있다.  
   - Pub/Sub은 그런 개념이 없다.  
   - 소비자 속도가 많이 달라지고, 그룹이 여러 개라면 Redis Pub/Sub만으로는 부족하다.

3. **이벤트 스트림을 “데이터 파이프라인”으로도 쓰고 싶은가?**
   - 예: 주문 이벤트를 기반으로 실시간 대시보드, 배치 통계, ML 피처 파이프라인까지  
   - 이런 “장기 보관 + 다양한 소비자” 시나리오에는 Kafka가 훨씬 잘 맞는다.

실무에서는 보통 이렇게 나눌 수 있다고 느꼈다.

- Redis Pub/Sub
  - 실시간 알림, 간단한 서비스 간 신호, UI 업데이트 트리거 등  
  - 유실을 어느 정도 허용할 수 있는 “신호/notification” 계열
- Kafka / RabbitMQ 등
  - 주문/결제/재고/로그처럼 **유실이 치명적인 도메인 이벤트**  
  - 데이터 파이프라인/분석/재처리가 중요한 영역

---

## 8. 마무리 – Pub/Sub은 메시징의 “입문용 도구”로 보기

Redis Pub/Sub을 쓰면서 느낀 건,

> “메시지 브로커의 모든 기능을 기대하기보다는,  
>  동기 호출에서 한 단계만 비동기로 나아가기 위한 입문용 도구로 보는 게 편하다.”

라는 점이었다.

이 글에서 정리한 내용을 기준으로,

- 지금 서비스에서 “동기 호출로 얽혀 있는 부분” 중  
  - 유실을 어느 정도 허용할 수 있고  
  - 실시간성이 중요하며  
  - 이벤트 방식으로 바꾸면 구조가 훨씬 단순해질 영역이 있는지

을 한 번 찾아보면 좋을 것 같다.

그 영역에 한정해서 Redis Pub/Sub을 도입해 보고,  
그 한계를 실제로 부딪혀 보는 것 자체가 좋은 학습 경험이 된다.

그 이후에 Kafka나 다른 브로커를 도입하게 되더라도,

- “어떤 기능이 있었으면 좋겠는지”  
- “어떤 한계를 해결하고 싶은지”

가 더 분명해져서, 도구 선택이 훨씬 수월해진다는 점을 마지막으로 남겨두고 싶다.
