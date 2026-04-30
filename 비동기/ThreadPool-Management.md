# 쓰레드 풀(Thread Pool) 관리 전략 정리

> 연결된 정리본:
> - [비동기 전략과 쓰레드 풀 관리](../../../TIL.wiki/async-strategy-and-thread-pool-management.md)


> “동시성을 늘리려고 쓰레드를 늘렸는데, 오히려 시스템이 더 느려졌다…”

---

## 1. 쓰레드 풀, 한 줄 개념 정리

- **쓰레드 풀(Thread Pool)**  
  - 미리 만들어둔 쓰레드 여러 개를 재사용하면서 작업(Runnable/Callable)을 실행하는 구조
  - 매 요청마다 새 쓰레드를 만들지 않고, 풀에서 빌려 쓰고 다시 돌려준다.

**왜 필요한가?**

- 쓰레드 생성/파괴 비용을 줄인다.  
- 동시에 실행되는 쓰레드 수를 제어해 **CPU/메모리/컨텍스트 스위칭**을 관리한다.  
- 갑작스러운 트래픽 증가에도 시스템이 “완전히 죽지 않도록” 완충 역할을 한다.

---

## 2. ThreadPoolExecutor의 핵심 파라미터

Java 기준으로 쓰레드 풀은 보통 `ThreadPoolExecutor`로 표현된다.

### 2.1 주요 구성 요소

- `corePoolSize`
  - 항상 유지하려는 기본 쓰레드 수
- `maximumPoolSize`
  - 큐가 꽉 찼을 때까지 늘어날 수 있는 최대 쓰레드 수
- `workQueue`
  - 실행을 기다리는 작업이 쌓이는 큐 (예: `LinkedBlockingQueue`, `ArrayBlockingQueue`)
- `keepAliveTime`
  - core를 넘어선 추가 쓰레드가 놀고 있을 때, 언제까지 살려둘 것인지
- `threadFactory`
  - 쓰레드 이름/우선순위/데몬 여부 등을 설정
- `rejectedExecutionHandler`
  - 큐도 꽉 차 있고, 풀도 max까지 찼을 때 **거부 전략** (`AbortPolicy`, `CallerRunsPolicy` 등)

### 2.2 Executors.* 헬퍼가 가진 함정

Java에서 제공하는 `Executors.newXXX` 메서드들은 간단하지만, 실무에서는 주의가 필요하다.

- `newFixedThreadPool(n)`
  - 고정 개수 쓰레드 + **무한 대기열(LinkedBlockingQueue)**  
  - 큐 길이가 무제한이라, 느린 작업이 쌓이면 메모리가 터질 수 있다.
- `newCachedThreadPool()`
  - 쓰레드를 무제한으로 늘릴 수 있어, 트래픽 스파이크 시 쓰레드 폭발/컨텍스트 스위칭 폭발 위험

실무에서는 보통:

- **`ThreadPoolExecutor`를 직접 생성**하거나  
- Spring이면 `ThreadPoolTaskExecutor`를 사용하면서,
- **반드시 큐 크기/최대 쓰레드 수를 제한**하는 방향으로 설정한다.

---

## 3. 쓰레드 풀 사이즈 전략 – CPU vs I/O

쓰레드 풀은 “얼마나 많이 만들 것인가?”를 잘못 잡으면,  
CPU는 100%인데 처리량은 오히려 낮아지는 상황이 온다.

### 3.1 CPU 바운드 작업

예: 숫자 계산, 암호화, 압축 등 **CPU를 많이 사용하는 작업**

- 보통 CPU 코어 수에 맞춰 쓰레드 수를 잡는 게 유리하다.
- 대략적인 기준:

> `threadCount ≈ 코어 수` (또는 약간 더 적게)

이유:

- 코어 수보다 쓰레드가 훨씬 많으면, **컨텍스트 스위칭 오버헤드**가 커진다.

### 3.2 I/O 바운드 작업

예: DB/HTTP 호출, 파일 I/O 등 **대기 시간이 많은 작업**

- 쓰레드가 대부분 I/O 대기로 놀고 있기 때문에,  
  CPU 코어 수보다 더 많은 쓰레드를 써도 된다.

흔히 나오는 공식:

> `threadCount ≈ 코어 수 × (1 + (대기 시간 / 계산 시간))`

실무에서는 정확한 계산보다는:

- “대부분 I/O 대기”라면 코어 수 × 2~4 정도에서 시작하고  
- 모니터링을 보면서 점진적으로 조정하는 식으로 가져가는 편이 현실적이다.

### 3.3 혼합 작업

하나의 풀에서 CPU 작업과 I/O 작업을 같이 돌리면,

- CPU 작업이 I/O 작업을 밀어내거나, 그 반대가 생길 수 있다.

가능하다면:

- **역할별로 풀을 분리**하는 걸 고려한다.  
  - 예: `cpuBoundExecutor`, `ioBoundExecutor`를 따로 두기

---

## 4. 큐와 거부 정책(RejectedExecutionHandler)

### 4.1 큐 선택

- **유한 큐(예: `ArrayBlockingQueue`)**
  - 큐 길이를 제한해 메모리 폭발을 막는다.
  - 큐가 꽉 차면 거부 정책이 동작한다.
- **무한 큐(예: 기본 `LinkedBlockingQueue`)**
  - 설정이 편하지만, 부하가 길게 이어지면 메모리 사용량이 계속 늘어날 수 있다.

실무에서는 보통:

- “이 API는 최대 몇 개까지 대기 허용?”을 정해  
- 큐 크기를 **적당한 상수로 제한**하는 전략이 안전했다.

