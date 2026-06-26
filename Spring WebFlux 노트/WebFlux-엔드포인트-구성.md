# Spring WebFlux 엔드포인트 구성 방법론

> 태그: `#webflux` `#router-function` `#controller`<br>
> 작성일: 2026-06-25<br>
> 최종 수정일: 2026-06-25

## 정의

Spring WebFlux는 API 엔드포인트를 어노테이션 기반 컨트롤러와 라우터 함수(Router Function)라는 두 가지 방식으로 구성할 수 있으며, 두 방식 모두 경로 변수·쿼리 파라미터·요청 헤더·요청 본문을 처리하는 공통 패턴을 제공하고 응답은 `Mono`/`Flux`로 감싸 반환한다.

## 특징 / 상세

### 0. 요약

- Spring WebFlux에서는 API 엔드포인트를 구성하는 두 가지 핵심 방식이 있다.
  - 어노테이션 기반 컨트롤러: `@Controller`, `@GetMapping` 등 MVC와 유사한 스타일
  - 라우터 함수(Router Function): 함수형 프로그래밍 스타일의 라우팅 방식
- 두 방식 모두 경로 변수(Path Variable), 쿼리 파라미터(Query Parameter), 요청 헤더(Request Header), 요청 본문(Request Body)를 처리하는 공통 패턴을 제공한다.
- 응답은 `Mono`, `Flux` 같은 리액티브 타입으로 감싸 반환하며, 이를 통해 비동기 논블로킹 처리를 표현한다.
- 실제 애플리케이션에서는 한 엔드포인트 안에서 여러 기법을 조합해서 사용하는 경우가 많다.

### 1. 엔드포인트와 계층 구조 개요

일반적인 웹 애플리케이션은 다음과 같이 세 계층으로 나뉜다.

- 애플리케이션 계층 (Application Layer) — 클라이언트의 HTTP 요청을 직접 받는 최상위 계층. WebFlux의 엔드포인트(컨트롤러/라우터)가 여기에 해당한다.
- 서비스 계층 (Service Layer) — 비즈니스 로직 처리
- 데이터 접근 계층 (Data Access Layer) — DB, 외부 시스템과의 연동

이 노트는 이 중 애플리케이션 계층, 즉 WebFlux 엔드포인트를 어떻게 구성하는지에 초점을 맞춘다.

WebFlux가 제공하는 두 가지 주요 방법:

1. 어노테이션 기반 컨트롤러
2. 라우터 함수(Router Function)

### 2. 방법론 1: 어노테이션 기반 컨트롤러

- Spring MVC와 매우 유사한 방식으로 엔드포인트를 정의한다.
- 차이점: 반환 타입을 `Mono<T>`, `Flux<T>`로 감싸 비동기·리액티브 처리를 명시한다.

#### 2.1 파라미터 없는 기본 요청 처리

- 어노테이션: `@GetMapping`
- 특징:
  - 추가 파라미터 없이 고정된 응답을 반환.
  - 반환 타입은 `Mono<String>` 등 리액티브 타입.
  - `Mono.just(...)`로 단일 값을 감싸서 반환.
- 예: `GET /hello` → `"Hello Around Hub Studio"`를 `Mono<String>`로 반환.

#### 2.2 경로 변수(@PathVariable)

- 어노테이션: `@PathVariable`
- 특징:
  - `@GetMapping("/hello/{name}")`처럼 경로에 `{name}`을 정의.
  - 메서드 파라미터로 `@PathVariable String name`을 받아 사용.
  - 파라미터 이름과 경로 변수 키가 같으면 자동 매핑.
- 예: `/hello/John` 요청 시 `"Hello John"` 응답.

#### 2.3 쿼리 파라미터(@RequestParam)

- 어노테이션: `@RequestParam`
- 특징:
  - `?key=value` 형태의 쿼리 스트링에서 값을 전달받는다.
  - `defaultValue`로 기본값, `required`로 필수 여부를 설정 가능.
  - `@RequestParam(name = "page", defaultValue = "0") int page` 등의 패턴.

#### 2.4 요청 헤더(@RequestHeader)

- 어노테이션: `@RequestHeader`
- 특징:
  - `"X-Request-Id"` 같은 특정 헤더 값을 주입받아 사용.
  - `"X-"` 접두사는 주로 사용자 정의(Custom) 헤더에 사용되는 관례.
  - 헤더 값이 없을 수 있으므로 null/Optional 처리 고려.

#### 2.5 요청 본문(@RequestBody)

- 어노테이션: `@RequestBody`
- 특징:
  - 주로 `POST`, `PUT` 요청의 JSON Body를 DTO 객체로 역직렬화.
  - WebFlux에서는 파라미터 타입을 `Mono<User>`처럼 리액티브 타입으로 받는 것이 일반적.
  - 내부 데이터 접근은 `.map(...)`, `.flatMap(...)` 등 연산자를 통해 수행.
