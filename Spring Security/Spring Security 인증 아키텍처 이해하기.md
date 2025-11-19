# Spring Security 인증 아키텍처 이해하기

> Spring Security의 핵심 인증 구조와 컴포넌트 간 협력 방식

## 들어가며

Spring Security는 엔터프라이즈 애플리케이션의 인증과 인가를 담당하는 강력한 프레임워크입니다. 하지만 처음 접하면 수많은 컴포넌트와 인터페이스 때문에 복잡하게 느껴집니다.

나 역시 처음엔 "왜 이렇게 많은 클래스가 필요한가?"라는 의문을 가졌습니다. 하지만 각 컴포넌트의 역할과 협력 구조를 이해하고 나니, 이 설계가 얼마나 유연하고 확장 가능한지 깨달았습니다.

이 문서는 Spring Security의 **인증 아키텍처(Authentication Architecture)**를 다룹니다. 주요 목표는 다음과 같습니다.

- **핵심 컴포넌트 이해**: SecurityContextHolder, Authentication, AuthenticationManager 등 각 구성 요소의 역할 파악
- **협력 구조 파악**: 컴포넌트들이 어떻게 상호작용하며 인증 프로세스를 완성하는지 이해
- **실무 적용**: 커스텀 인증 로직을 구현할 때 어느 지점을 확장해야 하는지 판단

---

## 1. 인증 아키텍처 전체 흐름

Spring Security의 인증 프로세스는 다음과 같은 흐름으로 동작합니다.

```
HTTP 요청
  ↓
① AuthenticationFilter (자격 증명 추출)
  ↓
② UsernamePasswordAuthenticationToken 생성
  ↓
③ AuthenticationManager에 인증 위임
  ↓
④ AuthenticationProvider(s) 호출 (실제 인증 수행)
  ↓
⑤ UserDetailsService로 사용자 조회
  ↓
⑥ UserDetails 반환
  ↓
⑦ AuthenticationProvider가 인증 성공 여부 판단
  ↓
⑧ 인증된 Authentication 객체 반환
  ↓
⑨ AuthenticationManager가 결과를 필터에 반환
  ↓
⑩ SecurityContextHolder에 Authentication 저장
```

이 흐름을 구성하는 핵심 컴포넌트들을 하나씩 살펴봅니다.

---

## 2. SecurityContextHolder: 인증 정보의 보관소

### 역할

SecurityContextHolder는 "누가 인증되었는가?"라는 정보를 저장하는 중앙 저장소입니다. Spring Security는 이곳에서 현재 인증된 사용자 정보를 조회합니다.

### 동작 방식

SecurityContextHolder는 기본적으로 **ThreadLocal**을 사용하여 인증 정보를 저장합니다. 덕분에 같은 스레드 내의 모든 메서드에서 명시적으로 전달하지 않아도 인증 정보에 접근할 수 있습니다.

```java
// SecurityContext 생성
SecurityContext context = SecurityContextHolder.createEmptyContext();

// Authentication 객체 생성 및 설정
Authentication authentication =
    new TestingAuthenticationToken("user", "password", "ROLE_USER");
context.setAuthentication(authentication);

// SecurityContextHolder에 저장
SecurityContextHolder.setContext(context);
```

**구조:**

```
SecurityContextHolder
  └── SecurityContext
        └── Authentication
              ├── principal (UserDetails)
              ├── credentials (비밀번호)
              └── authorities (권한 목록)
```

### 저장 전략

SecurityContextHolder는 세 가지 저장 전략을 지원합니다.

| 전략 | 설명 | 사용 시나리오 |
|-----|------|-------------|
| **MODE_THREADLOCAL** | 각 스레드마다 별도의 SecurityContext 유지 | 일반적인 웹 애플리케이션 (기본값) |
| **MODE_INHERITABLETHREADLOCAL** | 자식 스레드가 부모 스레드의 SecurityContext 상속 | 비동기 작업 처리 |
| **MODE_GLOBAL** | 애플리케이션 전역에서 하나의 SecurityContext 공유 | 독립 실행형 애플리케이션 |

