# Spring Security Servlet 아키텍처, 한 번은 구조부터 정리해보기

> “필터 하나 붙였을 뿐인데, 로그에는 필터가 열몇 개가 찍힌다.”

## 0. 필터 로그를 보고 멘붕이 왔던 날

Spring Security를 처음 붙였을 때 나는 아주 순진하게 생각했다.

> “`spring-boot-starter-security` 하나 넣고, `http.formLogin()` 정도만 설정하면 끝이겠지.”

그러다 디버깅을 하려고 로그 레벨을 DEBUG로 올려본 순간, 화면에 이런 로그가 쏟아졌다.

```text
Will secure any request with [
  SecurityContextHolderFilter,
  HeaderWriterFilter,
  CsrfFilter,
  LogoutFilter,
  UsernamePasswordAuthenticationFilter,
  BasicAuthenticationFilter,
  RequestCacheAwareFilter,
  SecurityContextHolderAwareRequestFilter,
  AnonymousAuthenticationFilter,
  SessionManagementFilter,
  ExceptionTranslationFilter,
  AuthorizationFilter
]
```

“아니, 나는 분명히 설정 파일에 필터를 하나도 안 만들었는데…  
이 수많은 필터들은 도대체 어디에서 튀어나온 걸까?”

게다가 공식 문서를 열어보면 처음부터 이런 단어들이 쏟아진다.

- `DelegatingFilterProxy`  
- `FilterChainProxy`  
- `SecurityFilterChain`

**“Servlet Filter 기반으로 동작합니다.”** 라는 문장은 이해했지만,  
**“그래서 내 요청이 실제로 어떤 경로를 타고 가는지”**는 전혀 감이 오지 않았다.

이 글은 그때의 혼란을 정리하기 위해 쓴 글이다.

Spring Security 공식 문서의 *Servlet Architecture* 섹션을 따라가면서,

- 요청이 들어와서 컨트롤러에 도달하기까지  
- Servlet 컨테이너, DelegatingFilterProxy, FilterChainProxy, SecurityFilterChain이  
  어떻게 협력하는지

를 “그림”과 “이야기” 중심으로 정리해본다.

인증 아키텍처(Provider, AuthenticationManager 등)까지 깊게 들어가는 이야기는  
나중에 따로 쓴 **“Spring Security 인증 아키텍처, 한 번은 끝까지 따라가보기”** 글에서 다루고,  
이번 글에서는 먼저 **“Servlet 레벨에서의 전체 구조”**에 집중해보자.

---

## 1. 왜 Servlet 아키텍처까지 알아야 할까?

처음에는 이런 생각이 들었다.

> “인증/인가만 잘 되면 되는 거 아닌가? 굳이 Servlet 필터 체인 구조까지 알아야 할까?”

하지만 실무에서 몇 번 부딪치고 나니, 이 구조를 모르면 생기는 문제가 너무 많았다.

### 1) 필터가 어디서 어떻게 끼어드는지 모르면, 디버깅이 안 된다

어느 날이었다.  
분명히 컨트롤러에서 200을 내려보냈다고 생각했는데,  
클라이언트에서는 302 리다이렉트나 403 Forbidden이 왔다.

로그를 열어보면 모든 일은 **컨트롤러 도달 전에 필터에서 벌어지고 있었다.**

- 세션이 만료되면 로그인 페이지로 리다이렉트  
- 인증이 안 된 요청이면 401/403 처리  
- CSRF 토큰이 맞지 않으면 403

즉, **“컨트롤러에 도달하기도 전에 이미 승부가 나버린다.”**

이 시점에서야 인정했다.

> “Spring Security를 쓴다는 건,  
>  ‘Servlet Filter 체인 위에 얹힌 보안 아키텍처’를 쓴다는 뜻이다.”  
>  **“그러면 필터 구조부터 이해하는 게 맞지 않을까?”**

### 2) 커스텀 필터를 추가할 때 “어디에 끼워 넣을지” 모른다

언젠가 이런 요구사항이 들어왔다.

