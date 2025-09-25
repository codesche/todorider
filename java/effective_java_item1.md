# 이펙티브 자바 Item 1: 생성자 대신 정적 팩터리 메서드를 고려하라

## 들어가며

클래스의 인스턴스를 얻는 전통적인 방법은 public 생성자를 사용하는 것입니다. 하지만 모든 프로그래머가 알아둬야 할 또 다른 기법이 있습니다. 바로 `정적 팩터리 메서드(static factory method)`이다.

## 정적 팩터리 메서드란?

정적 팩터리 메서드는 클래스의 인스턴스를 반환하는 단순한 정적 메서드입니다. 다음은 boolean 기본 타입의 박싱 클래스인 Boolean의 간단한 예제 코드다.

```java
public static Boolean valueOf(boolean b) {
    return b ? Boolean.TRUE : Boolean.FALSE;
}
```

## 정적 팩터리 메서드의 5가지 장점

### 1. 이름을 가질 수 있다

생성자는 클래스 이름과 동일해야 하지만, 정적 팩터리 메서드는 의미 있는 이름을 가질 수 있다.

```java
// 생성자 - 어떤 소수인지 명확하지 않음
BigInteger prime1 = new BigInteger(int bitLength, int certainty, Random rnd);

// 정적 팩터리 메서드 - 의도가 명확함
BigInteger prime2 = BigInteger.probablePrime(int bitLength, Random rnd);
```

**실무 예시: 사용자 객체 생성**

```java
public class User {
    private String email;
    private String name;
    private UserType type;
    
    private User(String email, String name, UserType type) {
        this.email = email;
        this.name = name;
        this.type = type;
    }
    
    // 일반 사용자 생성
    public static User createRegularUser(String email, String name) {
        return new User(email, name, UserType.REGULAR);
    }
    
    // 관리자 생성
    public static User createAdmin(String email, String name) {
        return new User(email, name, UserType.ADMIN);
    }
    
    // 게스트 사용자 생성
    public static User createGuest() {
        return new User("guest@example.com", "Guest", UserType.GUEST);
    }
}
```

### 2. 호출될 때마다 인스턴스를 새로 생성하지 않아도 된다

불변 클래스는 인스턴스를 미리 만들어 놓거나 새로 생성한 인스턴스를 캐싱하여 재활용할 수 있다.

```java
public final class Boolean implements Serializable {
    public static final Boolean TRUE = new Boolean(true);
    public static final Boolean FALSE = new Boolean(false);
    
    public static Boolean valueOf(boolean b) {
        return (b ? TRUE : FALSE); // 인스턴스 재사용
    }
}
```

**실무 예시: 연결 풀 관리**

```java
public class DatabaseConnection {
    private static final Map<String, DatabaseConnection> connectionPool = new HashMap<>();
    private static final int MAX_CONNECTIONS = 10;
    
    private String url;
    private boolean inUse;
    
    private DatabaseConnection(String url) {
        this.url = url;
        this.inUse = false;
    }
    
    public static DatabaseConnection getConnection(String url) {
        DatabaseConnection connection = connectionPool.get(url);
        
        if (connection == null && connectionPool.size() < MAX_CONNECTIONS) {
            connection = new DatabaseConnection(url);
            connectionPool.put(url, connection);
        }
        
        if (connection != null && !connection.inUse) {
            connection.inUse = true;
            return connection;
        }
        
        throw new RuntimeException("No available connections");
    }
}
```

### 3. 반환 타입의 하위 타입 객체를 반환할 수 있다

반환할 객체의 클래스를 자유롭게 선택할 수 있는 유연성을 제공한다.

```java
// 인터페이스 기반 프레임워크의 예
public interface PaymentProcessor {
    void processPayment(double amount);
    
    // 정적 팩터리 메서드
    static PaymentProcessor create(String type) {
        switch (type.toLowerCase()) {
            case "credit":
                return new CreditCardProcessor();
            case "paypal":
                return new PayPalProcessor();
            case "bank":
                return new BankTransferProcessor();
            default:
                throw new IllegalArgumentException("Unknown payment type: " + type);
        }
    }
}

class CreditCardProcessor implements PaymentProcessor {
    @Override
    public void processPayment(double amount) {
        System.out.println("Processing credit card payment: $" + amount);
    }
}
```

### 4. 입력 매개변수에 따라 매번 다른 클래스의 객체를 반환할 수 있다

반환 타입의 하위 타입이기만 하면 어떤 클래스의 객체를 반환하든 상관없다.

```java
public abstract class Animal {
    public abstract void makeSound();
    
    public static Animal createAnimal(String type, String name) {
        switch (type.toLowerCase()) {
            case "dog":
                return new Dog(name);
            case "cat":
                return new Cat(name);
            case "bird":
                return new Bird(name);
            default:
                return new UnknownAnimal(name);
        }
    }
}

class Dog extends Animal {
    private String name;
    
    public Dog(String name) {
        this.name = name;
    }
    
    @Override
    public void makeSound() {
        System.out.println(name + " says: Woof!");
    }
}
```

