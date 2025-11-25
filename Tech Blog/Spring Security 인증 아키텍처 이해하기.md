# Spring Security 인증 아키텍처, 한 번은 끝까지 따라가보기

> `http.formLogin()`만 쓰던 시절로는 돌아갈 수 없다.

## 0. 로그인 에러 하나에서 시작된 이야기

첫 번째 회사에서 Spring Security를 처음 붙였을 때, 나는 딱 여기까지만 알고 있었다.

```java
http
    .csrf(AbstractHttpConfigurer::disable)
    .formLogin(Customizer.withDefaults())
    .authorizeHttpRequests(auth -> auth.anyRequest().authenticated());
```

“잘 되네?”

로그인 화면도 뜨고, 세션도 알아서 만들어주고, 인증 객체도 어디선가 꺼내 쓰면 됐다.  
그래서 한동안은 “Spring Security는 그냥 설정 몇 줄로 끝나는 프레임워크”라고 생각했다.

문제는 **요구사항이 살짝만 바뀌었을 때**였다.

- 프론트가 “폼 로그인 말고 JSON으로 로그인하고 싶다”고 했을 때
- 일부 API는 세션 대신 **JWT**로, 일부는 기존 세션 기반으로 유지해야 할 때
- 401/403이 뒤섞여 나와서 “왜 여기서는 401이고 저기는 403이지?”를 설명해야 할 때

그제야 깨달았다.  
나는 `SecurityConfig`만 알고 있었지, **인증이 실제로 어떻게 흘러가는지**는 전혀 모른다는 걸.

그래서 어느 날, 아래와 같은 인증 아키텍처 다이어그램을 프린트해서 책상 옆에 붙여두고,  
**“요청 하나가 이 그림을 어떻게 타고 지나가는지 끝까지 따라가 보기”**를 목표로 공부를 시작했다.

![Spring Security Authentication Architecture](./images/spring-security-auth-architecture.png)

이 글은 그때의 경험을 정리한 것이다.  
“개념 → 코드”가 아니라, **“문제 → 이해 → 설계 → 코드”**의 순서로 이야기해보려고 한다.

---

## 1. 왜 굳이 인증 흐름까지 이해해야 할까?

“대충 되면 됐지, 굳이 내부 구조까지 알아야 하나?”  
나도 처음엔 그렇게 생각했다. 하지만 실무에서 부딪힌 몇 가지 상황은 내 생각을 바꾸어 놓았다.

### 1) 디버깅할 때 보이는 건 전부 필터였다

로그 레벨을 DEBUG로 올리고 요청을 한번 흘려보내면, 로그가 이렇게 쏟아진다.

- `... SecurityContextPersistenceFilter ...`
- `... UsernamePasswordAuthenticationFilter ...`
- `... ExceptionTranslationFilter ...`

이 중간 어디선가 인증이 실패하고 있는데,  
**“어떤 필터에서 Authentication이 어떻게 바뀌는지”**를 모르면 로그는 그냥 노이즈일 뿐이었다.

### 2) 인증 방식을 바꾸는 순간, “어디를 손대야 할지” 막막했다

“폼 로그인이 아니라, `/api/login`으로 JSON 요청을 받자”  
“웹은 세션, 모바일은 JWT”

요구사항은 간단하지만, 코드를 어디에 넣어야 할지는 전혀 간단하지 않았다.

- 컨트롤러에서 직접 비밀번호를 비교해야 하나?  
- 필터를 만들어야 한다면, 어디에 끼워 넣어야 하나?  
- JWT를 검증하는 건 컨트롤러 앞? 뒤?

이 질문에 답하려면 결국 이 한 줄이 필요했다.

> **“인증은 `AuthenticationFilter → AuthenticationManager → AuthenticationProvider → UserDetailsService → SecurityContextHolder` 흐름으로 흘러간다.”**

이 흐름을 한 번 이해하고 나니,  
**“어디에 코드를 넣어야 할지”**가 명확해졌다.  
그리고 더 이상 Spring Security를 “검은 상자”로 보지 않게 되었다.

