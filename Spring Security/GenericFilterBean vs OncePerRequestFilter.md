# GenericFilterBean vs OncePerRequestFilter 선택 전략

> 필터 중복 실행 방지와 올바른 필터 베이스 클래스 선택 가이드

## 들어가며

JWT 기반 인증 필터를 구현하던 중 이상한 문제를 발견했다. 로그를 확인해보니 같은 요청에 대해 인증 필터가 두 번 실행되고 있었다. 더 심각한 문제는 데이터베이스 조회도 두 번 발생한다는 것이었다.

원인을 찾아보니 `GenericFilterBean`을 상속받아 구현한 필터가 REQUEST 디스패치와 FORWARD 디스패치에서 각각 실행되고 있었다. 이를 해결하기 위해 Spring이 제공하는 `OncePerRequestFilter`로 변경했고, 문제는 즉시 해결되었다.

주요 니즈는 다음과 같았다:
- **중복 실행 방지**: 같은 요청에서 필터가 여러 번 실행되지 않도록
- **올바른 선택**: 상황에 맞는 필터 베이스 클래스 선택
- **성능 최적화**: 불필요한 데이터베이스 조회 제거
- **명확한 의도**: 필터의 실행 시점을 명확히 제어

---

## 1. GenericFilterBean과 OncePerRequestFilter의 차이

### GenericFilterBean

GenericFilterBean은 Spring이 제공하는 가장 기본적인 필터 베이스 클래스다.

```java
public abstract class GenericFilterBean
    implements Filter, BeanNameAware, EnvironmentAware,
               ServletContextAware, InitializingBean, DisposableBean {

    @Override
    public final void doFilter(ServletRequest request,
                               ServletResponse response,
                               FilterChain chain)
            throws IOException, ServletException {
        // 실제 필터 로직은 서브클래스에서 구현
    }
}
```

**핵심 특징:**

1. **Spring Bean 통합**
   - Spring의 ApplicationContext와 통합
   - `@Value`, `@Autowired` 등 Spring 기능 사용 가능
   - init-param을 Bean 프로퍼티로 자동 변환

2. **단순한 구조**
   - 표준 Servlet Filter 인터페이스를 구현
   - 추가적인 제약이나 로직이 없음
   - 서블릿 스펙 그대로 동작

3. **중복 실행 가능**
   - 같은 요청에서 여러 번 실행될 수 있음
   - REQUEST, FORWARD, INCLUDE, ERROR 디스패치마다 실행
   - 명시적으로 제어하지 않으면 중복 발생

### OncePerRequestFilter

OncePerRequestFilter는 GenericFilterBean을 확장하여 **요청당 한 번만 실행**을 보장한다.

```java
public abstract class OncePerRequestFilter extends GenericFilterBean {

    public static final String ALREADY_FILTERED_SUFFIX = ".FILTERED";

    @Override
    public final void doFilter(ServletRequest request,
                               ServletResponse response,
                               FilterChain filterChain)
            throws ServletException, IOException {

        if (!(request instanceof HttpServletRequest) ||
            !(response instanceof HttpServletResponse)) {
            throw new ServletException("OncePerRequestFilter only supports HTTP requests");
        }

        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;

        String alreadyFilteredAttributeName = getAlreadyFilteredAttributeName();
        boolean hasAlreadyFilteredAttribute = request.getAttribute(alreadyFilteredAttributeName) != null;

        if (hasAlreadyFilteredAttribute) {
            // 이미 실행되었으면 스킵
            filterChain.doFilter(request, response);
        } else {
            // 실행 표시하고 필터 로직 수행
            request.setAttribute(alreadyFilteredAttributeName, Boolean.TRUE);
            try {
                doFilterInternal(httpRequest, httpResponse, filterChain);
            } finally {
                request.removeAttribute(alreadyFilteredAttributeName);
            }
        }
    }

    protected abstract void doFilterInternal(HttpServletRequest request,
                                            HttpServletResponse response,
                                            FilterChain filterChain)
            throws ServletException, IOException;
}
```

**핵심 특징:**

1. **요청당 한 번 실행 보장**
   - 요청 속성에 필터 실행 여부를 기록
   - 같은 요청에서 다시 호출되면 스킵
   - FORWARD, INCLUDE 디스패치에서도 중복 방지

2. **HTTP 전용**
   - HttpServletRequest, HttpServletResponse만 지원
   - 타입 캐스팅 불필요
   - HTTP 중심의 웹 애플리케이션에 최적화

3. **유연한 디스패치 제어**
   - `shouldNotFilter()`: 특정 요청 스킵 가능
   - `shouldNotFilterAsyncDispatch()`: 비동기 디스패치 제어
   - `shouldNotFilterErrorDispatch()`: 에러 디스패치 제어

