# 데이터베이스의 보안 및 성능 이슈 해결법

데이터베이스는 서비스의 핵심 자산이 모이는 곳이다. 그래서 보안과 성능은 선택이 아니라 설계 단계부터 고려해야 할 필수 요소다. 이 글에서는 실제로 자주 발생하는 보안 취약점과 성능 저하 원인을 쿼리 수준에서 살펴보고, 어떻게 개선할 수 있는지 정리한다.

---

## 1. 보안에 취약할 수밖에 없는 쿼리

### SQL Injection

가장 대표적인 DB 보안 취약점이다. 사용자 입력값을 그대로 쿼리에 이어붙이면 공격자가 쿼리 구조 자체를 조작할 수 있다.

```sql
-- 취약한 쿼리 (동적 문자열 연결)
SELECT * FROM users WHERE username = '" + userInput + "';

-- 공격 예시: userInput = ' OR '1'='1
-- 실제 실행되는 쿼리:
SELECT * FROM users WHERE username = '' OR '1'='1';
-- 결과: 모든 사용자 데이터가 반환됨
```

**해결책: Prepared Statement (파라미터화 쿼리)**

```sql
-- 안전한 쿼리
SELECT * FROM users WHERE username = ?;
-- 입력값은 데이터로만 처리되고, 쿼리 구조를 바꿀 수 없다
```

### 과도한 권한을 가진 계정으로 쿼리 실행

애플리케이션이 `root` 또는 `admin` 계정으로 DB에 접근하는 경우, 쿼리 하나의 실수가 전체 데이터를 날릴 수 있다.

```sql
-- 위험: root 계정으로 일반 조회
CONNECT TO db AS root;
SELECT * FROM orders WHERE user_id = 42;

-- 만약 SQL Injection이 발생하면:
DROP TABLE orders; -- 실행 가능
```

**원칙:** 최소 권한 원칙(Least Privilege). 애플리케이션 계정에는 필요한 테이블에 대한 SELECT, INSERT, UPDATE 권한만 부여한다.

### 민감 정보 평문 저장

```sql
-- 위험: 비밀번호를 평문으로 저장
INSERT INTO users (username, password) VALUES ('alice', 'mypassword123');

-- 안전: 해시 처리 후 저장 (bcrypt 등 사용)
INSERT INTO users (username, password_hash) VALUES ('alice', '$2b$12$...');
```

---

## 2. 성능을 떨어뜨리는 쿼리

### N+1 문제

루프 안에서 쿼리를 반복 실행하는 패턴이다. 데이터가 많아질수록 쿼리 횟수가 선형으로 늘어난다.

```sql
-- 문제: 주문 목록 조회 후 각 주문마다 사용자 정보 조회 (N+1)
SELECT * FROM orders; -- 1번
-- 각 row마다:
SELECT * FROM users WHERE id = ?; -- N번 반복

-- 해결: JOIN으로 한 번에 가져오기
SELECT o.*, u.name, u.email
FROM orders o
JOIN users u ON o.user_id = u.id;
```

### 인덱스 없는 대용량 조회

```sql
-- 인덱스가 없는 컬럼으로 WHERE 필터링 → Full Table Scan 발생
SELECT * FROM logs WHERE created_at > '2026-01-01';

-- 인덱스 추가
CREATE INDEX idx_logs_created_at ON logs(created_at);
```

인덱스가 없으면 DB는 모든 행을 하나씩 읽어야 한다. 수백만 건 이상의 테이블에서 이 차이는 극적이다.

### SELECT * 사용

```sql
-- 나쁜 습관: 필요하지 않은 컬럼까지 전부 읽음
SELECT * FROM products;

-- 좋은 습관: 필요한 컬럼만 명시
SELECT id, name, price FROM products;
```

네트워크 전송량과 메모리 사용량이 줄어들고, 쿼리 실행 계획도 더 효율적으로 잡힌다.

### 정렬 + 페이지네이션 비효율

```sql
-- 느린 페이지네이션: OFFSET이 클수록 앞의 데이터를 모두 읽고 버림
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 10000;

-- 빠른 방법: Cursor-based pagination
SELECT * FROM posts
WHERE created_at < '2026-03-01 00:00:00'
ORDER BY created_at DESC
LIMIT 20;
```

---

## 3. 보안을 강화하는 쿼리 패턴

### Row-Level Security (RLS)

PostgreSQL에서는 테이블 단위가 아니라 행(row) 단위로 접근 제어가 가능하다.

```sql
-- RLS 활성화
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- 정책 정의: 본인 주문만 조회 가능
CREATE POLICY orders_self_only
ON orders
FOR SELECT
USING (user_id = current_user_id());
```

다중 테넌트 SaaS 구조에서 실수로 다른 사용자 데이터를 노출하는 것을 DB 레벨에서 막을 수 있다.

### 역할 기반 접근 제어 (RBAC)

