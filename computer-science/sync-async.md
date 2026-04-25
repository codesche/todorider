# 동기와 비동기 — 개념부터 실무 적용까지

## 1. 핵심 개념 정리

### 동기 (Synchronous)

작업을 순서대로 실행하고, 이전 작업이 완료될 때까지 다음 작업을 시작하지 않는 방식.

- 요청 → 응답 대기 → 다음 작업 진행
- 코드의 실행 흐름이 위에서 아래로 직관적으로 흐름
- 결과가 보장된 상태에서 다음 단계로 넘어감

```
[작업 A] ──완료──▶ [작업 B] ──완료──▶ [작업 C]
```

### 비동기 (Asynchronous)

작업을 요청한 후 응답을 기다리지 않고 다음 작업을 바로 실행하는 방식.

- 요청 → 다음 작업 바로 진행 → 응답이 오면 콜백/핸들러 실행
- 주로 I/O 바운드 작업(네트워크, DB, 파일)에서 효과적
- 대기 시간 동안 CPU가 다른 일을 처리할 수 있음

```
[작업 A 요청] ──▶ [작업 B 요청] ──▶ [작업 C 요청]
      ↓                  ↓                  ↓
  (응답 대기 중)      (응답 대기 중)      (응답 대기 중)
      ↓
  [A 완료 처리]
```

---

## 2. Blocking vs Non-Blocking

비동기/동기와 자주 혼용되지만 다른 개념.

| 구분 | 설명 |
| --- | --- |
| **Blocking** | 호출한 함수가 완료될 때까지 제어권을 돌려주지 않음 |
| **Non-Blocking** | 호출 즉시 제어권을 돌려줌 (결과는 나중에) |
| **Synchronous** | 결과를 기다리는 주체가 직접 확인 |
| **Asynchronous** | 결과 처리를 콜백/이벤트에 위임 |

4가지 조합이 가능하고, 실무에서는 **Non-Blocking + Async** 조합이 고성능의 핵심.

---

## 3. 비동기 처리 방식의 진화

### 3-1. Callback

가장 기초적인 비동기 처리 방식. 작업 완료 후 실행할 함수를 미리 넘겨줌.

```javascript
// JavaScript 예시
fs.readFile('data.txt', 'utf8', (err, data) => {
  if (err) {
    console.error('파일 읽기 실패:', err);
    return;
  }
  console.log('파일 내용:', data);
});
```

**문제점: Callback Hell**

```javascript
// 중첩이 깊어지면 가독성이 급격히 나빠짐
getUser(userId, (user) => {
  getOrders(user.id, (orders) => {
    getProduct(orders[0].productId, (product) => {
      getReview(product.id, (review) => {
        // 이 깊이에서 에러 처리까지...
      });
    });
  });
});
```

### 3-2. Promise

Callback Hell을 해결하기 위해 등장. 비동기 작업의 결과를 나타내는 객체.

- **Pending**: 아직 완료되지 않은 상태
- **Fulfilled**: 성공적으로 완료
- **Rejected**: 실패

```javascript
// Promise 체이닝
fetch('/api/user')
  .then(res => res.json())
  .then(user => fetch(`/api/orders/${user.id}`))
  .then(res => res.json())
  .then(orders => console.log(orders))
  .catch(err => console.error(err));
```

### 3-3. async/await

Promise를 동기 코드처럼 작성할 수 있게 해주는 문법적 설탕(syntactic sugar).

```javascript
async function fetchUserOrders(userId) {
  try {
    const userRes = await fetch(`/api/user/${userId}`);
    const user = await userRes.json();

    const ordersRes = await fetch(`/api/orders/${user.id}`);
    const orders = await ordersRes.json();

    return orders;
  } catch (err) {
    console.error('주문 조회 실패:', err);
    throw err;
  }
}
```

훨씬 읽기 쉽고, 에러 처리도 일반 try/catch로 가능.

---

## 4. 언어별 비동기 처리

### Java — CompletableFuture

```java
CompletableFuture<User> userFuture = CompletableFuture
    .supplyAsync(() -> userRepository.findById(userId))
    .thenApply(user -> enrichUser(user))
    .exceptionally(ex -> {
        log.error("사용자 조회 실패", ex);
        return null;
    });

// 여러 Future 병렬 실행
CompletableFuture<Void> allOf = CompletableFuture.allOf(
    CompletableFuture.runAsync(() -> task1()),
    CompletableFuture.runAsync(() -> task2()),
    CompletableFuture.runAsync(() -> task3())
);
allOf.join();
```

### Kotlin — Coroutine

```kotlin
suspend fun fetchUserData(userId: String): UserData {
    return coroutineScope {
        val user = async { userService.getUser(userId) }
        val orders = async { orderService.getOrders(userId) }

        // 두 요청을 병렬로 실행
        UserData(
            user = user.await(),
            orders = orders.await()
        )
    }
}

viewModelScope.launch {
    try {
        val data = fetchUserData("user-123")
        updateUI(data)
    } catch (e: Exception) {
        showError(e.message)
    }
}
```

### Python — asyncio

```python
import asyncio
import aiohttp

async def fetch_data(session, url):
    async with session.get(url) as response:
        return await response.json()

async def main():
    async with aiohttp.ClientSession() as session:
        tasks = [
            fetch_data(session, 'https://api.example.com/users'),
            fetch_data(session, 'https://api.example.com/orders'),
            fetch_data(session, 'https://api.example.com/products'),
        ]
        results = await asyncio.gather(*tasks)
        return results

asyncio.run(main())
```

---

## 5. 실무 적용 패턴

### 패턴 1: 병렬 실행으로 성능 개선

