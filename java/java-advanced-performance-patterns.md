# Java 고급 문법과 실무 성능 설계 패턴

실무에서 Java 성능 문제는 대부분 잘못된 자료구조 선택, 불필요한 객체 생성, 동시성 설계 미흡에서 비롯됨. 고급 문법을 정확히 이해하고 성능을 고려한 설계 습관을 갖추면 코드 품질과 처리량이 동시에 올라감.

---

## 1. Stream API — 성능을 고려한 올바른 사용법

Stream은 편리하지만 잘못 쓰면 오히려 for-loop보다 느려짐.

### 1-1. 기본형 스트림으로 박싱 비용 제거

```java
// 나쁜 예: Integer 박싱/언박싱 반복 발생
List<Integer> numbers = List.of(1, 2, 3, 4, 5);
int sum = numbers.stream()
    .mapToInt(n -> n)      // 불필요한 언박싱
    .sum();

// 좋은 예: IntStream 직접 사용
int sum = IntStream.rangeClosed(1, 5).sum();

// 대용량 데이터에서 차이가 두드러짐
long count = IntStream.range(0, 1_000_000)
    .filter(n -> n % 2 == 0)
    .count();
```

### 1-2. 병렬 스트림 — 무조건 빠르지 않음

```java
// 병렬 스트림이 유리한 케이스: CPU-bound, 대용량, 독립적 연산
List<String> results = largeList.parallelStream()
    .filter(s -> expensivePredicate(s))
    .map(s -> heavyTransform(s))
    .collect(Collectors.toList());

// 병렬 스트림이 불리한 케이스: 공유 상태, 소량 데이터, I/O-bound
// 공유 상태 변경 → race condition 발생
List<Integer> shared = new ArrayList<>();
IntStream.range(0, 1000).parallel().forEach(shared::add); // 위험!

// 올바른 병렬 집계: thread-safe한 collect 사용
List<Integer> safe = IntStream.range(0, 1000)
    .parallel()
    .boxed()
    .collect(Collectors.toList()); // 내부적으로 동기화 처리됨
```

### 1-3. Stream 중간 연산 순서 최적화

```java
List<String> names = employees.stream()
    // 나쁜 순서: map 먼저 → 모든 요소를 변환 후 filter
    .map(Employee::getFullName)
    .filter(name -> name.startsWith("김"))
    .collect(Collectors.toList());

List<String> names = employees.stream()
    // 좋은 순서: filter 먼저 → 대상을 줄인 뒤 map
    .filter(e -> e.getLastName().equals("김"))
    .map(Employee::getFullName)
    .collect(Collectors.toList());

// limit()와 조합하면 더 큰 효과
Optional<String> first = employees.stream()
    .filter(e -> e.getDeptId() == 10)
    .map(Employee::getName)
    .findFirst(); // 첫 번째 발견 즉시 단락(short-circuit) 처리
```

### 1-4. Collectors 고급 활용

```java
// groupingBy: 부서별 직원 그룹화
Map<Integer, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDeptId));

// groupingBy + downstream 집계
Map<Integer, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDeptId,
        Collectors.counting()
    ));

// 부서별 평균 급여
Map<Integer, Double> avgSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDeptId,
        Collectors.averagingInt(Employee::getSalary)
    ));

// partitioningBy: 조건 기준 이분 분류
Map<Boolean, List<Employee>> partitioned = employees.stream()
    .collect(Collectors.partitioningBy(
        e -> e.getSalary() >= 5000
    ));
// partitioned.get(true)  → 연봉 5000 이상
// partitioned.get(false) → 연봉 5000 미만

// toUnmodifiableMap: 불변 맵으로 수집
Map<Integer, String> idToName = employees.stream()
    .collect(Collectors.toUnmodifiableMap(
        Employee::getId,
        Employee::getName
    ));
```

---

## 2. 불변 객체(Immutable Object) & Value Object 설계

불변 객체는 thread-safe하고 캐싱이 쉬우며 예측 가능한 동작을 보장함. 실무에서 DTO, 도메인 값 객체에 적극 활용.

### 2-1. 불변 클래스 설계 원칙

