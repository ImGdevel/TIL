# CORS: 브라우저 보안과 현대 웹 아키텍처의 균형점

> 동일 출처 정책(SOP)을 안전하게 완화하여, 서로 다른 출처 간 리소스 공유를 가능하게 하는 HTTP 헤더 기반 보안 메커니즘

## 목차

- [CORS란 무엇인가](#cors란-무엇인가)
- [동일 출처 정책의 이해](#동일-출처-정책의-이해)
- [CORS가 필요한 이유](#cors가-필요한-이유)
- [CORS의 동작 원리](#cors의-동작-원리)
- [Spring에서의 CORS 설정](#spring에서의-cors-설정)
- [실무에서 주의할 점](#실무에서-주의할-점)
- [정리](#정리)
- [참고자료](#참고자료)

---

## CORS란 무엇인가

CORS(Cross-Origin Resource Sharing)는 웹 브라우저에서 실행되는 스크립트가 다른 출처(Origin)의 리소스에 접근할 수 있도록, HTTP 헤더를 통해 권한을 부여하는 보안 메커니즘입니다.

현대 웹 개발에서 프론트엔드 애플리케이션(`https://app.example.com`)이 백엔드 API 서버(`https://api.example.com`)와 통신하는 것은 매우 일반적입니다. 하지만 이 두 서버는 엄연히 다른 출처이기 때문에, 브라우저의 보안 정책에 의해 기본적으로 차단됩니다.

CORS는 이러한 제약을 서버의 명시적 허가를 통해 안전하게 우회할 수 있도록 만들어진 W3C 표준입니다.

---

## 동일 출처 정책의 이해

CORS를 이해하기 위해서는 먼저 브라우저의 핵심 보안 모델인 동일 출처 정책(Same-Origin Policy, SOP)을 알아야 합니다.

### 동일 출처 정책이란?

동일 출처 정책은 하나의 출처에서 로드된 문서나 스크립트가 다른 출처의 리소스와 상호작용하는 것을 제한하는 브라우저의 보안 메커니즘입니다.

여기서 '출처(Origin)'는 다음 세 가지 요소의 조합으로 정의됩니다.

- **프로토콜(Protocol)**: `http`, `https`
- **호스트(Host)**: `example.com`, `api.example.com`
- **포트(Port)**: `80`, `443`, `8080`

이 세 가지가 **모두 동일**해야 같은 출처로 간주됩니다. 하나라도 다르면 다른 출처입니다.

### 출처 비교 예시

기준 URL을 `https://example.com`으로 했을 때, 다음과 같이 판단됩니다.

| URL | 프로토콜 | 호스트 | 포트 | 동일 출처? | 사유 |
|-----|---------|--------|------|-----------|------|
| `https://example.com/api/users` | `https` | `example.com` | `443` | ✅ | 모든 요소 동일 |
| `http://example.com` | `http` | `example.com` | `80` | ❌ | 프로토콜 다름 |
| `https://api.example.com` | `https` | `api.example.com` | `443` | ❌ | 호스트 다름 |
| `https://example.com:8080` | `https` | `example.com` | `8080` | ❌ | 포트 다름 |

### 왜 이런 제약이 필요한가?

동일 출처 정책이 없다면, 악의적인 웹사이트가 사용자가 로그인한 은행 사이트의 API를 무단으로 호출하여 개인정보를 탈취하거나 송금 요청을 보낼 수 있습니다.

예를 들어, 사용자가 `https://mybank.com`에 로그인한 상태에서 악성 사이트 `https://evil.com`을 방문했다고 가정합니다. 만약 SOP가 없다면, `evil.com`의 자바스크립트 코드가 사용자의 쿠키를 이용해 `mybank.com`의 API를 호출하여 계좌 정보를 빼낼 수 있습니다.

SOP는 이런 공격을 원천적으로 차단하는 강력한 방어벽입니다.

---

## CORS가 필요한 이유

동일 출처 정책은 강력한 보안 장치이지만, 현대 웹 애플리케이션 아키텍처와는 충돌합니다.

### 현대 웹의 구조적 변화

과거의 웹은 서버에서 HTML을 모두 렌더링하여 제공하는 전통적인 방식이었습니다. 따라서 대부분의 리소스 요청이 동일 출처 내에서 이루어졌고, SOP가 큰 걸림돌이 되지 않았습니다.

하지만 오늘날의 웹 애플리케이션은 다음과 같은 구조를 갖습니다.

**1. SPA(Single Page Application) 아키텍처**

React, Vue, Angular 같은 프레임워크로 개발된 프론트엔드는 정적 파일 서버에 배포되고, 데이터는 별도의 API 서버와 비동기 통신으로 주고받습니다.

```
프론트엔드: https://app.example.com (Nginx, S3, Vercel 등)
백엔드 API: https://api.example.com (Spring Boot, Node.js 등)
```

이 둘은 명백히 다른 출처이므로, CORS 설정 없이는 통신이 불가능합니다.

**2. 마이크로서비스 아키텍처(MSA)**

기능별로 여러 개의 독립적인 API 서버를 운영하는 구조에서, 하나의 프론트엔드가 여러 서비스와 통신해야 합니다.

```
인증 서비스: https://auth.example.com
주문 서비스: https://order.example.com
결제 서비스: https://payment.example.com
```

**3. 외부 API 연동**

지도, 날씨, 소셜 로그인 등 외부 제공 API를 클라이언트에서 직접 호출하는 경우도 많습니다.

```javascript
// 클라이언트에서 외부 API 직접 호출
fetch('https://maps.googleapis.com/maps/api/geocode/json?address=Seoul')
```

### CORS의 역할

CORS는 이러한 현대적 아키텍처가 SOP라는 보안 제약 속에서도 안전하게 동작할 수 있도록 표준화된 절차를 제공합니다.

서버가 특정 HTTP 응답 헤더를 통해 "이 출처는 내 리소스에 접근해도 좋다"고 명시적으로 허락하면, 브라우저는 이를 확인하고 요청을 허용합니다. 즉, 보안 정책의 주체는 브라우저에 두되, 허용 여부의 결정권은 서버가 갖는 구조입니다.

만약 CORS가 없었다면, 개발자들은 JSONP와 같은 비표준적이고 보안에 취약한 우회 기법을 사용해야 했을 것입니다.

---

## CORS의 동작 원리

CORS 요청은 크게 **단순 요청(Simple Request)**과 **예비 요청(Preflight Request)** 두 가지 유형으로 나뉩니다. 브라우저는 요청의 특성을 판단하여 자동으로 적절한 방식을 선택합니다.

### 1. 단순 요청 (Simple Request)

다음 조건을 **모두** 만족하는 요청은 단순 요청으로 분류되어, 예비 요청 없이 바로 본 요청을 보냅니다.

**조건:**
- HTTP 메서드가 `GET`, `HEAD`, `POST` 중 하나
- 헤더가 `Accept`, `Accept-Language`, `Content-Language`, `Content-Type` 등 기본 헤더만 포함
- `Content-Type` 헤더 값이 `application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain` 중 하나

**동작 흐름:**

```
[1단계] 브라우저 → 서버: 요청
GET /api/users
Origin: https://app.example.com

[2단계] 서버 → 브라우저: 응답
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
Content-Type: application/json

[3단계] 브라우저의 검증
- Origin 헤더와 Access-Control-Allow-Origin 헤더 비교
- 일치하면 → 응답 데이터를 자바스크립트에 전달
- 불일치하면 → CORS 에러 발생, 응답 차단
```

브라우저는 자동으로 요청에 `Origin` 헤더를 추가하고, 서버의 응답 헤더 `Access-Control-Allow-Origin`을 검증하여 요청의 허용 여부를 결정합니다.

### 2. 예비 요청 (Preflight Request)

단순 요청의 조건을 벗어나는 요청은 **실제 요청 전에** `OPTIONS` 메서드를 사용한 예비 요청을 먼저 보냅니다.

다음과 같은 경우 예비 요청이 발생합니다.

- HTTP 메서드가 `PUT`, `DELETE`, `PATCH` 등
- `Content-Type`이 `application/json`
- 커스텀 헤더를 포함 (예: `Authorization`, `X-Custom-Header`)

예비 요청은 "내가 이런 요청을 보낼 건데, 허용 가능한가?"를 서버에 먼저 확인하는 과정입니다.

**동작 흐름:**

```
[1단계] 예비 요청 (Preflight)
브라우저 → 서버:
OPTIONS /api/users
Origin: https://app.example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Content-Type, Authorization

서버 → 브라우저:
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 3600

[2단계] 본 요청 (Main Request)
브라우저 검증 후 허용되면:

브라우저 → 서버:
PUT /api/users/123
Origin: https://app.example.com
Content-Type: application/json
Authorization: Bearer token123

서버 → 브라우저:
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
```

예비 요청의 응답을 보고 브라우저가 허용 여부를 판단합니다. 허용되지 않으면 본 요청은 전송되지 않고 CORS 에러가 발생합니다.

### 주요 CORS 응답 헤더

서버가 CORS 정책을 전달하기 위해 사용하는 주요 응답 헤더는 다음과 같습니다.

| 헤더 | 설명 | 예시 |
|------|------|------|
| `Access-Control-Allow-Origin` | 허용할 출처 | `https://app.example.com` 또는 `*` |
| `Access-Control-Allow-Methods` | 허용할 HTTP 메서드 | `GET, POST, PUT, DELETE` |
| `Access-Control-Allow-Headers` | 허용할 요청 헤더 | `Content-Type, Authorization` |
| `Access-Control-Allow-Credentials` | 자격 증명 허용 여부 | `true` |
| `Access-Control-Max-Age` | 예비 요청 캐시 시간(초) | `3600` |
| `Access-Control-Expose-Headers` | 브라우저에 노출할 헤더 | `X-Custom-Header` |

---

## Spring에서의 CORS 설정

Spring 환경에서 CORS를 설정하는 방법은 크게 두 가지입니다. Spring MVC 레벨과 Spring Security 레벨입니다.

### 1. Spring MVC 레벨: @CrossOrigin 어노테이션

컨트롤러나 메서드에 직접 `@CrossOrigin` 어노테이션을 붙이는 방식입니다.

```java
// [UserController.java]
@RestController
@RequestMapping("/api/users")
@CrossOrigin(origins = "https://app.example.com")
public class UserController {

    @GetMapping
    public List<User> getUsers() {
        return userService.findAll();
    }

    @PostMapping
    @CrossOrigin(origins = {"https://app.example.com", "http://localhost:3000"})
    public User createUser(@RequestBody User user) {
        return userService.save(user);
    }
}
```

이 방식은 간단하지만, 설정이 컨트롤러 곳곳에 분산되고 Spring Security의 필터 체인보다 늦게 적용될 수 있어 일관성 있는 보안 정책 관리가 어렵습니다.

### 2. Spring Security 레벨: SecurityFilterChain 설정 (권장)

Spring Security를 사용하는 환경에서는 `SecurityFilterChain`에 CORS 설정을 통합하는 것이 가장 권장되는 방법입니다. 모든 보안 규칙과 CORS 정책을 한 곳에서 중앙 관리할 수 있습니다.

```java
// [SecurityConfig.java]
package com.example.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

import java.util.List;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .cors(cors -> cors.configurationSource(corsConfigurationSource()));

        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();

        // 허용할 출처 설정
        configuration.setAllowedOrigins(List.of(
            "https://app.example.com",
            "http://localhost:3000"
        ));

        // 허용할 HTTP 메서드 설정
        configuration.setAllowedMethods(List.of(
            "GET", "POST", "PUT", "DELETE", "OPTIONS"
        ));

        // 허용할 요청 헤더 설정
        configuration.setAllowedHeaders(List.of(
            "Authorization",
            "Content-Type",
            "X-Requested-With"
        ));

        // 브라우저에 노출할 응답 헤더 설정
        configuration.setExposedHeaders(List.of(
            "X-Total-Count",
            "X-Page-Number"
        ));

        // 자격 증명(쿠키, Authorization 헤더 등) 포함 허용
        configuration.setAllowCredentials(true);

        // 예비 요청 결과 캐시 시간 (1시간)
        configuration.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        // 모든 경로에 대해 위 정책 적용
        source.registerCorsConfiguration("/**", configuration);

        return source;
    }
}
```

**핵심 포인트:**

1. `http.cors()`를 통해 CORS 필터를 활성화합니다.
2. `CorsConfigurationSource` 빈을 등록하여 세부 정책을 정의합니다.
3. `setAllowedOrigins()`에서 허용할 출처를 명시합니다.
4. `setAllowCredentials(true)`를 설정하면 쿠키와 인증 헤더를 포함한 요청을 허용합니다. 이 경우 `setAllowedOrigins("*")`는 사용할 수 없습니다.
5. `setMaxAge()`로 예비 요청의 결과를 캐시하여, 동일한 요청에 대해 매번 예비 요청을 보내지 않도록 최적화합니다.

### 환경별 설정 분리

실무에서는 개발 환경과 운영 환경의 출처가 다르므로, 설정을 외부화하는 것이 좋습니다.

```yaml
# [application.yml]
cors:
  allowed-origins:
    - https://app.example.com
    - https://admin.example.com
  allowed-methods:
    - GET
    - POST
    - PUT
    - DELETE
  max-age: 3600

# [application-local.yml]
cors:
  allowed-origins:
    - http://localhost:3000
    - http://localhost:8080
```

```java
// [SecurityConfig.java]
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Value("${cors.allowed-origins}")
    private List<String> allowedOrigins;

    @Value("${cors.allowed-methods}")
    private List<String> allowedMethods;

    @Value("${cors.max-age}")
    private Long maxAge;

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(allowedOrigins);
        configuration.setAllowedMethods(allowedMethods);
        configuration.setAllowedHeaders(List.of("*"));
        configuration.setAllowCredentials(true);
        configuration.setMaxAge(maxAge);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);

        return source;
    }
}
```

---

## 실무에서 주의할 점

### 1. `Access-Control-Allow-Origin: *`의 위험성

모든 출처를 허용하는 `*` 설정은 개발 편의성은 높지만, 보안상 매우 위험합니다.

```java
// ❌ 나쁜 예: 모든 출처 허용
configuration.setAllowedOrigins(List.of("*"));
```

악의적인 웹사이트가 여러분의 API를 자유롭게 호출할 수 있게 됩니다. 특히 인증이 필요한 API라면 더욱 위험합니다.

```java
// ✅ 좋은 예: 명시적 출처 지정
configuration.setAllowedOrigins(List.of(
    "https://app.example.com",
    "https://admin.example.com"
));
```

### 2. `AllowCredentials`와 `AllowedOrigins`의 조합

자격 증명(쿠키, Authorization 헤더)을 포함한 요청을 허용하려면 `setAllowCredentials(true)`를 설정해야 합니다. 하지만 이 경우 `setAllowedOrigins("*")`는 사용할 수 없습니다.

```java
// ❌ 에러 발생: Credentials와 * 조합 불가
configuration.setAllowedOrigins(List.of("*"));
configuration.setAllowCredentials(true);
```

브라우저는 보안상의 이유로 이 조합을 거부합니다. 반드시 명시적 출처를 지정해야 합니다.

```java
// ✅ 올바른 설정
configuration.setAllowedOrigins(List.of("https://app.example.com"));
configuration.setAllowCredentials(true);
```

### 3. @CrossOrigin과 SecurityFilterChain의 충돌

`@CrossOrigin` 어노테이션과 Spring Security의 CORS 설정을 동시에 사용하면 예상치 못한 동작이 발생할 수 있습니다.

Spring Security의 `CorsFilter`는 필터 체인의 앞쪽에 위치하므로, 여기서 요청이 거부되면 컨트롤러의 `@CrossOrigin` 설정은 실행되지 않습니다.

**권장사항:**
- Spring Security를 사용한다면 `SecurityFilterChain`에서만 CORS를 설정합니다.
- `@CrossOrigin`은 Spring Security를 사용하지 않거나, 특정 엔드포인트에만 예외적으로 다른 정책을 적용할 때만 사용합니다.

### 4. Preflight 요청과 인증의 관계

예비 요청(`OPTIONS`)은 실제 요청이 아니므로, 인증 헤더를 포함하지 않습니다. 따라서 인증이 필요한 엔드포인트라도 `OPTIONS` 요청은 인증 없이 허용해야 합니다.

```java
// [SecurityConfig.java]
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            // OPTIONS 요청은 인증 없이 허용
            .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()
            .anyRequest().authenticated()
        )
        .cors(cors -> cors.configurationSource(corsConfigurationSource()));

    return http.build();
}
```

만약 `OPTIONS` 요청에 인증을 요구하면, 예비 요청 단계에서 401 에러가 발생하고 본 요청은 전송되지 않습니다.

### 5. 실제 사례: CORS 에러 디버깅

프론트엔드에서 다음과 같은 에러가 발생했다고 가정합니다.

```
Access to fetch at 'https://api.example.com/users' from origin 'https://app.example.com'
has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present
on the requested resource.
```

**체크리스트:**

1. **서버가 응답 헤더를 제대로 보내고 있는가?**
   - 브라우저 개발자 도구의 Network 탭에서 응답 헤더 확인
   - `Access-Control-Allow-Origin` 헤더가 있는지 확인

2. **출처가 정확히 일치하는가?**
   - `https://app.example.com`과 `https://app.example.com/`은 다릅니다 (trailing slash)
   - 대소문자, 프로토콜, 포트까지 정확히 일치해야 합니다

3. **Preflight 요청이 실패했는가?**
   - Network 탭에서 `OPTIONS` 요청의 상태 코드 확인
   - 200이 아니라면 서버의 CORS 설정을 재확인

4. **인증 관련 설정이 올바른가?**
   - `withCredentials: true`를 사용했다면 서버에서 `AllowCredentials: true` 설정 확인
   - `AllowedOrigins`가 `*`가 아닌지 확인

### 6. 성능 최적화: MaxAge 활용

예비 요청은 추가적인 네트워크 왕복을 발생시키므로, 성능에 영향을 줄 수 있습니다. `Access-Control-Max-Age` 헤더로 예비 요청의 결과를 캐시하여 최적화할 수 있습니다.

```java
// 1시간 동안 예비 요청 결과 캐시
configuration.setMaxAge(3600L);
```

동일한 조건의 요청에 대해 1시간 동안은 예비 요청을 다시 보내지 않습니다.

---

## 정리

CORS는 브라우저의 동일 출처 정책(SOP)을 안전하게 완화하여, 서로 다른 출처 간의 리소스 공유를 가능하게 하는 HTTP 헤더 기반의 표준 보안 메커니즘입니다.

현대 웹 애플리케이션은 프론트엔드와 백엔드를 분리하거나, 마이크로서비스 아키텍처를 채택하는 경우가 많아 CORS 설정은 필수적입니다. Spring Security 환경에서는 `SecurityFilterChain`에 CORS 정책을 통합하여 중앙에서 일관되게 관리하는 것이 가장 권장됩니다.

**핵심 원칙:**
- 보안을 위해 `AllowedOrigins`는 명시적으로 지정합니다.
- 자격 증명을 사용한다면 `*` 대신 구체적 출처를 설정합니다.
- `OPTIONS` 요청은 인증 없이 허용합니다.
- 환경별로 설정을 외부화하여 관리합니다.

CORS는 보안과 유연성의 균형을 맞추는 메커니즘입니다. 올바르게 설정하면 현대적인 웹 아키텍처를 안전하게 구현할 수 있습니다.

---

## 참고자료

- [MDN Web Docs - CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [Spring Framework Documentation - CORS Support](https://docs.spring.io/spring-framework/reference/web/webmvc-cors.html)
- [Spring Security Documentation - CORS](https://docs.spring.io/spring-security/reference/servlet/integrations/cors.html)
- [W3C - Cross-Origin Resource Sharing Specification](https://www.w3.org/TR/cors/)
