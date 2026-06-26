# Redis 자료구조

> 태그: `#redis` `#data-structures`<br>
> 작성일: 2026-06-25<br>
> 최종 수정일: 2026-06-25

## 정의

Redis는 단순 Key-Value Store가 아니라 Value 자리에 String/List/Set/Sorted Set/Hash/Bitmap/HyperLogLog/Geo/Stream 같은 다양한 자료구조를 지원해, 애플리케이션 레벨에서 복잡하게 처리할 로직을 Redis 단에서 원자적으로 처리할 수 있게 한다.

## 특징 / 상세

### Redis가 단순 Key-Value가 아닌 이유

Redis는 Key-Value Store지만 Value에 다양한 자료구조를 지원한다. 상황에 맞는 자료구조를 선택하면 애플리케이션 레벨에서 복잡하게 처리할 로직을 Redis 단에서 원자적으로 처리할 수 있다.

### 0. 자료구조 전체 목록

| 자료구조 | 핵심 유스케이스 |
|---|---|
| String | 캐시, 단순 값 저장, 카운터 |
| List | 최근 목록, 작업 큐 |
| Set | 중복 제거, 집합 연산 (교/합/차) |
| Sorted Set | 랭킹, 지연 큐, 우선순위 큐 |
| Hash | 부분 읽기/수정이 잦은 객체 |
| Bitmap | 출석체크, 기능 플래그 |
| HyperLogLog | 대규모 유니크 카운팅 (근사값) |
| Geo | 위치 기반 검색 |
| Stream | 메시지 큐, 이벤트 로그 |

### 1. String

가장 기본 자료구조. 문자열뿐 아니라 숫자, JSON, 직렬화된 객체도 저장 가능하다.

```java
// 기본 저장/조회
redisTemplate.opsForValue().set("key", "value");
redisTemplate.opsForValue().get("key");

// 카운터 (원자적 증가)
redisTemplate.opsForValue().increment("page:view:1234");

// TTL과 함께
redisTemplate.opsForValue().set("cache:user:1", userData, Duration.ofMinutes(10));
```

**유스케이스**: 캐시, 분산 카운터, 세션 토큰

### 2. List

양방향 연결 리스트. 앞/뒤 삽입·삭제가 O(1)이지만 중간 인덱스 접근은 O(n)이다.

```java
// 왼쪽/오른쪽 삽입
redisTemplate.opsForList().leftPush("recent:viewed", productId);
redisTemplate.opsForList().rightPush("queue:tasks", taskId);

// 꺼내기
redisTemplate.opsForList().leftPop("queue:tasks");

// 크기 제한 (최근 20개만 유지)
redisTemplate.opsForList().leftPush("recent:viewed", productId);
redisTemplate.opsForList().trim("recent:viewed", 0, 19);
```

**유스케이스**
- 최근 본 상품 목록 — LPUSH + LTRIM으로 최근 N개 유지
- 알림 피드 — 새 알림 앞에 추가, 오래된 건 자동으로 밀려남
- 간단한 작업 큐 — RPUSH로 추가, LPOP으로 처리

### 3. Set

중복 없는 집합. 교집합·합집합·차집합 연산을 지원한다.

```java
// 추가/존재 확인
redisTemplate.opsForSet().add("followers:user:1", "user:2", "user:3");
redisTemplate.opsForSet().isMember("followers:user:1", "user:2");

// 집합 연산
redisTemplate.opsForSet().intersect("followers:user:1", "followers:user:2"); // 교집합
redisTemplate.opsForSet().union("tag:java", "tag:spring");                   // 합집합
redisTemplate.opsForSet().difference("followers:user:2", "followers:user:1"); // 차집합
```

**유스케이스**
```
친구 추천
  내 팔로워   = { A, B, C }
  철수 팔로워 = { B, C, D }
  차집합      = { D }  ← 철수는 팔로우하는데 나는 안 하는 사람 → 추천 대상

태그 기반 상품 필터
  "자바" 태그 상품  = { p1, p2, p3 }
  "백엔드" 태그 상품 = { p2, p3, p4 }
  교집합            = { p2, p3 }  ← 두 태그 모두 포함된 상품

DAU 집계
  SADD "visitors:2024-01-01" userId
  SCARD "visitors:2024-01-01"  ← 유니크 방문자 수
```

### 4. Sorted Set (ZSet)

