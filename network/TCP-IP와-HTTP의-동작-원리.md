# TCP/IP와 HTTP의 동작 원리

## 왜 이 글을 쓰는가

MSA 글을 쓰고, Reverse Proxy 글을 쓰고, Jenkins CI/CD를 정리했다.
그런데 막상 "TCP 연결이 어떻게 이루어지냐", "HTTP/1.1과 HTTP/2는 뭐가 다르냐"는 질문 앞에서 정확하게 말하기 어려웠다.

모든 서버 통신의 바닥에는 TCP/IP가 있고, 그 위에 HTTP가 있다.
이 계층을 제대로 이해하지 못하면 성능 문제가 생겼을 때, 네트워크 이슈가 생겼을 때 원인을 짚기 어렵다.

---

## 1. TCP/IP 모델이란

TCP/IP는 인터넷 통신의 표준 프로토콜 스택이다.
흔히 OSI 7계층과 함께 비교되지만, 실제 인터넷에서 쓰이는 건 TCP/IP 4계층 모델이다.

| 계층 | 이름 | 역할 | 대표 프로토콜 |
|---|---|---|---|
| 4 | 응용 계층 | 사용자와 직접 맞닿는 계층 | HTTP, HTTPS, DNS, FTP |
| 3 | 전송 계층 | 프로세스 간 통신, 신뢰성 보장 | TCP, UDP |
| 2 | 인터넷 계층 | 호스트 간 라우팅 | IP, ICMP |
| 1 | 네트워크 접근 계층 | 물리적 전송 | Ethernet, Wi-Fi |

패킷은 위 → 아래로 각 계층을 지나며 헤더가 붙고(캡슐화),
수신 측에서는 아래 → 위로 헤더를 벗기며(역캡슐화) 원래 데이터를 꺼낸다.

---

## 2. IP — 경로를 찾는 역할

IP(Internet Protocol)는 패킷을 목적지까지 전달하는 역할을 한다.
신뢰성은 보장하지 않는다. 패킷이 유실되거나 순서가 바뀌어도 IP는 모른다.

신뢰성은 상위 계층인 TCP가 담당한다.

---

## 3. TCP — 신뢰성 있는 연결

TCP(Transmission Control Protocol)는 데이터가 순서대로, 빠짐없이 도착한다는 것을 보장한다.

### 3-way Handshake — 연결 수립

```
클라이언트                    서버
    |                          |
    |------  SYN  ----------->|
    |<------ SYN + ACK  ------|
    |------  ACK  ----------->|
    |       (연결 수립됨)       |
```

- **SYN**: 클라이언트가 서버에 연결 요청
- **SYN + ACK**: 서버가 수락, 자신도 연결 요청
- **ACK**: 클라이언트가 확인

이 세 번의 교환으로 양방향 통신 채널이 열린다.

### 4-way Handshake — 연결 종료

```
클라이언트                    서버
    |                          |
    |------  FIN  ----------->|
    |<------  ACK  -----------|
    |<------  FIN  -----------|
    |------  ACK  ----------->|
    |       (연결 종료됨)       |
```

연결 종료는 4단계가 필요하다. 양쪽이 각자 독립적으로 연결을 끊기 때문이다.

### TCP vs UDP

| 특징 | TCP | UDP |
|---|---|---|
| 연결 방식 | 연결 지향 (3-way handshake) | 비연결 |
| 신뢰성 | 순서 보장, 재전송 | 없음 |
| 속도 | 상대적으로 느림 | 빠름 |
| 사용 예 | HTTP, HTTPS, FTP | DNS, 동영상 스트리밍, 게임 |

UDP는 느려도 안 되는 상황 — 실시간 영상, 게임, VoIP — 에 적합하다.
패킷 하나가 늦는 것보다 조금 끊기는 게 낫기 때문이다.

---

## 4. HTTP — 웹 통신의 언어

HTTP(HyperText Transfer Protocol)는 TCP 위에서 동작하는 응용 계층 프로토콜이다.
클라이언트가 요청(Request)을 보내고, 서버가 응답(Response)을 돌려주는 구조다.

### HTTP 요청 구조

```
GET /api/users HTTP/1.1
Host: example.com
Accept: application/json
Authorization: Bearer eyJhbGci...
```

- **시작줄**: 메서드 + 경로 + HTTP 버전
- **헤더**: 메타 정보 (Content-Type, Authorization 등)
- **바디**: POST/PUT일 때 실제 데이터

### HTTP 응답 구조

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 89

{"id": 1, "name": "codesche"}
```

### 주요 HTTP 메서드

| 메서드 | 의미 | 멱등성 |
|---|---|---|
| GET | 조회 | O |
| POST | 생성 | X |
| PUT | 전체 수정 | O |
| PATCH | 부분 수정 | X |
| DELETE | 삭제 | O |

멱등성(Idempotent): 같은 요청을 여러 번 해도 결과가 동일한 성질.
GET을 여러 번 해도 데이터는 안 바뀐다. POST는 요청할 때마다 새 리소스가 생긴다.

### HTTP 상태 코드

| 범위 | 의미 |
|---|---|
| 2xx | 성공 (200 OK, 201 Created) |
| 3xx | 리다이렉션 (301 Moved, 302 Found) |
| 4xx | 클라이언트 오류 (400 Bad Request, 401, 403, 404) |
| 5xx | 서버 오류 (500 Internal Server Error, 503) |

---

## 5. HTTP 버전별 진화

### HTTP/1.0 — 요청마다 새 연결

```
클라이언트 → [TCP 연결] → 요청 → 응답 → [TCP 종료]
클라이언트 → [TCP 연결] → 요청 → 응답 → [TCP 종료]  ← 매번 반복
```

요청 하나에 TCP 연결 하나. 3-way handshake 오버헤드가 매번 발생한다.

### HTTP/1.1 — Keep-Alive, Pipelining

```
클라이언트 → [TCP 연결]
             → 요청1 → 응답1
             → 요청2 → 응답2   ← 같은 연결 재사용
             → 요청3 → 응답3
             [TCP 종료]
