# Reactor subscribe / buffer / take 연산자

> 태그: `#webflux` `#reactor` `#backpressure` `#operator`<br>
> 작성일: 2026-06-25<br>
> 최종 수정일: 2026-06-25

## 정의

`subscribe`는 Mono/Flux의 구독을 시작해 데이터 소비를 트리거하는 진입점이고, `buffer`는 스트림을 종료하지 않은 채 요소를 List로 묶어 배치 처리하는 수집 연산자, `take`는 조건 충족 시 스트림 자체를 종료시키는 차단 연산자다.

## 특징 / 상세

### 1. subscribe 메서드: 리액티브 스트림의 시작

Reactive Stream의 생명주기는 Publisher에 대한 구독(Subscription)이 발생할 때 시작된다. `subscribe`는 그 트리거이며, Mono/Flux가 실제로 데이터를 내보내기 위해 호출해야 하는 마지막 단계다.

#### 1.1 오버로딩과 흐름 제어

- onNext 컨슈머: 발행된 각 요소를 처리한다.
- onError 컨슈머: 스트림 중 오류 발생 시 호출된다.
- onComplete 컨슈머: 스트림이 정상 종료되었음을 알린다.
- Context 전달: `initialContext` 매개변수를 통해 파이프라인 전반에서 공유할 상태/메타데이터를 넣을 수 있다. 각 연산자는 컨텍스트에서 값을 읽거나 수정할 수 있으며, 보조적인 통신 채널이 된다.
- Subscriber 직접 전달: `org.reactivestreams.Subscriber`를 직접 생성해 전달하면 `onSubscribe` 시점에 `Subscription`을 받아 `request` 호출로 백프레셔를 직접 제어할 수 있다. 복잡한 흐름 제어나 커스텀 동작이 필요한 경우 유용하다.

#### 1.2 Disposable과 구독 제어

- 모든 `subscribe`는 `reactor.core.Disposable` 구현체를 반환한다.
- `dispose()`를 호출하면 활성화된 구독이 즉시 취소되어 스트림이 종료되고 이후 신호를 받지 않는다.
- 옵저버 패턴에서 옵저버 등록 해지와 같은 개념이며, 필요 시 메모리/리소스를 해제할 때 사용한다.

### 2. 백프레셔의 개념과 BaseSubscriber 사례

백프레셔는 생산자(Publisher)가 소비자(Subscriber)의 처리 속도를 초과해 데이터를 방출할 때 생기는 과부하를 방지하는 전략이다. 소비자가 자신의 처리 능력만큼 `request(n)`으로 데이터 요청량을 조절한다.

#### 2.1 왜 필요한가

- 생산자가 소비자보다 빠르면 데이터 유실, 메모리 과부하, GC 압박이 생긴다.
- 백프레셔 없이 무작정 데이터를 받아들이면 시스템 전체가 불안정해지므로, 소비자가 능동적으로 요청량을 관리해야 한다.

#### 2.2 BaseSubscriber 커스터마이징 방식

- `hookOnSubscribe(Subscription subscription)`에서 최초 `request(1)`처럼 원하는 개수를 요청해 흐름을 제어한다.
- `hookOnNext(T value)` 내부에서 현재 상태를 점검하고, 처리가 끝난 뒤 다시 `request(n)`을 호출해 다음 데이터를 요청한다.

예시: `Flux.range(1, 100)`에 대해 매 초 하나씩 처리한다고 가정하면, `hookOnSubscribe`에서 `request(1)`, `hookOnNext`에서 처리 후 `Thread.sleep(1000)`을 수행하고 다시 `request(1)`을 호출하면 생산자 속도와 소비자 속도가 동기화된다. 추가로 시스템 부하 모니터링 조건을 넣어 요청량을 동적으로 변경하면 고도화된 전략이 완성된다.

### 3. buffer 연산자: 집합 단위 처리

`buffer`는 업스트림에서 나오는 요소들을 일정 조건에 따라 묶어 `List<T>`로 내려보내는 수집 연산자다. 결과적으로 다운스트림은 단일 요소가 아닌 묶음 단위로 데이터를 받는다.

#### 3.1 핵심 역할

- 버퍼는 데이터를 담는 그릇처럼 작동하며, 조건 충족 시 그릇 전체를 한 번에 전달한다.
- 요소를 일일이 처리하기보다 배치/집합 단위로 다루어 처리량을 제어하거나 상위 계층 호출 횟수를 줄일 수 있다.
- 스트림 자체는 종료시키지 않으며, 계속해서 새로운 버퍼를 생성해 데이터를 전달한다.

