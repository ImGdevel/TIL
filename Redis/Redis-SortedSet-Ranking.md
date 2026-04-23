# Redis Sorted Set(ZSET)으로 랭킹 캐싱 설계하기

> 연결된 정리본:
> - [Redis 개요와 캐싱](../../../TIL.wiki/redis-overview-and-caching.md)


> "ZSET은 단순한 캐시가 아니다. 정렬된 상태를 유지하면서 빠르게 조회/갱신까지 가능한, 랭킹을 위해 태어난 자료구조다."

---

## 1. 왜 일반 캐시(Map, String)로는 부족한가

랭킹 시스템의 핵심 요구사항은 두 가지다.

- **빠른 갱신**: 점수가 바뀔 때마다 순위를 재계산
- **빠른 조회**: 상위 N명, 특정 유저의 순위

일반 캐시(String/Hash)에 데이터를 넣어두면 매 조회마다 정렬을 따로 해야 한다.  
반면 ZSET은 삽입 시점에 이미 정렬된 상태를 유지하므로, 조회 시 별도 정렬이 필요 없다.

```
일반 캐시 방식:
Redis에 {user_id: score} 저장 → 조회할 때 전부 꺼내서 정렬 → O(N log N)

ZSET 방식:
ZADD leaderboard {score} {user_id} → ZREVRANGE로 바로 조회 → O(log N + M)
```

---

## 2. ZSET의 핵심 특성

### 2.1 점수 기반 자동 정렬 — O(log N)

- 내부적으로 **skiplist + hash** 구조를 사용
- 삽입/갱신 모두 O(log N) 보장
- 항상 score 오름차순으로 정렬된 상태를 유지

### 2.2 핵심 명령어

| 명령어 | 역할 | 시간 복잡도 |
|--------|------|------------|
| `ZADD key score member` | 삽입 / 점수 갱신 | O(log N) |
| `ZINCRBY key increment member` | 점수 증가 | O(log N) |
| `ZREVRANGE key 0 9` | 상위 10명 조회 | O(log N + M) |
| `ZREVRANK key member` | 특정 유저 순위 | O(log N) |
| `ZRANGE key min max BYSCORE` | 점수 구간 조회 | O(log N + M) |
| `ZCARD key` | 전체 멤버 수 | O(1) |

### 2.3 DB와 비교

```sql
-- DB에서 랭킹 조회
SELECT user_id, score
FROM ranking
ORDER BY score DESC
LIMIT 10;
-- 인덱스 있어도 트래픽 몰리면 병목
```

```
-- Redis ZSET
ZREVRANGE leaderboard 0 9 WITHSCORES
-- 메모리 기반, 항상 O(log N + M), 즉시 반환
```

---

## 3. 캐시 + 인덱스 역할을 동시에

보통 캐시는 "조회 속도 향상"만 담당한다.  
ZSET은 여기에 **정렬 인덱스 역할**까지 겸한다.

```
DB: write 중심 (점수 영속화)
Redis ZSET: read 중심 (정렬된 랭킹 서빙)
```

DB의 `ORDER BY` 부하를 Redis로 오프로드하는 구조로,  
DB는 write에 집중하고 Redis는 랭킹 조회를 전담한다.

---

## 4. 실시간 업데이트 패턴

```
// 점수 갱신 시
ZINCRBY leaderboard 10 "user:123"  // 10점 증가
ZADD leaderboard 500 "user:456"    // 절댓값으로 설정

// 상위 10명 조회
ZREVRANGE leaderboard 0 9 WITHSCORES

// 특정 유저의 순위 (0-based)
ZREVRANK leaderboard "user:123"
```

`ZINCRBY`는 기존 score를 읽어서 더한 뒤 다시 쓰는 게 아니라, atomic하게 증가시킨다.  
→ 동시성 이슈 없이 안전하게 점수 누적 가능

---

## 5. 확장 패턴

### 5.1 기간별 랭킹 분리

```
leaderboard:daily:2026-04-07
leaderboard:weekly:2026-W14
leaderboard:all-time
```

키를 분리하면 기간별 랭킹을 독립적으로 관리할 수 있다.

### 5.2 TTL 기반 캐싱 전략

주기적으로 DB에서 랭킹을 재계산하여 Redis에 밀어넣고, TTL로 자동 만료시킨다.

```
// 매 1분마다 재계산 (배치 or 스케줄러)
ZADD leaderboard:cache ... 
EXPIRE leaderboard:cache 60
```

"완전 실시간"이 불필요한 경우 DB 부하를 크게 줄일 수 있다.

### 5.3 병합 랭킹

```
// 두 리그의 랭킹 합산
ZUNIONSTORE leaderboard:merged 2 leaderboard:a leaderboard:b
```

---

## 6. ZSET이 유용한 다른 케이스

랭킹 외에도 "정렬 기준(score)이 명확하고, 범위/순위 조회가 필요한 문제"라면 대부분 어울린다.

| 케이스 | score 의미 | 핵심 명령 |
|--------|-----------|---------|
| 지연 큐 (Delayed Queue) | 실행 예정 timestamp | `ZRANGEBYSCORE now ~` |
| Rate Limiting (슬라이딩 윈도우) | 요청 timestamp | `ZREMRANGEBYSCORE` + `ZCARD` |
| 최근 활동 순 정렬 | last_activity_time | `ZREVRANGE` |
| 트렌딩 키워드 / Top-K 캐시 | 검색 횟수 | `ZINCRBY` + `ZREVRANGE` |
| 가중치 기반 추천 | 복합 score (조회수+좋아요 등) | `ZINCRBY` |

---

## 7. ZSET이 맞지 않는 케이스 — 채팅 이력

채팅 이력은 요구사항이 다르다.

- "정렬"이 아니라 **순서 보존 + append** 중심
- 별도 score가 필요 없고, 시간순 이미 정렬되어 있음

```
채팅에 적합한 자료구조:
- LIST: LPUSH/RPUSH + LRANGE → 가장 단순한 채팅 로그
- STREAM: 메시지 ID 자동 생성, consumer group 지원 → 실시간 채팅, 이벤트 스트림
```

단, 아래 경우라면 채팅에 ZSET을 쓸 수 있다.

- 읽지 않은 메시지 수 계산 (score = timestamp, last_read 기준 범위 조회)
- 좋아요/중요도 기반으로 메시지 정렬
- 오래된 메시지 TTL 기반 자동 제거 (score = expire_time)

---

## 8. 자료구조 선택 기준 정리

| 요구사항 | 자료구조 |
|---------|---------|
| 점수 기반 정렬 + 범위 조회 | **ZSET** |
| 순서대로 쌓기 (append-only) | LIST |
| 이벤트 스트림 / consumer group | STREAM |
| 키-값 단순 조회 | STRING / HASH |

---

## 9. 핵심 요약

ZSET이 랭킹에 최적인 이유는 단 세 가지다.

1. **캐시** — 메모리 기반, 빠른 접근
2. **정렬** — 삽입 시점에 자동 유지, 조회 시 추가 sort 불필요
3. **효율적인 랭킹 조회** — Top-N, 특정 순위, 점수 구간 모두 O(log N)

DB의 `ORDER BY` 부하를 오프로드하면서 동시에 정렬 인덱스 역할까지 하는,  
랭킹 시스템에서 Redis ZSET은 거의 필수 선택지다.
