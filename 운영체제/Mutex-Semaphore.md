# Mutex / Semaphore

> 태그: `#os` `#concurrency` `#mutex` `#semaphore`<br>
> 작성일: 2026-06-25<br>
> 최종 수정일: 2026-06-25

## 정의

`Mutex`는 한 번에 하나만 들어가는 `lock`이고, `Semaphore`는 허용 개수를 카운팅해서 동시에 N개까지 허용하는 동기화 도구다.

## 특징 / 상세

핵심 키워드: `mutual exclusion`, `counting`, `binary semaphore`, `critical section`, `deadlock`

뮤텍스는 보통 소유권이 있고, 세마포어는 자원 개수를 제어하는 모델로 이해하면 헷갈리지 않는다.

### 1. Mutex

> 임계 구역을 1개 스레드만 통과시키는 도구다.

- 보통 `lock/unlock`를 쓴다.
- 소유권이 있어서 잠근 스레드가 푸는 모델이 자연스럽다.

### 2. Semaphore

> 동시에 접근 가능한 개수를 제한하는 도구다.

- 카운팅 세마포어는 0..N 범위를 가진다.
- 바이너리 세마포어는 0/1이라 뮤텍스처럼 쓰이기도 한다.

### 3. 자주 나오는 포인트

- `Semaphore`는 풀을 관리하는 문제(예: 커넥션 풀)에 자주 나온다.
- 잘못 쓰면 `deadlock`, `starvation` 같은 문제가 생긴다.

### 4. 한 번에 구분하는 감각

> `Mutex`는 배타적 진입, `Semaphore`는 허용 개수 조절이다.

- mutex: "한 번에 하나"
- semaphore: "최대 N개"

### 5. 실전 포인트

> 공유 자원을 보호할지, 자원 사용량을 제한할지 먼저 생각하면 선택이 쉬워진다.

- 임계 구역 보호라면 mutex가 기본이다.
- 작업자 수 제한, 연결 수 제한, 생산자/소비자 제어라면 semaphore가 자연스럽다.

## 트레이드오프

해당 없음

## 실무 경험

해당 없음 (관련 실무 내용은 특징/상세 5장 "실전 포인트"에 포함)

## 참고

해당 없음 — 원본 TIL 노트에 출처가 명시되어 있지 않음.

## 관련 내용

- [Deadlock-Conditions](Deadlock-Conditions.md)
- [RaceCondition-DataRace-CriticalSection](RaceCondition-DataRace-CriticalSection.md)
- [Atomic](Atomic.md)
- [../jpa/동시성-제어와-락-전략](../jpa/동시성-제어와-락-전략.md)
