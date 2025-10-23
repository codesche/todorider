
# MySQL Lock과 DeadLock, 그리고 낙관적 락 그리고 JPA

DB에서 Lock은 “데이터를 동시에 건드리지 못하게 하는 안전장치”다.
문제는, 이 안전장치가 너무 많이 걸리면 **교통정체**처럼 시스템이 멈춰버린다는 점이다.
MySQL의 락 메커니즘 그리고 JPA/Hibernate가 이를 어떻게 다루는지 
또 낙관적 락이 어떤 역할을 하는지에 대해 정리해봤다.

## 1. Lock이란? - “식당 자리 맡기”

DB 락을 비유하자면, **식당 자리 맡기**와 같다.

- **Exclusive Lock(배타 락, 쓰기 락)**:
    - **비유: “이 자리 내거야. 아무도 못 앉아!”**
    → 누군가 UPDATE, DELETE 같은 수정 쿼리를 실행하면, 해당 행(row)은 쓰기 락이 걸린다.
    - 다른 트랜잭션은 그 행을 **읽을 수도, 쓸 수도 없다.**
- **Shared Lock(공유 락, 읽기 락):**
    - **비유:** **“같이 메뉴만 보자. 근데 자리 바꾸면 안 돼.”
    →** SELECT … FOR SHARE 같은 쿼리를 사용할 때 걸린다.
    여러 트랜잭션이 동시에 읽을 수 있지만, 수정은 불가능하다.

즉, **락은 동시에 같은 데이터를 조작하지 못하게 하는 일종의 약속**이다.

## 2. DeadLock(교착 상태) - “서로 의자를 잡고 못 놔주는 상황”

DeadLock은 두 트랜잭션이 **서로가 가진 락을 기다리는 무한 대기 상태**를 말한다.

> **비유:**
A가 식사 자리1을 맡고 식사 자리2를 기다림\
B가 식사 자리2를 맡고 식사 자리1을 기다림
→ 이렇게 되면 A와 B 둘 다 평생 밥을 먹을 수 없다!
> 

### 예시 (MySQL)

```sql
-- 트랜잭션 A
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 1;

-- 트랜잭션 B
BEGIN;
UPDATE account SET balance = balance + 100 WHERE id = 2;

-- A가 B의 데이터를 수정하려고 시도
UPDATE account SET balance = balance + 100 WHERE id = 2;

-- B가 A의 데이터를 수정하려고 시도
UPDATE account SET balance = balance - 100 WHERE id = 1;
```

이제 A와 B는 서로 상대의 락을 기다리며 **DeadLock** 상태에 빠진다.\
MySQL은 이때 **한쪽 트랜잭션을 강제 종료(Rollback)** 시켜버린다.

> **해결책:** 항상 **락 순서를 일관되게 유지**해야 한다.
(예: `id` 순으로 UPDATE 한다)
> 

## 3. JPA와 락 - “Hibernate는 DB 락을 어떻게 다루는가”

JPA에서 락은 크게 두 가지 방식으로 구분된다.

| 락 종류 | 설명 | MySQL 대응 |
| --- | --- | --- |
| 비관적 락 (Pessimistic Lock) | 충돌이 날 것을 **미리 걱정**하고 락을 걸어둔다. | SELECT FOR UPDATE |
| 낙관적 락 (Optimistic Lock) | 충돌이 **가끔 일어날 거라 가정**하고, 나중에 감지한다. | 버전(version) 컬럼 비교 |

## 4. 비관적 락 (Pessimistic Lock) - “내가 수정할 테니까 잠깐 건들지 마!”

Hibernate에서는 다음과 같이 사용한다.

```java
@Entity
public class Account {
    @Id
    private Long id;
    private int balance;
}
```

```java
// Service 내부
Account account = entityManager
    .find(Account.class, 1L, LockModeType.PESSIMISTIC_WRITE);

account.setBalance(account.getBalance() - 100);
```

이때 Hibernate는 다음 SQL을 날린다.

```sql
SELECT * FROM account WHERE id = 1 FOR UPDATE;
```

→ 이 쿼리는 **트랜잭션이 끝날 때까지 해당 행을 잠금 상태로 유지**한다.

다른 트랜잭션이 이 행을 건드리면 **대기 또는 DeadLock**이 발생할 수 있다.

> **!주의**: 비관적 락은 “**DB 교통 체증**”을 유발한다. 너무 많이 걸면 시스템이 느려질 수 있다.
> 

## 5. 낙관적 락 (Optimistic Lock) - “서로 예의 지키면서 수정하자!”

비관적 락은 즉시 자원을 점유하지만, **낙관적 락은 수정 시점에만 충돌을 감지**한다.

### 예시

```java
@Entity
public class Account {
    @Id
    private Long id;

    @Version
    private Long version;

    private int balance;
}
```

Hibernate는 이 엔티티를 수정할 때 다음과 같은 SQL을 생성한다.

```sql
UPDATE account
SET balance = ?, version = version + 1
WHERE id = ? AND version = ?
```

→ 여기서 **version 컬럼이 조건으로 포함된다.**

정리하자면,

- A가 version=1인 데이터를 읽고 수정을 시도
- B도 같은 데이터를 읽고 version=1에서 수정 시도
    - B가 먼저 커밋하면 version=2로 바뀌고
    - A가 나중에 커밋할 때 version mismatch로 **OptimisticLockException** 발생!

> 낙관적 락은 DB 락을 거는 게 아니라,
**데이터 버전으로 충돌을 감지하는 논리적 락**이다.
> 

## 6. JPA 낙관적 락의 실제 흐름

1. `@Version` 필드가 있는 엔티티를 조회
2. 수정 후 트랜잭션 커밋 시 Hibernate가 version 비교
3. version이 같으면 UPDATE + version++
4. 다를 경우 `OptimisticLockException` 발생
5. 애플리케이션 레벨에서 재시도 로직으로 해결 가능

```java
@Transactional
public void transfer(Long fromId, Long toId, int amount) {
    Account from = accountRepository.findById(fromId).orElseThrow();
    Account to = accountRepository.findById(toId).orElseThrow();

    from.withdraw(amount);
    to.deposit(amount);

    // version 충돌 발생 시 OptimisticLockException 발생
}
```

> 낙관적 락은 “**락**”이라기보단 **충돌 감지 장치**다.
따라서 성능 부담이 적거나 대규모 트래픽 환경에서 더 적합하다.
> 

## 7. 비유를 통한 정리

| 상황 | 비유 | 실제 동작 |
| --- | --- | --- |
| 비관적 락 | “회의실 먼저 예약해놓고 나중에 회의” | SELECT FOR UPDATE |
| 낙관적 락 | “회의 끝나면 기록 남기고, 중복 예약이면 취소” | @Version 필드 비교 |
| DeadLock | “두 팀이 서로 회의실 열쇠를 들고 기다림” | 락 순서 꼬임으로 인한 대기 |

## 8. 요약

- MySQL 락은 DB 단에서의 “**좌석 점유**”다.
- DeadLock은 “**서로 자리를 놓지 못해 밥을 못 먹는 상황**”이다.
- JPA에서 비관적 락은 `SELECT FOR UPDATE`,
    
    낙관적 락은 `@Version`을 활용한 버전 비교다.
    
- 비관적 락은 즉시 잠그고, 낙관적 락은 사후 검증한다.

**즉, 비관적 락은 “신중한 사람의 선택”, 낙관적 락은 “효율적인 사람의 선택”이라고 생각하면 된다.**