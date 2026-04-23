# Spring AOP - Pointcut, Advice, Advisor

> 연결된 정리본:
> - [Spring AOP 핵심 개념](../../../TIL.wiki/spring-aop-pointcut-advice-advisor.md)


## 📌 개요

Spring AOP(Aspect-Oriented Programming)는 **횡단 관심사(Cross-Cutting Concerns)**를 효과적으로 분리하고 관리하기 위한 프로그래밍 패러다임입니다. Spring AOP의 핵심은 다음 3가지 요소로 구성됩니다:

- **Pointcut**: 어디에 적용할지 (Where)
- **Advice**: 무엇을 할지 (What)
- **Advisor**: Pointcut + Advice의 조합 (Complete Unit)

### 실무에서의 활용 예시

```java
// 로깅, 트랜잭션, 성능 측정, 보안 검사 등이 필요한 상황
@Service
public class OrderService {
    public Order createOrder(Long userId, List<Item> items) {
        // 비즈니스 로직만 집중
        return orderRepository.save(new Order(userId, items));
    }
}

// AOP를 통해 횡단 관심사를 분리
@Aspect
@Component
public class OrderAspect {
    @Around("@annotation(LogExecutionTime)")
    public Object measureTime(ProceedingJoinPoint pjp) throws Throwable {
        // 실행 시간 측정 로직
    }
}
```

이렇게 비즈니스 로직과 부가 기능을 분리하면 코드의 가독성이 높아지고 유지보수가 용이해집니다.

---

## 1. Pointcut (포인트컷)

### 1.1 정의

**Pointcut**은 부가 기능(Advice)을 적용할 대상을 선별하는 **표현식** 또는 **필터**입니다.

| 용어 | 설명 |
|-----|------|
| Join Point | 프로그램 실행 중 Advice를 적용할 수 있는 지점 (Spring AOP에서는 메서드 실행만 해당) |
| Pointcut | Join Point 중에서 실제로 Advice를 적용할 지점을 선별하는 규칙 |

```java
// Pointcut 정의 예시
@Pointcut("execution(* com.example.service..*.*(..))")
public void serviceLayer() {}
```

위 표현식은 "com.example.service 패키지 및 하위 패키지의 모든 클래스의 모든 메서드"를 의미합니다.

### 1.2 사용 이유

| 이유 | 설명 |
|-----|------|
| **정밀한 타겟팅** | 부가 기능이 필요한 곳에만 선택적으로 적용하여 불필요한 오버헤드 방지 |
| **관심사의 분리** | "어디에 적용할지"와 "무엇을 할지"를 명확히 분리하여 코드 모듈성 향상 |
| **재사용성** | 한 번 정의한 Pointcut을 여러 Advice에서 재사용 가능 |
| **유지보수성** | 적용 범위 변경 시 Pointcut 표현식만 수정하면 되므로 유지보수 용이 |

### 1.3 사용 방법

#### 1.3.1 기본 사용법

```java
@Aspect
@Component
public class LoggingAspect {

    // Pointcut 정의
    @Pointcut("execution(* com.example.service.ProductService.find*(..))")
    public void productFindMethods() {}

    // Pointcut 참조
    @Before("productFindMethods()")
    public void logBeforeFind(JoinPoint joinPoint) {
        System.out.println("상품 조회 메서드 실행: " + joinPoint.getSignature().getName());
    }
}
```

#### 1.3.2 주요 Pointcut 지시자 (Designator)

| 지시자 | 설명 | 예시 |
|-------|------|------|
| `execution` | 메서드 실행 패턴 매칭 (가장 많이 사용) | `execution(* com.example..*.*(..))` |
| `within` | 특정 타입(클래스/패키지) 내의 메서드 | `within(com.example.service.*)` |
| `@annotation` | 특정 어노테이션이 붙은 메서드 | `@annotation(com.example.Loggable)` |
| `@within` | 특정 어노테이션이 붙은 클래스의 모든 메서드 | `@within(org.springframework.stereotype.Service)` |
| `bean` | 특정 Spring Bean의 메서드 | `bean(productService)` |
| `args` | 특정 타입의 인자를 받는 메서드 | `args(java.lang.String, ..)` |

#### 1.3.3 Execution 표현식 문법

**기본 패턴**:
```
execution(접근제어자? 반환타입 패키지.클래스.메서드(파라미터) 예외?)
```

**실전 예제**:

```java
@Aspect
@Component
public class ExecutionExamples {

    // 1. 모든 public 메서드
    @Pointcut("execution(public * *(..))")
    public void allPublicMethods() {}

    // 2. save로 시작하는 모든 메서드
    @Pointcut("execution(* save*(..))")
    public void allSaveMethods() {}

    // 3. Product로 끝나는 클래스의 모든 메서드
    @Pointcut("execution(* com.example.service.*Product.*(..))")
    public void allProductServiceMethods() {}

    // 4. service 패키지와 하위 패키지의 모든 메서드
    @Pointcut("execution(* com.example.service..*.*(..))")
    public void allServiceMethods() {}

    // 5. String을 반환하는 모든 메서드
    @Pointcut("execution(String com.example..*.*(..))")
    public void allStringReturnMethods() {}

    // 6. 첫 번째 파라미터가 Long 타입인 메서드
    @Pointcut("execution(* com.example..*.*(Long, ..))")
    public void methodsWithLongFirstParam() {}

    // 7. 파라미터가 없는 메서드
    @Pointcut("execution(* com.example.service.*.*())")
    public void noArgMethods() {}
}
```

**표현식 와일드카드**:

| 와일드카드 | 의미 | 예시 |
|-----------|------|------|
| `*` | 임의의 값 1개 | `*..*.*(..)` - 모든 패키지, 모든 클래스, 모든 메서드 |
| `..` | 0개 이상 (패키지 또는 파라미터) | `com.example..*` - example 하위 모든 패키지 |
| `(..)` | 0개 이상의 모든 파라미터 | `*(..)` - 파라미터 개수 무관 |

#### 1.3.4 복합 Pointcut 표현식

```java
@Aspect
@Component
public class CompositePointcuts {

    // 개별 Pointcut 정의
    @Pointcut("within(com.example.service..*)")
    public void inServiceLayer() {}

    @Pointcut("execution(public * *(..))")
    public void publicMethod() {}

    @Pointcut("@annotation(org.springframework.transaction.annotation.Transactional)")
    public void transactionalMethod() {}

    // AND 조합: 서비스 레이어의 public 메서드
    @Pointcut("inServiceLayer() && publicMethod()")
    public void publicServiceMethods() {}

    // OR 조합: 서비스 레이어이거나 트랜잭션이 있는 메서드
    @Pointcut("inServiceLayer() || transactionalMethod()")
    public void serviceOrTransactional() {}

    // NOT 조합: 서비스 레이어가 아닌 메서드
    @Pointcut("!inServiceLayer()")
    public void notInService() {}
}
```

#### 1.3.5 커스텀 어노테이션 기반 Pointcut (실무 권장)

```java
// 1. 커스텀 어노테이션 정의
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface TrackPerformance {
    String value() default "";
}

// 2. Aspect에서 Pointcut 정의
@Aspect
@Component
public class PerformanceAspect {

    @Pointcut("@annotation(com.example.annotation.TrackPerformance)")
    public void trackedMethods() {}

    @Around("trackedMethods()")
    public Object trackPerformance(ProceedingJoinPoint pjp) throws Throwable {
        long startTime = System.currentTimeMillis();
        Object result = pjp.proceed();
        long duration = System.currentTimeMillis() - startTime;

        System.out.println(pjp.getSignature().getName() + " 실행 시간: " + duration + "ms");
        return result;
    }
}

// 3. 사용
@Service
public class OrderService {

    @TrackPerformance
    public Order createOrder(Long userId, List<Item> items) {
        // 비즈니스 로직
        return new Order(userId, items);
    }
}
```

**커스텀 어노테이션의 장점**:
- 명시적이고 의도가 명확함
- IDE에서 리팩토링 시 안전함
- 타입 안전성 보장
- 성능 오버헤드 최소화

### 1.4 실무 팁

#### 💡 Tip 1: Pointcut 성능 최적화

```java
// ❌ 나쁜 예: 너무 광범위한 Pointcut
@Pointcut("execution(* *(..))")
public void everywhere() {}  // 모든 메서드 검사 - 성능 저하!

// ✅ 좋은 예: 구체적인 범위 지정
@Pointcut("execution(* com.example.service..*.*(..)) && within(com.example.service..*)")
public void serviceLayerOptimized() {}
```