### 5. 정적 팩터리 메서드를 작성하는 시점에는 반환할 객체의 클래스가 존재하지 않아도 된다

이는 서비스 제공자 프레임워크(Service Provider Framework)의 근간이 된다.

```java
// JDBC의 예시와 유사한 패턴
public interface DatabaseDriver {
    Connection connect(String url);
}

public class DriverManager {
    private static final Map<String, DatabaseDriver> drivers = new HashMap<>();
    
    // 드라이버 등록 (런타임에 동적으로 추가 가능)
    public static void registerDriver(String name, DatabaseDriver driver) {
        drivers.put(name, driver);
    }
    
    // 정적 팩터리 메서드 - 실제 구현체는 런타임에 결정
    public static Connection getConnection(String driverName, String url) {
        DatabaseDriver driver = drivers.get(driverName);
        if (driver == null) {
            throw new IllegalArgumentException("Unknown driver: " + driverName);
        }
        return driver.connect(url);
    }
}
```

## 정적 팩터리 메서드의 2가지 단점

### 1. 상속을 하려면 public이나 protected 생성자가 필요하다

정적 팩터리 메서드만 제공하면 하위 클래스를 만들 수 없다.

```java
public final class Collections {
    private Collections() {} // 인스턴스화 방지
    
    // 정적 팩터리 메서드들만 제공
    public static <T> List<T> emptyList() {
        return (List<T>) EMPTY_LIST;
    }
}

// Collections는 final 클래스이면서 private 생성자만 있어 상속 불가
// class MyCollections extends Collections {} // 컴파일 에러
```

### 2. 정적 팩터리 메서드는 프로그래머가 찾기 어렵다

생성자처럼 API 설명에 명확히 드러나지 않아 사용자가 정적 팩터리 메서드 방식 클래스를 인스턴스화할 방법을 찾아야 한다.

## 정적 팩터리 메서드 명명 규칙

- **from**: 매개변수를 하나 받아서 해당 타입의 인스턴스를 반환
  ```java
  Date d = Date.from(instant);
  ```

- **of**: 여러 매개변수를 받아 적합한 타입의 인스턴스를 반환
  ```java
  Set<Rank> faceCards = EnumSet.of(JACK, QUEEN, KING);
  ```

- **valueOf**: from과 of의 더 자세한 버전
  ```java
  BigInteger prime = BigInteger.valueOf(Integer.MAX_VALUE);
  ```

- **instance** 혹은 **getInstance**: 매개변수로 명시한 인스턴스를 반환
  ```java
  StackWalker luke = StackWalker.getInstance(options);
  ```

- **create** 혹은 **newInstance**: 매번 새로운 인스턴스를 생성해 반환
  ```java
  Object newArray = Array.newInstance(classObject, arrayLen);
  ```

## 실무 적용 예시

```java
@Entity
public class Order {
    @Id
    private Long id;
    private String customerEmail;
    private List<OrderItem> items;
    private OrderStatus status;
    private LocalDateTime createdAt;
    
    // 기본 생성자 (JPA 용)
    protected Order() {}
    
    private Order(String customerEmail, List<OrderItem> items, OrderStatus status) {
        this.customerEmail = customerEmail;
        this.items = new ArrayList<>(items);
        this.status = status;
        this.createdAt = LocalDateTime.now();
    }
    
    // 일반 주문 생성
    public static Order createOrder(String customerEmail, List<OrderItem> items) {
        validateItems(items);
        return new Order(customerEmail, items, OrderStatus.PENDING);
    }
    
    // 긴급 주문 생성
    public static Order createUrgentOrder(String customerEmail, List<OrderItem> items) {
        validateItems(items);
        Order order = new Order(customerEmail, items, OrderStatus.URGENT);
        // 긴급 주문 특별 처리 로직
        return order;
    }
    
    // 테스트용 주문 생성
    public static Order createTestOrder() {
        return new Order("test@example.com", 
                        Arrays.asList(OrderItem.createTestItem()), 
                        OrderStatus.TEST);
    }
    
    private static void validateItems(List<OrderItem> items) {
        if (items == null || items.isEmpty()) {
            throw new IllegalArgumentException("주문 항목이 비어있습니다.");
        }
    }
}
```

## 결론

정적 팩터리 메서드와 public 생성자는 각각의 쓰임새가 있으니 상대적인 장단점을 이해하고 사용하는 것이 좋다. 그렇다고 하더라도 정적 팩터리를 사용하는 게 유리한 경우가 더 많으므로 **무작정 public 생성자를 제공하던 습관이 있다면 고치는 것이 좋다**.

정적 팩터리 메서드는 객체 지향 설계에서 유연성과 가독성을 크게 향상시키는 도구이다. 특히 복잡한 객체 생성 로직이나 다양한 생성 시나리오가 있는 경우에 그 진가를 발휘한다.