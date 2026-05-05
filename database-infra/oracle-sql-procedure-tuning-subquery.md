# Oracle SQL 문법, 프로시저, 튜닝, 서브쿼리 분석 완전 정리

## 개요

Oracle DB는 엔터프라이즈 환경에서 가장 널리 사용되는 RDBMS 중 하나다. 단순한 SELECT 문법을 넘어서, 실무에서는 고급 쿼리 최적화, 프로시저 작성, 실행 계획 분석 능력이 필수다. 이 글에서는 Oracle의 핵심 SQL 문법부터 PL/SQL 프로시저, 쿼리 튜닝, 서브쿼리 분석까지 실무 중심으로 정리한다.

---

## 1. Oracle SQL 핵심 문법

### 1-1. 기본 SELECT 및 Oracle 전용 문법

```sql
-- ROWNUM: 결과 행 번호 부여 (페이징 처리에 활용)
SELECT *
FROM (
  SELECT ROWNUM AS rn, e.*
  FROM employees e
  WHERE ROWNUM <= 20
)
WHERE rn >= 11;

-- ROW_NUMBER() 윈도우 함수 활용 (12c 이상에서는 FETCH FIRST 사용 가능)
SELECT *
FROM employees
ORDER BY salary DESC
FETCH FIRST 10 ROWS ONLY;
```

> **ROWNUM vs ROW_NUMBER()**: ROWNUM은 WHERE 절 필터 이전에 부여되기 때문에, 정렬 후 페이징하려면 반드시 서브쿼리로 감싸야 한다.

### 1-2. NULL 처리

```sql
-- NVL: NULL이면 대체값 반환
SELECT NVL(commission_pct, 0) FROM employees;

-- NVL2: NULL 여부에 따라 다른 값 반환
SELECT NVL2(commission_pct, '있음', '없음') FROM employees;

-- NULLIF: 두 값이 같으면 NULL 반환
SELECT NULLIF(salary, 0) FROM employees;

-- COALESCE: 첫 번째 NULL이 아닌 값 반환 (ANSI 표준)
SELECT COALESCE(bonus, commission_pct, 0) FROM employees;
```

### 1-3. 날짜 함수

```sql
-- 오늘 날짜
SELECT SYSDATE FROM DUAL;

-- 날짜 연산
SELECT SYSDATE + 7 AS next_week FROM DUAL;

-- MONTHS_BETWEEN: 두 날짜 간 개월 수
SELECT MONTHS_BETWEEN(SYSDATE, hire_date) AS months_worked
FROM employees;

-- TRUNC: 날짜 절사
SELECT TRUNC(SYSDATE, 'MM') AS first_of_month FROM DUAL;

-- TO_DATE / TO_CHAR
SELECT TO_DATE('2024-01-15', 'YYYY-MM-DD') FROM DUAL;
SELECT TO_CHAR(SYSDATE, 'YYYY-MM-DD HH24:MI:SS') FROM DUAL;
```

### 1-4. 집계 및 분석 함수

```sql
-- 윈도우 함수: 부서별 급여 순위
SELECT
  employee_id,
  department_id,
  salary,
  RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dept_rank,
  DENSE_RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dense_rank,
  ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary DESC) AS row_num
FROM employees;

-- LAG / LEAD: 이전/다음 행 참조
SELECT
  employee_id,
  salary,
  LAG(salary, 1, 0) OVER (ORDER BY hire_date) AS prev_salary,
  LEAD(salary, 1, 0) OVER (ORDER BY hire_date) AS next_salary
FROM employees;

-- SUM 누적 합계
SELECT
  employee_id,
  salary,
  SUM(salary) OVER (ORDER BY hire_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cumulative_salary
FROM employees;
```

### 1-5. PIVOT / UNPIVOT

```sql
-- PIVOT: 행을 열로 변환
SELECT *
FROM (
  SELECT department_id, job_id, salary
  FROM employees
)
PIVOT (
  AVG(salary)
  FOR job_id IN ('IT_PROG' AS it_prog, 'SA_REP' AS sa_rep, 'MK_MAN' AS mk_man)
);

-- UNPIVOT: 열을 행으로 변환
SELECT *
FROM quarterly_sales
UNPIVOT (
  sales_amount
  FOR quarter IN (q1, q2, q3, q4)
);
```

---

## 2. PL/SQL 프로시저

### 2-1. 기본 구조