### 현재 인증 정보 조회

서비스 레이어나 컨트롤러에서 현재 인증된 사용자 정보를 가져오는 방식입니다.

```java
@Service
public class UserService {

    public String getCurrentUsername() {
        SecurityContext context = SecurityContextHolder.getContext();
        Authentication authentication = context.getAuthentication();

        if (authentication == null || !authentication.isAuthenticated()) {
            return "anonymous";
        }

        return authentication.getName();
    }
}
```

---

## 3. SecurityContext와 Authentication

### SecurityContext

SecurityContext는 SecurityContextHolder로부터 얻을 수 있는 객체로, **Authentication 객체를 담는 컨테이너** 역할을 합니다.

```java
public interface SecurityContext extends Serializable {
    Authentication getAuthentication();
    void setAuthentication(Authentication authentication);
}
```

SecurityContext 자체는 단순한 컨테이너이며, 실제 인증 정보는 Authentication 객체에 담겨 있습니다.

### Authentication 인터페이스

Authentication은 두 가지 목적으로 사용됩니다.

1. **인증 전**: AuthenticationManager의 입력으로 사용자가 제공한 자격 증명 전달
2. **인증 후**: SecurityContext에 저장되어 현재 인증된 사용자 정보 제공

**주요 메서드:**

```java
public interface Authentication extends Principal, Serializable {

    Collection<? extends GrantedAuthority> getAuthorities();

    Object getCredentials();

    Object getDetails();

    Object getPrincipal();

    boolean isAuthenticated();

    void setAuthenticated(boolean isAuthenticated);
}
```

**각 요소의 의미:**

| 요소 | 설명 | 예시 |
|-----|------|-----|
| **principal** | 사용자 식별자 | UserDetails 객체 또는 username 문자열 |
| **credentials** | 인증에 사용된 비밀번호 | 인증 후에는 보안상 삭제됨 |
| **authorities** | 사용자에게 부여된 권한 목록 | `ROLE_USER`, `ROLE_ADMIN` 등 |
| **details** | 추가 정보 | IP 주소, 세션 ID 등 |
| **authenticated** | 인증 완료 여부 | true/false |

**인증 전 Authentication 객체:**

```java
Authentication authRequest =
    new UsernamePasswordAuthenticationToken("user", "password");
```

이 시점에는 `isAuthenticated() = false`이며, 단순히 사용자가 입력한 자격 증명을 담고 있습니다.

**인증 후 Authentication 객체:**

```java
UserDetails userDetails = userDetailsService.loadUserByUsername("user");
Authentication authResult =
    new UsernamePasswordAuthenticationToken(
        userDetails,
        null,
        userDetails.getAuthorities()
    );
```

인증이 완료된 후에는 `principal`에 UserDetails 객체가 담기고, `credentials`는 보안상 null로 설정됩니다.

---

## 4. GrantedAuthority: 권한의 표현

### 역할

GrantedAuthority는 사용자에게 부여된 **고수준 권한**을 나타냅니다. 일반적으로 역할(Role)이나 범위(Scope)를 의미합니다.

### 특징

- **애플리케이션 전역 권한**: 특정 도메인 객체에 국한되지 않고, 애플리케이션 전체에 적용되는 권한
- **문자열 기반**: 대부분의 구현체는 권한을 문자열로 표현

```java
public interface GrantedAuthority extends Serializable {
    String getAuthority();
}
```

### 일반적인 사용 예시

**역할 기반 권한:**

```java
List<GrantedAuthority> authorities = List.of(
    new SimpleGrantedAuthority("ROLE_USER"),
    new SimpleGrantedAuthority("ROLE_ADMIN")
);
```

Spring Security의 규칙상 역할은 `ROLE_` 접두사를 사용합니다.

**OAuth2 스코프:**

