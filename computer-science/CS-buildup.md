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

