# 운영체제의 I/O 모델 — Blocking, Non-Blocking, Async

## 왜 이 글을 쓰는가

이전 글에서 Spring MVC의 문제를 다뤘다.
I/O 대기 중에 스레드가 점유 상태로 묶여 있다는 것.
그러면 "I/O 대기"라는 게 정확히 뭐이고, 대안은 뭔가.

Blocking I/O, Non-Blocking I/O, Synchronous, Asynchronous.
이 네 가지 개념은 종종 혼동되어 쓰인다.
이 글에서 정확히 구분하고, Spring WebFlux와 Kafka가 왜 이 개념들에 기반하는지 연결한다.

---

## 1. I/O란 무엇인가

I/O(Input/Output)는 CPU 외부와의 데이터 접근을 말한다.

- 파일 읽기/쓰기
- 네트워크 송수신
- DB 쿼리
- 터미널 입력

CPU는 매우 빠르다. I/O는 매우 느리다.
디스크 접근은 CPU보다 10만 배, 네트워크는 더 느리다.

I/O가 일어나는 동안 스레드를 어떻게 다루느냐의 차이가 모델 전체의 성능을 갈른다.

---

## 2. Blocking I/O

### 동작 방식

```
스레드 A
  |
  | read() 호출
  |-------> [OS 커널: 디스크/네트워크 I/O 시작]
  |                                              |
  | (BLOCKED — 스레드 대기)                      |
  |                                              |
  |<------- 데이터 반환 -------------------------+
  |
  | 이어서 실행
```

- **스레드가 I/O 완료될 때까지 순수하게 대기**
- I/O 동안 CPU는 다른 작업을 할 수 있지만, 스레드 자체는 어디도 못 감

### 문제점

```
[Spring MVC 시나리오]

Thread 1: [---요청 처리---][DB 쿼리 대기.............][---응답---]
                           ^ 이 시간 동안 Thread 1은 묶여 있음

Thread 2: [---요청 처리---][API 호출 대기.............][---응답---]
                           ^ Thread 2도 묶여 있음

Thread N: [---요청---][...................................................] 대기
```

스레드가 200개라면, 200개의 요청이 동시에 DB를 기다리는 순간 풀이 곧 차단된다.

---

## 3. Non-Blocking I/O

### 동작 방식

```
스레드 A
  |
  | read() 호출
  |-------> [OS 커널: I/O 시작]
  |<------- 즉시 리턴 (데이터 없으면 EAGAIN/EWOULDBLOCK)
  |
  | 다른 작업 계속...
  |
  | (매핑 구조를 통해 I/O 완료 시점 확인)
  |
  | 데이터 준비되면 실제 읽기
```

- **I/O 호출이 즉시 리턴**. 데이터가 없으면 없다고 바로 알려준다.
- 스레드는 블록되지 않고 다른 일을 할 수 있다.
- 단, 언제 데이터가 준비되었는지 알아야 하므로 **I/O 다중화(Multiplexing)** 와 주로 함께 쓴다.

### I/O 다중화: select / poll / epoll

```
[프로그램]
  |
  | epoll 등록: "fd1, fd2, fd3 준비되면 알려줘"
  |-------> [OS 커널]
  |
  | 다른 작업...
  |
  |<------- "fd2 준비 완료" (이벤트 도착)
  |
  | fd2 데이터 읽기
```

- **select/poll**: 등록된 디스크립터 전체를 순회하며 확인. O(n)
- **epoll** (Linux): 준비된 것만 이벤트로 알려줌. O(1). Nginx, Node.js가 사용

---

## 4. Synchronous vs Asynchronous

Blocking/Non-Blocking은 **I/O 호출 시 스레드가 대기하느냐**의 문제다.
Synchronous/Asynchronous는 **결과를 누가 처리하느냐**의 문제다.

| 구분 | 의미 |
|---|---|
| **Synchronous** | 호출한 스레드가 결과를 직접 확인하고 이어서 처리 |
| **Asynchronous** | 결과를 콜백/이벤트로 알려주면 다른 스레드가 이어서 처리 |

### 네 가지 조합

#### Synchronous Blocking (= 일반 Blocking I/O)

```java
// 프로그래머가 주로 접하는 일반적인 형태
byte[] data = file.read();  // 완료될 때까지 대기
process(data);              // 이어서 실행
```

호출한 스레드가 I/O가 끝날 때까지 대기하고, 직접 결과를 처리한다.

#### Synchronous Non-Blocking

```java
// 쿼리하는 폴링 방식 — 실무에서 거의 안 쓰임
while (true) {
    Result r = socket.readNonBlocking();
    if (r.isReady()) {
        process(r.data());
        break;
    }
    // 데이터 없으면 계속 폴링 ← CPU 낭비
}
```

I/O 호출은 즉시 리턴되지만, 완료 확인은 직접 한다. CPU를 낭비하여 실무에서 드물다.

#### Asynchronous Blocking

```java
// select/poll 유사 면선 — 이론적 조합
// epoll이 없는 상황에서만 해당 (Linux 2.5 이전)
fd = select(fds);           // 변화 발생할 때까지 대기 (blocking)
process(fd.data());         // 큐에서 데이터 가져옴 (async fetch)
```

