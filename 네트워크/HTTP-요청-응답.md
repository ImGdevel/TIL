# HTTP Request / Response

> 태그: `#network` `#http`<br>
> 작성일: 2026-06-23<br>
> 최종 수정일: 2026-06-23

## 정의

HTTP 통신은 클라이언트가 보내는 Request(메서드/URI/헤더/바디)와 서버가 돌려주는 Response(상태코드/헤더/바디)로 구성되며, 이 구조 자체가 HTTP가 "요청-응답" 기반 프로토콜임을 규정한다.

## 특징 / 상세

### Request 구조

```
GET /users/1 HTTP/1.1
Host: example.com
Authorization: Bearer xxx

(body, 필요한 경우)
```

- Method: 어떤 동작인지 (`GET`, `POST`, `PUT`, `DELETE` 등)
- URI: 어떤 리소스인지
- Header: 메타데이터 (인증, 컨텐츠 타입 등)
- Body: 실제 전송 데이터(필요한 메서드에서)

`Content-Type`은 지금 보내는 바디의 타입을 의미하고, `Accept`는 클라이언트가 응답으로 받고 싶은 타입을 의미한다. 둘은 방향이 반대다 — `Content-Type`은 "내가 보내는 것", `Accept`는 "내가 받고 싶은 것".

### Response 구조

```
HTTP/1.1 200 OK
Content-Type: application/json

{"id": 1, "name": "..."}
```

- Status Code: 결과 분류 (`2xx` 성공, `4xx` 클라이언트 오류, `5xx` 서버 오류 등)
- Header: 메타데이터
- Body: 실제 응답 데이터

### 자주 나오는 포인트

- 상태코드는 "분류"가 중요하다. 정확한 의미를 모른 채 200/4xx만 구분하면 디버깅이 어려워진다.
- 헤더는 캐싱, 인증, 컨텐츠 타입 등 실제 동작에 영향을 준다.

### 실전 포인트

문제를 볼 때는 Request/Response를 한 쌍으로 본다.

- 같은 4xx라도 헤더/바디 내용에 따라 원인이 다르다.
- 클라이언트가 보낸 것과 서버가 받은 것이 다르면(프록시, 게이트웨이 등) 그 경로도 같이 본다.

## 트레이드오프

해당 없음

## 실무 경험

해당 없음

## 참고

원본 학습 노트(TIL)에 인용 링크가 없음.

## 관련 내용

- [HTTP-무상태성](HTTP-무상태성.md)
- [REST](REST.md)
- [멱등성](멱등성.md)
