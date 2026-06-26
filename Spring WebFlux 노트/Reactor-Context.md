# Reactor Context

> 태그: `#webflux` `#reactor` `#context`<br>
> 작성일: 2026-06-25<br>
> 최종 수정일: 2026-06-25

## 정의

Reactor Context는 `ThreadLocal`이 비동기/논블로킹 환경에서 스레드 전환 시 값을 잃어버리는 한계를 해결하기 위해, 리액티브 체인 자체에 데이터를 담아 구독(subscribe) 시점부터 하향(top-down)으로 전파시키는 불변(immutable) 키-값 저장소다.

## 특징 / 상세

### 1. 정의와 목적

#### 1.1 ThreadLocal의 한계와 컨텍스트의 필요성

- `ThreadLocal`은 하나의 스레드에 값을 묶어두는 방식이라, 리액티브 체인이 `subscribeOn`/`publishOn`으로 스레드를 전환하면 이전 스레드에 저장된 값을 잃어버린다.
- 트랜잭션 ID, 사용자 인증 정보, 추적(Trace) ID처럼 요청 전체에 걸쳐 유지되어야 하는 데이터를 다루려면, 스레드가 아니라 "구독 체인"에 값을 묶어야 한다 — 이것이 Reactor Context의 존재 이유다.

### 2. 핵심 특징

#### 2.1 불변성(Immutability)

- Context는 한 번 생성되면 값을 변경할 수 없고, 값을 추가/수정하려면 새로운 Context 인스턴스가 생성된다.
- 불변성이 필요한 이유 1 — 데이터 안전성: 여러 연산자/스레드가 동시에 같은 Context에 접근하더라도, 불변 객체이므로 원본이 훼손되지 않는다.
- 불변성이 필요한 이유 2 — 동시성 이슈 방지: 가변 공유 상태가 없으므로 동기화(lock) 없이도 안전하게 여러 구독자/스레드 간에 전달할 수 있다.

#### 2.2 리액터 연산자와의 통합

- Context는 일반 변수처럼 직접 꺼내 쓰는 것이 아니라, `contextWrite`, `deferContextual` 같은 전용 연산자를 통해 체인에 결합된다.
- 체인 내의 각 연산자는 필요할 때 현재 Context를 읽어 자신의 동작에 활용할 수 있다.

#### 2.3 지연된 컨텍스트 설정

- Context 값은 구독이 실제로 일어나기 전까지는 "확정"되지 않고, 구독 시점에 체인을 따라 구성된다.
- 따라서 동일한 Publisher 정의라도 구독 시점/위치에 따라 다른 Context 값을 가질 수 있다.

### 3. 사용 방법과 동작 원리

#### 3.1 contextWrite

##### 3.1.1 컨텍스트 생성 및 추가

- `contextWrite`는 현재 체인에 새로운 키-값을 추가(또는 갱신)한 새로운 Context를 만들어 그 지점부터 하위로 전파한다.
- 같은 체인에 여러 번 `contextWrite`를 호출하면, 더 위쪽(상류, upstream 방향)에서 호출된 `contextWrite`가 이후 값을 덮어쓸 수 있다는 점에 유의해야 한다(전파 방향 참고).

#### 3.2 deferContextual

##### 3.2.1 개념적 예제 구조

- `deferContextual`은 현재 시점의 Context를 읽어와, 그 값을 기반으로 새로운 Publisher를 구성할 수 있게 해주는 연산자다.
- 일반 연산자 내부에서는 Context에 직접 접근할 수 없는 경우가 많으므로, Context 값을 읽어 분기 처리해야 할 때 `deferContextual`을 사용한다.

#### 3.3 전파 방향 및 적용 순서

##### 3.3.1 체이닝 시 주의사항

- Context는 구독 체인을 따라 "아래에서 위로" 요청이 전달되고, Context 값 자체는 "위에서 아래로(상류 → 하류)" 전파된다.
- `contextWrite`는 체인 상에서 자신보다 위(상류)에 있는 연산자들에게는 영향을 주지 못하고, 자신보다 아래(하류)에 있는 연산자에만 적용된다.
- 따라서 `contextWrite`의 위치를 체인의 어디에 두는지가 어떤 연산자가 그 값을 볼 수 있는지를 결정한다 — 일반적으로 체인의 가장 마지막(구독에 가장 가까운 쪽)에 두어야 의도한 범위 전체에 적용된다.

### 4. 결론

- Reactor Context는 `ThreadLocal`이 비동기 환경에서 갖는 한계를 해결하기 위해 등장한, 구독 체인에 결합된 불변 데이터 저장소다.
- `contextWrite`로 값을 추가하고 `deferContextual`로 값을 읽으며, 전파 방향(상류 → 하류)과 적용 순서를 정확히 이해해야 의도한 대로 동작한다.
- 트랜잭션 ID, 인증 정보, 추적 ID 같은 横단(cross-cutting) 데이터를 리액티브 체인 전체에 안전하게 전달하는 표준적인 방법이다.

## 트레이드오프

해당 없음 (개념 정의이며, WebFilter를 통한 TraceId 전파 등 실제 적용 예시는 [WebFlux-실습-예제](WebFlux-실습-예제.md) 9번 항목 참고)

## 실무 경험

해당 없음

## 참고

- 원본 TIL 노트에 별도 출처 인용 없음. 필요 시 [Project Reactor 공식 레퍼런스 - Adding a Context to a Reactive Sequence](https://projectreactor.io/docs/core/release/reference/)에서 직접 확인.

## 관련 내용

- [Reactor-Context-실습](Reactor-Context-실습.md)
- [Reactor-스케줄러](Reactor-스케줄러.md)
- [WebFlux-실습-예제](WebFlux-실습-예제.md)
