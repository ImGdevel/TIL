# 효율적인 JPA 조회 전략

> Devon의 N+1 문제 해결과 성능 최적화 원칙

## 들어가며

JPA를 사용하면서 가장 흔하게 마주치는 문제는 **N+1 쿼리**다. 목록을 조회했을 뿐인데 수백, 수천 개의 쿼리가 발생하는 것을 보고 당황했던 경험이 누구나 있을 것이다.

이 문서는 내가 실무에서 겪은 성능 이슈와 그 해결 과정을 정리한 것이다. 주요 니즈는:

- **N+1 문제 완전 제거**: 목록 조회 시 추가 쿼리가 발생하지 않도록
- **메모리 효율**: 필요한 데이터만 조회
- **명시적 의도**: 코드만 봐도 어떤 쿼리가 나갈지 예측 가능
- **일관된 패턴**: 팀원 누구나 따라할 수 있는 표준화된 방식

---

## 1. LAZY 로딩: 모든 것의 시작점

### 기본 전략

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "member_id", nullable = false)
private Member member;
```

**모든 연관관계는 LAZY 로딩으로 시작한다.**

### EAGER 로딩을 쓰지 않는 이유

많은 사람이 N+1 문제를 EAGER 로딩으로 해결하려 한다:

```java
// 나쁜 예
@ManyToOne(fetch = FetchType.EAGER)
private Member member;
```

이렇게 하면:
- Post를 조회할 때마다 **무조건** Member도 함께 조회된다
- Member 정보가 필요 없는 상황에도 조인이 발생한다
- 여러 연관관계가 EAGER면 불필요한 조인이 기하급수적으로 늘어난다

```java
// Member가 필요 없는데도 조인 발생
Post post = postRepository.findById(1L);  // SELECT post LEFT JOIN member...
```

LAZY 로딩을 사용하면:
- 필요할 때만 명시적으로 조회
- 불필요한 조인 방지
- 쿼리 예측 가능성 향상

**원칙**: 기본은 LAZY, 필요한 곳에서만 Fetch Join 또는 Constructor Projection으로 해결

---

## 2. 조회 전략의 양대 산맥

JPA 조회는 크게 두 가지 패턴으로 나뉜다. 각각 명확한 사용 시점이 있다.

### 2.1 Fetch Join - 단건 조회의 최적해

```java
@Query("SELECT p FROM Post p JOIN FETCH p.member WHERE p.id = :id")
Optional<Post> findByIdWithMember(@Param("id") Long id);
```

**사용 시점**: 하나의 Post와 그 Member를 조회할 때

**왜 사용하는가**:
1. **완전한 엔티티가 필요**: 영속성 컨텍스트에서 관리되는 실제 엔티티
2. **변경 감지**: 엔티티를 수정하면 자동으로 UPDATE 쿼리
3. **쿼리 1개**: Post와 Member를 한 번에 조회

```java
// Before - N+1 발생
Post post = postRepository.findById(1L).get();
String nickname = post.getMember().getNickname();  // 추가 쿼리!

