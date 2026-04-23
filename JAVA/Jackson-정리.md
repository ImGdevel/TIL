# Jackson 정리

> 연결된 정리본:
> - [Jackson과 JSON 처리](../../../TIL.wiki/jackson-json-processing.md)


## 1. Jackson이란?

- Java 진영에서 가장 널리 쓰이는 JSON 직렬화/역직렬화 라이브러리
- 객체(Object) ↔ JSON 문자열 간 변환을 담당
- Spring MVC, Spring WebFlux 등에서 기본으로 사용 (`MappingJackson2HttpMessageConverter`)
- 기본 핵심 모듈: `jackson-core`, `jackson-databind`, `jackson-annotations`

### 핵심 개념

- `ObjectMapper`: Jackson의 핵심 클래스. 직렬화/역직렬화 동작과 설정을 모두 여기서 관리.
- 직렬화(Serialization): Java 객체 → JSON 문자열로 변환
- 역직렬화(Deserialization): JSON 문자열 → Java 객체로 변환
- 애노테이션 기반 제어: `@JsonProperty`, `@JsonIgnore`, `@JsonInclude`, `@JsonFormat` 등으로 필드별 제어 가능

---

## 2. Jackson 사용 시 자주 발생하는 문제와 해결법

### 2.1 순환 참조(무한 재귀) 문제

**증상**
- 양방향 연관관계가 있을 때, 직렬화 과정에서 스택오버플로(StackOverflowError) 또는 무한 JSON 생성
  - 예: `Member` ↔ `Team` 양방향 연관관계

**원인**
- Jackson이 객체 그래프를 그대로 탐색하면서 직렬화하기 때문에, A → B → A → B ... 구조에서 무한 순환 발생

**해결 방법**
- `@JsonManagedReference` / `@JsonBackReference` 사용
  - 주로 “부모” 쪽에 `@JsonManagedReference`, “자식/역방향” 쪽에 `@JsonBackReference` 적용
- `@JsonIgnore`로 한쪽 참조를 직렬화 대상에서 제외
- `@JsonIdentityInfo`를 사용해 ID 기반으로 순환 참조를 끊고 동일 객체를 식별
- API 스펙 자체를 단방향으로 설계해 순환 구조를 제거 (가장 근본적인 해결)

---

### 2.2 날짜/시간 포맷 문제

**증상**
- `LocalDate`, `LocalDateTime`, `ZonedDateTime` 등을 JSON으로 변환할 때
  - 예상과 다른 포맷
  - 역직렬화 실패 (`com.fasterxml.jackson.databind.exc.InvalidFormatException` 등)

**원인**
- Java 8 날짜/시간 타입에 대한 모듈 미설치 혹은 포맷 미지정

**해결 방법**
- `jackson-datatype-jsr310` 의존성 추가 후 `ObjectMapper`에 등록
  - Spring Boot 2.x 이상은 보통 자동 등록되지만, 환경에 따라 추가 설정이 필요할 수 있음
- 필드에 `@JsonFormat`으로 포맷 지정
  - 예: `@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "Asia/Seoul")`
- 전역 설정
  - Spring Boot: `application.yml`에서 `spring.jackson.date-format`, `spring.jackson.time-zone` 등을 설정

---

### 2.3 알 수 없는 필드(Unknown Property) 문제

**증상**
- JSON에 Java 객체에 없는 필드가 포함되면 역직렬화 실패
  - 예: `UnrecognizedPropertyException` 발생

**원인**
- 기본 설정에서 “존재하지 않는 필드”를 허용하지 않도록 되어 있는 경우

**해결 방법**
- 클래스에 `@JsonIgnoreProperties(ignoreUnknown = true)` 적용
- 혹은 `ObjectMapper` 설정:
  - `objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);`
- API 계약이 명확하다면, 오타/필드 변경 등 계약 위반이 없는지 우선 확인 후 설정 조정

---

### 2.4 필드 이름/케이싱 문제 (snake_case vs camelCase)

