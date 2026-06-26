# 리액티브 스트림즈(Reactive Streams)

> 태그: `#webflux` `#reactive-streams` `#backpressure`<br>
> 작성일: 2026-06-25<br>
> 최종 수정일: 2026-06-25

## 정의

리액티브 스트림즈는 비동기 데이터 스트림을 논블로킹 백프레셔(Backpressure)와 함께 처리하기 위한 표준 사양으로, `Publisher`/`Subscriber`/`Subscription`/`Processor` 네 인터페이스로 구성되며 Spring WebFlux의 Reactor(`Mono`/`Flux`)가 이 사양의 구현체다.

## 특징 / 상세

### 1. 리액티브 스트림즈 개요

#### 1.1 정의

- 리액티브 스트림즈는 리액티브 프로그래밍 패러다임에서 비동기 데이터 스트림을 다루기 위한 공통 표준이다.
- 핵심 목표:
  1. 데이터를 논블로킹 방식으로 처리하고,
  2. 소비자가 처리할 수 있는 만큼만 데이터를 전달하는 백프레셔 메커니즘을 표준화하는 것.

#### 1.2 핵심 인터페이스와 역할

- `Publisher`
  - 데이터 스트림을 생성하고 emit(발행)하는 주체.
  - 여러 개의 데이터를 하나의 스트림으로 묶어, 필요한 만큼 Subscriber에게 전달한다.
- `Subscriber`
  - Publisher가 발행한 데이터를 받아 소비/처리하는 주체.
  - `onNext`, `onError`, `onComplete` 메서드로 데이터를 처리하고 스트림 종료를 감지한다.
- `Subscription`
  - Publisher와 Subscriber 사이의 구독 정보를 나타내는 객체.
  - `request(n)`: Subscriber가 처리 가능한 데이터 개수 `n`을 요청
  - `cancel()`: 더 이상 데이터를 받고 싶지 않을 때 구독 해지
- `Processor`
  - Publisher + Subscriber 기능을 모두 가진 중간 처리자.
  - 한쪽에서는 Subscriber로서 데이터를 입력받고, 다른 쪽에서는 Publisher로서 가공된 데이터를 다시 내보낸다.
  - 실무에서는 Reactor의 연산자 체인(`map`, `flatMap` 등)이 유사한 역할을 많이 담당해, Processor 자체를 직접 다루는 경우는 상대적으로 적다.

#### 1.3 주요 구현체

- 리액티브 스트림즈는 "인터페이스와 규약"만 정의하므로, 실제 사용을 위해서는 구현 라이브러리가 필요하다.
- 대표적인 구현체:
  - Project Reactor: Spring WebFlux에서 사용하는 구현체 (`Mono`, `Flux`)
  - RxJava
  - 그 외 다양한 언어나 런타임에서 해당 사양을 구현

### 2. 리액티브 선언문(Reactive Manifesto)

리액티브 선언문은 리액티브 시스템이 갖춰야 할 특성을 방법(Means) → 형태(Form) → 가치(Value)의 구조로 정리한 문서이다.

#### 2.1 방법 (Means): 메시지 기반(Message Driven)

- 리액티브 시스템의 가장 근본적인 동작 수단은 비동기 메시지 기반 통신이다.
- 모든 상호작용은 비동기 메시지를 통해 이루어져야 한다.
  - 요청/응답, 이벤트 전달, 상태 변경 등이 메시지 단위로 오간다.
- 이를 위해 필요하고 자주 사용되는 요소:
  - 메시지 큐(Message Queue) / 메시지 브로커(Kafka, SQS 등)
  - 백프레셔(Backpressure) 메커니즘
  - 비동기 메시지 처리 스레드, 이벤트 루프 등

#### 2.2 형태 (Form): 탄력성(Elasticity)과 회복성(Resilience)

##### 탄력성 (Elasticity)

- 부하 변화에 유연하게 대응할 수 있는 능력.
- 정의: 들어오는 요청/메시지의 양이 많아지더라도, 응답 시간을 일정 수준으로 유지할 수 있도록 시스템을 확장·분산하는 능력.
- 목표:
  - 오토스케일링(Auto-Scaling), 수평 확장(Scale-out) 등을 통해 특정 노드에 부하가 집중되는 병목을 방지.
  - 자원이 남을 때는 축소(Scale-in)하여 비용 효율성도 확보.

##### 회복성 (Resilience)

- 장애가 발생했을 때도 시스템 전체가 무너지지 않고 버티는 능력.
- 핵심 원칙:
  - 낮은 결합도(Loose Coupling) — 비동기 메시징을 통해 컴포넌트 간 직접적인 의존성을 줄임.
  - 장애 격리(Fault Isolation) — 장애는 발생한 컴포넌트/서비스 안에서만 봉쇄(containment)되어야 한다. 하나가 죽었다고 전체가 함께 죽지 않도록 설계.
  - 복구 프로세스 — 모니터링과 헬스 체크를 통해 장애를 감지하고, 자동 재시작, 재시도, 페일오버 등의 메커니즘으로 스스로 복구할 수 있어야 한다.

#### 2.3 가치 (Value): 반응성(Responsiveness)

- 방법과 형태를 잘 갖추었을 때 최종적으로 사용자에게 제공해야 할 가치가 반응성이다.
- 정의: 사용자 혹은 다른 시스템에게 빠르고 일관된 응답 시간을 제공하는 것.
- 목표:
  - 문제를 빠르게 감지하고, 적절한 대처를 통해 서비스 품질을 유지.
  - 성능뿐만 아니라, 장애 상황에서도 예측 가능한 행동과 응답을 제공하는 것이 중요하다.