**권장사항**:
- `execution`과 `within`을 함께 사용하면 매칭 성능 향상
- 가능한 한 구체적인 패키지와 클래스명 사용
- 커스텀 어노테이션 기반 Pointcut 사용 고려

#### 💡 Tip 2: Pointcut 재사용 - 공통 클래스 만들기

```java
// 공통 Pointcut 정의 클래스
@Aspect
public class CommonPointcuts {

    @Pointcut("execution(public * *(..))")
    public void publicMethod() {}

    @Pointcut("within(com.example.service..*)")
    public void inServiceLayer() {}

    @Pointcut("within(com.example.web..*)")
    public void inWebLayer() {}

    @Pointcut("within(com.example.repository..*)")
    public void inRepositoryLayer() {}
}

// 다른 Aspect에서 참조
@Aspect
@Component
public class LoggingAspect {

    @Before("com.example.aspect.CommonPointcuts.publicMethod() && " +
            "com.example.aspect.CommonPointcuts.inServiceLayer()")
    public void logServicePublicMethod(JoinPoint joinPoint) {
        System.out.println("서비스 public 메서드 호출: " + joinPoint.getSignature());
    }
}
```

#### 💡 Tip 3: 흔한 Pointcut 표현식 실수

```java
// ❌ 실수 1: 하위 패키지 미포함
execution(* com.example.service.*.*(..))  // service 직속 클래스만

// ✅ 올바른 예: 하위 패키지 포함
execution(* com.example.service..*.*(..))  // service 및 하위 모든 패키지

// ❌ 실수 2: 파라미터 패턴 혼동
execution(* *(*))     // 파라미터 정확히 1개
execution(* *(..))    // 파라미터 0개 이상

// ❌ 실수 3: Spring 내부 메서드에 AOP 적용
@Before("execution(* org.springframework..*(..))")  // 위험! 무한 루프 가능

// ✅ 올바른 예: 애플리케이션 코드만 대상
@Before("execution(* com.example..*(..))")
```

---

## 2. Advice (어드바이스)

### 2.1 정의

**Advice**는 특정 Join Point에서 실행되는 **실제 부가 기능 코드**입니다. Pointcut이 "어디에"를 결정한다면, Advice는 "무엇을"을 정의합니다.

Spring AOP는 5가지 타입의 Advice를 제공합니다:

| Advice 타입 | 실행 시점 | 사용 시나리오 |
|------------|----------|--------------|
| `@Before` | 메서드 실행 전 | 로깅, 파라미터 검증, 사전 조건 체크 |
| `@AfterReturning` | 메서드 정상 종료 후 | 반환값 검증, 캐시 업데이트, 감사 로깅 |
| `@AfterThrowing` | 메서드 예외 발생 시 | 예외 로깅, 알림 전송, 모니터링 |
| `@After` | 메서드 종료 후 (finally) | 리소스 해제, 정리 작업 |
| `@Around` | 메서드 실행 전후 제어 | 성능 측정, 트랜잭션 관리, 캐싱, 재시도 |

### 2.2 사용 이유

| 이유 | 설명 |
|-----|------|
| **기능의 모듈화** | 로그, 트랜잭션, 보안 등 공통 기능을 별도 클래스로 분리하여 재사용 |
| **코드 중복 제거** | 여러 곳에서 반복되는 코드를 한 곳에서 관리 |
| **관심사의 분리** | 비즈니스 로직과 횡단 관심사를 완전히 분리 |
| **유지보수 용이** | 공통 기능 수정 시 Advice 한 곳만 수정하면 전체에 반영 |

### 2.3 사용 방법

#### 2.3.1 @Before - 메서드 실행 전

```java
@Aspect
@Component
public class ValidationAspect {

    @Before("execution(* com.example.service.ProductService.createProduct(..))")
    public void validateBeforeCreate(JoinPoint joinPoint) {
        Object[] args = joinPoint.getArgs();

        if (args.length > 0 && args[0] instanceof Product) {
            Product product = (Product) args[0];

            if (product.getPrice() < 0) {
                throw new IllegalArgumentException("상품 가격은 음수일 수 없습니다.");
            }

            if (product.getName() == null || product.getName().isEmpty()) {
                throw new IllegalArgumentException("상품명은 필수입니다.");
            }
        }

        System.out.println("✅ 상품 생성 전 검증 완료");
    }
}
```

**특징**:
- 메서드 실행을 막을 수 없음 (예외를 던지지 않는 한)
- 반환값에 접근 불가
- 간단하고 명확함

#### 2.3.2 @AfterReturning - 메서드 정상 종료 후

```java
@Aspect
@Component
public class CacheAspect {

    private final ConcurrentHashMap<String, Product> cache = new ConcurrentHashMap<>();

    @AfterReturning(
        pointcut = "execution(* com.example.service.ProductService.findById(..))",
        returning = "product"
    )
    public void cacheProduct(JoinPoint joinPoint, Product product) {
        if (product != null) {
            Object[] args = joinPoint.getArgs();
            String productId = String.valueOf(args[0]);

            cache.put(productId, product);
            System.out.println("📦 캐시에 상품 저장: " + productId);
        }
    }
}
```

**특징**:
- 정상 종료 시에만 실행 (예외 발생 시 실행 안 됨)
- 반환값 읽기만 가능 (변경 불가)
- `returning` 속성의 이름이 메서드 파라미터 이름과 일치해야 함

**주의사항**:
```java
// ❌ 잘못된 예: 이름 불일치
@AfterReturning(pointcut = "...", returning = "returnValue")
public void afterReturning(Object result) {}  // "result"와 "returnValue" 불일치!

// ✅ 올바른 예
@AfterReturning(pointcut = "...", returning = "result")
public void afterReturning(Object result) {}  // 일치!
```

#### 2.3.3 @AfterThrowing - 메서드 예외 발생 시

```java
@Aspect
@Component
public class ExceptionMonitoringAspect {

    private final NotificationService notificationService;

    public ExceptionMonitoringAspect(NotificationService notificationService) {
        this.notificationService = notificationService;
    }

    @AfterThrowing(
        pointcut = "execution(* com.example.service.PaymentService.*(..))",
        throwing = "exception"
    )
    public void handlePaymentException(JoinPoint joinPoint, Exception exception) {
        String methodName = joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArgs();

        System.err.println("❌ 결제 처리 중 예외 발생: " + methodName);
        System.err.println("   파라미터: " + Arrays.toString(args));
        System.err.println("   예외: " + exception.getMessage());

        // 운영팀에 알림 전송
        notificationService.sendAlert(
            "결제 오류",
            "메서드: " + methodName + ", 오류: " + exception.getMessage()
        );
    }
}
```

**특징**:
- 예외 발생 시에만 실행
- 예외를 잡아서 처리하지 않음 (관찰만 가능)
- 특정 예외 타입만 필터링 가능

**특정 예외만 처리하기**:
```java
@AfterThrowing(
    pointcut = "execution(* com.example.service.*.*(..))",
    throwing = "ex"
)
public void handleDataAccessException(JoinPoint joinPoint, DataAccessException ex) {
    // DataAccessException과 그 하위 타입만 처리
}
```

#### 2.3.4 @After - 메서드 종료 후 (finally)

```java
@Aspect
@Component
public class ResourceCleanupAspect {

    @After("execution(* com.example.service.FileService.*(..))")
    public void cleanupResources(JoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().getName();

        // 정상 종료든 예외 발생이든 항상 실행
        System.out.println("🧹 리소스 정리: " + methodName);

        // 임시 파일 삭제, 연결 종료 등
        cleanupTempFiles();
    }

    private void cleanupTempFiles() {
        // 정리 로직
    }
}
```

**특징**:
- try-finally의 finally 블록처럼 동작
- 정상/예외 관계없이 항상 실행
- 반환값이나 예외에 접근 불가

#### 2.3.5 @Around - 메서드 실행 전후 제어

```java
@Aspect
@Component
@Slf4j
public class PerformanceMonitoringAspect {

    @Around("@annotation(trackPerformance)")
    public Object monitorPerformance(
        ProceedingJoinPoint pjp,
        TrackPerformance trackPerformance
    ) throws Throwable {

        String methodName = pjp.getSignature().toShortString();
        String label = trackPerformance.value();

        log.info("⏱️ [{}] 실행 시작: {}", label, methodName);
        long startTime = System.currentTimeMillis();

        Object result = null;
        try {
            // 실제 메서드 실행
            result = pjp.proceed();

            long duration = System.currentTimeMillis() - startTime;
            log.info("✅ [{}] 실행 완료: {}ms", label, duration);

            // 성능 임계값 체크
            if (duration > 1000) {
                log.warn("⚠️ [{}] 실행 시간이 1초를 초과했습니다: {}ms", label, duration);
            }

            return result;

        } catch (Throwable throwable) {
            long duration = System.currentTimeMillis() - startTime;
            log.error("❌ [{}] 실행 실패: {}ms, 예외: {}",
                label, duration, throwable.getMessage());
            throw throwable;
        }
    }
}

// 사용 예시
@Service
public class OrderService {

    @TrackPerformance("주문 생성")
    public Order createOrder(Long userId, List<Item> items) {
        // 비즈니스 로직
        return new Order(userId, items);
    }
}
```

