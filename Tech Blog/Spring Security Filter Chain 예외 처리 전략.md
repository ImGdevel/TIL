# Spring Security Filter Chain 예외 처리, 한 번 제대로 잡아보기

> “분명 401을 기대했는데, 브라우저는 자꾸 로그인 페이지로 튀어간다.”

## 0. @ControllerAdvice가 아무 일도 안 하던 그 날

처음 Spring Security로 REST API를 만들던 시절, 나는 꽤 자신감에 차 있었다.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<ErrorResponse> handleIllegalArgument(IllegalArgumentException e) {
        return ResponseEntity.badRequest()
            .body(new ErrorResponse(e.getMessage()));
    }
}
```

예외가 터지면 위 핸들러가 깔끔한 JSON을 내려줄 거라고 믿어 의심치 않았다.

그런데 실제로는 전혀 다른 일이 벌어졌다.

- 인증이 안 된 상태로 `/api/protected`에 접근하면  
  `@ControllerAdvice`는 꿈쩍도 안 하고, 브라우저는 갑자기 로그인 페이지로 리다이렉트된다.
- 권한이 없는 사용자로 `/api/admin`을 호출하면  
  마찬가지로 JSON이 아니라 HTML 응답이 떨어진다.

“도대체 내 예외 핸들러는 어디 갔지?”

로그를 한참 뒤져보니, 문제의 원인은 아주 단순했다.

> **Spring Security 예외는 컨트롤러까지 가지도 못 하고, 필터 체인에서 먼저 잡혀 버린다.**

이 글은 그때의 혼란을 정리하면서,

- Spring Security Filter Chain에서 예외가 어떻게 흐르고  
- 401/403 같은 인증/인가 실패를 어떻게 원하는 형태의 응답으로 바꿔낼 수 있는지

를 기록해 둔 글이다.

앞선 글들에서 Spring Security Servlet 아키텍처와 인증 아키텍처를 “뼈대”와 “혈관”으로 봤다면,  
이번 글은 그 위에서 **“에러가 났을 때 피를 어떻게 빼낼지”**에 가깝다.

---

## 1. 왜 굳이 Filter Chain 예외까지 신경 써야 할까?

솔직히 말해서, 처음에는 이렇게 생각했다.

> “예외 처리는 그냥 `@ControllerAdvice` 하나로 다 되지 않나?”

하지만 실무에서 잠깐만 겪어보면 아래와 같은 일들이 훅훅 튀어나온다.

### 1) 프론트가 계속 묻는다. “이 API는 왜 가끔 HTML을 반환하나요?”

REST API로 약속한 건 **항상 JSON 응답**이다.

그런데 인증이 끊기면:

- 어떤 요청은 302 + 로그인 페이지 HTML  
- 어떤 요청은 401 JSON  

프론트 입장에서는 지옥이다.  
“혹시 이 API는 레거시인가요?” 같은 질문을 듣기 시작하면 이미 늦었다.

### 2) 401과 403이 뒤섞여서 더 헷갈린다

- 인증이 안 되면 401  
- 인증은 됐지만 권한이 없으면 403

머리로는 알고 있는데, 실제로는 이런 상황이 나온다.

- 익명 사용자로 `/api/admin`을 호출 → 302 로그인 페이지  
- 인증된 사용자지만 ROLE_USER로 `/api/admin` 호출 → 403  

로깅을 잘 해보면 둘 다 `AccessDeniedException`처럼 보일 때도 있고,  
ExceptionTranslationFilter 안에서 처리 흐름이 갈라진다.

### 3) `@ControllerAdvice`로 잡으려다가 더 꼬인다

예외가 컨트롤러 이전 레이어(필터)에서 발생하는데,  
컨트롤러 이후 레이어(`@ControllerAdvice`)에서 잡으려 하니 계속 미끄러진다.

이쯤 되면 인정해야 한다.

> **“Spring Security 예외는 MVC 예외랑 레이어 자체가 다르다.”**  
> **“Filter Chain 안에서 해결해야 한다.”**

그래서 나는 공식 문서의 *ExceptionTranslationFilter* 섹션을 붙잡고,  
요청 하나가 필터 체인을 타며 예외가 어떻게 처리되는지 끝까지 따라가 보기로 했다.

---

## 2. 큰 그림: 예외는 컨트롤러까지 가지 못한다

우리가 익숙한 예외 흐름은 이렇다.

```text
요청 → DispatcherServlet → Controller → Service → 예외 발생
    → @ControllerAdvice → 응답 변환