점수(Score)를 기준으로 정렬된 집합. 같은 Score면 사전순 정렬.

```java
// 점수와 함께 추가
redisTemplate.opsForZSet().add("rank:weekly", "user:1", 1500.0);
redisTemplate.opsForZSet().incrementScore("rank:weekly", "user:1", 100.0);

// 상위 N개 조회 (내림차순)
redisTemplate.opsForZSet().reverseRange("rank:weekly", 0, 9);  // 1~10위

// 특정 Score 범위 조회
redisTemplate.opsForZSet().rangeByScore("delay:queue", 0, System.currentTimeMillis());
```

**지연 큐 (Delay Queue)** — Score를 실행 시각(timestamp)으로 설정

```java
// 30분 후 결제 취소 예약
double executeAt = System.currentTimeMillis() + 30 * 60 * 1000;
redisTemplate.opsForZSet().add("delay:queue", orderId, executeAt);

// 스케줄러가 주기적으로 실행할 작업 꺼내기
Set<String> jobs = redisTemplate.opsForZSet()
    .rangeByScore("delay:queue", 0, System.currentTimeMillis());

// 처리 후 삭제
jobs.forEach(job -> {
    process(job);
    redisTemplate.opsForZSet().remove("delay:queue", job);
});
```

Kafka나 별도 스케줄러 없이 Redis만으로 지연 실행을 구현할 수 있다.

**유스케이스**: 실시간 랭킹, 지연 큐, 우선순위 큐, 레이트 리미터

### 5. Hash

하나의 키 안에 여러 필드-값 쌍을 저장한다. JSON으로 직렬화하는 String 방식과 달리 특정 필드만 읽거나 수정할 수 있다.

#### 내부 인코딩

Redis Hash는 데이터 크기에 따라 두 가지 인코딩을 자동으로 선택한다.

```
필드 수 ≤ 128개 AND 값 크기 ≤ 64바이트
→ ziplist  (연속된 메모리 블록, 압축률 높음)

조건 초과
→ hashtable (일반 해시테이블)
```

ziplist 인코딩일 때 메모리 효율이 String JSON 대비 절반 이하다.

```
유저 세션 10만 명 저장 시
String JSON → 약 8MB
Hash ziplist → 약 3.5MB
```

#### 코드 예시

```java
// 세션 저장
String sessionKey = "session:" + sessionId;
Map<String, String> sessionData = Map.of(
    "userId", userId,
    "role", "admin",
    "lastSeen", String.valueOf(System.currentTimeMillis())
);
redisTemplate.opsForHash().putAll(sessionKey, sessionData);
redisTemplate.expire(sessionKey, Duration.ofHours(2));

// lastSeen만 업데이트 (전체 재저장 없이)
redisTemplate.opsForHash().put(sessionKey, "lastSeen",
    String.valueOf(System.currentTimeMillis()));

// 특정 필드만 읽기
String role = (String) redisTemplate.opsForHash().get(sessionKey, "role");

// 전체 읽기
Map<Object, Object> session = redisTemplate.opsForHash().entries(sessionKey);
```

String JSON 방식에서 `lastSeen`만 바꾸려면 전체 GET → 역직렬화 → 수정 → 직렬화 → SET이 필요하다. Hash는 `HSET` 한 줄이다.

#### 실시간 설정값 관리

```java
// 서비스 설정을 Hash로 관리
redisTemplate.opsForHash().putAll("config:payment", Map.of(
    "maxRetry", "3",
    "timeout", "5000",
    "feeRate", "0.03"
));

// 서버 재시작 없이 특정 설정만 실시간 변경
redisTemplate.opsForHash().put("config:payment", "feeRate", "0.025");
```

**언제 Hash를 쓰나**
```
✅ 하나의 객체를 저장하는데
✅ 필드를 부분적으로 읽거나 수정하는 경우가 잦고
✅ 필드 수가 128개 이하일 때 (ziplist 혜택)
```

### 6. Bitmap

비트 단위로 데이터를 저장하고 연산한다. 메모리 효율이 극단적으로 높다.

```java
// 유저 1234가 오늘(365번째 날) 출석
redisTemplate.opsForValue().setBit("attendance:2024:user:1234", 365, true);

// 출석 여부 확인
redisTemplate.opsForValue().getBit("attendance:2024:user:1234", 365);

// 올해 총 출석 일수
Long count = redisTemplate.execute(connection ->
    connection.bitCount("attendance:2024:user:1234".getBytes())
);
```

