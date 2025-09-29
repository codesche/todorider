# 이펙티브 자바 Item 2: 생성자에 매개변수가 많다면 빌더를 고려하라

## 들어가며

정적 팩터리와 생성자에는 똑같은 제약이 하나 있다. 선택적 매개변수가 많을 때 적절히 대응하기 어렵다는 점이다. 식품 포장의 영양정보를 표현하는 클래스를 생각해보자. 필수 항목 몇 개와 선택 항목 20개가 넘는 상황이라면 어떻게 해야 할까?

## 전통적인 해결책들의 문제점

### 1. 점층적 생성자 패턴 (Telescoping Constructor Pattern)

```java
public class NutritionFacts {
    private final int servingSize;  // (mL, 1회 제공량)     필수
    private final int servings;     // (회, 총 n회 제공량)  필수
    private final int calories;     // (1회 제공량당)       선택
    private final int fat;          // (g/1회 제공량)       선택
    private final int sodium;       // (mg/1회 제공량)      선택
    private final int carbohydrate; // (g/1회 제공량)       선택

    public NutritionFacts(int servingSize, int servings) {
        this(servingSize, servings, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories) {
        this(servingSize, servings, calories, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories, int fat) {
        this(servingSize, servings, calories, fat, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories, int fat, int sodium) {
        this(servingSize, servings, calories, fat, sodium, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories, int fat, int sodium, int carbohydrate) {
        this.servingSize = servingSize;
        this.servings = servings;
        this.calories = calories;
        this.fat = fat;
        this.sodium = sodium;
        this.carbohydrate = carbohydrate;
    }
}
```

**문제점:**
```java
// 실제 사용 시 - 매개변수 순서를 기억하기 어렵고 실수하기 쉽다
NutritionFacts cocaCola = new NutritionFacts(240, 8, 100, 0, 35, 27);
```

### 2. 자바빈즈 패턴 (JavaBeans Pattern)

```java
public class NutritionFacts {
    // 매개변수들은 기본값으로 초기화 (필수 매개변수는 기본값이 없으니 적절치 않다)
    private int servingSize = -1; // 필수; 기본값 없음
    private int servings = -1;    // 필수; 기본값 없음
    private int calories = 0;
    private int fat = 0;
    private int sodium = 0;
    private int carbohydrate = 0;

    public NutritionFacts() { }

    // Setters
    public void setServingSize(int val)  { servingSize = val; }
    public void setServings(int val)     { servings = val; }
    public void setCalories(int val)     { calories = val; }
    public void setFat(int val)          { fat = val; }
    public void setSodium(int val)       { sodium = val; }
    public void setCarbohydrate(int val) { carbohydrate = val; }
}
```

**사용법과 문제점:**
```java
NutritionFacts cocaCola = new NutritionFacts();
cocaCola.setServingSize(240);
cocaCola.setServings(8);
cocaCola.setCalories(100);
cocaCola.setSodium(35);
cocaCola.setCarbohydrate(27);

// 문제: 객체 하나를 만들려면 메서드를 여러 개 호출해야 함
// 문제: 객체가 완전히 생성되기 전까지는 일관성이 무너진 상태
// 문제: 불변 클래스를 만들 수 없음
```

## 해결책: 빌더 패턴 (Builder Pattern)

빌더 패턴은 점층적 생성자 패턴의 안전성과 자바빈즈 패턴의 가독성을 겸비했다.

```java
public class NutritionFacts {
    private final int servingSize;
    private final int servings;
    private final int calories;
    private final int fat;
    private final int sodium;
    private final int carbohydrate;

    public static class Builder {
        // 필수 매개변수
        private final int servingSize;
        private final int servings;

        // 선택 매개변수 - 기본값으로 초기화
        private int calories = 0;
        private int fat = 0;
        private int sodium = 0;
        private int carbohydrate = 0;

        public Builder(int servingSize, int servings) {
            this.servingSize = servingSize;
            this.servings = servings;
        }

        public Builder calories(int val) {
            calories = val;
            return this;
        }

        public Builder fat(int val) {
            fat = val;
            return this;
        }

        public Builder sodium(int val) {
            sodium = val;
            return this;
        }

        public Builder carbohydrate(int val) {
            carbohydrate = val;
            return this;
        }

        public NutritionFacts build() {
            return new NutritionFacts(this);
        }
    }

    private NutritionFacts(Builder builder) {
        servingSize = builder.servingSize;
        servings = builder.servings;
        calories = builder.calories;
        fat = builder.fat;
        sodium = builder.sodium;
        carbohydrate = builder.carbohydrate;
    }
}
```