// After - Fetch Join
Post post = postRepository.findByIdWithMember(1L).get();
String nickname = post.getMember().getNickname();  // 쿼리 없음
```

**단점**: 컬렉션 Fetch Join은 페이징이 깨진다 (뒤에서 설명)

### 2.2 Constructor Projection - 목록 조회의 정답

```java
@Override
public Page<PostQueryDto> findAllActiveWithMemberAsDto(Pageable pageable) {
    List<PostQueryDto> content = queryFactory
        .select(Projections.constructor(PostQueryDto.class,
            post.id, post.title, post.createdAt,
            post.viewsCount, post.likeCount, post.commentCount,
            member.id, member.nickname, member.email
        ))
        .from(post)
        .join(post.member, member)  // 일반 JOIN
        .where(isNotDeleted())
        .orderBy(getOrderSpecifiers(pageable))
        .offset(pageable.getOffset())
        .limit(pageable.getPageSize())
        .fetch();

    JPAQuery<Long> countQuery = queryFactory
        .select(post.count())
        .from(post)
        .where(isNotDeleted());

    return PageableExecutionUtils.getPage(content, pageable, countQuery::fetchOne);
}
```

**사용 시점**: 목록 조회, 검색, 페이징

**왜 사용하는가**:

1. **필요한 필드만 SELECT**
   ```sql
   -- Entity 조회 (Fetch Join)
   SELECT p.*, m.* FROM post p JOIN member m ...  -- 모든 컬럼

   -- DTO 조회 (Constructor Projection)
   SELECT p.id, p.title, ..., m.id, m.nickname FROM post p JOIN member m ...  -- 필요한 컬럼만
   ```

2. **메모리 효율**
   ```java
   // 100개 Post 조회 시
   // Entity: Post 객체 100개 + Member 객체 100개 = 200개 객체
   // DTO: PostQueryDto 100개 = 100개 객체
   ```

3. **N+1 문제 없음**
   ```java
   // JOIN으로 한 번에 모든 데이터 포함
   List<PostQueryDto> posts = postRepository.findAllActiveWithMemberAsDto(pageable);
   // Member 정보 접근해도 추가 쿼리 없음
   posts.forEach(dto -> System.out.println(dto.memberNickname()));
   ```

4. **영속성 컨텍스트 관리 불필요**
   - DTO는 영속 상태가 아니므로 변경 감지 오버헤드 없음
   - 읽기 전용 조회에 최적

### DTO 정의는 Record로

```java
public record PostQueryDto(
    Long postId,
    String title,
    Instant createdAt,
    Long viewsCount,
    Long likeCount,
    Long commentCount,
    Long memberId,
    String memberNickname,
    String memberEmail
) { }
```

**Record를 사용하는 이유**:
- 불변성 보장 (final 필드)
- 간결한 코드 (자동 getter, equals, hashCode)
- 의도 명확 (데이터 전달 객체임을 표현)

---

## 3. Fetch Join의 함정과 해결

### 컬렉션 Fetch Join + Pagination 문제

```java
// 위험한 코드
@Query("SELECT p FROM Post p JOIN FETCH p.comments")
Page<Post> findAllWithComments(Pageable pageable);
```

**문제점**:
1. **메모리에서 페이징**: DB가 아닌 애플리케이션 메모리에서 LIMIT 처리
2. **데이터 폭증**: Post 1개당 Comment N개면 Row가 N배로 늘어남
3. **OOM 위험**: 전체 데이터를 메모리에 로드

```sql
-- 실제 실행되는 쿼리
SELECT * FROM post p LEFT JOIN comment c ON ...
-- LIMIT 10 OFFSET 0  <-- 이게 없다!
```

Hibernate가 경고를 띄우는 이유:
```
WARN: HHH000104: firstResult/maxResults specified with collection fetch; applying in memory!
```

### 해결 방법 1: @BatchSize

```java
@Entity
public class Post {
    @OneToMany(mappedBy = "post", fetch = FetchType.LAZY)
    @BatchSize(size = 100)
    private List<Comment> comments;
}
```

또는 전역 설정:
```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 100
```

**동작 원리**:
```java
// Post 10개 조회
List<Post> posts = postRepository.findAll(PageRequest.of(0, 10));

// Comment 접근 시
posts.forEach(post -> post.getComments().size());

