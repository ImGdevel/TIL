# SSE vs WebSocket 차이 정리

> “실시간 통신을 해야 하는데, SSE로 할까 WebSocket으로 할까?”  
> 이 문서는 두 기술의 개념과 특징, 언제 어떤 것을 선택할지 정리한다.

---

## 1. 개념 한 줄 정리

- **SSE(Server-Sent Events)**  
  - 서버 → 클라이언트 **단방향 스트리밍**  
  - 클라이언트는 HTTP로 한 번 연결한 뒤, 서버가 이벤트를 계속 푸시

- **WebSocket**  
  - 서버 ↔ 클라이언트 **양방향 통신**  
  - 최초에 HTTP로 핸드셰이크 후, **별도의 WebSocket 프로토콜**로 업그레이드

---

## 2. 통신 방향과 사용 목적

- **SSE**
  - 방향: 서버 → 클라이언트 (단방향)
  - 목적: 서버가 일방적으로 “변경/이벤트를 계속 보내야 하는” 경우
  - 예: 알림 스트림, 실시간 로그/모니터링, 주기적 상태 업데이트

- **WebSocket**
  - 방향: 서버 ↔ 클라이언트 (양방향, 풀 듀플렉스)
  - 목적: **서로가 자유롭게 메시지를 주고받아야 하는** 경우
  - 예: 채팅, 협업 편집(문서/화이트보드), 게임, 실시간 제어

---

## 3. 프로토콜과 브라우저 지원

- **SSE**
  - 프로토콜: 순수 HTTP (텍스트 기반 스트리밍)
  - 브라우저 API: `EventSource`
  - 지원: 대부분의 현대 브라우저에서 지원,  
    다만 오래된 IE 계열은 별도 폴리필 필요

- **WebSocket**
  - 프로토콜: `ws://` 또는 `wss://` (HTTP 업그레이드 이후 별도 프로토콜)
  - 브라우저 API: `WebSocket`
  - 지원: 주요 브라우저/플랫폼에서 광범위 지원

---

## 4. 연결 방식과 네트워크 특성

- **SSE**
  - HTTP 단일 방향 연결을 유지하며, 서버가 텍스트 이벤트를 지속적으로 전송
  - 기본적으로 **자동 재연결/Last-Event-ID 지원** (브라우저 `EventSource` 구현)
  - 프록시/로드밸런서, HTTP 기반 인프라에서 비교적 다루기 쉬움

- **WebSocket**
  - 초기에는 HTTP로 핸드셰이크 → 이후 WebSocket 프로토콜로 업그레이드
  - 단일 TCP 연결 위에서 양방향 데이터 프레임 전송 (텍스트/바이너리 모두 가능)
  - 일부 프록시/네트워크 장비에서 WebSocket에 대한 별도 설정이 필요할 수 있음

---

## 5. 서버 구현 관점 (Spring 기준)

- **SSE (Spring MVC)**
  - `SseEmitter`, `Flux`(WebFlux) 등을 사용해 쉽게 구현
  - HTTP/REST 기반 코드와 자연스럽게 섞임
  - 예: `GET /notifications/stream` 엔드포인트 하나로 스트리밍 제공

- **WebSocket**
  - Spring WebSocket / STOMP / RSocket 등 별도 설정 필요
  - 핸드셰이크, 메시지 브로커, 세션 관리 등 고려할 점이 더 많음
  - 예: `/ws` 엔드포인트에 WebSocket 핸드셰이크, 주제(topic)/큐(queue) 구독 등

---

## 6. Spring 코드 예제

### 6.1 SSE 예제 (Spring MVC + SseEmitter)

**서버 – SSE 컨트롤러**

```java
@RestController
public class NotificationController {

    private final List<SseEmitter> emitters = new CopyOnWriteArrayList<>();

    @GetMapping("/notifications/stream")
    public SseEmitter stream() {
        SseEmitter emitter = new SseEmitter(0L); // 타임아웃 무제한
        emitters.add(emitter);

        emitter.onCompletion(() -> emitters.remove(emitter));
        emitter.onTimeout(() -> emitters.remove(emitter));

        // 초기 연결 응답
        try {
            emitter.send(SseEmitter.event()
                    .name("INIT")
                    .data("connected"));
        } catch (IOException e) {
            emitters.remove(emitter);
        }

        return emitter;
    }

    // 알림 발생 시 호출하는 메서드 (예: 서비스에서)
    public void notifyAllClients(String message) {
        for (SseEmitter emitter : emitters) {
            try {
                emitter.send(SseEmitter.event()
                        .name("NOTIFICATION")
                        .data(message));
            } catch (IOException e) {
                emitter.complete();
                emitters.remove(emitter);
            }
        }
    }
}
```