```java
List<GrantedAuthority> authorities = List.of(
    new SimpleGrantedAuthority("SCOPE_read"),
    new SimpleGrantedAuthority("SCOPE_write")
);
```

### UserDetailsService와의 관계

GrantedAuthority는 일반적으로 UserDetailsService가 사용자 정보를 로드할 때 함께 제공됩니다.

```java
@Override
public UserDetails loadUserByUsername(String username) {
    User user = userRepository.findByUsername(username)
        .orElseThrow(() -> new UsernameNotFoundException("User not found"));

    List<GrantedAuthority> authorities = user.getRoles().stream()
        .map(role -> new SimpleGrantedAuthority("ROLE_" + role.getName()))
        .collect(Collectors.toList());

    return new org.springframework.security.core.userdetails.User(
        user.getUsername(),
        user.getPassword(),
        authorities
    );
}
```

---

## 5. AuthenticationManager: 인증의 진입점

### 역할

AuthenticationManager는 "Spring Security의 필터가 인증을 수행하는 방식을 정의하는 API"입니다. 인터페이스 자체는 매우 단순합니다.

```java
public interface AuthenticationManager {
    Authentication authenticate(Authentication authentication)
        throws AuthenticationException;
}
```

### 책임

AuthenticationManager는 인증 요청을 받아 처리하고, 세 가지 결과 중 하나를 반환합니다.

1. **인증 성공**: 인증된 Authentication 객체 반환 (`isAuthenticated() = true`)
2. **인증 실패**: AuthenticationException 예외 발생
3. **판단 불가**: null 반환 (다음 AuthenticationProvider에게 위임)

### 필터와의 통합

AuthenticationManager는 AbstractAuthenticationProcessingFilter 같은 인증 필터에서 호출됩니다.

```java
public class CustomAuthenticationFilter extends AbstractAuthenticationProcessingFilter {

    private final AuthenticationManager authenticationManager;

    @Override
    public Authentication attemptAuthentication(
        HttpServletRequest request,
        HttpServletResponse response
    ) {
        String username = request.getParameter("username");
        String password = request.getParameter("password");

        Authentication authRequest =
            new UsernamePasswordAuthenticationToken(username, password);

        return authenticationManager.authenticate(authRequest);
    }
}
```

---

## 6. ProviderManager: AuthenticationManager의 구현체

### 역할

ProviderManager는 AuthenticationManager의 가장 일반적인 구현체로, **여러 AuthenticationProvider에게 인증을 위임**합니다.

### 동작 방식

ProviderManager는 AuthenticationProvider 목록을 순회하며 각 Provider에게 인증을 시도합니다.

```java
public class ProviderManager implements AuthenticationManager {

    private List<AuthenticationProvider> providers;
    private AuthenticationManager parent;

    @Override
    public Authentication authenticate(Authentication authentication) {
        for (AuthenticationProvider provider : providers) {
            if (!provider.supports(authentication.getClass())) {
                continue;
            }

            try {
                Authentication result = provider.authenticate(authentication);
                if (result != null) {
                    return result;
                }
            } catch (AuthenticationException e) {
                lastException = e;
            }
        }

        if (parent != null) {
            return parent.authenticate(authentication);
        }

        throw new ProviderNotFoundException("No provider found");
    }
}
```

**처리 흐름:**

```
ProviderManager
  ├── AuthenticationProvider 1 (예: DaoAuthenticationProvider)
  │     └── supports(UsernamePasswordAuthenticationToken)? → 인증 시도
  ├── AuthenticationProvider 2 (예: JwtAuthenticationProvider)
  │     └── supports(JwtAuthenticationToken)? → 건너뜀
  └── parent AuthenticationManager (선택사항)
```

### 부모 AuthenticationManager

ProviderManager는 선택적으로 부모 AuthenticationManager를 가질 수 있습니다. 모든 AuthenticationProvider가 인증을 처리하지 못했을 때, 부모에게 위임합니다.

이 구조는 여러 SecurityFilterChain이 공통 인증 로직을 공유할 때 유용합니다.