---

## 2. 그림 한 장으로 보는 인증 아키텍처

먼저 큰 그림을 말로 한번 풀어보자. 아래는 인증 다이어그램을 텍스트로 옮긴 것이다.

1. 클라이언트가 로그인 요청을 보낸다. (`Http Request`)
2. `AuthenticationFilter`가 요청에서 아이디/비밀번호를 꺼내 `UsernamePasswordAuthenticationToken`을 만든다.
3. 이 토큰을 `AuthenticationManager`에게 넘겨 “인증 좀 해줘”라고 부탁한다.
4. `AuthenticationManager`(대표 구현체: `ProviderManager`)는 등록된 여러 `AuthenticationProvider`에게 순서대로 물어본다.
5. 예를 들어 `DaoAuthenticationProvider`는 내부에서 `UserDetailsService`를 호출해 사용자 정보를 읽어온다.
6. `UserDetailsService`는 DB에서 사용자를 찾고, `UserDetails` 객체를 만들어 반환한다.
7. `AuthenticationProvider`는 비밀번호를 비교하고, 권한을 채워 넣어 **인증된 Authentication**을 만든다.
8. 이 인증된 객체를 다시 `AuthenticationManager`에게 돌려준다.
9. `AuthenticationManager`는 성공/실패를 `AuthenticationFilter`에게 알려준다.
10. 인증이 성공했다면 `AuthenticationFilter`는 이 인증 객체를 `SecurityContextHolder`에 저장한다.

한 줄로 줄이면 이렇다.

> **요청 → 필터 → 매니저 → 프로바이더 → 유저 조회 → 인증 결정 → SecurityContextHolder 저장**

이제 이 박스들을 하나씩, 조금 더 “사람처럼” 소개해 보자.

---

## 3. 각 컴포넌트를 사람으로 비유해 보기

### 3-1. AuthenticationFilter – 문 앞에서 신분증을 받는 경비원

`AuthenticationFilter`는 **“요청에서 자격 증명을 꺼내는 역할”**만 한다.

폼 로그인이라면 이렇게 생겼을 것이다.

```java
String username = request.getParameter("username");
String password = request.getParameter("password");

Authentication authRequest =
    new UsernamePasswordAuthenticationToken(username, password);
```

이 시점의 `Authentication`은 아직 미인증 상태다.  
경비원 입장에서는 “신분증을 받아 들고 경비실로 전달하는 정도”의 역할이다.

여기서 중요한 포인트는 두 가지였다.

- 필터는 **비밀번호 검증을 하지 않는다.**  
- 필터는 오직 `Authentication`을 만들어 **`AuthenticationManager`에게 던지는 역할**만 한다.

이걸 이해한 순간,  
JSON 로그인을 구현할 때도 “컨트롤러에서 로그인 처리”가 아니라  
**“바디에서 JSON을 파싱하는 커스텀 AuthenticationFilter를 만들자”**는 결론이 자연스럽게 나왔다.

### 3-2. AuthenticationManager & ProviderManager – 일을 분배하는 보안팀장

`AuthenticationManager`는 인터페이스고, 실제 구현체는 대부분 `ProviderManager`다.  
나는 이 둘을 “보안팀장”이라고 생각한다.

- 문 앞 경비원(필터)이 “이 사람 좀 확인해 달라”고 신분증을 건네면
- 팀장은 여러 명의 “전문 심사관(AuthenticationProvider)”에게 순서대로 일을 맡긴다.

`ProviderManager`의 핵심 로직은 정말 단순하다.

1. 등록된 `AuthenticationProvider` 목록을 순회하면서
2. `supports()`로 “이 토큰 타입 처리할 수 있어?”를 물어보고
3. 할 수 있으면 `authenticate()`를 호출해본다.

즉, `ProviderManager`는 **“하나의 애플리케이션에 여러 인증 방식을 동시에 붙일 수 있게 해주는 허브”**다.