```sql
CREATE OR REPLACE PROCEDURE procedure_name (
  p_param1 IN  VARCHAR2,
  p_param2 OUT NUMBER,
  p_param3 IN OUT DATE
) IS
  -- 지역 변수 선언
  v_local_var VARCHAR2(100);
BEGIN
  -- 로직
  v_local_var := p_param1 || '_processed';
  p_param2 := LENGTH(v_local_var);
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    DBMS_OUTPUT.PUT_LINE('데이터 없음');
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('오류: ' || SQLERRM);
END procedure_name;
/
```

> **IN / OUT / IN OUT 파라미터**
> - `IN`: 읽기 전용 입력값
> - `OUT`: 호출자에게 값 반환
> - `IN OUT`: 입력받아 수정 후 반환

### 2-2. 커서(Cursor) 활용

```sql
CREATE OR REPLACE PROCEDURE process_employees IS
  -- 명시적 커서 선언
  CURSOR emp_cursor IS
    SELECT employee_id, salary
    FROM employees
    WHERE department_id = 60;

  v_emp_id employees.employee_id%TYPE;
  v_salary  employees.salary%TYPE;
BEGIN
  OPEN emp_cursor;
  LOOP
    FETCH emp_cursor INTO v_emp_id, v_salary;
    EXIT WHEN emp_cursor%NOTFOUND;

    -- 처리 로직
    IF v_salary < 5000 THEN
      UPDATE employees
      SET salary = salary * 1.1
      WHERE employee_id = v_emp_id;
    END IF;
  END LOOP;
  CLOSE emp_cursor;
  COMMIT;
END;
/
```

```sql
-- FOR 루프로 간결하게 커서 처리
BEGIN
  FOR rec IN (SELECT employee_id, salary FROM employees WHERE department_id = 60) LOOP
    DBMS_OUTPUT.PUT_LINE('ID: ' || rec.employee_id || ', 급여: ' || rec.salary);
  END LOOP;
END;
/
```

### 2-3. 패키지(Package)

```sql
-- 패키지 스펙 (인터페이스 정의)
CREATE OR REPLACE PACKAGE emp_pkg IS
  g_dept_id NUMBER := 60;  -- 패키지 전역 변수

  PROCEDURE get_emp_info(p_emp_id IN NUMBER);
  FUNCTION calc_bonus(p_salary IN NUMBER) RETURN NUMBER;
END emp_pkg;
/

-- 패키지 바디 (구현부)
CREATE OR REPLACE PACKAGE BODY emp_pkg IS

  PROCEDURE get_emp_info(p_emp_id IN NUMBER) IS
    v_name VARCHAR2(100);
  BEGIN
    SELECT first_name || ' ' || last_name
    INTO v_name
    FROM employees
    WHERE employee_id = p_emp_id;
    DBMS_OUTPUT.PUT_LINE('직원명: ' || v_name);
  EXCEPTION
    WHEN NO_DATA_FOUND THEN
      DBMS_OUTPUT.PUT_LINE('해당 직원 없음');
  END get_emp_info;

  FUNCTION calc_bonus(p_salary IN NUMBER) RETURN NUMBER IS
  BEGIN
    RETURN p_salary * 0.15;
  END calc_bonus;

END emp_pkg;
/

-- 사용
EXECUTE emp_pkg.get_emp_info(100);
SELECT emp_pkg.calc_bonus(8000) FROM DUAL;
```

### 2-4. 예외 처리

```sql
DECLARE
  -- 사용자 정의 예외
  e_invalid_salary EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_invalid_salary, -20001);

  v_salary NUMBER := -100;
BEGIN
  IF v_salary < 0 THEN
    RAISE_APPLICATION_ERROR(-20001, '급여는 0 이상이어야 함');
  END IF;
EXCEPTION
  WHEN e_invalid_salary THEN
    DBMS_OUTPUT.PUT_LINE('사용자 예외 처리: ' || SQLERRM);
  WHEN ZERO_DIVIDE THEN
    DBMS_OUTPUT.PUT_LINE('0으로 나누기 오류');
  WHEN OTHERS THEN
    ROLLBACK;
    DBMS_OUTPUT.PUT_LINE('예상치 못한 오류: ' || SQLCODE || ' - ' || SQLERRM);
END;
/
```

### 2-5. 트리거(Trigger)

```sql
CREATE OR REPLACE TRIGGER audit_salary_change
BEFORE UPDATE OF salary ON employees
FOR EACH ROW
BEGIN
  IF :NEW.salary < :OLD.salary THEN
    RAISE_APPLICATION_ERROR(-20002, '급여 삭감 불가');
  END IF;

  INSERT INTO salary_audit_log (
    employee_id, old_salary, new_salary, changed_at, changed_by
  ) VALUES (
    :OLD.employee_id, :OLD.salary, :NEW.salary, SYSDATE, USER
  );
END;
/
```