---

## 2. 중복 실행 문제 상황

### 문제 재현

JSP나 Thymeleaf 같은 서버 사이드 렌더링 환경에서 `RequestDispatcher.forward()`를 사용하면 요청이 다시 필터 체인을 거친다.

```java
@Controller
public class UserController {

    @GetMapping("/users")
    public String getUsers(Model model) {
        List<User> users = userService.findAll();
        model.addAttribute("users", users);
        return "users";  // ViewResolver가 forward 수행
    }
}
```

**요청 흐름:**

```
클라이언트 요청: GET /users
  ↓
필터 체인 (REQUEST 디스패치)
  ↓ JwtAuthenticationFilter 실행 (1번째)
  ↓ DB에서 사용자 조회
  ↓
DispatcherServlet
  ↓
UserController.getUsers()
  ↓
ViewResolver
  ↓ forward("/WEB-INF/views/users.jsp")
  ↓
필터 체인 (FORWARD 디스패치)
  ↓ JwtAuthenticationFilter 실행 (2번째) ← 중복!
  ↓ DB에서 사용자 조회 (또 다시!)
  ↓
JSP 렌더링
```

### GenericFilterBean으로 구현한 경우

```java
public class JwtAuthenticationFilter extends GenericFilterBean {

    private final JwtTokenProvider tokenProvider;
    private final UserRepository userRepository;

    @Override
    public void doFilter(ServletRequest request,
                        ServletResponse response,
                        FilterChain chain) throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;

        String token = extractToken(httpRequest);

        if (token != null && tokenProvider.validateToken(token)) {
            String userId = tokenProvider.getUserId(token);

            // DB 조회 (요청당 여러 번 발생 가능!)
            User user = userRepository.findById(userId)
                .orElseThrow(() -> new UsernameNotFoundException("User not found"));

            Authentication authentication = new UsernamePasswordAuthenticationToken(
                user, null, user.getAuthorities()
            );

            SecurityContextHolder.getContext().setAuthentication(authentication);
        }

        chain.doFilter(request, response);
    }
}
```

**문제점:**

- REQUEST 디스패치: DB 조회 1회
- FORWARD 디스패치: DB 조회 1회 (중복!)
- 총 2회의 불필요한 DB 조회 발생

### OncePerRequestFilter로 해결

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider tokenProvider;
    private final UserRepository userRepository;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String token = extractToken(request);

        if (token != null && tokenProvider.validateToken(token)) {
            String userId = tokenProvider.getUserId(token);

            // DB 조회 (요청당 정확히 1회만 발생!)
            User user = userRepository.findById(userId)
                .orElseThrow(() -> new UsernameNotFoundException("User not found"));

            Authentication authentication = new UsernamePasswordAuthenticationToken(
                user, null, user.getAuthorities()
            );

            SecurityContextHolder.getContext().setAuthentication(authentication);
        }

        filterChain.doFilter(request, response);
    }
}
```

**효과:**

- 요청당 정확히 1회만 실행
- DB 조회도 1회로 감소
- 성능 개선 및 예측 가능한 동작

---

## 3. 언제 어떤 필터를 사용할까?

### OncePerRequestFilter를 사용해야 하는 경우

대부분의 Spring Security 커스텀 필터는 OncePerRequestFilter를 사용해야 한다.

**1. 인증 필터**

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String token = extractToken(request);

        if (token != null) {
            // 토큰 검증은 요청당 한 번만 수행되어야 함
            Authentication authentication = authenticate(token);
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }

        filterChain.doFilter(request, response);
    }
}
```

**이유:**
- 인증은 요청당 한 번만 수행되어야 함
- 데이터베이스 조회나 외부 API 호출이 중복되면 안 됨
- SecurityContext 설정도 한 번만 필요

**2. 로깅 필터**

```java
public class RequestLoggingFilter extends OncePerRequestFilter {

    private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        long startTime = System.currentTimeMillis();
        String requestId = UUID.randomUUID().toString();

        MDC.put("requestId", requestId);

        try {
            log.info("Request: {} {}", request.getMethod(), request.getRequestURI());
            filterChain.doFilter(request, response);
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            log.info("Response: {} - {}ms", response.getStatus(), duration);
            MDC.clear();
        }
    }
}
```

**이유:**
- 로그는 요청당 한 번만 기록되어야 함
- 중복 로그는 분석을 어렵게 만듦
- 요청 ID는 하나여야 추적이 가능

**3. 멀티 테넌트 필터**

