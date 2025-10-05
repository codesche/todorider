# Java 버전 비교 (8, 11, 17)

## Java 8 vs 11 vs 17 버전별 주요 차이점

### 요약
- **Java 8**: 현대 Java의 시작
- **Java 11**: 실용성 강화
- **Java 17**: 현대 Java의 완성

---

## Java 8: 현대 Java의 시작

### 핵심 기능

#### 1. Lambda 표현식
익명 함수를 간결하게 표현하는 방법이다. 기존의 익명 클래스를 대체하여 코드를 훨씬 간결하게 만든다.

```java
// Before - 익명 클래스 사용 (장황함)
list.sort(new Comparator<String>() {
    public int compare(String s1, String s2) {
        return s1.compareTo(s2);
    }
});

// Java 8 - Lambda 표현식
list.sort((s1, s2) -> s1.compareTo(s2));

// 메서드 레퍼런스 (더 간결하게)
list.sort(String::compareTo);
```

핵심: 코드가 5줄에서 1줄로 줄어들기에 가독성이 향상된다.

#### 2. Stream API
컬렉션 데이터를 선언적으로 처리할 수 있게 해준다. “어떻게”가 아닌 “무엇을” 할지 코드로 표현한다.

```java
// 이름이 A로 시작하는 것만 필터링하고 대문자로 변환
List<String> result = names.stream()
    .filter(name -> name.startsWith("A"))  // 필터링
    .map(String::toUpperCase)              // 변환
    .collect(Collectors.toList());         // 결과 수집
```

장점:
- 코드가 읽기 쉬워진다
- 병렬 처리가 간단하다(`.parallelStream()`)
- 중간 연산은 지연 실행된다 (성능 최적화)

#### 3. Optional
`null`을 안전하게 다루기 위한 컨테이너 클래스다. `NullPointerException`을 방지한다.

```java
// Before Optional:
if (value != null) {
    return value.toUpperCase();
} else {
    return "DEFAULT";
}
```

```java
// null일 수 있는 값을 Optional로 감싸기
Optional<String> optional = Optional.ofNullable(value);

// 값이 있으면 대문자로, 없으면 "DEFAULT" 반환
String result = optional
        .map(String::toUpperCase)
        .orElse("DEFAULT");
```

핵심: `null` 체크 코드가 사라지고 메서드 체이닝으로 간결해진다.

#### 4. java.time API
기존 `Date`, `Calendar`의 문제점을 해결한 새로운 날짜/시간 API다. **불변 객체**이며 **Thread-Safe** 하다.

```java
// 현재 날짜/시간
LocalDateTime now = LocalDateTime.now();

// 특정 날짜 생성
LocalDate date = LocalDate.of(2024, 1, 1);

// 타임존 포함
ZonedDateTime seoul = ZonedDateTime.now(ZoneId.of("Asia/Seoul"));

// 날짜 계산
LocalDate nextWeek = date.plusWeeks(1);
LocalDate lastMonth = date.minusMonths(1);
```

장점:
- Immutable (불변)
- Thread-Safe (멀티스레드 환경에서 안전)
- 명확한 API (year, month, day 명시적)

#### 5. Interface Default 메서드
인터페이스에 구현된 메서드를 추가할 수 있다. 기존 코드를 깨지 않고 인터페이스를 확장할 수 있다.

```java
interface Vehicle {
    void start(); // 추상 메서드

    // 기본 구현을 가진 메서드
    default void stop() {
        System.out.println("Stopped");
    }
}

// 구현 클래스는 stop()을 오버라이드하지 않아도 됨
class Car implements Vehicle {
    public void start() {
        System.out.println("Car started");
    }
    // stop()은 자동으로 상속됨
}
```

> 활용: Java 8의 Stream API도 이 기능으로 추가되었다 (Collection 인터페이스에 `.stream()` 메서드).

---

## Java 11: 실용성 강화

### 핵심 기능

#### 1. `var` 키워드 (타입 추론)
Java 10에서 도입되어 Java 11 LTS에 포함되었다. 변수 타입을 컴파일러가 추론하게 해준다.

