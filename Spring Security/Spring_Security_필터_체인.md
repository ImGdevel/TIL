# Spring Security: 필터 체인의 동작 원리와 커스터마이징

> Spring Security의 필터 체인 구조를 이해하고, 실무에서 커스텀 필터를 안전하게 추가하는 방법을 다룹니다.

---

## 문제 상황

REST API 서버에 JWT 기반 인증을 구현해야 합니다.
모든 요청의 Authorization 헤더에서 JWT 토큰을 추출하고, 유효성을 검증한 후 인증 정보를 SecurityContext에 설정해야 합니다.

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping
    public List<Order> getOrders() {
        // 현재 인증된 사용자의 주문 목록 조회
        // 하지만 JWT 토큰을 어떻게 검증하고 인증 정보를 설정할까?
    }
}
```

**발생 가능한 이슈:**
- JWT 토큰 검증 로직을 어디에 배치해야 할지 모호합니다.
- Spring Security의 기존 필터들과 어떻게 통합해야 할지 불분명합니다.
- 필터를 잘못된 위치에 추가하면 인증이 제대로 동작하지 않습니다.
- 예외 처리가 제대로 되지 않아 보안 취약점이 발생할 수 있습니다.

---

## 배경 지식

### Spring Security의 동작 원리

Spring Security는 **서블릿 필터 기반**으로 동작합니다.
클라이언트의 요청이 컨트롤러에 도달하기 전에 일련의 필터들을 거치면서 인증과 인가를 처리합니다.

이를 도서관 출입 절차에 비유하면 다음과 같습니다:

- **필터 체인** = 도서관 출입 절차의 여러 검문소
- **각각의 필터** = 신분증 확인, 출입증 발급, 금지 물품 검사 등의 각 단계
- **SecurityContext** = 출입증 (인증된 사용자 정보 보관)
- **컨트롤러** = 도서관 내부 (목적지)

### 서블릿 필터란?

서블릿 필터는 요청과 응답을 가로채서 전처리 또는 후처리를 수행하는 컴포넌트입니다.

```java
public interface Filter {
    void doFilter(ServletRequest request,
                  ServletResponse response,
                  FilterChain chain) throws IOException, ServletException;
}
```

**핵심 개념:**
- `doFilter()` 메서드에서 요청을 처리합니다.
- `chain.doFilter(request, response)`를 호출하여 다음 필터로 전달합니다.
- 다음 필터를 호출하지 않으면 요청이 더 이상 진행되지 않습니다.

---

## Spring Security 필터 체인의 구조

### 필터 체인의 전체 흐름

Spring Security는 기본적으로 15개 이상의 필터를 순차적으로 실행합니다.
주요 필터들의 실행 순서는 다음과 같습니다:

```
클라이언트 요청
    ↓
1. SecurityContextPersistenceFilter
    ↓
2. LogoutFilter
    ↓
3. UsernamePasswordAuthenticationFilter
    ↓
4. BasicAuthenticationFilter
    ↓
5. RequestCacheAwareFilter
    ↓
6. SecurityContextHolderAwareRequestFilter
    ↓
7. AnonymousAuthenticationFilter
    ↓
8. SessionManagementFilter
    ↓
9. ExceptionTranslationFilter
    ↓
10. FilterSecurityInterceptor
    ↓
컨트롤러
```

### 주요 필터의 역할

각 필터가 수행하는 역할을 이해해야 커스텀 필터를 올바른 위치에 배치할 수 있습니다.

| 필터 이름 | 역할 | 실행 시점 |
|----------|------|----------|
| SecurityContextPersistenceFilter | SecurityContext를 로드하고 저장 | 가장 먼저 |
| UsernamePasswordAuthenticationFilter | 폼 로그인 처리 | 인증 단계 |
| BasicAuthenticationFilter | HTTP Basic 인증 처리 | 인증 단계 |
| ExceptionTranslationFilter | 인증/인가 예외 처리 | 인가 직전 |
| FilterSecurityInterceptor | 최종 인가 결정 | 가장 마지막 |

### SecurityContext의 생명주기

```java
// 1. SecurityContextPersistenceFilter가 SecurityContext 생성
SecurityContext context = SecurityContextHolder.createEmptyContext();

