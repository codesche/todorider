
## 1. 트랜잭션(Transaction) 기본 개념

트랜잭션은 데이터베이스의 작업 단위(Unit of Work)를 의미한다. 즉, 여러
개의 쿼리를 하나의 묶음으로 처리하여 **모두 성공하거나(All or Nothing)
모두 실패**하도록 보장하는 매커니즘이다.

### 트랜잭션의 4가지 특성(ACID)

1.  **Atomicity (원자성)**
    1.  트랜잭션의 모든 작업이 하나의 단위로 처리됨. 일부만 성공하는
        일은 없음.
2.  **Consistency (일관성)**
    1.  트랜잭션 수행 전후의 데이터베이스는 항상 무결성을 유지해야 함.
3.  **Isolation (격리성)**
    1.  동시에 실행되는 트랜잭션이 서로 간섭하지 않아야 함.
4.  **Durability (지속성)**
    1.  트랜잭션이 성공적으로 끝나면 그 결과는 영구적으로 저장됨.

## 2. 격리 수준(Isolation Level)

동시에 여러 트랜잭션이 실행되면 **동시성 문제(Concurrency Issue)**가
발생할 수 있다. 이를 방지하기 위해 DBMS는 **격리 수준**을 제공한다.

### ⚠️ 대표적인 동시성 문제

-   **Dirty Read:** 커밋되지 않은 데이터를 읽어버림
-   **Non-repeatable Read:** 같은 데이터를 두 번 읽었는데 값이 달라짐
-   **Phantom Read:** 조건에 맞는 행을 두 번 읽었는데, 새 행이
    나타나거나 사라짐

### 📊 ANSI SQL 표준 격리 수준

| 격리 수준            | Dirty Read | Non-repeatable Read | Phantom Read | 특징                                       |
| -------------------- | ---------- | ------------------- | ------------ | ------------------------------------------ |
| **Read Uncommitted** | 발생       | 발생                | 발생         | 커밋 전 데이터도 읽을 수 있음              |
| **Read Committed**   | 방지       | 발생                | 발생         | 대부분의 DB 기본 수준 (Oracle, SQL Server) |
| **Repeatable Read**  | 방지       | 방지                | 발생 가능    | MySQL(InnoDB)    기본 수준                 |
| **Serializable**     | 방지       | 방지                | 방지         | 가장 안전하지만 성능 저하                  |

## 3. DBMS별 기본 격리 수준

-   **MySQL (InnoDB):** Repeatable Read
-   **PostgreSQL:** Read Committed
-   **Oracle:** Read Committed
-   **SQL Server:** Read Committed

👉 따라서 같은 쿼리를 실행해도 DBMS마다 동작이 달라질 수 있다.

## 4. 실무 적용 사례

### 💳 금융 서비스 예시

-   계좌 잔액 확인 및 출금 트랜잭션에서는 **Repeatable Read** 이상이
    필요하다.
    -   아니면 잔액을 확인한 뒤, 다른 트랜잭션에서 돈을 가져가버리는
        문제가 발생할 수 있음.

### 🛒 쇼핑몰 재고 관리

-   다수의 사용자가 동시에 상품을 구매할 경우,
    `SELECT 재고 FROM products WHERE id=?` 이후
    `UPDATE products SET 재고=재고-1` 로직에서 동시성 이슈 발생.
-   이 경우 **Pessimistic Lock(비관적 락)** 또는 **Optimistic
    Lock(낙관적 락)** 전략을 활용한다.

## 5. 면접 대비 포인트

-   트랜잭션의 ACID 특성을 설명할 수 있는가?
-   각 격리 수준에서 발생할 수 있는 문제를 말할 수 있는가?
-   MySQL과 PostgreSQL 기본 격리 수준이 다르다는 것을 아는가?
-   실무에서 락 전략(Optimistic vs Pessimistic)을 어떻게 선택할지 설명할
    수 있는가?

## 6. 정리

-   트랜잭션은 DB의 신뢰성을 보장하는 핵심 메커니즘.
-   격리 수준은 성능과 안정성 사이의 트레이드 오프.
-   실무에서는 DBMS 기본 설정을 이해하고, 서비스 특성에 맞춰 조정하는
    것이 중요하다.