**클라이언트 – 브라우저 JavaScript**

```js
const eventSource = new EventSource("/notifications/stream");

eventSource.addEventListener("INIT", (event) => {
  console.log("연결됨:", event.data);
});

eventSource.addEventListener("NOTIFICATION", (event) => {
  console.log("알림:", event.data);
});

eventSource.onerror = (error) => {
  console.error("SSE 에러", error);
};
```

---

### 6.2 WebSocket 예제 (Spring WebSocket + STOMP)

**서버 – WebSocket/STOMP 설정**

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*")
                .withSockJS(); // 옵션: SockJS 폴리필
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic"); // 구독용
        registry.setApplicationDestinationPrefixes("/app"); // 발신용
    }
}
```

**서버 – 메시지 핸들러**

```java
@Controller
public class ChatController {

    @MessageMapping("/chat/send")      // 클라이언트 → /app/chat/send
    @SendTo("/topic/chat")             // 서버 → 구독자들
    public ChatMessage send(ChatMessage message) {
        // 필요 시 비즈니스 로직 처리
        return message;
    }
}
```

**클라이언트 – 브라우저 JavaScript (STOMP + SockJS)**

```js
const socket = new SockJS("/ws");
const stompClient = Stomp.over(socket);

stompClient.connect({}, () => {
  // 구독
  stompClient.subscribe("/topic/chat", (message) => {
    const body = JSON.parse(message.body);
    console.log("수신:", body);
  });

  // 전송
  stompClient.send(
    "/app/chat/send",
    {},
    JSON.stringify({ sender: "me", content: "hello" })
  );
});
```

이 예제처럼, **SSE는 HTTP GET 엔드포인트 + `EventSource`** 조합으로 단방향 스트림을 제공하고,  
**WebSocket/STOMP는 `/ws` 엔드포인트 + pub/sub(topic) 구조**로 완전한 양방향 메세징을 구현한다.

---

## 6. 장단점 비교

### 6.1 SSE 장단점

**장점**
- HTTP 기반이라 **구현과 인프라 설정이 단순**
- 서버 → 클라이언트 스트리밍에 최적화 (알림/이벤트 스트림)
- 브라우저 `EventSource`가 자동 재연결, `Last-Event-ID` 등 지원

**단점**
- 기본적으로 **단방향(서버 → 클라이언트)**  
  클라이언트 → 서버는 별도 HTTP 요청 필요
- 바이너리 데이터 직접 전송 불가 (텍스트 기반)
- 연결 수가 많거나, 아주 낮은 레이턴시가 필요한 경우에는 제약이 있을 수 있음

### 6.2 WebSocket 장단점

**장점**
- **완전한 양방향 통신** (채팅/게임/협업 도구 등에 적합)
- 텍스트/바이너리 모두 지원
- 메시지 기반 프로토콜(STOMP 등)을 얹으면 pub/sub 모델 구축 가능

**단점**
- 설정과 인프라가 상대적으로 복잡
- HTTP 인프라와는 다른 특성(WebSocket 지원 프록시/로드밸런서 필요) 고려
- 서버/클라이언트 세션 관리, 스케일아웃 시 브로커/공유 저장소 설계 필요

---

## 7. 언제 무엇을 선택할까?

### SSE가 적합한 경우

- 서버가 “알려줄 게 있을 때만” 일방적으로 푸시하면 되는 경우
  - 예: 알림, 이벤트 스트림, 실시간 대시보드, 로그 모니터링
- 기존 HTTP 기반 인프라(Spring MVC, REST API) 위에 **간단히 실시간 알림을 얹고 싶을 때**
- 클라이언트에서 서버로 보내는 요청은 기존 REST API로 충분할 때

### WebSocket이 적합한 경우

- 클라이언트 ↔ 서버 간에 **양방향 상호작용**이 중요한 경우
  - 예: 채팅, 주식 호가창, 실시간 협업 에디터, 게임, 원격 제어
- 낮은 레이턴시와 지속적인 상호 통신이 핵심인 서비스
- 메시지 브로커 기반 pub/sub 구조까지 고려해야 하는 복잡한 실시간 시스템

---

## 8. 요약

- **SSE = 서버 → 클라이언트 단방향 스트리밍, HTTP 기반, 간단한 실시간 알림/이벤트에 적합**
- **WebSocket = 서버 ↔ 클라이언트 양방향 통신, 보다 복잡한 실시간 상호작용에 적합**

실무에서는,

- “알림/모니터링/이벤트 스트림” 수준이면 **SSE부터 고려**하고,
- “채팅/게임/협업 도구”처럼 상호작용이 강한 서비스라면 **WebSocket**을 선택하는 것이 자연스럽다.
