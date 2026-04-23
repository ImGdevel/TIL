# Notion curated 승격 맵 v2

이 문서는 `Notion/curated` 아래 문서 63개가 현재 `TIL.wiki`와 `Portfolio`에서 어떻게 처리되었는지 한 번에 추적하기 위한 운영 문서다.

기준은 아래 세 가지다.

- `Study wiki 직접 승격 완료`: `TIL.wiki`에 독립 문서가 존재
- `Study wiki 흡수 완료`: 별도 문서는 없지만 기존 대표 문서에 개념이 흡수됨
- `Portfolio 승격 완료`: 커리어, 프로젝트, 지원 준비 문서로 `Portfolio` 허브에 직접 승격

현재 판단은 다음과 같다.

- `Notion/curated` 전체 기준 신규 정리 잔여: `0`
- 기술 학습 문서는 `TIL.wiki`, 커리어/프로젝트 문서는 `Portfolio`로 분리 완료

## 1. Study wiki 직접 승격 완료

| Notion curated 문서 | 현재 wiki 문서 |
| --- | --- |
| `aspnet-core-vs-spring-selection-criteria.md` | `aspnet-core-vs-spring-selection-criteria.md` |
| `cqrs.md` | `cqrs.md` |
| `database-indexes.md` | `database-indexes.md` |
| `facade-service.md` | `facade-service.md` |
| `fetch-join-vs-entitygraph-vs-batch-size.md` | `fetch-join-vs-entitygraph-vs-batch-size.md` |
| `gradle-dependency-configurations.md` | `gradle-dependency-configurations.md` |
| `kafka-key-based-partitioning-and-ordering.md` | `kafka-key-based-partitioning-and-ordering.md` |
| `kafka-log-compaction.md` | `kafka-log-compaction.md` |
| `kafka-offset-commit-retry-dlt.md` | `kafka-offset-commit-retry-dlt.md` |
| `mysql-vs-postgresql.md` | `mysql-vs-postgresql.md` |
| `projection-vs-in-memory-join.md` | `projection-vs-in-memory-join.md` |
| `querydsl.md` | `querydsl.md` |
| `redisson.md` | `redisson.md` |
| `spring-security-authorization.md` | `spring-security-authorization.md` |
| `spring-security-filter-vs-handlerinterceptor.md` | `spring-security-filter-vs-handlerinterceptor.md` |
| `spring-security-hasrole-vs-hasauthority.md` | `spring-security-hasrole-vs-hasauthority.md` |
| `spring-security-servlet-architecture.md` | `spring-security-servlet-architecture.md` |
| `waf-firewall-and-proxy.md` | `waf-firewall-and-proxy.md` |
| `web-request-flow-and-https.md` | `web-request-flow-and-https.md` |

## 2. Study wiki 흡수 완료

