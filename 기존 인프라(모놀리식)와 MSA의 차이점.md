# 기존 인프라(모놀리식)와 MSA의 차이점

기존 인프라 시스템과 MSA(Microservices Architecture)는 소프트웨어를 구성하고 배포하는 방식에서 근본적인 차이를 보인다. 단순히 "어떻게 코드를 나누는가"의 문제가 아니라, 팀 구조, 배포 전략, 장애 대응, 확장 방식까지 모든 것이 달라진다.

---

## 모놀리식 아키텍처 (Monolithic Architecture)

모놀리식 아키텍처는 애플리케이션의 모든 기능(UI, 비즈니스 로직, 데이터 접근 계층)이 하나의 코드베이스에 존재하고, 단일 실행 파일로 배포되는 구조다.

```
┌──────────────────────────────────────┐
│           Monolithic App             │
│  ┌────────────┐  ┌────────────────┐  │
│  │  사용자 인증 │  │   주문 관리     │  │
│  ├────────────┤  ├────────────────┤  │
│  │   결제 처리  │  │   재고 관리     │  │
│  ├────────────┤  ├────────────────┤  │
│  │   알림 발송  │  │   정산 처리     │  │
│  └────────────┘  └────────────────┘  │
│           단일 데이터베이스             │
└──────────────────────────────────────┘
              ↓ 단일 배포
          WAS (Tomcat 등)
```

### 모놀리식의 장점

개발 초기 단계에서 명확한 이점이 있다.

- **단순한 배포**: JAR 또는 WAR 하나를 서버에 복사하면 끝난다
- **로컬 개발 편의성**: 하나의 IDE, 하나의 실행 환경으로 전체 시스템을 구동할 수 있다
- **디버깅 용이**: 모든 로직이 같은 프로세스 안에 있으므로 스택 트레이스 추적이 직관적이다
- **인메모리 통신**: 서비스 간 호출이 네트워크가 아닌 메서드 호출이므로 오버헤드가 없다
- **통합 테스트**: 전체 애플리케이션을 하나의 환경에서 E2E 테스트 가능하다
- **트랜잭션 관리 단순**: 로컬 ACID 트랜잭션을 그대로 사용할 수 있다

### 모놀리식의 한계

서비스가 성장하면서 다음 문제가 반드시 발생한다.

```java
// 문제 예시: 결제 로직의 버그가 전체 서비스를 다운시킨다
@Service
public class PaymentService {
    public void processPayment(Order order) {
        // OOM 또는 무한 루프 발생 시
        // 사용자 인증, 재고 조회, 알림 등 모든 기능이 함께 중단된다
        heavyProcessing(order);
    }
}
```

| 문제 | 설명 |
| --- | --- |
| **부분 확장 불가** | 결제 기능만 트래픽이 몰려도 전체 앱을 스케일아웃해야 한다 |
| **배포 리스크** | 작은 버그 수정 하나를 위해 전체 앱을 재배포해야 한다 |
| **기술 고착** | 일부 기능에만 새 프레임워크나 언어를 적용하기 어렵다 |
| **빌드 시간 증가** | 코드베이스가 커질수록 빌드와 테스트에 수십 분이 소요된다 |
| **팀 병목** | 여러 팀이 같은 코드베이스에서 작업하면 충돌과 조율 비용이 급증한다 |
| **장애 전파** | 하나의 모듈 장애가 전체 서비스 가용성에 영향을 준다 |

---

## MSA (Microservices Architecture)

MSA는 하나의 큰 애플리케이션을 독립적으로 배포 가능한 작은 서비스들의 집합으로 분해하는 아키텍처다. 각 서비스는 단일 비즈니스 기능에 집중하고, 자체 데이터베이스를 보유하며, 네트워크(HTTP/gRPC/메시지 큐)를 통해 다른 서비스와 통신한다.

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  인증 서비스   │  │  주문 서비스   │  │  결제 서비스   │
│  (Node.js)   │  │   (Java)     │  │  (Kotlin)    │
│   MySQL      │  │  MongoDB     │  │ PostgreSQL   │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       └─────────────────┴─────────────────┘
                         │
               API Gateway / Message Broker
                         │
                  ┌──────┴──────┐
                  │   Client    │
                  └─────────────┘
