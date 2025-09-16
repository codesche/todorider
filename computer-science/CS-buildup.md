# CS 기초 정리 빌드업


## 목차

1. 운영체제 (OS)  
   1.1. 프로세스 vs 스레드  
   1.2. 동기 vs 비동기, 블로킹 vs 논블로킹  
   1.3. 메모리 구조 (JVM 포함)

2. 네트워크  
   2.1. TCP vs UDP  
   2.2. HTTP vs HTTPS  
   2.3. REST API 원칙  
   2.4. CORS

3. 데이터베이스 (DB)  
   3.1. 인덱스  
   3.2. 트랜잭션 & ACID  
   3.3. Isolation Level  
   3.4. 조인 (JOIN)

4. 자료구조 & 알고리즘  
   4.1. 자료구조  
   4.2. 알고리즘 복잡도

5. Java & Spring  
   5.1. JVM & GC  
   5.2. Spring Bean Lifecycle  
   5.3. Spring AOP  
   5.4. Spring Security

6. 시스템 설계  
   6.1. 확장성 고려  
   6.2. 장애 대응  
   6.3. 트래픽 분산

---

## 1. 운영체제 (OS)

### 1.1. 프로세스 vs 스레드

- **프로세스**: 실행 중인 프로그램, 독립된 메모리 공간  
- **스레드**: 프로세스 내에서 실행되는 흐름 (코드/데이터/힙 공유, 스택은 개별)  
- **면접 포인트**:  
  - 스레드는 자원을 공유하므로 컨텍스트 스위칭 비용이 낮지만 동기화(synchronization) 이슈가 발생할 수 있음

### 1.2. 동기 vs 비동기, 블로킹 vs 논블로킹

- **동기 (Synchronous)**: 요청 후 결과를 받을 때까지 기다림  
- **비동기 (Asynchronous)**: 요청 후 결과를 기다리지 않고 다른 작업 수행 가능  
- **블로킹 (Blocking)**: 제어권을 반환하지 않고 기다리는 상태  
- **논블로킹 (Non‐blocking)**: 제어권을 반환함  
- **면접 포인트**:  
  - “동기-비동기는 작업 순서의 개념, 블로킹-논블로킹은 제어권 반환의 개념.”

### 1.3. 메모리 구조 (JVM 포함)

- **Stack**: 메서드 호출, 지역 변수  
- **Heap**: 객체 / 인스턴스  
- **Method Area (Class 영역)**: 클래스 메타데이터  
- **GC (Garbage Collection)**  
  - Heap 영역의 불필요한 객체 자동 수거  
- **면접 포인트**:  
  - “Stop-the-world, Minor GC / Full GC 차이 설명 가능해야 함.”

---

## 2. 네트워크

### 2.1. TCP vs UDP

- **TCP**: 연결 지향, 신뢰성 보장 (3-way handshake, 흐름제어(flow control), 혼잡제어(congestion control))  
- **UDP**: 비연결 지향, 속도가 빠르지만 신뢰성은 보장하지 않음 (예: 스트리밍, 게임)  
- **면접 포인트**:  
  - 서비스 특성에 따라 선택: 예를 들어 채팅은 TCP, 영상 스트리밍은 UDP

### 2.2. HTTP vs HTTPS

- **HTTP**: 평문 통신, 보안 취약  
- **HTTPS**: TLS/SSL 기반 암호화 통신  
- **면접 포인트**:  
  - 대칭키/비대칭키를 혼합하여 성능과 보안을 확보하는 구조 설명 가능해야 함

### 2.3. REST API 원칙

- Stateless (무상태성)  
- 클라이언트-서버 분리  
- 계층화 구조(layers)  
- 캐시 가능성(cacheability)  
- **면접 포인트**:  
  - HTTP 메서드(GET, POST, PUT, DELETE)를 리소스에 맞게 사용하는 것이 중요

### 2.4. CORS

- 브라우저 보안 정책: 다른 출처(origin)의 리소스 요청 제한  
- 해결 방법: 서버에서 `Access-Control-Allow-Origin` 헤더 설정

---

## 3. 데이터베이스 (DB)

### 3.1. 인덱스

- 주요한 구조: 보통 **B+ Tree**  
- 장점: 조회 속도 향상  
- 단점: 쓰기 성능 저하, 저장 공간 증가  
- **면접 포인트**:  
  - 조회가 많은 컬럼에는 인덱스가 적합하나, 쓰기가 잦은 경우 주의 필요

### 3.2. 트랜잭션 & ACID