```java
public class TenantFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String tenantId = extractTenantId(request);

        // 테넌트 컨텍스트 설정은 요청당 한 번만
        TenantContext.setCurrentTenant(tenantId);

        try {
            filterChain.doFilter(request, response);
        } finally {
            TenantContext.clear();
        }
    }

    private String extractTenantId(HttpServletRequest request) {
        String tenantId = request.getHeader("X-Tenant-ID");
        if (tenantId == null) {
            tenantId = request.getParameter("tenantId");
        }
        return tenantId;
    }
}
```

**이유:**
- ThreadLocal 기반 컨텍스트 설정은 한 번만 필요
- 중복 설정은 의미 없고 성능 낭비
- finally 블록의 정리도 한 번만 수행되어야 함

**4. 요청 본문 캐싱 필터**

```java
public class CachingRequestBodyFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        // 요청 본문을 여러 번 읽을 수 있도록 캐싱
        // 이 작업은 요청당 한 번만 수행되어야 함
        CachedBodyHttpServletRequest cachedRequest =
            new CachedBodyHttpServletRequest(request);

        filterChain.doFilter(cachedRequest, response);
    }
}
```

**이유:**
- 요청 본문 캐싱은 한 번만 수행되어야 함
- 중복 캐싱은 메모리 낭비
- InputStream은 한 번만 읽을 수 있으므로 래핑도 한 번만

### GenericFilterBean을 사용하는 경우

GenericFilterBean은 특수한 상황에서만 사용한다.

**1. 디스패치 타입별로 다른 동작이 필요한 경우**

```java
public class SpecialLoggingFilter extends GenericFilterBean {

    @Override
    public void doFilter(ServletRequest request,
                        ServletResponse response,
                        FilterChain chain) throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;
        DispatcherType dispatcherType = httpRequest.getDispatcherType();

        switch (dispatcherType) {
            case REQUEST:
                log.info("Original request: {}", httpRequest.getRequestURI());
                break;
            case FORWARD:
                log.info("Forwarded to: {}", httpRequest.getRequestURI());
                break;
            case INCLUDE:
                log.info("Included: {}", httpRequest.getRequestURI());
                break;
            case ERROR:
                log.info("Error handling: {}", httpRequest.getRequestURI());
                break;
        }

        chain.doFilter(request, response);
    }
}
```

**이유:**
- 디스패치 타입마다 다른 로깅이 필요
- 의도적으로 모든 디스패치에서 실행되어야 함

**2. 레거시 코드 유지**

```java
public class LegacySecurityFilter extends GenericFilterBean {

    @Override
    public void doFilter(ServletRequest request,
                        ServletResponse response,
                        FilterChain chain) throws IOException, ServletException {

        // 레거시 시스템과의 호환성 유지
        // 기존 동작을 그대로 보존해야 하는 경우

        chain.doFilter(request, response);
    }
}
```

**이유:**
- 기존 동작을 변경하면 안 되는 레거시 코드
- 리팩토링 시 신중하게 접근

하지만 대부분의 경우 OncePerRequestFilter로 전환하는 것이 낫다.

---

## 4. OncePerRequestFilter의 고급 기능

### 특정 요청 스킵하기

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private static final List<String> EXCLUDED_PATHS = List.of(
        "/api/public",
        "/api/auth/login",
        "/api/auth/register"
    );

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        String path = request.getRequestURI();
        return EXCLUDED_PATHS.stream()
            .anyMatch(path::startsWith);
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        // 인증 로직
        filterChain.doFilter(request, response);
    }
}
```

**효과:**
- 공개 API는 필터를 거치지 않음
- 불필요한 처리 제거
- 가독성 향상

### 비동기 요청 제어

```java
public class AsyncAwareFilter extends OncePerRequestFilter {

    @Override
    protected boolean shouldNotFilterAsyncDispatch() {
        // 비동기 디스패치에서는 필터를 실행하지 않음
        return true;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        log.info("Original request only");
        filterChain.doFilter(request, response);
    }
}
```

**활용 사례:**
- 초기 요청에서만 인증을 수행
- 비동기 처리에서는 이미 설정된 SecurityContext 사용
- 중복 인증 방지

### 에러 디스패치 제어

```java
public class ErrorHandlingFilter extends OncePerRequestFilter {

    @Override
    protected boolean shouldNotFilterErrorDispatch() {
        // 에러 디스패치에서는 필터를 실행하지 않음
        return true;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        // 정상 요청만 처리
        filterChain.doFilter(request, response);
    }
}
```

**이유:**
- 에러 페이지 렌더링 시 추가 처리 불필요
- 에러 상황에서 예외 발생 방지

---

## 5. 실무 사례: JWT 인증 필터 개선

### 문제 상황

초기에 GenericFilterBean으로 구현한 JWT 필터에서 성능 문제가 발생했다.

```java
public class JwtAuthenticationFilter extends GenericFilterBean {