```
SecurityFilterChain 1
  └── ProviderManager 1
        ├── OAuth2Provider
        └── parent: 공통 ProviderManager
              └── DaoAuthenticationProvider

SecurityFilterChain 2
  └── ProviderManager 2
        ├── JwtProvider
        └── parent: 공통 ProviderManager
              └── DaoAuthenticationProvider
```

### 자격 증명 삭제

ProviderManager는 보안을 위해 인증 성공 후 Authentication 객체에서 **민감한 자격 증명(비밀번호)을 자동으로 삭제**합니다.

```java
if (eraseCredentialsAfterAuthentication) {
    ((CredentialsContainer) result).eraseCredentials();
}
```

이 동작은 `eraseCredentialsAfterAuthentication` 속성으로 제어할 수 있습니다. 캐시를 사용하는 경우 주의가 필요합니다.

---

## 7. AuthenticationProvider: 실제 인증 수행

### 역할

AuthenticationProvider는 **특정 유형의 인증을 담당**하는 구체적인 구현체입니다.

```java
public interface AuthenticationProvider {

    Authentication authenticate(Authentication authentication)
        throws AuthenticationException;

    boolean supports(Class<?> authentication);
}
```

### 주요 메서드

**1. supports(Class<?> authentication)**

이 Provider가 특정 Authentication 타입을 처리할 수 있는지 판단합니다.

```java
@Override
public boolean supports(Class<?> authentication) {
    return UsernamePasswordAuthenticationToken.class
        .isAssignableFrom(authentication);
}
```

**2. authenticate(Authentication authentication)**

실제 인증 로직을 수행합니다.

```java
@Override
public Authentication authenticate(Authentication authentication) {
    String username = authentication.getName();
    String password = (String) authentication.getCredentials();

    UserDetails userDetails = userDetailsService.loadUserByUsername(username);

    if (!passwordEncoder.matches(password, userDetails.getPassword())) {
        throw new BadCredentialsException("Invalid password");
    }

    return new UsernamePasswordAuthenticationToken(
        userDetails,
        null,
        userDetails.getAuthorities()
    );
}
```

### 대표적인 구현체

| 구현체 | 인증 방식 | 처리하는 Authentication 타입 |
|--------|----------|----------------------------|
| **DaoAuthenticationProvider** | 사용자명/비밀번호 | UsernamePasswordAuthenticationToken |
| **JwtAuthenticationProvider** | JWT 토큰 검증 | JwtAuthenticationToken |
| **OAuth2LoginAuthenticationProvider** | OAuth2 로그인 | OAuth2LoginAuthenticationToken |
| **AnonymousAuthenticationProvider** | 익명 사용자 | AnonymousAuthenticationToken |

### 여러 Provider의 조합

ProviderManager가 여러 AuthenticationProvider를 가질 수 있기 때문에, 하나의 애플리케이션에서 여러 인증 메커니즘을 동시에 지원할 수 있습니다.

```java
@Bean
public AuthenticationManager authenticationManager(
    AuthenticationConfiguration config
) throws Exception {
    return config.getAuthenticationManager();
}

@Bean
public DaoAuthenticationProvider daoAuthenticationProvider() {
    DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
    provider.setUserDetailsService(userDetailsService);
    provider.setPasswordEncoder(passwordEncoder());
    return provider;
}

@Bean
public JwtAuthenticationProvider jwtAuthenticationProvider() {
    return new JwtAuthenticationProvider(jwtDecoder());
}
```

Spring Boot는 이 Provider들을 자동으로 ProviderManager에 등록합니다.

---

## 8. UserDetailsService: 사용자 정보 로드

### 역할

UserDetailsService는 **사용자명을 기반으로 사용자 정보를 조회**하는 인터페이스입니다.

```java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username)
        throws UsernameNotFoundException;
}
```

