# Kafka 제대로 이해하기

## 들어가며

최근 TRPG 게임인 **DungeonTalk** 프로젝트를 진행하며 메시징 시스템을 구축했다.  
Redis Pub/Sub과 WebSocket을 활용한 실시간 채팅 시스템인데, 이 과정에서  
“만약 사용자가 폭발적으로 늘어나면 어떻게 할 수 있을까?”라는 고민이 생겼다.

### 참고링크

- DungeonTalk Repository: https://github.com/DungeonTalk/dungeontalk-backend
- DungeonTalk 팀 위키: https://github.com/DungeonTalk/dungeontalk-backend/wiki

---

## DungeonTalk의 선택

현재 DungeonTalk은 **Redis Pub/Sub + WebSocket** 을 사용하고 있다.  
그 이유는 다음과 같다:

- 실시간 게임 특성상 낮은 지연시간 중요
- 빠른 시간에 MVP를 출시하는 것이 목표였음
- 간단한 구조 구성을 통하여 유지보수의 안정성 유지

하지만 다음과 같은 상황인 경우 Kafka 도입이 꼭 필요하다.

- 동시 접속자 10,000명 이상
- 게임 로그 분석 시스템 구축
- 실시간 이상 탐지 시스템 도입
- 멀티 리전 확장

---

## WebSocket vs Pub/Sub vs Kafka

### WebSocket - 실시간 양방향 통신의 기본

DungeonTalk에선 WebSocket STOMP 프로토콜을 사용한다.

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/sub")
                .setHeartbeatValue(new long[]{30_000, 30_000});
        registry.setApplicationDestinationPrefixes("/pub");
    }
}
```

**WebSocket의 특징**

- 클라이언트 ↔ 서버 간 실시간 양방향 통신
- HTTP 연결을 업그레이드하여 지속적인 연결 유지
- 낮은 지연시간(Low Latency)
- 단일 서버에서만 동작 → 확장성 제약

#### Redis Pub/Sub - 간단한 메시지 브로커

DungeonTalk의 멀티 서버 환경에서 메시지를 동기화하기 위해 Redis Pub/Sub 을 사용한다.

```java
@Component
public class RedisPublisher {
    private final RedisTemplate<String, Object> redisTemplate;

    // 채팅방에 메시지 발행
    public void publish(String roomId, String message) {
        redisTemplate.convertAndSend("chatroom." + roomId, message);
    }

    // AI 채팅에 메시지 발행
    public void publishAiChat(String aiGameRoomId, String message) {
        redisTemplate.convertAndSend("aichat." + aiGameRoomId, message);
    }
}
```

```java
@Service
public class RedisSubscriber implements MessageListener {
    private final SimpMessageSendingOperations messagingTemplate;

    public void onMessage(Message message, byte[] pattern) {
        String payload = new String(message.getBody());
        String topic = new String(message.getChannel());

        // chatroom.{roomId} 채널에서 메시지 수신
        String roomId = topic.substring("chatroom.".length());

        // WebSocket을 통해 클라이언트에 전달
        messagingTemplate.convertAndSend("/sub/chat/room/" + roomId, payload);
    }
}
```

**Redis Pub/Sub 특징**

- 메시지를 모든 구독자에게 브로드캐스트
- 메시지 저장 안 됨 (Fire and Forget)
- 구독자가 없으면 메시지 유실
- 간단한 구조, 빠른 속도
- 메시지 보장 없음 (메시지 손실 가능)

### Kafka - 대용량 트래픽의 해결사

Kafka의 핵심 철학: **“메시지는 영속적으로 저장되고, 소비자는 자신의 속도로 읽는다”**

```java
// Kafka Producer 예시
@Service
public class KafkaMessageProducer {
    private final KafkaTemplate<String, GameMessage> kafkaTemplate;