**특징**:
- 가장 강력하고 유연함
- 메서드 실행 여부 제어 가능
- 반환값 변경 가능
- 예외 처리 가능
- `ProceedingJoinPoint` 사용 필수

**@Around의 강력한 활용 - 재시도 로직**:

```java
@Aspect
@Component
public class RetryAspect {

    @Around("@annotation(retry)")
    public Object retry(ProceedingJoinPoint pjp, Retry retry) throws Throwable {
        int maxAttempts = retry.maxAttempts();
        int attempt = 0;

        while (attempt < maxAttempts) {
            try {
                attempt++;
                return pjp.proceed();

            } catch (Exception e) {
                if (attempt >= maxAttempts) {
                    System.err.println("❌ " + maxAttempts + "번 재시도 실패");
                    throw e;
                }

                System.out.println("⚠️ " + attempt + "번째 시도 실패, 재시도 중...");
                Thread.sleep(retry.delay());
            }
        }

        throw new RuntimeException("재시도 실패");
    }
}

// 커스텀 어노테이션
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Retry {
    int maxAttempts() default 3;
    long delay() default 1000;
}

// 사용
@Service
public class ExternalApiService {

    @Retry(maxAttempts = 5, delay = 2000)
    public ApiResponse callExternalApi() {
        // 외부 API 호출 (실패 가능)
        return externalApi.call();
    }
}
```

**@Around 사용 시 주의사항**:

```java
// ❌ 실수 1: proceed() 호출 누락
@Around("execution(* com.example..*(..))")
public Object doNothing(ProceedingJoinPoint pjp) throws Throwable {
    System.out.println("Before");
    // proceed() 호출 누락! 실제 메서드가 실행되지 않음!
    System.out.println("After");
    return null;  // 항상 null 반환
}

// ✅ 올바른 예
@Around("execution(* com.example..*(..))")
public Object correct(ProceedingJoinPoint pjp) throws Throwable {
    System.out.println("Before");
    Object result = pjp.proceed();  // 반드시 호출
    System.out.println("After");
    return result;  // 결과 반환
}

// ❌ 실수 2: 예외 처리 미흡
@Around("execution(* com.example..*(..))")
public Object wrongExceptionHandling(ProceedingJoinPoint pjp) {  // throws Throwable 누락!
    try {
        return pjp.proceed();
    } catch (Exception e) {  // Exception만 잡음 - Throwable을 잡아야 함
        return null;
    }
}

// ✅ 올바른 예
@Around("execution(* com.example..*(..))")
public Object correctExceptionHandling(ProceedingJoinPoint pjp) throws Throwable {
    try {
        return pjp.proceed();
    } catch (Throwable t) {  // Throwable 처리
        // 로깅 등 처리
        throw t;  // 다시 던지기
    }
}
```

### 2.4 실무 팁

#### 💡 Tip 1: Advice 타입 선택 가이드

```
┌─────────────────────────────────────────────────────────────┐
│  필요한 작업                      → 추천 Advice 타입         │
├─────────────────────────────────────────────────────────────┤
│  파라미터 검증, 로깅              → @Before                 │
│  반환값 검증, 캐시 업데이트        → @AfterReturning        │
│  예외 모니터링, 알림 전송          → @AfterThrowing         │
│  리소스 해제, 정리 작업            → @After                 │
│  성능 측정, 트랜잭션, 재시도       → @Around                │
└─────────────────────────────────────────────────────────────┘
```

**원칙**: **최소 권한의 Advice 타입 사용**
- 반환값만 필요하면 `@AfterReturning` 사용 (불필요하게 `@Around` 사용 X)
- 단순 로깅이면 `@Before` 사용 (복잡한 `@Around` 사용 X)

```java
// ❌ 과도한 권한: 반환값만 필요한데 @Around 사용
@Around("execution(* com.example..*(..))")
public Object unnecessary(ProceedingJoinPoint pjp) throws Throwable {
    Object result = pjp.proceed();
    validateResult(result);
    return result;
}

// ✅ 적절한 권한: @AfterReturning 사용
@AfterReturning(pointcut = "execution(* com.example..*(..))", returning = "result")
public void appropriate(Object result) {
    validateResult(result);
}
```

#### 💡 Tip 2: 여러 Advice의 실행 순서

**단일 Aspect 내에서의 실행 순서**:

```
진입 시:
@Around (before part) → @Before → 실제 메서드 실행

종료 시 (정상):
실제 메서드 → @After → @AfterReturning → @Around (after part)

종료 시 (예외):
실제 메서드 → @After → @AfterThrowing → @Around (catch)
```

**여러 Aspect 간의 실행 순서 - @Order 사용**:

```java
@Aspect
@Order(1)  // 낮은 숫자 = 높은 우선순위
@Component
public class SecurityAspect {

    @Before("execution(* com.example.service.*.*(..))")
    public void checkSecurity() {
        System.out.println("1️⃣ 보안 검사");
    }
}

@Aspect
@Order(2)
@Component
public class LoggingAspect {

    @Before("execution(* com.example.service.*.*(..))")
    public void log() {
        System.out.println("2️⃣ 로깅");
    }
}

@Aspect
@Order(3)
@Component
public class TransactionAspect {

    @Before("execution(* com.example.service.*.*(..))")
    public void beginTransaction() {
        System.out.println("3️⃣ 트랜잭션 시작");
    }
}

// 실행 결과:
// 진입: 1️⃣ 보안 검사 → 2️⃣ 로깅 → 3️⃣ 트랜잭션 시작 → 실제 메서드
// 종료: 실제 메서드 → 3️⃣ 트랜잭션 종료 → 2️⃣ 로깅 종료 → 1️⃣ 보안 종료 (역순)
```

**@Order 우선순위 규칙**:
- **진입 시**: 낮은 숫자(높은 우선순위)가 먼저 실행
- **종료 시**: 낮은 숫자(높은 우선순위)가 나중에 실행 (역순)
- 미지정 시: `Ordered.LOWEST_PRECEDENCE` (가장 낮은 우선순위)

#### 💡 Tip 3: JoinPoint vs ProceedingJoinPoint

| 구분 | JoinPoint | ProceedingJoinPoint |
|-----|-----------|---------------------|
| **사용 가능 Advice** | `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing` | `@Around` 전용 |
| **주요 목적** | 조인 포인트 정보 조회 | 타겟 메서드 실행 제어 |
| **핵심 메서드** | `getSignature()`, `getArgs()`, `getTarget()` | 위 메서드 + `proceed()` |
| **메서드 실행 제어** | 불가능 | 가능 (`proceed()` 호출 여부 결정) |

```java
// JoinPoint 사용 예
@Before("execution(* com.example..*(..))")
public void useJoinPoint(JoinPoint joinPoint) {
    String methodName = joinPoint.getSignature().getName();
    String className = joinPoint.getTarget().getClass().getSimpleName();
    Object[] args = joinPoint.getArgs();

    System.out.println("클래스: " + className);
    System.out.println("메서드: " + methodName);
    System.out.println("인자: " + Arrays.toString(args));
}

// ProceedingJoinPoint 사용 예
@Around("execution(* com.example..*(..))")
public Object useProceedingJoinPoint(ProceedingJoinPoint pjp) throws Throwable {
    // JoinPoint의 모든 기능 + proceed()
    String methodName = pjp.getSignature().getName();

    // 조건에 따라 실행 제어
    if (shouldExecute()) {
        return pjp.proceed();  // 실행
    } else {
        return null;  // 실행 안 함
    }
}
```

#### 💡 Tip 4: Advice에서 메서드 정보 활용

