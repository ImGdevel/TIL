# Study Wiki

이 위키는 `TIL`에 러프하게 쌓인 학습 기록을 다시 정리해 둔 문서 공간입니다.

`TIL`이 공부한 내용을 빠르게 적재하는 곳이라면, 이곳은 나중에 다시 읽고 다른 사람도 이해할 수 있게 다듬은 지식 문서입니다.

<br />

---

## 빠른 진입점

| 보고 싶은 내용 | 먼저 볼 문서 |
| --- | --- |
| Spring Security의 전체 요청 흐름 | [Servlet 아키텍처](https://github.com/ImGdevel/TIL/wiki/spring-security-servlet-architecture) |
| 인증과 인가의 차이 | [인증 아키텍처](https://github.com/ImGdevel/TIL/wiki/spring-security-authentication-architecture), [인가 구조](https://github.com/ImGdevel/TIL/wiki/spring-security-authorization) |
| Redis, Stream, Kafka 비교 | [메시징 패턴: MQ, Pub/Sub, Stream, Kafka](https://github.com/ImGdevel/TIL/wiki/messaging-patterns-pubsub-stream-kafka) |
| JPA 조회 최적화 | [Fetch Join, EntityGraph, Batch Size](https://github.com/ImGdevel/TIL/wiki/fetch-join-vs-entitygraph-vs-batch-size), [Projection과 In-memory Join](https://github.com/ImGdevel/TIL/wiki/projection-vs-in-memory-join) |
| 비동기와 실시간 통신 | [비동기 전략과 쓰레드 풀 관리](https://github.com/ImGdevel/TIL/wiki/async-strategy-and-thread-pool-management), [WebSocket과 SSE](https://github.com/ImGdevel/TIL/wiki/websocket-and-sse) |
| 설계 판단과 트레이드오프 | [CQRS](https://github.com/ImGdevel/TIL/wiki/cqrs), [Facade Service](https://github.com/ImGdevel/TIL/wiki/facade-service), [이벤트 드리븐 아키텍처](https://github.com/ImGdevel/TIL/wiki/event-driven-architecture) |

<br />

---

## 문서 지도

## Spring Security

Spring Security의 Servlet 기반 구조, 인증/인가 흐름, 필터 체인, CORS/CSRF, 커스텀 필터 배치를 정리합니다.

- [Servlet 아키텍처](https://github.com/ImGdevel/TIL/wiki/spring-security-servlet-architecture)
- [인증 아키텍처](https://github.com/ImGdevel/TIL/wiki/spring-security-authentication-architecture)
- [인가 구조](https://github.com/ImGdevel/TIL/wiki/spring-security-authorization)
- [Filter Chain 예외 처리](https://github.com/ImGdevel/TIL/wiki/spring-security-filter-chain-exception-handling)
- [CORS](https://github.com/ImGdevel/TIL/wiki/spring-security-cors)
- [CORS 원인 추적](https://github.com/ImGdevel/TIL/wiki/spring-security-cors-troubleshooting)
- [CSRF](https://github.com/ImGdevel/TIL/wiki/spring-security-csrf)
- [커스텀 필터 설계](https://github.com/ImGdevel/TIL/wiki/spring-security-custom-filter-design)
- [필터 베이스 클래스 선택](https://github.com/ImGdevel/TIL/wiki/spring-filter-base-classes)
- [Filter와 HandlerInterceptor](https://github.com/ImGdevel/TIL/wiki/spring-security-filter-vs-handlerinterceptor)
- [JWT 필터 배치](https://github.com/ImGdevel/TIL/wiki/spring-security-jwt-filter-positioning)
- [hasRole vs hasAuthority](https://github.com/ImGdevel/TIL/wiki/spring-security-hasrole-vs-hasauthority)

<br />

---

## Redis / Messaging

Redis의 캐싱과 분산 도구, Pub/Sub과 Stream, Kafka의 파티션/오프셋/재처리 전략을 함께 정리합니다.

- [Redis 개요와 캐싱](https://github.com/ImGdevel/TIL/wiki/redis-overview-and-caching)
- [Redisson](https://github.com/ImGdevel/TIL/wiki/redisson)
- [메시징 패턴: MQ, Pub/Sub, Stream, Kafka](https://github.com/ImGdevel/TIL/wiki/messaging-patterns-pubsub-stream-kafka)
- [Kafka offset, retry, DLT](https://github.com/ImGdevel/TIL/wiki/kafka-offset-commit-retry-dlt)
- [Kafka key, partition, ordering](https://github.com/ImGdevel/TIL/wiki/kafka-key-based-partitioning-and-ordering)
- [Kafka log compaction](https://github.com/ImGdevel/TIL/wiki/kafka-log-compaction)
- [로컬 캐시와 Caffeine](https://github.com/ImGdevel/TIL/wiki/local-cache-with-caffeine)

<br />

---

## JPA / Persistence / Data

JPA 매핑, 조회 전략, QueryDSL, DB 인덱스와 실행 계획, MySQL/PostgreSQL 차이를 정리합니다.

- [엔티티와 매핑 설계](https://github.com/ImGdevel/TIL/wiki/jpa-entity-and-mapping-design)
- [조회와 캐시 전략](https://github.com/ImGdevel/TIL/wiki/jpa-query-and-cache-strategies)
- [QueryDSL](https://github.com/ImGdevel/TIL/wiki/querydsl)
- [Projection과 In-memory Join](https://github.com/ImGdevel/TIL/wiki/projection-vs-in-memory-join)
- [Fetch Join, EntityGraph, Batch Size](https://github.com/ImGdevel/TIL/wiki/fetch-join-vs-entitygraph-vs-batch-size)
- [인덱스 설계와 실행 계획](https://github.com/ImGdevel/TIL/wiki/database-indexes)
- [MySQL과 PostgreSQL](https://github.com/ImGdevel/TIL/wiki/mysql-vs-postgresql)
- [DB 커넥션과 트랜잭션 관리](https://github.com/ImGdevel/TIL/wiki/database-connection-and-transaction-management)

<br />

---

## Web / Async

JavaScript 비동기 모델과 서버 관점의 비동기 처리, WebSocket/SSE 같은 실시간 통신 전략을 정리합니다.

- [함수 선언과 this](https://github.com/ImGdevel/TIL/wiki/javascript-function-declarations-and-this)
- [비동기 기초](https://github.com/ImGdevel/TIL/wiki/javascript-async-fundamentals)
- [Promise](https://github.com/ImGdevel/TIL/wiki/javascript-promise)
- [이벤트 루프](https://github.com/ImGdevel/TIL/wiki/javascript-event-loop)
- [비동기 전략과 쓰레드 풀 관리](https://github.com/ImGdevel/TIL/wiki/async-strategy-and-thread-pool-management)
- [WebSocket과 SSE](https://github.com/ImGdevel/TIL/wiki/websocket-and-sse)

<br />

---

## HTTP / Network

브라우저 요청 흐름, HTTPS, WAF, 방화벽, 프록시처럼 애플리케이션 바깥의 네트워크 경계를 정리합니다.

- [웹 요청 흐름과 HTTPS](https://github.com/ImGdevel/TIL/wiki/web-request-flow-and-https)
- [WAF, 방화벽, 프록시](https://github.com/ImGdevel/TIL/wiki/waf-firewall-and-proxy)

<br />

---

## Architecture / Design

애플리케이션 구조와 트레이드오프를 다룹니다. 특정 기술보다 설계 판단의 기준을 정리하는 영역입니다.

- [AI Chat에서 MVC와 WebFlux 선택](https://github.com/ImGdevel/TIL/wiki/ai-chat-mvc-vs-webflux)
- [이벤트 드리븐 아키텍처](https://github.com/ImGdevel/TIL/wiki/event-driven-architecture)
- [CQRS](https://github.com/ImGdevel/TIL/wiki/cqrs)
- [Facade Service](https://github.com/ImGdevel/TIL/wiki/facade-service)
- [ASP.NET Core vs Spring 선택 기준](https://github.com/ImGdevel/TIL/wiki/aspnet-core-vs-spring-selection-criteria)
- [응집도와 결합도](https://github.com/ImGdevel/TIL/wiki/software-design-cohesion-and-coupling)

<br />

---

## Infra / Build

테스트 전략, Docker, Gradle처럼 개발 결과물이 검증되고 실행 환경으로 전달되는 과정을 정리합니다.

- [테스트 전략과 테스트 피라미드](https://github.com/ImGdevel/TIL/wiki/testing-strategy-and-test-pyramid)
- [Docker 기초](https://github.com/ImGdevel/TIL/wiki/docker-basics-images-containers-volumes)
- [Gradle 의존성 설정](https://github.com/ImGdevel/TIL/wiki/gradle-dependency-configurations)

<br />

---

## Java / Spring Core

Java와 Spring 기반 개발에서 반복적으로 등장하는 핵심 도구와 개념을 정리합니다.

- [Jackson과 JSON 처리](https://github.com/ImGdevel/TIL/wiki/jackson-json-processing)
- [Spring AOP 핵심 개념](https://github.com/ImGdevel/TIL/wiki/spring-aop-pointcut-advice-advisor)

<br />

---

## 기준

- `TIL` 원문은 남긴다.
- `wiki`는 대표 문서만 올린다.
- 실험성 문서는 정리 후 대표 문서에 병합하거나 제거한다.
- 중복 문서는 삭제보다 병합을 우선한다.
- 공식 문서나 실제 자료를 참고한 경우 문서 마지막에 `참고자료`를 둔다.
