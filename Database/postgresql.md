# PostgreSQL

## 개요

PostgreSQL은 오픈소스 객체-관계형 데이터베이스다. 강력한 쿼리 플래너, 풍부한 타입 시스템, 다양한 인덱스 종류, Extension을 통한 확장성이 강점이다.

```
PostgreSQL이 유리한 상황
→ 쿼리가 복잡한 경우
→ 배열, JSONB, 범위 타입 등 확장 타입이 필요한 경우
→ 정교한 쿼리 최적화가 필요한 경우
→ 지리 데이터, 시계열 등 특수 도메인
```

---

## MVCC — Dead Tuple 방식

PostgreSQL은 MVCC(Multi-Version Concurrency Control)를 Dead Tuple 방식으로 구현한다.

```
UPDATE users SET name = '철수수' WHERE id = 1

테이블: { id: 1, name: "철수",   xmin: 100, xmax: 200 }  ← Dead Tuple
        { id: 1, name: "철수수", xmin: 200, xmax: null } ← Live Tuple
```

UPDATE/DELETE해도 기존 행을 즉시 삭제하지 않고 Dead Tuple로 표시한다. VACUUM이 Dead Tuple을 주기적으로 정리한다.

**xmin / xmax**
- `xmin` — 이 행을 생성한 트랜잭션 ID
- `xmax` — 이 행을 삭제/변경한 트랜잭션 ID (없으면 null)

읽기 시 현재 트랜잭션 기준으로 보여야 할 버전을 판단해서 일관된 스냅샷을 제공한다.

---

## VACUUM

PostgreSQL의 필수 유지보수 작업이다.

### VACUUM의 역할

```
1. Dead Tuple 정리         → Table Bloat 방지
2. 통계 정보 업데이트      → 쿼리 플래너 최적화
3. Transaction ID Wraparound 방지
```

### Transaction ID Wraparound

PostgreSQL은 트랜잭션 ID를 32비트 정수로 관리한다.

```
최대값: 약 42억
→ 42억 번 트랜잭션 후 ID가 다시 0으로 돌아옴
→ 과거 데이터가 미래 데이터처럼 보이는 문제 발생
→ 최악의 경우 데이터 손실
```

VACUUM이 오래된 트랜잭션 ID를 정리해서 이 문제를 방지한다.

### VACUUM 종류

```sql
-- 일반 VACUUM: Dead Tuple 제거, 공간 재사용 표시 (OS 반환 X)
VACUUM users;

-- VACUUM FULL: 테이블 완전 재구성, OS에 공간 반환
-- 단, 실행 중 테이블 전체 락 → 서비스 중단 위험
VACUUM FULL users;

-- ANALYZE: 통계 정보만 업데이트
ANALYZE users;

-- VACUUM + ANALYZE 동시
VACUUM ANALYZE users;
```

### Autovacuum 설정

```
# postgresql.conf
autovacuum = on
autovacuum_vacuum_threshold = 50       # Dead Tuple 50개 이상이면
autovacuum_vacuum_scale_factor = 0.2   # 또는 전체의 20% 이상이면 실행
```

쓰기가 폭발적으로 많으면 Autovacuum이 따라가지 못해 Table Bloat이 급격히 증가할 수 있다. 수동 VACUUM 또는 Autovacuum 설정 튜닝이 필요하다.

---

## 쿼리 플래너

PostgreSQL 쿼리 플래너는 여러 실행 계획 후보를 놓고 비용을 계산해서 최적을 선택한다.

```
복잡한 JOIN 순서 최적화
서브쿼리 → JOIN 변환 (Subquery Flattening)
파티션 프루닝
병렬 쿼리 실행
JIT 컴파일
```

### JIT (Just-In-Time) 컴파일

PostgreSQL 11부터 복잡한 쿼리를 LLVM으로 컴파일해서 실행한다.

```sql
-- JIT 활성화 여부 확인
EXPLAIN ANALYZE SELECT ...;
-- JIT: enabled
```