여러 독립적인 비동기 작업은 순차 실행하지 말고 병렬로 실행.

```javascript
// 나쁜 예 — 순차 실행 (시간: A + B + C)
const userProfile = await getUserProfile(id);
const userOrders = await getUserOrders(id);
const userReviews = await getUserReviews(id);

// 좋은 예 — 병렬 실행 (시간: max(A, B, C))
const [userProfile, userOrders, userReviews] = await Promise.all([
  getUserProfile(id),
  getUserOrders(id),
  getUserReviews(id),
]);
```

### 패턴 2: 타임아웃 처리

비동기 작업이 무한정 대기하지 않도록 반드시 타임아웃을 설정.

```javascript
function withTimeout(promise, ms) {
  const timeout = new Promise((_, reject) =>
    setTimeout(() => reject(new Error(`Timeout after ${ms}ms`)), ms)
  );
  return Promise.race([promise, timeout]);
}

const data = await withTimeout(fetchData(), 3000);
```

### 패턴 3: 재시도 (Retry) 로직

네트워크 오류 등 일시적 장애에 대한 재시도 처리.

```javascript
async function fetchWithRetry(url, maxRetries = 3, delay = 1000) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fetch(url).then(r => r.json());
    } catch (err) {
      if (attempt === maxRetries) throw err;
      console.warn(`시도 ${attempt} 실패, ${delay}ms 후 재시도...`);
      await new Promise(resolve => setTimeout(resolve, delay * attempt));
    }
  }
}
```

### 패턴 4: 동시 실행 수 제한 (Concurrency Control)

너무 많은 비동기 작업을 동시에 실행하면 서버/DB에 과부하.

```javascript
async function processInBatches(items, batchSize, processor) {
  const results = [];
  for (let i = 0; i < items.length; i += batchSize) {
    const batch = items.slice(i, i + batchSize);
    const batchResults = await Promise.all(batch.map(processor));
    results.push(...batchResults);
  }
  return results;
}

// 100개 항목을 10개씩 나눠서 처리
const results = await processInBatches(items, 10, processItem);
```

### 패턴 5: Event-Driven Architecture

백엔드에서 비동기를 극대화하는 구조. 요청자와 처리자를 분리.

```
[Producer] → [Message Queue (Kafka/RabbitMQ)] → [Consumer]
    ↑                                                ↓
  요청 즉시 응답                              비동기로 처리
```

```java
@Service
public class OrderService {

    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void placeOrder(Order order) {
        orderRepository.save(order);

        // 이벤트 발행 (비동기) — 결제, 재고 차감, 알림 등은 별도 컨슈머가 처리
        kafkaTemplate.send("order-placed", new OrderEvent(order));
    }
}

@KafkaListener(topics = "order-placed")
public void handleOrderPlaced(OrderEvent event) {
    inventoryService.deductStock(event.getOrderId());
}
```

---

## 6. 실무에서 주의할 점

### 에러 처리를 절대 빠뜨리지 말 것

비동기 코드에서 에러를 잡지 않으면 조용히 실패하고 디버깅이 매우 어려워짐.

```javascript
// 나쁜 예 — 에러 처리 없음
const data = await fetchData();

// 좋은 예 — 항상 try/catch 또는 .catch()
try {
  const data = await fetchData();
} catch (err) {
  logger.error('데이터 조회 실패:', err);
  // 적절한 fallback 처리
}
```

### async/await 남발 주의

`await`을 걸면 해당 지점에서 실행이 멈춤. 독립적인 작업에 `await`을 남발하면 오히려 순차 실행이 됨.

```javascript
// 나쁜 예 — 독립적인 작업인데 순차 실행
const a = await fetchA();
const b = await fetchB(); // A가 끝날 때까지 B 시작 안 함

// 좋은 예 — 동시에 시작하고 둘 다 끝나면 처리
const [a, b] = await Promise.all([fetchA(), fetchB()]);
```

### 상태 관리 (언마운트/취소 처리)

비동기 작업 중간에 컴포넌트가 언마운트되거나 상태가 변경될 수 있음. 클린업 처리 필수.

```javascript
// React 예시 — 비동기 중 언마운트 처리
useEffect(() => {
  let isMounted = true;

  async function loadData() {
    const data = await fetchData();
    if (isMounted) {
      setState(data);
    }
  }

  loadData();

  return () => {
    isMounted = false;
  };
}, []);
```

---

## 7. 언제 동기, 언제 비동기?

| 상황 | 권장 방식 | 이유 |
| --- | --- | --- |
| CPU 집약 작업 (계산, 암호화) | 동기 or 별도 스레드 | I/O 대기가 없음 |
| 네트워크 요청 (API 호출) | 비동기 | 대기 시간 동안 다른 작업 가능 |
| DB 쿼리 | 비동기 | I/O 바운드 작업 |
| 파일 읽기/쓰기 | 비동기 | I/O 바운드 작업 |
| 순서가 중요한 작업 | 동기 (await 체인) | 실행 순서 보장 필요 |
| 독립적인 여러 요청 | 비동기 병렬 (Promise.all) | 최대 성능 |

---

## 정리

비동기는 "기다리는 시간을 낭비하지 않는다"는 철학.

- 단순한 순차 흐름 → **동기**
- 네트워크/DB/파일 I/O → **비동기 기본**
- 독립적인 여러 작업 → **Promise.all / gather / allOf**
- 대규모 시스템 → **Event-Driven (메시지 큐)**

제대로 된 비동기 코드는 에러 처리, 타임아웃, 동시성 제어를 모두 포함해야 진짜 실무 수준.