### 구현 예시

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() ->
                new UsernameNotFoundException("User not found: " + username)
            );

        return org.springframework.security.core.userdetails.User.builder()
            .username(user.getUsername())
            .password(user.getPassword())
            .authorities(user.getRoles().stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role.getName()))
                .collect(Collectors.toList())
            )
            .accountExpired(!user.isActive())
            .accountLocked(user.isLocked())
            .credentialsExpired(user.isPasswordExpired())
            .disabled(!user.isEnabled())
            .build();
    }
}
```

### DaoAuthenticationProvider와의 관계

DaoAuthenticationProvider는 내부적으로 UserDetailsService를 호출하여 사용자 정보를 조회합니다.

```
UsernamePasswordAuthenticationToken
  ↓
DaoAuthenticationProvider
  ↓
UserDetailsService.loadUserByUsername()
  ↓
UserDetails 반환
  ↓
비밀번호 검증 (PasswordEncoder)
  ↓
인증된 Authentication 반환
```

---

## 9. AuthenticationEntryPoint: 인증 실패 시 처리

### 역할

AuthenticationEntryPoint는 **미인증 요청에 대해 클라이언트로부터 자격 증명을 요청하는 HTTP 응답을 보냅니다.**

```java
public interface AuthenticationEntryPoint {
    void commence(
        HttpServletRequest request,
        HttpServletResponse response,
        AuthenticationException authException
    ) throws IOException, ServletException;
}
```

### 사용 시나리오

| 시나리오 | 구현체 | 동작 |
|---------|--------|-----|
| **폼 로그인** | LoginUrlAuthenticationEntryPoint | 로그인 페이지로 리다이렉트 |
| **HTTP Basic** | BasicAuthenticationEntryPoint | WWW-Authenticate 헤더 전송 |
| **REST API** | HttpStatusEntryPoint | 401 Unauthorized 응답 |

### 구현 예시

```java
public class RestAuthenticationEntryPoint implements AuthenticationEntryPoint {

    @Override
    public void commence(
        HttpServletRequest request,
        HttpServletResponse response,
        AuthenticationException authException
    ) throws IOException {
        response.setContentType("application/json");
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);

        Map<String, String> error = Map.of(
            "error", "Unauthorized",
            "message", "Authentication required"
        );

        response.getWriter().write(
            new ObjectMapper().writeValueAsString(error)
        );
    }
}
```

### Spring Security 설정

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .exceptionHandling(exception -> exception
            .authenticationEntryPoint(new RestAuthenticationEntryPoint())
        );

    return http.build();
}
```

---

## 10. AbstractAuthenticationProcessingFilter: 인증 필터의 기본 구조

### 역할

AbstractAuthenticationProcessingFilter는 **사용자 자격 증명을 인증하는 기본 Filter 클래스**입니다. UsernamePasswordAuthenticationFilter 같은 구체적인 인증 필터들이 이를 상속합니다.

### 인증 프로세스 5단계

**1단계: 자격 증명 수집 및 Authentication 생성**

사용자가 제출한 자격 증명으로부터 Authentication 객체를 생성합니다.

```java
@Override
public Authentication attemptAuthentication(
    HttpServletRequest request,
    HttpServletResponse response
) {
    String username = request.getParameter("username");
    String password = request.getParameter("password");

    return new UsernamePasswordAuthenticationToken(username, password);
}
```

**2단계: AuthenticationManager 호출**

생성된 Authentication을 AuthenticationManager에 전달하여 인증을 시도합니다.

```java
Authentication authResult =
    authenticationManager.authenticate(authRequest);
```

**3단계: 인증 실패 처리**

인증이 실패하면 다음 작업을 수행합니다.

```
① SecurityContextHolder 초기화
② RememberMeServices.loginFail() 호출
③ AuthenticationFailureHandler 실행
```

```java
protected void unsuccessfulAuthentication(
    HttpServletRequest request,
    HttpServletResponse response,
    AuthenticationException failed
) {
    SecurityContextHolder.clearContext();

    rememberMeServices.loginFail(request, response);

    failureHandler.onAuthenticationFailure(request, response, failed);
}
```