```

하지만 Spring Security 예외는 이렇게 흐른다.

```text
요청
  ↓
Servlet Container Filter Chain
  ↓
Spring Security Filter Chain (여기서 Authentication/Authorization 예외 발생)
  ↓
DispatcherServlet
  ↓
Controller
  ↓
@ControllerAdvice
```

핵심 포인트는 하나다.

> **인증/인가 예외는 대부분 컨트롤러에 도달하기 전에 터진다.**

그리고 이 예외들을 전담해서 받아주는 필터가 있다.

- 이름: `ExceptionTranslationFilter`  
- 위치: `AuthorizationFilter` 바로 앞  
- 관심사: 오직 두 가지 예외
  - `AuthenticationException` (인증 실패)
  - `AccessDeniedException` (인가 실패)

다른 예외(`NullPointerException`, `IllegalArgumentException` 등)는  
이 필터를 그냥 통과해서 상위로 전파된다.

이 구조를 이해하는 순간, 머릿속에서 이런 정리가 됐다.

> “인증/인가 실패는 **ExceptionTranslationFilter + AuthenticationEntryPoint + AccessDeniedHandler** 조합으로 처리하고,  
>  그 외 시스템 에러는 별도 필터나 `@ControllerAdvice`에서 처리해야겠다.”

---

## 3. ExceptionTranslationFilter와 친구들 소개

### 3-1. ExceptionTranslationFilter – 보안 예외를 HTTP 응답으로 번역하는 통역사

이 필터의 역할을 한 줄로 요약하면 이렇다.

> **“필터 체인 아래에서 올라오는 보안 예외를 적절한 HTTP 응답(또는 리다이렉트)으로 바꿔주는 통역사”**

코드를 단순화해서 보면:

```java
try {
    chain.doFilter(request, response);
} catch (AuthenticationException ex) {
    // 인증 실패 → AuthenticationEntryPoint로
    handleAuthenticationException(request, response, ex);
} catch (AccessDeniedException ex) {
    // 인가 실패 → 익명/인증 여부에 따라 분기
    handleAccessDeniedException(request, response, ex);
}
```

핵심은:

- 이 필터는 **인증/인가 관련 예외 두 종류만** 관심 있다.  
- 그 외 예외는 그대로 위로 던진다. (그래서 따로 필터를 만들어야 한다.)

### 3-2. AuthenticationEntryPoint – “로그인 먼저 하세요”를 담당하는 안내원

`AuthenticationEntryPoint`는 **“미인증 상태에서 보호된 리소스에 접근했다”**라는 상황을 담당한다.

- 전통적인 웹 앱: 로그인 페이지로 리다이렉트  
- REST API: 401 Unauthorized + JSON 응답

인터페이스는 아주 단순하다.

```java
public interface AuthenticationEntryPoint {
    void commence(HttpServletRequest request,
                  HttpServletResponse response,
                  AuthenticationException authException)
        throws IOException, ServletException;
}
```

여기서는 `ResponseEntity` 같은 MVC 기능을 쓸 수 없다.  
**그냥 서블릿 응답을 직접 만지는 필터 레벨**이라는 걸 항상 기억해야 한다.

### 3-3. AccessDeniedHandler – “여기까지는 로그인했어도 들어올 수 없습니다”

`AccessDeniedHandler`는 **“인증은 됐는데 권한이 부족한 경우”**를 담당한다.

- 보통 403 Forbidden 응답을 내려준다.
- 상황에 따라 “권한 없음 페이지”로 리다이렉트할 수도 있다.

인터페이스 역시 간단하다.

```java
public interface AccessDeniedHandler {
    void handle(HttpServletRequest request,
                HttpServletResponse response,
                AccessDeniedException accessDeniedException)
        throws IOException, ServletException;
}
```

ExceptionTranslationFilter는 이렇게 사용자를 나눈다.

- 익명 사용자 + 인가 실패 → 사실상 “인증이 안 된 상태”로 보고 `AuthenticationEntryPoint` 호출  
- 인증된 사용자 + 인가 실패 → `AccessDeniedHandler` 호출

### 3-4. RequestCache – “로그인 성공 후, 원래 가려던 곳으로 돌려보내는 기억 저장소”

폼 로그인 환경에서 많이 등장하는 녀석이다.

인증이 안 된 상태에서 `/protected/page`에 접근하면:

1. RequestCache에 이 요청 정보를 저장해 두고  
2. 로그인 페이지로 리다이렉트  
3. 로그인 성공 후, 저장해 둔 원래 URL로 다시 리다이렉트

REST API에서는 보통 이 기능이 필요 없어서,  
`NullRequestCache`로 바꾸기도 한다.

```java
http.requestCache(cache -> cache
    .requestCache(new NullRequestCache())
);
```

---

## 4. 401과 403이 어떻게 만들어지는지, 요청 하나로 따라가 보기

말로만 듣고 있으면 감이 안 오니, 실제 시나리오 하나를 끝까지 따라가 보자.

### 시나리오 A: 익명 사용자가 `/api/admin`에 접근했을 때 (401)

1. 익명 사용자가 `GET /api/admin` 호출  
2. Security Filter Chain을 타고 내려가다 `AuthorizationFilter`에서 권한 부족 감지 → `AccessDeniedException` 발생  
3. 이 예외가 위로 올라오다가 `ExceptionTranslationFilter`에서 잡힌다.  
4. ExceptionTranslationFilter는 현재 사용자가 익명(anonymous)인지 확인한다.  
5. 익명이므로, “이건 사실상 인증이 안 된 상태”로 보고 `AuthenticationEntryPoint`를 호출한다.  
6. AuthenticationEntryPoint 구현에 따라:
   - 웹: 로그인 페이지로 302 리다이렉트  
   - REST API: 401 + JSON 응답 반환

이 시나리오를 처음 알았을 때, 나는 한동안 `AccessDeniedException`을 보면 전부 403인 줄 알았던 과거의 나를 떠올리며 머쓱해했다.

### 시나리오 B: 인증된 ROLE_USER가 `/api/admin`에 접근했을 때 (403)

1. ROLE_USER로 로그인한 사용자가 `GET /api/admin` 호출  
2. 마찬가지로 `AuthorizationFilter`에서 권한 부족 → `AccessDeniedException` 발생  
3. ExceptionTranslationFilter가 예외를 잡고, 이번에는 **인증된 사용자**라는 걸 확인  
4. `AccessDeniedHandler`를 호출해 403 응답을 만들게 한다.

즉, **같은 `AccessDeniedException`이라도 “현재 사용자가 익명인지 여부”에 따라 401/403으로 갈린다.**

### 시나리오 C: 필터 내부에서 NPE가 터졌을 때 (500)

이건 조금 다른 케이스다.

- Custom Filter 안에서 `NullPointerException` 발생  
- ExceptionTranslationFilter는 이 예외를 처리하지 않는다.  
- 그냥 상위로 던지고, 결국 컨테이너 레벨까지 올라가서 500이 되거나,  
  별도로 만든 예외 처리 필터에서 잡게 된다.

그래서 나는 `AuthenticationException`/`AccessDeniedException`을 제외한 예외는  
별도 필터(`FilterExceptionHandler`)에서 JSON으로 감싸주는 방식을 택했다.

---

## 5. 실전: REST API용 예외 처리 전략 세우기

예외 흐름을 이해하고 나서, 실제 프로젝트에서는 이렇게 정리했다.

1. 인증 실패/인가 실패 → 필터 레벨에서 401/403 JSON으로 통일  
2. 그 외 서버 에러 → 필터 또는 `@ControllerAdvice`에서 500 JSON  
3. Form Login이 필요한 화면은 따로 체인을 나눠서 리다이렉트 유지

### 5-1. JSON을 반환하는 AuthenticationEntryPoint와 AccessDeniedHandler

먼저, API용 예외 응답 포맷을 하나 정했다.

```java
@Getter
@Builder
public class ErrorResponse {
    private int status;
    private String error;
    private String message;
    private String path;
    private Instant timestamp;
}
```

그리고 이 DTO를 사용해서 두 가지 핸들러를 만들었다.

```java
public class JsonAuthenticationEntryPoint implements AuthenticationEntryPoint {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public void commence(HttpServletRequest request,
                         HttpServletResponse response,
                         AuthenticationException ex) throws IOException {

        ErrorResponse body = ErrorResponse.builder()
            .status(HttpStatus.UNAUTHORIZED.value())
            .error("Unauthorized")
            .message("인증이 필요합니다.")
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .build();

        response.setStatus(HttpStatus.UNAUTHORIZED.value());
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setCharacterEncoding(StandardCharsets.UTF_8.name());

        response.getWriter().write(objectMapper.writeValueAsString(body));
    }
}
```

```java
public class JsonAccessDeniedHandler implements AccessDeniedHandler {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public void handle(HttpServletRequest request,
                       HttpServletResponse response,
                       AccessDeniedException ex) throws IOException {

        ErrorResponse body = ErrorResponse.builder()
            .status(HttpStatus.FORBIDDEN.value())
            .error("Forbidden")
            .message("접근 권한이 없습니다.")
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .build();

        response.setStatus(HttpStatus.FORBIDDEN.value());
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setCharacterEncoding(StandardCharsets.UTF_8.name());

        response.getWriter().write(objectMapper.writeValueAsString(body));
    }
}
```

그리고 Security 설정에서 이렇게 연결했다.

```java
@Bean
public SecurityFilterChain apiSecurity(HttpSecurity http) throws Exception {
    http
        .securityMatcher("/api/**")
        .exceptionHandling(ex -> ex
            .authenticationEntryPoint(new JsonAuthenticationEntryPoint())
            .accessDeniedHandler(new JsonAccessDeniedHandler())
        )
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/public/**").permitAll()
            .anyRequest().authenticated()
        );

    return http.build();
}
```

이렇게 하고 나니 적어도 `/api/**` 구간에서는

- 인증 실패 → 항상 401 JSON  
- 인가 실패 → 항상 403 JSON

이라는 약속을 프론트와 안정적으로 맞출 수 있게 되었다.

### 5-2. 일반 예외를 한 번에 묶어주는 FilterExceptionHandler

이제 남은 문제는 `NullPointerException`, `IllegalStateException` 같은 일반 예외였다.

이 예외들은 필터 레벨에서 터질 수도 있고, 컨트롤러 이후에서 터질 수도 있다.  
나는 “필터 체인에서 터지는 예외”만 따로 한 번 더 감싸기로 했다.

```java
@Component
public class FilterExceptionHandler extends OncePerRequestFilter {

    private final ObjectMapper objectMapper;

    public FilterExceptionHandler(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
        throws ServletException, IOException {

        try {
            filterChain.doFilter(request, response);
        } catch (AuthenticationException | AccessDeniedException ex) {
            // 이 둘은 ExceptionTranslationFilter가 처리해야 하므로 다시 던진다.
            throw ex;
        } catch (Exception ex) {
            handleUnexpectedException(request, response, ex);
        }
    }

    private void handleUnexpectedException(HttpServletRequest request,
                                           HttpServletResponse response,
                                           Exception ex) throws IOException {

        ErrorResponse body = ErrorResponse.builder()
            .status(HttpStatus.INTERNAL_SERVER_ERROR.value())
            .error("Internal Server Error")
            .message(ex.getMessage())
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .build();

        response.setStatus(HttpStatus.INTERNAL_SERVER_ERROR.value());
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setCharacterEncoding(StandardCharsets.UTF_8.name());

        response.getWriter().write(objectMapper.writeValueAsString(body));
    }
}
```

그리고 이 필터를 적절한 위치에 끼워 넣었다.

```java
http.addFilterBefore(filterExceptionHandler, UsernamePasswordAuthenticationFilter.class);
```

이렇게 하면,

- `AuthenticationException`, `AccessDeniedException` → ExceptionTranslationFilter로  
- 그 외 예외 → FilterExceptionHandler에서 500 JSON으로 통일

이라는 그림이 필터 체인 위에서 자연스럽게 그려진다.

---

## 6. 실무에서 부딪히며 배운 것들

### 6-1. 레이어를 헷갈리지 말 것

- 컨트롤러 이후 예외 → `@ControllerAdvice`  
- Spring Security Filter Chain 예외 → `ExceptionTranslationFilter` + 커스텀 필터

이 경계를 이해하고 나니,  
“왜 여기서는 @ExceptionHandler가 안 먹지?”라는 질문이 확 줄었다.

### 6-2. AuthenticationException / AccessDeniedException은 필터에서 삼키지 않는다

일반 예외 처리 필터를 만들 때 가장 많이 하는 실수가 이거였다.

```java
catch (Exception ex) {
    // 여기서 모든 예외를 다 JSON으로 바꿔버림 → 401/403 흐름도 다 깨짐
}
```

이렇게 해버리면 ExceptionTranslationFilter까지 예외가 도달하지 못 해서,  
인증/인가 실패가 전부 500으로 뭉개진다.

그래서 항상 이렇게 써야 한다.

```java
} catch (AuthenticationException | AccessDeniedException ex) {
    throw ex; // 다시 던진다
} catch (Exception ex) {
    // 여기서만 처리
}
```

### 6-3. 환경별로 에러 메시지를 다르게

개발 환경에서는 상세 스택 메시지가 도움이 되지만,  
운영 환경에서는 보안상 노출하면 안 되는 정보가 될 수 있다.

그래서 보통은 설정으로 제어한다.

```java
@Bean
public AuthenticationEntryPoint jsonAuthenticationEntryPoint(
    @Value("${security.exception.include-detail:false}") boolean includeDetail
) {
    return new JsonAuthenticationEntryPoint(includeDetail);
}
```

핵심은 **“어떤 정보를 어디까지 보여줄지”**를 의도적으로 결정하는 것이다.

### 6-4. CORS와 예외가 겹칠 때

프론트에서 보면 가끔 이렇게 보인다.

- 예외는 잘 처리한 것 같은데, 브라우저 콘솔에는 CORS 에러만 찍힌다.

이럴 땐 CORS 필터가 인증 필터보다 앞에 제대로 배치되어 있는지,  
preflight(OPTIONS) 요청이 인증 없이 통과할 수 있는지 다시 확인하게 된다.

---

## 7. 마무리 – “어디에서 예외가 터졌는지”가 반이다

이 글을 정리하면서 가장 크게 남은 문장은 이거였다.

> **“Spring Security에서 예외 처리를 이해한다는 건,  
>  ‘예외가 어느 레이어에서 발생했는지’를 정확히 보는 법을 배우는 것이다.”**

- Filter Chain 안에서 터진 예외인지,  
- DispatcherServlet 이후에서 터진 예외인지,  
- 인증 단계인지, 인가 단계인지

이 구분이 되기 시작하자,

- 401/403 응답을 의도적으로 설계할 수 있게 되었고  
- 프론트와 “어떤 상황에서 어떤 에러를 받을지”를 합의하기가 훨씬 쉬워졌다.

지금 이 글을 읽고 있다면,  
한 번쯤은 실제 프로젝트에서 아래 세 가지를 직접 확인해 보길 추천한다.

1. `/api/protected`를 익명/인증/권한 부족 세 가지 상태에서 호출해 보고,  
2. 로그로 ExceptionTranslationFilter 안에서 어떤 예외가 어떻게 처리되는지 보고,  
3. AuthenticationEntryPoint와 AccessDeniedHandler가 언제 호출되는지 눈으로 따라가 보기.

그 과정을 지나고 나면,  
Spring Security의 예외 처리는 더 이상 “어디서 어떻게 튀어나올지 모르는 귀신”이 아니라,  
**예상 가능한 규칙을 가진 친구**에 조금 더 가까워져 있을 거다.

