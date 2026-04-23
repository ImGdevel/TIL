# DB Connection Management 정리

> 연결된 정리본:
> - [DB 커넥션과 트랜잭션 관리](../../../TIL.wiki/database-connection-and-transaction-management.md)


> “DB 커넥션은 ‘비싼 리소스’라서, 잘 관리하지 않으면 성능과 안정성이 같이 무너진다.”

---

## 1. 기본 개념

- **DB Connection**
  - 애플리케이션 프로세스 ↔ DB 서버 사이의 네트워크 세션
  - 생성/해제 비용이 크고, DB 서버가 동시에 처리할 수 있는 개수가 제한적이다.

- **Connection Pool**
  - 미리 일정 개수의 커넥션을 만들어 두고, 필요할 때 빌려 쓰고 반납하는 구조
  - 매 요청마다 새 커넥션을 만들지 않고 재사용해 성능/안정성을 확보한다.

핵심 아이디어:

- “커넥션은 자주 만들지 말고, 적당한 개수를 정해서 재사용하자”
- “풀의 크기를 DB 서버/애플리케이션 스펙에 맞게 맞춰야 한다”

---

## 2. 커넥션 풀 기본 구조

대표 구현체: HikariCP (Spring Boot 기본), Tomcat JDBC Pool, DBCP 등

공통적으로 가지고 있는 요소:

- 풀 크기: 최소/최대 커넥션 수
- 대기 큐: 커넥션을 다 써버렸을 때 대기하는 요청
- 타임아웃: 커넥션을 빌리거나 쿼리를 수행할 때 기다릴 수 있는 최대 시간
- 검증 로직: 죽은 커넥션/오래된 커넥션을 감지하고 교체하는 기능

흐름:

1. 애플리케이션이 커넥션을 요청한다.
2. 풀에 여유 커넥션이 있으면 바로 빌려준다.
3. 없으면 대기 큐에 들어가거나, 타임아웃 후 예외를 던진다.
4. 작업이 끝나면 `close()`를 호출해 커넥션을 **반납**한다. (실제로는 닫지 않고 풀로 돌아간다)

---

## 3. HikariCP 기준 핵심 파라미터

Spring Boot 2.x 이상에서 기본 커넥션 풀은 HikariCP다.

주요 설정 (application.yml 예시):

```yaml
spring:
  datasource:
    hikari:
      minimum-idle: 5
      maximum-pool-size: 20
      connection-timeout: 30000      # 커넥션 빌리는 최대 대기 시간(ms)
      idle-timeout: 600000           # 유휴 커넥션 정리 시간(ms)
      max-lifetime: 1800000          # 커넥션 최대 수명(ms)
      validation-timeout: 5000
```

설정 의미:

- `maximum-pool-size`
  - 동시에 활성화될 수 있는 커넥션 최대 수
  - 너무 크면 DB가 감당 못 하고, 너무 작으면 애플리케이션이 대기 시간 증가

- `connection-timeout`
  - 풀에 여유 커넥션이 없을 때 기다리는 최대 시간
  - 이 시간 안에 못 빌리면 예외 발생 (`SQLTransientConnectionException` 등)

- `max-lifetime`
  - 커넥션 하나가 살아 있을 수 있는 최대 시간
  - DB나 네트워크 장비에서 세션 타임아웃이 있을 경우, 그보다 약간 짧게 잡는 편이 안전

---

## 4. 풀 크기 설정 전략

### 4.1 DB와 애플리케이션 전체를 함께 고려

- **DB 서버의 최대 커넥션 수** (예: `max_connections`, `max_connections_per_hour` 등)
- **애플리케이션 인스턴스 수**
- 각 인스턴스별 풀 크기 합이 DB의 허용 범위를 넘지 않게 잡아야 한다.

예:

- DB `max_connections` = 200  
- 운영 인스턴스 = 4대  
- 각 인스턴스의 `maximum-pool-size` ≈ 40~45 (조금 여유를 남긴다)

### 4.2 트랜잭션/쿼리 시간이 길어질수록 풀 부담 증가

- 긴 쿼리/긴 트랜잭션은 **커넥션을 오래 붙잡고 있는 것**과 같다.
- 풀 크기를 키우기 전에:
  - 느린 쿼리 튜닝
  - 불필요하게 긴 트랜잭션 줄이기
  - 읽기/쓰기 분리
  등을 먼저 고려하는 게 좋다.

---

## 5. 자주 발생하는 문제와 패턴

