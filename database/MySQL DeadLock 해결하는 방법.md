# MySQL DeadLock 해결하는 방법

생성 일시: 2025년 10월 24일 오전 1:26
태그: MySQL
게시일: 2025년 10월 24일
최종 편집 일시: 2025년 10월 24일 오전 1:58

```yaml
floatFirstTOC: right
```

# 교통정체(DeadLock)은 미리 예방하자

DeadLock(교착상태)은 트랜잭션이 서로의 자원을 기다리며 무한 대기에 빠진 상태를 말한다.

MySQL에서는 InnoDB 스토리지 엔진이 **DeadLock을 자동 감지**하지만, 문제는

그 순간 한쪽 트랜잭션이 강제 롤백된다는 점이다. 즉, **DB가 대신 희생양을 선택하는 셈**이다.  ****

## 1. DeadLock의 원인 요약

DeadLock은 **락의 순서가 꼬이거나, 락 범위가 예상보다 넓을 때** 주로 발생한다.

대표적인 예시는 다음과 같다.

1. **락 순서 불일치**
    1. 트랜잭션 A: row 1 → row 2 순으로 UPDATE
    2. 트랜잭션 B: row 2 → row 1 순으로 UPDATE
    → 두 트랜잭션이 서로 반대 순서로 락을 걸면 DeadLock 발생 가능성이 높다.
2. **인덱스 미비**
    1. WHERE 절이 인덱스를 타지 않으면, MySQL은 **의도치 않게 여러 행을 스캔하면서 락을 건다.**
    2. 예를 들어, `UPDATE member SET status='Y' WHERE email='abc@xyz.com’` 에
    email 인덱스가 없다면, MySQL은 전체 테이블을 락하려고 시도한다.
3. 트랜잭션의 시간이 지나치게 긴 경우
    1. 긴 트랜잭션은 오랫동안 락을 점유한다.
    2. 그동안 다른 트랜잭션이 같은 자원을 접근하면 대기 → 교착 가능성 증가.

## 2. DeadLock의 예방 방법

### 1. 락 순서를 일관되게 유지한다

비유하자면, “모든 손님이 줄을 같은 방향으로 서게 하는 것”이다.

```sql
-- 잘못된 예시
-- 트랜잭션 A: id 1 → 2 순서로 수정
-- 트랜잭션 B: id 2 → 1 순서로 수정
```

이를 아래처럼 수정하면 DeadLock 가능성이 크게 줄어든다.

```sql
-- 모든 트랜잭션이 id 오름차순으로 UPDATE
UPDATE account SET balance = balance - 100 WHERE id = 1;
UPDATE account SET balance = balance + 100 WHERE id = 2;
```

> 트랜잭션에서 여러 자원을 수정할 때는 **항상 동일한 순서로 접근**하는 것이 좋다.
> 

### 2. 인덱스를 걸어 락 범위를 줄인다

MySQL은 인덱스가 없으면 “테이블 전체에 락을 건다.”

즉, 하나의 행만 바꾸려다가 테이블 전체가 잠길 수 있다.

```sql
-- 비효율적 (Full Table Scan)
UPDATE member SET status='Y' WHERE email='abc@xyz.com';

-- 효율적 (인덱스 타게끔)
ALTER TABLE member ADD INDEX idx_email (email);
```

> 인덱스는 단순히 검색 속도뿐 아니라 **락 범위를 줄이는 데도 중요하다.**
> 

### 3. 트랜잭션의 범위를 최소화한다

트랜잭션은 짧을수록 좋다. 가능하다면 다음 규칙을 지켜야 한다.

- SELECT 쿼리는 트랜잭션 밖에서 미리 수행한다.
- 필요 이상으로 많은 UPDATE/DELETE를 한 트랜잭션에서 처리하지 않는다.
- 비즈니스 로직 계산은 DB 밖 (서비스 계층)에서 처리한다.

