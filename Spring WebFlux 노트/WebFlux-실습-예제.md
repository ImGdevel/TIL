# Spring WebFlux 실습: 라우터·핸들러 예제

> 태그: `#webflux` `#practice` `#router-function` `#sse`<br>
> 작성일: 2026-06-25<br>
> 최종 수정일: 2026-06-25

## 정의

[WebFlux-기초](WebFlux-기초.md), [WebFlux-개요](WebFlux-개요.md), [WebFlux-개요-심화](WebFlux-개요-심화.md)에서 정리한 이론을 실제 코드 예제로 연습하기 위한 가이드로, 가장 단순한 WebFlux API부터 함수형 라우팅, 스트리밍 엔드포인트, 외부 API 호출, Context 전파, 채팅 브로드캐스트까지 직접 만들어 보는 실습 노트다.

## 특징 / 상세

### 1. 공통 준비 사항

- 프로젝트: Spring WebFlux 의존성이 포함된 Spring Boot 프로젝트라고 가정한다. 기본 엔트리 포인트는 `WebfluxApplication`(이미 존재).
- 패키지 예시: `com.study.webflux.handler`, `com.study.webflux.router`, `com.study.webflux.controller`
- 아래 예제들은 패키지 구조와 클래스 이름만 참고용이며, 실제 프로젝트 스타일에 맞게 조정하면 된다.

### 2. 예제 1 – 가장 단순한 @RestController + Mono/Flux

목표: 기존 MVC와 비슷한 느낌으로 진입하되, 리턴 타입을 `Mono`/`Flux`로 바꿔 보는 것부터 시작한다.

- 기능
  - `GET /hello` → 문자열 하나(`Mono<String>`)
  - `GET /numbers` → 숫자 여러 개(`Flux<Integer>`)
- 핵심 포인트
  - `@RestController` + `@GetMapping`은 MVC와 동일하다.
  - 차이는 반환 타입이 `String`/`List<T>`가 아니라 `Mono<T>`/`Flux<T>`라는 점.
  - 내부에서 별도의 스레드 작업을 하지 않아도, WebFlux는 이미 논블로킹 Netty 위에서 동작한다.

### 3. 예제 2 – 함수형 라우터(RouterFunction) + HandlerFunction

목표: WebFlux 고유의 함수형 라우팅 스타일을 연습한다.

- 구성
  1. `GreetingHandler` – 실제 비즈니스 로직 처리
  2. `GreetingRouter` – HTTP 메서드/경로에 따라 어떤 핸들러를 호출할지 정의
- 기능 예시
  - `GET /functional/hello` → `"Hello Functional WebFlux!"`
  - `GET /functional/user/{name}` → `"Hello, {name}!"`
- 핵심 포인트
  - 라우팅 정의는 `RouterFunction<ServerResponse>`를 반환하는 `@Bean` 메서드에서 한다.
  - `RouterFunctions.route()` + `RequestPredicates.GET("...")` 방식으로 구성.
  - 핸들러는 `ServerRequest` → `Mono<ServerResponse>` 형태의 함수를 구현한다.

### 4. 예제 3 – 스트리밍 엔드포인트 (Flux + interval)

목표: WebFlux의 "스트리밍" 감각을 느껴보기 위해, 일정 주기로 숫자를 계속 보내는 엔드포인트를 만들어 본다.

- 기능: `GET /stream/numbers` → 1초마다 증가하는 숫자를 계속 내보내는 SSE(Server-Sent Events) 스타일 스트림.
- 개념 흐름
  - `Flux.interval(Duration.ofSeconds(1))`로 0,1,2,... 를 생성.
  - 필요하면 `.map`으로 메시지를 문자열로 변환.
  - `MediaType.TEXT_EVENT_STREAM`으로 응답하면 브라우저에서 스트림으로 볼 수 있다.
- 관찰 포인트
  - 브라우저/HTTP 클라이언트에서 응답이 한 번에 끝나지 않고 계속 이어지는 형태를 확인한다.
  - WebFlux가 이처럼 연속 데이터 스트림을 다루기에 자연스럽다는 점을 체감한다.

### 5. 예제 4 – WebClient를 이용한 비동기 호출

목표: WebFlux 서버에서 또 다른 HTTP API를 비동기적으로 호출하는 패턴을 연습한다.

- 기능: `GET /client/hello` → 내부적으로 다른 서비스의 `/hello` 엔드포인트를 WebClient로 호출 후 응답 반환.
- 핵심 포인트
  - `WebClient`를 사용하면 블로킹 없이 외부 HTTP 호출을 할 수 있다.
  - 반환 타입 역시 `Mono`/`Flux`이므로, 전체 파이프라인이 리액티브하게 이어진다.
  - MVC에서도 WebClient를 쓸 수 있지만, WebFlux와 함께 사용하면 스택 전체가 리액티브가 된다.

### 6. 예제 5 – 간단한 서비스 레이어 분리 + Reactor 연산자

목표: 컨트롤러/핸들러에서 서비스 레이어를 분리하고, `map`, `flatMap` 같은 Reactor 연산자를 함께 써 보는 연습을 한다.

