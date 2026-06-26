# Spring WebFlux 웹 요청·응답 처리 인터페이스

> 태그: `#webflux` `#serverwebexchange` `#http`<br>
> 작성일: 2026-06-25<br>
> 최종 수정일: 2026-06-25

## 정의

Spring WebFlux에서 HTTP 요청·응답을 다룰 때 핵심이 되는 인터페이스는 클라이언트 요청 정보를 조회하는 `ServerHttpRequest`, 서버 응답을 구성하는 `ServerHttpResponse`, 하나의 요청-응답 교환을 캡슐화한 통합 컨텍스트인 `ServerWebExchange` 세 가지다.

## 특징 / 상세

### 0. 요약

- Spring WebFlux에서 HTTP 요청·응답을 다룰 때 핵심이 되는 인터페이스는 세 가지이다.
  - `ServerHttpRequest`: 클라이언트 요청 정보 조회 (헤더, 쿼리 파라미터, 쿠키, 바디 등)
  - `ServerHttpResponse`: 서버에서 보내는 응답 구성 (헤더, 쿠키, 상태 코드, 바디 작성)
  - `ServerWebExchange`: 하나의 요청-응답 교환을 캡슐화한 통합 컨텍스트 (요청·응답·세션·속성 공유)
- 이 인터페이스들은 `@PathVariable` 같은 어노테이션 기반 방식과 함께, 특히 핸들러 함수(Handler Functions) 환경에서 WebFlux의 HTTP 처리를 보다 근본적이고 저수준에 가깝게 다루는 수단을 제공한다.

### 1. 서론

- WebFlux는 리액티브 프로그래밍 모델을 기반으로, HTTP 요청과 응답을 효율적으로 처리하기 위한 다양한 메커니즘을 제공한다.
- 컨트롤러/핸들러 메서드의 매개변수로 직접 받을 수 있는 여러 타입 중, `ServerHttpRequest`, `ServerHttpResponse`, `ServerWebExchange`는 HTTP 통신의 핵심 요소를 직접 제어할 수 있게 해 주는 중요한 인터페이스이다.
- 이 노트에서는 세 인터페이스의 역할과 주요 메서드, 사용 시나리오를 정리한다.

### 2. ServerHttpRequest: 클라이언트 요청 정보 처리

- 역할: 클라이언트가 보낸 HTTP 요청에 담긴 모든 정보에 접근할 수 있는 인터페이스. 요청 라인, 헤더, 쿼리 파라미터, 쿠키, 바디 등을 조회한다.

#### 2.1 핵심 기능 및 메서드

| 기능 분류 | 메서드 예시 | 설명 |
| --- | --- | --- |
| 헤더 조회 | `getHeaders().getFirst("key")` | 요청 헤더에서 특정 키에 해당하는 첫 번째 값을 조회. 헤더는 복수 값 가능이므로 리스트 기반에서 첫 값을 꺼낸다고 이해하면 된다. |
| 쿼리 파라미터 | `getQueryParams().getFirst("key")` | URL 쿼리 스트링 `?key=value`에서 특정 키의 첫 값을 조회. 헤더 조회와 유사한 패턴. |
| 쿠키 조회 | `getCookies().getFirst("key")` | 요청에 포함된 쿠키 중 특정 이름을 가진 쿠키 값을 조회. 쿠키도 복수 개가 있을 수 있다. |
| 요청 바디 읽기 | `getBody()` | 요청 바디를 `Flux<DataBuffer>` 형태로 반환. 비동기 데이터 스트림이므로 `map`, `flatMap` 등으로 가공하며 사용. |

- 요약: `ServerHttpRequest`는 "들어온 요청을 읽어들이는 창구" 역할을 한다.

### 3. ServerHttpResponse: 서버 응답 구성

- 역할: 서버 비즈니스 로직 처리 후, 클라이언트로 보낼 HTTP 응답을 구성하는 인터페이스. 헤더/쿠키/상태 코드/바디를 설정한다.

#### 3.1 핵심 기능 및 메서드