```sql
-- 읽기 전용 역할 생성
CREATE ROLE readonly_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;

-- 특정 작업만 허용하는 역할
CREATE ROLE order_writer;
GRANT INSERT, UPDATE ON orders TO order_writer;
REVOKE DELETE ON orders FROM order_writer;
```

### 감사 로그 (Audit Log)

```sql
-- 민감한 테이블 변경 이력을 별도 테이블에 기록하는 트리거
CREATE OR REPLACE FUNCTION log_user_changes()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO audit_log (table_name, operation, changed_by, changed_at, old_data, new_data)
  VALUES (
    TG_TABLE_NAME,
    TG_OP,
    current_user,
    now(),
    row_to_json(OLD),
    row_to_json(NEW)
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_audit
AFTER UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION log_user_changes();
```

---

## 4. SaaS로 운영되는 DB 서비스의 장점과 한계

### 장점

**운영 부담 감소**가 가장 큰 이유다. 패치, 백업, 장애 복구, 스케일링을 서비스 제공자가 처리한다.

| 기능 | 자체 운영 | SaaS (RDS, Supabase 등) |
|------|-----------|------------------------|
| 백업 | 직접 구성 | 자동 (설정만) |
| 패치 | 직접 적용 | 자동 or 예약 |
| 스케일 업 | 서버 교체 | 콘솔 클릭 |
| HA/Failover | 별도 구성 | 내장 |
| 감사 로그 | 직접 구현 | 일부 기본 제공 |

보안 측면에서도 **네트워크 격리(VPC), TLS 강제, 암호화 at rest** 등이 기본 적용되는 경우가 많다.

### 한계

- **설정 자유도 제한**: `pg_hba.conf` 직접 편집, 특정 extension 설치, OS 레벨 튜닝이 불가능하거나 제한적이다.
- **벤더 종속(Vendor Lock-in)**: 특정 서비스의 고유 기능을 사용하면 마이그레이션 비용이 커진다.
- **비용**: 트래픽과 데이터가 커질수록 자체 운영 대비 비용이 급격히 올라갈 수 있다.
- **규정 준수**: 금융/의료 등 규제 산업에서는 데이터 위치(data residency)와 감사 요건을 직접 제어해야 해서 SaaS가 맞지 않는 경우도 있다.

---

## 5. 어떤 데이터베이스가 보안에 더 최적화되어 있을까

보안 기능의 충실도는 DB마다 차이가 크다.

### PostgreSQL

현재 오픈소스 DB 중 보안 기능이 가장 풍부하다.

- **Row-Level Security (RLS)** 네이티브 지원
- **컬럼 단위 암호화** (`pgcrypto` extension)
- **역할 기반 접근 제어** 세밀하게 구성 가능
- **SSL/TLS** 강제 설정 가능
- Supabase가 PostgreSQL 기반인 이유 중 하나가 RLS 때문이다

### MySQL / MariaDB

- RLS는 네이티브 미지원 → 애플리케이션 레벨에서 처리해야 함
- 권한 시스템은 충실하지만 PostgreSQL보다 세밀함이 떨어짐
- `Enterprise Edition`에서는 감사 플러그인 제공 (커뮤니티 버전은 별도 설치)

### Microsoft SQL Server

- **Always Encrypted**: 클라이언트 단에서 암호화되어 DB 관리자도 평문 조회 불가
- **Dynamic Data Masking**: 쿼리 결과에서 민감 컬럼을 자동 마스킹
- **Transparent Data Encryption (TDE)**: 디스크 레벨 암호화 기본 내장
- 엔터프라이즈 환경에서 보안 컴플라이언스 요건 충족이 용이함

### Oracle

- **VPD (Virtual Private Database)**: RLS와 유사한 행 수준 접근 제어
- **Oracle Audit Vault**: 전문 감사 솔루션
- 금융권에서 아직 많이 쓰이는 이유가 이런 엔터프라이즈 보안 기능 때문이다
- 단, 라이선스 비용이 매우 높다

### 요약

| DB | RLS | 컬럼 암호화 | 동적 마스킹 | 감사 로그 |
|----|-----|------------|------------|----------|
| PostgreSQL | ✅ 네이티브 | ✅ (pgcrypto) | ❌ (직접 구현) | ✅ (직접 구현) |
| MySQL | ❌ | 제한적 | ❌ | Enterprise만 |
| MSSQL | 유사 기능 | ✅ Always Encrypted | ✅ | ✅ |
| Oracle | ✅ VPD | ✅ | ✅ | ✅ 전문 솔루션 |

---

보안은 한 번 설정하고 끝나는 게 아니다. 쿼리 작성 습관, 권한 설계, 감사 추적, DB 선택까지 모두 연결된 문제다. 특히 SaaS 서비스를 만들 때는 처음부터 RLS 같은 DB 레벨 보안을 적용하면 애플리케이션 코드에서 발생할 수 있는 실수를 원천 차단할 수 있다.