```java
// 타입을 명시적으로 작성
ArrayList<String> names = new ArrayList<String>();
HashMap<String, List<Integer>> map = new HashMap<String, List<Integer>>();

// var로 간결하게 (타입은 컴파일러가 추론)
var names2 = new ArrayList<String>();
var map2 = new HashMap<String, List<Integer>>();
```

장점: 긴 제네릭 타입을 반복하지 않아도 된다.

주의사항:
```java
// ❌ 이런 경우는 사용 불가
var x;                 // 초기화 필수
var y = null;          // null 불가
var lambda = () -> "Hello"; // Lambda 불가

// ❌ 필드나 메서드 파라미터에는 사용 불가
class Example {
    var field = "value";   // 컴파일 에러
    void method(var param) { } // 컴파일 에러
}
```

#### 2. String 메서드 추가
문자열 처리를 더 편리하게 만드는 메서드들이 추가되었다.

```java
// isBlank() - 공백 문자열 체크 (공백 문자 포함)
" ".isBlank();  // true
"".isBlank();   // true
"a".isBlank();  // false

// isEmpty()와의 차이
" ".isEmpty();  // false (공백이 있으니 비어있지 않음)
"".isEmpty();   // true

// lines() - 줄 단위로 분리
String text = "Line1\nLine2\nLine3";
text.lines().forEach(System.out::println); // 출력: Line1, Line2, Line3 (각각 한 줄씩)

// repeat() - 문자열 반복
String line = "-".repeat(50); // --------------------------------------------------

// strip() - 유니코드 공백까지 제거 (trim()보다 강력)
"   Hello   ".strip(); // "Hello"
```

> 활용: 데이터 검증 시 `isBlank()` 가 매우 유용하다.

#### 3. 컬렉션 팩토리 메서드 (Java 9)
불변 컬렉션을 간단하게 생성할 수 있다.

```java
// Before - Arrays.asList() 사용
List<String> list = Arrays.asList("A", "B", "C");
list.add("D"); // UnsupportedOperationException

// Java 9 이후 - List.of()
List<String> list2 = List.of("A", "B", "C");
Set<String> set = Set.of("A", "B", "C");
Map<String, Integer> map3 = Map.of("A", 1, "B", 2);
```

특징:
- 완전한 불변 컬렉션 (수정 불가)
- `null` 값 불가
- 간결한 문법

#### 4. HTTP Client
표준 HTTP 클라이언트가 추가되었다. 기존에는 외부 라이브러리(Apache HttpClient)를 사용해야 했다.

```java
// HTTP Client 생성
HttpClient client = HttpClient.newHttpClient();

// 요청 생성
HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/users"))
        .header("Authorization", "Bearer token")
        .GET()
        .build();

// 동기 요청
HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

// 비동기 요청 (CompletableFuture)
client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
      .thenApply(HttpResponse::body)
      .thenAccept(System.out::println);
```

장점:
- 외부 라이브러리 불필요
- HTTP/2 지원
- 비동기 요청 간편

#### 5. Files 메서드
파일을 `String`으로 쉽게 읽고 쓸 수 있다.

```java
// 파일을 String으로 읽기
String content = Files.readString(Path.of("file.txt"));

// 파일에 String 쓰기
Files.writeString(Path.of("output.txt"), "Hello, World!");

// Before (Java 8)
List<String> lines = Files.readAllLines(Path.of("file.txt"));
String content2 = String.join("\n", lines);
```

#### 6. Optional 개선
```java
Optional<String> opt = Optional.of("value");

// isEmpty() - isPresent()의 반대
opt.isEmpty(); // false

// ifPresentOrElse() - 값이 있을 때와 없을 때 각각 처리
opt.ifPresentOrElse(
    value -> System.out.println("값: " + value),
    () -> System.out.println("값 없음")
);

// or() - 다른 Optional 제공
Optional<String> result = opt.or(() -> Optional.of("default"));
```

---

