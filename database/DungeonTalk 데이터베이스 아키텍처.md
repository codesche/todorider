# DungeonTalk 데이터베이스 아키텍처

## 블로그 링크
https://codesche.oopy.io/292de3f7-e3a8-8009-a38f-c2479948ca10

## DungeonTalk 프로젝트 링크

[https://github.com/DungeonTalk/dungeontalk-backend?tab=readme-ov-file](https://github.com/DungeonTalk/dungeontalk-backend?tab=readme-ov-file)

## 사용한 데이터베이스

- PostgreSQL, MongoDB, Redis(Valkey)

## 왜 여러 데이터베이스를 사용했는가?

| **요구사항** | **선택한 DB** | **선택 이유** |
| --- | --- | --- |
| 사용자, 인증 정보 | PostgreSQL | ACID 트랜잭션, 강한 일관성 |
| 채팅 메시지 (수백만 건 - 사용량 많음) | MongoDB | 수평 확장, 스키마 유연성 |
| 세션 관리, 캐싱 | Redis/Valkey | 빠른 조회, TTL 자동 만료 |

## PostgreSQL: 관계형 데이터베이스의 핵심

### 1. 아키텍처 개요

PostgreSQL은 DungeonTalk의 마스터 데이터를 저장한다.

- 사용자 정보 (Member)
- 인증 정보 (Auth)
- 게임 캐릭터 (GameCharacter)
- 종족별 스탯 (RaceStats)

**연결 풀: HikariCP**

```yaml
# application-dev.properties
# HikariCP Connection Pool Settings (Performance Optimization)
spring.datasource.hikari.maximum-pool-size=50
spring.datasource.hikari.minimum-idle=10
spring.datasource.hikari.connection-timeout=10000
spring.datasource.hikari.idle-timeout=300000
spring.datasource.hikari.max-lifetime=600000
spring.datasource.hikari.pool-name=DungeonTalkHikariCP
```

**Hikari 설정 설명:**

| **설정** | **값** | **의미** |
| --- | --- | --- |
| `maximum-pool-size` | 50 | 최대 연결 수 (부하 테스트 기준 최적값) |
| `minimum-idle` | 10 | 항상 유지할 대기 연결 수 |
| `connection-timeout` | 10000ms | 연결 획득 최대 대기 시간 |
| `idle-timeout` | 300000ms (5분) | 유휴 연결 유지 시간 |
| `max-lifetime` | 600000ms (10분) | 연결 최대 생존 시간 |

### 2. 엔티티 설계: BaseEntity와 UUID v7

**BaseEntity - 공통 필드 관리**

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public class BaseEntity {

    @Id
    private String id;  // UUIDv7 자동 생성

    @CreatedDate
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;

    @PrePersist
    public void generateIdIfNull() {
        if (this.id == null || this.id.isBlank()) {
            this.id = UuidV7Creator.create();  // UUIDv7 생성
        }
    }
}
```

**UUIDv7을 선택한 이유:**

| **방식** | **장점** | **단점** | **DungeonTalk 적용** |
| --- | --- | --- | --- |
| Auto Increment | 간단, 순차적 | 분산 환경 충돌 | ❌ |
| UUIDv4 | 완전 랜덤 | 인덱스 성능 저하 | ❌ |
| **UUIDv7** | 시간 순서 보장, 인덱스 효율 | - | ✅ **선택** |
| Snowflake | 순차적, 빠름 | 중앙 서버 필요 | ❌ |

**UUIDv7 구조:**

```java
unix_ts_ms (48비트) + rand_a (12비트) + rand_b (62비트)
└─ 시간 기반 정렬 가능 ─┘    └─ 충돌 방지 랜덤 ─┘

예시:
018d3f5a-3c7e-7000-8000-123456789abc
└─ 2025-10-17 10:30:00 UTC ─┘
```

**효과:**

- MongoDB와 PostgreSQL 모두 시간 순서 정렬에 효율적
- 분산 환경에서 중앙 ID 서버 없이도 충돌 방지 가능
- `created_at` 인덱스 없이도 시간 순서 조회 가능

## 3. 엔티티 관계 매핑

**Member 엔티티**

```java
@Entity
@Table(name = "member")
public class Member extends BaseEntity {

    @Column(name = "password", nullable = false)
    private String password;  // BCrypt 암호화

    @Column(name = "name", length = 20, unique = true)
    private String name;      // 로그인 ID

    @Column(name = "nick_name", unique = true)
    private String nickName;  // 게임 내 닉네임
}
```

**Auth 엔티티 - ManyToOne 관계**

```java
@Entity
@Table(name = "auth")
public class Auth extends BaseEntity {

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id")
    private Member member;

    @Column(name = "email", length = 50)
    private String email;

    @Column(name = "access_token", columnDefinition = "text")
    private String accessToken;

    @Column(name = "refresh_token", columnDefinition = "text")
    private String refreshToken;
}
```

**FetchType.LAZY의 중요성:**

```java
// ❌ EAGER Loading (기본값) - N+1 문제 발생
List<Auth> authList = authRepository.findAll();  // SELECT * FROM auth
for (Auth auth : authList) {
    auth.getMember().getName();  // SELECT * FROM member WHERE id = ? (N번 실행!)
}

// ✅ LAZY Loading + Fetch Join - 1번의 쿼리로 해결
@Query("SELECT a FROM Auth a JOIN FETCH a.member")
List<Auth> findAllWithMember();  // SELECT * FROM auth JOIN member ...
```

**관계 매핑 시 참고해야 할 사항**

```markdown
1. 기본값을 **LAZY**로 설정
2. 필요한 경우에만 **Fetch Join** 사용
3. **@EntityGraph**로 동적 로딩 제어

연관 관계의 기본 FetchType:
- **@ManyToOne**, **@OneToOne**: **EAGER** (기본값 위험!)
- **@OneToMany**, **@ManyToMany**: **LAZY**
→ 모든 관계를 **LAZY**로 명시하는 것이 안전
```

**GameCharacter 엔티티 - OneToOne 관계**

```java
@Entity
@Table(name = "game_character")
public class GameCharacter extends BaseEntity {

    @Column(name = "member_id")
    private String memberId;

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id", referencedColumnName = "id",
                insertable = false, updatable = false)
    private Member member;

    @Column(name = "race_id")
    private String raceId;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "race_id", referencedColumnName = "id",
                insertable = false, updatable = false)
    private RaceStats raceStats;

    // 스탯 필드들
    private Integer strength;
    private Integer willpower;
    private Integer intelligence;
    private Integer wisdom;
    private Integer dexterity;
    private Integer luck;
}
```

**insertable=false, updatable=false의 의미**:

- 읽기 전용 연관 관계에 한해서는 INSERT와 UPDATE가 실행되지 않도록 설정

```markdown
-- 문제 상황: 두 개의 필드가 같은 컬럼을 참조
CREATE TABLE game_character (
    id VARCHAR(255),
    member_id VARCHAR(255),  -- ← 이 컬럼을
    ...
);

