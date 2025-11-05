# Jackson으로 시작하는 Java JSON 처리

## 들어가며

Java로 웹 개발을 하다 보면 JSON을 다룰 일이 정말 많습니다. REST API를 만들 때, 외부 API를 호출할 때, 설정 파일을 읽을 때... 이런 상황에서 가장 많이 사용되는 라이브러리가 바로 **Jackson**입니다.

사실 Spring Boot를 사용한다면 이미 Jackson을 쓰고 계실 겁니다. Spring Boot에 기본으로 포함되어 있거든요. 그만큼 Java 생태계에서 JSON 처리의 사실상 표준이라고 할 수 있습니다.

## Jackson이 뭔가요?

Jackson은 간단히 말해서 **Java 객체와 JSON 사이의 통역사** 역할을 합니다.

예를 들어, 이런 User 객체가 있다고 해볼까요?

```java
User user = new User("Alice", 30);
```

Jackson을 사용하면 이 객체를 이렇게 JSON으로 바꿀 수 있습니다:

```json
{"name":"Alice","age":30}
```

반대로 JSON 문자열이 있으면 다시 User 객체로 만들 수도 있죠. 이 과정을 전문 용어로:

- **직렬화(Serialization)**: 객체 → JSON
- **역직렬화(Deserialization)**: JSON → 객체

라고 부릅니다.

## 어떤 상황에서 사용할까요?

Jackson은 생각보다 다양한 곳에서 활약합니다:

**REST API 개발**
클라이언트에서 보낸 JSON 요청을 받아 Java 객체로 만들고, 처리 결과를 다시 JSON으로 응답할 때

**외부 API 연동**
다른 서비스의 API를 호출하고 JSON 응답을 파싱할 때

**메시징 시스템**
Kafka나 RabbitMQ 같은 메시지 큐에서 메시지를 주고받을 때

**설정 파일 관리**
JSON 형식의 설정 파일을 읽어서 애플리케이션 설정에 적용할 때

**NoSQL 데이터베이스**
MongoDB 같은 문서 기반 DB와 데이터를 주고받을 때

## Jackson의 주요 특징

### 1. 사용이 간단합니다

복잡한 설정 없이도 기본적인 JSON 변환이 바로 가능합니다. ObjectMapper 하나면 대부분의 작업을 처리할 수 있어요.

### 2. 어노테이션으로 커스터마이징

필드 이름을 바꾸고 싶거나, 특정 필드를 JSON에서 제외하고 싶을 때 어노테이션만 붙이면 됩니다.

- `@JsonProperty`: JSON 필드명 지정
- `@JsonIgnore`: 특정 필드 제외
- `@JsonFormat`: 날짜/시간 형식 지정

### 3. 다양한 타입 지원

기본 타입은 물론이고 List, Map, Set 같은 컬렉션, Enum, 심지어 제네릭 타입까지 모두 처리할 수 있습니다.

### 4. 높은 성능

스트림 기반 처리를 지원해서 메모리를 효율적으로 사용하고, 처리 속도도 빠릅니다. 대용량 JSON을 다룰 때도 안정적이죠.

### 5. 유연한 처리 방식

간단한 작업은 ObjectMapper로, 복잡한 작업은 JsonParser/JsonGenerator로, 상황에 맞게 선택할 수 있습니다.

## 시작하기

### 의존성 추가

Gradle을 사용한다면:

```gradle
implementation 'com.fasterxml.jackson.core:jackson-databind:2.15.2'
```

Maven을 사용한다면:

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.15.2</version>
</dependency>
```

Spring Boot를 사용하신다면? 별도 추가 필요 없습니다. `spring-boot-starter-web`에 이미 포함되어 있어요.

## 기본 사용법

### 객체를 JSON으로 변환하기

```java
ObjectMapper mapper = new ObjectMapper();
User user = new User("Alice", 30);

// 객체를 JSON 문자열로
String json = mapper.writeValueAsString(user);
System.out.println(json);
// 출력: {"name":"Alice","age":30}
```

### JSON을 객체로 변환하기

```java
String json = "{\"name\":\"Bob\",\"age\":25}";
User user = mapper.readValue(json, User.class);

System.out.println(user.getName()); // 출력: Bob
System.out.println(user.getAge());  // 출력: 25
```

### 컬렉션 다루기

List 형태의 JSON을 처리할 때는 TypeReference를 사용합니다:

```java
String jsonArray = "[{\"name\":\"Alice\",\"age\":30},{\"name\":\"Bob\",\"age\":25}]";