---

## 3. 쿼리 튜닝

### 3-1. 실행 계획(Execution Plan) 읽기

```sql
-- EXPLAIN PLAN 생성
EXPLAIN PLAN FOR
SELECT e.employee_id, e.last_name, d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id
WHERE e.salary > 10000;

-- 실행 계획 조회
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

```
------------------------------------------------------------------------------------
| Id | Operation                    | Name        | Rows | Bytes | Cost (%CPU) |
------------------------------------------------------------------------------------
|  0 | SELECT STATEMENT             |             |   10 |   390 |     5   (0) |
|  1 |  NESTED LOOPS                |             |   10 |   390 |     5   (0) |
|  2 |   TABLE ACCESS BY INDEX ROWID| EMPLOYEES   |   10 |   260 |     2   (0) |
|  3 |    INDEX RANGE SCAN          | EMP_SAL_IDX |   10 |       |     1   (0) |
|  4 |   TABLE ACCESS BY INDEX ROWID| DEPARTMENTS |    1 |    13 |     1   (0) |
|  5 |    INDEX UNIQUE SCAN         | DEPT_PK     |    1 |       |     0   (0) |
------------------------------------------------------------------------------------
```

> **핵심 오퍼레이션 해석**
> - `FULL TABLE SCAN`: 인덱스 미사용, 전체 테이블 스캔 → 대용량 테이블에서 성능 저하
> - `INDEX RANGE SCAN`: 범위 조건으로 인덱스 스캔 → 일반적으로 효율적
> - `INDEX UNIQUE SCAN`: PK/유니크 인덱스 스캔 → 가장 효율적
> - `NESTED LOOPS`: 소용량 조인에 유리
> - `HASH JOIN`: 대용량 조인에 유리
> - `MERGE JOIN`: 정렬된 데이터 조인에 유리

### 3-2. 인덱스 튜닝

```sql
-- 단일 컬럼 인덱스
CREATE INDEX idx_emp_salary ON employees(salary);

-- 복합 인덱스 (선두 컬럼 순서 중요)
CREATE INDEX idx_emp_dept_sal ON employees(department_id, salary);

-- 함수 기반 인덱스
CREATE INDEX idx_emp_upper_name ON employees(UPPER(last_name));

-- 인덱스 조회
SELECT index_name, column_name, column_position
FROM user_ind_columns
WHERE table_name = 'EMPLOYEES'
ORDER BY index_name, column_position;
```

**인덱스가 무시되는 경우**

```sql
-- ❌ 인덱스 컬럼에 함수 적용
WHERE UPPER(last_name) = 'KING'
-- ✅ 함수 기반 인덱스 생성 or 데이터 정규화

-- ❌ 묵시적 형변환
WHERE employee_id = '100'  -- employee_id가 NUMBER 타입
-- ✅ 명시적 형변환
WHERE employee_id = 100

-- ❌ LIKE 앞부분 와일드카드
WHERE last_name LIKE '%ing'
-- ✅ 뒤에만 와일드카드
WHERE last_name LIKE 'King%'

-- ❌ NULL 비교 (B-Tree 인덱스는 NULL 미저장)
WHERE commission_pct IS NULL
-- ✅ 비트맵 인덱스 또는 복합 인덱스 고려
```

### 3-3. 힌트(Hint) 사용

```sql
-- 힌트는 옵티마이저에게 실행 계획을 강제 지정
SELECT /*+ INDEX(e idx_emp_salary) */ *
FROM employees e
WHERE salary > 10000;

-- 조인 방법 강제
SELECT /*+ USE_HASH(e d) */ e.last_name, d.department_name
FROM employees e, departments d
WHERE e.department_id = d.department_id;

-- FULL TABLE SCAN 강제
SELECT /*+ FULL(e) */ * FROM employees e;

-- 병렬 처리
SELECT /*+ PARALLEL(e, 4) */ * FROM employees e;
```

> 힌트는 SQL 내부에 주석 형태로 작성되며, 잘못된 힌트는 무시된다. 튜닝 목적 외에는 남용 금지.

### 3-4. 통계 정보 관리

```sql
-- 테이블 통계 수집
EXEC DBMS_STATS.GATHER_TABLE_STATS(
  ownname => 'HR',
  tabname => 'EMPLOYEES',
  estimate_percent => 30,
  method_opt => 'FOR ALL COLUMNS SIZE AUTO',
  cascade => TRUE
);

