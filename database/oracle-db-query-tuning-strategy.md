# Oracle DB 성능을 죽이는 쿼리 패턴과 튜닝 전략

Oracle DB에서 성능 저하는 대부분 잘못된 쿼리 패턴에서 비롯됨. 문제 패턴을 정확히 이해하고, 체계적인 튜닝 절차를 따르면 대부분의 병목을 해결할 수 있음.

---

## 1. DB 성능에 부정적인 영향을 주는 쿼리 패턴

### 1-1. 인덱스를 무력화하는 컬럼 가공

인덱스가 걸린 컬럼에 함수나 연산을 적용하면 Full Table Scan이 발생함.

```sql
-- 나쁜 예: 인덱스 무력화
SELECT * FROM orders WHERE TO_CHAR(order_date, 'YYYY') = '2024';
SELECT * FROM users WHERE UPPER(username) = 'ADMIN';
SELECT * FROM products WHERE price * 1.1 > 10000;

-- 좋은 예: 컬럼 가공 없이 범위 조건으로 처리
SELECT * FROM orders
WHERE order_date >= DATE '2024-01-01'
  AND order_date < DATE '2025-01-01';

-- Function-Based Index로 해결 가능
CREATE INDEX idx_upper_username ON users (UPPER(username));
SELECT * FROM users WHERE UPPER(username) = 'ADMIN';
```

### 1-2. 묵시적 형변환

데이터 타입 불일치 시 Oracle이 자동으로 형변환을 수행하며 인덱스를 타지 못함.

```sql
-- 나쁜 예: 컬럼이 VARCHAR2인데 숫자로 비교
SELECT * FROM employees WHERE emp_no = 1001;

-- 나쁜 예: 컬럼이 NUMBER인데 문자열로 비교
SELECT * FROM employees WHERE dept_id = '10';

-- 좋은 예: 타입을 명시적으로 일치시킴
SELECT * FROM employees WHERE emp_no = '1001';  -- VARCHAR2 컬럼
SELECT * FROM employees WHERE dept_id = 10;     -- NUMBER 컬럼
```

### 1-3. LIKE 앞자리 와일드카드

`LIKE '%value'` 패턴은 인덱스를 전혀 활용하지 못하고 Full Scan을 유발함.

```sql
-- 나쁜 예: 앞자리 와일드카드 → Full Table Scan 불가피
SELECT * FROM customers WHERE cust_name LIKE '%홍길동';
SELECT * FROM products WHERE prod_code LIKE '%ABC%';

-- 좋은 예: 뒤에만 와일드카드
SELECT * FROM customers WHERE cust_name LIKE '홍%';

-- 전문 검색이 필요하면 Oracle Text 사용
CREATE INDEX idx_cust_name_ctx ON customers (cust_name)
  INDEXTYPE IS CTXSYS.CONTEXT;
SELECT * FROM customers WHERE CONTAINS(cust_name, '홍길동') > 0;
```

### 1-4. SELECT *

불필요한 컬럼까지 모두 읽어 I/O 낭비가 발생하고, 커버링 인덱스 활용도 불가능해짐.

```sql
-- 나쁜 예
SELECT * FROM orders WHERE order_date >= SYSDATE - 30;

-- 좋은 예: 필요한 컬럼만 명시
SELECT order_id, customer_id, order_date, total_amount
FROM orders
WHERE order_date >= SYSDATE - 30;
```

### 1-5. 불필요한 DISTINCT / 서브쿼리 중복 실행

```sql
-- 나쁜 예: 반복 서브쿼리
SELECT emp_name,
       (SELECT dept_name FROM departments WHERE dept_id = e.dept_id) AS dept,
       (SELECT location FROM departments WHERE dept_id = e.dept_id) AS loc
FROM employees e;

-- 좋은 예: JOIN으로 단일 처리
SELECT e.emp_name, d.dept_name, d.location
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id;
```

### 1-6. NL Join 대신 Cartesian Product 유발

