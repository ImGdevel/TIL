# NoSQL TTL (Time To Live)

> 태그: `#db` `#nosql` `#redis` `#mongodb` `#ttl`<br>
> 작성일: 2026-06-23<br>
> 최종 수정일: 2026-06-23

## 정의

TTL은 데이터에 만료 시간을 설정해 DB가 자동 삭제하게 하는 기능으로, Redis/MongoDB 모두 지원하지만 만료 즉시 삭제를 보장하지 않으므로 보안 민감 로직(인증 코드 등)은 TTL에만 의존하면 안 된다.

## 특징 / 상세

### 개념

데이터에 만료 시간을 설정하면 DB가 자동으로 삭제하는 기능이다.

RDB에서는 만료된 데이터를 배치 잡이나 스케줄러로 직접 삭제해야 했다.

```sql
-- RDB: 직접 삭제 쿼리 작성 필요
DELETE FROM sessions WHERE expired_at < NOW();
```

NoSQL은 DB 레벨에서 기본 지원한다.

### Redis TTL

```java
// 세션 저장 + 30분 후 자동 삭제
redisTemplate.opsForValue().set(
    "session:user:1234",
    sessionData,
    Duration.ofMinutes(30)
);

// 남은 시간 확인
redisTemplate.getExpire("session:user:1234"); // 초 단위 반환

// TTL 갱신 (접속할 때마다 연장)
redisTemplate.expire("session:user:1234", Duration.ofMinutes(30));
```

### MongoDB TTL

TTL 인덱스를 생성하면 백그라운드에서 만료된 문서를 자동 삭제한다.

```js
// created_at 기준으로 1시간 후 자동 삭제
db.logs.createIndex(
    { created_at: 1 },
    { expireAfterSeconds: 3600 }
)
```

### 유스케이스

| 용도 | 설명 |
|---|---|
| 세션 관리 | 로그인 토큰, 일정 시간 후 자동 로그아웃 |
| 인증 코드 | SMS/이메일 인증 코드 (5분 후 만료) |
| 캐시 | 일정 시간 후 캐시 무효화 |
| 임시 데이터 | 장바구니, 임시 저장 |
| 로그/이벤트 | 보관 기간 지나면 자동 삭제 |

### TTL 만료 처리 방식

#### Redis

Redis는 두 가지 방식으로 만료를 처리한다.

**1. Lazy Expiration (지연 삭제)**

키에 접근할 때 만료됐는지 확인하고 그때 삭제한다.

```
TTL 만료됨 → 아무것도 안 함
나중에 GET 요청 → "이거 만료됐네" → 그때 삭제
```

접근하지 않으면 만료된 데이터가 메모리에 계속 남는다.

**2. Active Expiration (주기적 삭제)**

100ms마다 만료된 키를 랜덤 샘플링해서 삭제한다.

```
100ms마다
→ 랜덤으로 20개 키 샘플링
→ 만료된 키 삭제
→ 25% 이상 만료됐으면 즉시 반복
```

전체를 확인하지 않고 랜덤 샘플링이라 만료됐어도 한동안 살아있을 수 있다.

#### MongoDB

별도 스레드가 60초 주기로 TTL 인덱스를 스캔해서 만료된 문서를 삭제한다.

```
expireAfterSeconds: 3600 설정
→ 실제 삭제: 3600초 ~ 3660초 사이
→ 최대 60초 지연 발생 가능
```

Lazy Expiration은 없고 Active 방식만 있다.

#### 비교

| | Redis | MongoDB |
|---|---|---|
| Lazy | ✅ 접근 시 삭제 | ❌ 없음 |
| Active | ✅ 100ms마다 샘플링 | ✅ 60초마다 인덱스 스캔 |
| 즉시 삭제 보장 | ❌ | ❌ |
| 최대 지연 | 불확실 (샘플링) | 최대 60초 |

### TTL 만료 = 즉시 삭제가 아니다

**TTL이 만료됐어도 즉시 삭제된다는 보장이 없다.** 로직에서 "만료됐으면 없겠지"라고 가정하면 버그가 생긴다.