호출 자체는 블록되지만, 데이터를 가져오는 경로가 비동기적인 코너케이스다.
실무에서 사용하는 경우는 없다.

#### Asynchronous Non-Blocking (= 진정한 비동기 I/O)

```java
// Java NIO2 (AsynchronousFileChannel) 예시
AsynchronousFileChannel channel = AsynchronousFileChannel.open(path);
ByteBuffer buffer = ByteBuffer.allocate(1024);

channel.read(buffer, 0, null, new CompletionHandler<>() {
    @Override
    public void completed(Integer result, Object attachment) {
        // I/O 완료 시 다른 스레드가 콜백 실행
        process(buffer);
    }
});

// 여기선 즉시 리턴 — 대기 없음
// 호출한 스레드는 다른 일을 함
```

I/O 호출이 즉시 리턴되고, 완료되면 OS가 콜백으로 알려준다.
**호출한 스레드가 결과를 처리하지 않아도 된다.**

---

## 5. 정리하면

| 모델 | 대기 | 결과 처리 | 스레드 점유 |
|---|---|---|---|
| Sync Blocking | O | 직접 | O |
| Sync Non-Blocking | X | 직접 | X (but 폴링) |
| Async Blocking | O | 콜백 | O |
| Async Non-Blocking | X | 콜백 | X |

**성능에서 중요한 것은 주로 두 가지다.**
1. I/O 동안 스레드가 블록되는가 (Blocking vs Non-Blocking)
2. 완료 콜백으로 처리하는가 (Async)

---

## 6. 실무 연결

### Spring WebFlux와 Reactor

Spring WebFlux는 **Async Non-Blocking** 모델로 동작한다.

```java
// WebFlux 코드 예시
@GetMapping("/users/{id}")
public Mono<User> getUser(@PathVariable Long id) {
    return userRepository.findById(id)  // Non-Blocking DB 호출
        .map(user -> enrichUser(user));  // 콜백으로 이어 처리
}
```

- DB 쿼리가 나가는 동안 스레드가 다른 요청을 처리한다.
- 쿼리가 완료되면 OS가 이벤트로 알려주고, 콜백이 실행된다.
- 소수의 스레드로 많은 요청을 병렬 처리할 수 있다.

**주의할 점**: WebFlux를 쓰면서 JDBC(블로킹 드라이버)를 그대로 쓰면 이벤트 루프 스레드가 블록되어
스레드풀 전체가 멈출 수 있다. R2DBC를 쓰는 이유다.

### Node.js의 이벤트 루프

```
[단일 스레드]
  |
  |요청 1: DB 쿼리 등록 → epoll
  |요청 2: 파일 읽기 등록 → epoll
  |요청 3: 외부 API 등록 → epoll
  |
  | [루프]
  |   DB 쿼리 완료 콜백 실행
  |   파일 읽기 완료 콜백 실행
  |   ...
```

Node.js는 단일 스레드에 epoll 기반 이벤트 루프로 I/O 집약적 서비스에 적합하다.
단, CPU 연산이 많은 작업에는 절대적으로 부적합하다.

### Kafka 컨슈머 비동기 처리

```java
@KafkaListener(topics = "orders")
public void handleOrder(String message) {
    // 메시지가 도착하면 콜백 실행
    // 컨슈머는 메시지가 올 때까지 블록되지 않음
    processOrder(message);
}
```

Kafka 컨슈머는 Async Non-Blocking에 가깝다.
메시지가 도착하면 이벤트로 알려주고, 콜백이 처리한다.

---

## 7. 언제 어떤 모델을 선택하는가

| 상황 | 권장 모델 |
|---|---|
| I/O가 적고 CPU 작업이 많은 서비스 | Sync Blocking (Spring MVC) |
| I/O가 많고 동시 요청이 많은 서비스 | Async Non-Blocking (WebFlux) |
| 메시지 기반 마이크로서비스 간 통신 | Async (Kafka, RabbitMQ) |
| 실시간 스트리밍 | Async Non-Blocking (WebSocket + WebFlux) |

---

## 정리

| 개념 | 핵심 |
|---|---|
| Blocking I/O | I/O 동안 스레드 진단. 단순하지만 스레드 자원 낭비 |
| Non-Blocking I/O | I/O 호출 즉시 리턴. epoll로 완료 시점 감지 |
| Synchronous | 호출한 스레드가 직접 결과 확인/처리 |
| Asynchronous | 콜백/이벤트로 결과 통보. 처리는 다른 흐름에서 |
| Spring MVC | Sync Blocking. 스레드풀 크기가 동시 처리량 상한 |
| Spring WebFlux | Async Non-Blocking. 소수 스레드로 많은 I/O 병렬 처리 |
| Kafka Consumer | Async 이벤트 기반. 메시지 도착 시 콜백 |

1단계 네트워크/OS 기초를 여기서 마친다.
2단계 DB 심화에서는 이 I/O 모델 위에서 DB가 데이터를 어떻게 읽는지를 다룬다.