**증상**
- JSON 필드 이름과 Java 필드 이름이 맞지 않아 매핑 실패
  - 예: JSON은 `user_name`, Java 필드는 `userName`

**원인**
- 기본 전략은 Java 필드명을 그대로 사용 (camelCase)

**해결 방법**
- 필드에 `@JsonProperty("user_name")` 명시
- 전역 네이밍 전략 변경:
  - `objectMapper.setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);`
- Spring Boot:
  - `application.yml`에서 `spring.jackson.property-naming-strategy: SNAKE_CASE`

---

### 2.5 Enum 직렬화/역직렬화 문제

**증상**
- Enum 이름 변경 후 기존 JSON과 호환성 깨짐
- JSON 값이 Enum 상수와 정확히 일치하지 않아 에러 발생

**원인**
- 기본 전략은 Enum의 `name()` 값을 사용

**해결 방법**
- `@JsonValue`, `@JsonCreator`로 Enum 직렬화/역직렬화 전략 커스터마이징
- String → Enum 변환 실패 시의 처리 방식을 정의 (`READ_UNKNOWN_ENUM_VALUES_AS_NULL` 등 옵션)
- 외부에 노출되는 값은 Enum `name()` 대신 별도 코드 필드를 사용하고, JSON에는 코드만 사용하도록 설계

---

### 2.6 컬렉션/제네릭 타입 역직렬화 문제

**증상**
- `List<MyDto>`, `Map<String, MyDto>` 같은 제네릭 타입 역직렬화 시 타입 정보 손실로 인한 예외

**원인**
- 런타임 시 제네릭 타입 정보가 소거(erasure)되기 때문에, Jackson이 구체 타입 정보를 알 수 없음

**해결 방법**
- `new TypeReference<List<MyDto>>() {}` 형식으로 타입 정보를 명시
  - 예: `objectMapper.readValue(json, new TypeReference<List<MyDto>>() {});`
- Spring MVC에서는 주로 메서드 시그니처나 `ResponseEntity<List<MyDto>>` 형태를 사용하면 자동 처리되는 경우가 많음

---

### 2.7 직렬화 대상/비대상 필드 제어 문제

**증상**
- 민감 정보(비밀번호 등)가 JSON에 노출되거나, 특정 필드가 직렬화/역직렬화에서 빠짐

**원인**
- Jackson 애노테이션/기본 전략 미사용 혹은 잘못된 사용

**해결 방법**
- `@JsonIgnore`로 특정 필드 완전 제외
- `@JsonInclude(Include.NON_NULL)` / `Include.NON_EMPTY` 등으로 필요할 때만 포함
- 요청용 DTO / 응답용 DTO를 분리해, 엔티티를 직접 외부에 노출하지 않는 구조로 설계

---

### 2.8 성능 및 메모리 문제

**증상**
- 대용량 JSON 처리 시 GC 부하, 응답 지연, OOM 발생 가능

**원인**
- 한 번에 모든 JSON을 메모리에 올려놓고 처리

**해결 방법**
- Jackson 스트리밍 API 사용 (`JsonParser`, `JsonGenerator`)으로 스트림 처리
- `ObjectReader`, `ObjectWriter` 재사용을 통해 반복 생성 비용 감소
- Spring WebFlux + Jackson 사용 시, back-pressure를 고려한 설계

---

## 3. Jackson을 사용할 때의 기본 팁

- `ObjectMapper`를 매번 새로 생성하지 말고, 빈/싱글턴으로 재사용
- 도메인 엔티티에 Jackson 애노테이션을 과도하게 섞기보다는, DTO를 별도로 두고 Jackson 관련 애노테이션은 DTO에만 두는 것을 우선 고려
- Jackson 버전과 Spring Boot 버전을 맞춰서 사용하는 것이 중요 (호환성 이슈 방지)
- 테스트 코드에서 직렬화/역직렬화 케이스를 꼭 추가해, 실제 API 스펙과 일치하는지 검증

