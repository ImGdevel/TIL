# [네트워크] HTTP 요청/응답 구조

## 한 줄 요약

> <u>HTTP 메시지는 `start-line`, `headers`, `body`로 구성되고, 요청은 메서드/경로, 응답은 상태 코드로 시작한다.</u>

헤더는 메타데이터를 전달하고, 바디는 실제 데이터(예: JSON)를 담는다.

핵심 키워드: `request line`, `status line`, `header`, `body`, `Content-Type`

<br />
<br />

---

## 1. Request

- Start-line: `METHOD path HTTP/version`
- Headers
- Body (선택)

<br />
<br />

---

## 2. Response

- Status-line: `HTTP/version status-code reason`
- Headers
- Body (선택)

<br />
<br />

---

## 3. 자주 나오는 포인트

- `Content-Type`은 바디의 타입을 의미한다.
- `Accept`는 클라이언트가 받고 싶은 타입을 의미한다.

<br /><br />

---

## 4. 실전 포인트

> <u>헤더는 “부가 정보”, 바디는 “실제 데이터”라고 구분하면 흐름이 편하다.</u>

- 인증 정보도 보통 헤더로 보낸다.
- 캐시/압축/인코딩 같은 정보도 헤더에 실린다.
- 요청/응답 구조를 알면 디버깅할 때 어디를 봐야 하는지 빨라진다.