- 아이디/비밀번호 로그인 → `DaoAuthenticationProvider`  
- JWT 토큰 → `JwtAuthenticationProvider`  
- 소셜 로그인 → `OAuth2LoginAuthenticationProvider`

이 구조를 이해하고 나서야,  
“폼 로그인과 JWT를 동시에 써야 하는데, 둘이 섞여서 꼬일 것 같다”는 막연한 불안이 사라졌다.  
**각 인증 방식을 하나의 Provider로 만들고, ProviderManager에 줄 세우면 되는구나.**

### 3-3. AuthenticationProvider – 실제로 신분을 확인하는 심사관

`AuthenticationProvider`는 **“인증 로직이 실제로 들어가는 핵심”**이다.

- `supports()`로 내가 처리할 수 있는 토큰인지 확인하고
- `authenticate()`에서 비밀번호를 비교하거나, JWT를 검증하거나, 외부 서버를 호출한다.

예를 들어 아이디/비밀번호 로그인을 담당하는 `DaoAuthenticationProvider`는 대략 이런 흐름이다.

1. `UserDetailsService`로 사용자를 조회한다.  
2. `PasswordEncoder`로 비밀번호를 비교한다.  
3. 성공하면 권한이 포함된 **인증된 Authentication**을 만들어 반환한다.

JWT용 Provider라면 이렇게 바뀔 뿐이다.

1. 토큰 서명을 검증한다.  
2. 토큰 안에서 사용자 ID와 권한을 꺼낸다.  
3. 세션 없이 `Authentication`을 만들어 반환한다.

“인증 규칙이 복잡해지면 어떻게 하지?”라는 질문에 대한 Spring Security의 답이 바로 이 부분이다.  
**규칙이 바뀌어도 Provider 하나만 갈아 끼우면 된다.**

### 3-4. UserDetailsService & UserDetails – 사용자 정보를 찾아오는 사원 DB 담당자

`UserDetailsService`는 “아이디 하나를 받아 사용자 정보를 찾아오는 역할”이다.

- 우리 회사 DB에서는 `User` 엔티티로 저장되어 있고  
- Spring Security는 `UserDetails`라는 인터페이스를 원하니  
- 그 둘 사이를 어댑터처럼 이어주는 계층이다.

이 부분을 직접 구현하면서 크게 얻은 건 두 가지였다.

1. **도메인 모델과 보안 모델을 깔끔하게 분리할 수 있다.**  
   (`User` 엔티티는 비즈니스 용도, `UserDetails`는 보안 용도)
2. 나중에 사용자 저장소가 바뀌어도 (`RDB → LDAP → 외부 API`)  
   **UserDetailsService만 갈아끼우면 인증 로직은 그대로 쓸 수 있다.**

### 3-5. SecurityContextHolder – 현재 로그인한 사람의 신분증을 들고 있는 곳

마지막으로, 모든 인증의 결과가 모이는 곳이 `SecurityContextHolder`다.

- 성공한 `Authentication`은 여기 저장된다.  
- 서비스/컨트롤러 어디에서든 `SecurityContextHolder.getContext().getAuthentication()`으로 꺼낼 수 있다.  
- 내부적으로는 대부분 `ThreadLocal`에 담겨 있다.

처음에는 그냥 “전역 저장소” 정도로만 생각했는데,  
비동기 작업과 스케줄러를 건드리기 시작하면서 이게 얼마나 중요한지 알게 되었다.

- 요청 스레드가 끝나면 SecurityContext는 깨끗이 지워져야 한다.  
- 새로운 스레드에서 인증 정보를 재사용하려면 의도적으로 복사해야 한다.  
- 테스트 코드에서는 `SecurityContextHolder`를 직접 세팅해서 시나리오를 만들 수 있다.

이 과정에서 느낀 건 하나였다.

> **“인증은 단순히 로그인만의 문제가 아니라,  
>  시스템 전체에서 ‘누가 이 일을 했는가’를 설명하는 언어다.”**

---

## 4. 로그인 요청 하나를 손으로 끝까지 따라가 보기