List<User> users = mapper.readValue(
    jsonArray,
    new TypeReference<List<User>>() {}
);

System.out.println(users.size()); // 출력: 2
```

## 어노테이션으로 커스터마이징하기

### 필드명 변경하기

JSON의 키 이름과 Java 필드명이 다를 때 사용합니다:

```java
public class User {
    @JsonProperty("username")
    private String name;

    private int age;

    // getters, setters
}
```

이제 JSON은 `{"username":"Alice","age":30}` 형태로 직렬화됩니다.

### 특정 필드 제외하기

민감한 정보나 불필요한 필드를 JSON에서 제외할 때:

```java
public class User {
    private String name;

    @JsonIgnore
    private String password;  // JSON에 포함되지 않음

    private int age;

    // getters, setters
}
```

### 날짜 포맷 지정하기

날짜/시간 필드의 포맷을 원하는 대로 지정할 수 있습니다:

```java
public class Event {
    private String title;

    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime eventTime;

    // getters, setters
}
```

### null 값 처리하기

null 값을 JSON에서 제외하고 싶을 때:

```java
@JsonInclude(JsonInclude.Include.NON_NULL)
public class User {
    private String name;
    private String email;  // null이면 JSON에 포함되지 않음

    // getters, setters
}
```

또는 ObjectMapper 전체에 적용:

```java
ObjectMapper mapper = new ObjectMapper();
mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
```

## 실무에서 알아두면 좋은 것들

### 1. ObjectMapper는 재사용하세요

ObjectMapper 생성 비용이 꽤 큽니다. 매번 새로 만들지 말고 싱글톤처럼 하나를 만들어서 재사용하는 게 좋습니다:

```java
public class JacksonConfig {
    private static final ObjectMapper MAPPER = new ObjectMapper();

    public static ObjectMapper getMapper() {
        return MAPPER;
    }
}
```

### 2. 날짜 처리는 명시적으로

Java 8의 LocalDateTime, LocalDate 등을 사용한다면 JavaTimeModule을 등록해야 합니다:

```java
ObjectMapper mapper = new ObjectMapper();
mapper.registerModule(new JavaTimeModule());
mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
```

아니면 `@JsonFormat` 어노테이션으로 포맷을 명시하세요.

### 3. 알 수 없는 필드 처리

JSON에 정의되지 않은 필드가 있을 때 에러가 나는 걸 방지하려면:

```java
mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
```

### 4. 대용량 JSON은 스트리밍으로

메모리가 부족할 정도로 큰 JSON을 다룰 때는 JsonParser를 사용하세요:

```java
JsonFactory factory = new JsonFactory();
try (JsonParser parser = factory.createParser(new File("large.json"))) {
    while (parser.nextToken() != null) {
        // 토큰 단위로 처리
        if (parser.getCurrentToken() == JsonToken.FIELD_NAME) {
            String fieldName = parser.getCurrentName();
            // 처리 로직
        }
    }
}
```

### 5. 커스텀 직렬화/역직렬화

복잡한 변환 로직이 필요하다면 Custom Serializer를 만들 수 있습니다:

```java
public class CustomUserSerializer extends JsonSerializer<User> {
    @Override
    public void serialize(User user, JsonGenerator gen, SerializerProvider serializers)
            throws IOException {
        gen.writeStartObject();
        gen.writeStringField("fullName", user.getFirstName() + " " + user.getLastName());
        gen.writeNumberField("age", user.getAge());
        gen.writeEndObject();
    }
}

// 사용
public class User {
    @JsonSerialize(using = CustomUserSerializer.class)
    // ...
}
```

## Spring Boot와 함께 사용하기

Spring Boot에서는 Jackson이 기본 내장되어 있어서 특별한 설정 없이도 바로 사용할 수 있습니다.

### 자동 변환

`@RestController`를 사용하면 자동으로 객체가 JSON으로 변환됩니다:

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        // User 객체를 반환하면 자동으로 JSON으로 변환됨
        return userService.findById(id);
    }

    @PostMapping
    public User createUser(@RequestBody User user) {
        // JSON 요청이 자동으로 User 객체로 변환됨
        return userService.save(user);
    }
}
```

### ObjectMapper 커스터마이징

전역 설정을 바꾸고 싶다면:

```java
@Configuration
public class JacksonConfig {

    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();

        // null 값 제외
        mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);

        // 날짜를 문자열로
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);

        // Java 8 날짜/시간 지원
        mapper.registerModule(new JavaTimeModule());

        // 알 수 없는 속성 무시
        mapper.disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);

        return mapper;
    }
}
```