```java
@Aspect
@Component
public class DetailedLoggingAspect {

    @Before("execution(* com.example.service.*.*(..))")
    public void logDetailedInfo(JoinPoint joinPoint) {
        // 메서드 시그니처
        Signature signature = joinPoint.getSignature();
        String methodName = signature.getName();
        String declaringType = signature.getDeclaringTypeName();

        // 대상 객체
        Object target = joinPoint.getTarget();
        Class<?> targetClass = target.getClass();

        // 파라미터
        Object[] args = joinPoint.getArgs();

        // 상세 로그
        System.out.println("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
        System.out.println("📍 클래스: " + declaringType);
        System.out.println("📍 메서드: " + methodName);
        System.out.println("📍 실제 객체: " + targetClass.getSimpleName());
        System.out.println("📍 파라미터 개수: " + args.length);

        for (int i = 0; i < args.length; i++) {
            System.out.println("   [" + i + "] " + args[i]);
        }
        System.out.println("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
    }
}
```

---

## 3. Advisor (어드바이저)

### 3.1 정의

**Advisor**는 하나의 **Pointcut**과 하나의 **Advice**를 결합한 **AOP 적용의 기본 단위**입니다.

| 개념 | 설명 |
|-----|------|
| Advisor | Pointcut + Advice의 조합체 |
| Aspect | 여러 개의 Advice를 포함할 수 있는 모듈 |

```java
// Aspect (여러 Advice 포함 가능)
@Aspect
@Component
public class MyAspect {
    @Before("execution(...)")
    public void advice1() {}

    @After("execution(...)")
    public void advice2() {}
}

// Advisor (하나의 Advice만)
Advisor advisor = new DefaultPointcutAdvisor(pointcut, advice);
```

### 3.2 사용 이유

| 이유 | 설명 |
|-----|------|
| **AOP 적용 단위 표준화** | "어디에"(Pointcut)와 "무엇을"(Advice)을 하나로 묶어 명확히 관리 |
| **프로그래밍 방식 AOP** | `ProxyFactory` 등을 사용한 세밀한 제어가 필요할 때 |
| **단순한 구조** | 하나의 부가 기능만 적용하는 경우 Aspect보다 가벼움 |

### 3.3 사용 방법

#### 3.3.1 ProxyFactory를 이용한 Advisor 적용

```java
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
import org.springframework.aop.Advisor;
import org.springframework.aop.Pointcut;
import org.springframework.aop.framework.ProxyFactory;
import org.springframework.aop.support.DefaultPointcutAdvisor;
import org.springframework.aop.support.NameMatchMethodPointcut;

// 1. 타겟 인터페이스 및 구현체
public interface PaymentService {
    void processPayment(String orderId, double amount);
    void refundPayment(String orderId);
}

public class PaymentServiceImpl implements PaymentService {
    @Override
    public void processPayment(String orderId, double amount) {
        System.out.println("💳 결제 처리: 주문번호=" + orderId + ", 금액=" + amount);
    }

    @Override
    public void refundPayment(String orderId) {
        System.out.println("💸 환불 처리: 주문번호=" + orderId);
    }
}

// 2. Advice 구현 (MethodInterceptor)
public class PaymentLoggingAdvice implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        String methodName = invocation.getMethod().getName();
        Object[] args = invocation.getArguments();

        System.out.println("━━━━━━━━━━━━━━━━━━━━━━━━");
        System.out.println("📝 [Payment Log] 시작: " + methodName);
        System.out.println("   파라미터: " + Arrays.toString(args));

        long startTime = System.currentTimeMillis();
        Object result = invocation.proceed();  // 실제 메서드 실행
        long duration = System.currentTimeMillis() - startTime;

        System.out.println("✅ [Payment Log] 완료: " + duration + "ms");
        System.out.println("━━━━━━━━━━━━━━━━━━━━━━━━");

        return result;
    }
}

// 3. Pointcut 정의 (processPayment 메서드만 대상)
public class PaymentPointcut {
    public static Pointcut createPointcut() {
        NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
        pointcut.addMethodName("processPayment");  // 이 메서드만 매칭
        return pointcut;
    }
}

// 4. Advisor 생성 및 ProxyFactory 사용
public class AdvisorExample {
    public static void main(String[] args) {
        // 타겟 객체 생성
        PaymentService target = new PaymentServiceImpl();

        // Pointcut과 Advice 생성
        Pointcut pointcut = PaymentPointcut.createPointcut();
        PaymentLoggingAdvice advice = new PaymentLoggingAdvice();

        // Advisor 생성 (Pointcut + Advice)
        Advisor advisor = new DefaultPointcutAdvisor(pointcut, advice);

        // ProxyFactory로 프록시 생성
        ProxyFactory proxyFactory = new ProxyFactory(target);
        proxyFactory.addAdvisor(advisor);
        PaymentService proxy = (PaymentService) proxyFactory.getProxy();

        // 테스트
        System.out.println("\n[테스트 1] processPayment 호출 (Advice 적용됨)");
        proxy.processPayment("ORDER-001", 50000.0);

        System.out.println("\n[테스트 2] refundPayment 호출 (Advice 적용 안 됨)");
        proxy.refundPayment("ORDER-001");
    }
}
```

**실행 결과**:
```
[테스트 1] processPayment 호출 (Advice 적용됨)
━━━━━━━━━━━━━━━━━━━━━━━━
📝 [Payment Log] 시작: processPayment
   파라미터: [ORDER-001, 50000.0]
💳 결제 처리: 주문번호=ORDER-001, 금액=50000.0
✅ [Payment Log] 완료: 2ms
━━━━━━━━━━━━━━━━━━━━━━━━

[테스트 2] refundPayment 호출 (Advice 적용 안 됨)
💸 환불 처리: 주문번호=ORDER-001
```

**Pointcut 조건에 맞는 `processPayment`만 Advice가 적용되고, `refundPayment`는 적용되지 않음**을 확인할 수 있습니다.

#### 3.3.2 여러 Advisor 적용하기

```java
public class MultipleAdvisorsExample {
    public static void main(String[] args) {
        PaymentService target = new PaymentServiceImpl();

        // Advisor 1: 로깅
        Pointcut pointcut1 = Pointcut.TRUE;  // 모든 메서드
        PaymentLoggingAdvice loggingAdvice = new PaymentLoggingAdvice();
        Advisor loggingAdvisor = new DefaultPointcutAdvisor(pointcut1, loggingAdvice);

        // Advisor 2: 성능 측정
        Pointcut pointcut2 = Pointcut.TRUE;
        MethodInterceptor performanceAdvice = invocation -> {
            long start = System.currentTimeMillis();
            Object result = invocation.proceed();
            long duration = System.currentTimeMillis() - start;
            System.out.println("⏱️ 실행 시간: " + duration + "ms");
            return result;
        };
        Advisor performanceAdvisor = new DefaultPointcutAdvisor(pointcut2, performanceAdvice);

        // ProxyFactory에 여러 Advisor 추가
        ProxyFactory proxyFactory = new ProxyFactory(target);
        proxyFactory.addAdvisor(loggingAdvisor);      // 첫 번째 Advisor
        proxyFactory.addAdvisor(performanceAdvisor);  // 두 번째 Advisor

        PaymentService proxy = (PaymentService) proxyFactory.getProxy();
        proxy.processPayment("ORDER-002", 100000.0);

        // 실행 순서: loggingAdvisor → performanceAdvisor → 실제 메서드 → performanceAdvisor → loggingAdvisor
    }
}
```

### 3.4 실무 팁

#### 💡 Tip 1: Aspect vs Advisor 선택 기준

| 상황 | 추천 | 이유 |
|-----|------|------|
| 선언적 AOP (`@Aspect`) | **Aspect** | 간편하고 직관적, Spring Boot 환경에서 표준 |
| 프로그래밍 방식 AOP | **Advisor** | `ProxyFactory` 등을 통한 세밀한 제어 가능 |
| 여러 Advice 필요 | **Aspect** | 하나의 클래스에 여러 Advice 정의 가능 |
| 단일 Advice | **Advisor** | 가볍고 명확한 구조 |
| 프레임워크 개발 | **Advisor** | 동적으로 Advisor 추가/제거 가능 |

**실무에서는 대부분 `@Aspect`를 사용**하며, `Advisor`는 특수한 경우에만 사용합니다.

#### 💡 Tip 2: Advisor는 1 Pointcut + 1 Advice

```java
// ✅ Advisor의 기본 구조
public interface Advisor {
    Advice getAdvice();  // 하나의 Advice만
}

public interface PointcutAdvisor extends Advisor {
    Pointcut getPointcut();  // 하나의 Pointcut만
}

// DefaultPointcutAdvisor 사용 예
Advisor advisor = new DefaultPointcutAdvisor(
    pointcut,  // 1개
    advice     // 1개
);
```

#### 💡 Tip 3: Spring의 내장 Advisor들

Spring은 자주 사용되는 Advisor를 내장으로 제공합니다:

