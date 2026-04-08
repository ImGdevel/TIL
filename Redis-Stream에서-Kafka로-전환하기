# Redis Stream에서 Kafka로 전환하기

> "이건 메시지 브로커 교체 문제가 아니라, 추상화 레이어가 숨기고 있던 실행 의미(semantics)의 차이를 드러내는 작업이다."

---

## 1. Redis Streams vs Kafka: 본질적인 차이

### 1-1. 메시징 모델

| 항목 | Redis Streams | Kafka |
|---|---|---|
| 저장소 | In-memory (AOF/RDB로 보완) | 디스크 기반 append-only log |
| 성격 | **빠른 큐** | **영속 로그 + 이벤트 스트리밍 플랫폼** |
| retention | 메모리 한계에 따라 제한적 | 시간/용량 기반 정책 (무기한 가능) |
| replay | 제한적 | 언제든 offset rewind 가능 |
| 미소비 메시지 보호 | **보장 안됨** | 보장됨 |

Redis Streams는 `MAXLEN` 설정이나 메모리 압박(`maxmemory-policy`)으로 인해 **아직 소비하지 않은 메시지도 삭제될 수 있다.** Consumer Group의 Pending Entries List(PEL)에 ID가 남아있어도, stream에서 실제 메시지가 지워지면 복구 불가능하다.

> Redis Streams 기초와 PEL 개념은 [Redis Stream으로 이벤트 로그 다루기](./Redis%20Stream으로%20이벤트%20로그%20다루기.md)를 참고한다.

### 1-2. Ack / Commit 모델

| 항목 | Redis Streams | Kafka |
|---|---|---|
| 처리 확인 단위 | 메시지 단위 (`XACK`) | offset 단위 commit |
| 재처리 | pending list 기반 | offset rewind / seek |
| exactly-once | 없음 (애플리케이션 레벨) | EOS 지원 (제약 있음) |
| 실패 시 범위 | 해당 메시지만 | commit 안 하면 이전 batch 전체 |

Kafka에서 `ack(message)` 형태의 추상화를 그대로 쓰면 의미가 어긋난다. Kafka의 offset commit은 "여기까지 다 처리됨"을 선언하는 것이기 때문에, 중간 메시지 하나만 실패해도 앞뒤 메시지와 함께 재처리 대상이 될 수 있다.

---

## 2. Consumer 확장성: 어디서 막히나

### 2-1. 할당 단위가 다르다

Redis와 Kafka 둘 다 Consumer Group을 사용하지만 동작이 근본적으로 다르다.

**Redis Streams**: 메시지 단위로 consumer에게 분배된다.

```
Stream: [m1, m2, m3, m4]
  → Consumer A: m1, m3
  → Consumer B: m2, m4
```

**Kafka**: 파티션 단위로 consumer에게 할당된다.

```
Topic (4 partitions)
  → Consumer A: P0, P1
  → Consumer B: P2, P3
```

### 2-2. Kafka에서 병렬성의 상한은 partition 수

```
partition = 3
consumer = 10  → 7개는 아무것도 처리하지 않음
```

Redis에서 consumer를 늘리면 처리량이 자연스럽게 늘지만, Kafka에서는 partition 수가 병렬성의 상한이다. **설계 시점에 "최대 consumer 수"를 고려해서 partition 수를 결정해야 한다.**

### 2-3. 장애 복구 방식의 차이

| | Redis Streams | Kafka |
|---|---|---|
| Consumer 죽을 때 | pending 메시지를 다른 consumer가 `XCLAIM` | partition rebalance 발생 |
| 복구 단위 | 메시지 | partition |

---

## 3. 추상화가 깨지는 지점들

Redis Streams와 Kafka를 동일한 인터페이스로 감싸려 하면 반드시 의미가 어긋나는 지점이 생긴다.

### 3-1. stateless vs stateful consumer

Redis는 broker가 pending 상태를 관리하지만, Kafka는 **consumer가 offset 상태를 직접 관리**한다. stateless consumer처럼 보이는 추상화는 Kafka에서 장애 시 offset이 꼬이는 원인이 된다.

### 3-2. rebalancing이라는 개념의 등장

Kafka는 consumer가 추가/제거될 때 partition 재할당(rebalance)이 발생한다. in-flight 메시지의 중복 처리를 막으려면 rebalance lifecycle hook이 필요하다.

```
// Kafka에서 단순 subscribe(handler)는 충분하지 않음
onPartitionAssigned(partitions) → 상태 초기화
onPartitionRevoked(partitions)  → 진행 중 작업 정리
```

### 3-3. ordering 범위가 축소된다

| | Redis Streams | Kafka |
|---|---|---|
| ordering 범위 | stream 전체 (느슨하게 전역) | **partition 내부만** |