단순 쿼리는 JIT 컴파일 비용이 오히려 손해다. 복잡한 집계/분석 쿼리에서만 효과가 있다.

---

## 확장 타입 시스템

### 배열 타입

```sql
CREATE TABLE posts (
    id   INT,
    tags TEXT[]
);

INSERT INTO posts VALUES (1, ARRAY['java', 'spring', 'redis']);

-- 특정 값 포함 여부
SELECT * FROM posts WHERE 'java' = ANY(tags);

-- 배열 교집합 (겹치는 값 있으면 true)
SELECT * FROM posts WHERE tags && ARRAY['java', 'spring'];

-- 배열 원소 개수
SELECT id, array_length(tags, 1) FROM posts;
```

### JSONB

```
JSON  → 텍스트 그대로 저장, 인덱스 불가
JSONB → 바이너리로 파싱해서 저장, 인덱스 가능
```

실무에서는 거의 JSONB만 사용한다.

```sql
CREATE TABLE events (
    id      INT,
    payload JSONB
);

-- 필드 조회
SELECT payload->>'user_id' FROM events;
SELECT payload->'address'->>'city' FROM events;

-- GIN 인덱스
CREATE INDEX ON events USING GIN (payload);

-- 포함 여부 검색
SELECT * FROM events WHERE payload @> '{"status": "PAID"}';
```

### 범위 타입

```sql
CREATE TABLE reservations (
    id     INT,
    room   INT,
    period DATERANGE
);

INSERT INTO reservations VALUES (1, 101, '[2024-01-01, 2024-01-05)');

-- 겹치는 예약 찾기
SELECT * FROM reservations
WHERE period && '[2024-01-03, 2024-01-07)';

-- 특정 날짜 포함 여부
SELECT * FROM reservations
WHERE period @> '2024-01-03'::DATE;
```

숙박 예약, 회의실 예약처럼 기간 겹침 체크를 SQL 한 줄로 처리할 수 있다.

### 기타 타입

```sql
INET    -- IP 주소 (192.168.0.1)
CIDR    -- 네트워크 주소 (192.168.0.0/24)
UUID    -- UUID 타입
MONEY   -- 화폐 타입
```

---

## 인덱스 종류

### B-Tree (기본)

```sql
CREATE INDEX ON users (email);
-- 범위 조회, 정렬, 동등 비교 모두 지원
```

### GIN (Generalized Inverted Index)

배열, JSONB, 전문 검색에 사용한다. 역인덱스 구조로 "이 값이 포함된 행"을 빠르게 찾는다.

```sql
-- 배열 컬럼
CREATE INDEX ON posts USING GIN (tags);

-- JSONB
CREATE INDEX ON events USING GIN (payload);

-- 전문 검색
CREATE INDEX ON articles USING GIN (to_tsvector('english', content));

SELECT * FROM articles
WHERE to_tsvector('english', content) @@ to_tsquery('java & spring');
```

### BRIN (Block Range Index)

대용량 시계열 데이터에 특화된 인덱스다. 블록 범위만 저장해서 크기가 극도로 작다.

```sql
CREATE INDEX ON logs USING BRIN (created_at);
```

```
B-Tree 인덱스: 1억 건 → 수 GB
BRIN 인덱스:   1억 건 → 수 MB
```

데이터가 물리적으로 정렬된 순서로 쌓이는 경우에만 효과적이다. 시계열 로그에 적합하다.

### 부분 인덱스 (Partial Index)

```sql
CREATE INDEX ON orders (user_id)
WHERE status = 'ACTIVE';
```

전체 주문 중 1%만 ACTIVE라면 인덱스 크기와 쓰기 부담이 1/100로 줄어든다.

### 표현식 인덱스

```sql
CREATE INDEX ON users (LOWER(email));

SELECT * FROM users WHERE LOWER(email) = 'chul@example.com';
```

### 인덱스 종류 요약