| Advisor 클래스 | 설명 |
|---------------|------|
| `DefaultPointcutAdvisor` | 가장 일반적인 Advisor (Pointcut + Advice) |
| `NameMatchMethodPointcutAdvisor` | 메서드 이름 기반 Pointcut |
| `RegexpMethodPointcutAdvisor` | 정규표현식 기반 Pointcut |
| `AspectJExpressionPointcutAdvisor` | AspectJ 표현식 기반 Pointcut |

```java
// NameMatchMethodPointcutAdvisor 사용 예
NameMatchMethodPointcutAdvisor advisor = new NameMatchMethodPointcutAdvisor();
advisor.setMappedName("process*");  // process로 시작하는 메서드
advisor.setAdvice(myAdvice);
```

---

## 4. 실전 예제: 통합 시나리오

### 4.1 시나리오 설정

**상황**: E-Commerce 쇼핑몰의 주문 처리 시스템 개발

**요구사항**:
1. 모든 주문 관련 메서드의 실행 시간 측정 (1초 이상 시 경고)
2. 주문 생성/취소 시 감사 로그 기록
3. 관리자 권한이 필요한 메서드 보안 검사
4. 결제 관련 예외 발생 시 알림 전송

### 4.2 구현

#### 4.2.1 도메인 클래스

```java
// Order.java
@Getter
@Setter
public class Order {
    private Long id;
    private Long userId;
    private List<OrderItem> items;
    private double totalAmount;
    private OrderStatus status;

    public Order(Long userId, List<OrderItem> items) {
        this.userId = userId;
        this.items = items;
        this.totalAmount = items.stream()
            .mapToDouble(OrderItem::getPrice)
            .sum();
        this.status = OrderStatus.PENDING;
    }
}

@Getter
@Setter
public class OrderItem {
    private Long productId;
    private String productName;
    private int quantity;
    private double price;
}

public enum OrderStatus {
    PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
}
```

#### 4.2.2 커스텀 어노테이션

```java
// @AuditLog.java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface AuditLog {
    String action();
}

// @RequiresAdmin.java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequiresAdmin {
}

// @MonitorPerformance.java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MonitorPerformance {
    long warningThresholdMs() default 1000;
}
```

#### 4.2.3 Service 클래스

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final PaymentService paymentService;

    @Autowired
    public OrderService(OrderRepository orderRepository, PaymentService paymentService) {
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
    }

    @MonitorPerformance(warningThresholdMs = 1000)
    @AuditLog(action = "주문 생성")
    public Order createOrder(Long userId, List<OrderItem> items) {
        System.out.println("🛒 주문 생성 중...");

        // 비즈니스 로직
        Order order = new Order(userId, items);

        // 결제 처리
        paymentService.processPayment(order.getTotalAmount());

        // 주문 저장
        Order savedOrder = orderRepository.save(order);

        System.out.println("✅ 주문 생성 완료: " + savedOrder.getId());
        return savedOrder;
    }

    @MonitorPerformance(warningThresholdMs = 500)
    public Order getOrder(Long orderId) {
        System.out.println("🔍 주문 조회: " + orderId);
        return orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException("주문을 찾을 수 없습니다: " + orderId));
    }

    @RequiresAdmin
    @AuditLog(action = "주문 취소")
    public void cancelOrder(Long orderId) {
        System.out.println("🚫 주문 취소 중...");

        Order order = getOrder(orderId);
        order.setStatus(OrderStatus.CANCELLED);

        // 환불 처리
        paymentService.refundPayment(order.getTotalAmount());

        orderRepository.save(order);
        System.out.println("✅ 주문 취소 완료: " + orderId);
    }
}

@Service
public class PaymentService {

    public void processPayment(double amount) {
        System.out.println("💳 결제 처리: " + amount + "원");

        // 시뮬레이션: 10% 확률로 결제 실패
        if (Math.random() < 0.1) {
            throw new PaymentException("결제 처리 중 오류가 발생했습니다.");
        }
    }

    public void refundPayment(double amount) {
        System.out.println("💸 환불 처리: " + amount + "원");
    }
}
```

#### 4.2.4 Aspect 구현

**1. 성능 모니터링 Aspect**

```java
@Aspect
@Order(1)
@Component
@Slf4j
public class PerformanceMonitoringAspect {

    @Around("@annotation(monitorPerformance)")
    public Object monitorPerformance(
        ProceedingJoinPoint pjp,
        MonitorPerformance monitorPerformance
    ) throws Throwable {

        String methodName = pjp.getSignature().toShortString();
        long threshold = monitorPerformance.warningThresholdMs();

        log.info("⏱️ [성능 측정 시작] {}", methodName);
        long startTime = System.currentTimeMillis();

        Object result = null;
        try {
            result = pjp.proceed();
            return result;

        } finally {
            long duration = System.currentTimeMillis() - startTime;

            if (duration > threshold) {
                log.warn("⚠️ [성능 경고] {} 실행 시간: {}ms (임계값: {}ms 초과)",
                    methodName, duration, threshold);
            } else {
                log.info("✅ [성능 측정 완료] {} 실행 시간: {}ms", methodName, duration);
            }
        }
    }
}
```

**2. 감사 로그 Aspect**

```java
@Aspect
@Order(2)
@Component
public class AuditLoggingAspect {

    private final AuditLogRepository auditLogRepository;

    @Autowired
    public AuditLoggingAspect(AuditLogRepository auditLogRepository) {
        this.auditLogRepository = auditLogRepository;
    }

    @AfterReturning(
        pointcut = "@annotation(auditLog)",
        returning = "result"
    )
    public void logAudit(JoinPoint joinPoint, AuditLog auditLog, Object result) {
        String action = auditLog.action();
        String methodName = joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArgs();

        // 감사 로그 저장
        AuditLogEntry entry = AuditLogEntry.builder()
            .action(action)
            .methodName(methodName)
            .arguments(Arrays.toString(args))
            .result(result != null ? result.toString() : "void")
            .timestamp(LocalDateTime.now())
            .userId(SecurityContext.getCurrentUserId())  // 현재 사용자 ID
            .build();

        auditLogRepository.save(entry);

        System.out.println("📋 [감사 로그] " + action + " - 사용자: " + entry.getUserId());
    }
}
```

**3. 보안 검사 Aspect**

```java
@Aspect
@Order(3)
@Component
public class SecurityAspect {

    @Before("@annotation(com.example.annotation.RequiresAdmin)")
    public void checkAdminPermission(JoinPoint joinPoint) {
        String currentUser = SecurityContext.getCurrentUser();
        boolean isAdmin = SecurityContext.isAdmin();

        if (!isAdmin) {
            String methodName = joinPoint.getSignature().getName();
            System.err.println("🚨 [보안 거부] 권한 없음: " + currentUser + " → " + methodName);

            throw new AccessDeniedException(
                "관리자 권한이 필요합니다: " + methodName
            );
        }

        System.out.println("🔐 [보안 검사 통과] 관리자 권한 확인: " + currentUser);
    }
}
```

**4. 예외 알림 Aspect**

```java
@Aspect
@Order(4)
@Component
public class ExceptionNotificationAspect {

    private final NotificationService notificationService;

    @Autowired
    public ExceptionNotificationAspect(NotificationService notificationService) {
        this.notificationService = notificationService;
    }

    @AfterThrowing(
        pointcut = "execution(* com.example.service.PaymentService.*(..))",
        throwing = "exception"
    )
    public void notifyPaymentException(JoinPoint joinPoint, Exception exception) {
        String methodName = joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArgs();

        // 운영팀에 알림
        String message = String.format(
            "결제 처리 오류 발생\n" +
            "메서드: %s\n" +
            "파라미터: %s\n" +
            "예외: %s",
            methodName,
            Arrays.toString(args),
            exception.getMessage()
        );

        notificationService.sendAlert("결제 시스템 오류", message);

        System.err.println("📧 [알림 전송] 결제 오류 - " + exception.getMessage());
    }
}
```

#### 4.2.5 공통 Pointcut 정의

```java
@Aspect
public class CommonPointcuts {

    @Pointcut("within(com.example.service..*)")
    public void inServiceLayer() {}

    @Pointcut("execution(public * *(..))")
    public void publicMethod() {}

    @Pointcut("inServiceLayer() && publicMethod()")
    public void servicePublicMethods() {}
}
```

### 4.3 실행 흐름 분석

**시나리오**: 관리자가 주문을 취소하는 경우

```java
@SpringBootTest
public class OrderServiceTest {

    @Autowired
    private OrderService orderService;

