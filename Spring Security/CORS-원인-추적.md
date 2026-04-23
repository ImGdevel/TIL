# Spring Security와 CORS, 그리고 필터 체인으로 원인 추적하기

> 연결된 정리본:
> - [CORS 원인 추적](../../../TIL.wiki/spring-security-cors-troubleshooting.md)
> - [CORS](../../../TIL.wiki/spring-security-cors.md)


> “분명 CORS 설정을 했는데도, 왜 여전히 CORS 에러가 날까?”  
> 이 글은 **Spring Security 필터 체인**을 따라가며 CORS 문제가 어디서 터지는지 추적해 보는 기록이다.

---

## 1. CORS는 대체 무엇이고, 왜 Spring Security와 충돌할까?

### 1.1 CORS 한 줄 정의

- **CORS(Cross-Origin Resource Sharing)** 는  
  브라우저가 “다른 출처(origin)의 서버”에 요청을 보낼 때,  
  **서버가 응답 헤더로 ‘이 출처를 허용한다’는 신호를 주는 규약**이다.
- 핵심 헤더:
  - `Access-Control-Allow-Origin`
  - `Access-Control-Allow-Methods`
  - `Access-Control-Allow-Headers`
  - `Access-Control-Allow-Credentials`

### 1.2 Preflight 요청(OPTIONS)과 Spring Security

- 브라우저는 민감한 요청(예: `Authorization` 헤더, `PUT/DELETE` 메서드)을 보내기 전에  
  **먼저 `OPTIONS` 메서드로 Preflight 요청**을 보낸다.
- 이때 서버가 **적절한 CORS 응답 헤더**를 주지 못하면,  
  브라우저는 실제 요청 자체를 보내지 않고 **CORS 에러**를 띄운다.
- 문제는, **Spring Security 필터 체인에서 이 Preflight 요청이 막히거나,  
  CORS 처리가 해당 필터보다 뒤에 있으면** 브라우저까지 헤더가 제대로 가지 않는다는 것.

---

## 2. Spring Security 필터 체인 간단 구조 이해하기

Spring Security는 하나의 서블릿 필터가 아니라, **여러 개의 필터가 체인으로 연결된 구조**다.  
대표적인 필터들(순서 일부 예시):

- `WebAsyncManagerIntegrationFilter`
- `SecurityContextPersistenceFilter`
- `HeaderWriterFilter`
- `CsrfFilter`
- `LogoutFilter`
- `UsernamePasswordAuthenticationFilter`
- `AnonymousAuthenticationFilter`
- `ExceptionTranslationFilter`
- `FilterSecurityInterceptor`

여기에서 **CORS와 직접적으로 연결되는 필터**는 두 가지 레이어로 나눌 수 있다.

1. **Spring Web의 `CorsFilter` (서블릿 필터)**  
2. **Spring Security의 CORS 처리 (`http.cors()` + `CorsConfigurationSource`)**

둘을 어떻게 섞느냐, 그리고 **체인에서 어디에 놓이느냐**에 따라 CORS 문제가 생기기도, 사라지기도 한다.

---

## 3. Spring에서 CORS를 처리하는 두 가지 방식

### 3.1 Spring MVC / Web 레벨의 CORS (`CorsFilter`, `@CrossOrigin`)

- `@CrossOrigin` 애노테이션
- `WebMvcConfigurer`의 `addCorsMappings`
- 직접 등록한 `CorsFilter` 빈

이들은 **Spring Security 바깥 혹은 앞단**에서 동작할 수 있다.  
문제는, Spring Security 필터가 **Preflight 요청을 먼저 잡아버리면 이 레벨의 CORS 처리가 무시**된다는 것.

### 3.2 Spring Security 레벨의 CORS (`http.cors()`)

`SecurityFilterChain` 설정에서 CORS를 활성화하면:

```java
http
    .cors()  // CorsConfigurationSource 기반 CORS 처리
    .and()
    .csrf().disable()
    .authorizeHttpRequests(auth -> auth
        .requestMatchers("/public/**").permitAll()
        .anyRequest().authenticated()
    );
```

