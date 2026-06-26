# Reactor 스케줄러 실습

> 태그: `#webflux` `#reactor` `#scheduler` `#practice`<br>
> 작성일: 2026-06-25<br>
> 최종 수정일: 2026-06-25

## 정의

[Reactor-스케줄러](Reactor-스케줄러.md)에서 정리한 스케줄러 선택 기준을, 스케줄러 미적용 기본 흐름부터 I/O-bound/CPU-bound 작업별 적용, `subscribeOn`과 `publishOn`의 차이 체험, 테스트 코드 작성 시 주의점까지 예제 중심으로 정리한 실습 노트다.

## 특징 / 상세

### 1. 공통 준비

- 각 예제는 동일한 형태의 `Mono`/`Flux` 체인에 스케줄러 지정 여부, 위치, 종류만 바꿔가며 실행 스레드 이름을 로그로 출력해 비교하는 방식을 사용한다.

### 2. 예제 1 — 스케줄러 없이 기본 흐름

- 별도 스케줄러 지정 없이 체인을 구성하면, 구독을 호출한 스레드(보통 메인 스레드 또는 WebFlux의 이벤트 루프 스레드)에서 전체 체인이 동기적으로 실행된다.
- 이 기본 동작을 먼저 확인해야, 이후 예제에서 스케줄러 적용으로 인한 스레드 변화를 명확히 비교할 수 있다.

### 3. 예제 2 — I/O-bound 작업 + boundedElastic

- 블로킹 I/O(예: 블로킹 JDBC 호출, 파일 읽기)를 모사한 작업을 `subscribeOn(Schedulers.boundedElastic())`으로 감싸 실행한다.
- 메인 스레드는 블로킹되지 않고, `boundedElastic` 풀의 스레드가 블로킹 작업을 전담해 처리한다는 것을 스레드 이름 로그로 확인한다.

### 4. 예제 3 — CPU-bound 작업 + parallel

- CPU 집약적인 연산(예: 반복적인 수치 계산)을 `Schedulers.parallel()`로 감싸 실행한다.
- 코어 수만큼의 스레드가 작업을 나누어 처리하며, `boundedElastic`처럼 스레드 수가 동적으로 늘어나지 않고 고정된 크기로 동작하는 것을 확인한다.

### 5. 예제 4 — subscribeOn vs publishOn 차이

#### 케이스 A

- 체인의 가장 앞부분에 `subscribeOn`을 한 번 호출 — 체인 전체(업스트림 포함)가 지정한 스케줄러의 스레드에서 실행된다.

#### 케이스 B

- 체인 중간에 `publishOn`을 호출 — 호출 지점 이전까지는 원래 스레드에서, 호출 지점 이후부터는 새로 지정한 스케줄러의 스레드에서 실행된다.

#### 케이스 C

- `subscribeOn`과 `publishOn`을 함께 사용 — `subscribeOn`이 결정한 초기 실행 스레드 위에서 시작하다가, `publishOn` 호출 지점부터 다시 스레드가 전환된다. 두 연산자를 조합하면 체인의 구간별로 서로 다른 스레드 정책을 적용할 수 있다.

### 6. 예제 5 — 테스트 코드 주의점

- 비동기로 다른 스레드에서 실행되는 체인을 테스트할 때, 테스트 메서드의 메인 스레드가 결과를 기다리지 않으면 비동기 작업이 완료되기 전에 테스트가 (거짓으로) 성공 종료될 수 있다.
- `CountDownLatch`를 사용해 비동기 콜백(`doOnNext`, `doOnComplete` 등) 내부에서 `countDown()`을 호출하고, 메인 테스트 스레드는 `latch.await(timeout)`으로 명시적으로 대기하는 패턴을 사용한다.
- `StepVerifier`를 사용하는 경우 `verifyComplete()`/`verifyError()` 같은 검증 메서드 자체가 내부적으로 완료를 기다리므로 별도의 `CountDownLatch`가 필요 없는 경우가 많다 — 다만 무한 스트림이나 특수한 시간 제어가 필요하면 `StepVerifier.withVirtualTime()` 등 추가 도구를 검토해야 한다.

### 7. 마무리 정리

- 스케줄러를 다루는 실습의 핵심은 "어떤 스레드에서 무엇이 실행되는가"를 직접 로그로 확인하는 것이며, 이를 통해 `subscribeOn`(업스트림 전체 스레드 결정)과 `publishOn`(호출 지점 이후 스레드 전환)의 차이를 체감할 수 있다.
- 테스트 코드를 작성할 때는 비동기 실행으로 인한 거짓 성공(false positive)을 방지하기 위한 명시적 대기 메커니즘이 필수적이다.

## 트레이드오프

해당 없음 (실습 예제이며, 스케줄러 선택의 트레이드오프는 [Reactor-스케줄러](Reactor-스케줄러.md) 참고)

## 실무 경험

해당 없음 (관련 주의사항은 특징/상세 6번 "테스트 코드 주의점" 참고)

## 참고

- 원본 TIL 노트에 별도 출처 인용 없음. 필요 시 [Project Reactor 공식 레퍼런스 - Testing](https://projectreactor.io/docs/test/release/reference/)에서 `StepVerifier`와 가상 시간(Virtual Time) 테스트 기법을 직접 확인.

## 관련 내용

- [Reactor-스케줄러](Reactor-스케줄러.md)
- [Reactor-Context-실습](Reactor-Context-실습.md)
