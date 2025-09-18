
# 프로젝트 회고 (25. 06 ~ 25. 08)

## 들어가며

최근 몇 달 동안 진행한 세 가지 프로젝트를 회고 차원에서 정리해봤다.

- **MoodBook**: 감정 기반 도서 추천 플랫폼  
  - https://github.com/moodbook-space/moodbook-backend
- **DungeonTalk**: 실시간 TRPG 게임 플랫폼  
  - https://github.com/DungeonTalk/dungeontalk-backend
- **Auth 프로젝트:** 회원가입/로그인 인증 시스템  
  - https://github.com/codesche/devcenter/tree/main/authproject

**📐 프로젝트를 진행하면서 적용한 규칙**

- 아무리 AI를 잘 활용한다 할지라도 100% 프로젝트 요구사항을 만족시키기란 불가능하다.
- 어느 정도 도메인 지식과 개발 경험이 갖춰지지 않으면 코어 개발자로 성장하기 어렵다.
- AI를 활용하는 팀원들의 성향을 고려하여 몇 가지 규칙을 세워서 프로젝트를 진행하였다.
- 회고를 위해 프로젝트를 진행하면서 적용한 규칙과 그에 대한 사례와 예제를 정리해봤다.

## 규칙 요약 표

| 번호 | 규칙 |
| --- | --- |
| 1 | 전역 예외 처리(@ControllerAdvice) + 의미 있는 도메인 예외 |
| 2 | 공통 API 응답객체 필요 |
| 3 | 비즈니스 로직은 서비스 계층에만 작성 (컨트롤러 계층에는 작성하지 않음) |
| 4 | DTO 객체 활용하여 데이터 전달하기 |
| 5 | 캐싱을 도입하여 성능 개선에 대한 케이스 만들 수 있어야 함 |
| 6 | JWT 토큰 활용하여 기본적인 회원가입/로그인 기능 구현 (RefreshToken은 Redis에 저장) |
| 7 | 인증 관련 보안 수준을 어느 정도 고려할 수 있는 형태로 구현 (시큐어 코딩) |
| 8 | 전반적인 DB 설계나 개발 로직은 대규모 트래픽을 충분히 대비할 수 있는 형태로 개발 |
| 9 | 모니터링 + 부하테스트 할 수 있는 방향으로 개발 |
| 10 | LocalDateTime이 아닌 Instant 클래스로 적용 (확장성 고려) |
| 11 | record 쓰지 않고 dto로 작성 |
| 12 | @Setter 어노테이션은 지양 |
| 13 | 효율적인 디자인 패턴 적용 (생성, 구조, 행위 패턴) |
| 14 | JUnit 5 + Mockito 단위테스트 작성 |
| 15 | 가급적 kotlin 코드는 절대 나타나지 않도록 |
| 16 | 코드 작성 시 패키지 경로를 상세하게 나타내기 |
| 17 | 상황마다 적용하기 좋은 알고리즘 반영 |
| 18 | entity(dao)는 절대로 매개변수로 사용하지 말 것 |
| 19 | JPA N+1 문제 발생하지 않은 코드 작성 |
| 20 | 성능 최적화를 고려한 코드 작성 |
| 21 | 도메인별 서비스 분리 없이 한 클래스에 몰아넣지 않기 |
| 22 | 기본 LAZY, 조회 전용은 fetch join/EntityGraph/전용 쿼리 DTO 활용 |
| 23 | 엔티티를 응답으로 직렬화 |
| 24 | Service 메서드 단위 @Transactional (조회 전용은 readOnly=true) 로 작성 |
| 25 | 프로파일링 → 핫스팟 파악 → 캐시 전략(TTL/키/무효화) 설계 후 적용 |
| 26 | 요청/응답/도메인 DTO 분리, 검증은 요청 DTO에 @Valid |
| 27 | 조건 기반 페이징/필터 쿼리 설계, 인덱스 점검 |
| 28 | 단위(서비스) 우선 + 슬라이스(@DataJpaTest, @WebMvcTest) 병행 |
| 29 | Testcontainers/Embedded 활용, 시크릿은 테스트용 별도 |
| 30 | 무상태 서버, 확장 고려한 코드 작성 |
| 31 | 로그·메트릭·트레이스 구성 |
| 32 | Access는 클라이언트, Refresh는 서버측 저장(DB/Redis) + 회전 |
| 33 | 환경 변수/Secret Manager 사용, @ConfigurationProperties로 주입 |
| 34 | 외부 I/O는 트랜잭션 밖에서 수행하거나 SAGA/Outbox 패턴 |
| 35 | 낙관적 락 + 재시도를 기본, 필요한 곳만 비관적 락 |
| 36 | 패키지 의존 방향을 한쪽으로 고정 (도메인→애플리케이션→인프라) |
| 37 | 필요한 Origin/Method만 화이트리스트 |
| 38 | 마스킹/필터링 적용, 요청/응답 바디 로깅은 샘플링 |
| 39 | var 형식은 지양 |
| 40 | 코드를 작성함에 있어 충분히 이해할 수 있도록 주석 반영 |
| 41 | @Getter, @Builder 어노테이션 활용. Setter 사용은 지양 |
| 42 | Request/Response DTO 코딩 스타일 일관 유지 (from/toEntity 구분) |
| 43 | DTO에도 @Builder 적용, entity → dto 변환 적극 활용 |
| 44 | Service 계층은 인터페이스로 만들지 않음 |
| 45 | 패키지명 전체를 그대로 변수/클래스명으로 사용하지 않음 |
| 46 | yml, build.gradle 확실하게 작성 |
| 47 | entity 작성 시 상황에 따라 AccessLevel.PRIVATE/PROTECTED 반영 |
| 48 | JPA dirty-checking 적극 반영 |
| 49 | 부모-자식 PK-FK 관계 삭제 연계 처리 (연관된 객체 삭제 보장) |
| 50 | BaseTime을 적용해 생성/수정 시간 관리 |
| 51 | BaseTime이나 엔티티에 protected 키워드 사용 지양 |
| 52 | 자료형별 발생 가능한 예외 고려 |
| 53 | 글로벌 예외 처리와 공통 API 응답 객체 코드 스타일 일관성 유지 |
| 54 | REST API 규칙에 맞는 주소 형식 사용 |
| 55 | Spring Boot 3.5.x 기준, @MockitoBean 활용 테스트 작성 |
| 56 | 로그인/회원가입 시 @AuthenticationPrincipal 적용 |
| 57 | 로그인/회원가입 시 CustomUserDetail 적용 |
| 58 | 가독성이 좋은 코드 작성 |