    @Test
    public void testCancelOrder() {
        // 관리자 권한 설정
        SecurityContext.setCurrentUser("admin", true);

        // 주문 취소
        orderService.cancelOrder(12345L);
    }
}
```

**실행 순서 및 로그**:

```
1️⃣ [Order=1] SecurityAspect - @Before
   🔐 [보안 검사 통과] 관리자 권한 확인: admin

2️⃣ [Order=2] AuditLoggingAspect - @AfterReturning (대기 중)

3️⃣ [Order=3] (실제 메서드 실행 시작)
   🚫 주문 취소 중...
   🔍 주문 조회: 12345
   💸 환불 처리: 50000.0원
   ✅ 주문 취소 완료: 12345

4️⃣ [Order=2] AuditLoggingAspect - @AfterReturning
   📋 [감사 로그] 주문 취소 - 사용자: admin

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
최종 실행 순서 (Aspect Order 기준):
진입: SecurityAspect(1) → 실제 메서드
종료: 실제 메서드 → AuditLoggingAspect(2)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**예외 발생 시나리오** (결제 실패):

```
1️⃣ [Order=1] PerformanceMonitoringAspect - @Around (before)
   ⏱️ [성능 측정 시작] OrderService.createOrder(..)

2️⃣ (실제 메서드 실행)
   🛒 주문 생성 중...
   💳 결제 처리: 50000.0원

3️⃣ (PaymentService에서 예외 발생)
   ❌ PaymentException: 결제 처리 중 오류가 발생했습니다.

4️⃣ [Order=4] ExceptionNotificationAspect - @AfterThrowing
   📧 [알림 전송] 결제 오류 - 결제 처리 중 오류가 발생했습니다.

5️⃣ [Order=1] PerformanceMonitoringAspect - @Around (finally)
   ✅ [성능 측정 완료] OrderService.createOrder(..) 실행 시간: 125ms

6️⃣ (예외가 호출자에게 전파됨)
```

---

## 5. 실무에서 주의할 점

### 5.1 Self-Invocation 문제 (가장 흔한 실수!)

#### 문제 상황

```java
@Service
public class OrderService {

    @Transactional
    public void createOrder(Order order) {
        // 주문 생성 로직
        orderRepository.save(order);

        // ❌ Self-Invocation: 프록시를 거치지 않음!
        this.sendOrderConfirmation(order);
    }

    @Async  // 이 어노테이션이 동작하지 않음!
    public void sendOrderConfirmation(Order order) {
        // 주문 확인 이메일 발송
        emailService.send(order.getUserEmail(), "주문 확인");
    }
}
```

**원인**:
- Spring AOP는 **프록시 기반**으로 동작
- `this.method()` 호출은 프록시를 거치지 않고 **실제 객체의 메서드를 직접 호출**
- 따라서 `@Async`, `@Transactional` 등의 AOP가 적용되지 않음

#### 해결 방법

**방법 1: 코드 리팩토링 (권장)**

```java
@Service
public class OrderService {

    private final OrderNotificationService notificationService;

    @Autowired
    public OrderService(OrderNotificationService notificationService) {
        this.notificationService = notificationService;
    }

    @Transactional
    public void createOrder(Order order) {
        orderRepository.save(order);

        // ✅ 다른 빈을 통해 호출 - 프록시를 거침!
        notificationService.sendOrderConfirmation(order);
    }
}

@Service
public class OrderNotificationService {

    @Async
    public void sendOrderConfirmation(Order order) {
        emailService.send(order.getUserEmail(), "주문 확인");
    }
}
```

**방법 2: Self Injection**

```java
@Service
public class OrderService {

    @Autowired
    @Lazy  // 순환 참조 방지
    private OrderService self;  // 자기 자신의 프록시 주입

    @Transactional
    public void createOrder(Order order) {
        orderRepository.save(order);

        // ✅ 프록시를 통해 호출
        self.sendOrderConfirmation(order);
    }

    @Async
    public void sendOrderConfirmation(Order order) {
        emailService.send(order.getUserEmail(), "주문 확인");
    }
}
```

**방법 3: AopContext 사용 (비권장)**

```java
// 설정 클래스에서 exposeProxy 활성화
@Configuration
@EnableAspectJAutoProxy(exposeProxy = true)
public class AopConfig {
}

@Service
public class OrderService {

    @Transactional
    public void createOrder(Order order) {
        orderRepository.save(order);

        // ✅ 현재 프록시 가져오기
        OrderService proxy = (OrderService) AopContext.currentProxy();
        proxy.sendOrderConfirmation(order);
    }

    @Async
    public void sendOrderConfirmation(Order order) {
        emailService.send(order.getUserEmail(), "주문 확인");
    }
}
```

**권장 순서**: 리팩토링 > Self Injection > AopContext

### 5.2 프록시 관련 이슈

#### JDK Dynamic Proxy vs CGLIB

| 특성 | JDK Dynamic Proxy | CGLIB |
|------|------------------|-------|
| **요구사항** | 인터페이스 필요 | 인터페이스 불필요 |
| **작동 방식** | 인터페이스 기반 프록시 생성 | 클래스 상속 기반 프록시 생성 |
| **제약사항** | 인터페이스 메서드만 프록시 가능 | `final` 클래스/메서드 프록시 불가 |
| **성능** | 약간 느림 | 더 빠름 |
| **Spring Boot 기본값** | CGLIB (2.0+) | CGLIB (2.0+) |

#### 문제 상황 1: CGLIB의 final 제약

```java
// ❌ final 클래스 - CGLIB 프록시 생성 불가!
@Service
public final class PaymentService {

    @Transactional
    public void processPayment(double amount) {
        // ...
    }
}
// 에러: Cannot subclass final class

// ❌ final 메서드 - 프록시 불가!
@Service
public class PaymentService {

    @Transactional
    public final void processPayment(double amount) {
        // AOP가 적용되지 않음!
    }
}

// ✅ 해결: final 제거
@Service
public class PaymentService {

    @Transactional
    public void processPayment(double amount) {
        // AOP 정상 동작
    }
}
```

#### 문제 상황 2: private 메서드에 AOP 적용 불가

```java
@Service
public class OrderService {

    public void createOrder(Order order) {
        validateOrder(order);
        orderRepository.save(order);
    }

    // ❌ private 메서드 - AOP 적용 안 됨!
    @Transactional
    private void validateOrder(Order order) {
        // @Transactional이 동작하지 않음!
    }
}

// ✅ 해결: public으로 변경
@Service
public class OrderService {

    public void createOrder(Order order) {
        validateOrder(order);
        orderRepository.save(order);
    }

    @Transactional
    public void validateOrder(Order order) {
        // AOP 정상 동작
    }
}
```

#### CGLIB 강제 사용 설정

```java
// application.yml
spring:
  aop:
    proxy-target-class: true  # CGLIB 사용 (Spring Boot 기본값)

// Java Config
@Configuration
@EnableAspectJAutoProxy(proxyTargetClass = true)
public class AopConfig {
}
```

### 5.3 Pointcut 표현식 실수

#### 실수 1: 너무 광범위한 Pointcut

```java
// ❌ 매우 나쁜 예: 모든 메서드에 적용
@Around("execution(* *(..))")
public Object logEverything(ProceedingJoinPoint pjp) throws Throwable {
    // 심각한 성능 저하!
    // Spring 내부 메서드까지 포함되어 무한 루프 가능!
    return pjp.proceed();
}

// ✅ 좋은 예: 특정 패키지로 제한
@Around("execution(* com.example.service..*(..))")
public Object logServiceLayer(ProceedingJoinPoint pjp) throws Throwable {
    return pjp.proceed();
}
```

#### 실수 2: 와일드카드 혼동

```java
// ❌ 실수: service 직속 클래스만
execution(* com.example.service.*.*(..))
// 매칭: com.example.service.OrderService
// 미매칭: com.example.service.order.OrderServiceImpl

// ✅ 올바른 예: service 및 하위 패키지 전체
execution(* com.example.service..*.*(..))
// 매칭: com.example.service.OrderService
// 매칭: com.example.service.order.OrderServiceImpl
```

#### 실수 3: Spring 내부 메서드 간섭

```java
// ❌ 위험: Spring 내부 메서드에 AOP 적용
@Before("execution(* org.springframework..*(..))")
public void interceptSpringInternal() {
    // 무한 루프나 예상치 못한 동작 발생 가능!
}

// ✅ 안전: 애플리케이션 코드만 대상
@Before("execution(* com.example..*(..))")
public void interceptApplicationCode() {
    // 안전하게 동작
}
```

### 5.4 @Around Advice 실수

#### 실수 1: proceed() 호출 누락