```java
// 불변 객체 설계: final 클래스 + final 필드 + 방어적 복사
public final class Money {
    private final int amount;
    private final String currency;

    public Money(int amount, String currency) {
        if (amount < 0) throw new IllegalArgumentException("amount must be >= 0");
        if (currency == null || currency.isBlank()) throw new IllegalArgumentException("currency required");
        this.amount = amount;
        this.currency = currency;
    }

    // 상태를 바꾸는 대신 새 객체 반환
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("currency mismatch");
        }
        return new Money(this.amount + other.amount, this.currency);
    }

    public int getAmount()    { return amount; }
    public String getCurrency() { return currency; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Money m)) return false;
        return amount == m.amount && currency.equals(m.currency);
    }

    @Override
    public int hashCode() {
        return Objects.hash(amount, currency);
    }
}

// 사용 예
Money price  = new Money(10000, "KRW");
Money tax    = new Money(1000,  "KRW");
Money total  = price.add(tax); // 새 객체 반환, 원본 불변
```

### 2-2. Java 16+ Record로 간결한 불변 객체

```java
// Record: equals, hashCode, toString, 접근자 자동 생성
public record Point(double x, double y) {
    // 컴팩트 생성자로 검증 추가
    public Point {
        if (Double.isNaN(x) || Double.isNaN(y)) {
            throw new IllegalArgumentException("좌표에 NaN 불가");
        }
    }

    // 커스텀 메서드 추가 가능
    public double distanceTo(Point other) {
        double dx = this.x - other.x;
        double dy = this.y - other.y;
        return Math.sqrt(dx * dx + dy * dy);
    }
}

// DTO에 Record 활용 (Spring에서 요청/응답 객체)
public record OrderRequest(
    Long   productId,
    int    quantity,
    String address
) {}

public record OrderResponse(
    Long   orderId,
    String status,
    int    totalAmount
) {}
```

### 2-3. 방어적 복사로 가변 참조 차단

```java
// 실무에서는 Date 대신 LocalDate/LocalDateTime 사용 권장 (자체로 불변)
public final class Period {
    private final LocalDate start;
    private final LocalDate end;

    public Period(LocalDate start, LocalDate end) {
        Objects.requireNonNull(start, "start required");
        Objects.requireNonNull(end,   "end required");
        if (start.isAfter(end)) throw new IllegalArgumentException("start must be before end");
        this.start = start;
        this.end   = end;
    }

    public LocalDate getStart() { return start; } // LocalDate는 불변 → 방어적 복사 불필요
    public LocalDate getEnd()   { return end; }
}
```

---

## 3. 컬렉션 선택 전략 — 자료구조별 성능 특성

### 3-1. List 계열

```java
// ArrayList: 랜덤 접근 O(1), 중간 삽입/삭제 O(n), 일반적인 기본 선택
List<String> list = new ArrayList<>();
// 초기 용량 지정으로 resize 비용 제거 (크기를 미리 알 때)
List<String> list = new ArrayList<>(10_000);

// LinkedList: 양 끝 삽입/삭제 O(1), 랜덤 접근 O(n)
// Queue/Deque 용도로만 사용할 것
Deque<String> deque = new LinkedList<>();

// 불변 리스트: List.of() — 수정 불가, null 불가
List<String> immutable = List.of("a", "b", "c");

// 진짜 불변 복사본: List.copyOf()
List<String> truly = List.copyOf(source);
```

### 3-2. Map 계열

```java
// HashMap: 일반적인 키-값 저장, 평균 O(1)
// 초기 용량 설정: (예상 개수 / 0.75) + 1 로 resize 방지
int expectedSize = 100;
Map<String, Integer> map = new HashMap<>((int)(expectedSize / 0.75) + 1);

// LinkedHashMap: 삽입 순서 유지 (HashMap과 동일한 성능)
Map<String, Integer> ordered = new LinkedHashMap<>();

// TreeMap: 키 정렬 유지, O(log n)
Map<String, Integer> sorted = new TreeMap<>();

// computeIfAbsent: 키 없을 때만 계산 (람다는 필요할 때만 실행)
Map<String, List<String>> multiMap = new HashMap<>();
multiMap.computeIfAbsent("group1", k -> new ArrayList<>()).add("item");

// merge: 기존 값이 있으면 병합, 없으면 신규 삽입
wordCountMap.merge(word, 1, Integer::sum); // 단어 빈도 카운터
```