// 2. 인증 필터가 Authentication 객체 설정
Authentication authentication = new UsernamePasswordAuthenticationToken(user, null, authorities);
context.setAuthentication(authentication);
SecurityContextHolder.setContext(context);

// 3. 컨트롤러에서 사용
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();

// 4. 요청 종료 시 SecurityContextPersistenceFilter가 정리
SecurityContextHolder.clearContext();
```

**핵심 포인트:**
- SecurityContext는 요청마다 생성되고 소멸됩니다.
- ThreadLocal을 사용하여 스레드별로 독립적으로 관리됩니다.
- 인증 정보는 SecurityContext를 통해서만 안전하게 전달됩니다.

---

## 커스텀 필터 추가하기

### 1단계: 기본 필터 구현

먼저 간단한 로깅 필터를 만들어 보겠습니다.

[LoggingFilter.java]
```java
public class LoggingFilter extends OncePerRequestFilter {

    private static final Logger log = LoggerFactory.getLogger(LoggingFilter.class);

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {

        log.info("Request URI: {}", request.getRequestURI());
        log.info("Request Method: {}", request.getMethod());

        // 다음 필터로 전달
        filterChain.doFilter(request, response);

        log.info("Response Status: {}", response.getStatus());
    }
}
```

**핵심 포인트:**
- `OncePerRequestFilter`를 상속하면 요청당 한 번만 실행됩니다.
- 포워드나 에러 페이지 이동 시 중복 실행을 방지합니다.
- `doFilterInternal()` 메서드를 오버라이드합니다.

### 2단계: Security 설정에 필터 등록

[SecurityConfig.java]
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .addFilterBefore(new LoggingFilter(), UsernamePasswordAuthenticationFilter.class)
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            );

        return http.build();
    }
}
```

**필터 추가 메서드:**
- `addFilterBefore(filter, class)`: 지정한 필터 앞에 추가
- `addFilterAfter(filter, class)`: 지정한 필터 뒤에 추가
- `addFilterAt(filter, class)`: 지정한 필터와 같은 위치에 추가

---

## 필터 순서와 위치 선택

### 필터 위치 선택 기준

커스텀 필터를 추가할 때는 다음 질문에 답하면서 적절한 위치를 결정합니다:

**질문 1: 이 필터는 인증 정보가 필요한가?**
- **예** → 인증 필터(UsernamePasswordAuthenticationFilter) 이후에 배치
- **아니오** → 인증 필터 이전에 배치

**질문 2: 이 필터는 인증을 직접 수행하는가?**
- **예** → 기존 인증 필터와 같은 위치에 배치
- **아니오** → 적절한 전후 위치에 배치

**질문 3: 이 필터는 모든 요청에 적용되는가?**
- **예** → SecurityContextPersistenceFilter 이후 초반에 배치
- **아니오** → 특정 필터 근처에 배치하고 조건문으로 제어

