# Graph DB (그래프 데이터베이스)

## 개념

데이터를 **노드(Node)** 와 **엣지(Edge)** 로 표현한다. 관계 자체가 데이터다.

```
[철수] --친구--> [영희] --친구--> [민수]
[철수] --구매--> [아이폰]
[영희] --좋아요--> [아이폰]
```

---

## 왜 RDB로 관계를 표현하기 어려운가

SNS에서 "친구의 친구의 친구"를 찾는다고 하면 RDB는 Self Join을 해야 한다.

```sql
-- 1단계: 내 친구
SELECT friend_id FROM friends WHERE user_id = '철수'

-- 2단계: 친구의 친구
SELECT f2.friend_id
FROM friends f1
JOIN friends f2 ON f1.friend_id = f2.user_id
WHERE f1.user_id = '철수'

-- 3단계: 친구의 친구의 친구
SELECT f3.friend_id
FROM friends f1
JOIN friends f2 ON f1.friend_id = f2.user_id
JOIN friends f3 ON f2.friend_id = f3.user_id
WHERE f1.user_id = '철수'
```

깊이가 1 늘어날 때마다 JOIN이 추가된다. 두 가지 문제가 있다.

**코드 복잡도**: 깊이가 몇 단계인지 미리 알아야 한다. 동적으로 깊이를 조절하기 어렵다.

**성능 문제**:
```
users 1억 명, 1인당 평균 친구 50명

3단계 탐색
→ 1억 × 50 × 50 × 50 = 125조 번 비교
→ 인덱스를 써도 현실적으로 불가능
```

관계가 깊어질수록 지수적으로 느려진다.

---

## Index-Free Adjacency

Graph DB가 관계 탐색이 빠른 핵심 이유다.

RDB는 관계를 찾으려면 인덱스를 통해 테이블을 조회해야 한다.

Graph DB는 **노드가 자기 엣지 목록을 직접 가지고 있다.** 인덱스 없이 엣지를 따라가기만 하면 된다.

```mermaid
flowchart LR
    A[철수] -->|FRIEND| B[영희]
    B -->|FRIEND| C[민수]
    C -->|FRIEND| D[지수]
```

```
탐색
철수 노드 → 엣지 목록 직접 접근 → 영희
영희 노드 → 엣지 목록 직접 접근 → 민수
민수 노드 → 엣지 목록 직접 접근 → 지수
```

깊이가 깊어져도 엣지를 따라가는 비용은 O(관계 수)로 일정하다.

---

## 대표 DB

| DB | 특징 |
|---|---|
| Neo4j | 가장 널리 쓰이는 Graph DB. Cypher 쿼리 언어 |
| Amazon Neptune | AWS 완전관리형. Gremlin/SPARQL 지원 |
| ArangoDB | Multi-model DB. Graph + Document + Key-Value |

---

## Cypher 쿼리 언어 (Neo4j)

SQL 대신 **Cypher**를 사용한다. 노드와 관계를 시각적으로 표현한다.

```
(노드)
[:관계]
(노드1)-[:관계]->(노드2)
```

괄호가 노드, 화살표가 엣지다. 그림 그리듯 쿼리를 작성한다.

### 기본 조회

```cypher
-- 노드 조회
MATCH (u:User {name: "철수"})
RETURN u

-- 관계 조회 (철수의 친구)
MATCH (u:User {name: "철수"})-[:FRIEND]->(friend:User)
RETURN friend.name
```

SQL과 비교:
```sql
-- SQL
SELECT u2.name
FROM users u1
JOIN friends f ON u1.id = f.user_id
JOIN users u2 ON f.friend_id = u2.id
WHERE u1.name = '철수'
```

### 다단계 관계 탐색

```cypher
-- 2단계 (친구의 친구)
MATCH (u:User {name: "철수"})-[:FRIEND*2]->(fof:User)
RETURN fof.name

-- 동적 깊이 (1~6단계)
MATCH path = (u:User {name: "철수"})-[:FRIEND*1..6]->(fof:User)
RETURN fof.name, length(path) AS depth
```