이제 실제로 로그인 요청 하나가 그림을 어떻게 타고 흐르는지,  
조금 더 구체적으로 따라가 보자.

### 4-1. ① 클라이언트가 로그인 요청을 보낸다

```http
POST /login
Content-Type: application/x-www-form-urlencoded

username=devon&password=1234
```

폼 로그인이라면, 이 요청은 `UsernamePasswordAuthenticationFilter`에 걸린다.

### 4-2. ② 필터가 Authentication을 만든다

필터는 요청 파라미터에서 아이디/비밀번호를 꺼내  
이렇게 `UsernamePasswordAuthenticationToken`을 만든다.

```java
Authentication authRequest =
    new UsernamePasswordAuthenticationToken(username, password);
```

아직은 미인증 상태다. `isAuthenticated() == false`.

### 4-3. ③ AuthenticationManager에게 “이 사람 좀 확인해줘”라고 맡긴다

필터는 이렇게 말한다.

```java
Authentication authResult =
    this.authenticationManager.authenticate(authRequest);
```

여기서부터는 필터의 영역이 아니라,  
보안팀장(ProviderManager)의 영역이다.

### 4-4. ④ ProviderManager가 적절한 Provider를 찾는다

`authRequest`의 타입이 `UsernamePasswordAuthenticationToken`이므로,  
`supports(UsernamePasswordAuthenticationToken.class)`에 `true`를 반환하는 Provider를 찾는다.

대부분의 경우 `DaoAuthenticationProvider`가 선택된다.

### 4-5. ⑤ UserDetailsService로 사용자 정보를 읽어온다

`DaoAuthenticationProvider`는 내부에서 다음과 같은 일을 한다.

1. `userDetailsService.loadUserByUsername(username)` 호출  
2. DB에서 해당 사용자를 찾는다.  
3. 찾았다면 `UserDetails`로 변환해 반환한다.

이 단계에서 “사용자를 못 찾았다면” `UsernameNotFoundException`이 터진다.

### 4-6. ⑥ 비밀번호를 비교하고 인증을 완료한다

UserDetails에 담겨 있는 비밀번호(보통 해시된 값)와  
사용자가 보낸 비밀번호를 `PasswordEncoder`로 비교한다.

- 일치하면: 권한을 채워 넣은 **새로운 Authentication**을 만들어 반환  
- 불일치하면: `BadCredentialsException` 발생

중요한 점은,  
**인증에 성공한 이후에는 비밀번호를 보통 null로 만든다**는 것이다.  
(ProviderManager가 자격 증명을 지워준다.)

### 4-7. ⑦ 인증이 성공하면 SecurityContextHolder에 저장한다

필터는 최종적으로 다음과 같은 일을 한다.

```java
SecurityContext context = SecurityContextHolder.createEmptyContext();
context.setAuthentication(authResult);
SecurityContextHolder.setContext(context);
```

이제 이 요청을 처리하는 동안,  
서비스/컨트롤러 어디에서든 `SecurityContextHolder`를 통해  
현재 사용자의 정보를 확인할 수 있다.

---

## 5. 이해한 흐름으로 실제 프로젝트에 녹여보기

이제부터는 “이해한 걸 실제로 어떻게 썼는가”에 대한 이야기다.

### 5-1. UserDetailsService를 먼저 정리했다

내가 처음 한 일은 의외로 단순했다.  
**UserDetailsService를 내 도메인에 맞게 정리하는 것.**

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(username));

        return org.springframework.security.core.userdetails.User.builder()
            .username(user.getUsername())
            .password(user.getPassword())
            .authorities(user.getRoles().stream()
                .map(role -> "ROLE_" + role.getName())
                .toArray(String[]::new))
            .build();
    }
}
```

이 작업만으로도 얻은 게 많았다.

- “우리 서비스에서 **권한이란 무엇인가**”를 팀과 다시 한 번 정의할 수 있었고  
- 도메인 엔티티와 보안 모델을 깔끔하게 분리할 수 있었다.

### 5-2. JSON 로그인 – 컨트롤러가 아니라 “커스텀 AuthenticationFilter”로 풀기

프론트에서 이런 요청을 보내고 싶어했다.

```http
POST /api/login
Content-Type: application/json

