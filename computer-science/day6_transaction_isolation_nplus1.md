# Day 6 — 데이터베이스: 트랜잭션 / 격리수준 / N+1 문제

## 1. 핵심 요약

- **트랜잭션 (Transaction)**
  - 데이터베이스의 논리적 작업 단위
  - **ACID 원칙**
    - **Atomicity (원자성)**: 모두 수행되거나 모두 취소
    - **Consistency (일관성)**: 무결성 유지
    - **Isolation (고립성)**: 동시에 실행되는 트랜잭션 간 간섭 방지
    - **Durability (지속성)**: 커밋된 데이터는 영구 반영

- **트랜잭션 격리 수준 (Isolation Level)**
  - **Read Uncommitted**
    - 커밋되지 않은 데이터 읽기 허용 → Dirty Read 발생 가능
  - **Read Committed**
    - 커밋된 데이터만 읽음 → Non-Repeatable Read 가능
  - **Repeatable Read**
    - 동일 쿼리 시 항상 같은 결과 반환 → Phantom Read 가능
  - **Serializable**
    - 가장 엄격 (직렬화 수준) → 동시성 낮음, 무결성 보장

- **격리 수준에 따른 문제**
  - Dirty Read: 커밋되지 않은 값 읽음
  - Non-Repeatable Read: 같은 쿼리 결과가 다름
  - Phantom Read: 범위 조회 시 새로운 행 등장

- **N+1 문제 (ORM, JPA)**
  - 하나의 조회 쿼리(N) 실행 후 각 결과마다 추가 쿼리(+1) 발생
  - 예: 회원 목록 조회 → 각 회원의 주문 목록 조회 시 매번 쿼리 발생
  - 원인: 지연 로딩(Lazy Loading)
  - 해결:
    - Fetch Join 사용 (`join fetch`)
    - `@EntityGraph` 활용
    - Batch size 설정

---

## 2. 면접형 Q&A 예시

**Q1. 트랜잭션의 ACID 원칙을 설명해주세요.**  
👉 원자성, 일관성, 고립성, 지속성으로 요약됩니다. 하나라도 깨지면 데이터 무결성이 보장되지 않습니다.

**Q2. Read Committed와 Repeatable Read의 차이는 무엇인가요?**  
👉 Read Committed는 커밋된 데이터만 보장하지만, 동일 쿼리 결과가 달라질 수 있습니다. Repeatable Read는 동일 쿼리 시 항상 같은 결과를 반환하지만 Phantom Read가 발생할 수 있습니다.

**Q3. Phantom Read는 무엇인가요?**  
👉 같은 조건으로 범위 조회를 했을 때, 다른 트랜잭션이 데이터를 추가하여 새로운 행이 조회되는 현상입니다.

**Q4. JPA에서 N+1 문제가 발생하는 이유와 해결 방법은 무엇인가요?**  
👉 Lazy Loading으로 인해 연관 엔티티를 하나씩 추가 조회하기 때문입니다. 해결 방법은 Fetch Join, EntityGraph, Batch size 등을 활용하는 것입니다.

---

## 3. 오늘의 미션
- [ ] 블로그에 `DB-02: 트랜잭션 / 격리수준 / N+1 문제` 포스트 작성
- [ ] GitHub `cs-notes/db/transaction-isolation-nplus1.md` 업로드
- [ ] 직접 답변: “Repeatable Read에서 발생하는 현상은 무엇이고 어떻게 해결할 수 있나요?”
