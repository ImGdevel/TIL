# [CS/Network] TCP 3-way handshake

## 한 줄 요약

> <u>`3-way handshake`는 TCP 연결을 만들 때 양쪽의 초기 시퀀스 번호를 동기화하고 연결 상태를 확정하는 과정이다.</u>

SYN과 ACK를 교환해서 “서로 보낼 준비가 됐다”를 확인한다.

핵심 키워드: `SYN`, `ACK`, `ISN`, `sequence number`, `SYN flood`

<br />
<br />

---

## 1. 흐름

1. Client -> Server: `SYN`
2. Server -> Client: `SYN + ACK`
3. Client -> Server: `ACK`

<br />
<br />

---

## 2. 왜 3번인가

> <u>양쪽이 서로의 초기 시퀀스 번호를 알고, 연결 상태가 일치해야 한다.</u>

- 서버 입장에서도 클라이언트가 응답 가능한 상태인지 확인해야 한다.
- 두 번만 교환하면 한쪽 상태가 애매해질 수 있다.