```

### MSA의 핵심 원칙

**1. 단일 책임 (Single Responsibility)**

각 서비스는 하나의 비즈니스 도메인에만 집중한다. 도메인 주도 설계(DDD)의 Bounded Context 개념을 인프라 수준에서 구현한 것이다.

**2. 서비스별 독립 데이터베이스**

```
# 잘못된 MSA - 데이터베이스를 공유하면 결합도가 생긴다
Order Service   ─────┐
                     ├──→ 공유 DB  (안티패턴)
Payment Service ─────┘

# 올바른 MSA - 각 서비스가 자체 DB를 소유한다
Order Service   ──→ Order DB
Payment Service ──→ Payment DB
User Service    ──→ User DB
```

**3. 독립 배포**

서비스 A를 배포할 때 서비스 B, C는 영향받지 않는다. 이것이 MSA의 가장 핵심적인 운영 이점이다.

---

## 모놀리식 vs MSA 핵심 비교

| 항목 | 모놀리식 | MSA |
| --- | --- | --- |
| **배포 단위** | 전체 애플리케이션 | 서비스별 독립 배포 |
| **확장 방식** | 전체 앱 수평 확장 | 부하가 몰리는 서비스만 확장 |
| **장애 격리** | 하나의 버그가 전체에 영향 | 서킷 브레이커로 장애 격리 |
| **기술 스택** | 단일 언어/프레임워크 강제 | 서비스별 최적 기술 선택 가능 |
| **팀 구조** | 기능별 팀 (수평 분리) | 서비스별 팀 (수직 분리) |
| **데이터베이스** | 중앙집중식 단일 DB | 서비스별 독립 DB |
| **통신 방식** | 메서드 직접 호출 | HTTP REST, gRPC, 메시지 큐 |
| **로컬 개발** | 단순 (하나의 실행 환경) | 복잡 (Docker Compose 등 필요) |
| **운영 복잡도** | 낮음 | 높음 (서비스 디스커버리, 분산 추적 필요) |
| **트랜잭션** | 로컬 ACID 트랜잭션 (간단) | 분산 트랜잭션 (Saga 패턴 필요) |

---

## 인프라 관점의 차이

### 모놀리식 인프라 구성

```yaml
# 단순한 배포 구조
- app.jar 하나 배포
- Nginx (리버스 프록시)
- 단일 DB 서버
- 스케일아웃: 동일한 app.jar를 여러 서버에 복제

┌─────────────────────────────┐
│          Client             │
└──────────────┬──────────────┘
               │
┌──────────────▼──────────────┐
│       Nginx (L4/L7)         │
└──────┬──────────────┬───────┘
       │              │
┌──────▼──────┐ ┌─────▼───────┐
│  App #1     │ │   App #2    │  ← 동일한 JAR 복제
│ (app.jar)   │ │ (app.jar)   │
└──────┬──────┘ └─────┬───────┘
       └──────┬────────┘
       ┌──────▼───────┐
       │  단일 DB       │
       └──────────────┘
