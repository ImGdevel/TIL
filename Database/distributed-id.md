# 분산 환경에서의 ID 생성 전략

## 왜 중요한가

단일 DB에서는 AUTO_INCREMENT로 ID를 생성하면 됐다. 분산 환경에서는 여러 노드가 동시에 ID를 생성하므로 중복이 발생할 수 있다. 전역적으로 유일한 ID를 생성하는 전략이 필요하다.

좋은 분산 ID의 조건:
```
1. 전역 유일성   — 어느 노드에서 생성해도 중복 없음
2. 정렬 가능     — 시간 순서대로 정렬 가능하면 샤딩/인덱스에 유리
3. 생성 속도     — 높은 처리량 지원
4. 단조 증가     — 같은 시간이면 항상 증가하는 방향
```

---

## UUID

### 개념

128비트 랜덤 값. 전역 유일성을 확률적으로 보장한다.

```
550e8400-e29b-41d4-a716-446655440000
```

### 특징

- 구현이 매우 단순
- 중앙 서버 없이 각 노드에서 독립적으로 생성 가능
- **완전 랜덤** → 정렬 불가, 시간 정보 없음

### 단점

**인덱스 성능 저하**

RDB/NoSQL 모두 인덱스는 정렬된 상태를 유지한다. UUID는 랜덤값이라 새 레코드가 인덱스 중간에 계속 삽입된다.

```
B-Tree 인덱스에서
기존: [1, 3, 7, 12, 20]
UUID 삽입: [1, 3, 5_랜덤, 7, 9_랜덤, 12, 15_랜덤, 20]
→ 페이지 분할(Page Split) 빈번하게 발생 → 성능 저하
```

단조 증가 ID였다면 항상 맨 끝에 붙어서 페이지 분할이 없다.

### UUID v7 (2023~)

기존 UUID의 단점을 보완한 버전. 앞부분에 타임스탬프를 포함해서 시간순 정렬이 가능하다.

```
UUID v4: 완전 랜덤
UUID v7: [타임스탬프 48bit][랜덤 74bit]
```

---

## Snowflake ID

### 개념

Twitter가 2010년에 설계한 분산 ID 생성 방식. 64비트 정수로 구성된다.

```
┌──────────────────┬──────────────┬──────────────┬──────────────┐
│  timestamp (41)  │ datacenter(5)│ machine (5)  │ sequence(12) │
└──────────────────┴──────────────┴──────────────┴──────────────┘
```

| 필드 | 비트 | 설명 |
|---|---|---|
| timestamp | 41bit | 밀리초 단위 Unix 시간 (약 69년치) |
| datacenter | 5bit | 데이터센터 ID (최대 32개) |
| machine | 5bit | 머신 ID (데이터센터당 최대 32대) |
| sequence | 12bit | 같은 밀리초 내 순번 (최대 4096개/ms) |

### 생성 예시

```
timestamp:   1700000000000  (2023년 기준 특정 시각)
datacenter:  1
machine:     3
sequence:    42

→ ID: 7961600000000000042 (실제로는 비트 연산 결과)
```

### 특징

- **단조 증가** → 시간순 정렬 가능, 인덱스 성능 좋음
- **중앙 서버 불필요** → 각 머신이 독립적으로 생성
- **초당 최대 4,096,000개** 생성 가능 (머신 1대 기준)
- 64비트 정수 → Long 타입으로 저장, UUID보다 저장 공간 절반

### 주의사항

**시계 동기화 문제**

서버 시간이 뒤로 돌아가면 (NTP 동기화 등) 같은 ID가 생성될 수 있다.

```
t=1000 → ID 생성
시계 역행 발생
t=999  → 같은 구간의 ID 재생성 → 중복 위험
```

해결: 시계가 뒤로 가면 ID 생성을 대기하거나 예외 처리.

**머신 ID 관리**

각 서버에 고유한 머신 ID를 부여해야 한다. 자동화하지 않으면 동일 머신 ID가 중복 할당될 수 있다. 실무에서는 ZooKeeper나 Redis로 머신 ID를 동적 할당한다.

### 활용 사례

- Twitter — 원조
- Instagram — Snowflake 변형 사용
- Discord — Snowflake 기반 메시지 ID
- 카카오, 라인 등 대형 서비스 내부 ID 생성기

---

## ULID (Universally Unique Lexicographically Sortable Identifier)

### 개념

UUID의 유일성 + 시간순 정렬을 결합한 128비트 ID.

```
01ARZ3NDEKTSV4RRFFQ69G5FAV

[타임스탬프 48bit][랜덤 80bit]
```

### Snowflake와 비교

| | Snowflake | ULID |
|---|---|---|
| 비트 수 | 64bit | 128bit |
| 중앙 관리 | 머신 ID 필요 | 불필요 (랜덤) |
| 정렬 | 가능 | 가능 |
| 표현 | 숫자 | 문자열 (Base32) |
| 충돌 가능성 | 머신 ID 중복 시 | 랜덤 80bit라 사실상 없음 |

---

## AUTO_INCREMENT의 분산 환경 문제

단일 DB에서는 AUTO_INCREMENT가 편리하지만, 분산 환경에서는 문제가 생긴다.

```
노드 1: INSERT → ID 1 생성
노드 2: INSERT → ID 1 생성  ← 중복!
```

해결책으로 노드마다 시작값과 증가폭을 다르게 설정하는 방법이 있다.

```sql
-- 노드 1: 1, 3, 5, 7, ...
SET auto_increment_offset = 1;
SET auto_increment_increment = 2;

-- 노드 2: 2, 4, 6, 8, ...
SET auto_increment_offset = 2;
SET auto_increment_increment = 2;
```

단, 노드를 추가할 때마다 설정을 변경해야 하는 운영 부담이 있다.

---

## 전략 비교 및 선택 기준

| | UUID v4 | UUID v7 | Snowflake | ULID |
|---|---|---|---|---|
| 유일성 | ✅ | ✅ | ✅ | ✅ |
| 시간순 정렬 | ❌ | ✅ | ✅ | ✅ |
| 인덱스 성능 | 나쁨 | 좋음 | 좋음 | 좋음 |
| 구현 복잡도 | 매우 낮음 | 낮음 | 중간 | 낮음 |
| 중앙 서버 필요 | ❌ | ❌ | ❌ | ❌ |
| 저장 공간 | 16byte | 16byte | 8byte | 16byte |

### 선택 기준

```
단순함 최우선, 정렬 불필요      → UUID v4
정렬 필요, 구현 간단하게        → UUID v7 또는 ULID
대용량 처리, 저장 공간 최소화   → Snowflake
```

---

## 샤딩과의 관계

ID 생성 전략은 샤딩 성능과 직결된다.

```
UUID v4 (랜덤)  → 해시 샤딩에서 균등 분산 유리
                  범위 샤딩에서는 핫스팟 없음
                  인덱스 성능 나쁨

Snowflake (단조 증가) → 범위 샤딩에서 최신 데이터 핫스팟 위험
                       인덱스 성능 좋음
                       시간 기반 조회/파티셔닝에 유리
```

단조 증가 ID + 범위 샤딩은 핫스팟을 유발할 수 있으므로, 해시 샤딩과 조합하거나 랜덤 접두사 전략을 함께 사용한다.

---

## 참고 자료

- [Twitter Snowflake GitHub](https://github.com/twitter-archive/snowflake)
- [ULID 공식 스펙](https://github.com/ulid/spec)
- [UUID v7 RFC](https://www.rfc-editor.org/rfc/rfc9562)
- [System Design — ID Generator (ByteByteGo)](https://bytebytego.com/concepts/design-a-unique-id-generator)
