# Insty Backend Web - 포트폴리오

### 본 저장소는 공동 작업 프로젝트에서 **개인 기여분만 추출**한 포트폴리오용 코드베이스입니다.

> < 전체 프로젝트 Ropo ><br>
📎 **전체 프로젝트**: [insty-backend-web (원본 Repository)](https://github.com/Insty-team/insty-backend-web)

<details>
  <summary>해당 프로젝트는 포트폴리오 활용에 대한 공개 허가를 받았습니다 </summary>
    <img width="896" height="247" alt="스크린샷 2025-09-14 102219" src="https://github.com/user-attachments/assets/9c1acb48-bff6-46b1-a76d-5496b4c436ca" />
</details>

#### 프로젝트 기여 지표 |  [📎URL](https://github.com/Insty-team/insty-backend-web/graphs/contributors)



<br>

## 주요 담당 영역

### 도메인/기능 담당 영역 
- **커뮤니티 Q&A 시스템** - 질문/답변 CRUD 및 답변 채택 기능
- **멘션 시스템** - 사용자 태깅 및 스팸 방지 기능
- **알림 시스템** - 이벤트 기반 이메일 알림

### 성능 최적화 기여 영역
- **조회 성능 최적화** - N+1 문제 해결 및 조회 최적화
- **비동기 메일 알림** - 이벤트 기반 비동기 메일 알림

<br>

---


## 🛠 기술 스택

- **Backend**: Spring Boot 3.4.5, Java 17, Spring Security (JWT), QueryDSL 5.1.0
- **Database**: PostgreSQL, Redis, Spring Data JPA
- **Infrastructure**: AWS S3, CloudFront, Gmail SMTP
- **Testing**: JUnit 5, Mockito, Custom Tags (`@unit`, `@integration`)

<br>



## 🏗 아키텍처

### 멀티모듈 아키택처
```
insty-backend-web/
├── insty-api/          # REST API, 비즈니스 로직, 보안 설정
├── insty-domain/       # JPA 엔티티, 리포지토리, QueryDSL
├── insty-common/       # 공통 유틸리티, JWT, 예외 처리
└── insty-external/     # 외부 연동 (AWS S3, 이메일)
```

### Facade Service 패턴 + Read/Write 관심사 분리
- **Service Layer**: 비즈니스 로직 조합 및 트랜잭션 관리 (`CommunityQuestionService`)
- **Implement Layer**: 책임별 컴포넌트 분리 구조 (예시)
  - `CommunityQuestionReader` - 조회 전용 로직
  - `CommunityQuestionWriter` - 생성/수정 전용 로직
  - `CommunityQuestionFileReader/Writer` - 파일 처리 분리
  - `CommunityNotificationManager` - 알림 로직 분리

### 이벤트 기반 아키텍처
- **@TransactionalEventListener** + **@Async** 패턴
- **AFTER_COMMIT** 시점 이벤트 처리로 트랜잭션 분리
- **@Async** 어노테이션 기반 비동기 이메일 발송

<br>

---


## ⚡ 주요 구현 기능

### 1. 커뮤니티 Q&A 시스템 - 성능 최적화 중심 구현

#### 전체 시스템 기여 범위
- **REST API 전체 설계** - 13개 엔드포인트 (`CommunityController`)
- **동적 검색 시스템** - QueryDSL 기반 다중 필터 조건 쿼리
- **파일 업로드 처리** - Multipart 지원 (질문 2개, 답변 1개 제한)
- **권한 관리 시스템** - Spring Security + 역할 기반 접근 제어

#### 핵심 성능 최적화 구현

**1. QueryDSL DTO Projection으로 N+1 문제 해결**
> 📋 **구현 코드**: [`CommunityQuestionQueryRepositoryImpl.java:30-56`](insty-domain/src/main/java/insty/domain/community/persistence/CommunityQuestionQueryRepositoryImpl.java)

**🚨 문제 상황**
- 엔티티 조회 후 DTO 변환 시 연관된 User 엔티티로 인한 N+1 쿼리 발생
- 불필요한 전체 엔티티 로딩으로 메모리 오버헤드 증가

**💡 해결 방법**
- **FetchJoin 대신 DTO Projection**: 필요한 컬럼만 선택적 조회
- **Projections.constructor()**: 엔티티 거치지 않고 DTO 직접 매핑

```java
public List<CommunityQuestionSearchInfo> searchQuestions(PaginationReq paginationReq,
                                                        CommunityQuestionSearchFilter filter, String sort) {
    return select(
        Projections.constructor(
            CommunityQuestionSearchInfo.class,
            communityQuestion.id,
            Projections.constructor(
                UserInfo.class,
                user.id,
                user.nickname,
                user.userType
            ),
            communityQuestion.course.id,
            communityQuestion.title,
            communityQuestion.content,
            communityQuestion.status,
            communityQuestion.createdAt,
            communityQuestion.updatedAt
        )
    )
    .from(communityQuestion)
    .join(communityQuestion.user, user)
    .where(searchConditions(filter))
    .orderBy(createOrderSpecifier(sort))
    .offset(paginationReq.getOffset())
    .limit(paginationReq.pageSize())
    .fetch();
}
```

**📈 성과**: 엔티티 전체 로딩 → 필요 컬럼만 조회로 메모리 사용량 감소

<br>

**2. 답변 개수 조회 N+1 문제 해결**
> 📋 **구현 코드**: [`CommunityQuestionQueryRepositoryImpl.java:69-95`](insty-domain/src/main/java/insty/domain/community/persistence/CommunityQuestionQueryRepositoryImpl.java)

**🚨 문제 상황**
- 질문 목록 조회 시 각 질문의 답변 개수를 개별 쿼리로 조회
- 10개 질문 조회 시 1 + 10 = 11번 쿼리 실행

**💡 해결 방법**
- **배치 쿼리**: 모든 질문의 답변 개수를 한 번에 조회
- **GROUP BY + IN 조건**: 단일 쿼리로 다수 질문 처리

```java
public Map<Long, Long> getAnswerCountsByQuestionIds(List<Long> questionIds) {
    if (questionIds == null || questionIds.isEmpty()) {
        return Map.of();
    }

    NumberExpression<Long> countExpr = communityAnswer.id.count();
    List<Tuple> results = select(
        Projections.tuple(
            communityAnswer.communityQuestion.id,
            countExpr
        )
    )
    .from(communityAnswer)
    .where(
        communityAnswer.communityQuestion.id.in(questionIds),
        communityAnswer.isDeleted.eq(false)
    )
    .groupBy(communityAnswer.communityQuestion.id)
    .fetch();

    return results.stream()
        .collect(Collectors.toMap(
            tuple -> tuple.get(communityAnswer.communityQuestion.id),
            tuple -> tuple.get(countExpr)
        ));
}
```

**📈 성과**: 질문 10개 기준 11번 쿼리 → 1번 쿼리로 90% 감소

<br>

**3. 동적 검색 조건 빌더와 답변 채택 로직**
> 📋 **구현 코드**: [`CommunityQuestionQueryRepositoryImpl.java:97-123`](insty-domain/src/main/java/insty/domain/community/persistence/CommunityQuestionQueryRepositoryImpl.java) | [`CommunityAnswerAcceptManager.java:27-50`](insty-api/src/main/java/insty/domain/community/implement/CommunityAnswerAcceptManager.java)
```java
// 동적 검색 - 다중 필터 조건 처리
private BooleanExpression[] searchConditions(CommunityQuestionSearchFilter filter) {
    return new BooleanExpression[] {
        courseIdEq(filter.courseId()),
        statusesIn(filter.statuses()),
        queryContains(filter.query()),
        userIdEq(filter.userId()),
        communityQuestion.isDeleted.eq(false)
    };
}

private BooleanExpression queryContains(String query) {
    if (query == null || query.isBlank()) return null;
    return communityQuestion.title.containsIgnoreCase(query)
            .or(communityQuestion.content.containsIgnoreCase(query));
}

// 토글 방식 답변 채택 - 3가지 상태 관리
public AcceptAnswerResultRes acceptAnswer(CommunityQuestion question, CommunityAnswer answer) {
    CommunityAnswer currentAccepted = question.getAcceptedAnswer();
    if (currentAccepted == null) {
        question.acceptAnswer(answer);
        communityQuestionRepository.save(question);
        return new AcceptAnswerResultRes(answer.getId(), true);
    }
    if (currentAccepted.getId().equals(answer.getId())) {
        question.unacceptAnswer();
        communityQuestionRepository.save(question);
        return new AcceptAnswerResultRes(answer.getId(), false);
    }
    throw new CustomException(CommunityErrorCode.COMMUNITY_ALREADY_ACCEPTED_ANSWER);
}
```

<br>

### 2. 멘션 시스템 - 스팸 방지 및 메모리 최적화
> 📋 **구현 코드**: [`MentionParser.java:18-46`](insty-api/src/main/java/insty/domain/mention/implement/MentionParser.java)

**🚨 문제 상황**
- 무분별한 사용자 멘션으로 알림 폭탄 가능성
- 동일 사용자 중복 멘션 시 중복 알림 발송
- 대량 멘션 파싱 시 메모리 비효율

**💡 해결 방법**
- **LinkedHashMap 활용**: 입력 순서 보존하면서 중복 제거
- **최대 개수 제한**: 댓글당 2명으로 스팸 방지
- **자기 멘션 차단**: 불필요한 자기 알림 방지

```java
// 정규식 패턴과 제한 설정
private static final Pattern MENTION_PATTERN = Pattern.compile("@\\[([^\\]]+)\\]\\(([^)]+)\\)");
private static final int MAX_MENTIONS_PER_COMMENT = 2;

// LinkedHashMap으로 중복 제거 (입력 순서 보존)
List<MentionedUserInfo> deduped = parsed.stream()
    .collect(java.util.stream.Collectors.collectingAndThen(
        java.util.stream.Collectors.toMap(
            MentionedUserInfo::userId,
            java.util.function.Function.identity(),
            (a, b) -> a,  // 중복 시 첫 번째 유지
            java.util.LinkedHashMap::new  // 순서 보존
        ),
        m -> new java.util.ArrayList<>(m.values())
    ));

// 스팸 방지 검증
if (deduped.size() > MAX_MENTIONS_PER_COMMENT) {
    throw new CustomException(MentionErrorCode.MENTION_LIMIT_EXCEEDED);
}
if (mentionerUser != null && mentionerUser.getId() != null &&
    deduped.stream().anyMatch(u -> u.userId().equals(mentionerUser.getId()))) {
    throw new CustomException(MentionErrorCode.MENTION_SELF_ERROR);
}
```

**📈 성과**: 알림 스팸 완전 차단, 메모리 효율적 중복 제거

<br>

### 3. 이벤트 기반 알림 시스템
> 📋 **구현 코드**: [`MentionNotificationManager.java:24-27`](insty-api/src/main/java/insty/domain/mention/implement/MentionNotificationManager.java) | [`UserMentionNotificationHandler.java:25-27`](insty-api/src/main/java/insty/domain/notification/handler/UserMentionNotificationHandler.java)

**🚨 문제 상황**
- 동기 이메일 발송으로 API 응답 시간 지연
- 이메일 발송 실패 시 전체 비즈니스 로직 롤백 위험
- 트랜잭션 범위 내 외부 서비스 호출로 성능 저하

**💡 해결 방법**
- **@TransactionalEventListener(AFTER_COMMIT)**: 트랜잭션 커밋 후 이벤트 처리
- **@Async**: 별도 스레드에서 비동기 이메일 발송
- **이벤트 기반 분리**: 비즈니스 로직과 알림 로직 완전 분리

```java
// 이벤트 발행 - 비즈니스 로직에서 분리
for (Mention mention : mentions) {
    eventPublisher.publishEvent(
        new UserMentionedEvent(mention.getMentionedUser(), mention.getMentionerUser(), communityQuestion));
}

// 이벤트 처리 - 트랜잭션 외부에서 비동기 처리
@Async
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onMailEventHandler(UserMentionedEvent event) {
    try {
        if (!notificationValidator.validateUserNotification(event.receiver())) {
            return;
        }
        // 별도 트랜잭션에서 이메일 발송
        UserMentionMailContent mailContent = UserMentionMailContent.of(event);
        mailHelper.sendMail(mailContent.getTo(), mailContent.getSubject(), mailContent.getText());
    } catch (Exception e) {
        log.error("Failed to send mention notification", e);
    }
}
```

**📈 성과**: API 응답시간 약 1초 → 300ms 단축, 이메일 발송 실패 시에도 비즈니스 로직 안전 보장

<br>

---

## 📊 핵심 성과

- **13개 REST API 설계** - 커뮤니티 Q&A 전체 시스템 구현 ([`CommunityController.java`](insty-api/src/main/java/insty/domain/community/controller/CommunityController.java))
- **N+1 문제 해결** - QueryDSL DTO Projection으로 엔티티 로딩 최적화
- **배치 쿼리 최적화** - 개별 쿼리 11번 → 1번으로 90% 감소
- **이벤트 기반 알림** - `@TransactionalEventListener` + `@Async`로 API 응답시간 1초 → 300ms
- **멘션 스팸 방지** - LinkedHashMap 중복제거 + 최대 2명 제한
- **CQRS + Facade Service** - 책임 분리 기반 컴포넌트 구조 설계