```

### MSA 인프라 필수 구성 요소

MSA를 운영하려면 다음 인프라 구성 요소가 필수다.

**1. API Gateway**

클라이언트의 단일 진입점. 라우팅, 인증, 속도 제한(Rate Limiting), 로드밸런싱을 담당한다.

```yaml
# Spring Cloud Gateway 예시
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - AuthFilter
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
        - id: payment-service
          uri: lb://payment-service
          predicates:
            - Path=/api/payments/**
```

**2. 서비스 디스커버리 (Service Discovery)**

서비스 인스턴스의 위치(IP, 포트)를 동적으로 관리한다. Netflix는 Eureka를 직접 개발해 오픈소스로 공개했다.

```java
// Eureka 클라이언트 등록 예시
@SpringBootApplication
@EnableEurekaClient
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}

// application.yml
eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka/
  instance:
    prefer-ip-address: true
```

**3. 서킷 브레이커 (Circuit Breaker)**

연쇄 장애(Cascading Failure)를 방지하는 핵심 패턴이다. 호출 대상 서비스가 일정 횟수 이상 실패하면 회로를 차단하고 즉시 폴백 응답을 반환한다.

```java
// Resilience4j 서킷 브레이커 적용
@Service
public class OrderService {

    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    @TimeLimiter(name = "paymentService")
    public CompletableFuture<PaymentResult> processPayment(Long orderId) {
        return CompletableFuture.supplyAsync(() ->
            paymentClient.process(orderId)
        );
    }

    // 결제 서비스 장애 시 임시 응답 반환
    public CompletableFuture<PaymentResult> paymentFallback(Long orderId, Exception e) {
        log.warn("Payment service unavailable. orderId={}", orderId, e);
        return CompletableFuture.completedFuture(
            PaymentResult.pending(orderId, "결제 서비스 일시 중단 - 잠시 후 다시 시도해주세요")
        );
    }
}
```

```yaml
# Resilience4j 설정
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        failure-rate-threshold: 50          # 실패율 50% 이상이면 회로 차단
        wait-duration-in-open-state: 30s    # 30초 후 Half-Open 상태 전환
        sliding-window-size: 10             # 최근 10번 요청 기준
```

**4. 분산 추적 (Distributed Tracing)**

모놀리식에서는 스택 트레이스 하나로 문제를 찾지만, MSA에서는 요청이 여러 서비스를 거치기 때문에 분산 추적이 필수다.

```
요청 흐름 추적 예시 (Zipkin / Jaeger):

Client → API Gateway          [Trace ID: abc123, Span: gateway]
  → Order Service             [Trace ID: abc123, Span: order-create, 12ms]
    → Inventory Service       [Trace ID: abc123, Span: stock-check, 8ms]
    → Payment Service         [Trace ID: abc123, Span: payment-process, 45ms]
      → Notification Service  [Trace ID: abc123, Span: send-email, 5ms]

총 응답 시간: 70ms (각 서비스별 기여도 추적 가능)
```

```yaml
# Spring Boot + Micrometer Tracing 설정
management:
  tracing:
    sampling:
      probability: 1.0  # 전체 요청 추적 (운영환경은 0.1 권장)
spring:
  zipkin:
    base-url: http://zipkin:9411
```

**5. 컨테이너 오케스트레이션 (Kubernetes)**

수십~수백 개의 서비스를 수동으로 관리하는 것은 불가능하다. Kubernetes가 서비스 배포, 롤링 업데이트, 자동 복구, 오토스케일링을 담당한다.

```yaml
# Order Service Kubernetes Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # 무중단 배포
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: codesche/order-service:v1.2.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

**6. 분산 트랜잭션 - Saga 패턴**

모놀리식에서는 `@Transactional` 하나로 해결되던 것이, MSA에서는 Saga 패턴이 필요하다.

```java
// Choreography-based Saga 예시 (Kafka 이벤트 기반)

// 1. 주문 서비스: 주문 생성 후 이벤트 발행
@Service
public class OrderService {
    public Order createOrder(OrderRequest request) {
        Order order = orderRepository.save(new Order(request, OrderStatus.PENDING));
        eventPublisher.publish(new OrderCreatedEvent(order.getId(), order.getItems()));
        return order;
    }
}

// 2. 재고 서비스: 이벤트 수신 후 재고 차감
@KafkaListener(topics = "order-created")
public void handleOrderCreated(OrderCreatedEvent event) {
    try {
        inventoryService.reserve(event.getItems());
        eventPublisher.publish(new InventoryReservedEvent(event.getOrderId()));
    } catch (InsufficientStockException e) {
        // 보상 트랜잭션: 주문 취소 이벤트 발행
        eventPublisher.publish(new OrderCancelledEvent(event.getOrderId(), "재고 부족"));
    }
}

// 3. 결제 서비스: 재고 확보 확인 후 결제 진행
@KafkaListener(topics = "inventory-reserved")
public void handleInventoryReserved(InventoryReservedEvent event) {
    // 결제 처리 로직
}
```

---

## 실제 적용 사례

### Netflix - MSA의 교과서

2008년 Netflix는 데이터베이스 손상 사고로 3일간 DVD 배송이 중단되었다. 이 사고가 MSA 전환의 직접적인 계기가 되었다.

> "단일 장애점인 관계형 데이터베이스에서 벗어나 고가용성의 수평 확장 가능한 분산 시스템으로 이동해야 한다는 것을 깨달았다." — Netflix 엔지니어

**전환 전략:**
- 중요도가 낮은 서비스부터 점진적으로 분리
- AWS로 인프라를 전환하면서 MSA를 함께 도입
- Eureka(서비스 디스커버리), Zuul(API Gateway), Hystrix(서킷 브레이커)를 직접 개발해 오픈소스로 공개

**현재 상태:**
- 1,000개 이상의 마이크로서비스 운영
- 개발자들이 하루에 수백 번 코드를 배포
- 2억 3천만 명 이상의 사용자에게 중단 없는 서비스 제공
- Federated GraphQL을 도입해 클라이언트가 여러 마이크로서비스 데이터를 단일 쿼리로 조회

### Amazon - MSA를 AWS로 연결한 사례

2002년 Jeff Bezos는 직원들에게 다음 내용의 유명한 내부 지침 메일을 발송했다.

> "모든 팀은 서비스 인터페이스를 통해 데이터와 기능을 노출해야 한다. 팀 간의 모든 통신은 이 인터페이스를 통해서만 이루어져야 한다. 다이렉트 링크, 다른 팀의 데이터 스토어 직접 읽기, 공유 메모리 모델, 어떠한 백도어도 허용하지 않는다."

이 내부 플랫폼이 결국 AWS(Amazon Web Services)의 기반이 되었다. Amazon은 MSA를 20년에 걸쳐 준비한 셈이다.

### Uber - 급격한 성장으로 인한 전환

초창기 Uber는 단일 모놀리식 앱에서 출발했다. 탑승객과 드라이버가 REST API를 통해 모놀리스에 연결하는 구조였다.

글로벌 확장 과정에서 다음 문제가 발생했다:
- 새 기능 개발 속도 저하
- 여러 지역에서 모놀리식 앱을 운영하는 복잡도 증가
- 사소한 수정에도 전체 시스템에 대한 깊은 이해가 필요

MSA 전환 후, 각 팀이 특정 서비스 하나에만 집중하게 되면서 문제 해결 속도와 개발 품질이 즉각적으로 향상되었다. 트래픽이 몰리는 서비스만 집중적으로 스케일아웃할 수 있어 급격한 성장에도 효율적으로 대응 가능해졌다.

---

## MSA가 항상 정답은 아니다

| 상황 | 권장 아키텍처 |
| --- | --- |
| 초기 스타트업, MVP 단계 | 모놀리식 |
| 팀 규모 5명 이하 | 모놀리식 |
| 도메인이 단순하고 기능이 적음 | 모놀리식 |
| 빠른 프로토타이핑 필요 | 모놀리식 |
| 팀 규모가 크고 서비스가 복잡 | MSA |
| 트래픽 패턴이 기능별로 불균일 | MSA |
| 독립적인 배포 주기가 필요 | MSA |
| 기술 스택 다양화가 필요 | MSA |

Shopify는 수십억 달러 규모의 블랙프라이데이 트래픽을 Rails 모놀리스로 처리하고, Stack Overflow는 마이크로서비스 없이 수백만 개발자를 서비스한다. 아키텍처는 트렌드가 아니라 현재 팀의 규모와 문제에 맞게 선택해야 한다.

---

## 모놀리식에서 MSA로 전환할 때의 전략

**Strangler Fig Pattern (교살자 무화과 패턴)**

기존 모놀리스를 한 번에 다 바꾸는 것은 매우 위험하다. Martin Fowler가 제안한 Strangler Fig Pattern은 기존 시스템을 유지하면서 새 기능은 마이크로서비스로 개발하고, 점진적으로 기존 기능을 이전하는 방식이다.

```
[단계 1] 모놀리스 유지, 새 서비스는 별도 구성
  Client → Monolith (기존 기능 전담)
         → New Microservice (신규 기능 전담)

[단계 2] 점진적 이전
  Client → API Gateway → Monolith (잔여 기능)
                       → Service A (이전 완료)
                       → Service B (이전 완료)

[단계 3] 모놀리스 완전 제거
  Client → API Gateway → Service A
                       → Service B
                       → Service C
```

---

## 정리

모놀리식과 MSA는 서로 다른 문제를 해결하기 위한 도구다. 모놀리식은 초기 개발 속도와 단순성을 극대화하고, MSA는 서비스가 복잡해지고 팀이 커졌을 때 독립적인 확장성과 장애 격리를 제공한다.

Netflix, Uber, Amazon 모두 모놀리식으로 시작해 성장 과정에서의 구체적인 문제를 해결하기 위해 MSA로 전환했다. 아키텍처 결정은 항상 현재 팀의 규모, 기술 역량, 서비스의 복잡도를 기준으로 내려야 한다.