    public void sendGameMessage(String roomId, GameMessage message) {
        // 메시지를 Kafka 토픽으로 전송
        kafkaTemplate.send("game-messages", roomId, message)
            .whenComplete((result, ex) -> {
                if (ex == null) {
                    log.info("메시지 전송 성공: offset={}", result.getRecordMetadata().offset());
                } else {
                    log.error("메시지 전송 실패", ex);
                }
            });
    }
}
```

```java
// Kafka Consumer 예시
@Service
public class KafkaMessageConsumer {
    @KafkaListener(topics = "game-messages", groupId = "dungeontalk-group")
    public void consumeGameMessage(
        @Payload GameMessage message,
        @Header(KafkaHeaders.OFFSET) Long offset,
        @Header(KafkaHeaders.PARTITION) int partition
    ) {
        log.info("메시지 수신: partition={}, offset={}", partition, offset);
        // 메시지 처리 로직
        processGameMessage(message);
    }
}
```

**Kafka 특징**

- 메시지를 디스크에 영속적으로 저장 (기본 7일)
- 메시지 순서 보장 (파티션 내에서)
- 재처리 가능 (offset 조정으로 과거 메시지 재소비)
- 대용량 처리 (초당 수백만 메시지)
- 수평 확장 (파티션 추가로 무한 확장)

---

## Kafka vs RabbitMQ - 메시징 시스템 비교

### RabbitMQ - 전통적인 메시지 큐

```java
// RabbitMQ 메시지 전송
@Service
public class RabbitMQProducer {
    private final RabbitTemplate rabbitTemplate;

    public void sendToQueue(String queueName, String message) {
        // 큐에 메시지 전송 (큐가 비면 메시지 제거됨)
        rabbitTemplate.convertAndSend(queueName, message);
    }
}
```

**RabbitMQ의 특징**

- 메시지 큐 방식: 메시지를 소비하면 큐에서 제거
- 라우팅 기능 강력: Exchange를 통한 복잡한 라우팅
- 우선순위 큐 지원: 중요한 메시지 먼저 처리
- 즉시 전달 보장: push 방식으로 즉시 전달
- 메모리 기반: 빠르지만 대용량에는 부적합

### Kafka - 로그 기반 스트리밍 플랫폼

```java
// Kafka는 메시지를 삭제하지 않고 계속 보관
@Service
public class KafkaStreamProcessor {
    @KafkaListener(topics = "user-actions")
    public void processUserAction(UserAction action) {
        // 이 메시지는 소비되어도 Kafka에 남아있음
        // 다른 Consumer Group이 동일한 메시지를 다시 읽을 수 있음
        log.info("사용자 행동 처리: {}", action);
    }
}
```

**Kafka의 특징**

- 로그 기반: 메시지를 append-only 로그로 저장
- 메시지 영속성: 소비해도 메시지 유지
- 다중 소비자 그룹: 같은 메시지를 여러 그룹이 독립적으로 소비
- Pull 방식: 소비자가 자신의 속도로 메시지 가져옴
- 대용량 특화: 수평 확장에 최적화

### 비교표

| 특징       | Redis Pub/Sub | RabbitMQ     | Kafka         |
|------------|----------------|--------------|---------------|
| 메시지 저장 | 저장 안함       | 메모리/디스크 | 디스크 (영속)  |
| 메시지 보장 | 없음           | ACK 기반     | Offset 기반   |
| 재처리 가능 | 불가능         | 소비 후 삭제  | 가능          |
| 처리량      | ~100K msg/s   | ~10K msg/s   | ~1M msg/s     |
| 확장성      | 제한적         | 중간         | 우수          |
| 복잡도      | 낮음           | 중간         | 높음          |
| 사용 사례   | 실시간 알림     | 작업 큐       | 이벤트 스트리밍 |

---

## Kafka 핵심 개념 정리

### 1. Producer - 메시지를 생성하는 주체

```java
@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, GameEvent> producerFactory() {
        Map<String, Object> config = new HashMap<>();

        // Kafka 브로커 주소
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        // Key/Value 직렬화 설정
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);

        // 성능 튜닝 옵션
        config.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384); // 배치 크기
        config.put(ProducerConfig.LINGER_MS_CONFIG, 10);     // 배치 대기시간
        config.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy"); // 압축

        // 메시지 전송 보장 수준
        config.put(ProducerConfig.ACKS_CONFIG, "all"); // 모든 replica 확인

        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

**Producer의 역할**

- 메시지를 Kafka 토픽에 발행
- 파티션 선택 (키 기반 또는 라운드로빈)
- 배치 처리로 성능 최적화

### 2. Consumer - 메시지를 소비하는 주체