### 3-3. Set 계열 & 특수 컬렉션

```java
// HashSet: 중복 제거, 포함 여부 확인 O(1)
Set<Long> processedIds = new HashSet<>();

// EnumMap / EnumSet: Enum 키 전용, 배열 기반으로 HashMap보다 빠름
enum Status { PENDING, IN_PROGRESS, DONE, FAILED }

EnumMap<Status, List<Task>> taskByStatus = new EnumMap<>(Status.class);
EnumSet<Status> activeStatuses = EnumSet.of(Status.PENDING, Status.IN_PROGRESS);

// ArrayDeque: Stack/Queue 대용, LinkedList보다 빠름
Deque<String> stack = new ArrayDeque<>();
stack.push("first");
stack.push("second");
String top = stack.pop(); // "second"
```

---

## 4. 동시성 처리 — synchronized vs Lock vs Atomic

### 4-1. synchronized의 한계와 ReentrantLock

```java
// ReentrantLock: tryLock으로 데드락 방지, fairness 옵션
public class SafeCounter {
    private int count = 0;
    private final ReentrantLock lock = new ReentrantLock();

    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock(); // 반드시 finally에서 해제
        }
    }

    // 타임아웃 시도: 데드락 방지
    public boolean tryIncrement() throws InterruptedException {
        if (lock.tryLock(100, TimeUnit.MILLISECONDS)) {
            try {
                count++;
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false;
    }
}

// ReadWriteLock: 읽기는 동시, 쓰기만 독점
public class CachedData {
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private Map<String, String> cache = new HashMap<>();

    public String get(String key) {
        rwLock.readLock().lock();
        try {
            return cache.get(key);
        } finally {
            rwLock.readLock().unlock();
        }
    }

    public void put(String key, String value) {
        rwLock.writeLock().lock();
        try {
            cache.put(key, value);
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}
```

### 4-2. Atomic 클래스 — lock-free 카운터

```java
// AtomicInteger: CAS(Compare-And-Swap) 기반, lock 없이 thread-safe
public class RequestCounter {
    private final AtomicInteger count = new AtomicInteger(0);
    private final AtomicLong    totalBytes = new AtomicLong(0);

    public void record(int bytes) {
        count.incrementAndGet();
        totalBytes.addAndGet(bytes);
    }
}

// LongAdder: 고경쟁 환경에서 AtomicLong보다 빠른 카운터
public class HighThroughputCounter {
    private final LongAdder adder = new LongAdder();

    public void increment() { adder.increment(); }
    public long sum()       { return adder.sum(); } // 집계 시점에만 합산
}
```

### 4-3. 실무 패턴 — ConcurrentHashMap 활용

```java
// ConcurrentHashMap: 세그먼트 단위 락, 높은 동시성
Map<String, Integer> concurrentMap = new ConcurrentHashMap<>();

// compute: 원자적 읽기-수정-쓰기
concurrentMap.compute("key", (k, v) -> v == null ? 1 : v + 1);

// merge: 단어 빈도 카운터
concurrentMap.merge("word", 1, Integer::sum);

// computeIfAbsent: 값 계산이 비싼 경우 (캐시 패턴)
ConcurrentHashMap<Long, UserProfile> profileCache = new ConcurrentHashMap<>();
UserProfile profile = profileCache.computeIfAbsent(
    userId,
    id -> userRepository.findById(id) // 키 없을 때만 DB 조회
);
```

---

## 5. 지연 초기화 & 캐싱 패턴

### 5-1. 홀더 클래스 패턴 (권장 Singleton)

```java
// 홀더 클래스 패턴: 초기화 시점 보장 + lock 없음
public class HeavyService {
    private HeavyService() {}

    private static class Holder {
        static final HeavyService INSTANCE = new HeavyService();
    }

    public static HeavyService getInstance() {
        return Holder.INSTANCE; // 클래스 로딩 시 JVM이 thread-safe 보장
    }
}
```

### 5-2. TTL 캐시 구현