| 인덱스 | 용도 |
|---|---|
| B-Tree | 기본, 범위 조회, 정렬 |
| Hash | 동등 비교만 |
| GIN | 배열, JSONB, 전문 검색 |
| GiST | 지리 데이터, 범위 타입 |
| BRIN | 대용량 시계열 (물리 정렬 전제) |

---

## 테이블 파티셔닝

### 파티셔닝이 필요한 상황

```
orders 테이블: 50억 건
→ 이번 달 주문 조회
→ 인덱스 있어도 인덱스 자체가 너무 큼

파티셔닝 후
→ orders_2024_12: 500만 건만 스캔
→ 인덱스도 파티션별로 작음
```

### RANGE 파티셔닝

```sql
CREATE TABLE orders (
    id         BIGINT,
    created_at DATE,
    amount     DECIMAL
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE orders_2025 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
```

### LIST 파티셔닝

```sql
CREATE TABLE users (
    id      BIGINT,
    country TEXT
) PARTITION BY LIST (country);

CREATE TABLE users_kr PARTITION OF users FOR VALUES IN ('KR');
CREATE TABLE users_us PARTITION OF users FOR VALUES IN ('US');
```

### HASH 파티셔닝

```sql
CREATE TABLE events (
    id      BIGINT,
    user_id BIGINT
) PARTITION BY HASH (user_id);

CREATE TABLE events_0 PARTITION OF events
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE events_1 PARTITION OF events
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);
```

### 파티션 프루닝

```sql
EXPLAIN SELECT * FROM orders
WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31';

-- Append
--   → Seq Scan on orders_2024  ← 해당 파티션만 스캔
--   (orders_2023, orders_2025는 스캔 안 함)
```

---

## CTE (Common Table Expression)

### 기본 CTE

```sql
WITH monthly_sales AS (
    SELECT DATE_TRUNC('month', created_at) AS month,
           SUM(amount) AS total
    FROM orders
    WHERE status = 'PAID'
    GROUP BY month
)
SELECT month, total,
       total - LAG(total) OVER (ORDER BY month) AS diff
FROM monthly_sales
ORDER BY month;
```

### 재귀 CTE

자기 자신을 참조해서 계층 구조를 탐색한다.

```
1. 앵커 쿼리 (시작점)  → 처음 한 번 실행
2. 재귀 쿼리          → 결과가 없을 때까지 반복 실행
```

**카테고리 계층 구조**

```sql
WITH RECURSIVE sub_categories AS (
    SELECT id, name, parent_id, 0 AS level
    FROM categories
    WHERE name = '전자기기'

    UNION ALL

    SELECT c.id, c.name, c.parent_id, sc.level + 1
    FROM categories c
    JOIN sub_categories sc ON c.parent_id = sc.id
)
SELECT id, name, level
FROM sub_categories
ORDER BY level, name;
```

**SNS 친구 탐색**

```sql
WITH RECURSIVE friends_of_friends AS (
    SELECT friend_id, 1 AS depth
    FROM friends
    WHERE user_id = '철수'

    UNION ALL

    SELECT f.friend_id, fof.depth + 1
    FROM friends f
    JOIN friends_of_friends fof ON f.user_id = fof.friend_id
    WHERE fof.depth < 6
)
SELECT DISTINCT friend_id, depth
FROM friends_of_friends
ORDER BY depth;
```

---

## Window Function

집계는 하는데 행이 사라지지 않는 함수다.

```sql
-- GROUP BY: 행이 합쳐짐 → 부서 수만큼 행
SELECT department, AVG(salary) FROM employees GROUP BY department;

-- Window Function: 행이 유지됨 → 직원 수만큼 행
SELECT name, department, salary,
       AVG(salary) OVER (PARTITION BY department) AS dept_avg
FROM employees;
```

### 문법 구조

```sql
함수() OVER (
    PARTITION BY 그룹 기준 컬럼
    ORDER BY    정렬 기준 컬럼
    ROWS/RANGE BETWEEN ...
)
```

### 순위 함수

```sql
SELECT name, salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num,  -- 1,2,3,4,5
    RANK()       OVER (ORDER BY salary DESC) AS rank,     -- 1,2,2,4,5 (동점 건너뜀)
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense     -- 1,2,2,3,4 (동점 안 건너뜀)
FROM employees;
```