```

**Keep-Alive**로 하나의 TCP 연결을 재사용한다.
**Pipelining**으로 응답을 기다리지 않고 여러 요청을 연속으로 보낼 수 있다.

단, HOL(Head-of-Line) Blocking 문제가 있다. 앞 요청이 지연되면 뒤 요청도 다 기다린다.

### HTTP/2 — Multiplexing

HTTP/2의 핵심은 **스트림(Stream)** 개념이다.

```
[하나의 TCP 연결]
  ├── 스트림 1: GET /index.html  →  응답1 ┐
  ├── 스트림 2: GET /style.css   →  응답2 ├─ 동시에 처리
  └── 스트림 3: GET /script.js  →  응답3 ┘
```

하나의 TCP 연결에서 여러 요청/응답이 **순서 상관없이** 동시에 처리된다.
HOL Blocking이 HTTP 레벨에서는 해결된다.

추가로:
- **헤더 압축(HPACK)**: 반복되는 헤더를 압축해서 전송
- **서버 푸시**: 클라이언트가 요청하기 전에 서버가 리소스를 미리 전송

### HTTP/3 — QUIC 기반

HTTP/3는 TCP 대신 **QUIC**(UDP 기반)을 전송 계층으로 사용한다.

HTTP/2의 Multiplexing은 TCP 레벨의 HOL Blocking은 해결 못 했다.
TCP 패킷 하나가 유실되면 전체 스트림이 대기한다.

HTTP/3는 UDP 위의 QUIC 프로토콜이 연결 수립과 재전송을 직접 처리해서
이 문제를 근본적으로 해결한다.

| 버전 | 전송 프로토콜 | 특징 |
|---|---|---|
| HTTP/1.0 | TCP | 요청마다 새 연결 |
| HTTP/1.1 | TCP | Keep-Alive, Pipelining, HOL 문제 |
| HTTP/2 | TCP | Multiplexing, 헤더 압축, 서버 푸시 |
| HTTP/3 | QUIC (UDP) | TCP HOL 제거, 빠른 연결 수립 |

---

## 6. HTTPS — TLS로 암호화된 HTTP

HTTPS = HTTP + TLS(Transport Layer Security)

TCP 연결 이후 TLS Handshake가 추가된다.

```
1. 클라이언트 → 서버: ClientHello (지원 암호화 방식 목록)
2. 서버 → 클라이언트: ServerHello + 인증서
3. 클라이언트: 인증서 검증 (CA 서명 확인)
4. 세션 키 교환 (비대칭 키로 암호화 → 이후 대칭 키 사용)
5. 이후 HTTP 통신은 모두 대칭 키로 암호화
```

**왜 HTTPS가 느리다고 하는가?**
TLS Handshake 과정이 추가되기 때문이다.
HTTP/2는 TLS를 사실상 강제하면서도 Multiplexing으로 오버헤드를 상쇄한다.

---

## 7. DNS — 도메인을 IP로 변환

브라우저에 `https://example.com`을 입력하면 가장 먼저 DNS 조회가 일어난다.

```
브라우저 → DNS 캐시 확인
         → 없으면 OS의 hosts 파일 확인
         → 없으면 로컬 DNS 리졸버에 질의
         → 루트 DNS → TLD DNS (.com) → 권한 DNS
         → IP 주소 반환
```

이 과정이 끝난 뒤에야 TCP 연결 → TLS Handshake → HTTP 요청이 시작된다.

---

## 8. 전체 흐름 — 브라우저에서 서버까지

`https://api.example.com/users` 요청 하나가 가는 과정을 전부 이어보면:

```
1. DNS 조회 → IP 주소 획득
2. TCP 3-way Handshake → 연결 수립
3. TLS Handshake → 암호화 채널 수립
4. HTTP 요청 전송 (GET /users HTTP/2)
5. 서버 처리 → HTTP 응답 반환
6. (연결 재사용 or TCP 4-way Handshake로 종료)
```

---

## 정리

| 개념 | 핵심 |
|---|---|
| TCP | 신뢰성 있는 전송. 3-way로 연결, 4-way로 종료 |
| UDP | 빠르지만 비신뢰성. 실시간 통신에 적합 |
| HTTP/1.1 | Keep-Alive. HOL Blocking 존재 |
| HTTP/2 | Multiplexing. HTTP 레벨 HOL 해결 |
| HTTP/3 | QUIC 기반. TCP 레벨 HOL까지 해결 |
| HTTPS | TLS Handshake로 암호화 |
| DNS | 도메인 → IP 변환, 통신 전 가장 먼저 발생 |

MSA에서 서비스 간 통신이 왜 느린지, Reverse Proxy가 어떻게 TLS를 대신 처리하는지,
HTTP/2가 왜 성능상 유리한지 — 이 글의 내용이 그 모든 것의 기반이다.