연간 출석 데이터를 **365비트 = 약 46바이트**로 저장한다. Boolean 배열 JSON으로 저장하면 수백 배 크기다.

**유스케이스**: 출석체크, 기능 플래그(Feature Flag), 이벤트 참여 여부

### 7. HyperLogLog

중복 없는 카운팅을 **근사값(오차 0.81%)** 으로 처리한다. 데이터 크기와 무관하게 항상 **12KB** 고정 메모리를 사용한다.

```java
// 방문자 추가
redisTemplate.opsForHyperLogLog().add("visitors:2024-01-01", userId);

// 유니크 방문자 수 (근사값)
Long count = redisTemplate.opsForHyperLogLog().size("visitors:2024-01-01");

// 여러 날 합산
Long weeklyCount = redisTemplate.opsForHyperLogLog().union(
    "visitors:weekly",
    "visitors:2024-01-01", "visitors:2024-01-02", "visitors:2024-01-03"
);
```

Set으로 DAU를 집계하면 유저 수만큼 메모리가 늘어난다. 억 명이 와도 HyperLogLog는 12KB다.

**정확한 수치가 필요 없는** 대규모 유니크 카운팅에 적합하다.

### 8. Geo

위도/경도를 저장하고 거리 계산, 반경 내 검색을 지원한다. 내부적으로 Sorted Set으로 구현되어 있다. (위도/경도를 Geohash로 변환해서 Score로 저장)

```java
// 가게 위치 저장
redisTemplate.opsForGeo().add("stores",
    new Point(127.027, 37.497),  // 경도, 위도
    "스타벅스강남점"
);

// 현재 위치에서 5km 내 가게 검색
GeoResults<RedisGeoCommands.GeoLocation<String>> results =
    redisTemplate.opsForGeo().radius("stores",
        new Circle(
            new Point(127.025, 37.495),
            new Distance(5, Metrics.KILOMETERS)
        )
    );

// 두 지점 간 거리 계산
Distance distance = redisTemplate.opsForGeo().distance(
    "stores", "스타벅스강남점", "스타벅스역삼점", Metrics.KILOMETERS
);
```

**유스케이스**: 배달 앱 주변 가게 검색, 택시 앱 근처 기사 찾기, 반경 내 이벤트 검색

### 9. Stream

로그처럼 시간 순서가 보장된 메시지 스트림이다. Kafka와 유사한 Consumer Group 개념을 지원한다.

```java
// 메시지 발행
Map<String, String> message = Map.of("orderId", "001", "status", "PAID");
redisTemplate.opsForStream().add("order:events", message);

// Consumer Group 생성
redisTemplate.opsForStream().createGroup("order:events", "payment-service");

// 메시지 소비
List<MapRecord<String, String, String>> messages =
    redisTemplate.opsForStream().read(
        Consumer.from("payment-service", "consumer-1"),
        StreamReadOptions.empty().count(10),
        StreamOffset.create("order:events", ReadOffset.lastConsumed())
    );

// 처리 완료 확인
redisTemplate.opsForStream().acknowledge("order:events", "payment-service", messageId);
```

**Kafka와 비교**

| | Redis Stream | Kafka |
|---|---|---|
| 설치 복잡도 | 낮음 | 높음 |
| 처리량 | 중간 | 매우 높음 |
| 메시지 보관 | 메모리 (유실 가능) | 디스크 (영구) |
| 적합한 상황 | 소규모 이벤트 처리 | 대용량 스트리밍 |

**유스케이스**: 소규모 이벤트 처리, 실시간 알림, 활동 로그

## 트레이드오프

해당 없음 (특징/상세 0장 "자료구조 전체 목록" 표 및 각 자료구조별 유스케이스 참고)

## 실무 경험

해당 없음 (관련 실무 내용은 특징/상세 4장 Sorted Set 지연 큐, 5장 Hash 실시간 설정값 관리에 포함)

## 참고

- [Redis 공식 문서 — Data Types](https://redis.io/docs/data-types/)
- [Redis 자료구조 치트시트](https://redis.io/docs/data-types/tutorial/)

## 관련 내용

- [Redis-SortedSet-Ranking](Redis-SortedSet-Ranking.md)
- [Redis-Stream에서-Kafka로-전환하기](Redis-Stream에서-Kafka로-전환하기.md)