// 쿼리 1개로 처리
// SELECT * FROM comment WHERE post_id IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
```

### 해결 방법 2: Constructor Projection (권장)

```java
// Post 목록 + Comment 개수를 DTO로
.select(Projections.constructor(PostQueryDto.class,
    post.id,
    post.title,
    comment.count()  // 집계
))
.from(post)
.leftJoin(post.comments, comment)
.groupBy(post.id)
.offset(pageable.getOffset())
.limit(pageable.getPageSize())
.fetch();
```

**장점**:
- 정확한 페이징
- 메모리 효율
- N+1 문제 없음

---

## 4. QueryDSL: 동적 쿼리의 정답

### 왜 QueryDSL인가

JPQL은 동적 쿼리가 어렵다:

```java
// 검색 조건이 있을 수도, 없을 수도
public List<Post> search(String keyword, LocalDate startDate) {
    String jpql = "SELECT p FROM Post p WHERE 1=1";
    if (keyword != null) jpql += " AND p.title LIKE :keyword";
    if (startDate != null) jpql += " AND p.createdAt >= :startDate";
    // 문자열 조작... 끔찍하다
}
```

QueryDSL은 타입 안전하고 읽기 쉽다:

```java
@Override
public Page<PostQueryDto> searchByTitleOrContent(String keyword, Pageable pageable) {
    BooleanExpression condition = isNotDeleted().and(keywordSearch(keyword));

    List<PostQueryDto> content = queryFactory
        .select(Projections.constructor(PostQueryDto.class, ...))
        .from(post)
        .join(post.member, member)
        .where(condition)
        .orderBy(getOrderSpecifiers(pageable))
        .offset(pageable.getOffset())
        .limit(pageable.getPageSize())
        .fetch();

    JPAQuery<Long> countQuery = queryFactory
        .select(post.count())
        .from(post)
        .where(condition);

    return PageableExecutionUtils.getPage(content, pageable, countQuery::fetchOne);
}
```

### BooleanExpression 분리의 힘

```java
// 조건을 메서드로 분리
private BooleanExpression isNotDeleted() {
    return post.isDeleted.eq(false);
}

private BooleanExpression titleContains(String keyword) {
    return keyword != null ? post.title.containsIgnoreCase(keyword) : null;
}

private BooleanExpression contentContains(String keyword) {
    return keyword != null ? post.content.containsIgnoreCase(keyword) : null;
}

// 조합
private BooleanExpression keywordSearch(String keyword) {
    return titleContains(keyword).or(contentContains(keyword));
}
```

**장점**:
1. **재사용성**: 같은 조건을 여러 쿼리에서 사용
2. **가독성**: `where(isNotDeleted().and(keywordSearch(keyword)))`
3. **Null 안전**: null을 반환하면 조건에서 제외됨
4. **조합 가능**: and, or로 자유롭게 조합

### Count 쿼리 분리 최적화

```java
// Content 쿼리 - JOIN 필요
List<PostQueryDto> content = queryFactory
    .select(...)
    .from(post)
    .join(post.member, member)  // JOIN
    .where(...)
    .orderBy(...)
    .fetch();

// Count 쿼리 - JOIN 불필요
JPAQuery<Long> countQuery = queryFactory
    .select(post.count())
    .from(post)
    .where(...);  // 같은 조건, 하지만 JOIN 없음

return PageableExecutionUtils.getPage(content, pageable, countQuery::fetchOne);
```

**왜 분리하는가**:
- Count 쿼리는 JOIN이 필요 없는 경우가 많음 (where 조건만 같으면 됨)
- ORDER BY도 불필요
- 성능 향상

```sql
-- Content 쿼리
SELECT p.id, p.title, m.nickname
FROM post p JOIN member m ON p.member_id = m.id
WHERE p.is_deleted = false
ORDER BY p.created_at DESC
LIMIT 10 OFFSET 0;

-- Count 쿼리
SELECT COUNT(p.id)
FROM post p
WHERE p.is_deleted = false;
-- JOIN 없음, ORDER BY 없음
```

---

## 5. N+1 해결 무기고

실무에서 마주친 N+1 문제와 해결 방법들을 정리했다.

### 5.1 Scalar Projection - 특정 값만 조회

**문제 상황**:
```java
// Comment의 postId만 필요한데 전체 Post를 조회?
Comment comment = commentRepository.findById(commentId).get();
Long postId = comment.getPost().getId();  // Post SELECT 쿼리 발생
```

**해결**:
```java
@Query("SELECT c.post.id FROM Comment c WHERE c.id = :commentId")
Optional<Long> findPostIdByCommentId(@Param("commentId") Long commentId);

