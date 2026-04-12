# DB · 인프라 · 클라우드 실무 핵심 요약

백엔드 개발자로 일하다 보면 코드 외에도 알아야 할 게 많다. 데이터베이스 설계와 튜닝, 서버 인프라 구성, 클라우드 서비스 활용까지 세 영역이 맞물려 돌아가야 서비스가 안정적으로 운영된다. 이 글에서는 각 영역의 핵심 개념과 실무에서 어떻게 적용하는지를 요약 형태로 정리한다.

---

## 1. 데이터베이스(DB)

### 인덱스 전략

인덱스는 조회 성능을 좌우한다. 잘 설계하면 수십 배 빠르지만, 잘못 쓰면 쓰기 성능을 오히려 떨어뜨린다.

```sql
-- 자주 조회하는 컬럼에 단일 인덱스
CREATE INDEX idx_user_email ON users(email);

-- WHERE 조건 + ORDER BY 함께 쓰는 패턴엔 복합 인덱스
CREATE INDEX idx_order_user_created ON orders(user_id, created_at DESC);

-- 인덱스가 무력화되는 패턴 (함수 사용 금지)
SELECT * FROM users WHERE LOWER(email) = 'test@email.com'; -- 인덱스 안 탐
SELECT * FROM users WHERE email = 'test@email.com';         -- 인덱스 탐
```

**실무 체크리스트:**
- 자주 조회하는 컬럼, JOIN 키, WHERE 조건 컬럼에 인덱스 추가
- 복합 인덱스는 선택도(카디널리티)가 높은 컬럼을 앞에
- 인덱스가 너무 많으면 INSERT/UPDATE 성능 저하
- `EXPLAIN` / `EXPLAIN ANALYZE`로 실행 계획 반드시 확인

---

### 트랜잭션과 격리 수준

동시 요청이 많을수록 트랜잭션 처리가 중요해진다.

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| READ UNCOMMITTED | 발생 | 발생 | 발생 |
| READ COMMITTED | 방지 | 발생 | 발생 |
| REPEATABLE READ | 방지 | 방지 | 발생 |
| SERIALIZABLE | 방지 | 방지 | 방지 |

실무에서는 대부분 **READ COMMITTED** 또는 **REPEATABLE READ**를 기본으로 쓴다. MySQL InnoDB의 기본값은 REPEATABLE READ다.

```sql
-- 트랜잭션 처리 기본
BEGIN;
UPDATE accounts SET balance = balance - 10000 WHERE id = 1;
UPDATE accounts SET balance = balance + 10000 WHERE id = 2;
COMMIT;

-- 격리 수준 변경
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

---

### 커넥션 풀(Connection Pool)

DB 커넥션을 매번 새로 맺는 건 비용이 크다. 커넥션 풀을 써서 미리 맺어둔 커넥션을 재사용한다.

```yaml
# Spring Boot - HikariCP 설정 예시
spring:
  datasource:
    hikari:
      maximum-pool-size: 10      # 최대 커넥션 수
      minimum-idle: 5            # 최소 유지 커넥션
      connection-timeout: 30000  # 커넥션 대기 타임아웃 (ms)
      idle-timeout: 600000       # 유휴 커넥션 유지 시간 (ms)
      max-lifetime: 1800000      # 커넥션 최대 수명 (ms)
```

**실무 팁:** `maximum-pool-size`는 CPU 코어 수 × 2 + 유효 디스크 수를 기준으로 잡는다. 무조건 크게 설정하면 오히려 컨텍스트 스위칭 비용이 늘어난다.

---

### 캐시 전략 (Redis 활용)

자주 읽히고 잘 안 바뀌는 데이터는 Redis에 캐싱해서 DB 부하를 줄인다.

```
Cache-Aside 패턴 (가장 일반적):

1. 클라이언트 → 캐시 조회
2. 캐시 HIT  → 바로 반환
3. 캐시 MISS → DB 조회 → 캐시 저장 → 반환
```

```java
// Spring Cache + Redis 예시
@Cacheable(value = "users", key = "#id")
public User getUser(Long id) {
    return userRepository.findById(id)
        .orElseThrow(() -> new EntityNotFoundException("사용자 없음"));
}

@CacheEvict(value = "users", key = "#user.id")
public User updateUser(User user) {
    return userRepository.save(user);
}
```

---

## 2. 인프라

### 서버 구성 기본 패턴

```
[클라이언트]
     │
     ▼
[로드 밸런서] (L4/L7)
     ├──> [앱 서버 1]
     ├──> [앱 서버 2]  ──> [DB Primary]
     └──> [앱 서버 3]       └──> [DB Replica]
```

- **로드 밸런서**: 트래픽을 여러 서버로 분산. L4(TCP)는 빠르고, L7(HTTP)는 경로 기반 라우팅 가능
- **DB 이중화**: Primary에 쓰고, Replica에서 읽음. 읽기 트래픽 분산 + 장애 대비
- **무중단 배포**: Blue-Green, Rolling, Canary 배포로 서비스 중단 없이 업데이트

---

### Docker와 컨테이너

```dockerfile
# 멀티 스테이지 빌드 예시 (이미지 용량 최소화)
FROM gradle:8-jdk17 AS builder
WORKDIR /app
COPY . .
RUN gradle build -x test

FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```yaml
# docker-compose.yml 기본 구조
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
    depends_on:
      - db
      - redis

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: mydb
    volumes:
      - mysql_data:/var/lib/mysql

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  mysql_data:
```

**실무 팁:**
- 애플리케이션 설정값(비밀번호, API 키)은 절대 이미지에 넣지 않는다. 환경 변수나 Secret Manager로 주입한다.
- 멀티 스테이지 빌드를 쓰면 최종 이미지에 빌드 도구가 포함되지 않아 용량이 줄어든다.