- 시나리오
  - `HelloService`: `Mono<String> getGreeting(String name)` – 비동기적으로 인사 메시지 생성.
  - 컨트롤러/핸들러: `GET /hello/{name}` → `HelloService`를 호출해 결과 반환.
  - 연산자: `map`으로 문자열을 가공하거나, `flatMap`으로 다른 `Mono`/`Flux`를 이어 붙인다.
- 관찰 포인트
  - 기존 Service/Repository 계층 구조를 유지하면서도 반환 타입만 `Mono`/`Flux`로 바뀌었을 때, 전체 흐름이 어떻게 달라지는지 감각을 익힌다.

### 7. 테스트 및 실행 아이디어

- 실행: `WebfluxApplication`을 실행한 뒤, 브라우저 또는 `curl`, Postman으로 엔드포인트 호출. 스트리밍 엔드포인트는 브라우저 탭에서 직접 확인하면 이해가 쉽다.
- 간단한 검사: 각 엔드포인트에서 스레드 이름을 로그로 찍어 두고, 요청을 여러 번 보내면서 어떤 스레드에서 처리되는지 관찰해 본다. 이후 `Schedulers`를 함께 사용하면서 스레드가 어떻게 바뀌는지도 연계해서 실습하면 좋다.

### 8. 예제 6 – 외부 API 호출(WebClient)

목표: `WebClient`를 사용해 외부 서비스를 비동기적으로 호출하고, 결과를 그대로 전달하거나 환경에 맞게 가공하여 응답한다.

- 엔드포인트
  - `GET /api/external/joke` → [https://api.chucknorris.io/jokes/random](https://api.chucknorris.io/jokes/random)에 비동기 요청 후 결과 반환
  - `Accept-Language` 헤더를 읽어 응답 값 앞에 `[lang=xx]` 프리픽스를 붙임
- 포인트
  - `JokeService`는 `WebClient.Builder`를 주입받아 외부 API를 호출
  - 실패 시 `onErrorResume`으로 대체 메시지 응답
  - 전체 파이프라인이 `Mono<JokeResponse>` 형태로 처리되며 논블로킹

### 9. 예제 7 – TraceId 기반 Reactor Context + WebFilter

목표: `WebFilter`에서 `traceId`를 생성하여 응답 헤더와 Reactor Context에 저장하고, 핸들러가 해당 값으로 로그를 찍도록 구성한다.

- 흐름
  - 클라이언트가 `X-Trace-Id` 헤더를 보내면 그대로 사용하고, 없으면 UUID를 생성.
  - `TraceIdFilter`가 응답 헤더에 `X-Trace-Id`를 추가하고 `.contextWrite(ctx -> ctx.put("traceId", traceId))`로 Reactor Context에 저장.
  - `TimeController` 등에서는 `Mono.deferContextual`/`Flux.deferContextual`을 사용해 traceId를 꺼내 로그나 응답에 활용할 수 있다.
  - 필요하다면 `ServerWebExchange` 속성이나 응답 본문에도 traceId를 노출할 수 있음.

### 10. 예제 8 – 채팅 메시지 브로드캐스트

목표: `Sinks.Many`를 이용해 POST 요청으로 들어온 메시지를 브로드캐스트하고, 클라이언트가 SSE로 실시간으로 구독하도록 구성한다.

- 엔드포인트
  - `POST /api/chat/messages` → `ChatMessageRequest`를 받아 Sinks에 메시지 등록 (201 Created)
  - `GET /api/chat/stream` → `Flux<ChatMessage>`를 SSE로 스트리밍
- 핵심 포인트
  - `ChatService`는 `Sinks.many().multicast().onBackpressureBuffer()`를 사용해 다수 구독자에게 메시지를 전달
  - 스트리밍 구간에서 메시지는 `Instant` 기반 타임스탬프와 함께 전달

### 11. 마무리

- 이 노트의 목표는 "WebFlux가 뭔지 알겠다"에서 한 단계 더 나아가, "직접 간단한 WebFlux API를 만들고, 스트림처럼 동작하는 것을 확인해 봤다" 수준까지 올라가는 것이다.
- 실제 프로젝트에서는 인증/로깅/컨텍스트/스케줄러/에러 처리 등 앞에서 정리한 내용들을 조합해 더 복잡한 흐름을 만들게 되므로, 여기 예제들을 발판으로 점차 범위를 넓혀 가면 좋다.

## 트레이드오프

해당 없음 (실습 예제이며, 개별 기법의 트레이드오프는 각 관련 노트 참고)

## 실무 경험

해당 없음

## 참고

- 예제 8번(외부 API 호출)에서 사용한 [https://api.chucknorris.io/jokes/random](https://api.chucknorris.io/jokes/random)은 원본 TIL 노트에 실습용 외부 API로 기재되어 있으나, 별도 공식 출처 인용은 없음. 그 외 예제는 원본 노트에 별도 출처 인용 없음. 필요 시 [Spring WebFlux 공식 레퍼런스](https://docs.spring.io/spring-framework/reference/web/webflux.html)와 [Project Reactor 공식 레퍼런스](https://projectreactor.io/docs/core/release/reference/)에서 직접 확인.

## 관련 내용

- [WebFlux-엔드포인트-구성](WebFlux-엔드포인트-구성.md)
- [Reactor-Context-실습](Reactor-Context-실습.md)
- [Reactor-에러처리](Reactor-에러처리.md)
- [Reactor-스케줄러-실습](Reactor-스케줄러-실습.md)