- 모든 요청에 대해 테넌트 ID를 헤더에서 읽어와서 검증하고 싶다.  
- 일부 API는 JWT로 인증하고, 나머지는 기존 세션 기반으로 유지하고 싶다.

검색을 하면 이런 코드들이 쏟아진다.

```java
http.addFilterBefore(customFilter, UsernamePasswordAuthenticationFilter.class);
http.addFilterAfter(anotherFilter, SecurityContextHolderFilter.class);
```

문제는, **“이 필터들이 실제로 전체 체인에서 몇 번째인지”**를 모르면  
`addFilterBefore`, `addFilterAfter`가 그냥 주문 외우기가 된다는 거였다.

Servlet 아키텍처를 이해하고 나니,

- 어떤 필터 앞에 두면 “인증 전에 동작하는 필터”가 되는지  
- 어떤 필터 뒤에 두면 “인증 결과를 믿고 쓰는 필터”가 되는지

를 의도적으로 선택할 수 있게 되었다.

### 3) 여러 SecurityFilterChain을 쓰기 시작하면, 아키텍처를 모른다는 게 바로 티가 난다

프로젝트가 커지면 대부분 이렇게 된다.

- `/api/**` → REST API, JWT 기반, Stateless  
- `/admin/**` → 관리자 페이지, 세션/폼 로그인  
- `/public/**` → 누구나 접근 가능한 페이지

Spring Security는 이걸 위해 **여러 개의 SecurityFilterChain**을 지원하는데,  
이 구조를 제대로 이해하지 못한 채 설정을 추가하다 보면,

- 체인 매칭 순서가 꼬여서 엉뚱한 설정이 적용되고  
- 예상치 못한 403, 302가 여기저기서 터진다.

결론은 하나였다.

> **“인증/인가를 잘 쓰고 싶다면, 그 집의 ‘뼈대’인 Servlet 아키텍처부터 이해해야 한다.”**

---

## 2. 그림 한 장으로 보는 Spring Security Servlet 아키텍처

먼저 큰 그림을 텍스트로 정리해보자.

```text
클라이언트
  ↓
Servlet Container Filter Chain
  ↓
DelegatingFilterProxy (springSecurityFilterChain)
  ↓
FilterChainProxy
  ↓
매칭되는 SecurityFilterChain 하나 선택
  ↓
선택된 SecurityFilterChain 안의 Security Filters가 순서대로 실행
  ↓
DispatcherServlet → 컨트롤러
```

한 줄로 줄이면 이렇게 말할 수 있다.

> **“컨테이너 필터 체인 → DelegatingFilterProxy → FilterChainProxy → (여러 필터가 들어있는) SecurityFilterChain 하나 → DispatcherServlet”**

조금 더 풀어서 보면:

1. **Servlet Container Filter Chain**  
   - 톰캣 같은 컨테이너가 관리하는 필터 목록  
   - 인코딩 필터, 로깅 필터, CORS 필터 등도 여기 포함된다.
2. **DelegatingFilterProxy (`springSecurityFilterChain`)**  
   - 컨테이너 세계와 Spring 세계를 연결하는 어댑터  
   - 실제 Spring Security 로직은 여기서 바로 실행되지 않는다.  
3. **FilterChainProxy**  
   - Spring Security의 진짜 진입점  
   - 현재 요청에 맞는 `SecurityFilterChain`을 고르고, 그 안의 필터를 순서대로 호출한다.  
4. **SecurityFilterChain**  
   - “이 URL 패턴에는 이런 필터 조합을 적용하겠다”라는 규칙 한 묶음  
   - 여러 개를 만들 수 있고, 보통 URL 패턴별로 나눈다.  
5. **Security Filters**  
   - `SecurityContextHolderFilter`, `CsrfFilter`, `UsernamePasswordAuthenticationFilter`,  
     `ExceptionTranslationFilter`, `AuthorizationFilter` 같은 실제 보안 필터들.

이제 이 박스들을 하나씩, 사람처럼 소개해 보자.

---

## 3. 각 구성요소를 사람으로 비유해 보기

### 3-1. Servlet Container Filter Chain – 건물 입구 관리인

