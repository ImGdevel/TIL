# [운영체제] Mutex / Semaphore

## 한 줄 요약

> <u>`Mutex`는 한 번에 하나만 들어가는 `lock`이고, `Semaphore`는 허용 개수를 카운팅해서 동시에 N개까지 허용하는 동기화 도구다.</u>

뮤텍스는 보통 소유권이 있고, 세마포어는 자원 개수를 제어하는 모델로 이해하면 헷갈리지 않는다.

핵심 키워드: `mutual exclusion`, `counting`, `binary semaphore`, `critical section`, `deadlock`

<br />
<br />

---

## 1. Mutex

> <u>임계 구역을 1개 스레드만 통과시키는 도구다.</u>

- 보통 `lock/unlock`를 쓴다.
- 소유권이 있어서 잠근 스레드가 푸는 모델이 자연스럽다.

<br />
<br />

---

## 2. Semaphore

> <u>동시에 접근 가능한 개수를 제한하는 도구다.</u>

- 카운팅 세마포어는 0..N 범위를 가진다.
- 바이너리 세마포어는 0/1이라 뮤텍스처럼 쓰이기도 한다.

<br />
<br />

---

## 3. 자주 나오는 포인트

- `Semaphore`는 풀을 관리하는 문제(예: 커넥션 풀)에 자주 나온다.
- 잘못 쓰면 `deadlock`, `starvation` 같은 문제가 생긴다.