-- 통계 정보 확인
SELECT table_name, num_rows, last_analyzed
FROM user_tables
WHERE table_name = 'EMPLOYEES';
```

> 통계 정보가 오래되면 옵티마이저가 잘못된 실행 계획을 선택한다. 대량 DML 후에는 반드시 통계를 재수집해야 한다.

### 3-5. SQL 튜닝 체크리스트

| 항목 | 확인 사항 |
|------|-----------|
| SELECT | 필요한 컬럼만 명시, SELECT * 지양 |
| WHERE | 인덱스 활용 가능한 조건 우선 배치 |
| JOIN | 드라이빙 테이블 선택, 조인 순서 최적화 |
| 서브쿼리 | 가능하면 JOIN 또는 인라인 뷰로 변환 |
| 인덱스 | 선두 컬럼 조건 충족 여부 확인 |
| 통계 | 최신 통계 정보 유지 |
| 파티셔닝 | 대용량 테이블 파티션 프루닝 활용 |

---

## 4. 서브쿼리 분석 및 최적화

### 4-1. 서브쿼리 종류

```sql
-- 단일 행 서브쿼리
SELECT last_name, salary
FROM employees
WHERE salary = (
  SELECT MAX(salary) FROM employees
);

-- 다중 행 서브쿼리 (IN, ANY, ALL 사용)
SELECT last_name, salary
FROM employees
WHERE department_id IN (
  SELECT department_id
  FROM departments
  WHERE location_id = 1700
);

-- 상관 서브쿼리 (외부 쿼리 참조)
SELECT last_name, salary, department_id
FROM employees e
WHERE salary > (
  SELECT AVG(salary)
  FROM employees
  WHERE department_id = e.department_id
);
```

### 4-2. 인라인 뷰 (Inline View)

```sql
-- FROM 절의 서브쿼리 = 인라인 뷰
SELECT a.department_id, a.avg_salary, b.department_name
FROM (
  SELECT department_id, AVG(salary) AS avg_salary
  FROM employees
  GROUP BY department_id
) a
JOIN departments b ON a.department_id = b.department_id
WHERE a.avg_salary > 8000;
```

### 4-3. WITH 절 (CTE, Common Table Expression)

```sql
WITH
  dept_avg AS (
    SELECT department_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
  ),
  high_dept AS (
    SELECT department_id
    FROM dept_avg
    WHERE avg_salary > 8000
  )
SELECT e.last_name, e.salary, e.department_id
FROM employees e
JOIN high_dept h ON e.department_id = h.department_id
ORDER BY e.department_id, e.salary DESC;
```

> **WITH 절 장점**: 동일 서브쿼리 반복 방지, 가독성 향상. Oracle은 WITH 절을 임시 테이블처럼 재사용 (MATERIALIZE 힌트 적용 가능).

### 4-4. EXISTS vs IN 성능 비교

```sql
-- IN: 서브쿼리 전체 결과를 메모리에 올려 비교
SELECT last_name FROM employees
WHERE department_id IN (
  SELECT department_id FROM departments WHERE location_id = 1700
);

-- EXISTS: 조건 만족하는 첫 번째 행 발견 시 즉시 반환 (Short-circuit)
SELECT last_name FROM employees e
WHERE EXISTS (
  SELECT 1 FROM departments d
  WHERE d.department_id = e.department_id
  AND d.location_id = 1700
);
```

| 비교 | IN | EXISTS |
|------|-------|--------|
| 서브쿼리 NULL | NULL 포함 시 결과 없음 | 영향 없음 |
| 대용량 외부 테이블 | 불리 | 유리 |
| 대용량 서브쿼리 결과 | 불리 | 유리 (Short-circuit) |
| 인덱스 활용 | 제한적 | 상관 서브쿼리 인덱스 활용 가능 |

### 4-5. 서브쿼리 → JOIN 변환

```sql
-- ❌ 상관 서브쿼리 (행마다 서브쿼리 실행 → 성능 저하)
SELECT e.last_name,
       (SELECT d.department_name FROM departments d WHERE d.department_id = e.department_id) AS dept_name
FROM employees e;

-- ✅ JOIN으로 변환 (1회 조인)
SELECT e.last_name, d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id;
```

### 4-6. 실행 계획 분석 도구

```sql
-- V$SQL: 실행된 SQL 및 실행 계획 통계 조회
SELECT sql_id, executions, elapsed_time, cpu_time, buffer_gets, sql_text
FROM V$SQL
WHERE sql_text LIKE '%employees%'
ORDER BY elapsed_time DESC;

