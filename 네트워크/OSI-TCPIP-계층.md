# OSI 7계층 vs TCP/IP 4계층

> 태그: `#network` `#osi` `#tcp-ip` `#fundamentals`<br>
> 작성일: 2026-06-23<br>
> 최종 수정일: 2026-06-23

## 정의

OSI 7계층은 네트워크 통신을 일곱 단계로 쪼갠 이론적 참조 모델이고, TCP/IP 4계층은 실제 인터넷에서 쓰이는 구현 중심 모델로, OSI의 여러 계층이 TCP/IP의 한 계층에 합쳐져 대응된다.

## 특징 / 상세

### OSI 7계층

| 계층 | 이름 | 역할(요약) |
|---|---|---|
| 7 | Application | 사용자와 가장 가까운 응용 프로토콜 (HTTP 등) |
| 6 | Presentation | 데이터 표현/인코딩/암호화 |
| 5 | Session | 연결(세션) 관리 |
| 4 | Transport | 종단 간 신뢰성/순서 보장 (TCP/UDP) |
| 3 | Network | 경로 선택, IP 주소 |
| 2 | Data Link | 같은 네트워크 내 전달 (MAC) |
| 1 | Physical | 실제 전기/光 신호 |

### TCP/IP 4계층

| 계층 | 대응되는 OSI 계층(대략) |
|---|---|
| Application | Application + Presentation + Session |
| Transport | Transport |
| Internet | Network |
| Link | Data Link + Physical |

### 자주 나오는 포인트

- OSI는 교육/이론용 참조 모델로 더 자주 언급된다.
- 실제 인터넷 스택은 TCP/IP 모델에 가깝게 동작한다.
- "이 프로토콜은 몇 계층이냐"는 질문은 OSI 기준으로 묻는 경우가 많다(예: HTTP=7계층, TCP=4계층, IP=3계층).

## 트레이드오프

해당 없음

## 실무 경험

해당 없음

## 참고

원본 학습 노트(TIL)에 인용 링크가 없음.

## 관련 내용

- [TCP-UDP-비교](TCP-UDP-비교.md)
- [웹-요청-흐름](웹-요청-흐름.md)