SQL에서 JOIN을 N번 해야 했던 걸 `*2`, `*1..6` 으로 표현한다. 깊이를 동적으로 조절할 수 있다.

### 엣지에 속성 부여

Graph DB는 엣지 자체에도 속성을 저장할 수 있다.

```cypher
-- 관계 생성 (친구가 된 날짜, 친밀도)
MATCH (a:User {name: "철수"}), (b:User {name: "영희"})
CREATE (a)-[:FRIEND {since: 2023, closeness: 0.8}]->(b)

-- 관계 속성으로 필터링
MATCH (u:User {name: "철수"})-[r:FRIEND]->(friend)
WHERE r.since > 2020
RETURN friend.name, r.closeness
```

RDB에서는 관계 테이블에 컬럼을 추가해야 했다.

### 데이터 생성

```cypher
-- 노드 생성
CREATE (u:User {name: "철수", age: 25})
CREATE (p:Product {name: "아이폰", price: 1200000})

-- 관계 생성
MATCH (a:User {name: "철수"}), (p:Product {name: "아이폰"})
CREATE (a)-[:PURCHASED {date: "2024-01-01"}]->(p)
```

### 추천 시스템

```cypher
-- "철수가 구매한 상품을 산 다른 유저들이 구매한 상품" 추천
MATCH (u:User {name: "철수"})-[:PURCHASED]->(p:Product)
      <-[:PURCHASED]-(other:User)-[:PURCHASED]->(rec:Product)
WHERE NOT (u)-[:PURCHASED]->(rec)  -- 철수가 아직 안 산 것만
RETURN rec.name, count(*) AS score
ORDER BY score DESC
LIMIT 10
```

SQL로 구현하면 서브쿼리가 3~4단계 중첩되지만, Cypher는 관계 패턴으로 직관적으로 표현된다.

---

## RDB vs Graph DB 비교

| | RDB | Graph DB |
|---|---|---|
| 관계 표현 | 외래키 + JOIN | 엣지 직접 저장 |
| 관계 탐색 깊이 | 깊어질수록 지수적 느려짐 | O(관계 수)로 일정 |
| 관계 속성 | 별도 테이블 컬럼 | 엣지 속성으로 직접 저장 |
| 스키마 | 고정 | 유연 |
| 트랜잭션 | 강력한 ACID | 지원 (Neo4j ACID 지원) |
| 집계/분석 | 강력 | 약함 |

---

## 유스케이스

| 도메인 | 예시 |
|---|---|
| SNS 친구 추천 | 친구의 친구, 공통 친구 수 |
| 추천 시스템 | 협업 필터링, 구매 패턴 분석 |
| 사기 탐지 | 계좌/기기 간 관계 패턴 분석 |
| 권한 관리 | 조직도, 역할 계층, 접근 권한 전파 |
| 지식 그래프 | 개념 간 연관 관계 |
| 물류/경로 최적화 | 최단 경로, 네트워크 분석 |

---

## 주의사항

**Graph DB가 불리한 상황**

```
대용량 단순 쓰기    → Cassandra가 적합
유연한 문서 저장    → MongoDB가 적합
복잡한 집계/분석    → RDB가 적합
단순 키-값 조회     → Redis가 적합
```

Graph DB는 **관계 탐색이 핵심인 도메인**에서만 진가를 발휘한다. 범용 DB가 아니므로 대부분의 서비스에서 RDB/NoSQL과 함께 사용한다.

**데이터 모델링이 중요**

노드와 엣지 설계가 잘못되면 성능이 급격히 저하된다. 특히 **슈퍼 노드(Super Node)** 문제가 있다.

```
팔로워 1억 명인 유명인 노드
→ 엣지가 1억 개
→ 이 노드를 거치는 모든 쿼리가 느려짐
```

슈퍼 노드는 설계 단계에서 별도 처리 방법을 고려해야 한다.

---

## 참고 자료

- [Neo4j 공식 문서](https://neo4j.com/docs/)
- [Cypher 쿼리 언어 가이드](https://neo4j.com/docs/cypher-manual/current/)
- [Amazon Neptune 공식 문서](https://docs.aws.amazon.com/neptune/)