**4단계: 인증 성공 처리**

인증이 성공하면 다음 작업을 수행합니다.

```
① SessionAuthenticationStrategy에 새 로그인 알림
② SecurityContextHolder에 Authentication 설정
③ SecurityContextRepository에 저장
④ RememberMeServices.loginSuccess() 호출
⑤ InteractiveAuthenticationSuccessEvent 발행
⑥ AuthenticationSuccessHandler 실행
```

```java
protected void successfulAuthentication(
    HttpServletRequest request,
    HttpServletResponse response,
    FilterChain chain,
    Authentication authResult
) {
    SecurityContext context = SecurityContextHolder.createEmptyContext();
    context.setAuthentication(authResult);
    SecurityContextHolder.setContext(context);

    securityContextRepository.saveContext(context, request, response);

    rememberMeServices.loginSuccess(request, response, authResult);

    eventPublisher.publishEvent(
        new InteractiveAuthenticationSuccessEvent(authResult, this.getClass())
    );

    successHandler.onAuthenticationSuccess(request, response, authResult);
}
```

**5단계: 후속 처리**

성공 핸들러가 실행되어 리다이렉트 또는 응답을 처리합니다.

---

## 11. 전체 아키텍처 다이어그램 분석

제공된 다이어그램을 기반으로 각 단계를 분석합니다.

```
① HTTP Request → AuthenticationFilter
② AuthenticationFilter → UsernamePasswordAuthenticationToken 생성
③ AuthenticationFilter → AuthenticationManager 호출
④ AuthenticationManager → AuthenticationProvider(s) 위임
⑤ AuthenticationProvider → UserDetailsService 호출
⑥ UserDetailsService → UserDetails 반환
⑦ AuthenticationProvider → UserDetails 검증 후 인증 완료
⑧ AuthenticationProvider → 인증된 Authentication 반환
⑨ AuthenticationManager → AuthenticationFilter에 결과 반환
⑩ AuthenticationFilter → SecurityContextHolder에 저장
```

### 각 단계별 상세 설명

**① HTTP 요청 도착**

클라이언트가 로그인 요청을 보냅니다.

```
POST /login
Content-Type: application/x-www-form-urlencoded

username=user&password=1234
```

**② UsernamePasswordAuthenticationToken 생성**

AuthenticationFilter가 요청에서 자격 증명을 추출하여 Authentication 객체를 생성합니다.

```java
UsernamePasswordAuthenticationToken authRequest =
    new UsernamePasswordAuthenticationToken("user", "1234");
```

이 시점의 Authentication은 `isAuthenticated() = false`입니다.

**③ AuthenticationManager 호출**

```java
Authentication authResult =
    authenticationManager.authenticate(authRequest);
```

**④ AuthenticationProvider(s)에 위임**

ProviderManager는 등록된 Provider 중 적절한 Provider를 찾습니다.

```java
for (AuthenticationProvider provider : providers) {
    if (provider.supports(authRequest.getClass())) {
        return provider.authenticate(authRequest);
    }
}
```

DaoAuthenticationProvider가 선택됩니다.

**⑤ UserDetailsService 호출**

DaoAuthenticationProvider는 UserDetailsService를 호출하여 사용자 정보를 조회합니다.

```java
UserDetails userDetails =
    userDetailsService.loadUserByUsername("user");
```

**⑥ UserDetails 반환**

데이터베이스에서 조회한 사용자 정보를 UserDetails 객체로 반환합니다.

```java
return User.builder()
    .username("user")
    .password("{bcrypt}$2a$10$...")
    .authorities("ROLE_USER")
    .build();
```

**⑦ AuthenticationProvider 검증**

DaoAuthenticationProvider는 입력된 비밀번호와 저장된 비밀번호를 비교합니다.

```java
if (!passwordEncoder.matches(rawPassword, encodedPassword)) {
    throw new BadCredentialsException("Invalid password");
}
```