-- member_id 컬럼을 두 곳에서 사용
1. private String memberId;              -- 직접 관리 (INSERT/UPDATE)
2. private Member member;                -- 읽기 전용 (JOIN)

-- insertable=false, updatable=false:
-- "이 연관 관계는 읽기 전용입니다. INSERT/UPDATE 시 무시하세요"
```

- 예시

```java
// ✅ 올바른 사용법
GameCharacter character = new GameCharacter();
character.setMemberId("member-123");  // FK 직접 설정
characterRepository.save(character);

Member member = character.getMember();  // LAZY 로딩으로 Member 조회 (읽기 전용)

// ❌ 잘못된 사용법
character.setMember(member);  // insertable=false이므로 DB에 반영 안 됨!
```

### 4. JPA 쿼리 최적화

- **N+1 문제 해결 사례**

**문제 코드:**

```java
// 1번의 쿼리로 GameCharacter 조회
List<GameCharacter> characters = characterRepository.findAll();

// N번의 추가 쿼리 발생!
for (GameCharacter character : characters) {
    String raceName = character.getRaceStats().getName();  // SELECT * FROM race_stats WHERE id = ?
}

// 총 쿼리 수: 1 + N (100명이면 101번!)
```

**해결 방법 1: Fetch Join**

```java
@Query("SELECT gc FROM GameCharacter gc JOIN FETCH gc.raceStats")
List<GameCharacter> findAllWithRaceStats();

// 생성되는 SQL:
SELECT gc.*, rs.*
FROM game_character gc
INNER JOIN race_stats rs ON gc.race_id = rs.id
```

**해결 방법 2: @EntityGraph**

```java
@EntityGraph(attributePaths = {"member", "raceStats"})
List<GameCharacter> findAll();

