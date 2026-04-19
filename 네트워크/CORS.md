# [네트워크] CORS

## 한 줄 요약

> <u>`CORS`는 브라우저의 동일 출처 정책(SOP) 때문에 막히는 교차 출처 요청을, 서버가 응답 헤더로 “허용 범위”를 명시해 풀어주는 규칙이다.</u>

중요한 점은 CORS는 “브라우저 보안 모델”이라는 것이다. 서버-서버 통신에는 보통 적용되지 않는다.

핵심 키워드: `origin`, `same-origin policy`, `preflight`, `OPTIONS`, `credentials`

<br /><br />

---

## 1. Origin과 SOP

> <u>브라우저는 출처가 다른 리소스 접근을 기본적으로 제한한다.</u>

- Origin: `scheme + host + port`
- 제한 대상: 대표적으로 XHR/fetch 기반 요청

<br /><br />

---

## 2. Simple request vs Preflight

> <u>조건이 단순하면 바로 보내고, 복잡하면 먼저 허용 여부를 묻는다.</u>

- preflight: 브라우저가 `OPTIONS`로 “이 요청 해도 되나?”를 확인
- 서버가 허용하면 실제 요청을 보낸다.

<br /><br />

---

## 3. credentials(쿠키/인증정보) 주의

> <u>쿠키를 포함한 교차 출처 요청은 허용 조건이 더 엄격하다.</u>

- 서버는 `Access-Control-Allow-Credentials` 등을 적절히 설정해야 한다.
- 임의의 출처(`*`)와 credentials 조합은 위험하거나 금지되는 경우가 많다.

