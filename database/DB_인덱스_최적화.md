# DungeonTalk 프로젝트의 DB 인덱스 최적화

## 인덱스 최적화의 시작

데이터베이스는 인덱스는 쿼리 성능을 좌우하는 핵심 요소다. 
적절한 인덱스 없이는 테이블 전체를 스캔하는 Full Table Scan이 발생하여 응답 시간이 급격히 느려진다.
DungeonTalk 프로젝트를 진행하면성 PostgreSQL과 MongoDB 두 가지 데이터베이스를
인덱스에 적용하고 성능을 개선한 경험을 공유하고자 한다.

### DungeonTalk 프로젝트 링크

[https://github.com/DungeonTalk/dungeontalk-backend?tab=readme-ov-file](https://github.com/DungeonTalk/dungeontalk-backend?tab=readme-ov-file)

## 프로젝트 배경

DungeonTalk은 AI와 함께하는 실시간 채팅 기반 멀티플레이 게임 플랫폼이다. 

데이터베이스를 기준으로 간략하게 주요 기술스택을 정리하자면 다음과 같다.

- PostgreSQL: 사용자 정보, 게임 캐릭터, 세계관 등 구조화된 데이터 관리
- MongoDB: 채팅 메시지와 같이 빈번하게 생성되는 데이터 관리
- Spring Boot: 백엔드 애플리케이션 프레임워크

두 가지 데이터베이스를 함께 사용하면서 각 DB의 특성에 맞는 인덱스 전략을 수립해야 했다.

## 문제 발견: WorldType(세계관) 조회 성능 이슈

### 이슈

서버 시작 시 동일한 `world_types` 테이블 조회 쿼리가 3번 반복 실행되는 현상을 발견했다.

```sql
SELECT wt1_0.id, wt1_0.code, wt1_0.created_at, ...
FROM world_types wt1_0
WHERE wt1_0.is_active=true
ORDER BY wt1_0.sort_order, wt1_0.id
```

### 원인 분석

문제의 근본적인 원인은 두 가지였다.

1. **Thread Safety 이슈:** 수동 캐싱 로직이 **thread-safe**하지 않아 여러 스레드가 동시에 DB를 조회했다.
2. **캐시 만료:** 30초 TTL 설정으로 인해 주기적으로 DB 조회가 발생했다.

```java
// 문제 코드
public List<WorldType> getAllActiveWorldTypes() {
    long currentTime = System.currentTimeMillis();

    // 여러 스레드가 동시에 이 체크를 통과할 수 있음!
    if (cachedActiveWorldTypes == null ||
        (currentTime - lastCacheUpdate) > CACHE_DURATION_MS) {
        cachedActiveWorldTypes = worldTypeService.findAllActiveWorldTypeEntities();
        lastCacheUpdate = currentTime;
    }

    return cachedActiveWorldTypes;
}
```

## PostgreSQL 인덱스 적용

### 1. WorldType 엔티티

세계관 정보를 저장하는 `WorldType` 엔티티에는 다음과 같은 인덱스를 적용했다.

```java
@Entity
@Table(name = "world_types")
public class WorldType {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 50)
    private String code;      // UNIQUE 제약으로 자동 인덱스 생성

    @Column(nullable = false)
    private Boolean isActive;

    @Column(nullable = false)
    private Integer sortOrder;
}
```

**적용 포인트:**

- `code` 컬럼에 `unique` 제약 조건을 설정하여 자동으로 인덱스가 생성된다.
- `findByCode()` 쿼리가 빠르게 실행된다.
- `isActive` 와 `sortOrder` 로 자주 조회되므로 복합 인덱스를 고려할 수 있다.

### 2. Member 엔티티

회원 정보를 저장하는 `Member` 엔티티도 unique 제약을 활용했다.

```java
@Entity
@Table(name = "member")
public class Member extends BaseEntity {

    @Column(name = "name", length = 20, unique = true)
    private String name; // ✅ 자동 인덱스 생성

    @Column(name = "nick_name", unique = true)
    private String nickName; // ✅ 자동 인덱스 생성
}
```

**적용 포인트:**

- 로그인 시 `name` 으로 회원 조회가 빠르게 실행된다.
- 중복 검사 쿼리(`existsByName`, `existsByNickName`)도 인덱스를 활용한다.

### 3. Caffeine 로컬 캐시 적용

인덱스만으로는 부족하다고 판단하여 Caffeine 로컬 캐시를 추가로 적용했다.

```java
@Configuration
@EnableCaching
public class CaffeineCacheConfig {

    @Bean
    @Primary
    public CacheManager caffeineCacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager("worldTypes");

        cacheManager.setCaffeine(Caffeine.newBuilder()
                .maximumSize(100)
                .expireAfterWrite(1, TimeUnit.HOURS) // 1시간 캐시
                .recordStats());

        return cacheManager;
    }
}
```

```java
// 서비스 레이어
@Cacheable(value = "worldTypes", key = "'activeWorldTypes'")
public List<WorldType> findAllActiveWorldTypeEntities() {
    log.info("DB에서 활성화된 세계관 목록 조회 (캐시 미스)");
    return worldTypeRepository.findActiveWorldTypesOrderBySortOrder();
}

@CacheEvict(value = "worldTypes", allEntries = true)
public WorldTypeResponse createWorldType(WorldTypeCreateRequest request) {
    // 생성 로직
}
```

**개선 결과:**

- **Before**: 서버 시작 시 3번 조회 + 30초마다 추가 조회
- **After**: 서버 시작 시 1번만 조회 → 1시간 동안 캐시 사용

## MongoDB 인덱스 적용

### 1. ChatMessage 복합 인덱스

채팅 메시지 같은 경우 특정 방의 메시지를 시간 순으로 조회하는 경우를 고려하여 복합 인덱스를 적용했다.

```java
@Document(collection = "chat_messages")
@CompoundIndex(name = "room_created_idx", def = "{'roomId': 1, 'createdAt': 1}")
public class ChatMessage {

    @Id
    private String messageId;

    @Indexed
    private String roomId; // ✅ 단일 인덱스

    @CreatedDate
    private Instant createdAt;
}
```

**복합 인덱스를 적용한 이유**

- 복합 인덱스 `{'roomId': 1, 'createdAt': 1}`는 다음 쿼리들을 모두 커버한다.
    - `findByRoomId()`
    - `findByRoomIdOrderByCreatedAtDesc()`
    - `findTop50ByRoomIdOrderByCreatedAtDesc()`

**실제 Repository 쿼리 메서드:**

```java
@Repository
public interface ChatMessageRepository extends MongoRepository<ChatMessage, String> {

    Page<ChatMessage> findByRoomIdOrderByCreatedAtDesc(String roomId, Pageable pageable);

    // 초기 로드: 최근 50건
    List<ChatMessage> findTop50ByRoomIdOrderByCreatedAtDesc(String roomId);

    // 방 삭제 시 메시지 일괄 삭제
    long deleteByRoomId(String roomId);
}
```

### 2. ChatRoom 인덱스

채팅방 조회 시 방 이름으로 검색하는 기능을 위해 인덱스를 추가했다.

```java
@Document(collection = "chat_rooms")
public class ChatRoom {

    @Id
    private String id;

    @Indexed
    private String roomName; // ✅ 방 이름 검색용 인덱스
}
```

## **MongoDB vs PostgreSQL 인덱스 차이**

두 데이터베이스에서 인덱스를 적용하며 느낀 차이점은 다음과 같다.

| **구분** | **PostgreSQL** | **MongoDB** |
| --- | --- | --- |
| **인덱스 생성** | DDL 또는 JPA 어노테이션 | `@Indexed`, `@CompoundIndex` 어노테이션 |
| **복합 인덱스** | `CREATE INDEX ON table(col1, col2)` | `@CompoundIndex(def = "{'col1': 1, 'col2': 1}")` |
| **Unique 제약** | `@Column(unique = true)` | `@Indexed(unique = true)` |
| **인덱스 순서** | 자동 결정 (옵티마이저) | 명시적 지정 (`1`: 오름차순, `-1`: 내림차순) |
| **성능 확인** | `EXPLAIN ANALYZE` | `explain("executionStats")` |

## 성능 개선 결과

인덱스와 캐시 적용 후 다음과 같은 성능 개선을 확인할 수 있었다.

### WorldType 조회 성능

- **쿼리 횟수**: 3번 → 1번 (66% 감소)
- **캐시 히트율**: 99%+ (1시간 TTL)
- **평균 응답 시간**: 직접 메모리 접근으로 네트워크 I/O 제거

### **ChatMessage 조회 성능**

- **인덱스 적용 전**: Collection Scan (전체 문서 조회)
- **인덱스 적용 후**: Index Scan (특정 roomId만 조회)

## 배운 점과 고려사항

### 1. 마스터 데이터 같은 경우 로컬 캐시가 효과적이다

WorldType과 같이 변경이 거의 없는 마스터 데이터는 Redis보다 Caffeine 로컬 캐시가 더 적합하다.

| **구분** | **Redis Cache** | **Caffeine Cache** |
| --- | --- | --- |
| 저장 위치 | Redis 서버 | 애플리케이션 메모리 |
| 직렬화 | 필요 
(LinkedHashMap 문제 발생 가능) | 불필요 (객체 그대로) |
| 성능 | 네트워크 I/O | 메모리 직접 접근 (더 빠름) |
| 적합한 용도 | 분산 캐시 | 로컬 마스터 데이터 |

### 2. 복합 인덱스 설계는 쿼리 패턴을 따라야 한다

MongoDB의 복합 인덱스는 왼쪽부터 순서대로 사용된다.

`{’roomId’: 1, ‘createdAt’: 1}` 인덱스 같은 경우

- `roomId`만 사용하는 쿼리 → 인덱스 활용 가능
- `createdAt`만 사용하는 쿼리 → 인덱스 활용 불가

⇒ 따라서 가장 우선순위가 높은 컬럼을 앞에 배치해야 한다.

### 3. Unique 제약은 자동으로 인덱스를 생성한다

PostgreSQL과 MongoDB 모두 unique 제약 조건을 설정하면 자동으로 인덱스가 생성된다.

비즈니스 로직상 중복이 허용되지 않는 컬럼(`code`, `name`, `nickName`)에는 반드시 unique 제약을 설정하는 것이 좋다.

### 4. 인덱스는 결코 공짜가 아니다

인덱스는 조회 성능을 향상시키지만 다음과 같은 비용이 발생한다.

- **쓰기 성능 저하:** INSERT/UPDATE/DELETE 시 인덱스도 함께 갱신된다.
- **저장 공간 증가**: 인덱스도 디스크 공간을 차지한다.
- **메모리 사용**: 인덱스는 메모리에 캐싱되므로 메모리를 소비한다.

⇒ 따라서 무분별하게 인덱스를 추가하기보다는 실제 쿼리 패턴을 분석하여 필요한 인덱스만 추가해야 한다.