# Day 4 — 네트워크: HTTP/HTTPS/TLS

## 1. 핵심 요약

- **HTTP (HyperText Transfer Protocol)**
  - 클라이언트-서버 통신 규약 (비연결성, 무상태성)
  - 주요 메서드: GET, POST, PUT, DELETE, PATCH
  - 상태 코드: 2xx(성공), 3xx(리다이렉션), 4xx(클라이언트 에러), 5xx(서버 에러)
  - 캐시 활용: Cache-Control, ETag, Last-Modified

- **HTTPS (HTTP Secure)**
  - HTTP + SSL/TLS
  - 암호화, 무결성, 인증 보장
  - 기본 포트: 443

- **TLS (Transport Layer Security)**
  - **핸드셰이크 과정**
    1. 클라이언트 → 서버: ClientHello (지원 암호화 방식, 랜덤 값)
    2. 서버 → 클라이언트: ServerHello (선택된 암호화 방식, 인증서)
    3. 클라이언트: 인증서 검증 → Pre-Master Secret 생성 후 서버 공개키로 암호화 전송
    4. 서버: 개인키로 복호화 → 세션 키 생성
    5. 양측: 대칭키 기반 데이터 암호화 통신 시작

- **HTTP/2 (H2)**
  - 이진 프레이밍, 헤더 압축(HPACK)
  - 멀티플렉싱 (동일 연결에서 여러 요청 동시 처리)
  - HOL(Head-of-line) 블로킹 해결

- **REST vs gRPC**
  - REST: 텍스트 기반(JSON), 범용성 높음
  - gRPC: 바이너리 기반(Protocol Buffers), 성능 우수, 양방향 스트리밍 가능

---

## 2. 면접형 Q&A 예시

**Q1. HTTP와 HTTPS의 차이는 무엇인가요?**  
👉 HTTPS는 HTTP 위에 SSL/TLS를 추가하여 암호화, 무결성, 인증을 제공합니다. HTTP는 평문 전송이라 보안에 취약합니다.

**Q2. TLS 핸드셰이크 과정을 설명해주세요.**  
👉 클라이언트와 서버가 서로 암호화 방식을 협상하고, 인증서를 통해 서버 신뢰성을 검증한 뒤, 세션 키를 교환하여 대칭키 암호화로 통신을 시작합니다.

**Q3. HTTP/2가 HTTP/1.1보다 빠른 이유는 무엇인가요?**  
👉 멀티플렉싱을 지원하여 하나의 TCP 연결에서 여러 요청을 병렬 처리할 수 있고, 헤더 압축으로 네트워크 전송량을 줄이기 때문입니다.

**Q4. REST와 gRPC의 장단점은 무엇인가요?**  
👉 REST는 단순하고 범용성이 높아 웹에서 많이 쓰이고, gRPC는 속도가 빠르고 양방향 스트리밍을 지원하지만 진입장벽이 높습니다.

---

## 3. 오늘의 미션
- [ ] 블로그에 `NW-02: HTTP/HTTPS/TLS` 포스트 작성
- [ ] GitHub `cs-notes/nw/http-https-tls.md` 업로드
- [ ] 직접 답변: “HTTP/2의 성능 개선 포인트는?” (→ 멀티플렉싱, 헤더 압축)