검증이 성공하면 인증된 Authentication 객체를 생성합니다.

```java
return new UsernamePasswordAuthenticationToken(
    userDetails,
    null,
    userDetails.getAuthorities()
);
```

**⑧ AuthenticationProvider → AuthenticationManager**

인증된 Authentication 객체가 ProviderManager를 거쳐 필터로 반환됩니다.

**⑨ AuthenticationManager → AuthenticationFilter**

필터가 인증 결과를 받습니다.

**⑩ SecurityContextHolder에 저장**

인증된 Authentication을 SecurityContextHolder에 저장하여 애플리케이션 전역에서 접근 가능하게 합니다.

```java
SecurityContext context = SecurityContextHolder.createEmptyContext();
context.setAuthentication(authResult);
SecurityContextHolder.setContext(context);
```

---

## 12. 실무 적용: 커스텀 인증 구현

### 시나리오: JWT 기반 인증 추가

기존의 폼 로그인에 더해 JWT 토큰 기반 인증을 추가하는 시나리오입니다.

**1단계: JwtAuthenticationToken 정의**

```java
public class JwtAuthenticationToken extends AbstractAuthenticationToken {

    private final String token;
    private final Object principal;

    public JwtAuthenticationToken(String token) {
        super(null);
        this.token = token;
        this.principal = null;
        setAuthenticated(false);
    }

    public JwtAuthenticationToken(
        Object principal,
        Collection<? extends GrantedAuthority> authorities
    ) {
        super(authorities);
        this.token = null;
        this.principal = principal;
        setAuthenticated(true);
    }

    @Override
    public Object getCredentials() {
        return token;
    }

    @Override
    public Object getPrincipal() {
        return principal;
    }
}
```

**2단계: JwtAuthenticationProvider 구현**

```java
@Component
public class JwtAuthenticationProvider implements AuthenticationProvider {

    private final JwtDecoder jwtDecoder;
    private final UserDetailsService userDetailsService;

    @Override
    public Authentication authenticate(Authentication authentication) {
        JwtAuthenticationToken jwtAuth = (JwtAuthenticationToken) authentication;
        String token = (String) jwtAuth.getCredentials();

        Jwt jwt = jwtDecoder.decode(token);
        String username = jwt.getSubject();

        UserDetails userDetails = userDetailsService.loadUserByUsername(username);

        return new JwtAuthenticationToken(userDetails, userDetails.getAuthorities());
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return JwtAuthenticationToken.class.isAssignableFrom(authentication);
    }
}
```

**3단계: JwtAuthenticationFilter 구현**

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final AuthenticationManager authenticationManager;

    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain
    ) throws ServletException, IOException {

        String header = request.getHeader("Authorization");

        if (header == null || !header.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        String token = header.substring(7);
        JwtAuthenticationToken authRequest = new JwtAuthenticationToken(token);

        try {
            Authentication authResult =
                authenticationManager.authenticate(authRequest);

            SecurityContextHolder.getContext().setAuthentication(authResult);
        } catch (AuthenticationException e) {
            SecurityContextHolder.clearContext();
        }

        filterChain.doFilter(request, response);
    }
}
```

**4단계: SecurityFilterChain 설정**

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/public/**").permitAll()
            .anyRequest().authenticated()
        )
        .addFilterBefore(
            jwtAuthenticationFilter(),
            UsernamePasswordAuthenticationFilter.class
        );

    return http.build();
}

@Bean
public AuthenticationManager authenticationManager(
    AuthenticationConfiguration config
) throws Exception {
    return config.getAuthenticationManager();
}
```

이렇게 하면 기존 폼 로그인 인증과 JWT 인증이 동시에 동작합니다.

```
ProviderManager
  ├── DaoAuthenticationProvider (폼 로그인)
  └── JwtAuthenticationProvider (JWT 인증)
```

---

## 13. 실무 관점의 주의사항

### 1. ThreadLocal의 정리

