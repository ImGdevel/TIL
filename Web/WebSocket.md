# WebSocket 정리 – 언제, 어떻게 써야 할까

> 연결된 정리본:
> - [WebSocket과 SSE](../../../TIL.wiki/websocket-and-sse.md)


> “실시간 양방향 통신이 필요할 때, WebSocket을 어디까지 믿고 쓸 것인가?”

---

## 1. WebSocket 한 줄 개념 정리

- **WebSocket**
  - 브라우저와 서버 사이에 **하나의 TCP 연결 위에서 양방향(full-duplex) 통신**을 가능하게 해주는 프로토콜
  - 처음에는 HTTP로 핸드셰이크를 하고, 그 이후에는 WebSocket 프로토콜로 업그레이드하여 동작

특징:

- 서버 ↔ 클라이언트 모두 임의 시점에 메시지를 보낼 수 있다.
- 헤더 오버헤드가 적어, 짧은 메시지를 자주 주고받는 시나리오에서 효율적이다.
- 텍스트/바이너리 프레임 모두 지원한다.

---

## 2. 언제 WebSocket이 필요한가?

다음과 같은 요구사항이 있을 때 WebSocket을 먼저 떠올리게 된다.

1. **양방향 실시간 상호작용**
   - 예: 채팅, 협업 편집(문서/화이트보드), 게임, 실시간 제어 패널

2. **서버에서 클라이언트로 푸시해야 하는 데이터가 많을 때**
   - 예: 다수 클라이언트에게 잦은 상태 업데이트, 멀티 플레이 게임 상태 전파

3. **HTTP 요청/응답 모델로는 오버헤드가 너무 큰 경우**
   - 짧은 메시지를 빠르게 주고받아야 하는데, 매번 HTTP 요청을 날리는 건 비효율적인 상황

반대로,

- 단순한 서버 → 클라이언트 알림/스트리밍 정도라면 **SSE(Server-Sent Events)**가 더 단순하고 적합할 수 있다. (`SSE-vs-WebSocket.md` 참고)

---

## 3. 프로토콜 흐름 간단히 보기

1. **HTTP 핸드셰이크**
   - 클라이언트가 `Upgrade: websocket` 헤더와 함께 HTTP 요청을 보낸다.
   - 서버가 이를 수락하면, 응답에서 프로토콜을 WebSocket으로 업그레이드한다.

2. **연결 유지**
   - 연결이 열린 상태에서, 양쪽 모두 언제든지 메시지 프레임을 보낼 수 있다.
   - Ping/Pong 프레임으로 연결 상태를 확인할 수 있다.

3. **종료**
   - 어느 한쪽이 Close 프레임을 보내면서 연결을 종료한다.

HTTP와 달리,

- 요청/응답 쌍이 아니라 **연결 하나 안에서 여러 메시지를 주고받는 구조**라는 점이 가장 큰 차이다.

---

## 4. 브라우저에서 WebSocket 사용 예시

기본 JavaScript API는 단순하다.

```js
const socket = new WebSocket("wss://example.com/ws");

socket.onopen = () => {
  console.log("연결 열림");
  socket.send(JSON.stringify({ type: "JOIN", roomId: "room-1" }));
};

socket.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log("메시지 수신:", data);
};

socket.onclose = (event) => {
  console.log("연결 종료", event.code, event.reason);
};

socket.onerror = (error) => {
  console.error("WebSocket 에러", error);
};
```

실제로는,

- 재연결 로직
- 하트비트(Ping/Pong) 처리
- 메시지 타입/버전 관리

까지 넣어야 실서비스에서 안정적으로 돌아간다.

---

## 5. Spring에서 WebSocket 사용하기 (STOMP 기반)

Spring에서는 WebSocket + STOMP 조합을 많이 쓴다.

- WebSocket: 소켓 레벨 프로토콜
- STOMP: 그 위에서 동작하는 **메시징 프로토콜** (topic/queue, subscribe/send 등 추상화)

### 5-1. 설정

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*")
                .withSockJS(); // 필요 시: SockJS 폴리필
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic", "/queue"); // 서버 → 클라이언트
        registry.setApplicationDestinationPrefixes("/app"); // 클라이언트 → 서버
    }
}
```

### 5-2. 메시지 핸들러

```java
@Controller
public class ChatController {