// 두 개의 연관 관계를 한 번에 로딩
```

**성능 비교**

```java
Before (N+1 발생):
- 쿼리 수: 101번 (100명 조회 시)
- 응답 시간: ~500ms

After (Fetch Join):
- 쿼리 수: 1번
- 응답 시간: ~50ms

→ 90% 성능 향상!
```

## MongoDB: 대용량 메시지 저장소

### 1. 왜 MongoDB를 선택했는가?

- **채팅 메시지의 특성:**
    - 대량 데이터 (일 수십만 ~ 수백만 건)
        - 대용량, 입력/저장이 빈번한 데이터
    - 읽기 중심 (Write Once, Read Many)
    - 시간 역순 정렬 조회 빈번
    - 스키마 유연성 필요 (메시지 타입별 추가 필드)

- **PostgreSQL vs MongoDB 비교:**

| **항목** | **PostgreSQL** | **MongoDB** |
| --- | --- | --- |
| 수평 확장 | 어려움 (Sharding 복잡) | 쉬움 (Built-in Sharding) |
| 시간 역순 정렬 | 인덱스 필수 | Timestamp 기반 효율적 |
| 스키마 변경 | ALTER TABLE 비용 높음 | 유연함 |

### 2. ChatMessage

```java
@Document(collection = "chat_messages")
@CompoundIndex(name = "room_created_idx", def = "{'roomId': 1, 'createdAt': 1}")
public class ChatMessage {

    @Id
    private String messageId;  // UUIDv7

    @Indexed
    private String roomId;     // 채팅방 ID (파티셔닝 키 후보)

    private String senderId;
    private String receiverId;
    private String content;

    private MessageType type;  // JOIN, TALK, LEAVE

    @CreatedDate
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;
}
```

**인덱스 전략:**

```java
// 1. 단일 필드 인덱스
db.chat_messages.createIndex({ "roomId": 1 })

// 2. 복합 인덱스 (쿼리 최적화)
db.chat_messages.createIndex({ "roomId": 1, "createdAt": 1 })

// 실제 쿼리:
db.chat_messages.find({ roomId: "room-123" })
    .sort({ createdAt: -1 })   // 인덱스로 정렬 커버
    .limit(50)
```

**인덱스 선택 기준**

- **자주 조회되는 필드**: `roomId`
- **정렬 필드**: `createdAt`
- **복합 인덱스 순서**: 등호(’=’) 조건 → 범위 조건 → 정렬

**왜 (roomId, createdAt) 순서인가?**

- `roomId`: 등호 조건 (=)
- `createdAt`: 정렬 조건 (ORDER BY)
- 인덱스 스캔으로 정렬까지 해결 (Sort 연산 제거)

### 3. ChatRoomMember - 멤버 상태 관리

```java
@Document(collection = "chat_room_members")
public class ChatRoomMember {

    @Id
    private String id;

    @Indexed
    private String roomId;

    @Indexed
    private String memberId;

    @Enumerated(EnumType.STRING)
    private Status status;  // ONLINE, OFFLINE