### 5.1 커넥션 누수(Connection Leak)

**증상**

- 시간이 지날수록 풀에서 사용 중인 커넥션 수가 늘어만 간다.
- 결국 풀/DB가 꽉 차서 더 이상 커넥션을 빌릴 수 없게 된다.

**원인**

- 예외/분기 등으로 인해 `connection.close()`가 호출되지 않는 경우
- try-with-resources를 사용하지 않거나, 프레임워크가 관리하는 커넥션 범위를 벗어났을 때

**예방**

- Spring/JPA를 쓴다면 직접 커넥션을 열고 닫지 않고, `DataSource`, `JdbcTemplate`, `EntityManager`에 맡긴다.
- 직접 JDBC를 쓴다면 반드시 try-with-resources 사용:

```java
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql);
     ResultSet rs = ps.executeQuery()) {
    // ...
}
```

**HikariCP leakDetectionThreshold**

```yaml
spring:
  datasource:
    hikari:
      leak-detection-threshold: 20000 # 20초 이상 반환 안 되면 로그 경고
```

이 옵션을 켜면, 정한 시간 안에 반납되지 않은 커넥션에 대해 스택 트레이스를 찍어준다.

---

### 5.2 Pool Exhausted (풀 고갈)

**증상**

- `Connection is not available, request timed out` 등 예외
- 애플리케이션에서 DB 쿼리 시도 시 대기 타임아웃 후 실패

**원인**

- 트랜잭션/쿼리가 너무 오래 걸려 커넥션이 회수되지 않음
- 풀 크기가 실제 트래픽 대비 너무 작게 설정됨
- 커넥션 누수

**대응**

1. 느린 쿼리/긴 트랜잭션 제거/튜닝  
2. 인스턴스 수와 DB 설정을 고려해 풀 크기/DB max connection 재설정  
3. leak detection 켜고 누수 위치 파악

---

### 5.3 트랜잭션 경계와 커넥션 점유

- Spring의 `@Transactional`은 트랜잭션 범위 동안 커넥션을 붙잡고 있다.
- 불필요하게 넓은 범위에 `@Transactional`을 걸어두면,  
  그 시간 동안 커넥션을 다른 작업이 사용할 수 없다.

권장:

- 트랜잭션은 **가능한 한 짧게**  
- Controller 레벨보다는 Service 레벨에서, “정말 필요한 최소 작업 단위”에만 걸기

---

## 6. 모니터링 포인트

커넥션 풀 관련해서는 다음 지표들을 보는 게 좋다.

- Active connections (사용 중인 커넥션 수)
- Idle connections (대기 중인 커넥션 수)
- Pending requests (커넥션을 기다리는 요청 수)
- Connection acquisition time (커넥션 빌리는 데 걸리는 시간)
- 쿼리/트랜잭션 실행 시간 (특히 p95/p99)

HikariCP + Micrometer/Prometheus를 쓰면:

- `hikaricp.connections.active`  
- `hikaricp.connections.idle`  
- `hikaricp.connections.pending`  
- `hikaricp.connections.timeout`  
같은 메트릭을 바로 활용할 수 있다.

이 지표들을 보면서:

- 항상 Active가 Max에 가깝고 Pending이 많다면 → 풀/쿼리/트랜잭션 쪽 병목 의심  
- Idle이 너무 많고 Active가 적다면 → 풀 크기를 줄여도 되는지 검토

---

## 7. 정리 – “풀 크기”보다 먼저 볼 것들

DB 커넥션 문제를 겪을 때 처음 떠오르는 해결책은 “풀 크기를 늘리자”일 때가 많다.  
하지만 실제로는 그 전에 볼 게 많다.

- 쿼리가 너무 느리진 않은지 (인덱스, N+1, 비효율적인 조인)  
- 트랜잭션 범위가 과도하게 넓진 않은지  
- 커넥션을 적절히 반납하고 있는지 (누수 여부)  
- DB 서버의 `max_connections`/리소스와 애플리케이션 풀 설정이 맞는지

이 부분을 점검한 뒤에,

- 트래픽 패턴과 인스턴스 수를 고려해 풀 크기를 조정하고  
- 모니터링을 통해 Active/Idle/Pending이 어떤 패턴을 보이는지 관찰하면서 튜닝해 나가면,

“감에 의존한 커넥션 관리”에서 조금씩 벗어나 보다 예측 가능한 상태에 가까워질 수 있다.