// 사용
Long postId = commentRepository.findPostIdByCommentId(commentId).get();
// Post를 조회하지 않고 FK만 가져옴
```

**효과**: Entity 로딩 없이 필요한 값만 조회

### 5.2 getReferenceById() - Proxy 활용

**문제 상황**:
```java
// Comment 생성 시 Post 전체가 필요한가?
Post post = postRepository.findById(postId).get();  // SELECT 쿼리
Comment comment = Comment.create(member, post, content);
commentRepository.save(comment);  // post_id만 필요한데...
```

**해결**:
```java
// Post 존재 여부만 확인
validatePostExists(postId);

// Proxy 객체만 가져옴 (SELECT 없음)
Post post = postRepository.getReferenceById(postId);

Comment comment = Comment.create(member, post, content);
commentRepository.save(comment);  // post의 FK만 사용
```

**getReferenceById vs findById**:

| 메서드 | 반환 | 쿼리 | 사용 시점 |
|-------|------|------|----------|
| **findById** | 실제 엔티티 | SELECT 즉시 실행 | 엔티티 데이터가 필요할 때 |
| **getReferenceById** | Proxy 객체 | FK 접근 시에만 실행 | FK만 필요할 때 |

**주의**: Proxy이므로 실제 필드에 접근하면 쿼리가 나간다. FK 설정 용도로만 사용.

### 5.3 Bulk Update - 엔티티 없이 업데이트

**문제 상황**:
```java
// 좋아요 카운터 증가를 위해 Post를 조회?
Post post = postRepository.findById(postId).get();  // SELECT
post.incrementLikeCount();  // 변경 감지로 UPDATE
// 총 2개 쿼리
```

**해결**:
```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("UPDATE Post p SET p.likeCount = p.likeCount + 1 WHERE p.id = :postId")
int incrementLikeCount(@Param("postId") Long postId);

// 사용
postRepository.incrementLikeCount(postId);
// UPDATE 쿼리 1개만
```

**파라미터 설명**:
- `clearAutomatically = true`: 쿼리 실행 후 영속성 컨텍스트 자동 초기화
- `flushAutomatically = true`: 쿼리 실행 전 영속성 컨텍스트 자동 flush

**왜 필요한가**:
```java
// clearAutomatically 없이 실행하면
Post post = postRepository.findById(1L);  // likeCount = 10
postRepository.incrementLikeCount(1L);     // DB: likeCount = 11
System.out.println(post.getLikeCount());   // 10 (영속성 컨텍스트 캐시)
```

영속성 컨텍스트와 DB 간 불일치를 방지하기 위해 자동 초기화가 필요하다.

### 5.4 existsById vs count

**문제**:
```java
// 존재 여부만 확인하는데 count?
long count = postRepository.count();  // SELECT COUNT(*) FROM post
if (count > 0) { ... }
```

**해결**:
```java
if (postRepository.existsById(postId)) { ... }
```

**차이**:
```sql
-- existsById
SELECT 1 FROM post WHERE id = ? LIMIT 1

-- count
SELECT COUNT(*) FROM post WHERE id = ?
```

existsById는 첫 번째 매치에서 즉시 반환하므로 더 빠르다.

### 5.5 Batch Fetch Size - 최후의 방어선

```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 100
```

**동작 원리**:
```java
// LAZY 로딩된 연관 엔티티를 IN 쿼리로 일괄 조회
List<Post> posts = postRepository.findAll();  // 100개
posts.forEach(post -> post.getMember().getNickname());

// Batch Size 없으면: 100개 쿼리
// SELECT * FROM member WHERE id = 1
// SELECT * FROM member WHERE id = 2
// ...

