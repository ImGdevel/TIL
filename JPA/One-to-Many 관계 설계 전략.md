# JPA One-to-Many 관계 설계 전략

> Devon의 양방향 관계 최소화와 도메인 분리 원칙

## 들어가며

JPA를 사용하다 보면 **양방향 관계**를 습관적으로 선언하게 된다. Post와 Comment가 있다면 자연스럽게 다음과 같이 작성한다:

```java
@Entity
public class Post {
    @OneToMany(mappedBy = "post", fetch = FetchType.LAZY)
    private List<Comment> comments = new ArrayList<>();
}
```

"나중에 필요할 수도 있으니까", "양방향 탐색이 편하니까" 같은 이유로 말이다.

하지만 이것이 **진짜 필요한 관계인가**? 이 문서는 내가 실무에서 One-to-Many 양방향 관계를 최소화하게 된 이유와 그 전략을 정리한 것이다.

---

## 1. 핵심 원칙: 양방향 관계는 지양하자

### 나의 코드 스타일

```java
@Entity
@Table(name = "post")
public class Post extends BaseTimeEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;
    private String content;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id", nullable = false)
    private Member member;

    // ❌ OneToMany는 선언하지 않는다
    // @OneToMany(mappedBy = "post", fetch = FetchType.LAZY)
    // private List<Comment> comments = new ArrayList<>();
}
```

```java
@Entity
@Table(name = "comment")
public class Comment extends BaseTimeEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String content;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "post_id", nullable = false)
    private Post post;  // ✅ 단방향만 유지

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id", nullable = false)
    private Member member;
}
```

**핵심**: Comment는 Post를 알아야 하지만, **Post는 Comment를 몰라도 된다.**

---

## 2. 왜 양방향을 피하는가: 3가지 근본적인 이유

### 2.1 Fetch Join 시 메모리 문제

#### 문제 상황

Post 목록을 조회하면서 Comment도 함께 가져오고 싶다면?

```java
@Query("SELECT p FROM Post p JOIN FETCH p.comments")
List<Post> findAllWithComments();
```

**실행되는 SQL**:
```sql
SELECT p.*, c.*
FROM post p
LEFT JOIN comment c ON p.id = c.post_id
```

**결과 데이터**:
```
post_id | post_title        | comment_id | comment_content
--------|-------------------|------------|------------------
1       | "JPA 입문"         | 1          | "좋은 글이네요"
1       | "JPA 입문"         | 2          | "감사합니다"
1       | "JPA 입문"         | 3          | "도움됐어요"
2       | "Spring Boot"     | 4          | "잘 봤습니다"
2       | "Spring Boot"     | 5          | "따라해볼게요"
```

Post 2개 + Comment 5개 = **5개의 Row**가 반환된다.

#### 메모리에서 일어나는 일

```java
List<Post> posts = postRepository.findAllWithComments();
// posts.size() = 2 (Hibernate가 중복 제거)

// 하지만 DB에서는 5개 Row를 읽었다!
```

**문제점**:

1. **데이터 폭증**: Post 1개당 Comment가 100개면 100배 증가
2. **네트워크 비용**: 같은 Post 데이터가 중복 전송 (post_id, post_title이 100번 반복)
3. **메모리 낭비**: Hibernate가 중복 제거를 메모리에서 수행
4. **페이징 붕괴**: LIMIT/OFFSET이 메모리에서 실행됨

#### 페이징이 깨지는 실제 예시

```java
@Query("SELECT p FROM Post p JOIN FETCH p.comments")
Page<Post> findAllWithComments(Pageable pageable);

// 실행
Page<Post> posts = postRepository.findAllWithComments(PageRequest.of(0, 10));
```

**Hibernate 경고**:
```
WARN HHH000104: firstResult/maxResults specified with collection fetch;
applying in memory!
```

**실제 쿼리**:
```sql
SELECT p.*, c.*
FROM post p
LEFT JOIN comment c ON p.id = c.post_id
-- LIMIT 10 OFFSET 0 ← 이게 없다!
```