```java
// ❌ 잘못된 예
@Around("execution(* com.example.service.*.*(..))")
public Object wrongAround(ProceedingJoinPoint pjp) throws Throwable {
    System.out.println("Before");

    // proceed() 호출 누락! 실제 메서드가 실행되지 않음!

    System.out.println("After");
    return null;  // 항상 null 반환
}

// ✅ 올바른 예
@Around("execution(* com.example.service.*.*(..))")
public Object correctAround(ProceedingJoinPoint pjp) throws Throwable {
    System.out.println("Before");
    Object result = pjp.proceed();  // 반드시 호출!
    System.out.println("After");
    return result;  // 결과 반환
}
```

#### 실수 2: 예외 처리 미흡

```java
// ❌ 잘못된 예: throws Throwable 누락
@Around("execution(* com.example.service.*.*(..))")
public Object wrongException(ProceedingJoinPoint pjp) {  // throws Throwable 없음!
    try {
        return pjp.proceed();
    } catch (Exception e) {  // Exception만 잡음
        return null;
    }
    // Error 계열 예외는 처리되지 않음!
}

// ✅ 올바른 예
@Around("execution(* com.example.service.*.*(..))")
public Object correctException(ProceedingJoinPoint pjp) throws Throwable {
    try {
        return pjp.proceed();
    } catch (Throwable t) {  // Throwable로 모든 예외 처리
        log.error("오류 발생", t);
        throw t;  // 다시 던지기
    }
}
```

### 5.5 성능 고려사항

#### 문제: Hot Path에 AOP 적용

```java
// ❌ 나쁜 예: 초당 수천 번 호출되는 메서드에 무거운 AOP
@Around("execution(* com.example.cache.CacheService.get*(..))")
public Object heavyProcessing(ProceedingJoinPoint pjp) throws Throwable {
    // 복잡한 로깅, DB 조회 등 무거운 처리
    // 성능 저하 심각!
    return pjp.proceed();
}

// ✅ 좋은 예: 경량화된 로직만
@Around("execution(* com.example.cache.CacheService.get*(..))")
public Object lightweightProcessing(ProceedingJoinPoint pjp) throws Throwable {
    long start = System.nanoTime();  // 가벼운 시간 측정만
    Object result = pjp.proceed();
    long duration = System.nanoTime() - start;

    // 비동기로 무거운 작업 처리
    if (duration > THRESHOLD) {
        executorService.submit(() -> logSlowAccess(pjp, duration));
    }

    return result;
}
```

#### AspectJ 사용 고려

**Spring AOP 한계**:
- 프록시 기반이라 메서드 호출에만 적용 가능
- Self-invocation 문제
- 프록시 생성 및 호출 오버헤드

**AspectJ 사용 시점**:
- 필드 접근, 생성자 호출에도 AOP 필요
- 극도의 성능이 중요한 경우 (컴파일 타임 위빙으로 8~35배 빠름)
- Self-invocation 문제 근본 해결 필요

```xml
<!-- Maven에 AspectJ 플러그인 추가 -->
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>aspectj-maven-plugin</artifactId>
    <version>1.14.0</version>
    <configuration>
        <complianceLevel>17</complianceLevel>
        <source>17</source>
        <target>17</target>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>compile</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

---

## 6. Best Practices

### 6.1 최소 권한의 Advice 타입 사용

```java
// ❌ 나쁜 예: 반환값만 필요한데 @Around 사용
@Around("execution(* com.example.service.*.*(..))")
public Object overkill(ProceedingJoinPoint pjp) throws Throwable {
    Object result = pjp.proceed();
    validateResult(result);
    return result;
}

// ✅ 좋은 예: @AfterReturning 사용
@AfterReturning(pointcut = "execution(* com.example.service.*.*(..))", returning = "result")
public void appropriate(Object result) {
    validateResult(result);
}
```

**선택 가이드**:
```
반환값만 필요 → @AfterReturning
사전 검증만 → @Before
정리만 필요 → @After
예외 처리만 → @AfterThrowing
메서드 제어 필요 → @Around
```

### 6.2 Pointcut 재사용

```java
// ❌ 나쁜 예: Pointcut 중복 정의
@Before("execution(* com.example.service.*.*(..))")
public void logBefore() {}

@After("execution(* com.example.service.*.*(..))")
public void logAfter() {}

@Around("execution(* com.example.service.*.*(..))")
public Object logAround(ProceedingJoinPoint pjp) throws Throwable {
    return pjp.proceed();
}

// ✅ 좋은 예: Pointcut 재사용
@Aspect
@Component
public class LoggingAspect {

    @Pointcut("execution(* com.example.service.*.*(..))")
    private void serviceMethods() {}

    @Before("serviceMethods()")
    public void logBefore() {}

    @After("serviceMethods()")
    public void logAfter() {}

    @Around("serviceMethods()")
    public Object logAround(ProceedingJoinPoint pjp) throws Throwable {
        return pjp.proceed();
    }
}
```

### 6.3 관심사의 명확한 분리

```java
// ❌ 나쁜 예: 하나의 Aspect에 여러 관심사
@Aspect
@Component
public class MixedAspect {
    @Before("execution(* com.example.service.*.*(..))")
    public void doEverything() {
        // 로깅
        // 보안 체크
        // 성능 측정
        // 너무 많은 책임!
    }
}

// ✅ 좋은 예: 관심사별로 분리
@Aspect
@Order(1)
@Component
public class SecurityAspect {
    @Before("execution(* com.example.service.*.*(..))")
    public void checkSecurity() {
        // 보안 체크만
    }
}

@Aspect
@Order(2)
@Component
public class LoggingAspect {
    @Before("execution(* com.example.service.*.*(..))")
    public void log() {
        // 로깅만
    }
}

@Aspect
@Order(3)
@Component
public class PerformanceAspect {
    @Around("execution(* com.example.service.*.*(..))")
    public Object measurePerformance(ProceedingJoinPoint pjp) throws Throwable {
        // 성능 측정만
        return pjp.proceed();
    }
}
```

### 6.4 설정 외부화

```java
// application.yml
aop:
  logging:
    enabled: true
  performance:
    threshold: 1000
    enabled: true

// Aspect
@Aspect
@Component
public class ConfigurableAspect {

    @Value("${aop.logging.enabled:true}")
    private boolean loggingEnabled;

    @Value("${aop.performance.threshold:1000}")
    private long performanceThreshold;

    @Around("execution(* com.example.service.*.*(..))")
    public Object configurableAdvice(ProceedingJoinPoint pjp) throws Throwable {
        if (!loggingEnabled) {
            return pjp.proceed();
        }

        long start = System.currentTimeMillis();
        Object result = pjp.proceed();
        long duration = System.currentTimeMillis() - start;

        if (duration > performanceThreshold) {
            log.warn("Slow method: {} took {}ms",
                pjp.getSignature().getName(), duration);
        }

        return result;
    }
}
```

### 6.5 테스트 가능한 구조

```java
// ❌ 테스트하기 어려운 구조
@Aspect
@Component
public class HardToTestAspect {

    @Around("execution(* com.example.service.*.*(..))")
    public Object measure(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = pjp.proceed();
        long duration = System.currentTimeMillis() - start;

        // 하드코딩된 로거 - 테스트 어려움
        System.out.println("Duration: " + duration);

        return result;
    }
}

// ✅ 테스트 가능한 구조
@Aspect
@Component
public class TestableAspect {

    private final MetricsService metricsService;

    @Autowired
    public TestableAspect(MetricsService metricsService) {
        this.metricsService = metricsService;
    }

    @Around("execution(* com.example.service.*.*(..))")
    public Object measure(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = pjp.proceed();
        long duration = System.currentTimeMillis() - start;

        // 의존성 주입 - Mock으로 테스트 가능
        metricsService.record(pjp.getSignature().getName(), duration);

        return result;
    }
}

// 테스트 코드
@Test
void testAspect() {
    MetricsService mockService = mock(MetricsService.class);
    TestableAspect aspect = new TestableAspect(mockService);

    // 테스트 가능
    verify(mockService, times(1)).record(anyString(), anyLong());
}
```

### 6.6 AOP 남용 방지

**AOP를 사용하지 말아야 할 경우**:
- 핵심 비즈니스 로직
- 단순한 메서드 위임
- 한두 곳에서만 사용되는 로직

**AOP를 사용해야 할 경우**:
- 횡단 관심사 (로깅, 보안, 트랜잭션 등)
- 여러 곳에서 반복되는 공통 로직
- 비즈니스 로직과 독립적인 기능

```java
// ❌ 나쁜 예: AOP로 비즈니스 로직 구현
@Aspect
@Component
public class BusinessLogicAspect {
    @Around("execution(* com.example.service.OrderService.placeOrder(..))")
    public Object calculateDiscount(ProceedingJoinPoint pjp) throws Throwable {
        // 할인 계산 - 이건 비즈니스 로직!
        // AOP가 아니라 서비스 레이어에 있어야 함
        return pjp.proceed();
    }
}

