# Reactor Context 실습

> 태그: `#webflux` `#reactor` `#context` `#practice`<br>
> 작성일: 2026-06-25<br>
> 최종 수정일: 2026-06-25

## 정의

[Reactor-Context](Reactor-Context.md)에서 정리한 `contextWrite`/`deferContextual`의 동작 원리를, 기본 저장·조회부터 전파 방향 체험, WebFlux 핸들러·스케줄러 조합, 테스트 코드 검증까지 예제 중심으로 정리한 실습 노트다.

## 특징 / 상세

### 1. 공통 준비

- 모든 예제는 `Mono`/`Flux` 체인에 `contextWrite`로 값을 넣고, 체인 내부 연산자(`flatMap`, `map` 등)에서 `deferContextual` 또는 `Mono.deferContextual(ctx -> ...)` 형태로 값을 읽는 패턴을 공통적으로 사용한다.

### 2. 예제 1 — 기본 저장/조회

- 체인 마지막에 `contextWrite(ctx -> ctx.put("key", "value"))`를 붙이고, 체인 중간의 연산자에서 `Mono.deferContextual(ctx -> Mono.just(ctx.get("key")))`로 값을 꺼내 읽는다.
- 핵심 확인 포인트: Context는 일반 변수처럼 "어디서든 꺼내 쓰는" 것이 아니라, 전용 연산자를 통해서만 접근 가능하다는 것.

### 3. 예제 2 — put으로 값 추가/변경

- 기존 Context에 `ctx.put(key, newValue)`를 호출하면 원본은 그대로 두고 새로운 Context 인스턴스가 반환된다(불변성).
- 동일한 키에 다른 값을 두 번 `put`하면, 더 하류에 가까운 `contextWrite`의 값이 최종적으로 사용된다.

### 4. 예제 3 — 전파 방향과 덮어쓰기 체험

#### 시나리오 A

- 체인 상류에서 `contextWrite(A)`, 하류에서 `contextWrite(B)`를 호출한 뒤 값을 읽으면 — 하류의 `contextWrite(B)`가 우선 적용되어 B 값이 읽힌다. Context 값은 상류 → 하류로 전파되지만, 더 하류에서 다시 쓰면 그 지점부터는 새 값으로 덮어써진다.

#### 시나리오 B

- 값을 읽는 연산자가 `contextWrite`보다 상류에 위치하면, 그 `contextWrite`가 추가한 값을 볼 수 없다 — Context 전파는 자신보다 위(상류)에는 영향을 주지 못한다는 규칙을 직접 보여주는 사례다.

### 5. 예제 4 — WebFlux 핸들러와 컨텍스트 개념

- WebFlux 핸들러(Router Function 또는 컨트롤러)에서 요청별로 고유한 값(예: 요청 ID)을 `contextWrite`로 주입하고, 서비스 레이어의 체인 내부에서 해당 값을 꺼내 로깅 등에 활용하는 개념적 구조를 정리한다.
- 요청 단위로 격리된 값을 전달해야 하는 웹 요청 처리 흐름에 Context가 적합한 이유를 보여준다.

### 6. 예제 5 — 컨텍스트와 스케줄러 조합

- `subscribeOn`/`publishOn`으로 스레드가 전환되어도 Context 값은 체인을 따라 그대로 유지된다 — 이는 Context가 스레드가 아니라 구독 체인에 묶여 있기 때문이다.
- 이 지점이 `ThreadLocal`과의 본질적 차이를 가장 명확히 보여주는 예제다.

### 7. 예제 6 — 테스트 코드에서 검증

- `StepVerifier`를 사용해 체인이 기대한 Context 값을 가지고 있는지 검증한다.
- 예: `StepVerifier.create(mono).expectAccessibleContext().contains("key", "value").then().expectNext(...).verifyComplete()`와 같은 형태로, Context까지 포함한 체인 동작을 테스트 단계에서 명시적으로 검증할 수 있다.

### 8. 마무리 정리

- 모든 예제에서 공통적으로 확인되는 점: Context는 (1) 전용 연산자로만 접근, (2) 불변, (3) 상류→하류 전파, (4) 스레드와 무관하게 체인에 종속된다는 네 가지 특성을 갖는다.
- 실무에서는 요청 단위 추적 정보(TraceId 등)를 WebFilter에서 주입하고 체인 전체에서 일관되게 사용하는 패턴으로 가장 많이 활용된다.

## 트레이드오프

해당 없음 (실습 예제이며, 개념적 트레이드오프는 다루지 않음)

## 실무 경험

해당 없음 (관련 적용 예시는 [WebFlux-실습-예제](WebFlux-실습-예제.md) 9번 "TraceId 기반 Reactor Context + WebFilter" 참고)

## 참고

- 원본 TIL 노트에 별도 출처 인용 없음. 필요 시 [Project Reactor 공식 레퍼런스](https://projectreactor.io/docs/core/release/reference/)와 [reactor-test StepVerifier 문서](https://projectreactor.io/docs/test/release/reference/)에서 직접 확인.

## 관련 내용

- [Reactor-Context](Reactor-Context.md)
- [Reactor-스케줄러-실습](Reactor-스케줄러-실습.md)
- [WebFlux-실습-예제](WebFlux-실습-예제.md)