**무슨 일이 일어나는가**:
1. DB에서 **전체 데이터*를 메모리로 가져온다
2. Hibernate가 **메모리에서** 중복 제거를 수행한다
3. 메모리에서 **10개만 잘라낸다**

Post가 10,000개, Comment가 평균 50개면:
- DB에서 500,000개 Row 조회
- 메모리에 500,000개 Row 로딩
- OutOfMemoryError 위험

### 2.2 도메인 책임 분리

#### 단일 책임 원칙 위반

```java
// Post가 Comment까지 관리해야 하는가?
Post post = postRepository.findById(1L).get();

// Post의 책임인가? Comment의 책임인가?
List<Comment> comments = post.getComments();
```

**Post의 책임**:
- 게시글 제목, 내용 관리
- 게시글 조회수, 좋아요 수 관리
- 게시글 수정, 삭제

**Comment의 책임**:
- 댓글 내용 관리
- 댓글 목록 조회
- 댓글 정렬 (최신순, 좋아요순)
- 댓글 페이징

이 둘은 **독립적인 도메인**이다. Post가 Comment를 컬렉션으로 들고 있을 이유가 없다.

#### 실무 예시: 댓글 조회 요구사항

```
- 댓글을 최신순/좋아요순으로 정렬
- 댓글 페이징 (20개씩)
- 대댓글 지원
- 신고된 댓글 필터링
- 작성자가 삭제한 댓글은 "삭제된 댓글입니다" 표시
```

이걸 `post.getComments()`로 해결하는가? **불가능하다.**

별도의 Comment 조회 로직이 필요하다:

```java
public record CommentQueryDto(
    Long commentId,
    String content,
    Long memberId,
    String memberNickname,
    Instant createdAt,
    Long likeCount,
    boolean isDeleted
) {}

// CommentRepository
@Query("""
    SELECT new CommentQueryDto(
        c.id, c.content, m.id, m.nickname,
        c.createdAt, c.likeCount, c.isDeleted
    )
    FROM Comment c
    JOIN c.member m
    WHERE c.post.id = :postId
        AND c.isReported = false
    ORDER BY c.createdAt DESC
    """)
Page<CommentQueryDto> findByPostId(
    @Param("postId") Long postId,
    Pageable pageable
);
```

Post와 Comment는 **별도로 조회하는 것이 자연스럽다.**

### 2.3 Comment 페이징과 정렬 기능 확장성

#### @OneToMany로는 페이징이 불가능하다

```java
@Entity
public class Post {
    @OneToMany(mappedBy = "post")
    private List<Comment> comments;  // 전체를 메모리에 로드
}

// 사용
Post post = postRepository.findById(1L).get();
List<Comment> comments = post.getComments();
// 댓글이 1,000개면 1,000개를 전부 메모리에!
```

**문제**:
- 댓글이 많은 게시글은 로딩이 느리다
- 메모리 사용량이 폭증한다
- 페이징이 불가능하다 (Java Stream으로 잘라내야 함)

#### 별도 조회로 자유로운 페이징

```java
// 첫 페이지: 댓글 20개만
Page<CommentQueryDto> firstPage = commentRepository.findByPostId(
    postId,
    PageRequest.of(0, 20, Sort.by("createdAt").descending())
);

// 두 번째 페이지
Page<CommentQueryDto> secondPage = commentRepository.findByPostId(
    postId,
    PageRequest.of(1, 20, Sort.by("createdAt").descending())
);
```

**장점**:
- 필요한 만큼만 조회 (20개씩)
- DB에서 LIMIT/OFFSET으로 페이징
- 정렬 기준을 자유롭게 변경 가능
- 메모리 효율적

#### 정렬 기준 확장

```java
// 최신순
Sort.by("createdAt").descending()

// 좋아요순
Sort.by("likeCount").descending()

// 작성자 닉네임순
Sort.by("member.nickname").ascending()

// 복합 정렬: 좋아요순 -> 최신순
Sort.by("likeCount").descending()
    .and(Sort.by("createdAt").descending())
```

`post.getComments()`로는 이런 유연성을 확보할 수 없다.

---

## 3. N+1 문제 원천 차단

### 양방향 관계가 있을 때의 위험성