```java
@Transactional
public void updateBalance(Long id, int diff) {
    Account account = accountRepository.findById(id).orElseThrow();
    account.setBalance(account.getBalance() + diff);
    // 여기서 외부 API 호출이나 sleep() 넣는 건 절대 금지
}
```

> 트랜잭션 내부에서는 “DB I/O에만 집중”하는 것이 DeadLock 방지의 핵심이다.
> 

### 4. 락 모드를 명확히 지정한다 (JPA/Hibernate)

Hibernate에서 비관적 락을 쓸 때, 명시적으로 `LockModeType.PESSIMISTIC_WRITE` 

또는 `READ` 를 지정하면 의도치 않은 락 경쟁을 줄일 수 있다.

```java
Account account = entityManager.find(
    Account.class,
    1L,
    LockModeType.PESSIMISTIC_WRITE
);
```

> Hibernate는 내부적으로 `SELECT … FOR UPDATE` 를 실행한다.
이때 명시적으로 락 모드를 주면, 락 충돌 범위를 제어할 수 있다.
> 

### 5. 낙관적 락(Optimistic Lock)으로 구조적인 해결

DeadLock을 피하는 가장 강력한 방법은 **락 자체를 걸지 않는 것**이다.

즉, “논리적으로 충돌을 감지하는 방식”으로 바꾸는 것이다.

```java
@Entity
public class Product {
    @Id
    private Long id;

    @Version
    private Long version;

    private int stock;
}
```

- 동시에 두 사용자가 같은 상품의 재고를 변경하더라도,
- 나중에 커밋한 쪽은 `OptimisticLockException` 으로 롤백된다.
- DeadLock 대신 **낙관적 충돌 감지**가 발생한다.

> 결론: DeadLock이 자주 나는 구간은 낙관적 락으로 구조를 바꾸는 게 더 낫다.
> 

## 3. DeadLock이 발생했을 때의 대처법

### 1. 에러 감지 및 재시도 로직

MySQL은 DeadLock 발생 시 다음 메시지를 반환한다.

```sql
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

Hibernate는 이를 j`avax.persistence.PersistenceException` 또는

`DeadlockLoserDataAccessException` 으로 감싼다.

따라서 다음과 같이 재시도 로직을 적용할 수 있다.

```java
@Transactional
public void safeTransfer(Long fromId, Long toId, int amount) {
    for (int i = 0; i < 3; i++) {
        try {
            transfer(fromId, toId, amount);
            return;
        } catch (DeadlockLoserDataAccessException e) {
            log.warn("DeadLock 발생, {}번째 재시도 중...", i + 1);
        }
    }
    throw new RuntimeException("DeadLock 3회 발생, 처리 실패");
}
```

> 핵심: DeadLock은 완전히 제거할 수는 없으므로, **“감지 후 재시도”** **전략을 반드시 포함해야 한다.**
> 

### 2. DeadLock 로그 분석

MySQL에서 DeadLock 발생 시 `SHOW ENGINE INNODB STATUS;` 명령으로

최근 DeadLock 정보를 볼 수 있다.

```sql
SHOW ENGINE INNODB STATUS;
```

출력 예시 중 `“LATEST DETECTED DEADLOCK”` 

섹션을 보면 어떤 트랜잭션이 어떤 테이블, 어떤 인덱스에서 충돌했는지 확인 가능하다.

> 이 정보를 기반으로 쿼리 순서, 인덱스, 락 범위를 조정하면 된다.
> 

## 정리 — DeadLock은 완전히 막을 수 없지만, 줄일 수는 있다

| 전략 | 효과 |
| --- | --- |
| 락 순서 일관성 유지 | DeadLock 확률 대폭 감소 |
| 인덱스 최적화 | 락 범위 최소화 |
| 짧은 트랜잭션 | 락 점유 시간 단축 |
| 명시적 락 모드 지정 | Hibernate 예측 가능성 향상 |
| 낙관적 락 전환 | DeadLock 구조 자체 제거 |
| 재시도 로직 추가 | 시스템 안정성 확보 |