## Java 17: 현대 Java의 완성

### 핵심 기능

#### 1. Record
데이터를 담는 **불변 클래스**를 한 줄로 정의할 수 있다. DTO, VO를 만들 때 매우 유용하다.

```java
public record UserResponse(
    Long id,
    String name,
    String email,
    LocalDateTime createdAt
) {
    // 추가 검증 로직 (compact constructor)
    public UserResponse {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Name required");
        }
    }
}
```

#### 2. Sealed Classes
클래스 상속을 제한할 수 있다. “이 클래스는 특정 클래스만 상속 가능”하다고 명시한다.

```java
// Shape는 Circle, Rectangle, Triangle만 상속 가능
public sealed class Shape permits Circle, Rectangle, Triangle {}

// 허용된 하위 클래스들
public final class Circle extends Shape { }        // final: 더 이상 상속 불가
public final class Rectangle extends Shape { }
public non-sealed class Triangle extends Shape { } // non-sealed: 추가 상속 허용
```

왜 필요한가?
- 제한된 상속 계층으로 코드 안정성 향상
- 모든 하위 타입을 알 수 있어 `switch`에서 완전성 체크 가능

#### 3. Pattern Matching (instanceof)
`instanceof` 검사 후 바로 해당 타입으로 사용할 수 있다. 캐스팅이 사라진다.

```java
// Before - instanceof 후 캐스팅
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}

// Java 17 - Pattern Matching
if (obj instanceof String s) {
    System.out.println(s.length()); // 바로 사용 가능!
}

// 조건과 함께 사용
if (obj instanceof String s && s.length() > 5) {
    System.out.println(s.toUpperCase());
}
```

장점: 타입 체크와 캐스팅을 한 번에 처리

#### 4. Pattern Matching (switch) - Preview
`switch`에서 타입 패턴 매칭을 사용할 수 있다.

```java
// 타입별로 다른 처리
static String format(Object obj) {
    return switch (obj) {
        case Integer i -> "정수: " + i;
        case String s  -> "문자열: " + s;
        case null      -> "null 값";
        default        -> "알 수 없음";
    };
}

// Guard 패턴 (when 절) - 추가 조건
static String check(Object obj) {
    return switch (obj) {
        case String s when s.length() > 5 -> "긴 문자열";
        case String s                     -> "짧은 문자열";
        case Integer i when i > 0         -> "양수";
        case Integer i                    -> "0 또는 음수";
        default                           -> "기타";
    };
}
```

활용: 다형성 처리, 타입별 분기 처리에 유용

#### 5. Text Blocks
여러 줄 문자열을 쉽게 작성할 수 있다. JSON, SQL, HTML 작성 시 매우 편리하다.

```java
// Before - 이스케이프 문자로 가독성 최악
String json = "{\n" +
              "  \"name\": \"John\",\n" +
              "  \"age\": 30\n" +
              "}";

// Java 17 - Text Blocks
String json2 = \"\"\"
{
  "name": "John",
  "age": 30
}
\"\"\";

// SQL 쿼리
String query = \"\"\"
SELECT u.id, u.name, o.order_date
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.age > 18
ORDER BY o.order_date DESC
\"\"\";

// 변수 삽입 (.formatted() 사용)
String name = "John";
int age = 30;
String message = \"\"\"
이름: %s
나이: %d세
\"\"\".formatted(name, age);
```

장점:
- 가독성 향상
- 이스케이프 문자 불필요
- 들여쓰기 자동 처리

#### 6. `Stream.toList()`
`Stream`을 `List`로 변환하는 간편한 메서드가 추가되었다.

```java
// Before
List<String> list = stream.collect(Collectors.toList());

// Java 17
List<String> list2 = stream.toList(); // 불변 리스트 반환
```

주의: `toList()`는 **불변 리스트**를 반환한다.

---

## 주요 기능 비교