```java
@Entity
public class Post {
    @OneToMany(mappedBy = "post", fetch = FetchType.LAZY)
    private List<Comment> comments;
}

// Service Layer
@Transactional(readOnly = true)
public List<PostDto> getAllPosts() {
    List<Post> posts = postRepository.findAll();

    return posts.stream()
        .map(post -> new PostDto(
            post.getId(),
            post.getTitle(),
            post.getComments().size()  // ⚠️ 여기서 N+1 발생!
        ))
        .toList();
}
```

**실행되는 쿼리**:
```sql
-- 1. Post 조회
SELECT * FROM post;

-- 2. 각 Post마다 Comment 조회 (N번)
SELECT * FROM comment WHERE post_id = 1;
SELECT * FROM comment WHERE post_id = 2;
SELECT * FROM comment WHERE post_id = 3;
...
```

Post 100개면 **101개의 쿼리**가 실행된다.

### 원천 차단: @OneToMany 자체를 제거

```java
@Entity
public class Post {
    // ✅ comments 필드 자체가 없다
    // @OneToMany(mappedBy = "post")
    // private List<Comment> comments;
}
```

**효과**:
- `post.getComments()` 자체가 **컴파일 에러**
- 개발자가 실수로 N+1을 유발할 가능성이 **0%**
- 명시적으로 Comment를 별도 조회해야 함을 강제

### 올바른 조회 방법

#### 1) Comment Count만 필요한 경우

```java
@Query("""
    SELECT new PostWithCountDto(
        p.id, p.title, p.createdAt,
        COUNT(c.id)
    )
    FROM Post p
    LEFT JOIN Comment c ON c.post.id = p.id AND c.isDeleted = false
    GROUP BY p.id, p.title, p.createdAt
    """)
Page<PostWithCountDto> findAllWithCommentCount(Pageable pageable);
```

**쿼리 1개**로 모든 Post + Comment 개수 조회.

#### 2) Comment 상세 정보가 필요한 경우

```java
// 1. Post 목록 조회
Page<PostQueryDto> posts = postRepository.findAll(pageable);

// 2. 필요한 경우에만 Comment 조회
Long postId = posts.getContent().get(0).postId();
Page<CommentQueryDto> comments = commentRepository.findByPostId(postId, commentPageable);
```

**분리된 쿼리**, 하지만 **명시적이고 제어 가능**.

---

## 4. 실전 비교: 양방향 vs 단방향

### 시나리오: 게시글 상세 + 댓글 목록 조회

#### ❌ 양방향 관계 + Fetch Join (문제 많음)

```java
@Entity
public class Post {
    @OneToMany(mappedBy = "post", fetch = FetchType.LAZY)
    private List<Comment> comments;
}

// Repository
@Query("SELECT p FROM Post p JOIN FETCH p.comments WHERE p.id = :id")
Optional<Post> findByIdWithComments(@Param("id") Long id);

// Service
Post post = postRepository.findByIdWithComments(postId).get();
List<Comment> comments = post.getComments();
// ⚠️ 전체 댓글을 메모리에 로드
// ⚠️ 페이징 불가
// ⚠️ 정렬 기준 변경 불가
```

#### ✅ 단방향 관계 + 별도 조회 (권장)

```java
@Entity
public class Post {
    // comments 필드 없음
}

@Entity
public class Comment {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "post_id")
    private Post post;
}

// Repository
// Post
@Query("""
    SELECT new PostDetailDto(
        p.id, p.title, p.content, p.createdAt,
        m.id, m.nickname
    )
    FROM Post p
    JOIN p.member m
    WHERE p.id = :id AND p.isDeleted = false
    """)
Optional<PostDetailDto> findDetailById(@Param("id") Long id);

// Comment
@Query("""
    SELECT new CommentDto(
        c.id, c.content, c.createdAt,
        m.id, m.nickname
    )
    FROM Comment c
    JOIN c.member m
    WHERE c.post.id = :postId AND c.isDeleted = false
    ORDER BY c.createdAt DESC
    """)
Page<CommentDto> findByPostId(
    @Param("postId") Long postId,
    Pageable pageable
);

// Service
@Transactional(readOnly = true)
public PostDetailResponse getPostDetail(Long postId, Pageable commentPageable) {
    // 1. Post 조회 (쿼리 1개)
    PostDetailDto post = postRepository.findDetailById(postId)
        .orElseThrow(() -> new NotFoundException("Post not found"));

    // 2. Comment 조회 (쿼리 1개)
    Page<CommentDto> comments = commentRepository.findByPostId(
        postId,
        commentPageable
    );

    return new PostDetailResponse(post, comments);
}
```