// ✅ 좋은 예: 서비스에 비즈니스 로직
@Service
public class OrderService {
    public Order placeOrder(Order order) {
        // 할인 계산은 여기서
        order.applyDiscount(discountService.calculate(order));
        return orderRepository.save(order);
    }
}

// AOP는 횡단 관심사만
@Aspect
@Component
public class AuditAspect {
    @AfterReturning("execution(* com.example.service.OrderService.placeOrder(..))")
    public void auditOrder() {
        // 감사 로깅만
    }
}
```

### 6.7 명확한 문서화

```java
/**
 * 서비스 레이어 성능 모니터링 Aspect
 *
 * <p>모든 서비스 메서드의 실행 시간을 측정하고,
 * 임계값(기본 1000ms)을 초과하는 경우 경고를 기록합니다.</p>
 *
 * <h3>설정</h3>
 * <ul>
 *   <li>aop.performance.enabled: 성능 모니터링 활성화 여부 (기본: true)</li>
 *   <li>aop.performance.threshold: 경고 임계값 밀리초 (기본: 1000)</li>
 * </ul>
 *
 * <h3>적용 범위</h3>
 * <ul>
 *   <li>com.example.service 패키지 및 하위 패키지의 모든 public 메서드</li>
 * </ul>
 *
 * @author Your Name
 * @since 1.0
 * @see MetricsService
 */
@Aspect
@Order(10)
@Component
public class PerformanceMonitoringAspect {

    /**
     * 서비스 레이어의 모든 public 메서드
     */
    @Pointcut("execution(public * com.example.service..*.*(..))")
    public void serviceMethods() { }

    /**
     * 메서드 실행 시간을 측정하고 임계값 초과 시 경고
     *
     * @param pjp ProceedingJoinPoint
     * @return 메서드 실행 결과
     * @throws Throwable 메서드 실행 중 발생한 예외
     */
    @Around("serviceMethods()")
    public Object measurePerformance(ProceedingJoinPoint pjp) throws Throwable {
        // 구현
    }
}
```

---

## 7. FAQ

### Q1. @Aspect에 왜 @Component도 함께 붙여야 하나요?

**A**: `@Aspect`는 해당 클래스가 AOP의 Aspect임을 **선언**하는 역할만 하며, 스프링 컨테이너에 빈으로 등록하지는 않습니다. Spring AOP가 동작하려면 Aspect 클래스 자체가 스프링 빈이어야 하므로, `@Component`를 함께 사용하여 컴포넌트 스캔 대상으로 만들어야 합니다.

```java
// ❌ @Component 없으면 AOP가 동작하지 않음
@Aspect
public class MyAspect {
    // 스프링 빈이 아니므로 무시됨
}

// �� @Component로 빈 등록
@Aspect
@Component
public class MyAspect {
    // 스프링 빈으로 등록되어 AOP 동작
}
```

**대안**: `@Service`, `@Repository`, `@Configuration` 등 `@Component`를 포함하는 다른 스테레오타입 어노테이션도 사용 가능합니다.

### Q2. ProceedingJoinPoint는 언제 사용하나요?

**A**: `ProceedingJoinPoint`는 **오직 `@Around` Advice에서만** 사용할 수 있습니다. 그 이유는 `proceed()` 메서드를 통해 타겟 메서드의 실행을 직접 제어할 수 있기 때문입니다.

| Advice 타입 | 사용 가능한 파라미터 |
|------------|---------------------|
| `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing` | `JoinPoint` |
| `@Around` | `ProceedingJoinPoint` |

```java
// ❌ @Before에서는 ProceedingJoinPoint 사용 불가
@Before("execution(* com.example..*(..))")
public void before(ProceedingJoinPoint pjp) {  // 컴파일 에러!
    // ...
}

// ✅ @Before에서는 JoinPoint 사용
@Before("execution(* com.example..*(..))")
public void before(JoinPoint jp) {
    // ...
}

// ✅ @Around에서만 ProceedingJoinPoint 사용
@Around("execution(* com.example..*(..))")
public Object around(ProceedingJoinPoint pjp) throws Throwable {
    return pjp.proceed();  // 실행 제어 가능
}
```

### Q3. 여러 Aspect의 실행 순서는 어떻게 되나요?

**A**: `@Order` 어노테이션으로 우선순위를 지정할 수 있습니다.

**규칙**:
- **낮은 숫자 = 높은 우선순위**
- 진입 시: 높은 우선순위가 먼저 실행
- 종료 시: 높은 우선순위가 나중에 실행 (역순)

```java
@Aspect
@Order(1)  // 가장 먼저 실행
@Component
public class FirstAspect {
    @Before("execution(* com.example..*(..))")
    public void first() {
        System.out.println("1️⃣ First");
    }
}

@Aspect
@Order(2)
@Component
public class SecondAspect {
    @Before("execution(* com.example..*(..))")
    public void second() {
        System.out.println("2️⃣ Second");
    }
}

// 실행 결과:
// 진입: 1️⃣ First → 2️⃣ Second → 실제 메서드
// 종료: 실제 메서드 → 2️⃣ Second → 1️⃣ First (역순)
```

### Q4. Self-invocation 문제 해결법은?

**A**: 가장 권장되는 방법은 **코드 리팩토링**입니다.

**해결 방법 우선순위**:
1. **코드 리팩토링** (권장): 기능을 별도의 빈으로 분리
2. **Self Injection**: 자기 자신의 프록시를 주입받기
3. **AopContext**: `AopContext.currentProxy()` 사용 (비권장)

```java
// ✅ 방법 1: 리팩토링 (가장 권장)
@Service
public class OrderService {
    private final OrderNotificationService notificationService;

    public void createOrder(Order order) {
        orderRepository.save(order);
        notificationService.send(order);  // 다른 빈 호출
    }
}

// ✅ 방법 2: Self Injection
@Service
public class OrderService {
    @Autowired @Lazy
    private OrderService self;

    public void createOrder(Order order) {
        orderRepository.save(order);
        self.sendNotification(order);  // 프록시 호출
    }
}
```

### Q5. 성능이 중요할 때는 어떻게 하나요?

**A**: 다음과 같은 전략을 고려하세요:

1. **Pointcut 최적화**: 가능한 한 구체적인 표현식 사용
2. **경량화**: Hot path에는 최소한의 로직만 적용
3. **비동기 처리**: 무거운 작업은 비동기로 처리
4. **AspectJ 사용**: 극도의 성능이 필요하면 컴파일 타임 위빙 사용

```java
// ✅ 성능 최적화 예시
@Around("execution(* com.example.hotpath.*.*(..))")
public Object lightweight(ProceedingJoinPoint pjp) throws Throwable {
    long start = System.nanoTime();
    Object result = pjp.proceed();
    long duration = System.nanoTime() - start;

    // 무거운 작업은 비동기로
    if (duration > THRESHOLD) {
        executorService.submit(() -> heavyLogging(pjp, duration));
    }

    return result;
}
```

---

## 8. 참고 자료

### 공식 문서
- [Spring Framework AOP Documentation](https://docs.spring.io/spring-framework/reference/core/aop.html)
- [AspectJ Documentation](https://www.eclipse.org/aspectj/docs.php)

### 추가 학습 자료
- Spring AOP vs AspectJ 비교
- AOP 디자인 패턴
- 프록시 패턴의 이해

---

## 📝 마무리

Spring AOP의 **Pointcut, Advice, Advisor**는 횡단 관심사를 효과적으로 분리하고 관리하는 핵심 요소입니다.

**핵심 요약**:
- **Pointcut**: 어디에 적용할지 (Where)
- **Advice**: 무엇을 할지 (What)
- **Advisor**: Pointcut + Advice의 조합 (Complete Unit)

**실무 적용 시 기억할 점**:
1. ⚠️ **Self-invocation 문제 주의** (가장 흔한 실수)
2. 🎯 **최소 권한의 Advice 타입 사용**
3. 🔄 **Pointcut 재사용으로 중복 제거**
4. 📦 **관심사별로 Aspect 분리**
5. ⚡ **성능 고려사항 체크**

이 가이드가 Spring AOP를 실무에 효과적으로 적용하는 데 도움이 되길 바랍니다!