### 일반적인 커스텀 필터 배치 위치

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        // 1. 요청 로깅, 트레이싱 필터
        .addFilterBefore(requestLoggingFilter, SecurityContextPersistenceFilter.class)

        // 2. JWT 인증 필터 (세션 기반 인증 대체)
        .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)

        // 3. 추가 검증 필터 (인증 이후)
        .addFilterAfter(customValidationFilter, BasicAuthenticationFilter.class)

        // 4. 커스텀 예외 처리
        .addFilterBefore(customExceptionFilter, ExceptionTranslationFilter.class);

    return http.build();
}
```

---

## 실전 구현: JWT 인증 필터

### 요구사항

JWT 기반 인증 시스템을 구현해야 합니다:

1. Authorization 헤더에서 "Bearer {token}" 형식의 토큰을 추출합니다.
2. JWT 토큰의 유효성을 검증합니다 (만료 시간, 서명).
3. 토큰에서 사용자 정보를 추출하여 SecurityContext에 설정합니다.
4. 토큰이 없거나 유효하지 않으면 401 응답을 반환합니다.

### 구현

[JwtAuthenticationFilter.java]
```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtTokenProvider;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {

        // 1. 헤더에서 JWT 토큰 추출
        String token = resolveToken(request);

        try {
            // 2. 토큰 유효성 검증
            if (token != null && jwtTokenProvider.validateToken(token)) {

                // 3. 토큰에서 사용자 정보 추출
                String username = jwtTokenProvider.getUsername(token);

                // 4. UserDetails 조회
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                // 5. Authentication 객체 생성
                Authentication authentication = new UsernamePasswordAuthenticationToken(
                    userDetails,
                    null,
                    userDetails.getAuthorities()
                );

                // 6. SecurityContext에 인증 정보 설정
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        } catch (JwtException e) {
            // 토큰 검증 실패 시 로그만 남기고 계속 진행
            // ExceptionTranslationFilter가 처리하도록 위임
            logger.error("JWT validation failed", e);
        }

        // 7. 다음 필터로 전달
        filterChain.doFilter(request, response);
    }

    private String resolveToken(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");

        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }

        return null;
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        // 공개 API는 필터를 건너뜀
        String path = request.getRequestURI();
        return path.startsWith("/api/public/") || path.startsWith("/api/auth/");
    }
}
```

**핵심 포인트:**
- `@Component`로 등록하여 의존성 주입을 받습니다.
- `shouldNotFilter()`로 특정 경로를 필터링에서 제외합니다.
- 예외가 발생해도 다음 필터로 전달하여 표준 예외 처리를 따릅니다.
- SecurityContext에 설정된 인증 정보는 요청이 끝나면 자동으로 정리됩니다.

### JWT 토큰 제공자 구현

[JwtTokenProvider.java]
```java
@Component
public class JwtTokenProvider {

    @Value("${jwt.secret}")
    private String secretKey;

    @Value("${jwt.expiration}")
    private long validityInMilliseconds;

    private SecretKey key;

    @PostConstruct
    protected void init() {
        this.key = Keys.hmacShaKeyFor(secretKey.getBytes(StandardCharsets.UTF_8));
    }

    /**
     * JWT 토큰 생성
     */
    public String createToken(String username, Collection<? extends GrantedAuthority> authorities) {
        Claims claims = Jwts.claims().setSubject(username);
        claims.put("roles", authorities.stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.toList()));

        Date now = new Date();
        Date validity = new Date(now.getTime() + validityInMilliseconds);

