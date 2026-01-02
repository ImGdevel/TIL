# Local Cache로 Caffeine 쓰면서 배운 것들

> “Redis 붙이기 전에, 애플리케이션 안에서 할 수 있는 캐싱부터 제대로 써보자.”

## 0. 이 글을 쓰게 된 계기

캐시라고 하면 자연스럽게 Redis부터 떠올렸다.  
나도 처음에는 “캐시 = Redis, 네트워크 뒤에 있는 뭔가 빠른 저장소” 정도로만 생각했다.

근데 실제로 API를 최적화해 보니,  
모든 캐시를 굳이 네트워크 너머에 둘 필요는 없다는 걸 느꼈다.

- “JVM 안에서만 써도 되는 값”  
- “인스턴스별로 달라도 괜찮은 값”  
- “조금 부정확해도 괜찮은, 읽기 위주의 값”

이런 것들은 오히려 **로컬 캐시(Local Cache)**가 더 잘 맞는 경우가 많았다.  
그 와중에 Spring에서 가장 손쉽게 쓸 수 있는 라이브러리 중 하나가 바로 **Caffeine**이다.

이 글은,

- Caffeine이 어떤 특징을 가진 로컬 캐시인지  
- Spring Cache와 `@Cacheable`을 통해 어떻게 사용하는지  
- AOP 기반 캐싱에서 자주 만나는 함정(특히 self-invocation, 프록시)들을 어떻게 봐야 하는지

를 한 번에 정리해 보려는 시도다.

---

## 1. 왜 굳이 Local Cache(Caffeine)를 써야 할까?

Redis 같은 분산 캐시가 있는데, 굳이 애플리케이션 내부 로컬 캐시를 쓸 이유가 있을까?

실무에서 느낀 이유는 대략 이렇다.

1. **네트워크 비용 없이, 그냥 메모리에서 바로 읽고 싶다**
   - Redis는 빠르지만, 그래도 네트워크 hop은 있다.  
   - “진짜 자주 쓰고, 인스턴스별로 분리되어도 되는 데이터”라면 로컬 캐시가 더 가볍다.

2. **인스턴스마다 달라도 괜찮은 데이터가 있다**
   - 예: 특정 설정값, 트래픽에 따라 변하는 통계, HealthCheck용 값 등  
   - 굳이 모든 인스턴스에서 동일할 필요 없으면, 공유 캐시보다는 로컬 캐시가 낫다.

3. **분산 캐시 인프라 없이도 손쉽게 캐시를 붙이고 싶을 때**
   - PoC 단계나 내부 도구처럼, Redis를 따로 깔기 부담스러운 상황  
   - 이럴 때 Caffeine + Spring Cache 조합은 “붙였다 뗐다” 하기 쉬운 옵션이다.

물론 Local Cache는,

- 인스턴스마다 캐시 내용이 다를 수 있고  
- 인스턴스 재시작 시 캐시가 싹 날아간다는 한계가 있다.

그래서 **데이터 특성에 따라 Local Cache / Redis / DB를 적절히 섞어 쓰는 설계**가 필요하다.

---

## 2. Caffeine 한 줄 소개 – Guava Cache 후계자

내가 Caffeine을 한 줄로 요약한다면 이렇게 말할 것 같다.

> **“JVM 안에서 쓰는 고성능 캐시 라이브러리 (Guava Cache의 사실상 후속)”**

조금 더 풀면 이런 특징이 있다.

- **동시성에 최적화된 설계**
  - 멀티 스레드 환경에서 경쟁을 줄이기 위해 내부적으로 많은 튜닝이 들어가 있다.
- **유연한 만료/용량 정책**
  - `maximumSize`, `expireAfterWrite`, `expireAfterAccess`, `refreshAfterWrite` 등  
  - 꽤 다양한 정책을 조합해 사용할 수 있다.
- **통계/모니터링 지원**
  - `recordStats()`를 켜면 hit/miss 같은 통계를 수집할 수 있다.
- **Spring Cache 연동이 잘 되어 있다**
  - 별도의 복잡한 코드 없이 `@EnableCaching` + `CaffeineCacheManager` 설정 정도로 바로 얹을 수 있다.

실제로는 **Caffeine 자체 API를 직접 써도 되지만**,  
Spring을 쓰고 있다면 대부분 **Spring Cache 추상화를 통해 `@Cacheable`로 감싸서** 사용하는 패턴이 흔하다.

---

## 3. Spring에서 Caffeine Local Cache 사용하기

가장 기본적인 구성은 다음 세 단계였다.

1. 의존성 추가  
2. `CacheManager` 설정 (Caffeine 연동)  
3. 서비스 메서드에 `@Cacheable`/`@CacheEvict` 붙이기