```java
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConsumerFactory<String, GameEvent> consumerFactory() {
        Map<String, Object> config = new HashMap<>();

        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

        // Consumer Group ID (중요!)
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "dungeontalk-game-processor");

        // Offset 자동 커밋 설정
        config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        // 처음 시작 시 어디서부터 읽을지
        config.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);

        return new DefaultKafkaConsumerFactory<>(config);
    }
}
```

**Consumer Group의 기능 사례**

```java
// Consumer Group 1: 실시간 게임 처리
@KafkaListener(topics = "game-events", groupId = "game-processor")
public void processGameEvent(GameEvent event) {
    log.info("실시간 게임 처리: {}", event);
}

// Consumer Group 2: 통계 분석 (같은 메시지를 독립적으로 소비)
@KafkaListener(topics = "game-events", groupId = "analytics")
public void analyzeGameEvent(GameEvent event) {
    log.info("통계 분석: {}", event);
}

// Consumer Group 3: 알림 발송 (같은 메시지를 또 독립적으로 소비)
@KafkaListener(topics = "game-events", groupId = "notification")
public void sendNotification(GameEvent event) {
    log.info("알림 발송: {}", event);
}
```

### 3. Topic & Partition - 메시지의 저장소

```markdown
Topic: game-events (파티션 3개)

Partition 0: [msg1] [msg4] [msg7] ...
Partition 1: [msg2] [msg5] [msg8] ...
Partition 2: [msg3] [msg6] [msg9] ...
```

**파티션의 역할**

- 메시지를 병렬로 처리
- 같은 키를 가진 메시지는 같은 파티션으로 (순서 보장)
- 파티션 수만큼 Consumer를 병렬 실행 가능

```java
// 게임방 ID를 키로 사용하여 같은 방의 메시지는 순서 보장
public void sendGameMessage(String roomId, GameMessage message) {
    // roomId를 키로 사용 → 같은 roomId는 같은 파티션으로
    kafkaTemplate.send("game-messages", roomId, message);
}
```

### 4. Offset - 메시지의 위치

```markdown
Partition 0의 메시지 로그:
[0] [1] [2] [3] [4] [5] [6] [7] [8] [9] [10]
             ↑ 현재 Consumer Offset: 4
Consumer는 offset 5부터 읽기 시작
```

**수동 Offset 커밋**

```java
@KafkaListener(topics = "game-events")
public void consume(ConsumerRecord<String, GameEvent> record, Acknowledgment ack) {
    try {
        // 메시지 처리
        processGameEvent(record.value());
        // 처리 성공 시 offset 커밋
        ack.acknowledge();
    } catch (Exception e) {
        // 처리 실패 시 커밋 안함 → 재시도
        log.error("메시지 처리 실패, 재시도 예정", e);
    }
}
```

---

## DungeonTalk에 Kafka를 적용한다면?

### 현재 구조 (Redis Pub/Sub + WebSocket)

```markdown
Client → WebSocket → Spring Boot → Redis Pub/Sub → Spring Boot → WebSocket → Client
                                  ↓
                              PostgreSQL
```

**장점**

- 간단한 구조
- 낮은 지연시간
- 실시간성 우수

**단점**

- 메시지 유실 가능
- 재처리 불가능
- 대용량 트래픽에 취약
- 확장성 제한

### Kafka 적용 구조

```markdown
Client → WebSocket → Spring Boot → Kafka
                               ↓
                    Consumer Group 1 (실시간 전송)
                               ↓
                    Consumer Group 2 (DB 저장)
                               ↓
                    Consumer Group 3 (분석)
```

```java
// 1. 메시지 수신 시 Kafka로 발행
@MessageMapping("/aichat/send")
public void sendMessage(AiGameMessageSendRequest request) {
    // Kafka에 메시지 발행
    GameMessage message = GameMessage.builder()
        .roomId(request.getRoomId())
        .content(request.getContent())
        .timestamp(Instant.now())
        .build();

    kafkaProducer.sendGameMessage(message);
}

// 2. Consumer Group 1: 실시간 WebSocket 전송
@KafkaListener(topics = "game-messages", groupId = "websocket-sender")
public void sendToWebSocket(GameMessage message) {
    messagingTemplate.convertAndSend(
        "/sub/aichat/room/" + message.getRoomId(), message
    );
}

// 3. Consumer Group 2: DB 저장 (비동기)
@KafkaListener(topics = "game-messages", groupId = "db-writer")
public void saveToDatabase(GameMessage message) {
    aiGameMessageRepository.save(message.toEntity());
}

// 4. Consumer Group 3: 실시간 분석
@KafkaListener(topics = "game-messages", groupId = "analytics")
public void analyzeMessage(GameMessage message) {
    // 사용자 활동 분석, 이상 탐지 등
    analyticsService.analyze(message);
}
```