        return Jwts.builder()
            .setClaims(claims)
            .setIssuedAt(now)
            .setExpiration(validity)
            .signWith(key, SignatureAlgorithm.HS256)
            .compact();
    }

    /**
     * 토큰에서 사용자 이름 추출
     */
    public String getUsername(String token) {
        return Jwts.parserBuilder()
            .setSigningKey(key)
            .build()
            .parseClaimsJws(token)
            .getBody()
            .getSubject();
    }

    /**
     * 토큰 유효성 검증
     */
    public boolean validateToken(String token) {
        try {
            Jwts.parserBuilder()
                .setSigningKey(key)
                .build()
                .parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }
}
```

### Security 설정에 필터 등록

[SecurityConfig.java]
```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthenticationFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/api/auth/**", "/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(exception -> exception
                .authenticationEntryPoint(new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED))
            );

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

**설정 설명:**
- `SessionCreationPolicy.STATELESS`: 세션을 사용하지 않습니다 (JWT 방식).
- `csrf().disable()`: REST API에서는 CSRF 보호가 불필요합니다.
- `addFilterBefore()`: JWT 필터를 폼 로그인 필터 앞에 배치합니다.
- `authenticationEntryPoint()`: 인증 실패 시 401 응답을 반환합니다.

---

## 검증

### 테스트 코드

JWT 필터가 올바르게 동작하는지 검증합니다.

[JwtAuthenticationFilterTest.java]
```java
@SpringBootTest
@AutoConfigureMockMvc
class JwtAuthenticationFilterTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private JwtTokenProvider jwtTokenProvider;

    @Test
    void 유효한_JWT_토큰으로_인증_성공() throws Exception {
        // given
        List<GrantedAuthority> authorities = List.of(new SimpleGrantedAuthority("ROLE_USER"));
        String token = jwtTokenProvider.createToken("testuser", authorities);

        // when & then
        mockMvc.perform(get("/api/orders")
                .header("Authorization", "Bearer " + token))
            .andExpect(status().isOk());
    }

    @Test
    void JWT_토큰_없이_요청_시_401_반환() throws Exception {
        // when & then
        mockMvc.perform(get("/api/orders"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    void 잘못된_JWT_토큰으로_요청_시_401_반환() throws Exception {
        // when & then
        mockMvc.perform(get("/api/orders")
                .header("Authorization", "Bearer invalid.token.here"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    void 공개_API는_토큰_없이_접근_가능() throws Exception {
        // when & then
        mockMvc.perform(get("/api/public/products"))
            .andExpect(status().isOk());
    }

    @Test
    void ADMIN_권한이_필요한_API는_USER_권한으로_접근_불가() throws Exception {
        // given
        List<GrantedAuthority> authorities = List.of(new SimpleGrantedAuthority("ROLE_USER"));
        String token = jwtTokenProvider.createToken("testuser", authorities);

        // when & then
        mockMvc.perform(get("/api/admin/users")
                .header("Authorization", "Bearer " + token))
            .andExpect(status().isForbidden());
    }
}
```

### 실행 결과

```
MockHttpServletRequest:
      HTTP Method = GET
      Request URI = /api/orders
       Parameters = {}
          Headers = [Authorization:"Bearer eyJhbGciOiJIUzI1NiJ9..."]

MockHttpServletResponse:
           Status = 200
    Error message = null
          Headers = [Content-Type:"application/json"]

[테스트 통과]
- 유효한_JWT_토큰으로_인증_성공 ✓
- JWT_토큰_없이_요청_시_401_반환 ✓
- 잘못된_JWT_토큰으로_요청_시_401_반환 ✓
- 공개_API는_토큰_없이_접근_가능 ✓
- ADMIN_권한이_필요한_API는_USER_권한으로_접근_불가 ✓
```

---

## 심화: 필터 체인 디버깅

### 필터 순서 확인하기

실제로 어떤 필터들이 어떤 순서로 실행되는지 확인하는 방법입니다.

[FilterChainProxy를 통한 디버깅]
```java
@Component
@RequiredArgsConstructor
public class FilterChainLogger {

    private final FilterChainProxy filterChainProxy;

    @PostConstruct
    public void logFilters() {
        List<SecurityFilterChain> filterChains = filterChainProxy.getFilterChains();

        for (SecurityFilterChain chain : filterChains) {
            List<Filter> filters = chain.getFilters();

            System.out.println("=== Security Filter Chain ===");
            for (int i = 0; i < filters.size(); i++) {
                System.out.printf("%d. %s%n", i + 1, filters.get(i).getClass().getName());
            }
        }
    }
}
```

### 출력 결과

```
=== Security Filter Chain ===
1. org.springframework.security.web.context.SecurityContextPersistenceFilter
2. org.springframework.security.web.header.HeaderWriterFilter
3. org.springframework.security.web.csrf.CsrfFilter
4. org.springframework.security.web.authentication.logout.LogoutFilter
5. com.example.security.JwtAuthenticationFilter  ← 우리가 추가한 필터
6. org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter
7. org.springframework.security.web.savedrequest.RequestCacheAwareFilter
8. org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter
9. org.springframework.security.web.authentication.AnonymousAuthenticationFilter
10. org.springframework.security.web.session.SessionManagementFilter
11. org.springframework.security.web.access.ExceptionTranslationFilter
12. org.springframework.security.web.access.intercept.FilterSecurityInterceptor
```

### 필터 실행 추적하기

각 필터의 실행 시간과 순서를 로깅하는 방법입니다.

[FilterPerformanceLogger.java]
```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class FilterPerformanceLogger extends OncePerRequestFilter {

    private static final Logger log = LoggerFactory.getLogger(FilterPerformanceLogger.class);

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {

        long startTime = System.currentTimeMillis();

        try {
            filterChain.doFilter(request, response);
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            log.info("Request to {} took {} ms", request.getRequestURI(), duration);
        }
    }
}
```

### 필터 체인 중단 시나리오

특정 조건에서 필터 체인을 중단하고 즉시 응답을 반환하는 방법입니다.

```java
@Override
protected void doFilterInternal(HttpServletRequest request,
                                HttpServletResponse response,
                                FilterChain filterChain) throws ServletException, IOException {

    String apiKey = request.getHeader("X-API-Key");

    if (apiKey == null || !isValidApiKey(apiKey)) {
        // 필터 체인을 중단하고 즉시 응답 반환
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.getWriter().write("{\"error\": \"Invalid API Key\"}");
        return; // chain.doFilter() 호출하지 않음
    }

    // API 키가 유효하면 다음 필터로 진행
    filterChain.doFilter(request, response);
}
```

**주의사항:**
- `chain.doFilter()`를 호출하지 않으면 요청이 더 이상 진행되지 않습니다.
- 응답을 직접 작성한 후에는 반드시 return으로 메서드를 종료해야 합니다.
- 이 방식은 조기 검증(rate limiting, API key 검증 등)에 유용합니다.

---

## 정리

이 글에서 다룬 내용을 정리하면 다음과 같습니다:

1. **Spring Security의 필터 체인 구조**: 15개 이상의 필터가 순차적으로 실행되며 각각 명확한 역할을 수행합니다.
2. **필터 추가 방법**: `addFilterBefore()`, `addFilterAfter()`, `addFilterAt()`을 사용하여 적절한 위치에 배치합니다.
3. **JWT 인증 필터 구현**: Authorization 헤더에서 토큰을 추출하고 검증하여 SecurityContext에 인증 정보를 설정합니다.
4. **필터 위치 선택**: 인증 정보 필요 여부, 인증 수행 여부, 적용 범위를 고려하여 결정합니다.
5. **예외 처리**: 필터에서 발생한 예외는 ExceptionTranslationFilter가 처리하도록 위임합니다.

**핵심:** Spring Security 필터 체인의 순서와 각 필터의 역할을 이해하면, 커스텀 필터를 안전하고 효과적으로 통합할 수 있습니다.

---

## 이 글에서 다루지 않은 내용

- OAuth 2.0 및 소셜 로그인 통합
- 멀티 테넌트 환경에서의 필터 구성
- Spring Security의 메서드 레벨 시큐리티 (@PreAuthorize, @Secured)
- CORS 설정과 필터의 상호작용
- WebFlux 환경에서의 Security 필터

이 주제들은 별도의 글에서 다루도록 하겠습니다.

---

## 참고자료

- [Spring Security Reference - Servlet Security: The Big Picture](https://docs.spring.io/spring-security/reference/servlet/architecture.html)
- [Spring Security Reference - Security Filters](https://docs.spring.io/spring-security/reference/servlet/architecture.html#servlet-security-filters)
- [Baeldung - Spring Security Filter Chain](https://www.baeldung.com/spring-security-custom-filter)
- [JWT.io - JWT Introduction](https://jwt.io/introduction)
- [Spring Security GitHub - FilterChainProxy.java](https://github.com/spring-projects/spring-security/blob/main/web/src/main/java/org/springframework/security/web/FilterChainProxy.java)

---

**다음 학습 추천:**

이 글을 이해했다면 다음 주제를 학습해보세요:

1. Spring Security의 인증 아키텍처 (AuthenticationManager, ProviderManager)
2. Spring Security Method Security (메서드 레벨 권한 제어)
3. OAuth 2.0 Resource Server 구현하기
