# Deadlock 조건

> 태그: `#os` `#concurrency` `#deadlock`<br>
> 작성일: 2026-06-25<br>
> 최종 수정일: 2026-06-25

## 정의

`Deadlock`은 서로가 가진 자원을 기다리면서 영원히 진행하지 못하는 상태이고, 4가지 조건이 동시에 성립할 때 발생한다.

## 특징 / 상세

핵심 키워드: `mutual exclusion`, `hold and wait`, `no preemption`, `circular wait`

대표적으로 락을 잡은 상태에서 다른 락을 기다리는 형태로 나타난다.

### 1. 4가지 필요 조건 (Coffman)

| 조건 | 의미 |
| --- | --- |
| Mutual Exclusion | 자원을 동시에 공유할 수 없다 |
| Hold and Wait | 자원을 가진 채로 다른 자원을 기다린다 |
| No Preemption | 강제로 빼앗을 수 없다 |
| Circular Wait | 대기 사이클이 존재한다 |

### 2. 예방/회피 핵심

> 4가지 중 하나라도 깨면 deadlock을 막을 수 있다.

- 자원 순서 부여로 `circular wait` 제거
- 타임아웃/재시도로 `hold and wait` 완화
- 선점 가능 자원 설계로 `no preemption` 완화

### 3. 왜 어려운가

> 교착은 "에러"보다 "멈춤"에 가깝게 나타나서 더 늦게 발견된다.

- 재현 조건이 민감하다.
- 실서비스에서는 일부 요청만 멈추는 형태로 보일 수 있다.

### 4. 실전 포인트

> 락 순서를 통일하고, 오래 잡는 자원을 줄이는 것이 가장 기본적인 대응이다.

- 한 번에 여러 락을 잡는 구조를 최소화한다.
- 필요하면 try-lock, timeout, 작업 분리로 위험을 낮춘다.

## 트레이드오프

해당 없음. 다만 1장 "4가지 필요 조건" 표가 예방 전략 선택의 기준표 역할을 한다 — 어떤 조건을 깨는 전략을 쓸지에 따라 구현 복잡도와 성능 트레이드오프가 달라진다.

## 실무 경험

해당 없음 (관련 실무 내용은 특징/상세 4장 "실전 포인트"에 포함)

## 참고

해당 없음 — 원본 TIL 노트에 출처가 명시되어 있지 않음. 검증 시 1차 출처(예: OS 교과서의 Coffman conditions 챕터) 보강 권장.

## 관련 내용

- [Mutex-Semaphore](Mutex-Semaphore.md)
- [RaceCondition-DataRace-CriticalSection](RaceCondition-DataRace-CriticalSection.md)
- [../jpa/동시성-제어와-락-전략](../jpa/동시성-제어와-락-전략.md)