```java
// 위험한 방식 — TTL 만료됐는데 아직 존재할 수 있음
String code = redis.get("auth:code:1234");
if (code != null) {
    validate(code);  // 만료된 코드가 유효한 것처럼 처리될 수 있음
}

// 안전한 방식 — 만료 시간을 별도로 저장하고 직접 검증
AuthCode authCode = redis.get("auth:code:1234");
if (authCode != null && authCode.getExpiredAt().isAfter(LocalDateTime.now())) {
    validate(authCode.getCode());
}
```

인증 코드 만료 처리 같은 보안 로직은 TTL에만 의존하지 않고 애플리케이션 레벨에서 직접 만료 시간을 검증해야 한다.

### Redis Eviction 정책

메모리가 꽉 차면 TTL 설정과 무관하게 데이터가 삭제될 수 있다.

```yaml
# redis.conf
maxmemory 1gb
maxmemory-policy allkeys-lru
```

| 정책 | 동작 |
|---|---|
| noeviction | 메모리 꽉 차면 쓰기 오류 반환 |
| allkeys-lru | 전체 키 중 최근 사용 안 한 것 삭제 |
| volatile-lru | TTL 있는 키 중 최근 사용 안 한 것 삭제 |
| volatile-ttl | TTL 가장 짧게 남은 것 먼저 삭제 |
| allkeys-random | 전체 키 중 랜덤 삭제 |

TTL을 설정했어도 메모리 부족 시 **만료 전에 삭제**될 수 있다. Redis를 캐시로 사용할 때는 반드시 eviction 정책을 명시해야 한다.

> Redis를 캐시로 사용: `allkeys-lru` 권장
> Redis를 영구 저장소로 사용: `noeviction` 또는 `volatile-*` 계열

### Redis Keyspace Notification (만료 이벤트 구독)

TTL이 만료되는 순간을 애플리케이션에서 감지할 수 있다. Redis의 Pub/Sub을 이용해 만료 이벤트를 구독하는 방식이다.

```yaml
# redis.conf — 만료 이벤트 활성화
notify-keyspace-events Ex
# E: Keyspace 이벤트, x: 만료(expired) 이벤트
```

```java
@Component
public class RedisKeyExpiredListener extends KeyExpirationEventMessageListener {

    public RedisKeyExpiredListener(RedisMessageListenerContainer container) {
        super(container);
    }

    @Override
    public void onMessage(Message message, byte[] pattern) {
        String expiredKey = message.toString();
        // "session:user:1234" 만료됨 → 후처리 로직
        log.info("키 만료됨: {}", expiredKey);

        // 예: 만료된 세션 정리, 알림 발송 등
        if (expiredKey.startsWith("session:user:")) {
            String userId = expiredKey.split(":")[2];
            handleSessionExpired(userId);
        }
    }
}
```

**주의사항**: Keyspace Notification도 Active Expiration 기반이라 만료 즉시 이벤트가 발생한다는 보장이 없다. 또한 Redis 클러스터 환경에서는 각 노드별로 구독 설정이 필요하다.

**언제 쓰나**
- 세션 만료 시 사용자에게 알림
- 주문 결제 대기 시간 초과 시 자동 취소
- 임시 예약 만료 시 재고 복구

## 트레이드오프

해당 없음 — Redis/MongoDB TTL 방식 차이는 위 특징/상세 참고.

## 실무 경험

해당 없음

## 참고

원본 학습 노트(TIL)에서 이전한 링크. 확인일 미기재 — 필요 시 재검증.

- [Redis TTL 공식 문서](https://redis.io/docs/manual/keyspace/#key-expiration)
- [Redis Eviction 정책 공식 문서](https://redis.io/docs/manual/eviction/)
- [MongoDB TTL 인덱스 공식 문서](https://www.mongodb.com/docs/manual/core/index-ttl/)

## 관련 내용

- [nosql-개요](nosql-개요.md)
- [nosql-데이터-모델링](nosql-데이터-모델링.md)
- [nosql-인덱스](nosql-인덱스.md)