- 이때 Spring Security는 내부적으로 `CorsFilter`와 유사한 동작을 하는 필터를 **필터 체인 초반**에 등록한다.
- `CorsConfigurationSource` 또는 전역 CORS 설정 정보를 읽어와,  
  **Preflight 요청을 우선 처리**하고 필요한 CORS 헤더를 추가한다.

**정리:**  
실무에서는 보통 **`http.cors()` + `CorsConfigurationSource` 조합**을 쓰거나,  
Spring MVC 레벨의 CORS 설정을 Spring Security가 읽어가도록 하는 방식으로 통일하는 것이 좋다.

---

## 4. CORS 문제가 Spring Security에서 어떻게 발생하는가?

이제 실제로 **CORS 에러가 발생하는 전형적인 시나리오**를  
Spring Security 필터 체인 관점에서 살펴보자.

### 4.1 시나리오 1 – Preflight 요청이 Security에서 막히는 경우

증상:

- 브라우저 콘솔에 CORS 에러
- 서버 로그에는 `OPTIONS /api/...` 요청이 401/403 등으로 막힌 흔적

흐름:

1. 브라우저가 `OPTIONS /api/...` Preflight 요청 전송
2. 요청이 Spring Security 필터 체인으로 진입
3. `CorsFilter`나 `http.cors()` 설정이 없거나, 필터 순서가 뒤에 있어
   - `CsrfFilter`, `ExceptionTranslationFilter`, `FilterSecurityInterceptor` 등에서 먼저 막힘
4. 결과적으로 **적절한 `Access-Control-*` 헤더 없이 401/403 응답 전송**
5. 브라우저는 “CORS 정책 위반”으로 실제 요청 자체를 보내지 않음

핵심 원인:

- Preflight 요청을 **“보안 검사 전에 CORS로 먼저 처리”**해야 하는데,  
  이 순서가 지켜지지 않아서 발생.

### 4.2 시나리오 2 – CORS 설정은 했는데, Spring Security가 모르는 경우

증상:

- `WebMvcConfigurer#addCorsMappings`로 설정했는데도 계속 CORS 에러.

흐름:

1. `WebMvcConfigurer`에서 `addCorsMappings` 구현
2. 하지만 Security 쪽에서 `http.cors()`를 활성화하지 않음
3. Spring Security 필터 체인은 해당 CORS 설정을 참고하지 않고 **그냥 자신의 규칙으로만 요청을 처리**
4. 따라서 CORS 응답 헤더가 붙지 않거나, Preflight가 막힘

핵심 원인:

- **Web 레벨 CORS 설정과 Security 레벨 CORS 처리가 따로 놀고 있음**

### 4.3 시나리오 3 – 허용 Origin/헤더/메서드가 실제 요청과 맞지 않는 경우

증상:

- 특정 API는 잘 되는데, 다른 API나 일부 메서드는 CORS 에러

흐름:

1. `CorsConfiguration`에서 `allowedOrigins`, `allowedMethods`, `allowedHeaders`를 제한적으로 설정
2. 실제 브라우저 요청의 Origin, 메서드, 헤더가 이 설정과 맞지 않음
3. Spring Security의 CORS 필터가 Preflight를 보고 “허용하지 않는다”고 판단
4. CORS 헤더 없이 403 비슷한 응답 → 브라우저는 CORS 에러

핵심 원인:

- **CORS 설정은 있지만 “요청 스펙”과 정확히 맞지 않음**

---

## 5. 필터 체인을 따라 CORS 문제 원인 추적하기

실제로 CORS 문제가 발생했을 때, 아래 순서로 필터 체인을 따라가면 원인을 찾기 좋다.

### 5.1 1단계: Preflight 요청이 서버까지 오는지 확인

- 브라우저 개발자 도구 → Network 탭에서 `OPTIONS` 요청 존재 여부 확인
- 서버 로그에 `OPTIONS /...`가 찍히는지 확인
- 아예 로그가 없다면:
  - 프록시/게이트웨이, 로드밸런서 레벨에서 막혔을 가능성도 고려

