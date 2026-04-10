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