### 4.2 거부 정책

`RejectedExecutionHandler`는 큐/풀 모두 꽉 찼을 때 동작한다.

대표적인 정책:

- `AbortPolicy` (기본)
  - `RejectedExecutionException` 던짐
- `CallerRunsPolicy`
  - 현재 쓰레드(호출자)가 직접 실행
- `DiscardPolicy` / `DiscardOldestPolicy`
  - 조용히 버리거나, 가장 오래된 작업을 버린다.

어떤 정책을 쓸지는 **서비스 특성**에 따라 다르지만,

- API에서 거절을 명확히 알려야 한다면 `AbortPolicy` + 예외 처리  
- “어차피 처리 못 할 바엔 버리는 게 낫다”면 discard 계열  
- 부하를 호출자 쪽으로 밀어내고 싶다면 `CallerRunsPolicy`

등으로 선택할 수 있다.

---

## 5. Spring에서의 쓰레드 풀 관리 (@Async, ThreadPoolTaskExecutor)

Spring을 쓰는 경우, `@Async`와 `ThreadPoolTaskExecutor`를 같이 쓰는 패턴이 많다.

### 5.1 기본 패턴

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}
```

```java
@Service
public class NotificationService {

    @Async
    public void sendEmail(...) {
        // 비동기 이메일 전송 로직
    }
}
```

주의할 점:

- 별도 설정 없이 `@Async`만 쓰면, `SimpleAsyncTaskExecutor`가 쓰이는데  
  **실제 풀링이 아니라 “요청마다 새 쓰레드 생성”에 가깝다.**  
  → 반드시 `ThreadPoolTaskExecutor` 같은 풀 기반 executor를 설정하자.

### 5.2 AOP/self-invocation 이슈

`@Async`도 내부적으로 AOP 프록시를 사용하기 때문에:

- 같은 클래스 내부에서 `this.asyncMethod()`를 호출하면,  
  @Async가 적용되지 않고 **동기 실행**될 수 있다.
- 별도 빈으로 분리하거나, 외부에서 호출되도록 구조를 바꾸는 게 안전하다.

---

## 6. 모니터링과 튜닝 포인트

쓰레드 풀은 “만들어 놓고 끝”이 아니라,  
**지표를 보면서 조정해야 제 역할을 한다.**

볼 만한 지표:

- Active thread count (현재 실행 중인 쓰레드 수)  
- Pool size / Core size / Max size  
- Queue size / Queue remaining capacity  
- Task completion rate, 평균 처리 시간  
- 거부된 작업 수 (rejected count)

Spring + Micrometer/Prometheus 등을 쓰면:

- `ThreadPoolTaskExecutor`를 MeterRegistry에 등록해  
  위 지표들을 메트릭으로 수집/대시보드화할 수 있다.

튜닝 방향:

- 큐가 항상 가득 차 있다면 → 풀/큐 사이즈를 늘리거나, 작업 자체를 가볍게  
- 반대로 풀/큐 모두 항상 한가하다면 → 리소스를 줄여도 되는지 검토

---

## 7. 자주 하는 실수와 주의점

1. **무한 큐 + 고정 쓰레드 풀 조합**
   - `newFixedThreadPool` 기본 전략  
   - 느린 작업이 많이 들어오면 큐가 끝없이 쌓이면서 메모리 압박 → OOM 위험

2. **쓰레드 풀을 종료하지 않음**
   - `ExecutorService`를 직접 사용할 때 `shutdown()`/`shutdownNow()`를 호출하지 않으면,  
     애플리케이션이 종료되지 않거나, 테스트가 끝없이 대기하는 상황이 생길 수 있다.

3. **ForkJoinPool.commonPool()에 블로킹 작업 밀어 넣기**
   - parallel stream, `CompletableFuture.supplyAsync()` 기본 executor 등  
   - 여기에 I/O 블로킹 작업을 많이 넣으면, 다른 parallel 연산까지 영향을 받을 수 있다.

4. **CPU 작업과 I/O 작업을 같은 풀에서 돌리기**
   - 특정 타입 작업이 풀을 잠가 버려, 다른 작업이 굶을 수 있다.  
   - 가능하면 성격에 따라 풀을 분리한다.

5. **쓰레드 개수만 늘리면 성능이 무조건 좋아진다고 믿기**
   - 어느 지점부터는 쓰레드가 늘어날수록 컨텍스트 스위칭과 캐시 미스 때문에 **성능이 떨어진다.**  
   - 항상 모니터링과 벤치마크를 통해 최적점을 찾는 게 필요하다.

---

## 8. 마무리 – “몇 개 만들까?”가 아니라 “어떤 일을 맡길까?”부터

쓰레드 풀을 쓰면서 느낀 점은,

> “쓰레드 풀을 몇 개, 몇 개의 쓰레드로 만들까?”보다  
> “어떤 종류의 일을 어떤 풀에 맡길까?”를 먼저 정하는 게 훨씬 중요하다.

이 문서의 내용을 기준으로,

- 우리 서비스에서 CPU 바운드 / I/O 바운드 작업이 어디에 있는지  
- 현재 쓰레드 풀/큐 설정이 그 작업 특성에 맞게 되어 있는지  
- 모니터링 지표를 통해 병목이 어디서 생기고 있는지

를 한 번 점검해 보면 좋다.  
그 위에서 조금씩 풀/큐/정책을 조정해 나가면, “감으로 쓰던 동시성”이 조금씩 더 통제 가능한 상태에 가까워진다.