    private Instant joinedAt;
    private Instant lastSeenAt;
}
```

**왜 PostgreSQL이 아닌 MongoDB에 저장하는가?**

| **측면** | **이유** |
| --- | --- |
| 데이터 양 | 누적 참여 이력 (수백만 건) |
| 쿼리 패턴 | roomId 기반 조회 → Sharding 효율적 |
| 일관성 요구 | 최종 일관성으로 충분 (실시간 세션은 Redis 사용) |
| 확장성 | 채팅방 증가에 따라 수평 확장 |

## Redis/Valkey: 세션과 캐시의 이중 전략

### 0. Valkey 참고

| **구분** | **Valkey** | **Redis** |
| --- | --- | --- |
| **핵심 개발 방향** | 성능 최적화 (멀티스레딩, RDMA 지원, 메모리 효율성 등) | AI, 편의 기능(VectorSet, AutoComplete 등) 추가 |
| **라이선스** | BSD 라이선스 | SPL 라이선스 (2024년 3월 변경) |
| **성능** | 멀티스레딩 도입으로 인한 높은 성능과 낮은 지연 시간 | 8.0 버전 이후 멀티스레딩 지원이 강화되었으나, 초기 Valkey는 더 나은 성능을 보여주었음 |
| **호환성** | Redis 7.2.x와 완벽하게 호환 | Valkey와 같은 오픈소스 버전은 호환성을 유지하나, Redis 8.0 이후로는 일부 기능에서 차이 발생 가능 |
| **커뮤니티** | Linux Foundation이 후원하는 빠른 성장의 오픈 소스 커뮤니티 | 기존의 방대한 사용자 기반 |

**참고링크:**

[https://www.threads.com/@bear_dba/post/DJVBluQT7iC/redis와-valkey-무엇을-선택해야-할까redis-724-이후-라이선스-변경으로-valkey가-포크되면서-이제는-redis-vs-valke](https://www.threads.com/@bear_dba/post/DJVBluQT7iC/redis%EC%99%80-valkey-%EB%AC%B4%EC%97%87%EC%9D%84-%EC%84%A0%ED%83%9D%ED%95%B4%EC%95%BC-%ED%95%A0%EA%B9%8Credis-724-%EC%9D%B4%ED%9B%84-%EB%9D%BC%EC%9D%B4%EC%84%A0%EC%8A%A4-%EB%B3%80%EA%B2%BD%EC%9C%BC%EB%A1%9C-valkey%EA%B0%80-%ED%8F%AC%ED%81%AC%EB%90%98%EB%A9%B4%EC%84%9C-%EC%9D%B4%EC%A0%9C%EB%8A%94-redis-vs-valke)

### 1. 왜 Redis 인스턴스를 2개 사용하는가?

DungeonTalk는 **Session용**과 **Cache용** Valkey(Redis 호환)를 분리 운영하고 있다.

```yaml
# Session Valkey - ElastiCache
spring.session.store-type=${SPRING_SESSION_STORE_TYPE}
spring.data.session.host=${SPRING_DATA_SESSION_HOST}
spring.data.session.port=${SPRING_DATA_SESSION_PORT}

# Cache Valkey - EC2 로컬
spring.cache.type=${SPRING_CACHE_TYPE}
spring.redis.cache.host=${SPRING_REDIS_CACHE_HOST}
spring.redis.cache.port=${SPRING_REDIS_CACHE_PORT}
```

**분리 이유:**

| **항목** | **Session Valkey** | **Cache Valkey** |
| --- | --- | --- |
| **용도** | 세션 관리, 실시간 데이터 | 조회 결과 캐싱 |
| **데이터 특성** | 휘발성 높음 | 상대적으로 안정적 |
| **TTL** | 짧음 (30분) | 길 수 있음 (1시간+) |
| **트래픽 패턴** | 매 요청 (읽기/쓰기 균형) | 캐시 미스 시만 쓰기 |
| **장애 영향** | 치명적 (세션 유실) | 제한적 (DB 폴백) |

### 2. Lettuce Connection Pool 설정

```yaml
# Lettuce Connection Pool Settings (Performance Optimization)
spring.data.redis.lettuce.pool.max-active=50
spring.data.redis.lettuce.pool.max-idle=20
spring.data.redis.lettuce.pool.min-idle=10
spring.data.redis.lettuce.pool.max-wait=3000ms
spring.data.redis.timeout=5000ms
```

### 3. ValkeyConfig - 이중 Redis 설정

```java
@Configuration
public class ValkeyConfig {

    @Value("${spring.data.session.host}")
    private String sessionRedisHost;

    @Value("${spring.redis.cache.host}")
    private String cacheRedisHost;

    // Session Redis 연결 팩토리
    @Bean(name = "sessionRedisConnectionFactory")
    @Primary
    public RedisConnectionFactory sessionRedisConnectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
        config.setHostName(sessionRedisHost);
        config.setPort(sessionRedisPort);

        LettuceClientConfiguration.LettuceClientConfigurationBuilder builder =
            LettuceClientConfiguration.builder()
                .commandTimeout(Duration.ofSeconds(10));

        if (sessionRedisSslEnabled) {
            builder.useSsl();  // AWS ElastiCache SSL 지원
        }