---

## 적용한 규칙에 대한 정리

### 규칙 1 — 전역 예외 처리(@ControllerAdvice)

**설명:** 전역 예외 처리를 한 곳에서 처리하여 try-catch를 제거하고, 예외를 의미 있는 메시지로 변환한다.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(DomainException.class)
    public ResponseEntity<ApiResponse<?>> handleDomainException(DomainException e) {
        return ResponseEntity.badRequest().body(ApiResponse.fail(e.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<?>> handleException(Exception e) {
        return ResponseEntity.internalServerError().body(ApiResponse.fail("예상치 못한 오류 발생"));
    }
}
```

> **회고:** 전역 예외 처리기를 생성함으로써 API 호출 시 발생하는 에러 메시지를 일관된 형태로 맞춤과 동시에 모든 에러가 `ApiResponse.fail()` 형태로 반환되어 협업 효율이 증가했다.

---

### 규칙 2 — 공통 API 응답 객체 필요

**설명:** 모든 API 응답을 일관된 형태로 관리하여 유지보수성을 높인다.

```java
@Getter
@Builder
public class ApiResponse<T> {
    private boolean success;
    private String message;
    private T data;

    public static <T> ApiResponse<T> ok(T data) {
        return ApiResponse.<T>builder()
                .success(true)
                .message("success")
                .data(data)
                .build();
    }

    public static <T> ApiResponse<T> fail(String message) {
        return ApiResponse.<T>builder()
                .success(false)
                .message(message)
                .build();
    }
}
```

> **회고:** Auth 프로젝트에서 로그인 API 응답이 처음엔 그냥 `Map<String, String>` 이었는데 `ApiResponse`를 도입하고 나니 테스트 코드 작성이 훨씬 수월해졌다.

---

### 규칙 3 — 비즈니스 로직은 서비스 계층에만 작성

**설명:** 컨트롤러는 요청/응답 DTO 매핑만 담당하고, 비즈니스 로직은 서비스 계층에서만 작성한다.

```java
// ❌ 나쁜 예시
@PostMapping("/books")
public ResponseEntity<ApiResponse<BookResponse>> createBook(@RequestBody BookRequest request) {
    Book book = new Book(request.getTitle(), request.getAuthor());
    bookRepository.save(book);
    return ResponseEntity.ok(ApiResponse.ok(BookResponse.fromEntity(book)));
}

// ✅ 좋은 예시
@PostMapping("/books")
public ResponseEntity<ApiResponse<BookResponse>> createBook(@RequestBody BookRequest request) {
    return ResponseEntity.ok(ApiResponse.ok(bookService.createBook(request)));
}
```

---

### 규칙 4 — DTO 객체 활용하여 데이터 전달하기

**설명:** 엔티티를 직접 노출하지 않고 DTO를 통해 데이터 전달.

```java
@Getter
@Builder
public class BookResponse {
    private String title;
    private String author;

    public static BookResponse fromEntity(Book book) {
        return BookResponse.builder()
                .title(book.getTitle())
                .author(book.getAuthor())
                .build();
    }
}
```

---

### 규칙 5 — 캐싱 도입으로 성능 개선

```java
@Service
@RequiredArgsConstructor
public class BookCacheService {

    private final RedisTemplate<String, Object> redisTemplate;

    public void cacheBook(String key, BookResponse book) {
        redisTemplate.opsForValue().set(key, book, 10, TimeUnit.MINUTES);
    }

    public BookResponse getBook(String key) {
        return (BookResponse) redisTemplate.opsForValue().get(key);
    }
}
```

> **회고:** 인기 도서 조회 API가 반복 호출될 때 Redis 캐싱을 적용했으면 하는 아쉬움이 남는다.

---

### 규칙 6 — JWT 토큰 기반 회원가입/로그인 (RefreshToken은 Redis 저장)

**설명:** AccessToken은 클라이언트, RefreshToken은 Redis에 TTL 적용.

```java
public AuthResponse login(AuthRequest request) {
    Member member = memberRepository.findByUsername(request.getUsername())
            .orElseThrow(() -> new AuthException("존재하지 않는 회원이다"));

    // 비밀번호 체크
    if (!passwordEncoder.matches(request.getPassword(), member.getPassword())) {
        throw new AuthException("비밀번호가 일치하지 않는다");
    }

    String accessToken = jwtTokenProvider.createAccessToken(member.getId().toString());
    String refreshToken = jwtTokenProvider.createRefreshToken();

    redisTemplate.opsForValue().set("RT:" + member.getId(), refreshToken, 7, TimeUnit.DAYS);

    return new AuthResponse(accessToken, refreshToken);
}
```

> **회고:** Redis TTL을 활용하니 만료 처리와 토큰 회전이 훨씬 쉬워졌다.

---

### 규칙 7 — 인증 관련 보안 수준 고려 (시큐어 코딩)

**설명:** 비밀번호는 평문 저장 금지, JWT Secret은 환경 변수로 관리.

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

---

### 규칙 8 — 대규모 트래픽 대비 DB 설계

**설명:** 인덱스 설계, 파티셔닝 고려, 읽기/쓰기 분리.

```sql
CREATE INDEX idx_books_title ON books(title);
```

> **회고:** MoodBook에서 도서 검색 API가 느려서 title에 인덱스를 추가했더니 쿼리 속도가 10배 정도 빨라졌다.

---

### 규칙 10 — LocalDateTime 대신 Instant 사용

**설명:** 타임존 이슈 없이 전 세계 어디서든 동일한 시간 기록 가능.

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class BaseTime {

    @CreatedDate
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;
}
```

---

### 규칙 11 — record 쓰지 않고 DTO 작성

**설명:** record가 편하긴 하지만, 확장성과 `@Builder`, `@Valid` 적용을 고려하면 DTO는 class로 작성하는 것이 좋다.

```java
@Getter
@NoArgsConstructor
public class BookRequest {

    @NotBlank(message = "제목은 필수 값이다")
    private String title;

    @NotBlank(message = "저자는 필수 값이다")
    private String author;

    @Builder
    public BookRequest(String title, String author) {
        this.title = title;
        this.author = author;
    }
}
```

> **회고:** record DTO를 썼다가, Lombok `@Builder` 와 Validation을 적용할 수 없어서 class로 교체했다. class 기반 DTO가 확장성 면에 있어 확실히 유연하다.

---

### 규칙 12 — @Setter 어노테이션 지양

**설명:** 불변성을 지켜 엔티티/DTO가 예측 불가능하게 변경되지 않도록 하였다.

```java
@Getter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class MemberResponse {
    private String username;
    private String nickName;
}
```

> **회고:** MoodBook 초기에는 `@Setter`를 남발하다가, 어디서 값이 변경되는지 추적이 안 됐다. `@Builder`와 생성자만으로 값 주입을 통제하면서 안정성이 올라갔다.

---

### 규칙 14 — JUnit 5 + Mockito 단위 테스트 작성

**설명:** 서비스 단위의 독립적인 테스트로 안정성을 확보한다.

```java
@ExtendWith(MockitoExtension.class)
class ChatMessageServiceTest {

    @Mock
    ChatMessageRepository chatMessageRepository;

    @Mock
    MemberRepository memberRepository;

    @Mock
    RedisPublisher redisPublisher;

    @Mock
    ObjectMapper objectMapper;

    @Mock
    ChatRoomService chatRoomService;

    @InjectMocks
    ChatMessageService chatMessageService;

    @Test
    @DisplayName("TALK: 저장 후 Redis로 브로드캐스트하고 DTO 반환")
    void process_TALK_publishAndReturn() throws Exception {
        // given
        ChatMessageSendRequestDto dto = talkReq("room-1", "u-1", "hello");

        Member sender = mock(Member.class);
        given(sender.getNickName()).willReturn("Neo");
        given(memberRepository.findById("u-1")).willReturn(Optional.of(sender));

        // save 호출 시 저장된 엔티티 그대로 반환
        given(chatMessageRepository.save(any(ChatMessage.class)))
            .willAnswer(inv -> inv.getArgument(0));

        given(objectMapper.writeValueAsString(any(ChatMessageDto.class)))
            .willReturn("{json}");

        // when
        ChatMessageDto result = chatMessageService.processMessage(dto);

        // then
        assertThat(result).isNotNull();
        assertThat(result.getSenderNickname()).isEqualTo("Neo");
        then(redisPublisher).should().publish(eq("room-1"), eq("{json}"));
        then(chatRoomService).should(never()).joinRoom(anyString(), anyString());
        then(chatRoomService).should(never()).leaveRoom(anyString(), anyString());
    }
}
```

---

### 규칙 15 — Kotlin 코드는 절대 나타나지 않도록

**설명:** 팀 내 기술 스택 일관성을 위해 Kotlin 금지.

```java
// ✅ Java 기반 DTO
@Getter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class ChatRequest {
    private String roomId;
    private String message;
}
```

> **회고:** 팀원 중 한 명이 Kotlin 코드로 DTO를 작성했는데, 빌드 파이프라인과 스타일이 꼬여버렸다. 규칙으로 못 박아두니 유지보수가 훨씬 쉬워졌다.

---

### 규칙 18 — entity(DAO)는 매개변수로 사용하지 않기

**설명:** 요청/응답은 DTO로만 처리하고, 엔티티는 서비스/리포지토리 내부에서만 다룬다.

```java
// ❌ 나쁜 예시
public void createBook(Book book) { bookRepository.save(book); }

// ✅ 좋은 예시
public void createBook(BookRequest request) {
    Book book = Book.builder()
            .title(request.getTitle())
            .author(request.getAuthor())
            .build();
    bookRepository.save(book);
}
```

> **회고:** Auth 프로젝트 초반에 엔티티를 컨트롤러 매개변수로 받았다가, Lazy 로딩 문제와 직렬화 문제를 겪고 나서 DTO 기반으로 전환했다.

---

### 규칙 19 — JPA N+1 문제 방지

**설명:** fetch join, EntityGraph, DTO Projection, QueryDSL 활용.

```java
// Book과 Author를 fetch join
@Query("SELECT b FROM Book b JOIN FETCH b.author WHERE b.id = :id")
Optional<Book> findBookWithAuthor(@Param("id") Long id);

// Projection DTO 활용
@Query("""
        SELECT new org.com.moodbook.book.dto.BookResponse(
            b.id, b.isbn13, b.title, b.author, b.publisher, b.pubDate,
            b.reputation, b.coverImage, b.description, b.categoryName,
            b.createdAt, coalesce(bc.viewCount, 0)
        )
        FROM Book b
        LEFT JOIN BookCount bc ON b.id = bc.book.id
        ORDER BY b.pubDate DESC
    """)
Page<BookResponse> findAllByCreatedAt(Pageable pageable);

/* N + 1 문제 처리 */
@EntityGraph(attributePaths = "bookCount")
List<Book> findByIsbn13In(Collection<String> isbn13);

/* N + 1 문제 처리 */
@Query("SELECT b FROM Book b LEFT JOIN FETCH b.bookCount")
List<Book> findAllBooks();
```

---

### 규칙 20 — 성능 최적화를 고려한 코드 작성

**설명:** 쿼리 최적화, 캐시, 적절한 자료구조 사용.

```java
// HashMap 활용하여 사용자별 도서 추천 캐싱
private final Map<String, List<Book>> userRecommendCache = new HashMap<>();

public List<Book> getUserRecommendations(String userId) {
    return userRecommendCache.computeIfAbsent(userId, id -> loadRecommendations(id));
}
```

---

### 규칙 21 — 도메인별 서비스 분리 (한 클래스에 몰아넣지 않기)

**설명:** 여러 도메인의 로직을 한 서비스에 몰아넣지 않고, 각 도메인마다 서비스를 분리한다.

```java
// ❌ 나쁜 예시
@Service
public class CommonService {
    public void createBook(BookRequest request) { /* ... */ }
    public void login(AuthRequest request) { /* ... */ }
    public void sendChat(ChatRequest request) { /* ... */ }
}

// ✅ 좋은 예시
@Service
public class BookService { /* 도서 관련 로직 */ }

@Service
public class AuthService { /* 인증 관련 로직 */ }

@Service
public class ChatService { /* 채팅 관련 로직 */ }
```

---

### 규칙 22 — 기본 LAZY, 조회 전용은 fetch join/EntityGraph/DTO 활용

**설명:** 연관 관계는 기본 LAZY로 두고, 조회 전용 시 fetch join을 활용한다.

```java
@Entity
public class Book {
    @ManyToOne(fetch = FetchType.LAZY)
    private Author author;
}

@Query("SELECT b FROM Book b JOIN FETCH b.author WHERE b.id = :id")
Optional<Book> findBookWithAuthor(@Param("id") Long id);
```

---

### 규칙 23 — 엔티티를 응답으로 직렬화 금지(→ DTO 변환)

**설명:** DTO 변환을 거친 뒤 JSON 직렬화를 수행한다.

```java
@Getter
@Builder
public class BookResponse {
    private String title;
    private String author;

    public static BookResponse fromEntity(Book book) {
        return BookResponse.builder()
                .title(book.getTitle())
                .author(book.getAuthor().getName())
                .build();
    }
}
```

---

### 규칙 24 — Service 메서드 단위 @Transactional(readOnly 적용)

**설명:** 각 서비스 메서드에 명확히 트랜잭션을 지정하고, 조회 전용에는 `readOnly=true` 를 적용한다.

```java
@Service
@RequiredArgsConstructor
public class BookService {

    private final BookRepository bookRepository;

    @Transactional(readOnly = true)
    public BookResponse getBook(Long id) {
        Book book = bookRepository.findById(id)
                .orElseThrow(() -> new NotFoundException("존재하지 않는 책"));
        return BookResponse.fromEntity(book);
    }
}
```

---

### 규칙 26 — 요청/응답/도메인 DTO 분리, 요청 DTO는 @Valid

**설명:** 요청/응답/도메인 DTO를 명확하게 나누고, 요청 DTO에는 검증 어노테이션을 적용한다.

```java
@Getter
@NoArgsConstructor
public class AuthRequest {
    @NotBlank(message = "아이디는 필수 값이다")
    private String username;

    @NotBlank(message = "비밀번호는 필수 값이다")
    private String password;
}
```

---

### 규칙 30 — 무상태 서버, 확장 고려

**설명:** 세션 대신 JWT + Redis, 파일은 S3 등 외부 저장소 활용.

```java
// 무상태 인증 (Spring Security Filter)
public class JwtTokenFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String token = resolveToken(request);
        if (token != null && jwtTokenProvider.validateToken(token)) {
            Authentication auth = jwtTokenProvider.getAuthentication(token);
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        filterChain.doFilter(request, response);
    }
}
```

> **회고:** DungeonTalk에서 세션 기반 로그인으로 시작했는데, 멀티 서버 환경에서 세션 동기화 이슈가 생겼다. JWT + Redis 구조로 바꾼 뒤 확장성이 확보됐다.

---

### 규칙 32 — Access는 클라이언트, Refresh는 서버 저장(DB/Redis) + 회전

**설명:** 클라이언트는 AccessToken만 보관하고, RefreshToken은 서버에서 Redis/DB에 저장해 보안성을 강화한다.

```java
redisTemplate.opsForValue()
    .set("RT:" + member.getId(), refreshToken, 7, TimeUnit.DAYS);
```

> **회고:** Auth 프로젝트 초반에 RefreshToken을 클라이언트에 전달했더니 탈취 위험이 확인되어 Redis에 RefreshToken을 저장하고 토큰 회전을 적용하니 보안성이 훨씬 강화됐다.

---

### 규칙 33 — 환경 변수/Secret Manager 사용, @ConfigurationProperties 주입

**설명:** 비밀번호, 시크릿 키는 `.yml` 에 직접 쓰지 않고 환경 변수나 Secret Manager로 관리한다.

```java
@ConfigurationProperties(prefix = "jwt")
@Getter
@Setter
public class JwtProperties {
    private String secret;
    private long accessTokenValidity;
    private long refreshTokenValidity;
}
```

```yaml
jwt:
  secret: ${JWT_SECRET}
  access-token-validity: 3600
  refresh-token-validity: 604800
```

---

### 규칙 42 — Request/Response DTO 코딩 스타일 일관 유지

**설명:** DTO 변환 메서드명을 `fromEntity`, `toEntity` 처럼 통일하고, 요청/응답 DTO는 명확히 구분한다.

```java
@Getter
@NoArgsConstructor
public class BookRequest {
    private String title;
    private String author;

    public Book toEntity() {
        return Book.builder()
                .title(title)
                .author(author)
                .build();
    }
}

@Getter
@Builder
public class BookResponse {
    private String title;
    private String author;

    public static BookResponse fromEntity(Book book) {
        return BookResponse.builder()
                .title(book.getTitle())
                .author(book.getAuthor())
                .build();
    }
}
```

> **회고:** 프로젝트 초반엔 `convert` 같은 제각각의 메서드명을 쓰다보니 가독성이 떨어졌다. `fromEntity`, `toEntity`로 통일하니 팀원 간 코드 이해가 쉬워졌다.

---

### 규칙 43 — DTO에도 @Builder 적용, entity → dto 변환 적극 활용

**설명:** DTO 변환 시 @Builder를 활용해 가독성과 확장성을 높인다.

```java
@Getter
@Builder
public class MemberResponse {
    private String username;
    private String nickName;

    public static MemberResponse fromEntity(Member member) {
        return MemberResponse.builder()
                .username(member.getUsername())
                .nickName(member.getNickName())
                .build();
    }
}
```

---

### 규칙 44 — Service 계층은 인터페이스로 만들지 않음

**설명:** 불필요한 인터페이스 계층을 만들지 않고, 바로 구현 클래스를 사용한다.

```java
// ❌ 불필요한 인터페이스
public interface BookService {
    BookResponse getBook(Long id);
}
@Service
public class BookServiceImpl implements BookService { ... }

// ✅ 단일 클래스
@Service
@RequiredArgsConstructor
public class BookService {
    public BookResponse getBook(Long id) { /* ... */ }
}
```

---

### 규칙 47 — entity 작성 시 AccessLevel.PRIVATE/PROTECTED 활용

**설명:** 엔티티 기본 생성자는 `protected` 나 `private` 으로 막고, 빌더나 팩토리 메서드로만 생성한다.

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Member {
    @Id 
    @GeneratedValue
    private Long id;

    private String username;
    private String password;

    @Builder
    private Member(String username, String password) {
        this.username = username;
        this.password = password;
    }
}
```

---

### 규칙 48 — JPA dirty-checking 활용

**설명:** setter 대신 엔티티 값 변경 시 JPA 더티체킹으로 update 처리.

```java
@Transactional
public void updateNickName(Long memberId, String newNickName) {
    Member member = memberRepository.findById(memberId)
            .orElseThrow(() -> new NotFoundException("존재하지 않는 회원"));
    member.changeNickName(newNickName); // 엔티티 내부 메서드
}
```

> **회고:** `save()` 호출이 필요 없고 코드가 간결해졌다.

---

### 규칙 50 — BaseTime 적용

**설명:** 엔티티 공통 시간 필드를 `@MappedSuperclass` 로 관리.

```java
@Getter
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseTime {
    @CreatedDate
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;
}
```

---

### 규칙 51 — BaseTime이나 엔티티에 protected 지양

**설명:** 생성자 접근 제어 시 불필요하게 `protected`를 남발하지 않는다.

```java
@NoArgsConstructor(access = AccessLevel.PROTECTED) // ✅ 필요한 경우만
public class Book { /* ... */ }
```

---

### 규칙 56 — @AuthenticationPrincipal 활용

**설명:** 로그인 사용자 정보를 컨트롤러에서 편리하게 주입받는다.

```java
@GetMapping("/me")
public ApiResponse<MemberResponse> getCurrentUser(@AuthenticationPrincipal CustomUserDetail userDetail) {
    return ApiResponse.ok(MemberResponse.fromEntity(userDetail.getMember()));
}
```

> **회고:** Auth 프로젝트에서 매번 `SecurityContextHolder`에서 꺼내 쓰던 코드를 없애고, `@AuthenticationPrincipal`을 쓰니 컨트롤러가 간결해졌다.

---

### 규칙 57 — CustomUserDetail 적용

**설명:** `UserDetails`를 커스터마이징하여 엔티티와 연동.

```java
@Getter
public class CustomUserDetail implements UserDetails {
    private final Member member;

    public CustomUserDetail(Member member) {
        this.member = member;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of(new SimpleGrantedAuthority("ROLE_USER"));
    }

    @Override
    public String getPassword() { 
        return member.getPassword(); 
    }

    @Override
    public String getUsername() { 
        return member.getUsername(); 
    }
}
```