### Kafka 적용 후 얻게 되는 장점

1. **메시지 보장**

    ```java
    // 메시지 전송 실패 시 자동 재시도
    @Retryable(
        value = { KafkaException.class },
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000)
    )
    public void sendWithRetry(GameMessage message) {
        kafkaTemplate.send("game-messages", message);
    }
    ```

2. **재처리 가능**

    ```java
    // 특정 시점부터 메시지 재처리
    @KafkaListener(topics = "game-messages")
    public void reprocessMessages(ConsumerRecord<String, GameMessage> record) {
        // offset을 조정하여 과거 메시지 재처리 가능
        log.info("재처리 offset: {}", record.offset());
    }
    ```

3. **부하 분산**

    ```markdown
    Partition 0 → Consumer 1
    Partition 1 → Consumer 2
    Partition 2 → Consumer 3
    ```

4. **장애 복구**

    ```java
    // Consumer가 장애로 중단되어도, 재시작 시 마지막 offset부터 계속 처리
    @KafkaListener(topics = "game-messages")
    public void consume(GameMessage message) {
        // 처리 중 서버가 재시작되어도
        // 마지막 커밋된 offset 이후부터 자동으로 계속 처리
    }
    ```

---

## 대용량 트래픽 처리 전략

### 1. 파티션 전략

```java
// 사용자 수에 따라 파티션 수 결정
// 동시 접속자 10,000명 예상 → 파티션 30개 (약 333명/파티션)
@Configuration
public class KafkaTopicConfig {

    @Bean
    public NewTopic gameMessagesTopic() {
        return TopicBuilder.builder()
            .name("game-messages")
            .partitions(30) // 파티션 30개
            .replicas(3)    // 복제본 3개 (장애 대비)
            .config(TopicConfig.RETENTION_MS_CONFIG, "604800000") // 7일 보관
            .build();
    }
}
```

### 2. 배치 처리

```java
// 메시지를 모아서 한 번에 처리 (처리량 향상)
@KafkaListener(
    topics = "game-messages",
    containerFactory = "batchContainerFactory"
)
public void consumeBatch(List<GameMessage> messages) {
    // 100개 메시지를 한 번에 처리
    log.info("배치 처리: {}개 메시지", messages.size());

    // Bulk Insert로 성능 향상
    aiGameMessageRepository.saveAll(
        messages.stream()
            .map(GameMessage::toEntity)
            .collect(Collectors.toList())
    );
}
```

### 3. 압축

```java
@Bean
public ProducerFactory<String, GameMessage> producerFactory() {
    Map<String, Object> config = new HashMap<>();

    // Snappy 압축 (빠른 속도, 적당한 압축률)
    config.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");

    // 또는 GZIP (느린 속도, 높은 압축률)
    // config.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "gzip");

    return new DefaultKafkaProducerFactory<>(config);
}
```

### 4. 병렬 Consumer

```yaml
# application.yml
spring:
  kafka:
    consumer:
      concurrency: 10 # 동시에 10개 Consumer 실행
```

```java
@KafkaListener(
    topics = "game-messages",
    groupId = "game-processor",
    concurrency = "10" // 10개의 Consumer 인스턴스가 병렬 처리
)
public void consume(GameMessage message) {
    processGameMessage(message);
}
```

---

## 결론

### Redis Pub/Sub을 선택해야 할 때

- 실시간성이 가장 중요할 때 (10ms 이하)
- 메시지 유실이 허용될 때
- 간단한 브로드캐스트가 필요할 때
- 동시 접속자가 적을 때 (< 1,000명)

### Kafka를 선택해야 할 때

- 메시지 유실이 절대 안 될 때
- 대용량 트래픽 처리가 필요할 때(> 10,000 msg/s)
- 메시지 재처리가 필요할 때
- 여러 시스템이 같은 메시지를 소비할 때
- 이벤트 소싱 패턴을 적용할 때