**부서별 순위**

```sql
SELECT name, department, salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM employees;
```

### LAG / LEAD

```sql
SELECT order_date, amount,
    LAG(amount)  OVER (ORDER BY order_date) AS prev_amount,
    LEAD(amount) OVER (ORDER BY order_date) AS next_amount,
    amount - LAG(amount) OVER (ORDER BY order_date) AS diff
FROM orders;
```

### 누적 합계 / 이동 평균

```sql
-- 누적 합계
SELECT order_date, amount,
    SUM(amount) OVER (ORDER BY order_date) AS cumulative_sum
FROM orders;

-- 최근 7일 이동 평균
SELECT order_date, amount,
    AVG(amount) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7d
FROM orders;
```

---

## Extension (확장 기능)

```sql
SELECT * FROM pg_extension;
CREATE EXTENSION IF NOT EXISTS extension_name;
```

### pg_stat_statements — 쿼리 성능 모니터링

```sql
CREATE EXTENSION pg_stat_statements;

SELECT query, calls,
       total_exec_time / calls AS avg_ms,
       rows / calls AS avg_rows
FROM pg_stat_statements
ORDER BY avg_ms DESC
LIMIT 10;
```

### PostGIS — 지리/공간 데이터

```sql
CREATE EXTENSION postgis;

CREATE TABLE stores (
    id       INT,
    name     TEXT,
    location GEOMETRY(Point, 4326)
);

INSERT INTO stores VALUES (
    1, '스타벅스강남점',
    ST_SetSRID(ST_MakePoint(127.027, 37.497), 4326)
);

SELECT name,
       ST_Distance(location, ST_MakePoint(127.025, 37.495)::geography) AS distance
FROM stores
WHERE ST_DWithin(location::geography, ST_MakePoint(127.025, 37.495)::geography, 5000)
ORDER BY distance;
```

### uuid-ossp — UUID 생성

```sql
CREATE EXTENSION "uuid-ossp";

CREATE TABLE users (
    id   UUID DEFAULT uuid_generate_v4(),
    name TEXT
);
```

### pg_trgm — 유사 문자열 검색

LIKE 앞에 `%` 붙이면 인덱스를 못 쓰는 문제를 해결한다.

```sql
CREATE EXTENSION pg_trgm;

CREATE INDEX ON products USING GIN (name gin_trgm_ops);

SELECT * FROM products WHERE name LIKE '%아이폰%';

SELECT name, similarity(name, '아이폰') AS sim
FROM products
WHERE similarity(name, '아이폰') > 0.3
ORDER BY sim DESC;
```

### TimescaleDB — 시계열 데이터

```sql
CREATE EXTENSION timescaledb;

SELECT create_hypertable('sensor_data', 'time');

SELECT time_bucket('1 hour', time) AS hour,
       AVG(temperature)
FROM sensor_data
WHERE time > NOW() - INTERVAL '24 hours'
GROUP BY hour
ORDER BY hour;
```

### Extension 요약

| Extension | 용도 |
|---|---|
| pg_stat_statements | 쿼리 성능 모니터링 |
| PostGIS | 지리/공간 데이터 |
| uuid-ossp | UUID 생성 |
| pg_trgm | 유사 문자열 검색, LIKE 인덱스 |
| TimescaleDB | 시계열 데이터 최적화 |

---

## 참고 자료

- [PostgreSQL 공식 문서](https://www.postgresql.org/docs/)
- [VACUUM 공식 문서](https://www.postgresql.org/docs/current/sql-vacuum.html)
- [PostgreSQL 인덱스 타입](https://www.postgresql.org/docs/current/indexes-types.html)
- [PostgreSQL Window Function](https://www.postgresql.org/docs/current/tutorial-window.html)
- [PostGIS 공식 문서](https://postgis.net/documentation/)
- [TimescaleDB 공식 문서](https://docs.timescale.com/)