### 5.2 2단계: Spring Security 디버그 로그 켜기

`application.yml` 예시:

```yaml
logging:
  level:
    org.springframework.security: DEBUG
```

- 이렇게 설정하면, 요청이 Spring Security 필터 체인을 통과하는 과정에서  
  어떤 필터가 동작했는지 로그로 확인 가능하다.
- `OPTIONS` 요청에 대해 **어느 필터에서 거절(deny)되었는지** 체크한다.

### 5.3 3단계: `http.cors()` 활성화 여부 확인

Security 설정에서:

```java
http
    .cors() // 이게 없으면 Security는 CORS 설정을 고려하지 않는다
    .and()
    .csrf().disable();
```

- `http.cors()`가 없다면, WebMvc의 CORS 설정만 믿고 있었다는 의미.
- 이 한 줄을 추가하고, Spring이 어떤 CORS 설정을 사용하는지 다시 확인한다.

### 5.4 4단계: `CorsConfigurationSource`와 매핑 확인

예시:

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration configuration = new CorsConfiguration();
    configuration.setAllowedOrigins(List.of("http://localhost:3000"));
    configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    configuration.setAllowedHeaders(List.of("*"));
    configuration.setAllowCredentials(true);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", configuration);
    return source;
}
```

- 실제 프론트엔드 Origin, 메서드, 헤더와 이 설정이 **정확히 맞는지** 확인
- `allowedOrigins` 대신 `allowedOriginPatterns`를 써야 하는 케이스(`*`와 credentials 조합)도 주의

### 5.5 5단계: OPTIONS 요청에 대한 인증/인가 규칙 확인

- `HttpSecurity` 설정에서 `OPTIONS` 요청을 별도로 허용할지 여부를 검토:

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()
    .anyRequest().authenticated()
);
```

- Preflight는 “실제 비즈니스 요청”이 아니므로,  
  보통 인증/인가 체크 없이 통과시키는 편이 안전하다.

---

## 6. 자주 하는 실수 정리

1. **WebMvc에 CORS 설정만 하고, Spring Security에 `http.cors()`를 안 켠 경우**
   - → Security 필터가 CORS를 전혀 모르고 가로막음

2. **직접 등록한 `CorsFilter`의 순서가 Spring Security 앞이 아닌 경우**
   - → Security가 선행해 버려서 CORS 헤더가 붙지 않거나, Preflight 자체가 막힘

3. **허용 Origin/메서드/헤더 설정이 실제 요청과 달라서, CORS 필터가 거부하는 경우**

4. **개발/운영 환경 Origin이 달라졌는데 CORS 설정은 그대로인 경우**
   - 예: 로컬은 `http://localhost:3000`, 운영은 `https://app.example.com`인데  
     CORS 설정에는 로컬만 들어 있는 상황

---

## 7. 정리 – CORS는 결국 “필터 순서와 설정 일치”의 문제

- CORS 자체는 **브라우저와 서버 간의 규약**일 뿐, 복잡한 로직은 아니다.
- 실제 문제는 대부분:
  - Preflight 요청이 **어느 필터에서 먼저 처리되는지(필터 순서)**,
  - Spring Web과 Spring Security 간에 **CORS 설정이 일치하는지**,
  - 실제 요청의 Origin/메서드/헤더와 **CORS 설정이 정확히 맞는지**
  에서 발생한다.

정리하면, CORS 에러가 보일 때는 다음 세 질문을 순서대로 던져보면 좋다.

1. `OPTIONS` Preflight 요청이 **서버까지 도달하는가?**
2. Spring Security 필터 체인에서 **어느 필터가 이 요청을 막고 있는가?**
3. `http.cors()` + `CorsConfigurationSource` (혹은 `CorsFilter`) 설정이  
   **실제 요청 스펙과 정확히 일치하는가?**

이 세 가지를 필터 체인을 따라 하나씩 확인해 나가면,  
대부분의 Spring Security + CORS 문제는 충분히 “추적 가능한 버그”로 바뀐다.