- **Atomicity**: 원자성  
- **Consistency**: 일관성  
- **Isolation**: 격리성  
- **Durability**: 영속성  
- **면접 포인트**:  
  - 실무에서는 Isolation Level 조정으로 동시성(concurrency) vs 성능 trade-off 관리

### 3.3. Isolation Level

- **READ UNCOMMITTED**: Dirty Read 허용  
- **READ COMMITTED**: Dirty Read 방지 (예: Oracle 기본)  
- **REPEATABLE READ**: Non-Repeatable Read 방지 (예: MySQL InnoDB 기본)  
- **SERIALIZABLE**: 완전한 격리 수준, 성능 저하 큼  
- **면접 포인트**:  
  - 현업에서는 보통 READ COMMITTED 또는 REPEATABLE READ 많이 사용

### 3.4. 조인 (JOIN)

- 종류: Inner Join, Left Join, Right Join, Full Outer Join  
- 실무에서는 주로 Inner, Left Join 사용  
- **면접 포인트**:  
  - 데이터 양이 많을 때 조인 순서, 인덱스 설계 고려 필요

---

## 4. 자료구조 & 알고리즘

### 4.1. 자료구조

- Array vs LinkedList: 탐색(search)은 Array, 삽입/삭제(insertion/deletion)는 LinkedList 유리  
- HashMap: 평균 시간 복잡도 O(1), 충돌(collision) 발생 시 체이닝(chaining) 또는 오픈 어드레싱(open addressing) 사용  
- Heap: 우선순위 큐(Priority Queue) 구현  
- Tree / Binary Search Tree: 탐색, 정렬 등에 활용

### 4.2. 알고리즘 복잡도

- Big-O 표기법: O(1), O(log N), O(N), O(N log N), O(N²) 등  
- 면접에서 자주 나오는 질문: “이 로직의 시간 복잡도는?”

---

## 5. Java & Spring

### 5.1. JVM & GC

- JVM 메모리 구조: Stack, Heap, Metaspace  
- GC 종류: Minor GC (Young 영역), Major / Full GC (Old 영역)  
- **면접 포인트**:  
  - Stop-the-world 현상을 어떻게 줄일지

### 5.2. Spring Bean Lifecycle

- 객체 생성 → 의존성 주입(Dependency Injection) → 초기화(Initialization) → 소멸(Destruction)  
- Bean 스코프: Singleton, Prototype 등

### 5.3. Spring AOP

- 횡단 관심사(cross-cutting concerns) 분리: 로깅, 트랜잭션 등  
- Proxy 기반 동작

### 5.4. Spring Security

- 필터 체인 구조(Filter Chain)  
- JWT 인증 구현 시 `OncePerRequestFilter` 활용 가능

---

## 6. 시스템 설계

### 6.1. 확장성 고려

- Scale-up vs Scale-out  
- 캐시(Cache) 사용 (예: Redis)  
- 메시지 큐(Message Queue): Kafka, RabbitMQ 등

### 6.2. 장애 대응

- DB 복제(Replication), 페일오버(Failover)  
- 무중단 배포(Zero Downtime Deployment): Blue-Green, Canary 배포 방식

### 6.3. 트래픽 분산

- 로드 밸런서 (L4 / L7)  
- CDN 활용

---

# 📌 CS 기초 면접 예상 질문 & 모범 답변

## 1. 운영체제 (OS)

**Q1. 프로세스와 스레드의 차이를 설명해주세요.**  
👉 **A.** 프로세스는 실행 중인 프로그램으로 독립적인 메모리 공간을 가지고 있습니다.  
스레드는 프로세스 내에서 실행되는 작업 단위로, 프로세스의 메모리를 공유합니다.  
따라서 스레드는 컨텍스트 스위칭 비용이 적고 효율적이지만, 공유 자원으로 인해 동기화 문제가 발생할 수 있습니다.  

**Q2. 동기/비동기와 블로킹/논블로킹의 차이를 설명해주세요.**  
👉 **A.** 동기/비동기는 **작업의 순서** 개념이고, 블로킹/논블로킹은 **제어권 반환 여부** 개념입니다.  
예를 들어 동기-블로킹은 결과가 올 때까지 기다리는 것이고, 비동기-논블로킹은 요청만 보내고 다른 작업을 진행하다가 결과를 콜백으로 받습니다.  

---

## 2. 네트워크

**Q3. TCP와 UDP의 차이는 무엇인가요?**  
👉 **A.** TCP는 연결 지향적이고 신뢰성을 보장하기 때문에 채팅, 파일 전송 등에 사용됩니다.  
UDP는 비연결 지향적이고 빠르지만 신뢰성이 낮아 스트리밍이나 온라인 게임 같은 서비스에 적합합니다.  