**사용법:**
```java
NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8)
        .calories(100)
        .sodium(35)
        .carbohydrate(27)
        .build();
```

## 계층적으로 설계된 클래스와 빌더 패턴

빌더 패턴은 계층적으로 설계된 클래스와 함께 쓰기에 좋다. 각 계층의 클래스에 관련 빌더를 멤버로 정의하자.

```java
public abstract class Pizza {
    public enum Topping { HAM, MUSHROOM, ONION, PEPPER, SAUSAGE }
    final Set<Topping> toppings;

    abstract static class Builder<T extends Builder<T>> {
        EnumSet<Topping> toppings = EnumSet.noneOf(Topping.class);

        public T addTopping(Topping topping) {
            toppings.add(Objects.requireNonNull(topping));
            return self();
        }

        abstract Pizza build();

        // 하위 클래스는 이 메서드를 재정의(override)하여
        // "this"를 반환하도록 해야 한다.
        protected abstract T self();
    }

    Pizza(Builder<?> builder) {
        toppings = builder.toppings.clone();
    }
}
```

**뉴욕 피자 구현:**
```java
public class NyPizza extends Pizza {
    public enum Size { SMALL, MEDIUM, LARGE }
    private final Size size;

    public static class Builder extends Pizza.Builder<Builder> {
        private final Size size;

        public Builder(Size size) {
            this.size = Objects.requireNonNull(size);
        }

        @Override public NyPizza build() {
            return new NyPizza(this);
        }

        @Override protected Builder self() { return this; }
    }

    private NyPizza(Builder builder) {
        super(builder);
        size = builder.size;
    }

    @Override public String toString() {
        return toppings + "로 토핑한 뉴욕 피자";
    }
}
```

**칼초네 피자 구현:**
```java
public class Calzone extends Pizza {
    private final boolean sauceInside;

    public static class Builder extends Pizza.Builder<Builder> {
        private boolean sauceInside = false; // 기본값

        public Builder sauceInside() {
            sauceInside = true;
            return this;
        }

        @Override public Calzone build() {
            return new Calzone(this);
        }

        @Override protected Builder self() { return this; }
    }

    private Calzone(Builder builder) {
        super(builder);
        sauceInside = builder.sauceInside;
    }

    @Override public String toString() {
        return String.format("%s로 토핑한 칼초네 피자 (소스는 %s에)",
                toppings, sauceInside ? "안" : "바깥");
    }
}
```

**사용 예시:**
```java
NyPizza pizza = new NyPizza.Builder(SMALL)
        .addTopping(SAUSAGE)
        .addTopping(ONION)
        .build();

Calzone calzone = new Calzone.Builder()
        .addTopping(HAM)
        .sauceInside()
        .build();
```

## 실무에서의 빌더 패턴 활용

### 1. HTTP 요청 빌더

```java
public class HttpRequest {
    private final String method;
    private final String url;
    private final Map<String, String> headers;
    private final String body;
    private final int timeout;

    public static class Builder {
        private final String method;
        private final String url;
        private Map<String, String> headers = new HashMap<>();
        private String body = "";
        private int timeout = 5000;

        public Builder(String method, String url) {
            this.method = Objects.requireNonNull(method);
            this.url = Objects.requireNonNull(url);
        }

        public Builder header(String key, String value) {
            headers.put(key, value);
            return this;
        }

        public Builder body(String body) {
            this.body = body;
            return this;
        }

        public Builder timeout(int timeout) {
            this.timeout = timeout;
            return this;
        }

        public HttpRequest build() {
            return new HttpRequest(this);
        }
    }

    private HttpRequest(Builder builder) {
        this.method = builder.method;
        this.url = builder.url;
        this.headers = new HashMap<>(builder.headers);
        this.body = builder.body;
        this.timeout = builder.timeout;
    }
}

// 사용 예시
HttpRequest request = new HttpRequest.Builder("POST", "/api/users")
        .header("Content-Type", "application/json")
        .header("Authorization", "Bearer token123")
        .body("{\"name\":\"John\", \"email\":\"john@example.com\"}")
        .timeout(10000)
        .build();
```

### 2. 데이터베이스 쿼리 빌더