{ "username": "devon", "password": "1234" }
```

처음엔 컨트롤러에서 직접 비밀번호를 비교하는 코드가 떠올랐다.  
하지만 인증 흐름을 이해하고 나니, 방향을 바꿨다.

> “이건 컨트롤러에서 할 일이 아니라,  
>  `AuthenticationFilter`를 하나 더 만들어서 AuthenticationManager에게 맡겨야 한다.”

그래서 이런 식의 필터를 만들었다.

```java
public class JsonUsernamePasswordAuthenticationFilter
        extends AbstractAuthenticationProcessingFilter {

    public JsonUsernamePasswordAuthenticationFilter() {
        super("/api/login");
    }

    @Override
    public Authentication attemptAuthentication(
        HttpServletRequest request,
        HttpServletResponse response
    ) throws IOException {

        LoginRequest login = objectMapper.readValue(
            request.getInputStream(), LoginRequest.class);

        Authentication authRequest =
            new UsernamePasswordAuthenticationToken(
                login.getUsername(), login.getPassword());

        return this.getAuthenticationManager().authenticate(authRequest);
    }
}
```

설정에서는 기존 폼 로그인 대신 이 필터를 쓰도록 바꿨다.

```java
http
    .csrf(AbstractHttpConfigurer::disable)
    .addFilterAt(jsonLoginFilter, UsernamePasswordAuthenticationFilter.class);