### 3-1. 의존성 추가

Gradle 기준으로는 보통 이렇게 추가한다.

```groovy
implementation "org.springframework.boot:spring-boot-starter-cache"
implementation "com.github.ben-manes.caffeine:caffeine"
```

그리고 `@EnableCaching`을 설정 클래스에 붙여서 Spring Cache를 활성화한다.

### 3-2. Caffeine 기반 CacheManager 설정

예를 들어, 이렇게 `CaffeineCacheManager`를 Bean으로 등록할 수 있다.

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager("userCache", "configCache");
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .recordStats());
        return cacheManager;
    }
}
```

여기서 중요한 포인트는:

- 캐시 이름(`"userCache"`, `"configCache"`)을 미리 선언해 두고  
- 각각에 대해 **같은 정책(max size, TTL 등)**이 적용된다는 점이다.  
  (더 세밀한 정책이 필요하면 `CacheManager` 구현을 커스터마이징하거나 캐시별로 나눌 수 있다.)

### 3-3. 서비스 메서드에 @Cacheable/@CacheEvict 사용

이제 실제 서비스 코드에서 이렇게 사용할 수 있다.

```java
@Service
public class UserService {

    @Cacheable(cacheNames = "userCache", key = "#userId")
    public UserDto getUser(Long userId) {
        // 여기서부터는 DB 조회나 외부 API 호출 등
        return loadUserFromDatabase(userId);
    }

    @CacheEvict(cacheNames = "userCache", key = "#userId")
    public void updateUser(Long userId, UpdateUserRequest request) {
        updateUserInDatabase(userId, request);
    }
}
```

흐름을 정리해 보면:

1. `getUser(1L)` 첫 호출 → 캐시 miss → 실제 로직 실행 → 결과를 `userCache`에 저장  
2. `getUser(1L)` 두 번째 호출 → 캐시 hit → Caffeine에서 바로 반환 (메서드 본문 실행 안 됨)  
3. `updateUser(1L, ...)` 호출 → DB 업데이트 + 해당 키 캐시 제거  
4. 이후 `getUser(1L)` 호출 → 다시 캐시 miss → 최신 값으로 재채움

여기까지는 Spring Cache 공식 문서 수준의 내용이다.  
하지만 실무에서 막상 써보면, **AOP/프록시 구조 때문에 생각보다 이상한(?) 곳에서 캐시가 안 먹는 상황**을 종종 만나게 된다.

---

## 4. @Cacheable과 AOP – “프록시를 안 거치면 캐시도 안 걸린다”

Spring Cache는 내부적으로 **AOP 기반**으로 동작한다.

조금 거칠게 표현하면,

> “`@Cacheable`이 붙은 메서드를 직접 호출하는 게 아니라,  
> 프록시 객체를 통해 호출될 때만 캐시 로직이 개입한다.”

이 구조 때문에 몇 가지 함정이 생긴다.

### 4-1. 같은 클래스 내부에서 자기 메서드를 호출할 때 (self-invocation)

가장 흔한 케이스가 이거였다.

```java
@Service
public class UserService {

    @Cacheable(cacheNames = "userCache", key = "#userId")
    public UserDto getUser(Long userId) { ... }

    public UserDto getUserForAdmin(Long userId) {
        // 여기서 캐시가 적용되길 기대했지만...
        return getUser(userId);
    }
}
```

겉으로 보기에는,

- `getUserForAdmin()` → `getUser()`를 호출하니까  
- `getUser()`에 걸려 있는 `@Cacheable`이 동작할 것처럼 보인다.

하지만 실제로는 **캐시가 적용되지 않는다.**

이유는 단순하다.

- `UserService` 내부에서 `this.getUser(userId)`를 호출하면  
- 프록시를 거치는 게 아니라 **자기 자신의 실제 인스턴스 메서드를 바로 호출**한다.

Spring Cache AOP는 **프록시에서 메서드를 가로채면서** 캐시 로직을 넣는데,  
이미 실제 인스턴스 안에 들어와 있는 상황에서는 그걸 가로챌 수가 없다.

해결 방법은 여러 가지가 있다.

1. 캐시 메서드를 다른 빈/서비스로 분리한다.
2. `ApplicationContext`에서 자기 자신을 프록시 타입으로 다시 주입받아 호출한다. (개인적으로는 선호하지 않음)
3. 아예 `getUserForAdmin()`에도 `@Cacheable`을 붙여 별도 캐시 전략을 가져간다.

실제로는 **“캐시할 메서드는 별도의 서비스 계층으로 잘라내기”**가 구조상 더 깔끔했다.

### 4-2. public 메서드만 캐시가 적용된다

Spring의 기본 프록시 기반 AOP는 **public 메서드**에만 적용되는 게 일반적이다.  
(프록시 방식, 설정에 따라 다를 수 있지만 기본은 그렇다.)

그래서 아래 코드는 의도와 다르게 동작할 수 있다.

```java
@Service
public class UserService {