| 기능 분류 | 메서드 예시 | 설명 |
| --- | --- | --- |
| 헤더 설정 | `getHeaders().add("key", "value")` | 응답 헤더에 새로운 값 추가. 기존 헤더 맵을 가져와 수정하는 방식. |
| 쿠키 추가 | `addCookie(ResponseCookie.from(...).build())` | 응답에 쿠키를 추가. `ResponseCookie.from(...)` 팩토리 메서드로 쿠키 생성 후 전달. |
| 상태 코드 설정 | `setStatusCode(HttpStatus.OK)` | HTTP 응답 상태 코드를 설정. `HttpStatus` enum 사용으로 상태를 명확하게 표현. |
| 응답 바디 작성 | `writeWith(Publisher<DataBuffer>)` | 응답 바디를 작성. `Flux<DataBuffer>` 같은 Publisher를 사용해 비동기 스트림을 전송. |

- 요약: `ServerHttpResponse`는 "무엇을 어떻게 돌려줄지 결정하는 출력 창구" 역할을 한다.

### 4. ServerWebExchange: 요청·응답의 통합 컨텍스트

- 역할: 단일 HTTP 요청-응답 교환(Exchange)의 전체 컨텍스트를 캡슐화하는 인터페이스. 내부에 `ServerHttpRequest`, `ServerHttpResponse`를 모두 가지고 있으며, 추가로 속성, 세션 등도 함께 관리한다.

#### 4.1 핵심 기능 및 특징

- 중심적 역할: WebFlux 내부 처리 과정(전통 MVC의 `DispatcherServlet` 흐름에 해당하는 부분)에서 핸들러 체인, 필터 체인 등에 전달되는 핵심 컨텍스트 객체이다.
- 요청·응답 객체 접근
  - `getRequest()` → `ServerHttpRequest` 반환
  - `getResponse()` → `ServerHttpResponse` 반환
  - 하나의 `exchange`로 요청과 응답 모두를 제어할 수 있다.
- 속성(Attribute) 공유
  - `getAttributes()` → `Map<String, Object>` 형태의 저장소
  - 필터, 핸들러 등 여러 컴포넌트 간에 데이터를 공유하는 용도로 사용 가능.
- 세션 관리
  - `getSession()` → `Mono<WebSession>`
  - 여러 요청에 걸쳐 상태를 유지해야 하는 시나리오에서 WebSession 접근을 제공한다.
- 통합적 데이터 처리 예시
  - `exchange.getRequest().getHeaders().getFirst("User-Agent")` — 요청 헤더에서 User-Agent 조회.
  - 조회한 값을 가공해 `exchange.getResponse()`를 통해 응답 구성.
- 요약: `ServerWebExchange`는 "하나의 요청-응답 사이클 전체를 담고 있는 컨테이너"라고 이해할 수 있다.

### 5. 결론 및 추가 컨텍스트

- `ServerHttpRequest`, `ServerHttpResponse`, `ServerWebExchange`는 Spring WebFlux에서 HTTP 요청·응답을 다루는 가장 기본적인 API이다.
- 일반적으로는 컨트롤러에서 `@PathVariable`, `@RequestParam`, `@RequestBody`, `@RequestHeader` 등의 어노테이션을 사용하는 것이 더 단순하고 읽기 쉽다.
- 하지만, 보다 정밀하게 HTTP 레벨을 제어해야 하거나, 핸들러 함수(Handler Functions) 기반 WebFlux 애플리케이션을 구성할 때는 이 인터페이스들을 직접 사용하는 방식이 더 자연스럽고 간결한 경우가 많다.
- 따라서 이 세 인터페이스의 역할과 주요 메서드를 이해하는 것은, WebFlux를 단순 컨트롤러 수준을 넘어 더 깊게 활용하기 위한 필수 기반 지식이라고 할 수 있다.

## 트레이드오프

해당 없음 (저수준 인터페이스 레퍼런스이며, 어노테이션 기반 방식과의 선택 기준은 특징/상세 5번 "결론 및 추가 컨텍스트" 참고)

## 실무 경험

해당 없음

## 참고

- 원본 TIL 노트에 별도 출처 인용 없음. 필요 시 [Spring WebFlux 공식 Javadoc - ServerWebExchange](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/server/ServerWebExchange.html)에서 직접 확인.

## 관련 내용

- [WebFlux-엔드포인트-구성](WebFlux-엔드포인트-구성.md)
- [WebFlux-실습-예제](WebFlux-실습-예제.md)