```

바뀐 건 딱 두 가지뿐이었다.

- 자격 증명을 가져오는 방식: `form-data → JSON`  
- 나머지: 여전히 `AuthenticationManager`, `AuthenticationProvider`, `UserDetailsService`가 처리

**“인증 흐름을 알고 있었기 때문에 바꿀 필요 없는 것들을 확실히 건드리지 않을 수 있었다.”**  
이게 생각보다 큰 안정감을 줬다.

### 5-3. JWT 도입 – Provider 하나로 정리하기

그 다음 요구사항은 이랬다.

- 웹은 기존처럼 세션 기반  
- 모바일 앱은 JWT 기반

처음엔 “JWT를 컨트롤러에서 검증해야 하나?”라는 생각이 들었다.  
하지만 이미 머릿속에는 그림이 있었다.

> “JWT도 결국 인증이니까, Authentication으로 바꿔서 Provider에 맡기면 된다.”

그래서 다음과 같은 구조를 만들었다.

1. `JwtAuthenticationFilter` (OncePerRequestFilter)  
   - 헤더에서 토큰을 꺼낸다.  
   - 토큰이 있으면 `JwtAuthenticationToken`을 만들어 `AuthenticationManager`에게 넘긴다.  
2. `JwtAuthenticationProvider`  
   - 토큰 서명을 검증한다.  
   - 클레임에서 userId와 권한을 꺼낸다.  
   - 인증된 `Authentication`을 만들어 반환한다.  
3. 설정  
   - `JwtAuthenticationFilter`를 `UsernamePasswordAuthenticationFilter` 앞단에 끼워 넣는다.  
   - `DaoAuthenticationProvider`와 `JwtAuthenticationProvider` 둘 다 ProviderManager에 등록한다.

결과적으로 ProviderManager는 이렇게 동작하게 되었다.

- `/login`에서는 `UsernamePasswordAuthenticationToken`이 들어오고, `DaoAuthenticationProvider`가 처리  
- 보호된 API에서 JWT 토큰이 들어오면, `JwtAuthenticationToken`을 만들어 `JwtAuthenticationProvider`가 처리

**“서로 다른 인증 방식을 하나의 공통된 흐름 위에 올려놓는 경험”**을 하면서,  
Spring Security의 설계가 왜 이렇게 되어 있는지 몸으로 느낄 수 있었다.

---

## 6. SecurityContextHolder를 직접 만지면서 배운 것들

인증 흐름을 이해하고 나면, 언젠가 한 번쯤은 `SecurityContextHolder`를 직접 다루게 된다.

- 테스트 코드에서 특정 사용자로 로그인된 상태를 만들고 싶을 때  
- 스케줄러에서 “시스템 계정”으로 무언가 작업해야 할 때  
- 다른 스레드로 인증 정보를 넘겨야 할 때

처음엔 이렇게만 썼다.

```java
SecurityContext context = SecurityContextHolder.createEmptyContext();
context.setAuthentication(authentication);
SecurityContextHolder.setContext(context);
```

그러다 비동기 작업을 쓰기 시작하면서 몇 가지를 배웠다.

1. **ThreadLocal이라는 사실을 반드시 기억해야 한다.**  
   요청이 끝난 뒤 `clearContext()`를 호출하지 않으면,  
   쓰레드 풀에서 재사용된 스레드가 이전 요청의 인증 정보를 물고 갈 수 있다.

2. 스케줄러나 배치 작업에서 인증을 흉내 낼 때는  
   “누가, 어떤 권한을 갖고 있는지”를 명확히 남겨두는 것이 중요하다.  
   (예: `SYSTEM`이라는 principal과 `ROLE_SYSTEM` 권한)

3. 테스트 코드에서 SecurityContext를 직접 세팅해 보면  
   애플리케이션이 “현재 사용자”를 어떤 방식으로 의존하고 있는지 드러난다.

이 과정에서 느낀 건 하나였다.

> **“인증은 단순히 로그인만의 문제가 아니라,  
>  시스템 전체에서 ‘누가 이 일을 했는가’를 설명하는 언어다.”**

---

## 7. 마무리 – 설계가 보이면, 두려움이 줄어든다

처음 Spring Security를 쓸 때 나는 이렇게 생각했다.

> “이 프레임워크는 뭔가 어려운 일을 대신해주지만,  
>  건드리면 망가질 것 같은 검은 상자다.”

하지만 인증 아키텍처 다이어그램을 붙잡고  
HTTP 요청 하나가 **필터 → 매니저 → 프로바이더 → 서비스 → 컨텍스트**로 흐르는 과정을 끝까지 따라가 본 뒤에는,  
그 생각이 조금 바뀌었다.

- `AuthenticationFilter`는 **자격 증명을 꺼내기만 한다.**  
- `AuthenticationManager`/`ProviderManager`는 **여러 인증 방식을 조율한다.**  
- `AuthenticationProvider`는 **실제 인증 규칙을 담고 있다.**  
- `UserDetailsService`는 **우리 도메인과 보안 모델을 연결한다.**  
- `SecurityContextHolder`는 **현재 요청이 누구의 이름으로 실행되는지 기록한다.**

이 역할들을 이해하고 나니,

- JSON 로그인, 소셜 로그인, JWT, 세션 혼합 같은 요구사항도  
  “어디에 무엇을 꽂아야 할지”가 보이기 시작했고  
- 로그를 볼 때도 “지금 어떤 필터에서 Authentication이 어떻게 바뀌고 있는지”를 따라갈 수 있게 되었다.

이 글을 읽고 있는 지금,  
이미 프로젝트에 Spring Security를 쓰고 있다면  
아마도 어딘가에서 Authentication이 생성되고, 위 다이어그램을 따라 흘러가고 있을 것이다.

한 번쯤은 실제 요청 하나를 골라서,

1. 어떤 필터가 자격 증명을 꺼내는지,  
2. 어떤 Provider가 인증을 처리하는지,  
3. 최종 Authentication이 어떤 모양으로 SecurityContextHolder에 들어가는지,

로그와 코드로 끝까지 따라가 보길 추천한다.

그 과정을 지나고 나면,  
Spring Security는 더 이상 “건드리면 터지는 검은 상자”가 아니라,  
**내가 이해한 설계 위에서 확장할 수 있는 도구**가 되어 있을 것이다.