        return new LettuceConnectionFactory(config, builder.build());
    }

    // Cache Redis 연결 팩토리
    @Bean(name = "cacheRedisConnectionFactory")
    public RedisConnectionFactory cacheRedisConnectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
        config.setHostName(cacheRedisHost);
        config.setPort(cacheRedisPort);
        return new LettuceConnectionFactory(config);
    }

    // Session용 RedisTemplate
    @Bean(name = "sessionRedisTemplate")
    public RedisTemplate<String, String> sessionRedisTemplate(
            @Qualifier("sessionRedisConnectionFactory") RedisConnectionFactory factory) {
        RedisTemplate<String, String> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new StringRedisSerializer());
        return template;
    }
}
```

**@Qualifier의 중요성**:

```java
// ❌ 여러 Bean이 있을 때 자동 주입 실패
@Autowired
private RedisConnectionFactory connectionFactory;  // 어떤 Factory를 주입할지 모호!

// ✅ @Qualifier로 명시
@Autowired
@Qualifier("sessionRedisConnectionFactory")
private RedisConnectionFactory connectionFactory;
```

### 4. 세션 관리: TTL 기반 자동 만료

```java
@Service
@RequiredArgsConstructor
public class ChatSessionService {

    private final ValkeyService valkeyService;
    private final ChatRoomRepository chatRoomRepository;
    private final ObjectMapper objectMapper;

    // ChatConstants에 정의된 상수
    private static final int DEFAULT_SESSION_TIMEOUT_SECONDS = 1800;  // 30분

    /**
     * 세션 시작 (채팅방 입장)
     */
    @Transactional
    public ChatSessionDto startSession(String roomId, String memberId,
                                       String nickname, String websocketSessionId) {
        log.info("채팅 세션 시작 요청: roomId={}, memberId={}, nickname={}", roomId, memberId, nickname);

        // 채팅방 존재 확인
        ChatRoom room = chatRoomRepository.findById(roomId)
                .orElseThrow(() -> new ChatException(ErrorCode.CHAT_ROOM_NOT_FOUND, "roomId=" + roomId));

        Instant now = Instant.now();

        ChatSessionDto sessionDto = ChatSessionDto.builder()
                .roomId(roomId)
                .memberId(memberId)
                .nickname(nickname)
                .status(Status.ONLINE)
                .joinedAt(now)
                .lastActivity(now)
                .websocketSessionId(websocketSessionId)
                .build();

        String sessionKey = buildSessionKey(roomId, memberId);  // "chat:session:{roomId}:{memberId}"
        String sessionData = serializeSessionData(sessionDto);

        log.info("Valkey 세션 저장: sessionKey={}, timeout={}초", sessionKey, DEFAULT_SESSION_TIMEOUT_SECONDS);
        valkeyService.setWithExpiration(sessionKey, sessionData, DEFAULT_SESSION_TIMEOUT_SECONDS);

        return sessionDto;
    }

    /**
     * 세션 연장 (활동 시간 갱신)
     */
    public void extendSession(String roomId, String memberId) {
        String sessionKey = buildSessionKey(roomId, memberId);

        if (!valkeyService.exists(sessionKey)) {
            log.warn("세션 연장 실패 - 세션이 존재하지 않음: roomId={}, memberId={}", roomId, memberId);
            return;
        }

        try {
            // 기존 세션 데이터 조회
            String sessionData = valkeyService.get(sessionKey);
            ChatSessionDto sessionDto = objectMapper.readValue(sessionData, ChatSessionDto.class);

            // 마지막 활동 시간 갱신
            ChatSessionDto updatedSession = sessionDto.toBuilder()
                    .lastActivity(Instant.now())
                    .build();

            // Valkey에 갱신된 세션 저장 및 TTL 연장
            valkeyService.setWithExpiration(sessionKey, serializeSessionData(updatedSession),
                                           DEFAULT_SESSION_TIMEOUT_SECONDS);

            log.debug("채팅 세션 연장: roomId={}, memberId={}", roomId, memberId);
        } catch (JsonProcessingException e) {
            log.error("세션 데이터 역직렬화 실패: roomId={}, memberId={}", roomId, memberId, e);
        }
    }

    /**
     * Heartbeat 처리 (세션 유지)
     */
    public boolean heartbeat(String roomId, String memberId) {
        String sessionKey = buildSessionKey(roomId, memberId);

        if (!valkeyService.exists(sessionKey)) {
            log.warn("Heartbeat 실패 - 세션이 존재하지 않음: roomId={}, memberId={}", roomId, memberId);
            return false;
        }

        // TTL만 연장 (데이터 재전송 없이 효율적)
        valkeyService.expire(sessionKey, DEFAULT_SESSION_TIMEOUT_SECONDS);
        log.debug("Heartbeat 처리: roomId={}, memberId={}", roomId, memberId);

        return true;
    }