```java
public class SimpleCache<K, V> {
    private final Map<K, CacheEntry<V>> store = new ConcurrentHashMap<>();
    private final long ttlMillis;

    public SimpleCache(long ttlMillis) {
        this.ttlMillis = ttlMillis;
    }

    public void put(K key, V value) {
        store.put(key, new CacheEntry<>(value, System.currentTimeMillis()));
    }

    public Optional<V> get(K key) {
        CacheEntry<V> entry = store.get(key);
        if (entry == null) return Optional.empty();
        if (System.currentTimeMillis() - entry.createdAt > ttlMillis) {
            store.remove(key);
            return Optional.empty();
        }
        return Optional.of(entry.value);
    }

    private record CacheEntry<V>(V value, long createdAt) {}
}

// 사용 예
SimpleCache<Long, UserProfile> cache = new SimpleCache<>(5 * 60 * 1000L); // 5분 TTL

UserProfile profile = cache.get(userId).orElseGet(() -> {
    UserProfile fresh = userRepository.findById(userId);
    cache.put(userId, fresh);
    return fresh;
});
```

---

## 6. 메모리 효율적인 객체 설계

### 6-1. String 최적화

```java
// 나쁜 예: 루프 내 String 연결 → O(n²) 메모리 할당
String result = "";
for (String item : list) {
    result += item + ", "; // 매 반복마다 새 String 객체 생성
}

// 좋은 예: String.join() 또는 Collectors.joining()
String result = String.join(", ", list);
String result = list.stream().collect(Collectors.joining(", "));
```

### 6-2. 원시 타입 vs 래퍼 타입

```java
// 나쁜 예: 불필요한 래퍼 타입 사용
Long sum = 0L;
for (long value : values) {
    sum += value; // Long ↔ long 박싱/언박싱 반복
}

// 좋은 예: 원시 타입 사용
long sum = 0L;
for (long value : values) {
    sum += value;
}

// null 체크 없이 언박싱 → NPE 위험
int total = map.getOrDefault("key", 0) + 1; // 안전하게 처리
```

### 6-3. Optional 올바른 활용

```java
// orElse vs orElseGet 차이
User user1 = findById(id).orElse(User.guest());         // 항상 User.guest() 실행
User user2 = findById(id).orElseGet(() -> User.guest()); // 필요할 때만 실행

// DB 조회 같은 비용이 큰 기본값은 반드시 orElseGet 사용
User user3 = findById(id).orElseGet(() -> userRepository.createGuest());

// 함수형으로 처리 (isPresent + get 패턴 지양)
findById(id)
    .map(User::getProfile)
    .filter(p -> p.isActive())
    .ifPresent(profile -> sendNotification(profile));
```

---

## 7. 실무 디자인 패턴

### 7-1. Builder 패턴 — 복잡한 객체 생성

```java
public class HttpRequest {
    private final String method;
    private final String url;
    private final Map<String, String> headers;
    private final String body;
    private final int timeoutMs;

    private HttpRequest(Builder builder) {
        this.method    = builder.method;
        this.url       = builder.url;
        this.headers   = Map.copyOf(builder.headers);
        this.body      = builder.body;
        this.timeoutMs = builder.timeoutMs;
    }

    public static class Builder {
        private String method = "GET";
        private String url;
        private Map<String, String> headers = new HashMap<>();
        private String body;
        private int timeoutMs = 3000;

        public Builder url(String url)           { this.url = Objects.requireNonNull(url); return this; }
        public Builder method(String method)     { this.method = method;                   return this; }
        public Builder header(String k, String v){ this.headers.put(k, v);                return this; }
        public Builder body(String body)         { this.body = body;                       return this; }
        public Builder timeoutMs(int ms)         { this.timeoutMs = ms;                    return this; }

        public HttpRequest build() {
            if (url == null) throw new IllegalStateException("url required");
            return new HttpRequest(this);
        }
    }
}

// 사용 예
HttpRequest request = new HttpRequest.Builder()
    .url("https://api.example.com/users")
    .method("POST")
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer " + token)
    .body(requestBody)
    .timeoutMs(5000)
    .build();
```

### 7-2. Strategy 패턴 — 알고리즘 교체 가능하게 설계