가장 바깥쪽에는 **서블릿 컨테이너가 관리하는 필터 체인**이 있다.

- 여기에는 Spring Security 말고도,  
  요청/응답 압축, 인코딩, 로깅, CORS 같은 다양한 필터가 섞여 있다.
- Spring을 모르던 시절에 `@WebFilter`로 등록하던 필터들이 바로 이 레벨에서 돌고 있다.

건물로 비유하면 “건물 입구에서 가방 검사하고, 방문자 명부 적는 관리인” 정도다.

이 단계에서 Spring Security 쪽으로 연결해 주는 필터가 바로 **DelegatingFilterProxy**다.

### 3-2. DelegatingFilterProxy – “보안팀장한테 가세요” 안내만 하는 안내 데스크

`DelegatingFilterProxy`는 이름 그대로, **다른 필터 Bean에게 일을 위임하는 필터**다.

```java
public class DelegatingFilterProxy extends GenericFilterBean {

    @Override
    public void doFilter(
        ServletRequest request,
        ServletResponse response,
        FilterChain filterChain
    ) throws ServletException, IOException {

        Filter delegate = getFilterBean("springSecurityFilterChain");
        delegate.doFilter(request, response, filterChain);
    }
}
```

여기서 `"springSecurityFilterChain"`이라는 이름을 가진 Bean이 바로 **FilterChainProxy**다.

중요한 포인트는 두 가지였다.

1. **DelegatingFilterProxy는 Spring Security의 로직을 모른다.**  
   그냥 “이 요청, springSecurityFilterChain한테 넘겨주세요” 하고 연결만 해준다.
2. **생명주기 문제를 해결해준다.**  
   서블릿 컨테이너는 Spring보다 먼저 뜨지만, DelegatingFilterProxy는  
   Spring 컨텍스트가 준비된 뒤에 그 안에서 진짜 필터 Bean을 찾아서 위임한다.

개발자 입장에서는 굳이 건들 일이 거의 없지만,  
**“Servlet → Spring 세계로의 게이트”**라고 생각하면 편하다.

### 3-3. FilterChainProxy – 보안팀의 “총괄 팀장”

`FilterChainProxy`는 Spring Security의 **실질적인 진입점**이다.

역할을 요약하면:

- 현재 요청에 맞는 `SecurityFilterChain`을 고른다.  
- 그 체인 안에 있는 Security Filter들을 순서대로 호출한다.  
- 요청이 끝나면 `SecurityContext`를 정리한다.  
- `HttpFirewall`을 적용해 수상한 요청을 차단한다.

공식 문서에서는 “SecurityFilterChain을 선택하고 실행하는 역할”이라고만 나오지만,  
내가 보기에는 **“보안팀 전체 일을 조율하는 팀장”**에 가깝다.

특히 여러 개의 SecurityFilterChain을 쓰기 시작하면 이 역할이 중요해진다.

```java
@Bean
@Order(1)
public SecurityFilterChain apiFilterChain(HttpSecurity http) throws Exception {
    http
        .securityMatcher("/api/**")
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .httpBasic(Customizer.withDefaults());

    return http.build();
}

@Bean
@Order(2)
public SecurityFilterChain webFilterChain(HttpSecurity http) throws Exception {
    http
        .securityMatcher("/**")
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .formLogin(Customizer.withDefaults());

    return http.build();
}
```

- `/api/**` 요청 → `apiFilterChain`이 선택  
- 그 외 요청 → `webFilterChain`이 선택  
- **첫 번째로 매칭된 체인만 실행**된다는 점이 핵심이다.

이 구조를 이해하고 나니,  
“JWT는 `/api/**`에만 적용하고, 나머지는 세션 기반으로 두자” 같은 설계를  
겁먹지 않고 시도할 수 있었다.

### 3-4. SecurityFilterChain – 상황별로 준비된 “보안 체크리스트”

`SecurityFilterChain`은 한 마디로 말하면 **“특정 요청 집합에 적용할 보안 필터 묶음”**이다.

- `/api/**`에는 CSRF를 끄고, httpBasic만 쓰자  
- `/admin/**`에는 CSRF도 켜고, 폼 로그인 + 세션 관리까지 하자