    private final UserDetailsService userDetailsService;
    private final JwtTokenProvider tokenProvider;

    @Override
    public void doFilter(ServletRequest request,
                        ServletResponse response,
                        FilterChain chain) throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;

        String token = resolveToken(httpRequest);

        if (token != null && tokenProvider.validateToken(token)) {
            String username = tokenProvider.getUsername(token);

            // DB 조회 - FORWARD 디스패치에서도 다시 실행됨!
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);

            UsernamePasswordAuthenticationToken authentication =
                new UsernamePasswordAuthenticationToken(
                    userDetails, null, userDetails.getAuthorities()
                );

            SecurityContextHolder.getContext().setAuthentication(authentication);
        }

        chain.doFilter(request, response);
    }
}
```

**발생한 문제:**

1. **성능 저하**
   - 하나의 요청에서 DB 조회 2회 발생
   - 부하가 높은 시간대에 DB 커넥션 풀 고갈

2. **로그 중복**
   - 같은 토큰 검증 로그가 2번 출력
   - 로그 분석 시 혼란

3. **비효율적인 리소스 사용**
   - 이미 설정된 SecurityContext를 다시 설정
   - 메모리와 CPU 낭비

### 해결 방법

OncePerRequestFilter로 변경하고 추가 최적화를 적용했다.

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final UserDetailsService userDetailsService;
    private final JwtTokenProvider tokenProvider;

    private static final List<String> EXCLUDED_PATHS = List.of(
        "/api/auth/login",
        "/api/auth/register",
        "/api/public"
    );

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        String path = request.getRequestURI();
        return EXCLUDED_PATHS.stream().anyMatch(path::startsWith);
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        try {
            String token = resolveToken(request);

            if (token != null && tokenProvider.validateToken(token)) {
                String username = tokenProvider.getUsername(token);

                // 요청당 정확히 1회만 DB 조회
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities()
                    );

                SecurityContextHolder.getContext().setAuthentication(authentication);

                log.debug("Set authentication for user: {}", username);
            }
        } catch (Exception ex) {
            log.error("Cannot set user authentication", ex);
        }

        filterChain.doFilter(request, response);
    }

    private String resolveToken(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

**개선 효과:**

| 항목 | 변경 전 | 변경 후 | 개선율 |
|-----|---------|---------|--------|
| 요청당 DB 조회 횟수 | 2회 | 1회 | 50% 감소 |
| 평균 응답 시간 | 120ms | 85ms | 29% 개선 |
| DB 커넥션 사용량 | 높음 | 정상 | 안정화 |
| 로그 중복 | 있음 | 없음 | 명확해짐 |

---

## 6. 비교 정리

### 기능 비교표

| 기능 | GenericFilterBean | OncePerRequestFilter |
|-----|------------------|---------------------|
| **Spring Bean 통합** | O | O |
| **요청당 한 번 실행 보장** | X | O |
| **HTTP 전용** | X | O |
| **타입 캐스팅 필요** | O | X |
| **특정 요청 스킵** | 수동 구현 | shouldNotFilter() |
| **비동기 디스패치 제어** | 수동 구현 | shouldNotFilterAsyncDispatch() |
| **에러 디스패치 제어** | 수동 구현 | shouldNotFilterErrorDispatch() |
| **사용 복잡도** | 낮음 | 낮음 |
| **권장 사용처** | 특수 상황 | 대부분의 경우 |

### 선택 가이드

**OncePerRequestFilter를 사용하라:**

- 인증/인가 필터를 만들 때
- 데이터베이스 조회를 하는 필터
- 외부 API를 호출하는 필터
- 로깅이나 모니터링 필터
- ThreadLocal 컨텍스트를 설정하는 필터
- 요청 본문이나 응답 본문을 캐싱하는 필터

**GenericFilterBean을 사용하라:**

- 디스패치 타입별로 다른 동작이 필요할 때
- 레거시 시스템 유지가 필요할 때
- 의도적으로 모든 디스패치에서 실행되어야 할 때

**의심스러우면 OncePerRequestFilter를 선택하라.** 대부분의 경우 올바른 선택이다.

---

## 7. 실무 관점의 주의사항

### 필터 중복 등록 방지

Spring Boot에서 필터를 `@Component`로 선언하면 자동으로 Servlet Container에 등록된다. 이를 방지하려면:

```java
@Bean
public FilterRegistrationBean<JwtAuthenticationFilter> jwtFilterRegistration(
        JwtAuthenticationFilter filter) {

    FilterRegistrationBean<JwtAuthenticationFilter> registration =
        new FilterRegistrationBean<>(filter);
    registration.setEnabled(false);  // Servlet Container 등록 비활성화

    return registration;
}

