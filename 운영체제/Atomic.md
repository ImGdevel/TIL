# Atomic

> 태그: `#os` `#concurrency` `#atomic`<br>
> 작성일: 2026-06-25<br>
> 최종 수정일: 2026-06-25

## 정의

`Atomic`은 어떤 연산이 "쪼개지지 않고 한 번에 실행된 것처럼" 보이는 성질(원자성)이며, 공유 데이터 경쟁에서 깨지면 `race condition`의 원인이 된다.

## 특징 / 상세

핵심 키워드: `atomicity`, `read-modify-write`, `CAS`, `lock-free`, `critical section`

원자성은 동시성 문제를 줄이는 핵심 도구지만, 원자성만으로 모든 동기화가 해결되지는 않는다.

### 1. Atomic(원자성)이란

> 연산 도중에 다른 스레드가 "중간 상태"를 관찰할 수 없게 만드는 성질이다.

- 단순한 읽기/쓰기도 항상 원자적이라고 가정하면 위험하다. (플랫폼/정렬/타입에 따라 다를 수 있다)
- 특히 `i++` 같은 연산은 보통 "읽기 -> 계산 -> 쓰기"로 쪼개질 수 있다.

### 2. Atomic 연산이 필요한 상황

> 공유 변수를 여러 실행 흐름이 동시에 갱신할 때다.

- 카운터 증가
- 플래그(상태) 전환
- 한 번만 실행(초기화) 같은 패턴

### 3. Mutex와의 관계

> 원자 연산은 "작은 단위"에 강하고, 뮤텍스는 "임계 구역 전체"를 보호한다.

- atomic: 변수 단위의 원자 갱신(CAS 등)에 유리
- mutex: 여러 변수/불변식(invariant)을 함께 지켜야 할 때 유리

### 4. 왜 atomic만으로 부족할 수 있나

> 값 하나는 안전하게 바꿔도, 여러 단계의 규칙은 깨질 수 있다.

- 예를 들어 잔액 차감과 로그 기록, 재고 감소와 주문 생성처럼 여러 상태를 함께 맞춰야 하는 경우는 락이 더 단순하다.
- atomic은 "경합을 줄이는 도구"이지, 복잡한 비즈니스 규칙 전체를 대체하지는 않는다.

### 5. 실전 포인트

> atomic은 빠른 경로에 두고, 복잡한 갱신은 락으로 묶는 식의 혼합 설계가 많다.

- 카운터/플래그/최신 값 같은 단순 상태에 잘 맞는다.
- `CAS` 기반 루프는 재시도가 잦아질 수 있어서, 경합이 심하면 오히려 손해가 날 수 있다.

## 트레이드오프

해당 없음

## 실무 경험

해당 없음 (관련 실무 내용은 특징/상세 5장 "실전 포인트"에 포함)

## 참고

해당 없음 — 원본 TIL 노트에 출처가 명시되어 있지 않음. 검증 시 1차 출처(예: [Java `java.util.concurrent.atomic` 패키지 문서](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/atomic/package-summary.html), OS/동시성 교과서의 동기화 챕터) 보강 권장.

## 관련 내용

- [Mutex-Semaphore](Mutex-Semaphore.md)
- [RaceCondition-DataRace-CriticalSection](RaceCondition-DataRace-CriticalSection.md)