// Batch Size 있으면: 1개 쿼리
// SELECT * FROM member WHERE id IN (1, 2, 3, ..., 100)
```

**권장 값**: 100~1000 (트래픽과 데이터 특성에 따라)

**효과**:
- Fetch Join을 깜빡했을 때의 최후 방어선
- LAZY 로딩을 안전하게 사용 가능
- 코드 수정 없이 전역 적용

---

## 6. 조회 전략 선택 가이드

| 시나리오 | 전략 | 이유 |
|---------|------|------|
| **단건 + 연관 엔티티** | Fetch Join | 완전한 엔티티, 쿼리 1개 |
| **목록 + 연관 데이터** | Constructor Projection | 필요한 필드만, 메모리 효율 |
| **동적 검색** | QueryDSL + BooleanExpression | 조건 조합, 가독성 |
| **카운터 증감** | Bulk Update | Entity 로딩 불필요 |
| **FK만 필요** | getReferenceById() | Proxy, 쿼리 없음 |
| **존재 확인** | existsById | count보다 빠름 |
| **특정 값만** | Scalar Projection | Entity 로딩 불필요 |
| **컬렉션 조회** | @BatchSize 또는 별도 쿼리 | 페이징 안전 |

---

## 7. 안티패턴과 해결

### ❌ EAGER 로딩으로 N+1 해결 시도

```java
// 나쁜 예
@ManyToOne(fetch = FetchType.EAGER)
private Member member;
```

**문제**: 필요 없을 때도 항상 조인, 제어 불가

**해결**: LAZY + 명시적 Fetch Join 또는 Constructor Projection

### ❌ 컬렉션 Fetch Join + Pagination

```java
// 위험
@Query("SELECT p FROM Post p JOIN FETCH p.comments")
Page<Post> findAll(Pageable pageable);
```

**문제**: 메모리에서 페이징, OOM 위험

**해결**: Constructor Projection 또는 @BatchSize

### ❌ 불필요한 Entity 로딩

```java
// 비효율
Post post = postRepository.findById(postId).get();
post.incrementViewCount();
```

**해결**: Bulk Update 사용

### ❌ N+1 발생 방치

```java
// 위험
List<Post> posts = postRepository.findAll();
posts.forEach(post -> post.getMember().getNickname());  // N개 쿼리
```

**해결**: Fetch Join, Constructor Projection, 또는 Batch Size

---

## 8. 핵심 원칙 정리

### 모든 연관관계는 LAZY
불필요한 조인을 방지하고, 조회를 명시적으로 제어한다.

### 단건은 Fetch Join, 목록은 Constructor Projection
각각의 사용 시점이 명확하다. 혼동하지 말자.

### QueryDSL로 동적 쿼리
JPQL 문자열 조작은 버그의 온상. QueryDSL로 타입 안전하게.

### Entity 로딩 최소화
FK만 필요하면 getReferenceById, 카운터 수정은 Bulk Update.

### Batch Fetch Size는 필수
깜빡한 N+1을 막아주는 최후의 보루. 반드시 설정.

### 읽기는 readOnly
```java
@Transactional(readOnly = true)
```
Dirty Checking 비활성화로 성능 향상, 의도 명확화.

---

## 마치며

JPA 성능 최적화는 **쿼리를 예측 가능하게** 만드는 일이다. 코드를 보면 몇 개의 쿼리가 나갈지 알 수 있어야 한다.

이 문서의 전략들은 완벽하지 않다. 하지만 대부분의 상황에서 "이 경우엔 이렇게"라는 명확한 가이드를 제공한다.

중요한 건 **왜 이 전략을 선택했는가**를 이해하는 것이다. 맥락이 다르다면 다른 전략을 써야 한다. 다만, 항상 쿼리 개수를 세어보자.

N+1 문제는 코드 리뷰에서 잡기 어렵다. 실행해봐야 안다. 그래서 원칙이 중요하다.