| 기능 | Java 8 | Java 11 | Java 17 |
|---|---|---|---|
| Lambda & Stream | ✅ | ✅ | ✅ |
| Optional | ✅ | ✅ (개선) | ✅ (개선) |
| java.time | ✅ | ✅ | ✅ |
| var |  | ✅ | ✅ |
| HTTP Client |  | ✅ | ✅ |
| 컬렉션 팩토리 |  | ✅ (Java 9) | ✅ |
| Record |  |  | ✅ |
| Sealed Classes |  |  | ✅ |
| Pattern Matching |  |  | ✅ |
| Text Blocks |  |  | ✅ |

---

## 버전 선택 가이드

### Java 8
- 안정성 검증됨
- 대부분 라이브러리 호환
- 현대적 기능 부족  
**추천:** 레거시 유지보수

### Java 11
- 안정성 + 최신 기능 균형
- 프로덕션 검증 완료
- Record, Text Blocks 없음  
**추천:** 안정성 우선 프로젝트

### Java 17 (권장)
- 최신 LTS (2029년까지)
- Record로 생산성 향상
- 성능 최적화  
**추천:** 신규 프로젝트

---

## 실무 활용

### 1. 대용량 데이터 처리 (Stream + 병렬 처리)
```java
// 100만 건 데이터 필터링
List<User> activeUsers = users.parallelStream()
    .filter(User::isActive)
    .filter(u -> u.getAge() > 18)
    .collect(Collectors.toList());
```

### 2. API 응답 DTO (Record 활용)
```java
// Java 17
public record UserResponse(
    Long id,
    String name,
    String email,
    LocalDateTime createdAt
) {
    // 추가 검증 로직
    public UserResponse {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Name required");
        }
    }
}
```

### 3. 에러 처리 (Optional + Stream)
```java
public Optional<User> findUserByEmail(String email) {
    return userRepository.findByEmail(email);
}

// 사용
String userName = findUserByEmail(email)
        .map(User::getName)
        .orElseThrow(() -> new UserNotFoundException(email));
```

### 4. 날짜 처리 (java.time)
```java
// 30일 이내 데이터 조회
LocalDateTime thirtyDaysAgo = LocalDateTime.now().minusDays(30);

List<Order> recentOrders = orders.stream()
    .filter(o -> o.getCreatedAt().isAfter(thirtyDaysAgo))
    .collect(Collectors.toList());
```

### 5. HTTP 통신 (Java 11 HTTP Client)
```java
HttpClient client = HttpClient.newHttpClient();

HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/users"))
    .header("Authorization", "Bearer " + token)
    .GET()
    .build();

CompletableFuture<String> response = client
    .sendAsync(request, HttpResponse.BodyHandlers.ofString())
    .thenApply(HttpResponse::body);
```

---

## Java 8 vs 11 vs 17 버전 정보

| 버전 | 출시일 | LTS | 지원 종료 |
|---|---|---|---|
| Java 8  | 2014.03 | LTS | 2032.01 |
| Java 11 | 2018.09 | LTS | 2031.09 |
| Java 17 | 2021.09 | LTS | 2029.09 |

---

## 주의사항 및 Best Practices

### Java 8
```java
// ✅ Good
list.stream()
    .filter(x -> x > 0)
    .collect(Collectors.toList());

// ❌ Bad - Stream 재사용 불가
Stream<String> stream = list.stream();
stream.forEach(System.out::println);
stream.count(); // IllegalStateException
```

### Java 11
```java
// ✅ Good - var 사용 적절한 경우
var users = new ArrayList<User>();
var result = userService.findAll();

// ❌ Bad - var 부적절한 경우
var data = getData(); // 타입 불명확
var x = null;         // 컴파일 에러
```

### Java 17
```java
// ✅ Good - Record 활용
public record Point(int x, int y) {
    public Point {
        if (x < 0 || y < 0) {
            throw new IllegalArgumentException();
        }
    }
}

// ❌ Bad - Record는 불변
public record Person(String name) {
    public void setName(String name) { } // 불가능
}
```

---

## 출처
- 원문: https://codesche.oopy.io/283de3f7-e3a8-80aa-aab5-f3113b5a4ae3
