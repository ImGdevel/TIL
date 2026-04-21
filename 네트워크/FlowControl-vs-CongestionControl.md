# [네트워크] Flow Control vs Congestion Control

## 한 줄 요약

> <u>`Flow Control`은 “수신자 버퍼 보호”, `Congestion Control`은 “네트워크 혼잡 완화”를 목표로 전송량을 조절한다.</u>

둘 다 전송 속도를 낮출 수 있지만, 무엇을 보호하는지(수신자 vs 네트워크)가 다르다.

핵심 키워드: `rwnd`, `cwnd`, `window`, `loss`, `RTT`

<br /><br />

---

## 1. Flow Control(흐름 제어)

> <u>수신자가 처리 가능한 범위를 넘지 않게 한다.</u>

- 기준: 수신자의 버퍼/처리 속도
- 신호: 수신 윈도우(Receive Window, `rwnd`) 광고
- 결과: 송신자는 `rwnd` 범위 내에서만 보낸다.

<br /><br />

---

## 2. Congestion Control(혼잡 제어)

> <u>네트워크가 감당할 수 있는 범위를 넘지 않게 한다.</u>

- 기준: 네트워크 혼잡(큐 적체, 손실, RTT 증가 등)
- 개념: 혼잡 윈도우(Congestion Window, `cwnd`)를 조절
- 결과: 송신자는 대체로 `min(rwnd, cwnd)`만큼 전송한다.

<br /><br />

---

## 3. 자주 헷갈리는 포인트

- Flow control은 “수신자”를 보호한다.
- Congestion control은 “네트워크 전체(경로)”를 보호한다.
- 둘 중 하나만 있어도 문제가 생긴다. (버퍼 오버플로우 or 네트워크 붕괴)

<br /><br />

---

## 4. 실전 포인트

> <u>보낼 수 있는 양은 보통 `rwnd`와 `cwnd` 중 작은 쪽으로 제한된다.</u>

- 수신자가 느리면 flow control이 먼저 걸린다.
- 네트워크가 막히면 congestion control이 먼저 줄어든다.

<br /><br />

---

## 5. 왜 중요한가

> <u>속도 문제라고 해서 전부 같은 원인은 아니다.</u>

- 애플리케이션이 느린지, 수신자가 느린지, 네트워크가 막힌지 구분해야 한다.
- 이 구분이 안 되면 튜닝 방향이 엇갈린다.