### 3. 적용 범위와 현대적 아키텍처

리액티브 선언문의 원칙은 특정 기술 스택에 한정되지 않고, 여러 수준에서 적용된다.

#### 3.1 애플리케이션 레벨

- 예: MVC 패턴에서 Controller, Service, Repository 간 책임 분리.
  - 각 계층이 자신의 역할에만 집중하도록 설계하여 결합도를 낮춤.
  - 내부에서도 이벤트/메시지 기반으로 모듈 간 상호작용을 설계할 수 있다.

#### 3.2 인프라스트럭처 / 아키텍처 레벨

- 마이크로서비스 아키텍처(MSA)
  - 각 서비스가 독립적으로 배포되고, 다른 서비스의 장애에 직접적으로 영향받지 않도록 설계.
  - 서비스 간 통신은 주로 메시지 기반(이벤트, 메시지 큐 등)으로 처리.
- 클라우드·컨테이너 환경
  - 쿠버네티스(Kubernetes): 애플리케이션을 작은 단위(Pod)로 나누어 배포. 자동 스케일링, 롤링 업데이트, 셀프 힐링 등을 통해 탄력성과 회복성을 지원.
  - 메시징 시스템 (Kafka, AWS SQS 등): 서비스 간의 통신을 메시지 큐로 위임하여 비동기 처리와 느슨한 결합을 실현.

### 4. 리액티브 스트림즈 상호작용 흐름

리액티브 스트림즈에서 `Publisher`, `Subscriber`, `Subscription`은 다음 순서로 상호작용한다.

1. 구독 요청 (`subscribe`) — `Subscriber`가 `Publisher`에게 구독을 요청한다. 예: `publisher.subscribe(subscriber);`
2. 구독 정보 전달 (`onSubscribe`) — `Publisher`는 `Subscription` 객체를 생성한 뒤, `Subscriber.onSubscribe(subscription)`를 호출해 전달한다. 이 시점부터 두 컴포넌트 사이에 공식적인 연결이 성립된다.
3. 데이터 요청 (`request`) — `Subscriber`는 전달받은 `Subscription`을 통해 `request(n)`을 호출한다. 여기서 `n`은 "한 번에 처리할 수 있는 데이터 개수"를 의미하며, 이것이 백프레셔의 핵심이다.
4. 데이터 전달 (`onNext`) — `Publisher`는 요청받은 개수만큼 데이터를 생성/발행하고, 각 데이터를 `Subscriber.onNext(T item)`을 통해 차례로 전달한다. 필요할 경우 `Subscriber`는 다시 `request()`를 호출하여 추가 데이터를 요청한다.
5. 스트림 종료 (`onComplete` / `onError`) — 모든 데이터를 정상적으로 전송 완료하면 `onComplete()`를 호출하여 스트림 종료를 알린다. 중간에 오류가 발생하면 `onError(Throwable e)`를 호출하여 비정상 종료를 알린다.
6. 구독 취소 (`cancel`) — `Subscriber`는 언제든지 `Subscription.cancel()`을 호출해 더 이상 데이터를 받지 않겠다고 선언할 수 있다. 이후에는 `onNext`, `onComplete`, `onError` 등 어떤 신호도 더 이상 수신하지 않는다.

이러한 규약 덕분에, 데이터 생산 속도가 소비 속도를 초과하더라도 시스템 자원을 효율적으로 사용하면서 안정적으로 스트림을 처리할 수 있다.

### 5. 정리

- 리액티브 스트림즈는 비동기 데이터 스트림 처리와 논블로킹 백프레셔를 표준화한 사양이다.
- 네 가지 인터페이스(Publisher, Subscriber, Subscription, Processor)를 통해 데이터 흐름과 상호작용 방식을 명확히 정의한다.
- 리액티브 선언문은 이러한 기술적 기반 위에 메시지 기반 통신(방법), 탄력성과 회복성(형태), 반응성(가치)을 갖춘 시스템을 설계해야 한다고 강조한다.
- Spring WebFlux에서 사용하는 Reactor 역시 이 리액티브 스트림즈 사양을 따르므로, `Mono` / `Flux` 를 사용할 때는 단순한 API 호출이 아니라 표준화된 데이터 스트림 프로토콜 위에서 동작하고 있다는 관점을 가질 수 있다.

## 트레이드오프

해당 없음 (사양/개념 정의이며, 구체적 설계 선택에 따른 트레이드오프는 [Reactor-백프레셔](Reactor-백프레셔.md)의 전략 비교에서 다룬다)

## 실무 경험

해당 없음

## 참고

- 원본 TIL 학습 노트에 별도 출처 인용이 없어, 1차 출처(리액티브 스트림즈 공식 사양 등) 대비 정확도는 별도로 검증되지 않았다. 필요 시 [Reactive Streams 공식 사양](https://www.reactive-streams.org/)과 [Reactive Manifesto](https://www.reactivemanifesto.org/)에서 직접 확인.

## 관련 내용

- [Reactor-Mono-Flux-개요](Reactor-Mono-Flux-개요.md)
- [Reactor-백프레셔](Reactor-백프레셔.md)
- [WebFlux-개요](WebFlux-개요.md)