이런 식으로 URL 패턴과 필터 구성을 묶어 둔 것이 하나의 SecurityFilterChain이다.

`HttpSecurity` 설정은 결국 이 SecurityFilterChain을 만드는 DSL이라고 보면 된다.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf.disable())
        .httpBasic(Customizer.withDefaults())
        .formLogin(Customizer.withDefaults())
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/public/**").permitAll()
            .anyRequest().authenticated()
        );

    return http.build();
}
```

위 설정은 내부적으로 대략 이런 필터 목록을 가진 SecurityFilterChain 하나를 만든다.

```
SecurityContextHolderFilter
→ HeaderWriterFilter
→ CsrfFilter
→ LogoutFilter
→ UsernamePasswordAuthenticationFilter
→ BasicAuthenticationFilter
→ RequestCacheAwareFilter
→ SecurityContextHolderAwareRequestFilter
→ AnonymousAuthenticationFilter
→ SessionManagementFilter
→ ExceptionTranslationFilter
→ AuthorizationFilter
```

이 순서를 보면 알 수 있듯이,  
**“어디에 커스텀 필터를 끼워 넣을지”**가 곧 설계가 된다.

### 3-5. Security Filters – 실제로 몸을 쓰는 보안 요원들

SecurityFilterChain 안에 들어 있는 각각의 필터는  
보안 팀에서 각자 역할이 다른 팀원들 같다.

예를 들어:

- `SecurityContextHolderFilter`  
  - 세션 등에서 `SecurityContext`를 꺼내 ThreadLocal에 올리고, 요청이 끝나면 정리한다.
- `UsernamePasswordAuthenticationFilter`  
  - `/login` 같은 폼 로그인 요청을 가로채서 인증을 시도한다.
- `ExceptionTranslationFilter`  
  - 인증/인가 예외를 잡아서 로그인 페이지로 리다이렉트하거나, 401/403을 내려준다.
- `AuthorizationFilter`  
  - “이 요청을 이 사용자가 해도 되는지” 최종적으로 판단한다.

필터 하나하나를 깊게 보는 건 다음 글들의 주제고,  
이 글에서는 **“이들이 어떤 순서로 줄을 서 있는지”**만 기억해도 충분하다.

---

## 4. 요청 하나를 손으로 끝까지 따라가 보기

이제 실제로 요청 하나가 이 구조를 어떻게 통과하는지 따라가 보자.

상황은 이렇다.

- URL: `GET /api/orders/1`  
- 설정: `/api/**`는 httpBasic + JWT, `/admin/**`는 폼 로그인

### 4-1. 클라이언트 → Servlet Container Filter Chain

브라우저가 `/api/orders/1`을 호출하면,  
먼저 톰캣 같은 Servlet 컨테이너가 가진 **필터 체인**을 돈다.

- 인코딩 필터  
- 로깅 필터  
- (있다면) CORS 필터  
- …
- 그리고 그중 하나가 `DelegatingFilterProxy (springSecurityFilterChain)`이다.

### 4-2. DelegatingFilterProxy → FilterChainProxy

DelegatingFilterProxy는 이렇게 동작한다.

> “아, 이 요청은 Spring Security한테도 한 번 맡겨야지.”  
> “springSecurityFilterChain Bean을 찾아서 대신 doFilter를 호출해야겠다.”

그래서 Spring 컨텍스트에서 `FilterChainProxy` Bean을 꺼내서,  
그냥 그대로 `doFilter`를 넘겨준다.

### 4-3. FilterChainProxy → 맞는 SecurityFilterChain 고르기

`FilterChainProxy`는 현재 요청을 보고, 등록된 `SecurityFilterChain`들 중 하나를 고른다.

예를 들어:

- `@Order(1)` `/api/**`용 체인  
- `@Order(2)` `/admin/**`용 체인  
- `@Order(3)` 나머지 기본 체인

`/api/orders/1` 요청이 들어왔으니,  
가장 먼저 `/api/**`에 매칭되는 체인을 선택하고, 그 체인 안의 필터 목록을 꺼낸다.

중요한 규칙:

> **여러 체인이 매칭되더라도, 첫 번째로 매칭된 체인만 실행된다.**

### 4-4. 선택된 SecurityFilterChain 안에서 필터들이 릴레이

이제 선택된 SecurityFilterChain 안의 필터들이 순서대로 실행된다.

흐름을 단순화해서 보면:

1. `SecurityContextHolderFilter`  
   - 세션/헤더 등에서 `SecurityContext`를 로드해서 ThreadLocal에 올린다.
2. (여러 보조 필터들…)  
3. `UsernamePasswordAuthenticationFilter` 또는 JWT 인증 필터  
   - 인증이 필요한 요청이라면, 여기에서 실제 인증 시도  
4. `AnonymousAuthenticationFilter`  
   - 인증 정보가 없다면 익명 사용자로 설정  
5. `ExceptionTranslationFilter`  
   - 뒤에서 발생하는 `AuthenticationException`, `AccessDeniedException`을 처리할 준비  
6. `AuthorizationFilter`  
   - 이 사용자가 이 요청을 할 수 있는지 최종 판정
7. 모든 Security Filter를 통과하면 → 다음으로 넘김

마지막 필터까지 통과하면, 비로소 `DispatcherServlet`으로 요청이 넘어간다.  
그 뒤부터는 우리가 익숙한 컨트롤러, 핸들러 매핑, 서비스 호출의 세계다.

---

## 5. 이해한 구조를 실제 설정에 녹여보기

이 구조를 이해하고 나서, 기존 프로젝트에 가장 먼저 적용했던 건 두 가지였다.

1. URL 패턴별로 SecurityFilterChain을 분리하는 것  
2. 커스텀 필터를 의도한 위치에 정확히 끼워 넣는 것

### 5-1. API와 웹을 분리한 SecurityFilterChain

먼저, `/api/**`와 나머지 요청에 대해 서로 다른 보안 정책을 적용했다.

```java
@Bean
@Order(1)
public SecurityFilterChain apiSecurity(HttpSecurity http) throws Exception {
    http
        .securityMatcher("/api/**")
        .csrf(csrf -> csrf.disable())
        .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .httpBasic(Customizer.withDefaults());

    return http.build();
}

@Bean
@Order(2)
public SecurityFilterChain webSecurity(HttpSecurity http) throws Exception {
    http
        .securityMatcher("/**")
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .formLogin(Customizer.withDefaults());

    return http.build();
}
```

이제 FilterChainProxy 입장에서 보면 이렇게 된다.

- `/api/**` → `apiSecurity` 체인의 필터들만 실행  
- 그 외 → `webSecurity` 체인의 필터들만 실행

이걸 머릿속에서 “두 개의 서로 다른 SecurityFilterChain이 존재한다”로 그릴 수 있게 되자,  
설정이 훨씬 덜 두려워졌다.

### 5-2. 커스텀 테넌트 필터를 “인증 이후”에 배치하기

멀티 테넌트 환경에서는 이런 요구가 자주 나온다.

> “요청 헤더에 `X-Tenant-ID`가 들어오는데,  
>  이 사용자가 해당 테넌트에 접근 권한이 있는지 검증해줘.”

이 필터는 **“인증된 사용자 정보(Authentication)를 필요로 하는 필터”**다.  
그래서 자연스럽게 “인증 필터 이후”에 배치하는 게 맞다.

```java
http.addFilterAfter(tenantFilter, UsernamePasswordAuthenticationFilter.class);
```

예전에는 이 코드가 “기계적으로 따라 쓰는 주문”에 가까웠다면,  
지금은 머릿속에 이런 그림이 함께 떠오른다.

```
... → UsernamePasswordAuthenticationFilter (인증)
    → TenantFilter (테넌트 권한 검증)
    → ExceptionTranslationFilter
    → AuthorizationFilter
    → DispatcherServlet
```

**필터 위치를 이해하고 선택한다**는 게 이런 느낌이라는 걸 몸으로 알게 되었다.

---

## 6. 실무에서 도움이 되었던 팁들

### 6-1. 필터 체인 로그를 친구처럼 자주 본다

`application.properties`에 다음 한 줄을 넣으면:

```properties
logging.level.org.springframework.security=DEBUG
```

애플리케이션이 뜰 때마다 Spring Security가 각 SecurityFilterChain과 필터 목록을 출력해준다.

문제가 생기면 나는 거의 반사적으로 이 로그부터 본다.

- 이 URL에 어떤 SecurityFilterChain이 매칭되는지  
- 그 체인 안에 어떤 필터들이 어떤 순서로 서 있는지

### 6-2. FilterChainProxy를 직접 주입해서 필터 목록 찍어보기

개발 환경에서는 아예 `FilterChainProxy`를 주입받아서 필터 체인을 출력해보기도 했다.

```java
@Component
@RequiredArgsConstructor
public class FilterChainInspector {

    private final FilterChainProxy filterChainProxy;

    @PostConstruct
    public void printFilterChains() {
        filterChainProxy.getFilterChains().forEach(chain -> {
            System.out.println("=== SecurityFilterChain ===");
            chain.getFilters().forEach(f ->
                System.out.println(" - " + f.getClass().getSimpleName())
            );
        });
    }
}
```

이걸 한 번 찍어보고 나면,

- “이 필터는 이런 순서에 있구나”  
- “여기 사이에 내가 만든 필터를 넣으면 되겠구나”

라는 감각이 눈으로 들어온다.

### 6-3. SecurityContext 정리는 FilterChainProxy에게 맡기되, 존재를 잊지는 말 것

Spring Security는 요청이 끝날 때 `SecurityContextHolder.clearContext()`를 호출해서  
ThreadLocal에 남아 있는 인증 정보를 정리해 준다.

그래서 보통은 우리가 직접 건드릴 일이 없지만,

- 비동기 작업, 스케줄러, 다른 스레드에서 SecurityContext를 옮겨 다닐 때는  
  이 정리 과정을 직접 신경 써야 한다.

Servlet 아키텍처를 이해하고 나니,

- “요청 범위를 벗어나는 작업에서는 SecurityContext를 어떻게 다룰지”  
- “어디까지 Spring Security에 맡기고, 어디서부터는 내가 책임져야 하는지”

의 경계가 좀 더 또렷해졌다.

---

## 7. 마무리 – 구조를 알고 나니, 그 다음 글이 더 잘 보였다

이 글을 쓰기 전까지, 나는 Spring Security를 이렇게 보고 있었다.

> “편리하지만, 내부를 건드리기엔 약간 무서운 검은 상자”

하지만 Servlet 아키텍처를 한 번 끝까지 따라가 보니,  
조금은 생각이 바뀌었다.

- Servlet 컨테이너 필터 체인 위에  
- DelegatingFilterProxy가 있고  
- 그 뒤에 FilterChainProxy와 여러 SecurityFilterChain이 줄줄이 연결돼 있다는 사실을 알고 나니,

이제는 최소한  
**“요청이 어디에서 차단되고 있는지, 내가 만든 필터는 어디에 끼어들고 있는지”**를  
말로 설명할 수 있게 되었다.

이 글이 끝이 아니다.  
Servlet 아키텍처는 말 그대로 “뼈대”에 가깝고,  
그 위에서 **인증(Authentication)**과 **인가(Authorization)**가 어떻게 돌아가는지는  
다음 글에서 더 깊게 파볼 예정이다.

이 글을 읽고 있는 지금,

한 번쯤은 실제 프로젝트에서 자주 쓰는 URL 하나를 골라서,

1. 어떤 SecurityFilterChain에 매칭되는지,  
2. 그 체인 안에 어떤 필터들이 어떤 순서로 서 있는지,  
3. 요청이 어디에서 차단되거나 통과하는지,

로그와 코드를 보면서 끝까지 따라가 보길 추천한다.

그 과정을 지나고 나면,  
Spring Security는 “건드리면 터질 것 같은 검은 상자”가 아니라,  
**내가 이해한 구조 위에 필요한 조각을 하나씩 얹을 수 있는 프레임워크**처럼 느껴질 것이다.