    private String buildSessionKey(String roomId, String memberId) {
        return CHAT_SESSION_PREFIX + roomId + ":" + memberId;  // "chat:session:"
    }
}
```

**TTL 전략의 장점**

1. **메모리 누수방지**

```markdown
**문제:** WebSocket 비정상 종료 시 세션 정리 안 됨
**해결:** 30분 후 Redis가 자동 삭제
**결과:** 메모리 누수 없음, 명시적 정리 코드 불필요
```

1. **세션 유효성 검증**

```java
// 매번 DB 조회 없이 Redis 존재 여부로 판단
if (!valkeyService.exists(sessionKey)) {
    throw new SessionExpiredException("세션이 만료되었습니다.");
}
```

1. **성능 최적화**

```markdown
Heartbeat 처리:
Before: Session 데이터 전체 재전송 (1KB × 1000명 = 1MB/30초)
After: EXPIRE 명령만 전송 (1B × 1000명 = 1KB/30초)
→ 네트워크 트래픽 감소
```

### 5. Caffeine 로컬 캐시 - 마스터 데이터 최적화

```java
@Configuration
@EnableCaching
public class CaffeineCacheConfig {

    @Bean
    @Primary
    public CacheManager caffeineCacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager("worldTypes", "chatRooms");

        cacheManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(100)                     // 최대 100개 항목
            .expireAfterWrite(1, TimeUnit.HOURS)  // 1시간 후 만료
            .recordStats());                      // 캐시 통계 수집

        return cacheManager;
    }
}
```

**Redis Cache vs Caffeine Cache**

| **항목** | **Redis Cache** | **Caffeine Cache** |
| --- | --- | --- |
| 저장 위치 | Redis 서버 | JVM 힙 메모리 |
| 네트워크 | 필요 (1-5ms) | 불필요 |
| 성능 | ~1ms | **~1μs (마이크로초)** |
| 데이터 공유 | 여러 인스턴스 간 공유 | 인스턴스별 독립 |
| 적합한 데이터 | 자주 변경되는 데이터 | **변경 없는 마스터 데이터** |

**적용 사례**

```java
/**
 * 모든 활성화된 WorldType 엔티티 조회 (내부 서비스 간 호출용)
 * Caffeine 로컬 캐시 적용: 1시간 동안 메모리에 캐시 유지
 */
@Cacheable(value = "worldTypes", key = "'activeWorldTypes'")
public List<WorldType> findAllActiveWorldTypeEntities() {
    log.info("DB에서 활성화된 세계관 목록 조회 (캐시 미스)");
    return worldTypeRepository.findActiveWorldTypesOrderBySortOrder();
}

// 실제 Repository 쿼리:
@Query("SELECT w FROM WorldType w WHERE w.isActive = true ORDER BY w.sortOrder ASC, w.id ASC")
List<WorldType> findActiveWorldTypesOrderBySortOrder();

// 성능 비교:
Before (DB 직접 조회):
- 응답 시간: ~50ms
- 동시 요청 시 DB 부하 증가

After (Caffeine Cache):
- 응답 시간: ~0.1ms (500배 향상)
- 서버 시작 후 1번만 DB 조회
```

**Cache Stampede 문제**

```java
// 문제 상황 (docs/worldtypes-query-optimization.md 참고)
// 서버 시작 시 여러 스레드가 동시에 DB 조회
public List<WorldType> getAllActiveWorldTypes() {
    if (cachedActiveWorldTypes == null || isExpired()) {
        // 여러 스레드가 동시에 이 체크를 통과!
        cachedActiveWorldTypes = worldTypeRepository.findActiveWorldTypesOrderBySortOrder();
        // DB 쿼리 3번 실행됨!
    }
    return cachedActiveWorldTypes;
}

// 해결: Caffeine 캐시 사용
@Cacheable(value = "worldTypes", key = "'activeWorldTypes'")
public List<WorldType> findAllActiveWorldTypeEntities() {
    log.info("DB에서 활성화된 세계관 목록 조회 (캐시 미스)");
    // Caffeine이 내부적으로 동기화 처리
    // 첫 요청만 DB 조회, 나머지는 대기 후 캐시 반환
    return worldTypeRepository.findActiveWorldTypesOrderBySortOrder();
}
```