#### 3.2 대표 메서드

| 메서드 | 설명 |
| --- | --- |
| `buffer(int size)` | 지정된 개수만큼 쌓이면 리스트로 묶어 방출한다. |
| `buffer(Duration timespan)` | 주어진 시간 동안 쌓인 데이터를 하나로 묶는다. |
| `bufferTimeout(int size, Duration timespan)` | 개수 또는 시간이 먼저 충족되면 방출한다. |
| `bufferUntil(Predicate)` | 조건이 true가 될 때까지 모아 방출한다. |
| `bufferWhile(Predicate)` | 조건이 true인 동안 모아 방출한다. |

#### 3.3 주의할 점

- 핫 퍼블리셔에서 크기 기반 버퍼만 쓰면 마지막 미충족 버퍼가 방출되지 않아 데이터 유실이 생길 수 있다.
- `bufferUntil`/`bufferWhile`의 조건이 영원히 만족되지 않으면 데이터가 계속 쌓여 OOM이 발생할 수 있다.
- 이러한 경우 `bufferTimeout`처럼 시간 조건을 결합하거나, 조건이 오래 지속되면 강제로 플러시하는 로직을 추가하는 보완이 필요하다.

### 4. take 연산자: 흐름 제한

`take`는 업스트림에서 방출된 요소를 특정 조건까지 통과시킨 뒤 스트림을 종료하는 차단 연산자다. 종료된 이후에는 아무런 데이터도 전달되지 않는다.

#### 4.1 핵심 역할

- 데이터 흐름을 제한해 필요한 만큼만 처리하거나, 특정 시점에 스트림을 종료하는 역할을 한다.
- 완료된 이후에는 `onComplete`가 downstream으로 전달되어 스트림이 닫힌다.
- `buffer`와 달리 take는 데이터를 그대로 전달하고, 조건 만족 시 스트림 자체를 닫는다.

#### 4.2 대표 메서드

| 메서드 | 설명 |
| --- | --- |
| `take(long n)` | 처음 `n`개의 요소만 통과시키고 종료한다. |
| `take(Duration timespan)` | 지정된 시간 동안만 데이터를 통과시키고 종료한다. |
| `takeWhile(Predicate)` | 조건이 true인 동안 요소를 통과시키고 false가 되면 종료한다. |
| `takeUntil(Predicate)` | 조건이 true가 될 때까지 통과시키고, true인 순간 종료한다. |
| `takeUntilOther(Publisher)` | 다른 퍼블리셔가 신호를 보내면 즉시 종료한다. |

### 5. buffer vs take 비교

| 구분 | buffer | take |
| --- | --- | --- |
| 목적 | 데이터를 수집/변환하여 집합 단위로 처리 | 스트림을 제한/종료해 불필요한 흐름을 막음 |
| 동작 | T → List<T>로 변환하고 스트림을 유지 | 데이터를 T 그대로 통과시키다가 조건 충족 시 종료 |
| 스트림 영향 | 스트림을 종료시키지 않음 | 조건 만족 즉시 `onComplete`를 내려보내고 종료 |
| 비유 | 데이터를 담는 그릇 | 흐름을 차단하는 문지기 |

소비자 관점에서 `buffer`는 데이터를 묶어서 배치 처리하고, `take`는 흐름을 걸러내 스트림을 닫는다. 두 연산자를 적절히 조합하면 대용량 데이터 처리에서 안정성과 성능을 모두 확보할 수 있다.

## 트레이드오프

해당 없음 (개별 연산자 사용법이며, buffer 크기/시간 설정에 따른 실질적 트레이드오프는 [Reactor-백프레셔](Reactor-백프레셔.md)의 6.1절 참고)

## 실무 경험

해당 없음 (관련 주의사항은 특징/상세 3.3번 "주의할 점" 참고)

## 참고

- 원본 TIL 노트에 별도 출처 인용 없음. 필요 시 [Project Reactor 공식 레퍼런스](https://projectreactor.io/docs/core/release/reference/)에서 `buffer`/`take`/`BaseSubscriber` API를 직접 확인.

## 관련 내용

- [Reactor-Mono-Flux-개요](Reactor-Mono-Flux-개요.md)
- [Reactor-백프레셔](Reactor-백프레셔.md)