    @Cacheable(cacheNames = "userCache", key = "#userId")
    private UserDto getUserInternal(Long userId) { ... }
}
```

“어차피 같은 클래스 안에서만 쓰니까 private로 숨겨야지”라고 생각했는데,  
이 경우에는 **캐시가 전혀 적용되지 않을 수 있다.**

그래서 캐싱을 걸고 싶은 메서드는

- 가급적 **public**으로 두고  
- 필요한 경우 별도의 “내부 헬퍼 메서드”로 분리하는 식으로 가져가는 편이 안전했다.

### 4-3. @Transactional과 @Cacheable 순서/조합

또 하나 자주 헷갈린 부분은 `@Transactional`과 `@Cacheable`의 조합이었다.

```java
@Service
public class UserService {

    @Transactional
    @Cacheable(cacheNames = "userCache", key = "#userId")
    public UserDto getUser(Long userId) { ... }
}
```

여기서 중요한 건,

- Spring AOP가 어떻게 프록시 체인을 구성하는지  
- 트랜잭션 시작/종료와 캐시 조회/저장 시점이 어떤 순서로 일어나는지

실무에서 체감한 포인트는 이 정도였다.

1. 읽기 전용 조회 메서드에 `@Transactional(readOnly = true)` + `@Cacheable`을 같이 쓰는 건 보통 무난했다.  
2. 쓰기/갱신 로직에서는 **`@CacheEvict`를 어디에 붙일지**를 더 신경 써야 했다.  
   - 트랜잭션이 롤백되면 캐시도 롤백되어야 하는지?  
   - 아니면 “실패했는데 캐시만 먼저 지워진 상태”가 되어도 괜찮은지?

결국 답은 “비즈니스 요구사항에 따라” 달라지긴 하지만,  
**트랜잭션 경계와 캐시 갱신/무효화 사이의 순서를 의식적으로 설계**해야 한다는 점은 분명했다.

---

## 5. Caffeine과 Redis를 함께 쓸 때의 전략

Local Cache(Caffeine)와 분산 캐시(Redis)를 함께 쓰는 패턴도 제법 많다.  
이때는 보통 다음과 같은 구조를 생각하게 된다.

1. **1st level – Caffeine (JVM 로컬)**
   - 같은 인스턴스에서 반복되는 호출을 막는다.
2. **2nd level – Redis (분산 캐시)**
   - 여러 인스턴스 간에 값을 공유한다.
3. **3rd level – DB/원천 저장소**
   - 최종 진실의 원천

실제로 Spring에서는,

- Caffeine을 기본 `CacheManager`,  
- Redis를 별도 캐시 영역/네임스페이스로 두고

“이 키는 로컬, 저 키는 Redis” 식으로 나눠 쓰는 방식도 가능하다.

이때 기억해 두고 싶은 포인트는 하나였다.

> **“Local Cache는 언제나 ‘조금 더 빠른 최적화 레이어’일 뿐,  
>  시스템의 정합성은 여전히 Redis/DB 쪽에서 보장해야 한다.”**

즉, Caffeine 캐시는 공격적으로 비워도 되는 쪽에 두는 게 마음이 편했다.

---

## 6. 마무리 – 캐시는 “어디에, 어떤 범위로” 두느냐의 문제

Caffeine을 포함한 Local Cache는,

- “어떻게 쓰면 되지?” 보다는  
- **“어디까지를 이 캐시에 맡길 것인가?”**를 먼저 정의해야 편했다.

이 글에서 정리한 내용을 기준으로,

- 우리 서비스에서 **JVM 내부에서만 캐시해도 괜찮은 데이터**는 무엇인지  
- 그 데이터를 `@Cacheable` + Caffeine으로 감싸는 게 적절한지  
- AOP/프록시 구조 때문에 캐시가 먹지 않는 구간은 없는지 (특히 self-invocation)

을 한 번 점검해 보면 좋을 것 같다.

추가로, 이미 Redis 캐시를 쓰고 있다면,

1. 가장 호출 빈도가 높은 조회 API 하나를 고르고  
2. 그 중 “인스턴스별로 달라도 괜찮은 부분”만 골라 Caffeine으로 한 겹 더 둘러본 다음  
3. 모니터링을 통해 hit/miss와 레이턴시 개선 폭을 확인해 보는 것

정도가 Local Cache를 도입해 보는 좋은 출발점이라고 느꼈다.