```java
public class QueryBuilder {
    private final StringBuilder query = new StringBuilder();
    private final List<Object> parameters = new ArrayList<>();

    public static QueryBuilder select(String... columns) {
        QueryBuilder builder = new QueryBuilder();
        builder.query.append("SELECT ");
        if (columns.length == 0) {
            builder.query.append("*");
        } else {
            builder.query.append(String.join(", ", columns));
        }
        return builder;
    }

    public QueryBuilder from(String table) {
        query.append(" FROM ").append(table);
        return this;
    }

    public QueryBuilder where(String condition, Object... params) {
        query.append(" WHERE ").append(condition);
        parameters.addAll(Arrays.asList(params));
        return this;
    }

    public QueryBuilder orderBy(String column) {
        query.append(" ORDER BY ").append(column);
        return this;
    }

    public QueryBuilder limit(int count) {
        query.append(" LIMIT ").append(count);
        return this;
    }

    public String build() {
        return query.toString();
    }

    public List<Object> getParameters() {
        return new ArrayList<>(parameters);
    }
}

// 사용 예시
QueryBuilder queryBuilder = QueryBuilder
        .select("id", "name", "email")
        .from("users")
        .where("age > ? AND status = ?", 18, "ACTIVE")
        .orderBy("name")
        .limit(10);

String sql = queryBuilder.build();
// SELECT id, name, email FROM users WHERE age > ? AND status = ? ORDER BY name LIMIT 10
```

### 3. 설정 객체 빌더

```java
public class DatabaseConfig {
    private final String host;
    private final int port;
    private final String database;
    private final String username;
    private final String password;
    private final int maxConnections;
    private final int connectionTimeout;
    private final boolean useSSL;

    public static class Builder {
        // 필수 필드
        private final String host;
        private final String database;
        private final String username;
        private final String password;

        // 선택 필드 (기본값 설정)
        private int port = 5432;
        private int maxConnections = 10;
        private int connectionTimeout = 30000;
        private boolean useSSL = false;

        public Builder(String host, String database, String username, String password) {
            this.host = host;
            this.database = database;
            this.username = username;
            this.password = password;
        }

        public Builder port(int port) {
            if (port <= 0 || port > 65535) {
                throw new IllegalArgumentException("Invalid port: " + port);
            }
            this.port = port;
            return this;
        }

        public Builder maxConnections(int maxConnections) {
            if (maxConnections <= 0) {
                throw new IllegalArgumentException("Max connections must be positive");
            }
            this.maxConnections = maxConnections;
            return this;
        }

        public Builder connectionTimeout(int connectionTimeout) {
            if (connectionTimeout < 0) {
                throw new IllegalArgumentException("Connection timeout cannot be negative");
            }
            this.connectionTimeout = connectionTimeout;
            return this;
        }

        public Builder useSSL(boolean useSSL) {
            this.useSSL = useSSL;
            return this;
        }

        public DatabaseConfig build() {
            return new DatabaseConfig(this);
        }
    }

    private DatabaseConfig(Builder builder) {
        this.host = builder.host;
        this.port = builder.port;
        this.database = builder.database;
        this.username = builder.username;
        this.password = builder.password;
        this.maxConnections = builder.maxConnections;
        this.connectionTimeout = builder.connectionTimeout;
        this.useSSL = builder.useSSL;
    }
}

// 사용 예시
DatabaseConfig config = new DatabaseConfig.Builder("localhost", "mydb", "user", "pass")
        .port(5433)
        .maxConnections(20)
        .useSSL(true)
        .build();
```

## 빌더 패턴의 단점

1. **객체를 만들려면 빌더부터 만들어야 한다.** 빌더 생성 비용이 크지는 않지만 성능에 민감한 상황에서는 문제가 될 수 있다.

2. **코드가 장황해서 매개변수가 4개 이상은 되어야 값어치를 한다.** 하지만 API는 시간이 지날수록 매개변수가 많아지는 경향이 있다.

## 결론

**생성자나 정적 팩터리가 처리해야 할 매개변수가 많다면 빌더 패턴을 선택하는 게 더 낫다.** 매개변수 중 다수가 필수가 아니거나 같은 타입이면 특히 더 그렇다. 빌더는 점층적 생성자보다 클라이언트 코드를 읽고 쓰기가 훨씬 간결하고, 자바빈즈보다 훨씬 안전하다.