@Bean
public SecurityFilterChain filterChain(HttpSecurity http,
                                      JwtAuthenticationFilter jwtFilter) throws Exception {
    http
        .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);

    return http.build();
}
```

### 예외 처리 전략

OncePerRequestFilter 내부에서 발생한 예외는 ExceptionTranslationFilter로 전달되지 않을 수 있다.

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        try {
            String token = resolveToken(request);

            if (token != null) {
                validateAndAuthenticate(token);
            }

            filterChain.doFilter(request, response);

        } catch (JwtException ex) {
            // JWT 예외는 직접 처리
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.getWriter().write("{\"error\": \"Invalid token\"}");

        } catch (Exception ex) {
            // 기타 예외는 상위로 전달
            throw new ServletException("Authentication failed", ex);
        }
    }
}
```

### 성능 모니터링

필터 실행 시간을 모니터링하여 성능 병목 지점을 파악한다.

```java
public class PerformanceMonitoringFilter extends OncePerRequestFilter {

    private static final Logger log = LoggerFactory.getLogger(PerformanceMonitoringFilter.class);

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        long startTime = System.currentTimeMillis();

        try {
            filterChain.doFilter(request, response);
        } finally {
            long duration = System.currentTimeMillis() - startTime;

            if (duration > 1000) {
                log.warn("Slow request: {} {} - {}ms",
                    request.getMethod(),
                    request.getRequestURI(),
                    duration);
            }
        }
    }
}
```

---

## 8. 핵심 원칙 정리

### 기본 선택은 OncePerRequestFilter

99%의 경우 OncePerRequestFilter가 올바른 선택이다. 중복 실행 방지, HTTP 전용 타입, 편의 메서드 제공 등 실무에서 필요한 모든 기능을 갖추고 있다.

### 중복 실행은 버그의 근원

같은 필터가 여러 번 실행되면 데이터베이스 조회 중복, 로그 중복, 컨텍스트 설정 중복 등 예측하기 어려운 문제가 발생한다. OncePerRequestFilter로 원천 차단한다.

### shouldNotFilter 활용

모든 요청을 필터링할 필요는 없다. 공개 API나 정적 리소스는 shouldNotFilter()로 제외하여 성능을 최적화한다.

### 예외 처리는 명시적으로

필터 내부의 예외는 직접 처리하거나, 명시적으로 상위로 전달한다. 방치하면 클라이언트는 500 에러만 받게 된다.

### 디버깅 가능하게

로그를 적절히 남겨서 필터가 언제 실행되고, 어떤 결정을 내렸는지 추적 가능하게 만든다.

---

## 마치며

GenericFilterBean과 OncePerRequestFilter의 차이는 단순해 보이지만, 실무에서는 성능과 안정성에 직접적인 영향을 미친다.

JWT 필터를 처음 만들었을 때 GenericFilterBean을 사용했다가 중복 실행 문제로 고생한 경험이 있다. 로그를 보고서야 같은 요청이 두 번 처리되고 있다는 것을 알았고, OncePerRequestFilter로 변경하면서 문제가 해결되었다. 이후 모든 커스텀 필터는 기본적으로 OncePerRequestFilter를 사용하고 있다.

**OncePerRequestFilter를 선택해야 하는 이유는 명확하다:**
- 요청당 한 번 실행 보장으로 성능 최적화
- HTTP 전용 타입으로 타입 안정성 확보
- 편의 메서드로 코드 간결화
- Spring Security 공식 필터들도 대부분 OncePerRequestFilter 사용

중요한 건 **언제 필터가 실행되는가**를 명확히 이해하는 것이다. 디스패치 타입, 요청 속성, SecurityContext 생명주기를 이해하면 올바른 필터를 만들 수 있다.

---

## 참고자료

- [GenericFilterBean - Spring Framework API](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/filter/GenericFilterBean.html)
- [OncePerRequestFilter - Spring Framework API](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/filter/OncePerRequestFilter.html)
- [OncePerRequestFilter Source Code - Spring Framework GitHub](https://github.com/spring-projects/spring-framework/blob/main/spring-web/src/main/java/org/springframework/web/filter/OncePerRequestFilter.java)
- [Baeldung - What Is OncePerRequestFilter?](https://www.baeldung.com/spring-onceperrequestfilter)
