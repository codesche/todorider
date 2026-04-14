# 조건 필터링은 WAS에서 할까, DB에서 할까

개발하다 보면 이런 고민을 자주 하게 된다. 특정 조건에 맞는 데이터만 골라서 보여줄 때, 그 필터링을 어디서 처리할 건지 — DB에서 SQL로 걸러낼 건지, 아니면 전부 가져와서 WAS(애플리케이션 서버) 코드로 걸러낼 건지.

결론부터 말하면 **가능한 한 DB에서 처리하는 게 맞다**. 근데 무조건 그런 건 아니고, 상황에 따라 WAS에서 해야 할 때도 있다. 그 경계를 이해하는 게 중요하다.

---

## DB에서 필터링할 때의 장점

### 1. 네트워크 트래픽이 확 줄어든다

DB에서 `WHERE` 절로 1,000건만 골라오면 1,000건만 WAS로 넘어온다. 근데 WAS에서 필터링하면 일단 전체 100만 건을 다 가져와야 한다. 그 과정에서 DB → WAS 구간 네트워크가 터진다.

```sql
-- DB에서 처리: 필요한 것만 가져온다
SELECT * FROM orders
WHERE status = 'PENDING'
  AND created_at >= '2026-01-01';

-- WAS에서 처리하면: 전체 다 가져온 다음 Java/Node로 필터링
SELECT * FROM orders;  -- 이게 문제
```

### 2. 인덱스를 제대로 쓸 수 있다

DB에는 인덱스가 있다. `status`, `created_at` 컬럼에 인덱스가 걸려 있으면 DB는 Full Scan 없이 빠르게 찾는다. WAS에서 전체 데이터를 가져와 루프 돌리는 건 O(n)이지만, DB 인덱스는 B-Tree 기준 O(log n)이다.

```sql
-- 인덱스 활용 예시 (MySQL)
CREATE INDEX idx_orders_status_date ON orders(status, created_at);

-- 이 쿼리는 인덱스를 탄다
SELECT * FROM orders
WHERE status = 'PENDING'
  AND created_at >= '2026-01-01';
```

### 3. WAS 메모리 부하가 줄어든다

100만 건을 WAS 메모리에 올려놓고 필터링하면 GC 압박이 심해진다. 특히 Java 같은 경우 힙 메모리가 터질 수 있다. DB가 그 작업을 대신 해주면 WAS는 결과만 받아서 처리하면 된다.

### 4. DB 엔진 최적화를 그냥 공짜로 받는다

MySQL, PostgreSQL, Oracle 같은 DB는 수십 년간 쿼리 최적화를 해왔다. `EXPLAIN` 으로 실행계획 보면 알겠지만, DB는 조건에 따라 최적 접근 방식을 자동으로 선택한다. WAS에서 for-loop 돌리는 건 그냥 순진한 방식이다.

---

## WAS에서 필터링해야 하는 경우

무조건 DB가 좋은 건 아니다. 이런 경우엔 WAS에서 처리하는 게 맞다.

### 1. 비즈니스 로직이 복잡하고 DB로 표현이 안 될 때

SQL은 집합 연산에 특화돼 있다. 근데 조건이 복잡한 객체 그래프를 순회한다거나, 외부 API 응답 결과랑 조합해서 필터링해야 하는 경우엔 SQL로 쓰기가 어렵다.

```java
// 이런 건 WAS에서 하는 게 맞다
List<Order> orders = orderRepository.findAll();
return orders.stream()
    .filter(o -> pricingService.getDiscountedPrice(o) < threshold) // 외부 서비스 호출
    .collect(Collectors.toList());
```

### 2. DB에 부하를 주면 안 될 때

DB는 공유 자원이다. 무거운 연산을 DB에 계속 밀어 넣으면 다른 쿼리들이 영향을 받는다. 배치성 작업이나 통계 연산은 차라리 데이터를 가져와서 WAS에서 처리하거나, 별도 분석 DB(DW)를 쓰는 게 낫다.

### 3. 캐시된 데이터를 기반으로 필터링할 때

Redis 같은 인메모리 캐시에서 데이터를 가져와서 필터링하는 경우엔 당연히 WAS에서 해야 한다.

```java
// Redis 캐시에서 꺼내서 WAS에서 필터링
List<Product> cached = redisTemplate.opsForList().range("products", 0, -1);
return cached.stream()
    .filter(p -> p.getCategory().equals(targetCategory))
    .collect(Collectors.toList());
```

### 4. 권한/보안 관련 필터링이 레이어드 아키텍처 상 WAS에 있어야 할 때

특정 데이터를 볼 수 있는 사용자 권한이 복잡한 경우, Spring Security나 미들웨어 레이어에서 처리하는 게 아키텍처상 맞다. DB에 권한 로직을 심으면 유지보수가 힘들어진다.

