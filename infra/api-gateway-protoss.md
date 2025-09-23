# 🚪 API Gateway 이해하기 – 프로토스 게이트웨이와 MSA, Spring Boot

## 🎮 프로토스의 Gateway와 API Gateway

### 1. 프로토스 게이트웨이란?

![alt text](image.png)

- 스타크래프트에서 **게이트웨이(Gateway)**는 **프로토스의 기본 병력(질럿, 드라군, 템플러 등)을 생산하는 핵심 건물**이다.
- 프로토스 병력은 게이트웨이를 통해 **외부(전장)**에 등장한다(~~아이어를 위하여~!~~).
- 즉, **모든 병력 생산의 단일 진입점(Entry Point)** 역할을 하는 건물이다.

```
[ Nexus (본진) ] 
     │
     ▼
[ Gateway ] → [ Zealot / Dragoon / High Templar ... ]
```

### 2. API Gateway와의 비유

<div style="display: flex; justify-content: space-between; align-items: center;">
  <!-- 왼쪽 정렬 -->
  <img src="image-1.png" alt="Left" width="120" style="align-self: flex-start;"/>

  <!-- 가운데 정렬 -->
  <img src="image-2.png" alt="Center" width="160" style="align-self: center;"/>

  <!-- 오른쪽 정렬 -->
  <img src="image-3.png" alt="Right" width="120" style="align-self: flex-end;"/>
</div>

<br/>

- **API Gateway = 프로토스 게이트웨이**
  - 마이크로서비스(MSA)의 모든 요청이 **API Gateway를 통해서만 외부로 노출**된다.
  - 프로토스가 병력을 아무데서나 소환할 수 없는 것처럼, MSA도 서비스가 외부에 직접 노출되지 않는다.
- **마이크로서비스 = 프로토스 병력**
  - User Service, Order Service, Payment Service 같은 개별 마이크로서비스는 각각의 병력 유닛과 같다.
- **클라이언트 = 전장**
  - 실제 사용자는 전장에서 유닛(서비스)의 기능을 활용하지만, 그 유닛을 생산하는 과정은 게이트웨이를 통해서만 가능하다.

```
[ Client (전장) ]
       │
       ▼
[ API Gateway (프로토스 게이트웨이) ]
       ├──> [ User Service (Zealot) ]
       ├──> [ Order Service (Dragoon) ]
       └──> [ Payment Service (Dark Templar) ]
```

### 3. 게이트웨이의 역할 비교

| 역할 | 프로토스 게이트웨이 | API Gateway |
|------|-------------------|-------------|
| **생산/분배** | 유닛 생산 | 요청을 각 서비스로 라우팅 |
| **확장** | 다수 게이트웨이 건설로 병력 생산량 증가 | 다수 인스턴스로 부하 분산 (로드 밸런싱) |
| **보안** | 사이오닉 스톰, 캐논과 함께 기지 방어 | JWT 인증, 보안 필터 |
| **중앙 관리** | 유닛 훈련 중앙화 | 서비스 엔드포인트 중앙 관리 |
| **제한** | 게이트웨이 없으면 병력 생산 불가 | API Gateway 없으면 서비스 접근 불가 |

### 4. Spring Boot API Gateway 예시

```yaml
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        - id: zealot-service        # 질럿 유닛 = User Service
          uri: lb://USER-SERVICE
          predicates:
            - Path=/users/**
        - id: dragoon-service       # 드라군 유닛 = Order Service
          uri: lb://ORDER-SERVICE
          predicates:
            - Path=/orders/**
```

> 마치 `게이트웨이` 건물에서 `질럿` 또는 `드라군`을 뽑듯이,  
> API Gateway는 `/users/**` 요청을 User Service로, `/orders/**` 요청을 Order Service로 전달한다.

### 5. 정리

![alt text](image-4.png)

- **API Gateway는 마이크로서비스의 “생산 건물”이다.**
- 클라이언트는 직접 병력(서비스)에게 가지 않고, **게이트웨이(생산 건물)**를 통해 필요한 유닛을 얻는다.
- 게이트웨이가 많아질수록 → 트래픽 처리량(병력 생산량)이 증가한다.
- 게이트웨이가 무너지면 → 전장에 더 이상 유닛이 투입되지 못한다 (**SPOF 위험**).
  - SPOF(단일 장애 지점, Single Point of Failure): **시스템의 특정 구성 요소 중 하나가 고장나면 전체 시스템이 중단되는 취약점**
  - 게이트웨이가 파일런이 없어서 **Unpowered** 상태가 되는 걸 생각하면 됨