**장점**:
- 쿼리 2개로 명확하게 분리
- Comment 페이징 가능 (20개씩)
- Post와 Comment를 각각 최적화 가능
- 필요한 필드만 SELECT (DTO 프로젝션)
- 메모리 효율적

---

## 5. 양방향 관계가 필요한 경우는?

모든 One-to-Many가 나쁜 것은 아니다. **진짜 필요한 경우**도 있다.

### 5.1 부모-자식이 강하게 결합된 경우

#### 예시: Order - OrderItem

```java
@Entity
public class Order {
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> orderItems = new ArrayList<>();

    // 주문 생성 시 항목도 함께 생성
    public void addItem(OrderItem item) {
        orderItems.add(item);
        item.setOrder(this);
    }

    // 주문 삭제 시 항목도 함께 삭제
    public void clearItems() {
        orderItems.clear();
    }
}
```

**이유**:
- Order와 OrderItem은 **생명주기를 같이한다**
- Order 없이 OrderItem은 존재할 수 없다
- 주문 생성 시 항목이 함께 저장되어야 한다
- `cascade`, `orphanRemoval`로 일관성 보장

### 5.2 Aggregate Root 패턴

DDD의 Aggregate Root 개념에서:
- Order는 Aggregate Root
- OrderItem은 Aggregate 내부 엔티티
- 외부에서 OrderItem을 직접 조회/수정하지 않는다
- 모든 작업은 Order를 통해서만 이루어진다

이 경우 양방향 관계가 **도메인 무결성**을 지키는 데 도움이 된다.

### 5.3 양방향이 적합한지 판단하는 질문

1. **생명주기가 같은가?**
   - Order-OrderItem: ✅ 같다
   - Post-Comment: ❌ 독립적

2. **항상 함께 조회하는가?**
   - Order-OrderItem: ✅ 주문 조회 시 항목도 필요
   - Post-Comment: ❌ Post만 볼 때가 많다

3. **부모를 통해서만 자식을 관리하는가?**
   - Order-OrderItem: ✅ Order를 통해서만 추가/삭제
   - Post-Comment: ❌ Comment는 독립적으로 CRUD

4. **자식의 개수가 적은가? (< 50개)**
   - Order-OrderItem: ✅ 보통 1~20개
   - Post-Comment: ❌ 수백, 수천 개 가능

**모두 ✅면 양방향 고려, 하나라도 ❌면 단방향**

---

## 6. 대안 전략: Comment Count 최적화

### 문제: Post 목록에 댓글 개수 표시

```
[게시글 목록]
- "JPA 입문" | 작성자: Devon | 댓글 15개
- "Spring Boot" | 작성자: Devon | 댓글 42개
```

양방향 관계 없이 어떻게 댓글 개수를 조회하는가?

### 전략 1: 조인 + 집계

```java
public record PostListDto(
    Long postId,
    String title,
    Instant createdAt,
    String memberNickname,
    Long commentCount  // 댓글 개수
) {}

@Query("""
    SELECT new PostListDto(
        p.id, p.title, p.createdAt,
        m.nickname,
        COUNT(c.id)
    )
    FROM Post p
    JOIN p.member m
    LEFT JOIN Comment c ON c.post.id = p.id AND c.isDeleted = false
    WHERE p.isDeleted = false
    GROUP BY p.id, p.title, p.createdAt, m.nickname
    ORDER BY p.createdAt DESC
    """)
Page<PostListDto> findAllWithCommentCount(Pageable pageable);
```

**쿼리 1개**로 Post + Comment 개수 조회.

### 전략 2: 역정규화 - commentCount 컬럼