    @MessageMapping("/chat.send")      // 클라이언트 → /app/chat.send
    @SendTo("/topic/chat")             // 서버 → /topic/chat 구독자들
    public ChatMessage send(ChatMessage message) {
        // 비즈니스 로직/검증 등
        return message;
    }
}
```

### 5-3. 클라이언트(STOMP + SockJS) 예시

```js
const socket = new SockJS("/ws");
const stompClient = Stomp.over(socket);

stompClient.connect({}, () => {
  // 구독
  stompClient.subscribe("/topic/chat", (message) => {
    const body = JSON.parse(message.body);
    console.log("수신:", body);
  });

  // 발송
  stompClient.send("/app/chat.send", {}, JSON.stringify({ content: "Hello" }));
});
```

---

## 6. WebSocket의 장점

1. **진짜 양방향 실시간 통신**
   - 서버/클라이언트 모두 필요할 때 아무 때나 메시지 전송 가능

2. **낮은 오버헤드**
   - HTTP 요청/응답 반복에 비해 헤더 오버헤드가 적다.
   - 짧은 메시지를 자주 주고받는 환경에서 효율적.

3. **텍스트/바이너리 모두 지원**
   - 채팅 메시지는 텍스트, 게임 상태/이미지 조각은 바이너리 등 혼용 가능

4. **표준화 + 브라우저/플랫폼 지원**
   - 네이티브 브라우저 API, 다양한 서버 라이브러리(Spring, Node.js, etc.) 지원

---

## 7. WebSocket의 단점·주의사항

1. **연결 수 관리**
   - 각 클라이언트와 **지속 연결**을 유지해야 한다.  
   - 많은 동시 접속을 처리하려면, 서버당 동시 연결 수를 충분히 감당할 수 있는 구조/인프라가 필요.

2. **상태 관리**
   - 연결 상태(연결/해제/재연결)를 직접 추적해야 한다.
   - 로드 밸런서 뒤에서 여러 서버를 사용할 때, 세션 스티키니스/세션 공유/메시지 브로커 연동 등을 고민해야 한다.

3. **프록시/네트워크 환경**
   - 일부 방화벽/프록시/회사 네트워크는 WebSocket 트래픽을 제한하거나 타임아웃을 짧게 잡기도 한다.
   - 이 경우, SSE/롱 폴링 등 fallback 전략이 필요할 수 있다.

4. **보안**
   - `ws://` 대신 `wss://`(TLS) 사용이 기본 전제  
   - 인증/인가를 어떻게 유지할지 (초기 토큰 검증 후 세션 유지, 주기적 재검증 등) 설계가 필요하다.

5. **백엔드 확장**
   - 단일 서버에서는 비교적 단순하지만,  
   - 여러 서버/인스턴스로 수평 확장하려면:
     - 중앙 메시지 브로커(Redis Pub/Sub, Kafka 등)  
     - 세션 정보 공유(예: Redis, 외부 세션 스토어)
     를 엮어야 한다.

---

## 8. WebSocket vs SSE 간단 비교 (요약)

자세한 내용은 `SSE-vs-WebSocket.md`에 있지만, 짧게 정리하면:

- **WebSocket**
  - 양방향, 텍스트/바이너리, 고빈도 상호작용
  - 채팅/게임/협업 등 “서로 말이 많은” 상황에 적합

- **SSE**
  - 서버 → 클라이언트 단방향, 텍스트 스트리밍
  - 알림, 로그 스트림, 간단한 상태 업데이트 등 “서버가 말이 많은” 상황에 적합

> 클라이언트가 서버로 자주 말을 걸어야 한다면 WebSocket,  
> 서버가 클라이언트에게 일방적으로 계속 알려주기만 해도 된다면 SSE를 먼저 고려한다.

---

## 9. 마무리 – “실시간” 요구를 구체적인 케이스로 바꾸기

WebSocket을 도입할지 고민할 때는,

> “실시간이 필요하다”는 말 대신  
> “누가, 언제, 얼마나 자주, 얼마나 많은 데이터를 주고받아야 하는가?”

를 구체적으로 적어보는 게 도움이 된다.

그 질문에 답하다 보면,

- 단순 폴링/SSE로 충분한 케이스와  
- WebSocket이 있어야만 자연스럽게 풀리는 케이스가 나뉜다.

그 위에서,

- 연결 수/확장성/보안/운영 난이도까지 함께 고려해  
WebSocket이 “정말 필요한 곳에만” 쓰도록 제한하는 것이 실무적으로 더 안정적이라는 점을 마지막으로 남겨두고 싶다.

