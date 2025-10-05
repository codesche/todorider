# Day 7 — Spring Boot 실무화

## 1. 핵심 요약

- **프로젝트 구조 재정돈**
  - `domain`: dto → entity → repository → service → controller
  - `global`: config, exception, util, security 등 공통 모듈
  - Controller는 DTO만 반환, 비즈니스 로직은 Service 계층에 집중

- **전역 예외 처리**
  - `@RestControllerAdvice` + `@ExceptionHandler` 활용
  - 일관된 응답 객체(RsData) 반환 → 클라이언트/테스트 코드 일관성 확보

- **표준 응답 구조 (예시)**
  ```json
  {
    "success": true,
    "message": "요청 성공",
    "data": { ... }
  }
  ```

- **테스트 환경 구성**
  - `application-test.yml` 분리
  - DB: H2 인메모리 or Testcontainers 활용
  - JUnit5 기반 단위/통합 테스트 준비

- **품질 가드**
  - 로깅 표준화 (Slf4j + MDC)
  - DTO 변환 메서드는 DTO 내부 정적 메서드로 이동
  - 공통 유틸: 비밀번호 인코더, 날짜 포맷 등 모듈화

---

## 2. 면접형 Q&A 예시

**Q1. Controller에 비즈니스 로직을 두면 어떤 문제가 생기나요?**  
👉 테스트가 어려워지고 관심사가 분리되지 않아 유지보수가 힘들어집니다. 따라서 Controller는 요청/응답 처리에 집중하고, Service가 핵심 비즈니스 로직을 담당해야 합니다.

**Q2. Spring Boot에서 예외 처리를 일관되게 적용하는 방법은 무엇인가요?**  
👉 `@RestControllerAdvice`와 `@ExceptionHandler`를 활용하여 글로벌하게 예외를 처리하고, 표준 응답 객체를 통해 클라이언트에 전달합니다.

**Q3. 테스트 환경에서 실제 DB 대신 H2를 사용하는 이유는 무엇인가요?**  
👉 테스트 속도를 높이고 독립적인 환경에서 테스트 가능하게 하며, 로컬/CI 환경에서 재현성을 높일 수 있습니다.

---

## 3. 오늘의 미션
- [ ] 기존 프로젝트 구조 정리 (`domain`, `global` 패키지 분리)
- [ ] 글로벌 예외 핸들러 작성 (`@RestControllerAdvice`)
- [ ] 표준 응답 객체(RsData) 도입 및 적용
- [ ] 테스트 환경 구성 (`application-test.yml`, H2 세팅)
- [ ] GitHub에 PR 생성: `chore: project scaffold`

---

## 4. 셀프 체크
- Controller에 비즈니스 로직이 0%인가?
- 모든 API 응답이 RsData 표준 응답을 따르는가?
- 테스트 환경이 실제 DB와 분리되어 독립적으로 동작하는가?