SecurityContextHolder는 기본적으로 ThreadLocal을 사용합니다. 요청 처리가 끝난 후 반드시 정리해야 메모리 누수를 방지할 수 있습니다.

Spring Security의 FilterChainProxy가 자동으로 정리하므로, 일반적인 웹 애플리케이션에서는 신경 쓸 필요가 없습니다. 하지만 비동기 작업이나 스케줄러에서는 주의가 필요합니다.

```java
@Scheduled(fixedRate = 60000)
public void scheduledTask() {
    try {
        SecurityContextHolder.setContext(...);

    } finally {
        SecurityContextHolder.clearContext();
    }
}
```

### 2. 자격 증명 삭제 정책

ProviderManager는 기본적으로 인증 성공 후 비밀번호를 삭제합니다. 캐시를 사용하는 경우 이 동작을 비활성화해야 할 수 있습니다.

```java
@Bean
public ProviderManager authenticationManager(
    List<AuthenticationProvider> providers
) {
    ProviderManager manager = new ProviderManager(providers);
    manager.setEraseCredentialsAfterAuthentication(false);
    return manager;
}
```

### 3. 다중 SecurityFilterChain 구성

하나의 애플리케이션에 여러 SecurityFilterChain을 구성할 때, 각 체인의 순서와 매칭 규칙을 명확히 해야 합니다.

```java
@Bean
@Order(1)
public SecurityFilterChain apiSecurityFilterChain(HttpSecurity http) throws Exception {
    http
        .securityMatcher("/api/**")
        .authorizeHttpRequests(auth -> auth
            .anyRequest().authenticated()
        )
        .oauth2ResourceServer(oauth2 -> oauth2.jwt());

    return http.build();
}

@Bean
@Order(2)
public SecurityFilterChain webSecurityFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .anyRequest().authenticated()
        )
        .formLogin(Customizer.withDefaults());

    return http.build();
}
```

`@Order` 어노테이션으로 우선순위를 지정합니다. 숫자가 낮을수록 먼저 평가됩니다.

### 4. 디버깅 팁

인증 문제를 디버깅할 때는 다음 로그 레벨을 활성화하면 유용합니다.

```properties
logging.level.org.springframework.security=DEBUG
```

각 필터의 실행 순서와 AuthenticationProvider의 호출 과정을 상세히 확인할 수 있습니다.

---

## 정리

Spring Security의 인증 아키텍처는 명확한 책임 분리와 유연한 확장성을 제공합니다.

**핵심 컴포넌트:**

- **SecurityContextHolder**: 인증 정보를 저장하는 중앙 저장소
- **Authentication**: 인증 전후의 사용자 정보를 담는 객체
- **AuthenticationManager**: 인증의 진입점 (인터페이스)
- **ProviderManager**: AuthenticationManager의 구현체로, 여러 Provider에게 위임
- **AuthenticationProvider**: 실제 인증 로직을 수행하는 구현체
- **UserDetailsService**: 사용자 정보를 조회하는 인터페이스
- **AuthenticationEntryPoint**: 미인증 요청에 대한 처리

**협력 구조:**

```
요청 → Filter → AuthenticationManager → ProviderManager
→ AuthenticationProvider → UserDetailsService → 검증
→ SecurityContextHolder 저장
```

각 컴포넌트는 단일 책임 원칙을 따르며, 인터페이스 기반 설계로 확장성을 보장합니다. 커스텀 인증 로직을 추가하려면 AuthenticationProvider를 구현하거나, 필터를 추가하면 됩니다.

중요한 건 **각 컴포넌트의 역할과 책임을 이해하는 것**입니다. 그러면 복잡한 인증 요구사항도 Spring Security의 아키텍처 안에서 자연스럽게 해결할 수 있습니다.

---

## 참고자료

- [Spring Security Reference - Authentication Architecture](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html)
- [Spring Security - SecurityContextHolder](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-securitycontextholder)
- [Spring Security - AuthenticationManager](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-authenticationmanager)
