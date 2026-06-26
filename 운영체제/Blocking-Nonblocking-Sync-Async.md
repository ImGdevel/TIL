# Blocking / Non-blocking / Sync / Async

> 태그: `#os` `#concurrency` `#io-model`<br>
> 작성일: 2026-06-25<br>
> 최종 수정일: 2026-06-25

## 정의

`Blocking/Non-blocking`은 호출이 제어권을 잡고 기다리는지 여부이고, `Sync/Async`는 작업 완료를 누가/어떻게 통지받아 흐름을 이어가는지의 차이다.

## 특징 / 상세

핵심 키워드: `control flow`, `wait`, `callback`, `polling`, `event loop`

두 축은 서로 독립이라 "Non-blocking + Sync", "Blocking + Async" 같은 조합도 가능하다.

### 1. Blocking vs Non-blocking

> 호출한 쪽이 멈춰 기다리면 blocking, 바로 돌아오면 non-blocking이다.

- Blocking: 결과가 준비될 때까지 호출이 반환하지 않는다.
- Non-blocking: 바로 반환하고, 결과는 나중에 확인하거나 통지받는다.

### 2. Sync vs Async

> 작업 완료 시점과 흐름 제어를 "누가 책임지느냐" 관점으로 보면 편하다.

- Sync: 호출한 쪽이 완료 시점을 책임지고 확인한다. (대기/반복 확인 등)
- Async: 호출된 쪽이 완료를 통지한다. (callback, event 등)

### 3. 자주 나오는 포인트

- `Non-blocking`이어도 결과를 계속 확인하면 `Sync`일 수 있다.
- `Async`라고 해서 항상 빠른 건 아니고, 구조적으로 다른 방식이다.

### 4. 한 번에 구분하는 감각

> `Blocking/Non-blocking`은 "지금 멈추는가", `Sync/Async`는 "완료를 누구 책임으로 받는가"다.

- Blocking: 호출한 쪽이 기다린다.
- Non-blocking: 호출한 쪽이 즉시 돌아온다.
- Sync: 완료 여부를 호출한 쪽이 직접 챙긴다.
- Async: 완료 통지가 나중에 들어온다.

### 5. 실전 포인트

> 프레임워크나 라이브러리 선택보다, 호출 흐름이 어디서 멈추는지부터 봐야 한다.

- 브라우저/이벤트 루프 계열은 `Async` 설계를 많이 쓴다.
- 서버 코드에서는 "Non-blocking인데 Sync처럼 반복 확인하는" 형태도 흔하다.
- 면접에서는 네 개를 독립 축으로 말해주면 혼동을 줄일 수 있다.

## 트레이드오프

해당 없음

## 실무 경험

해당 없음 (관련 실무 내용은 특징/상세 5장 "실전 포인트"에 포함)

## 참고

해당 없음 — 원본 TIL 노트에 출처가 명시되어 있지 않음.

## 관련 내용

- [Coroutine](Coroutine.md)
- [Concurrency-vs-Parallelism](Concurrency-vs-Parallelism.md)