**Q4. HTTP와 HTTPS의 차이를 설명해주세요.**  
👉 **A.** HTTP는 평문 통신이라 보안에 취약합니다.  
HTTPS는 TLS/SSL 암호화를 사용하며, 비대칭키로 세션 키를 교환한 후 대칭키로 데이터를 빠르게 암호화합니다.  

**Q5. CORS가 무엇이고, 어떻게 해결할 수 있나요?**  
👉 **A.** CORS는 브라우저의 보안 정책으로, 다른 출처의 리소스 요청을 제한합니다.  
해결 방법은 서버에서 `Access-Control-Allow-Origin` 헤더를 설정하여 허용된 도메인을 명시하는 것입니다.  

---

## 3. 데이터베이스 (DB)

**Q6. 인덱스의 장단점을 설명해주세요.**  
👉 **A.** 인덱스는 주로 B+ Tree 구조로, 조회 속도를 빠르게 하지만 쓰기 성능은 저하되고 저장 공간이 추가로 필요합니다.  
따라서 조회가 많은 컬럼에는 인덱스를 두고, 잦은 쓰기 작업에는 신중하게 사용합니다.  

**Q7. 트랜잭션의 ACID 특성을 설명해주세요.**  
👉 **A.** 
- 원자성(Atomicity): 모두 성공하거나 모두 실패.  
- 일관성(Consistency): 데이터 일관성 유지.  
- 격리성(Isolation): 트랜잭션 간 간섭 차단.  
- 영속성(Durability): 커밋된 결과는 보존.  

**Q8. Isolation Level에 따른 차이를 설명해주세요.**  
👉 **A.**  
- Read Uncommitted: Dirty Read 발생 가능.  
- Read Committed: Dirty Read 방지. (Oracle 기본값)  
- Repeatable Read: Non-Repeatable Read 방지. (MySQL InnoDB 기본값)  
- Serializable: 완전 격리, 성능 저하 큼.  

---

## 4. 자료구조 & 알고리즘

**Q9. Array와 LinkedList의 차이는 무엇인가요?**  
👉 **A.** Array는 인덱스를 통한 빠른 탐색이 가능하지만, 삽입/삭제가 비효율적입니다.  
LinkedList는 삽입/삭제가 빠르지만 탐색은 순차 접근만 가능해 느립니다.  

**Q10. HashMap의 시간복잡도는 어떻게 되나요?**  
👉 **A.** 평균적으로 O(1)입니다. 하지만 충돌이 많으면 체이닝이나 오픈 어드레싱 방식 때문에 최악의 경우 O(N)까지 될 수 있습니다.  

---

## 5. Java & Spring

**Q11. JVM 메모리 구조를 설명해주세요.**  
👉 **A.** JVM은 메서드 영역(클래스 메타데이터), 스택(지역 변수/메서드 호출), 힙(객체 저장)으로 나눠집니다.  
GC는 힙 영역의 불필요한 객체를 제거하며, Young 영역에서 Minor GC, Old 영역에서 Full GC가 발생합니다.  

**Q12. Spring Bean의 라이프사이클을 설명해주세요.**  
👉 **A.** 객체 생성 → 의존성 주입(DI) → 초기화 → 소멸 순서로 진행됩니다.  
기본 스코프는 Singleton이며, Prototype 스코프도 설정할 수 있습니다.  

**Q13. Spring AOP가 무엇인지 설명해주세요.**  
👉 **A.** AOP는 로깅, 트랜잭션 관리 같은 횡단 관심사를 비즈니스 로직과 분리하는 기술입니다.  
Spring에서는 Proxy 기반으로 동작합니다.  

**Q14. JWT 인증을 Spring Security에서 어떻게 구현할 수 있나요?**  
👉 **A.** OncePerRequestFilter를 확장한 JwtTokenFilter를 구현하여, 매 요청마다 토큰을 검증하고 사용자 정보를 SecurityContextHolder에 등록하는 방식으로 구현할 수 있습니다.  

---

## 6. 시스템 설계

**Q15. 트래픽이 급격히 증가했을 때 어떻게 대응하시겠습니까?**  
👉 **A.** Scale-out 방식으로 서버를 확장하고, 로드밸런서를 통해 트래픽을 분산합니다.  
또한 Redis 캐시를 적용해 DB 부하를 줄이고, 메시지 큐를 활용해 비동기 처리로 응답 속도를 개선할 수 있습니다.  

**Q16. 장애 대응 방안을 말씀해주세요.**  
👉 **A.** DB Replication을 활용해 Failover 환경을 구축하고, Blue-Green 또는 Canary 배포로 무중단 배포를 실현할 수 있습니다. 