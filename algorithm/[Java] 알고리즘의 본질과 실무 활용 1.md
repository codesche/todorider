# [Java] 알고리즘의 본질과 실무 활용 1

# Java 알고리즘의 본질과 실무 활용

> **초급 → 중급 → 고급** 순서로 정리
> 

## 알고리즘, 왜 알아야 하는 걸까?

대부분의 실무는 알고리즘을 직접 구현하기보다,

프레임워크(Spring, Hibernate 등)와 라이브러리 위에서 진행된다.

하지만 **효율적인 데이터 처리, 메모리 관리, 캐싱 전략, 검색 최적화** 등은

결국 알고리즘적 사고에서 비롯된다.

- API 응답 속도를 2초 → 200ms로 단축시킨 것도 결국 **탐색 알고리즘 최적화**
- Redis나 Kafka의 내부 로직 역시 **Queue + HashMap + Heap** 조합 구조
- QueryDSL 튜닝이나 MongoDB 인덱싱도 결국 **정렬, 탐색의 시간복잡도** 이해와 직결

⇒ 즉, “실무형 알고리즘”은 면접용이 아닌 **운영 시스템의 안정성과 성능**을 만드는 기술이다.

## 초급: 정렬(Sorting) 알고리즘과 실무적 의미

### 개념

정렬은 데이터를 순서대로 배치하는 과정이며, 대부분의 서비스는 정렬을 전제로 동작한다.

| 알고리즘 | 시간복잡도 | 공간복잡도 | 안정성 | 비고 |
| --- | --- | --- | --- | --- |
| Bubble Sort | O(n²) | O(1) | ✅ | 가장 단순하지만 비효율 |
| Merge Sort | O(n log n) | O(n) | ✅ | 안정적이고 병렬 처리에 적합 |
| Quick Sort | O(n log n) ~ O(n²) | O(log n) | ❌ | 평균적으론 가장 빠름 |

### 실무 예시

```java
import java.time.LocalDateTime;
import java.util.*;

class Log {
    String userId;
    LocalDateTime loginTime;

    public Log(String userId, LocalDateTime loginTime) {
        this.userId = userId;
        this.loginTime = loginTime;
    }
}

public class LogSortExample {
    public static void main(String[] args) {
        List<Log> logs = new ArrayList<>();
        logs.add(new Log("user1", LocalDateTime.of(2025,10,20,10,0)));
        logs.add(new Log("user2", LocalDateTime.of(2025,10,20,9,30)));
        logs.add(new Log("user3", LocalDateTime.of(2025,10,20,11,15)));

        logs.sort(Comparator.comparing(log -> log.loginTime)); // MergeSort 기반 (TimSort)
        logs.forEach(log -> System.out.println(log.userId + " : " + log.loginTime));
    }
}

```

- Java의 `List.sort()`는 내부적으로 TimSort (Merge + Insertion 혼합)을 사용
- 대용량 로그 분석 시 `Stream.sorted()`도 내부적으로 TimSort를 사용
- 로그/이벤트를 시간순으로 정렬한 뒤 `stream().collect(groupingBy())` 로 집계 가능

## 중급: 해시(Hash)와 캐싱 전략

### 개념

Hash 알고리즘은 데이터를 O(1) 시간에 접근할 수 있도록 키 기반으로 매핑한다. 
실무에서는 다음과 같은 곳에 적용된다.

- `HashMap`, `ConcurrentHashMap` → API 응답 캐시
- Redis, Memcached → 외부 캐시 서버
- JWT 토큰 서명/검증 → SHA256, HMAC

### 실무 예시: 캐시된 검색 결과 관리

```java
import java.util.*;

public class SearchCache {
    private static final Map<String, String> cache = new HashMap<>();

    public static String search(String query) {
        if (cache.containsKey(query)) {
            System.out.println("Cache hit for: " + query);
            return cache.get(query);
        }
        System.out.println("Cache miss for: " + query);
        String result = callExternalAPI(query); // 외부 API 호출 가정
        cache.put(query, result);
        return result;
    }

    private static String callExternalAPI(String query) {
        return "Result for " + query; // 가짜 API 응답
    }

    public static void main(String[] args) {
        search("Spring Boot");
        search("Spring Boot");
        search("Redis");
    }
}

```

- HashMap은 동시성 문제를 고려해야 함 → `ConcurrentHashMap` or `Caffeine Cache` 사용
- Redis를 연동하면 네트워크 캐시로 확장 가능
- Hash 충돌 관리 기법(Chaining vs Open Addressing)은 Redis의 내부 구조 이해에도 도움

## 고급: 그래프(Graph) 탐색과 추천 시스템

### 개념

그래프 알고리즘은 실무에서 **연관성 분석,** **추천 시스템**, **네트워크 탐색**에 활용된다.

**예:**

- 추천 서비스: 유저-아이템 관계 그래프
- 소셜 네트워크: 친구/팔로우 관계
- 경로 탐색: 물류, 지도 서비스, API 호출 경로 최적화

### 실무 예시: BFS 기반 “감정 유사도 기반 도서 추천”

```java
import java.util.*;

public class EmotionGraph {
    private final Map<String, List<String>> graph = new HashMap<>();

    public void addRelation(String emotion, String related) {
        graph.computeIfAbsent(emotion, k -> new ArrayList<>()).add(related);
    }

    public List<String> recommend(String start, int depth) {
        List<String> result = new ArrayList<>();
        Queue<String> queue = new LinkedList<>();
        Set<String> visited = new HashSet<>();

        queue.add(start);
        visited.add(start);

        while (!queue.isEmpty() && depth-- > 0) {
            int size = queue.size();
            for (int i = 0; i < size; i++) {
                String current = queue.poll();
                for (String next : graph.getOrDefault(current, List.of())) {
                    if (!visited.contains(next)) {
                        visited.add(next);
                        queue.add(next);
                        result.add(next);
                    }
                }
            }
        }
        return result;
    }

    public static void main(String[] args) {
        EmotionGraph eg = new EmotionGraph();
        eg.addRelation("위로", "치유");
        eg.addRelation("위로", "희망");
        eg.addRelation("희망", "성장");
        System.out.println(eg.recommend("위로", 2));
    }
}
```

- BFS는 **Redis Stream + Pub/Sub** 기반 알림 시스템에서 이벤트 브로드캐스트에 응용 가능
- DFS는 **GraphQL Resolver** 설계 시 순환 호출 방지 로직에 사용
- Graph 탐색 알고리즘은 “데이터 간 연결성”을 모델링할 때 매우 유용

## 마무리 - 알고리즘은 코드의 ‘뼈대’

> Framework는 근육이고, 알고리즘은 뼈대이다.
> 

Spring, JPA, Redis, Kafka, Elasticsearch 등 어떤 기술을 쓰더라도
그 내부는 결국 **정렬, 탐색, 해시, 그래프, 트리**로 구성된다.

## 다음 주제

| 단계 | 주제 | 실무 연결 |
| --- | --- | --- |
| #2 | 이진 탐색 & DB 인덱스 구조 | B-Tree / Clustered Index |
| #3 | 다익스트라 & API 라우팅 최적화 | API Gateway / Network Path |
| #4 | DP (Dynamic Programming) | 캐시 최적화 / 서브쿼리 최적화 |
| #5 | 트라이(Trie) & 자동완성 기능 | 검색 서비스 구현 |
| #6 | 힙(Heap) & 우선순위 큐 | 메시지 큐 / 작업 스케줄러 |