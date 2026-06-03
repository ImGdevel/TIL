# NoSQL 인덱스 전략

## COLLSCAN 문제

NoSQL은 스키마가 없기 때문에 특정 값을 조회하려면 모든 문서를 열어봐야 한다. 이를 **COLLSCAN(Collection Scan)** 이라 한다. RDB의 Full Table Scan과 같은 개념이다.

```
인덱스 없이 age > 20 조회
→ 전체 컬렉션 순차 탐색
→ 문서 1억 개면 1억 개 전부 확인
→ 데이터 증가에 비례해 선형으로 느려짐
```

인덱스를 걸면 B-Tree 구조로 값 → 문서 위치를 빠르게 찾을 수 있다. NoSQL에서 인덱스는 선택이 아니라 필수다.

---

## RDB 인덱스와의 차이

기본 구조(B-Tree, 값 → 위치 매핑)는 같지만 NoSQL만의 특징이 있다.

### 중첩 필드 인덱스

JSON 구조의 중첩 필드에 직접 인덱스를 걸 수 있다.

```js
// 중첩 필드 인덱스
db.users.createIndex({ "user.address.city": 1 })

// 조회
db.users.find({ "user.address.city": "서울" })
```

RDB였으면 별도 테이블 + JOIN이 필요한 구조다.

### 배열 필드 인덱스 (Multikey Index)

배열 필드에 인덱스를 걸면 배열의 각 요소가 인덱스 엔트리가 된다.

```json
{ "name": "철수", "tags": ["java", "spring", "redis"] }
```

```js
db.posts.createIndex({ "tags": 1 })
```

```
인덱스 엔트리
"java"   → 철수 문서
"spring" → 철수 문서
"redis"  → 철수 문서
```

```js
// 인덱스 탐색 가능
db.posts.find({ tags: "spring" })
```

RDB였으면 별도 태그 테이블 + JOIN이 필요하다.

---

## 복합 인덱스 (Compound Index)

여러 필드를 묶어서 인덱스를 만든다.

```js
db.orders.createIndex({ user_id: 1, created_at: -1 })
```

```js
// 인덱스 탐색 가능 ✅
db.orders.find({ user_id: "123" }).sort({ created_at: -1 })

// 인덱스 탐색 불가 ❌ (선두 필드 없이 뒤 필드만 사용)
db.orders.find().sort({ created_at: -1 })
```

### ESR 규칙

복합 인덱스 필드 순서 원칙이다. RDB와 NoSQL 공통으로 적용되지만, NoSQL 옵티마이저는 RDB보다 단순해서 순서를 잘못 잡으면 COLLSCAN으로 빠지는 경우가 많다.

```
E — Equality  (일치 조건) 먼저
S — Sort      (정렬 조건) 다음
R — Range     (범위 조건) 마지막
```

**이유**: E로 후보군을 최대한 좁히고, S로 정렬, R로 범위 필터링하는 게 효율적이다.

```js
// 좋은 예: ESR 순서
db.orders.createIndex({ user_id: 1, created_at: -1, amount: 1 })
// E: user_id, S: created_at, R: amount

// 나쁜 예: R이 앞에 오면 범위 필터 후 정렬 → 비효율
db.orders.createIndex({ amount: 1, user_id: 1, created_at: -1 })
```

---

## 부분 인덱스 (Partial Index)

조건에 맞는 문서에만 인덱스를 건다. 인덱스 크기가 줄고 쓰기 성능이 좋아진다.

```js
// status가 "active"인 문서에만 인덱스 생성
db.orders.createIndex(
    { user_id: 1 },
    { partialFilterExpression: { status: "active" } }
)
```

주문 중 99%가 `completed`, 1%만 `active`인데 실제 조회는 `active`만 한다면 전체에 인덱스 거는 건 낭비다.

---

## Covered Query (인덱스 전용 조회)

쿼리에 필요한 모든 필드가 인덱스에 포함되면 **문서를 읽지 않고 인덱스만으로 응답**한다. 디스크 I/O가 없어서 매우 빠르다.

```js
// user_id, created_at, status 복합 인덱스
db.orders.createIndex({ user_id: 1, created_at: -1, status: 1 })

// 조회 필드가 전부 인덱스에 있으면 → Covered Query
db.orders.find(
    { user_id: "123" },
    { _id: 0, created_at: 1, status: 1 }  // 프로젝션으로 인덱스 필드만 요청
)
```

`_id: 0`을 명시하는 이유: `_id`는 기본적으로 반환되는데, 인덱스에 없으면 Covered Query가 깨진다.

---

## Sparse Index

값이 존재하는 문서에만 인덱스를 건다. `null`이나 필드 자체가 없는 문서는 인덱스에 포함되지 않는다.

```js
// email 필드가 있는 문서에만 인덱스
db.users.createIndex(
    { email: 1 },
    { sparse: true }
)
```

```json
{ "name": "철수", "email": "chul@example.com" }  ← 인덱스 포함
{ "name": "영희" }                                 ← email 없음, 인덱스 제외
```

선택적으로 입력하는 필드(소셜 로그인 사용자는 이메일 없을 수 있음)에 유니크 인덱스를 걸 때 유용하다. 일반 유니크 인덱스는 `null`도 중복으로 처리하지만, Sparse + Unique 조합은 값이 있는 문서끼리만 중복 체크한다.

---

## 인덱스 설계 주의사항

**인덱스가 많을수록 쓰기가 느려진다.**

쓰기 발생 시 인덱스도 함께 업데이트해야 한다. 읽기 최적화와 쓰기 성능 사이의 트레이드오프를 고려해야 한다.

```
읽기 많은 서비스  → 인덱스 적극 활용
쓰기 많은 서비스  → 인덱스 최소화
```

**실행 계획 확인**

```js
// explain으로 인덱스 사용 여부 확인
db.orders.find({ user_id: "123" }).explain("executionStats")

// COLLSCAN이면 인덱스 추가 필요
// IXSCAN이면 인덱스 사용 중
```

---

## 참고 자료

- [MongoDB 인덱스 공식 문서](https://www.mongodb.com/docs/manual/indexes/)
- [MongoDB ESR 규칙](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-rule/)