- `asynchronous-processing.md` -> `async-strategy-and-thread-pool-management.md`
- `backend-learning-roadmap-and-testing-principles.md` -> `testing-strategy-and-test-pyramid.md`
- `caching-strategies.md` -> `redis-overview-and-caching.md`, `local-cache-with-caffeine.md`
- `community-domain-refactoring-case-study.md` -> `facade-service.md`, `cqrs.md`, `projection-vs-in-memory-join.md`, `fetch-join-vs-entitygraph-vs-batch-size.md`
- `database-performance-interview-qa.md` -> `database-indexes.md`, `database-connection-and-transaction-management.md`, `mysql-vs-postgresql.md`
- `devops-docker-cicd-and-deployment-basics.md` -> `docker-basics-images-containers-volumes.md`, `gradle-dependency-configurations.md`
- `jpa-caching.md` -> `jpa-query-and-cache-strategies.md`
- `junit.md` -> `testing-strategy-and-test-pyramid.md`
- `jwt.md` -> `spring-security-authentication-architecture.md`, `spring-security-jwt-filter-positioning.md`
- `kafka-consumer-basics.md` -> `messaging-patterns-pubsub-stream-kafka.md`, `kafka-offset-commit-retry-dlt.md`
- `kafka-overview.md` -> `messaging-patterns-pubsub-stream-kafka.md`
- `kafka-producer-basics.md` -> `messaging-patterns-pubsub-stream-kafka.md`, `kafka-key-based-partitioning-and-ordering.md`
- `kafka-topics-partitions-replication-offsets.md` -> `messaging-patterns-pubsub-stream-kafka.md`, `kafka-key-based-partitioning-and-ordering.md`
- `kafka-transition-criteria.md` -> `messaging-patterns-pubsub-stream-kafka.md`
- `mockito.md` -> `testing-strategy-and-test-pyramid.md`
- `mysql-to-postgresql-migration.md` -> `mysql-vs-postgresql.md`
- `persistence-context-internals.md` -> `jpa-query-and-cache-strategies.md`
- `redis-operational-pitfalls.md` -> `redis-overview-and-caching.md`
- `redis-pubsub-vs-streams.md` -> `messaging-patterns-pubsub-stream-kafka.md`
- `redis-streams-vs-kafka-and-mq.md` -> `messaging-patterns-pubsub-stream-kafka.md`
- `redis-streams.md` -> `messaging-patterns-pubsub-stream-kafka.md`
- `redis.md` -> `redis-overview-and-caching.md`
- `spring-security-authentication-flow.md` -> `spring-security-authentication-architecture.md`
- `sync-vs-async-boundary.md` -> `async-strategy-and-thread-pool-management.md`

## 3. Portfolio 승격 완료

| Notion curated 문서 | Portfolio 문서 |
| --- | --- |
| `assignment-screening-keyword-guide.md` | `assignment-screening-keyword-guide.md` |
| `backend-portfolio-writing-guide.md` | `backend-portfolio-writing-guide.md` |
| `coding-test-language-selection-guide.md` | `coding-test-language-selection-guide.md` |
| `coding-test-preparation-strategy.md` | `coding-test-preparation-strategy.md` |
| `cover-letter-core-competency-map.md` | `cover-letter-core-competency-map.md` |
| `cover-letter-keyword-to-proof-map.md` | `cover-letter-keyword-to-proof-map.md` |
| `developer-job-market-and-study-strategy.md` | `developer-job-market-and-study-strategy.md` |
| `gpt.md` | `gpt.md` |
| `insty-project-overview.md` | `insty-project-overview.md` |
| `interview-claim-priority-and-correction-guide.md` | `interview-claim-priority-and-correction-guide.md` |
| `interview-feedback-charlie-guide.md` | `interview-feedback-charlie-guide.md` |
| `maplestory-worlds-contribution-fit.md` | `maplestory-worlds-contribution-fit.md` |
| `nexon-application-project-summary.md` | `nexon-application-project-summary.md` |
| `nexon-backend-interview-preparation-guide.md` | `nexon-backend-interview-preparation-guide.md` |
| `nexon-interview-blank-note-method.md` | `nexon-interview-blank-note-method.md` |
| `nexon-role-and-service-analysis.md` | `nexon-role-and-service-analysis.md` |
| `personal-project-backend-highlights.md` | `personal-project-backend-highlights.md` |
| `resume-feedback-and-positioning-checklist.md` | `resume-feedback-and-positioning-checklist.md` |
| `resume-portfolio-proof-checklist.md` | `resume-portfolio-proof-checklist.md` |
| `submission-introduction-and-positioning.md` | `submission-introduction-and-positioning.md` |

## 4. 현재 판단

- 기술 학습 위키 기준 주요 기술 문서는 직접 승격 또는 기존 문서 흡수로 정리 완료했다.
- 남아 있던 커리어, 포트폴리오, 프로젝트 문서는 `Portfolio` 허브로 직접 승격 완료했다.
- 따라서 `Notion/curated` 전체 기준으로 현재 남은 미정리 문서는 `0`으로 본다.