- 쉽게 정리:
  - API Gateway는 프로토스의 게이트웨이다.
  - 클라이언트는 게이트웨이를 통해 필요한 병력(서비스)을 공급받는다.
  - ~~앤 타로 아둔 제라툴~~

---

## API Gateway와 MSA, 그리고 Spring Boot 적용

### 1. 본격적으로 들어가며

- 앞서 설명한 프로토스 게이트웨이에 비유하여 API Gateway에 대해 좀 더 자세히 알아본다.
- **API Gateway**가 **MSA(Microservice Architecture)**에서 어떤 역할을 하고, 왜 중요한지, 그리고 **Java + Spring Boot**를 활용해 어떻게 적용할 수 있는지 정리해본다.
- MSA 환경에서 API Gateway는 단순한 요청 분배를 넘어 **보안, 라우팅, 모니터링**까지 담당하는 핵심 컴포넌트이다.

### 2. MSA(Microservice Architecture)란?

- **정의:** 하나의 거대한 애플리케이션을 작은 서비스 단위로 나누어 독립적으로 개발/배포/운영할 수 있도록 하는 아키텍처
- **특징**
  - 각 서비스는 독립적인 배포 가능
  - 서로 다른 기술 스택 사용 가능
  - 서비스 간 통신은 주로 **REST API** 또는 **gRPC** 활용
  - **확장성(Scalability)** 및 **유연성(Flexibility)** 강화

> 하지만 서비스가 많아질수록 클라이언트와 직접 연결되면 복잡성이 증가한다.  
> 이를 해결하는 것이 바로 **API Gateway**이다.

### 3. API Gateway란?

- **정의:** 클라이언트 요청을 받아 적절한 마이크로서비스로 전달하고, 응답을 다시 클라이언트에 반환하는 **단일 진입점(Entry Point)**
- **주요 역할:**
  1. **라우팅**: 요청을 해당 서비스로 전달
  2. **로드 밸런싱**: 여러 인스턴스 간 트래픽 분산
  3. **보안**: 인증/인가, JWT 토큰 검증
  4. **API 관리:** 요청/응답 로깅, 모니터링
  5. **변환:** REST ↔ gRPC, 데이터 포맷 변환
  6. **속도 제한/캐싱**: Rate Limiting, Response Caching

### 4. MSA와 API Gateway의 연계 구조

```
[ Client ] 
     │
     ▼
[ API Gateway ]
     ├──> [ Auth Service ]
     ├──> [ User Service ]
     ├──> [ Order Service ]
     └──> [ Payment Service ]
```

- **클라이언트는 Gateway만 바라봄**
- 서비스별 Endpoint는 외부에 노출되지 않음
- API Gateway가 **트래픽 조율자** 역할 수행

### 5. Spring Boot 기반 API Gateway 구현

Spring Boot에서 API Gateway를 구현할 때는 보통 **Spring Cloud Gateway**를 사용한다.

**Gradle 의존성 추가**

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
    implementation 'org.springframework.cloud:spring-cloud-starter-gateway'
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client' // 서비스 디스커버리
}
```

**application.yml 예시**

```yaml
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://USER-SERVICE
          predicates:
            - Path=/users/**
        - id: order-service
          uri: lb://ORDER-SERVICE
          predicates:
            - Path=/orders/**
```

- **lb://** : Eureka(서비스 디스커버리)와 연동 시 사용
- `/users/**` 요청은 User Service로 전달
- `/orders/**` 요청은 Order Service로 전달

**JWT 인증 필터 추가 예시**

```java
@Component
public class JwtAuthenticationFilter implements GatewayFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");

        if (token == null || !token.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        // JWT 검증 로직 (간단 예시)
        try {
            String jwt = token.substring(7);
            // 검증 로직 (예: Jwts.parser().setSigningKey(...).parseClaimsJws(jwt))
        } catch (Exception e) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        return chain.filter(exchange);
    }
}
```

### 6. API Gateway 도입 장점

- **클라이언트 단순화**: 여러 서비스 호출을 Gateway가 대신 처리
- **보안 강화**: 모든 요청을 중앙에서 제어
- **운영 편의성**: 모니터링, 로깅 일원화
- **확장성**: 트래픽 증가 시 Gateway 단에서 분산 처리 가능

### 7. 고려해야 할 단점

- **단일 장애 지점(SPOF, Single Point of Failure)**
- **추가 Latency 발생 가능성**
- **Gateway 자체의 성능/확장성 관리 필요**

> 따라서 Kubernetes + Service Mesh(Istio, Linkerd)와 연계하는 방식도 고려된다.