```sql
-- 나쁜 예: WHERE 조인 조건 누락 → Cartesian Product
SELECT e.emp_name, d.dept_name
FROM employees e, departments d;

-- 좋은 예
SELECT e.emp_name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id;
```

---

## 2. DB 튜닝 빌드업 (단계적 접근)

튜닝은 문제 식별 → 원인 분석 → 개선 적용 → 검증의 순서로 진행함.

### Step 1. 문제 쿼리 식별

```sql
-- V$SQL: 누적 실행 통계 기준으로 상위 쿼리 추출
SELECT sql_id,
       executions,
       elapsed_time / 1000000 AS elapsed_sec,
       elapsed_time / NULLIF(executions, 0) / 1000000 AS avg_sec,
       disk_reads,
       buffer_gets,
       SUBSTR(sql_text, 1, 100) AS sql_preview
FROM v$sql
WHERE executions > 0
ORDER BY elapsed_time DESC
FETCH FIRST 20 ROWS ONLY;

-- AWR: Snapshot 간 Top SQL (라이선스 필요)
SELECT *
FROM dba_hist_top_sql
WHERE snap_id BETWEEN :start_snap AND :end_snap;
```

### Step 2. 실행 계획 분석

```sql
-- EXPLAIN PLAN으로 실행 계획 확인
EXPLAIN PLAN FOR
SELECT e.emp_name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.hire_date >= DATE '2020-01-01';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- 실제 실행 계획 (실행 후 통계 포함)
SELECT /*+ GATHER_PLAN_STATISTICS */
       e.emp_name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.hire_date >= DATE '2020-01-01';

SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST')
);
```

**실행 계획에서 확인할 포인트:**
- `TABLE ACCESS FULL` → Full Scan 여부
- `COST`, `ROWS` → 옵티마이저 추정치와 실제 rows(A-Rows) 차이
- `NESTED LOOPS` vs `HASH JOIN` vs `MERGE JOIN` → 조인 방식 적합성

### Step 3. 통계 정보 최신화

```sql
-- 테이블 통계 수집
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    ownname   => 'SCHEMA_NAME',
    tabname   => 'EMPLOYEES',
    estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
    method_opt => 'FOR ALL COLUMNS SIZE AUTO',
    cascade    => TRUE
  );
END;
/

-- 스키마 전체 통계 수집
BEGIN
  DBMS_STATS.GATHER_SCHEMA_STATS(
    ownname => 'SCHEMA_NAME',
    estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE
  );
END;
/
```

### Step 4. 인덱스 전략 수립

```sql
-- 컬럼 선택도(Selectivity) 확인
SELECT column_name,
       num_distinct,
       num_rows,
       ROUND(num_distinct / num_rows * 100, 2) AS selectivity_pct
FROM user_tab_col_statistics
WHERE table_name = 'ORDERS'
ORDER BY selectivity_pct;

-- 복합 인덱스: 선택도 높은 컬럼을 앞에 배치
CREATE INDEX idx_orders_date_status
  ON orders (order_date, status);

-- 인덱스 사용 현황 모니터링
ALTER INDEX idx_orders_date_status MONITORING USAGE;

SELECT index_name, used, start_monitoring, end_monitoring
FROM v$object_usage
WHERE index_name = 'IDX_ORDERS_DATE_STATUS';
```

### Step 5. 힌트(Hint)로 실행 계획 유도 (최후 수단)

```sql
-- 인덱스 강제 사용
SELECT /*+ INDEX(e idx_emp_hire_date) */
       e.emp_name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.hire_date >= DATE '2020-01-01';

-- Full Scan 강제
SELECT /*+ FULL(o) */ order_id, total_amount
FROM orders o
WHERE status = 'COMPLETE';

-- 조인 방식 강제
SELECT /*+ USE_HASH(e d) */
       e.emp_name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id;
```

> 힌트는 통계 수집, 인덱스 정비 후에도 계획이 개선되지 않을 때 적용함. 유지보수 부담이 생기므로 최후 수단으로 사용.

---

## 3. DB 성능 개선 방법

### 3-1. 파티셔닝으로 대용량 테이블 분할