- 개념: `Mono<User> userMono`를 받아 `.map(user -> user.getName())` 등으로 내부 User 필드에 접근.

### 3. 방법론 2: 라우터 함수(Router Function)

- 함수형 프로그래밍 스타일로 라우팅 규칙과 핸들러를 코드로 정의하는 방식.

#### 3.1 핵심 구성 요소

- `RouterFunctions.route(...)` — 라우팅 규칙을 정의하는 시작점.
- `RequestPredicates.GET`, `POST` 등 — HTTP 메서드 + 경로 패턴을 조합하여 요청을 판별.
- `HandlerFunction` — `ServerRequest`를 입력으로 받아 `Mono<ServerResponse>`를 반환하는 함수. 람다 또는 메서드 참조로 구현.
- 구조 예시(개념): `.GET("/path", request -> handler(request))`. 여러 규칙은 `.andRoute(...)` 또는 메서드 체이닝으로 연속 등록.

#### 3.2 파라미터 없는 기본 요청 처리

- 특징:
  - 단순히 특정 경로로 들어온 요청에 대해 고정 응답을 생성.
  - `ServerResponse.ok().bodyValue("...")` 형태로 상태 코드와 응답 Body 설정.

#### 3.3 경로 변수(Path Variable)

- 메서드: `request.pathVariable("key")`
- 특징:
  - 라우팅 정의에서 `/hello/{name}`처럼 `{name}` 사용.
  - 핸들러 내에서 `request.pathVariable("name")`으로 값 읽기.

#### 3.4 쿼리 파라미터(Query Parameter)

- 메서드: `request.queryParam("key")`
- 특징:
  - 반환 타입이 `Optional<String>`이므로 `.orElse("default")` 등으로 기본값을 지정하기 쉽다.

#### 3.5 요청 헤더(Header)

- 접근 방법: `request.headers().firstHeader("key")`
- 특징:
  - `request.headers()`로 전체 헤더에 접근.
  - `firstHeader("X-Request-Id")` 등으로 특정 키의 첫 값을 가져온다.

#### 3.6 요청 본문(Request Body)

- 메서드: `request.bodyToMono(User.class)`
- 특징:
  - 요청 Body를 지정한 타입의 `Mono<User>`로 변환.
  - 이후 `.flatMap(user -> ServerResponse.ok().bodyValue(...))` 형태로 처리.
  - 비동기·리액티브 방식이므로 `flatMap`으로 체인 구성이 중요하다.

### 4. 핵심 고려사항 및 정리

- 혼합 사용 가능
  - 경로 변수, 쿼리 파라미터, 헤더, 본문 처리 기법은 서로 독립적인 것이 아니다.
  - 실제 엔드포인트에서는 이들을 자유롭게 조합해서 사용한다.
  - 각 기법은 이해를 위해 분리해 설명했을 뿐이다.
- 비동기·리액티브 처리의 중요성
  - WebFlux 엔드포인트의 가장 큰 특징은 반환 타입이 `Mono`/`Flux`라는 점이다.
  - 이는 단순한 타입 차이가 아니라, 논블로킹·리액티브 데이터 스트림을 다루겠다는 선언이다.
  - 기존 MVC의 동기 처리와 다른 이 부분이 WebFlux의 성능·확장성을 결정하는 핵심이다.
- 컨트롤러 vs 라우터 함수 선택
  - MVC 스타일에 익숙하다면: 먼저 어노테이션 기반 컨트롤러로 시작한 뒤, 필요에 따라 일부 엔드포인트만 라우터 함수로 도입해 볼 수 있다.
  - 함수형 스타일과 DSL을 선호하거나, 라우팅을 코드로 명시적으로 제어하고 싶다면 라우터 함수가 잘 맞을 수 있다.

## 트레이드오프

해당 없음 (단일 프레임워크 내 두 가지 엔드포인트 구성 방식의 선택 문제이며, 구조적 차이는 특징/상세 4번 "컨트롤러 vs 라우터 함수 선택" 참고)

## 실무 경험

해당 없음

## 참고

- 원본 TIL 노트에 별도 출처 인용 없음. 필요 시 [Spring WebFlux 공식 레퍼런스 - Functional Endpoints](https://docs.spring.io/spring-framework/reference/web/webflux-functional.html)에서 직접 확인.

## 관련 내용

- [WebFlux-개요](WebFlux-개요.md)
- [WebFlux-실습-예제](WebFlux-실습-예제.md)
- [WebFlux-요청-응답-인터페이스](WebFlux-요청-응답-인터페이스.md)
