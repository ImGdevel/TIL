# [운영체제] Blocking / Non-blocking / Sync / Async

## 한 줄 요약

> <u>`Blocking/Non-blocking`은 호출이 제어권을 잡고 기다리는지 여부이고, `Sync/Async`는 작업 완료를 누가/어떻게 통지받아 흐름을 이어가는지의 차이다.</u>

두 축은 서로 독립이라 “Non-blocking + Sync”, “Blocking + Async” 같은 조합도 가능하다.

핵심 키워드: `control flow`, `wait`, `callback`, `polling`, `event loop`

<br />
<br />

---

## 1. Blocking vs Non-blocking

> <u>호출한 쪽이 멈춰 기다리면 blocking, 바로 돌아오면 non-blocking이다.</u>

- Blocking: 결과가 준비될 때까지 호출이 반환하지 않는다.
- Non-blocking: 바로 반환하고, 결과는 나중에 확인하거나 통지받는다.

<br />
<br />

---

## 2. Sync vs Async

> <u>작업 완료 시점과 흐름 제어를 “누가 책임지느냐” 관점으로 보면 편하다.</u>

- Sync: 호출한 쪽이 완료 시점을 책임지고 확인한다. (대기/반복 확인 등)
- Async: 호출된 쪽이 완료를 통지한다. (callback, event 등)

<br />
<br />

---

## 3. 자주 나오는 포인트

- `Non-blocking`이어도 결과를 계속 확인하면 `Sync`일 수 있다.
- `Async`라고 해서 항상 빠른 건 아니고, 구조적으로 다른 방식이다.