```java
@FunctionalInterface
public interface DiscountStrategy {
    int apply(int originalPrice);
}

public class DiscountStrategies {
    public static final DiscountStrategy NONE      = price -> price;
    public static final DiscountStrategy TEN_PCT   = price -> (int)(price * 0.9);
    public static final DiscountStrategy FLAT_1000 = price -> Math.max(0, price - 1000);

    public static DiscountStrategy combined(double rate, int flat) {
        return price -> Math.max(0, (int)(price * (1 - rate)) - flat);
    }
}

// 회원 등급별 전략 매핑
Map<MemberGrade, DiscountStrategy> strategyMap = new EnumMap<>(MemberGrade.class);
strategyMap.put(MemberGrade.NORMAL, DiscountStrategies.NONE);
strategyMap.put(MemberGrade.SILVER, DiscountStrategies.TEN_PCT);
strategyMap.put(MemberGrade.GOLD,   DiscountStrategies.combined(0.2, 500));

DiscountStrategy strategy = strategyMap.getOrDefault(member.getGrade(), DiscountStrategies.NONE);
int finalPrice = orderService.calculatePrice(basePrice, strategy);
```

### 7-3. Template Method 패턴 — 공통 흐름 추상화

```java
public abstract class ReportGenerator {

    // 템플릿 메서드: 흐름 고정
    public final Report generate(LocalDate from, LocalDate to) {
        List<RawData> raw      = fetchData(from, to);
        List<RawData> filtered = filter(raw);
        List<Row>     rows     = transform(filtered);
        return buildReport(rows);
    }

    protected abstract List<RawData> fetchData(LocalDate from, LocalDate to);
    protected abstract List<Row>     transform(List<RawData> data);

    // 훅 메서드: 기본 구현 제공, 필요 시 오버라이드
    protected List<RawData> filter(List<RawData> data) { return data; }

    private Report buildReport(List<Row> rows) {
        return new Report(rows, LocalDateTime.now());
    }
}
```

### 7-4. 함수형 파이프라인 패턴

```java
public class OrderProcessor {

    public ProcessResult process(OrderRequest request) {
        return validate(request)
            .flatMap(this::checkInventory)
            .flatMap(this::calculatePrice)
            .flatMap(this::saveOrder)
            .map(order -> ProcessResult.success(order.getId()))
            .orElse(ProcessResult.failure("처리 실패"));
    }

    private Optional<OrderRequest> validate(OrderRequest req) {
        if (req.productId() == null || req.quantity() <= 0) return Optional.empty();
        return Optional.of(req);
    }

    private Optional<OrderRequest> checkInventory(OrderRequest req) {
        boolean available = inventoryService.isAvailable(req.productId(), req.quantity());
        return available ? Optional.of(req) : Optional.empty();
    }

    private Optional<PricedOrder> calculatePrice(OrderRequest req) {
        int price = pricingService.calculate(req.productId(), req.quantity());
        return Optional.of(new PricedOrder(req, price));
    }

    private Optional<Order> saveOrder(PricedOrder pricedOrder) {
        return Optional.of(orderRepository.save(pricedOrder));
    }
}
```

---

## 요약

| 주제 | 핵심 원칙 |
|---|---|
| Stream API | filter 먼저, 기본형 스트림 우선, 병렬은 CPU-bound + 대용량 한정 |
| 불변 객체 | final 클래스 + final 필드 + 방어적 복사, Record 적극 활용 |
| 컬렉션 선택 | 접근 패턴 먼저 파악, Enum 키는 EnumMap, 스택/큐는 ArrayDeque |
| 동시성 | 카운터 → Atomic/LongAdder, 캐시 → ConcurrentHashMap, 복잡 제어 → Lock |
| 캐싱 | 홀더 클래스 Singleton, computeIfAbsent로 lazy 초기화 |
| 메모리 | String 연결은 join/joining, 원시 타입 우선, orElseGet으로 지연 평가 |
| 디자인 패턴 | Builder로 가독성, Strategy로 OCP, Template Method로 공통 흐름 분리 |

성능 설계는 병목이 어디 있는지 먼저 측정하고, 측정된 근거를 바탕으로 순차적으로 개선을 적용하는 게 핵심임.