```java
@Entity
public class Post {
    private Long commentCount = 0L;  // 캐시 컬럼

    public void incrementCommentCount() {
        this.commentCount++;
    }

    public void decrementCommentCount() {
        if (this.commentCount > 0) {
            this.commentCount--;
        }
    }
}

// Comment 생성 시
Comment comment = Comment.create(member, post, content);
commentRepository.save(comment);
postRepository.incrementCommentCount(comment.getPost().getId());

// Comment 삭제 시
commentRepository.delete(comment);
postRepository.decrementCommentCount(postId);
```

**장점**:
- 조회 시 조인 불필요
- 빠른 조회 성능

**단점**:
- 동기화 로직 필요
- 정합성 보장 필요 (동시성 제어)

### 전략 3: Redis 캐싱

```java
@Service
public class PostService {
    private final RedisTemplate<String, Long> redisTemplate;

    public PostListDto getPostWithCommentCount(Long postId) {
        // 1. Redis에서 캐시 조회
        String key = "post:comment_count:" + postId;
        Long count = redisTemplate.opsForValue().get(key);

        // 2. 캐시 미스 시 DB 조회 후 캐싱
        if (count == null) {
            count = commentRepository.countByPostId(postId);
            redisTemplate.opsForValue().set(key, count, 1, TimeUnit.HOURS);
        }

        return new PostListDto(..., count);
    }

    // Comment 생성 시 캐시 무효화
    public void createComment(CommentCreateRequest request) {
        Comment comment = commentRepository.save(...);

        String key = "post:comment_count:" + comment.getPost().getId();
        redisTemplate.delete(key);  // 캐시 삭제
    }
}
```

---

## 7. 실무 체크리스트

### One-to-Many를 선언하기 전에

- [ ] 부모와 자식의 생명주기가 같은가?
- [ ] 항상 함께 조회하는가?
- [ ] 자식의 개수가 충분히 적은가? (< 50개)
- [ ] 자식을 별도로 페이징할 필요가 없는가?
- [ ] 부모를 통해서만 자식을 관리하는가?

**5개 모두 Yes**: 양방향 관계 고려
**1개라도 No**: 단방향 관계 권장

### 단방향으로 설계할 때

- [ ] Comment → Post 참조만 유지 (ManyToOne)
- [ ] Comment 조회는 별도 Repository 메서드
- [ ] Comment Count는 조인 집계 또는 역정규화
- [ ] 페이징과 정렬은 Comment Repository에서 담당
- [ ] Post와 Comment의 책임을 명확히 분리

---

## 8. 핵심 원칙 정리

### 양방향 관계는 마지막 수단
필요하지 않으면 **선언하지 않는다**. 편의 메서드가 아니라 책임의 문제다.

### Post와 Comment는 독립적 도메인
각각 별도로 조회하는 것이 자연스럽다. 억지로 한 번에 가져오려 하지 말자.

### N+1은 원천 차단이 최고
`post.getComments()`가 없으면 N+1이 발생할 수 없다. 컴파일 에러가 런타임 에러보다 낫다.

### 페이징은 DB에서
`List<Comment>` 전체를 메모리에 올리지 말자. DB의 LIMIT/OFFSET을 활용하자.

### DTO 프로젝션으로 필요한 것만
Comment 전체가 필요한가? 아니면 일부 필드만? 명확히 하고 DTO로 받자.

---

## 마치며

One-to-Many 양방향 관계는 **편리하지만 위험하다**.

- Fetch Join 시 메모리 폭증
- 페이징이 메모리에서 실행됨
- N+1 문제 유발 가능성
- 도메인 책임이 불명확해짐

반면 **단방향 + 별도 조회**는:

- 각 도메인의 책임이 명확
- 페이징과 정렬을 자유롭게 제어
- 쿼리가 예측 가능
- N+1 문제 원천 차단

양방향이 필요한 경우도 분명 있다. 하지만 **기본은 단방향**이어야 한다. "나중에 필요할지도"라는 이유로 양방향을 선언하지 말자.

**원칙**: 필요할 때 추가하기는 쉽지만, 잘못 추가한 것을 제거하기는 어렵다.