Kafka에서는 같은 aggregate(userId, orderId 등)가 동일한 partition으로 라우팅되어야 순서가 보장된다. `key = aggregateId`를 반드시 지정해야 한다. key 없이 발행하면 partition이 랜덤 배정되어 순서가 깨진다.

### 3-4. DLQ로 "이 메시지만 재처리"는 완전히 동일하지 않다

DLQ를 써서 실패한 메시지를 분리할 수 있지만, Redis의 pending 기반 재처리와는 차이가 있다.

| | Redis Pending | Kafka + DLQ |
|---|---|---|
| retry | 자동 (broker 관리) | 수동 / 별도 파이프라인 |
| 위치 | 원래 stream | 별도 DLQ topic |
| ordering | 유지 가능 (후속 차단 전략) | 깨짐 (m3가 m2보다 먼저 처리됨) |

DLQ는 "실패를 격리하는 전략"이지, Redis pending의 완전한 대체가 아니다.

> DLQ, retry, dead letter 개념은 [메시지 큐(MQ)로 서비스 간 작업을 나눠보기](./메시지%20큐%28MQ%29로%20서비스%20간%20작업을%20나눠보기.md)를 참고한다.

---

## 4. Outbox 패턴과 Kafka 결합 시 주의사항

Outbox 패턴은 본질적으로 **at-least-once delivery** 보장 패턴이다. Kafka의 EOS(exactly-once semantics)를 쓰더라도 DB commit과 Kafka publish는 atomic하지 않기 때문에 중복 발생은 필연적이다.

```
DB Transaction commit
  → Kafka publish 시도
  → 실패 시 retry → 중복 발행 가능
```

### 4-1. partition key 전략

같은 aggregate의 이벤트가 다른 partition에 들어가면 순서가 깨진다.

```
key = aggregateId (userId, orderId 등)
```

이 원칙이 지켜지지 않으면 상태 처리에서 버그가 발생한다.

### 4-2. relay 병렬화 시 ordering 주의

Redis에서는 relay 1개로도 충분했을 수 있지만, Kafka에서 throughput을 높이기 위해 relay를 병렬화할 경우 동일 key의 메시지가 여러 relay에서 발행되면 순서가 뒤바뀔 수 있다. **relay를 shard 기준으로 분할**하거나 Kafka partition key 전략을 활용해야 한다.

---

## 5. 점진적 마이그레이션 전략

"같은 시스템을 유지하려고 하면 실패하고, Kafka에 맞게 의미를 재정의하면 성공한다."

### 1단계: Dual Write

```
Outbox Relay
  → Redis Streams (기존)
  → Kafka         (신규, 동시 발행)
```

Kafka consumer를 새로 구현하고, Redis consumer와 처리 결과를 비교 검증한다.

### 2단계: Consumer 분리 검증

기존 Redis consumer는 유지하고, Kafka consumer를 병렬로 운영하면서 다음을 검증한다:
- 이벤트 유실 없는가
- 중복 처리 발생 시 idempotent하게 처리되는가
- ordering이 깨져도 문제없는가

### 3단계: Semantics 재정의 (가장 중요)

이 단계에서 명시적으로 새로운 계약을 선언해야 한다.

```
변경 전 (Redis 기준)
  - 메시지 단위 재처리 가능
  - global ordering 보장 (느슨하게)
  - 실패 메시지는 pending에서 자동 retry

변경 후 (Kafka 기준)
  - at-least-once: 중복 발생 가능
  - ordering은 key 단위 partition 내부에서만 보장
  - consumer는 idempotent해야 한다
  - 실패는 DLQ + retry topic으로 처리
```

이 차이를 버그로 보지 않고 **새로운 정상 동작**으로 받아들여야 마이그레이션이 성공한다.

### 4단계: Redis 제거

Kafka가 안정화된 것을 확인한 후 Redis publish를 중단한다.

---

## 6. 정리: 마이그레이션 성공 기준

Kafka로 안정적으로 이행하기 위한 시스템 계약:

| 항목 | 내용 |
|---|---|
| delivery | at-least-once (중복 허용) |
| idempotency | eventId 기반 중복 처리 필수 |
| ordering | key(aggregateId) 단위 partition 내에서만 |
| 실패 처리 | DLQ + retry topic 구성 |
| consumer | offset 상태를 직접 관리 (stateful) |
| 확장 | partition 수 = 병렬성 상한, 먼저 설계 |
| 관찰 | consumer lag, DLQ rate, processing latency |

**한 줄 핵심**: Kafka는 "큐"가 아니라 "로그 기반 상태 시스템"이므로, 기존 추상화를 복제하려 하면 반드시 깨진다.