-- AWR 기반 TOP SQL 조회 (DBA 권한 필요)
SELECT sql_id, plan_hash_value, elapsed_time_total, executions_total
FROM DBA_HIST_SQLSTAT
ORDER BY elapsed_time_total DESC
FETCH FIRST 10 ROWS ONLY;
```

---

## 5. 실무 적용 패턴

### 5-1. 페이징 처리

```sql
-- Oracle 12c 이상: OFFSET-FETCH 방식
SELECT employee_id, last_name, salary
FROM employees
ORDER BY salary DESC
OFFSET 10 ROWS FETCH NEXT 10 ROWS ONLY;

-- 12c 미만: ROWNUM 이중 서브쿼리
SELECT *
FROM (
  SELECT ROWNUM rn, a.*
  FROM (
    SELECT employee_id, last_name, salary
    FROM employees
    ORDER BY salary DESC
  ) a
  WHERE ROWNUM <= 20
)
WHERE rn >= 11;
```

### 5-2. MERGE 문 (Upsert)

```sql
MERGE INTO employees_backup t
USING employees s
ON (t.employee_id = s.employee_id)
WHEN MATCHED THEN
  UPDATE SET t.salary = s.salary, t.updated_at = SYSDATE
WHEN NOT MATCHED THEN
  INSERT (employee_id, last_name, salary, created_at)
  VALUES (s.employee_id, s.last_name, s.salary, SYSDATE);
```

### 5-3. 대량 데이터 처리 (BULK COLLECT + FORALL)

```sql
DECLARE
  TYPE emp_id_tab IS TABLE OF employees.employee_id%TYPE;
  TYPE salary_tab IS TABLE OF employees.salary%TYPE;
  v_emp_ids   emp_id_tab;
  v_salaries  salary_tab;
BEGIN
  -- BULK COLLECT: 한 번에 대량 데이터를 컬렉션에 담기
  SELECT employee_id, salary
  BULK COLLECT INTO v_emp_ids, v_salaries
  FROM employees
  WHERE department_id = 60;

  -- FORALL: 컬렉션 기반 일괄 DML (컨텍스트 스위칭 최소화)
  FORALL i IN 1..v_emp_ids.COUNT
    UPDATE employees
    SET salary = v_salaries(i) * 1.05
    WHERE employee_id = v_emp_ids(i);

  COMMIT;
  DBMS_OUTPUT.PUT_LINE(v_emp_ids.COUNT || '건 처리 완료');
END;
/
```

> **BULK COLLECT + FORALL**: 행 단위 처리 대비 수십~수백 배 성능 향상 가능. 컨텍스트 스위칭(PL/SQL ↔ SQL 엔진) 횟수를 줄이는 것이 핵심.

---

## 6. 영어로 배우는 Oracle 핵심 용어

| 한국어 | English Term | 설명 |
|--------|-------------|------|
| 실행 계획 | Execution Plan | Query optimizer's chosen strategy |
| 옵티마이저 | CBO (Cost-Based Optimizer) | Chooses plan based on statistics |
| 인덱스 범위 스캔 | Index Range Scan | Scans a range of index entries |
| 전체 테이블 스캔 | Full Table Scan (FTS) | Reads every block in a table |
| 힌트 | Optimizer Hint | Directive to override optimizer |
| 바인드 변수 | Bind Variable | Prevents hard parsing, improves reuse |
| 커서 공유 | Cursor Sharing | Reuse of parsed SQL in shared pool |
| 파티션 프루닝 | Partition Pruning | Skip irrelevant partitions |
| 상관 서브쿼리 | Correlated Subquery | References outer query columns |
| 누적 합계 | Running Total | Cumulative sum via window function |

---

## 정리

Oracle 실무에서 가장 중요한 역량은 단순히 SQL을 작성하는 것이 아니라 **실행 계획을 해석하고 병목을 파악하는 능력**이다. 핵심 정리는 다음과 같다.

- **PL/SQL 프로시저**: 커서, 패키지, 예외 처리를 구조적으로 작성
- **튜닝**: 실행 계획 → 인덱스 점검 → 힌트 → 통계 갱신 순으로 접근
- **서브쿼리**: 상관 서브쿼리는 JOIN으로, 반복 참조는 WITH 절로 최적화
- **대량 처리**: BULK COLLECT + FORALL로 컨텍스트 스위칭 최소화
- **EXISTS vs IN**: NULL 처리와 데이터 규모에 따라 선택