```sql
CREATE TABLE orders_partitioned (
  order_id     NUMBER,
  order_date   DATE,
  customer_id  NUMBER,
  total_amount NUMBER(12, 2)
)
PARTITION BY RANGE (order_date) (
  PARTITION p2023 VALUES LESS THAN (DATE '2024-01-01'),
  PARTITION p2024 VALUES LESS THAN (DATE '2025-01-01'),
  PARTITION p2025 VALUES LESS THAN (DATE '2026-01-01'),
  PARTITION p_max VALUES LESS THAN (MAXVALUE)
);

EXPLAIN PLAN FOR
SELECT * FROM orders_partitioned
WHERE order_date BETWEEN DATE '2024-01-01' AND DATE '2024-12-31';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### 3-2. Materialized View로 집계 쿼리 최적화

```sql
CREATE MATERIALIZED VIEW mv_monthly_sales
BUILD IMMEDIATE
REFRESH COMPLETE ON DEMAND
ENABLE QUERY REWRITE
AS
SELECT
  TRUNC(order_date, 'MM') AS sales_month,
  product_id,
  SUM(amount)             AS total_sales,
  COUNT(*)                AS order_cnt
FROM orders
GROUP BY TRUNC(order_date, 'MM'), product_id;

BEGIN
  DBMS_MVIEW.REFRESH('MV_MONTHLY_SALES', 'C');
END;
/
```

### 3-3. WITH절 (CTE)로 중복 연산 제거

```sql
WITH dept_stats AS (
  SELECT dept_id,
         AVG(salary) AS avg_sal,
         MAX(salary) AS max_sal,
         COUNT(*)    AS emp_cnt
  FROM employees
  GROUP BY dept_id
)
SELECT d.dept_name, s.avg_sal, s.max_sal, s.emp_cnt
FROM departments d
JOIN dept_stats s ON d.dept_id = s.dept_id;
```

### 3-4. 페이징 쿼리 최적화 (ROW_NUMBER 방식)

```sql
-- Oracle 12c 이상 권장
SELECT order_id, customer_id, order_date, total_amount
FROM orders
ORDER BY order_date DESC
OFFSET 10 ROWS FETCH NEXT 10 ROWS ONLY;
```

### 3-5. 바인드 변수로 하드 파싱 방지

```sql
-- 나쁜 예: 하드 파싱 유발
SELECT * FROM orders WHERE customer_id = 1001;
SELECT * FROM orders WHERE customer_id = 1002;

-- 좋은 예: 바인드 변수
SELECT * FROM orders WHERE customer_id = :cust_id;

-- 하드 파싱 비율 확인
SELECT sql_text, parse_calls, executions,
       ROUND(parse_calls / NULLIF(executions, 0) * 100, 1) AS hard_parse_ratio
FROM v$sql
WHERE sql_text LIKE '%orders%'
ORDER BY parse_calls DESC;
```

### 3-6. 결과 캐시 (Result Cache) 활용

```sql
SELECT /*+ RESULT_CACHE */
       dept_id, dept_name, location
FROM departments
WHERE active_flag = 'Y';

SELECT name, value
FROM v$result_cache_statistics
WHERE name IN ('Find Count', 'Found Count');
```

---

## 요약

| 문제 유형 | 원인 | 해결책 |
|---|---|---|
| Full Table Scan | 함수 적용, 묵시적 형변환, LIKE 앞자리 % | FBI 생성, 타입 일치, Oracle Text |
| 잘못된 실행 계획 | 통계 정보 미갱신 | DBMS_STATS 수행 |
| 반복 집계 쿼리 | 매번 전체 집계 | Materialized View |
| 하드 파싱 과다 | 리터럴 SQL | 바인드 변수 |
| 대용량 스캔 | 파티션 미적용 | Range/List 파티셔닝 |
| 중복 서브쿼리 | 동일 로직 반복 | WITH절(CTE), JOIN |

성능 튜닝은 실행 계획을 먼저 읽고, 문제의 근본 원인을 파악한 뒤 순차적으로 개선을 적용하는 게 핵심임.