---

### Nginx 리버스 프록시

Nginx는 리버스 프록시, 로드 밸런서, 정적 파일 서빙을 모두 처리할 수 있어 실무에서 자주 쓰인다.

```nginx
# /etc/nginx/nginx.conf 핵심 설정

upstream app_servers {
    server app1:8080;
    server app2:8080;
    server app3:8080;
}

server {
    listen 80;
    server_name example.com;

    # HTTP → HTTPS 리다이렉트
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/ssl/cert.pem;
    ssl_certificate_key /etc/ssl/key.pem;

    # 리버스 프록시
    location /api/ {
        proxy_pass http://app_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # 정적 파일 직접 서빙
    location /static/ {
        root /var/www;
        expires 30d;
    }
}
```

---

## 3. 클라우드 (AWS 기준)

### 핵심 서비스 요약

| 서비스 | 역할 | 실무 사용 예 |
|---|---|---|
| EC2 | 가상 서버 | 앱 서버, 배치 서버 |
| RDS | 관리형 DB | MySQL, PostgreSQL 운영 |
| ElastiCache | 관리형 캐시 | Redis 클러스터 |
| S3 | 객체 스토리지 | 이미지, 파일 업로드 |
| CloudFront | CDN | 정적 자원 빠른 전달 |
| ECS/EKS | 컨테이너 오케스트레이션 | Docker 컨테이너 운영 |
| ALB | L7 로드 밸런서 | 경로 기반 라우팅 |
| Route 53 | DNS | 도메인 관리, 헬스체크 |
| IAM | 권한 관리 | 서비스 간 권한 제어 |
| CloudWatch | 모니터링 | 로그, 메트릭, 알람 |

---

### VPC 구성 기본 패턴

```
[인터넷]
    │
[Internet Gateway]
    │
[Public Subnet]          [Private Subnet]
  - ALB (로드밸런서)  →    - EC2 앱 서버
  - Bastion Host      →    - RDS (DB)
                      →    - ElastiCache (Redis)
```

- **Public Subnet**: 인터넷 직접 접근 가능. ALB, Bastion 호스트만 둔다.
- **Private Subnet**: 인터넷 직접 접근 불가. 앱 서버, DB는 여기에 위치시켜 보안을 강화한다.
- **NAT Gateway**: Private Subnet의 서버가 외부 API를 호출할 때 필요 (단방향 아웃바운드)

---

### S3 파일 업로드 패턴

클라이언트가 파일을 직접 서버로 올리면 서버 부하가 커진다. Presigned URL을 쓰면 클라이언트가 S3에 직접 업로드하게 할 수 있다.

```
일반 업로드:
클라이언트 → 서버 → S3  (서버 부하 큼)

Presigned URL 방식:
1. 클라이언트 → 서버: "업로드 URL 줘"
2. 서버 → S3: Presigned URL 발급 요청
3. 서버 → 클라이언트: URL 전달
4. 클라이언트 → S3: 직접 업로드  (서버 부하 없음)
```

```java
// AWS SDK v2 - Presigned URL 발급 예시
S3Presigner presigner = S3Presigner.create();

PutObjectPresignRequest presignRequest = PutObjectPresignRequest.builder()
    .signatureDuration(Duration.ofMinutes(10))
    .putObjectRequest(r -> r
        .bucket("my-bucket")
        .key("uploads/" + fileName)
        .contentType("image/jpeg"))
    .build();

PresignedPutObjectRequest presignedRequest = presigner.presignPutObject(presignRequest);
String uploadUrl = presignedRequest.url().toString();
```

---

### CI/CD 파이프라인 기본 구조

```
코드 Push (GitHub)
    │
    ▼
CI (GitHub Actions)
    ├── 테스트 실행
    ├── 빌드
    └── Docker 이미지 빌드 & ECR 푸시
    │
    ▼
CD (자동 배포)
    └── ECS / EC2에 새 이미지 배포
```

```yaml
# .github/workflows/deploy.yml 핵심 구조
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-2

      - name: Build and push Docker image
        run: |
          docker build -t my-app .
          docker tag my-app:latest $ECR_REGISTRY/my-app:latest
          docker push $ECR_REGISTRY/my-app:latest

      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster my-cluster \
            --service my-service \
            --force-new-deployment
```

---

### 모니터링과 알람

서비스가 죽었을 때 내가 먼저 알아야 한다. 사용자가 먼저 알면 이미 늦다.

```
모니터링 기본 지표:
- CPU 사용률 > 80% → 스케일 아웃 검토
- 메모리 사용률 > 85% → OOM 위험
- DB 커넥션 수 > 최대치의 80% → 커넥션 풀 점검
- 응답 시간(P95) > 1000ms → 쿼리/로직 최적화 필요
- 5xx 에러율 > 1% → 즉시 조사 필요
```

```yaml
# CloudWatch 알람 예시 (Terraform)
resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "high-cpu-utilization"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 120
  statistic           = "Average"
  threshold           = 80
  alarm_actions       = [aws_sns_topic.alerts.arn]
}
```

---

## 마치며

DB, 인프라, 클라우드는 각각 독립된 영역처럼 보이지만 실무에서는 항상 함께 움직인다. 쿼리가 느린 건 인덱스 문제일 수도 있고, 커넥션 풀 설정 문제일 수도 있고, 인프라 스펙 문제일 수도 있다. 각 영역의 기본 원리를 이해하고 있어야 장애 상황에서 빠르게 원인을 좁혀나갈 수 있다. 이 글에서 다룬 내용을 뼈대로 삼고, 필요한 부분을 깊게 파고드는 방식으로 학습하는 걸 추천한다.