---

## Oracle / MySQL / PostgreSQL에서 WAS와 연동하는 방식

세 DB 모두 WAS와 연동하는 기본 방식은 JDBC(Java Database Connectivity) 또는 각 언어의 드라이버를 통한 커넥션 풀이다.

### 공통 구조

```
WAS (Java/Node/Python)
  └─ Connection Pool (HikariCP / pgBouncer / 등)
       └─ DB Driver (JDBC / psycopg2 / cx_Oracle / mysql-connector)
            └─ DB Server (Oracle / MySQL / PostgreSQL)
```

WAS는 직접 DB에 소켓을 열지 않고, **커넥션 풀**을 통해 미리 만들어둔 연결을 재사용한다. 매 요청마다 새 연결을 열면 오버헤드가 크기 때문이다.

---

### Oracle 연동

Oracle은 JDBC Thin Driver 또는 OCI Driver를 쓴다. 보통 엔터프라이즈 환경에서는 JNDI나 Spring의 `DataSource`를 통해 연결한다.

```xml
<!-- Maven 의존성 -->
<dependency>
    <groupId>com.oracle.database.jdbc</groupId>
    <artifactId>ojdbc11</artifactId>
    <version>23.3.0.23.09</version>
</dependency>
```

```java
// HikariCP + Oracle 설정 (Spring Boot)
@Configuration
public class OracleDataSourceConfig {

    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:oracle:thin:@//localhost:1521/FREEPDB1");
        config.setUsername("myuser");
        config.setPassword("mypass");
        config.setMaximumPoolSize(10);
        config.setConnectionTimeout(30000);
        return new HikariDataSource(config);
    }
}
```

Oracle 특유의 문법 몇 가지:

```sql
-- 페이징 (Oracle 12c 이상)
SELECT * FROM orders
WHERE status = 'PENDING'
ORDER BY created_at DESC
OFFSET 0 ROWS FETCH NEXT 20 ROWS ONLY;

-- 12c 미만은 ROWNUM 사용
SELECT * FROM (
    SELECT a.*, ROWNUM AS rn FROM (
        SELECT * FROM orders
        WHERE status = 'PENDING'
        ORDER BY created_at DESC
    ) a WHERE ROWNUM <= 20
) WHERE rn > 0;
```

---

### MySQL 연동

MySQL은 `mysql-connector-j` (구 `mysql-connector-java`) 를 쓴다.

```xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>8.3.0</version>
</dependency>
```

```yaml
# application.yml (Spring Boot)
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb?useSSL=false&characterEncoding=UTF-8&serverTimezone=Asia/Seoul
    username: myuser
    password: mypass
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      maximum-pool-size: 10
      connection-timeout: 30000
```

MySQL에서 필터링 + 페이징:

```sql
-- 필터링 + LIMIT/OFFSET 페이징
SELECT order_id, user_id, status, created_at
FROM orders
WHERE status = 'PENDING'
  AND created_at >= '2026-01-01'
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;

-- 복합 인덱스로 커버링 인덱스 활용
CREATE INDEX idx_cover ON orders(status, created_at, order_id, user_id);
```

---

### PostgreSQL 연동

PostgreSQL은 `postgresql` JDBC 드라이버를 쓴다.

```xml
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <version>42.7.2</version>
</dependency>
```

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: myuser
    password: mypass
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 10
```

PostgreSQL은 필터링 관련해서 강력한 기능이 많다. 특히 `GIN`, `GiST` 인덱스로 JSON이나 배열 필터링도 가능하다.

```sql
-- JSONB 컬럼 필터링
SELECT * FROM products
WHERE metadata @> '{"category": "electronics"}';

-- GIN 인덱스로 빠르게
CREATE INDEX idx_products_meta ON products USING GIN(metadata);

-- 배열 필터링
SELECT * FROM users
WHERE 'admin' = ANY(roles);

-- partial index로 조건부 인덱싱
CREATE INDEX idx_pending_orders ON orders(created_at)
WHERE status = 'PENDING';  -- PENDING인 것만 인덱싱
```

---

## 정리

단순하게 정리하면 이렇다.

| 조건 | 어디서? |
|------|--------|
| 컬럼 값 기반 단순 필터링 | DB (WHERE 절) |
| 범위 조건, 정렬, 페이징 | DB |
| 외부 API 결과와 조합 | WAS |
| 캐시된 데이터 필터링 | WAS |
| 복잡한 권한/비즈니스 로직 | WAS |
| JSONB, 배열, 전문검색 | DB (PostgreSQL 강점) |

기본 원칙은 **데이터를 이동시키지 말고, 연산을 데이터가 있는 곳으로 보내라**는 거다. 네트워크는 비싸고, DB 인덱스는 공짜가 아니지만 이미 있는 거다. 잘 쓰면 그만큼 이득이다.