또는 `application.yml`에서 설정:

```yaml
spring:
  jackson:
    serialization:
      write-dates-as-timestamps: false
    deserialization:
      fail-on-unknown-properties: false
    default-property-inclusion: non_null
```

## Jackson의 3가지 처리 방식

Jackson은 사용 목적에 따라 세 가지 API를 제공합니다:

### 1. Data Binding (가장 많이 사용)

ObjectMapper를 사용하는 방식으로, 객체와 JSON을 직접 매핑합니다. 가장 편하고 직관적이에요.

```java
ObjectMapper mapper = new ObjectMapper();
User user = mapper.readValue(json, User.class);
```

### 2. Tree Model

JSON을 트리 구조로 읽어서 탐색합니다. 구조가 복잡하거나 동적일 때 유용합니다:

```java
JsonNode root = mapper.readTree(json);
String name = root.get("name").asText();
int age = root.get("age").asInt();
```

### 3. Streaming API

대용량 데이터를 메모리 효율적으로 처리할 때 사용합니다. JsonParser와 JsonGenerator를 직접 다룹니다.

## 자주 하는 실수와 해결법

### 기본 생성자가 없어요

```java
// ❌ 이렇게만 있으면 역직렬화 실패
public class User {
    private String name;

    public User(String name) {
        this.name = name;
    }
}

// ✅ 기본 생성자 추가
public class User {
    private String name;

    public User() {} // 필수!

    public User(String name) {
        this.name = name;
    }
}
```

### getter/setter가 없어요

Jackson은 기본적으로 getter/setter를 통해 필드에 접근합니다:

```java
// ✅ 방법 1: getter/setter 추가
public class User {
    private String name;

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}

// ✅ 방법 2: 필드 직접 접근 허용
@JsonAutoDetect(fieldVisibility = JsonAutoDetect.Visibility.ANY)
public class User {
    private String name;
}
```

### 순환 참조 에러

양방향 관계에서 발생합니다:

```java
// ❌ 무한 루프 발생
public class Post {
    private List<Comment> comments;
}

public class Comment {
    private Post post;
}

// ✅ 해결 방법
public class Post {
    private List<Comment> comments;
}

public class Comment {
    @JsonBackReference  // 역방향 참조는 무시
    private Post post;
}
```

## 성능 최적화 팁

1. **ObjectMapper 재사용**: 싱글톤으로 만들어서 사용하세요
2. **어노테이션 활용**: 불필요한 필드는 `@JsonIgnore`로 제외
3. **스트리밍 API**: 대용량 JSON은 JsonParser 사용
4. **직렬화 뷰**: `@JsonView`로 필요한 필드만 선택적으로 직렬화
5. **캐싱**: 자주 사용하는 TypeReference는 재사용

## Jackson vs Gson

궁금해하실 수 있는 Gson과의 비교:

| 항목 | Jackson | Gson |
|------|---------|------|
| 성능 | 더 빠름 | 조금 느림 |
| 기능 | 풍부 (어노테이션, 스트리밍 등) | 심플 |
| 사용성 | 약간 복잡 | 매우 간단 |
| 생태계 | Spring 등 많은 프레임워크 기본 채택 | 주로 Android |
| 유지보수 | 활발 | 활발 |

결론: 특별한 이유가 없다면 Jackson을 선택하는 게 좋습니다. 특히 Spring 생태계를 사용한다면 더욱요.

## 마치며

Jackson은 Java에서 JSON을 다루는 가장 강력하고 유연한 도구입니다. 처음에는 ObjectMapper로 간단한 변환부터 시작하고, 필요에 따라 어노테이션과 고급 기능들을 익혀가시면 됩니다.

실무에서 가장 중요한 건:
- ObjectMapper를 재사용할 것
- 날짜 처리는 명시적으로 할 것
- 에러 처리를 잘 할 것 (알 수 없는 필드 등)

이것만 기억하셔도 대부분의 상황을 잘 처리하실 수 있을 겁니다.

## 참고 자료

- [Jackson 공식 문서](https://github.com/FasterXML/jackson-docs)
- [Baeldung Jackson Tutorial](https://www.baeldung.com/jackson)
- [Spring Boot Reference - Jackson](https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html#howto.spring-mvc.customize-jackson-objectmapper)
- [Jackson Annotations](https://github.com/FasterXML/jackson-annotations/wiki/Jackson-Annotations)

---

*작성일: 2025-11-03*
