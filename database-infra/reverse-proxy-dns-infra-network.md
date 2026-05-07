# Reverse Proxy, DNS, 인프라 & 네트워크 기초 완전 정리

## 개요

백엔드 개발자라면 코드만 잘 짜는 것으로 부족하다. 실무에서는 내가 만든 서비스가 어떤 네트워크 경로를 타고 사용자에게 전달되는지, 요청이 어디서 어떻게 라우팅되는지를 파악하는 능력이 반드시 필요하다. 이 글에서는 Reverse Proxy, DNS, 그리고 인프라·네트워크의 핵심 개념을 실무 관점에서 정리한다.

---

## 1. 네트워크 기초

### 1-1. OSI 7계층 vs TCP/IP 4계층

실무에서 문제를 추적할 때 어느 계층에서 문제가 발생했는지 파악하는 것이 트러블슈팅의 시작이다.

| OSI 7계층 | TCP/IP 4계층 | 주요 프로토콜 | 역할 |
|-----------|-------------|--------------|------|
| 7. 응용 (Application) | 응용 (Application) | HTTP, HTTPS, FTP, DNS, SMTP | 사용자 인터페이스 |
| 6. 표현 (Presentation) | ^ | TLS/SSL | 인코딩, 암호화 |
| 5. 세션 (Session) | ^ | — | 세션 관리 |
| 4. 전송 (Transport) | 전송 (Transport) | TCP, UDP | 포트 기반 통신, 신뢰성 |
| 3. 네트워크 (Network) | 인터넷 (Internet) | IP, ICMP, ARP | 논리 주소(IP), 라우팅 |
| 2. 데이터링크 (Data Link) | 네트워크 액세스 | Ethernet, Wi-Fi | MAC 주소, 프레임 |
| 1. 물리 (Physical) | ^ | — | 전기 신호, 케이블 |

> 트러블슈팅 순서: **물리(ping) → IP(traceroute) → TCP(nc/telnet) → 응용(curl)** 순으로 좁혀 나간다.

### 1-2. IP 주소 체계

```
# IPv4 클래스 구분
A 클래스: 1.0.0.0   ~ 126.255.255.255  (대규모 네트워크)
B 클래스: 128.0.0.0 ~ 191.255.255.255  (중규모 네트워크)
C 클래스: 192.0.0.0 ~ 223.255.255.255  (소규모 네트워크)

# 사설 IP 대역 (Private IP)
10.0.0.0/8       → 10.0.0.0   ~ 10.255.255.255
172.16.0.0/12    → 172.16.0.0 ~ 172.31.255.255
192.168.0.0/16   → 192.168.0.0 ~ 192.168.255.255

# CIDR 표기법
192.168.1.0/24   → 호스트 254개 사용 가능 (/24 = 서브넷 마스크 255.255.255.0)
10.0.0.0/16      → 호스트 65,534개 사용 가능
```

### 1-3. TCP vs UDP

| 항목 | TCP | UDP |
|------|-----|-----|
| 연결 방식 | 연결 지향 (3-way handshake) | 비연결 (connectionless) |
| 신뢰성 | 보장 (재전송, 순서 보장) | 미보장 |
| 속도 | 느림 (오버헤드 존재) | 빠름 |
| 흐름 제어 | 있음 | 없음 |
| 주요 용도 | HTTP, HTTPS, FTP, SSH | DNS, 스트리밍, 게임, VoIP |

```
# TCP 3-way Handshake
Client → Server : SYN
Client ← Server : SYN-ACK
Client → Server : ACK
(이후 데이터 전송)

# TCP 4-way Handshake (연결 종료)
Client → Server : FIN
Client ← Server : ACK
Client ← Server : FIN
Client → Server : ACK
```

### 1-4. 포트(Port)와 소켓(Socket)

```bash
# 잘 알려진 포트 (Well-Known Ports: 0~1023)
22   → SSH
53   → DNS
80   → HTTP
443  → HTTPS
3306 → MySQL
5432 → PostgreSQL
6379 → Redis
8080 → HTTP 대체 (개발/프록시)

# 현재 열려있는 포트 확인
sudo ss -tlnp
sudo netstat -tlnp

# 특정 포트 연결 테스트
telnet 192.168.1.10 3306
nc -zv 192.168.1.10 3306

# 소켓 = IP 주소 + 포트 번호
# 예: 192.168.1.10:8080
```

---

## 2. DNS (Domain Name System)

### 2-1. DNS란

DNS는 사람이 읽을 수 있는 도메인 이름(`example.com`)을 IP 주소(`93.184.216.34`)로 변환하는 분산 계층 데이터베이스 시스템이다.

```
[브라우저에서 example.com 입력 시 DNS 조회 흐름]

1. 브라우저 캐시 확인
2. OS 캐시 확인 (/etc/hosts 포함)
3. 로컬 DNS 리졸버 (ISP 또는 8.8.8.8 등)
4. 루트 네임서버 (.) 질의
5. TLD 네임서버 (.com) 질의
6. 권한 네임서버 (example.com) 질의
7. A 레코드 반환 → IP 주소 획득
```

### 2-2. DNS 레코드 종류

| 레코드 | 설명 | 예시 |
|--------|------|------|
| A | 도메인 → IPv4 주소 | example.com → 93.184.216.34 |
| AAAA | 도메인 → IPv6 주소 | example.com → 2001:db8::1 |
| CNAME | 도메인 → 다른 도메인 (별칭) | www → example.com |
| MX | 메일 서버 지정 | example.com → mail.example.com |
| TXT | 텍스트 정보 (SPF, DKIM 등) | v=spf1 include:... |
| NS | 권한 네임서버 지정 | example.com → ns1.provider.com |
| PTR | IP → 도메인 (역방향 DNS) | 34.216.184.93.in-addr.arpa |
| SOA | DNS 존 권한 정보 | 최초 레코드, TTL 정책 |

```bash
# DNS 조회 명령어
nslookup example.com
dig example.com
dig example.com A        # A 레코드만 조회
dig example.com MX       # MX 레코드 조회
dig +trace example.com   # 전체 조회 경로 추적

# 역방향 DNS 조회
dig -x 93.184.216.34
nslookup 93.184.216.34

# /etc/hosts 파일 (로컬 DNS 우선 적용)
127.0.0.1   localhost
192.168.1.10 myapp.local
```

### 2-3. TTL (Time To Live)

```
TTL = DNS 응답이 캐시에 유지되는 시간(초)

- TTL 300   → 5분마다 DNS 재조회
- TTL 3600  → 1시간 캐시 유지
- TTL 86400 → 24시간 캐시 유지

실무 팁:
- 서버 이전 예정 전 미리 TTL을 300~600으로 낮춰둔다
- 이전 완료 후 다시 TTL 높임
- TTL이 높으면 변경 전파가 느림 (전파 지연 최대 TTL 시간 소요)
```

### 2-4. DNS와 로드밸런서 연동

```
# DNS Round Robin: 동일 도메인에 여러 IP 등록
example.com A 1.2.3.4
example.com A 1.2.3.5
example.com A 1.2.3.6
→ 클라이언트마다 다른 IP 반환 (단순 부하분산, 헬스체크 없음)

# 더 고도화된 방법: AWS Route 53, Cloudflare DNS
- 지역 기반 라우팅 (Geo Routing)
- 헬스체크 기반 페일오버
- 가중치 기반 라우팅 (Weighted Routing)
```

---

## 3. Reverse Proxy

### 3-1. Proxy vs Reverse Proxy

```
[Forward Proxy]
클라이언트 → [Proxy 서버] → 인터넷 → 목적지 서버
- 클라이언트를 대신해서 요청
- 클라이언트 IP 숨김
- 기업 내부망에서 인터넷 접근 제어
- 캐싱으로 트래픽 절감

[Reverse Proxy]
클라이언트 → 인터넷 → [Reverse Proxy] → 백엔드 서버
- 서버를 대신해서 요청 수신
- 백엔드 서버 IP/구조 숨김
- 로드밸런싱, SSL 종료, 캐싱
- 보안 게이트웨이 역할
```

### 3-2. Reverse Proxy의 핵심 기능

**① 로드밸런싱 (Load Balancing)**

```nginx
upstream backend_servers {
    # Round Robin (기본값)
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;

    # 가중치 기반
    # server 10.0.0.1:8080 weight=3;
    # server 10.0.0.2:8080 weight=1;

    # IP Hash (세션 고정)
    # ip_hash;

    # Least Connections
    # least_conn;

    # 헬스체크
    # server 10.0.0.3:8080 backup;   # 장애 시 백업
    # server 10.0.0.4:8080 down;     # 사용 중지
}

server {
    listen 80;
    location / {
        proxy_pass http://backend_servers;
    }
}
```

**② SSL/TLS 종료 (SSL Termination)**

```nginx
# HTTPS → HTTP 변환 (내부망은 평문 통신)
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/ssl/certs/example.com.crt;
    ssl_certificate_key /etc/ssl/private/example.com.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://backend_servers;  # 내부는 HTTP
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# HTTP → HTTPS 리다이렉트
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

**③ 정적 파일 캐싱**

```nginx
server {
    listen 80;

    # 정적 파일 직접 서빙 (백엔드 부하 감소)
    location /static/ {
        root /var/www;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # API 요청만 백엔드로 전달
    location /api/ {
        proxy_pass http://backend_servers;
        proxy_cache my_cache;
        proxy_cache_valid 200 10m;
        proxy_cache_valid 404 1m;
    }
}
```

**④ 요청 필터링 / 보안**

```nginx
server {
    # 요청 크기 제한
    client_max_body_size 10M;

    # 특정 IP 차단
    deny 192.168.1.100;
    allow all;

    # 헤더 보안 추가
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header Strict-Transport-Security "max-age=31536000";

    # 민감한 경로 차단
    location ~* /(admin|config|backup) {
        deny all;
    }
}
```

### 3-3. Nginx vs HAProxy vs Traefik

| 항목 | Nginx | HAProxy | Traefik |
|------|-------|---------|--------|
| 주 용도 | 웹서버 + 리버스 프록시 | L4/L7 로드밸런서 | 컨테이너 환경 프록시 |
| 설정 방식 | 정적 설정 파일 | 정적 설정 파일 | 동적 자동 설정 |
| 컨테이너 연동 | 수동 설정 필요 | 수동 설정 필요 | Docker/K8s 레이블 자동 감지 |
| 성능 | 매우 높음 | 매우 높음 | 높음 |
| SSL 관리 | 수동 or Certbot | 수동 | Let's Encrypt 자동 갱신 |
| 적합 환경 | 일반 웹 서비스 | 고성능 금융/엔터프라이즈 | 마이크로서비스, K8s |

### 3-4. 실무 Nginx 전체 설정 예시

```nginx
# /etc/nginx/nginx.conf
events {
    worker_connections 1024;
}

http {
    # 업스트림 정의
    upstream app_cluster {
        least_conn;
        server 10.0.1.10:8080 weight=3;
        server 10.0.1.11:8080 weight=2;
        server 10.0.1.12:8080 backup;
        keepalive 32;
    }

    # 캐시 영역 정의
    proxy_cache_path /var/cache/nginx levels=1:2
        keys_zone=my_cache:10m max_size=1g
        inactive=60m use_temp_path=off;

    # HTTPS 서버
    server {
        listen 443 ssl http2;
        server_name api.example.com;

        ssl_certificate     /etc/ssl/api.example.com.crt;
        ssl_certificate_key /etc/ssl/api.example.com.key;
        ssl_session_cache   shared:SSL:10m;
        ssl_session_timeout 10m;

        # 압축
        gzip on;
        gzip_types application/json text/plain;

        # 타임아웃
        proxy_connect_timeout 5s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;

        location / {
            proxy_pass         http://app_cluster;
            proxy_http_version 1.1;
            proxy_set_header   Upgrade $http_upgrade;
            proxy_set_header   Connection 'upgrade';
            proxy_set_header   Host $host;
            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Proto https;
            proxy_cache_bypass $http_upgrade;
        }

        location /health {
            access_log off;
            return 200 'OK';
            add_header Content-Type text/plain;
        }
    }

    # HTTP → HTTPS 리다이렉트
    server {
        listen 80;
        server_name api.example.com;
        return 301 https://$server_name$request_uri;
    }
}
```

---

## 4. 인프라 핵심 개념

### 4-1. 서버 아키텍처 패턴

```
[단일 서버 구조]
클라이언트 → 서버 (웹 + WAS + DB 동일 서버)
- 장점: 구성 단순
- 단점: SPOF(단일 장애점), 확장 불가

[N-Tier 구조]
클라이언트
    ↓
[Nginx: Reverse Proxy + Static]
    ↓
[WAS 클러스터: App Server x N]
    ↓
[DB 클러스터: Primary + Replica]
- 각 계층 독립 확장 가능
- 장애 격리 가능

[마이크로서비스 구조]
클라이언트
    ↓
[API Gateway]
    ↓
[Service A] [Service B] [Service C]
    ↓           ↓           ↓
[DB-A]      [DB-B]      [DB-C]
- 독립 배포, 독립 확장
- 복잡성 증가
```

### 4-2. 로드밸런서 (Load Balancer)

```
[L4 로드밸런서 - 전송 계층]
- TCP/UDP 수준에서 트래픽 분산
- IP + 포트 기반 라우팅
- 빠름, 단순
- 예: AWS NLB, HAProxy L4 모드

[L7 로드밸런서 - 응용 계층]
- HTTP/HTTPS 수준에서 트래픽 분산
- URL 경로, 헤더, 쿠키 기반 라우팅
- SSL 종료 가능
- 예: Nginx, AWS ALB, HAProxy L7 모드

로드밸런싱 알고리즘:
- Round Robin      : 순서대로 분산
- Weighted RR      : 가중치 비율로 분산
- Least Connections: 현재 연결 수 가장 적은 서버 우선
- IP Hash          : 클라이언트 IP 기반 고정 서버 배정
- Random           : 무작위 분산
```

### 4-3. NAT (Network Address Translation)

```
[SNAT - Source NAT]
내부 사설 IP → 외부 공인 IP로 변환
사용 사례: 내부 서버가 인터넷으로 요청할 때
10.0.0.5 → (NAT Gateway) → 52.10.20.30

[DNAT - Destination NAT]
외부 요청의 목적지 IP를 내부 서버 IP로 변환
사용 사례: 외부에서 내부 서버로 요청 들어올 때
52.10.20.30:443 → (NAT) → 10.0.0.5:443

[PAT - Port Address Translation]
NAT + 포트 매핑 (공유기의 포트 포워딩이 이것)
공인 IP:8080 → 사설 IP:8080
```

### 4-4. 방화벽 (Firewall)

```bash
# ufw (Ubuntu Firewall)
sudo ufw enable
sudo ufw allow 22/tcp    # SSH 허용
sudo ufw allow 80/tcp    # HTTP 허용
sudo ufw allow 443/tcp   # HTTPS 허용
sudo ufw deny 3306/tcp   # MySQL 외부 차단
sudo ufw status verbose

# iptables (저수준 방화벽)
# 들어오는 SSH 허용
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
# 들어오는 HTTP 허용
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
# 이미 수립된 연결 허용
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
# 나머지 차단
iptables -A INPUT -j DROP

# AWS Security Group 원칙
# 인바운드: 필요한 포트만 명시적 허용
# 아웃바운드: 기본 전체 허용 (필요 시 제한)
```

### 4-5. VPC와 서브넷 (AWS 기준)

```
[VPC 구조 예시]
VPC: 10.0.0.0/16

+-- Public Subnet: 10.0.1.0/24  (AZ-a)
|   +-- NAT Gateway
|   +-- Load Balancer
|   +-- Bastion Host
|
+-- Public Subnet: 10.0.2.0/24  (AZ-b)
|   +-- Load Balancer
|
+-- Private Subnet: 10.0.10.0/24 (AZ-a)
|   +-- App Server 1
|   +-- App Server 2
|
+-- Private Subnet: 10.0.11.0/24 (AZ-b)
|   +-- App Server 3
|
+-- DB Subnet: 10.0.20.0/24 (AZ-a, b)
    +-- RDS Primary
    +-- RDS Replica

핵심 원칙:
- Public Subnet: 인터넷 게이트웨이 연결, 외부 통신 가능
- Private Subnet: 외부 직접 접근 불가, NAT 통해서만 아웃바운드
- DB는 항상 Private에 배치
```

### 4-6. CDN (Content Delivery Network)

```
[CDN 동작 원리]

클라이언트 (서울) → CDN 엣지 노드 (서울)
    캐시 HIT → 즉시 응답
    캐시 MISS → 오리진 서버 요청 → 캐시 저장 → 응답

[CDN 사용 이유]
- 지리적으로 가까운 서버에서 응답 → 레이턴시 감소
- 오리진 서버 트래픽 분산
- DDoS 방어 (엣지에서 흡수)
- 정적 파일 (이미지, JS, CSS) 캐싱

[주요 CDN 서비스]
- AWS CloudFront
- Cloudflare
- Fastly
- Akamai

[캐시 무효화]
CDN 캐시 갱신 방법:
1. TTL 만료 대기
2. 파일명에 버전/해시 포함 (bundle.a3f2c1.js)
3. 강제 캐시 퍼지 (Invalidation)
```

---

## 5. HTTP 통신 심화

### 5-1. HTTP/1.1 vs HTTP/2 vs HTTP/3

| 항목 | HTTP/1.1 | HTTP/2 | HTTP/3 |
|------|----------|--------|--------|
| 전송 프로토콜 | TCP | TCP | QUIC (UDP 기반) |
| 멀티플렉싱 | 없음 (파이프라이닝 제한적) | 있음 (스트림 다중화) | 있음 |
| 헤더 압축 | 없음 | HPACK | QPACK |
| 서버 푸시 | 없음 | 있음 | 있음 |
| HOL Blocking | TCP 레벨 발생 | TCP 레벨 발생 | 없음 (스트림 독립) |
| 연결 설정 | 1 RTT | 1 RTT | 0 RTT (재연결) |

### 5-2. HTTPS 동작 원리 (TLS Handshake)

```
1. Client Hello
   → 지원하는 TLS 버전, 암호화 스위트 목록, 랜덤값 전송

2. Server Hello
   ← 선택된 TLS 버전, 암호화 스위트, 서버 인증서, 랜덤값 전송

3. Certificate Verify
   → 클라이언트가 서버 인증서를 CA(인증기관)로 검증

4. Key Exchange
   → Pre-Master Secret 생성, 서버 공개키로 암호화 전송
   ← 서버가 개인키로 복호화
   → 양측 Master Secret(세션키) 생성

5. Finished
   → 이후 통신은 대칭키(세션키)로 암호화

[인증서 체인]
Root CA → Intermediate CA → Server Certificate
브라우저는 Root CA 목록을 내장하여 신뢰 여부 판단
```

### 5-3. HTTP 상태 코드 실무 정리

```
[2xx - 성공]
200 OK            : 정상 응답
201 Created       : 리소스 생성 성공 (POST)
204 No Content    : 성공했지만 응답 본문 없음 (DELETE)

[3xx - 리다이렉션]
301 Moved Permanently : 영구 이동 (SEO에 영향, 브라우저 캐싱)
302 Found             : 임시 이동
304 Not Modified      : 캐시 유효 (ETag/Last-Modified 검증)

[4xx - 클라이언트 오류]
400 Bad Request   : 잘못된 요청 형식
401 Unauthorized  : 인증 필요
403 Forbidden     : 인가 거부
404 Not Found     : 리소스 없음
429 Too Many Requests : 요청 한도 초과 (Rate Limit)

[5xx - 서버 오류]
500 Internal Server Error : 서버 내부 오류
502 Bad Gateway           : 업스트림 서버 응답 이상 (Nginx→WAS 오류)
503 Service Unavailable   : 서버 과부하 or 점검 중
504 Gateway Timeout       : 업스트림 서버 응답 시간 초과
```

---

## 6. 인프라 트러블슈팅 명령어

### 6-1. 네트워크 진단

```bash
# 기본 연결 확인
ping 8.8.8.8                      # ICMP 패킷으로 연결 확인
traceroute google.com             # 패킷 경로 추적 (Linux)
tracert google.com                # Windows

# DNS 진단
dig google.com                    # DNS 조회
dig @8.8.8.8 google.com          # 특정 DNS 서버로 조회
nslookup google.com               # 대화형 DNS 조회

# 포트/소켓 진단
ss -tlnp                          # 열린 포트 목록
nc -zv 10.0.0.1 3306             # 특정 호스트:포트 연결 테스트
curl -v https://example.com      # HTTP 상세 응답
curl -I https://example.com      # HTTP 헤더만 출력
curl -w "%{time_total}" -o /dev/null -s https://example.com  # 응답시간 측정

# 인터페이스 및 라우팅
ip addr show                      # IP 인터페이스 목록
ip route show                     # 라우팅 테이블
route -n                          # 라우팅 테이블 (구버전)

# 패킷 캡처
tcpdump -i eth0 port 80          # eth0 인터페이스의 80 포트 캡처
tcpdump -i any host 10.0.0.5    # 특정 호스트 패킷 캡처
```

### 6-2. 서버 리소스 진단

```bash
# CPU / 메모리
top                               # 실시간 프로세스 모니터링
htop                              # 개선된 top
free -h                           # 메모리 사용량
vmstat 1 5                        # 1초 간격 5회 시스템 통계

# 디스크
df -h                             # 디스크 사용량
du -sh /var/log/*                 # 디렉토리별 용량
iostat -x 1                       # 디스크 I/O 통계

# 프로세스
ps aux | grep nginx               # nginx 프로세스 확인
lsof -i :80                       # 80포트 점유 프로세스
kill -HUP $(cat /var/run/nginx.pid)  # Nginx 무중단 재로드

# 로그 실시간 확인
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
journalctl -u nginx -f            # systemd 서비스 로그
```

### 6-3. 실무 장애 대응 흐름

```
[서비스 응답 없음 시 점검 순서]

1. 서비스 상태 확인
   systemctl status nginx
   systemctl status myapp

2. 포트 오픈 여부 확인
   ss -tlnp | grep :80
   ss -tlnp | grep :443

3. 업스트림 서버 확인
   curl -s http://10.0.0.1:8080/health
   curl -s http://10.0.0.2:8080/health

4. 로그 확인
   tail -100 /var/log/nginx/error.log
   journalctl -u myapp --since '10 minutes ago'

5. 리소스 확인
   free -h      # 메모리 부족?
   df -h        # 디스크 꽉 참?
   top          # CPU 과부하?

6. DNS 확인
   dig example.com
   dig @내부DNS서버 example.com

7. 방화벽 확인
   ufw status
   iptables -L -n
```

---

## 7. 영어로 배우는 인프라 핵심 용어

| 한국어 | English Term | 설명 |
|--------|-------------|------|
| 역방향 프록시 | Reverse Proxy | Sits in front of backend servers |
| 로드밸런서 | Load Balancer | Distributes incoming traffic |
| DNS 전파 | DNS Propagation | Time for DNS changes to spread globally |
| SSL 종료 | SSL Termination | Decrypting HTTPS at the proxy layer |
| 단일 장애점 | SPOF (Single Point of Failure) | Component whose failure stops everything |
| 고가용성 | HA (High Availability) | System designed to minimize downtime |
| 페일오버 | Failover | Switching to redundant system on failure |
| 오토스케일링 | Auto Scaling | Automatically adjust server capacity |
| 엣지 노드 | Edge Node | CDN server geographically close to user |
| 업스트림 | Upstream | Backend server receiving proxied requests |
| 헬스체크 | Health Check | Periodic test to verify server is alive |
| 인그레스 | Ingress | Entry point for incoming cluster traffic |

---

## 정리

인프라와 네트워크는 백엔드 개발자에게 선택이 아닌 필수 영역이다. 핵심은 다음과 같다.

- **DNS**: 도메인 → IP 변환 시스템, TTL 관리가 서버 이전의 핵심
- **Reverse Proxy**: 로드밸런싱, SSL 종료, 캐싱, 보안을 한 곳에서 처리
- **네트워크 계층**: OSI 모델을 이해하면 트러블슈팅이 체계적으로 가능
- **VPC/서브넷**: Public/Private 분리로 보안 기본 아키텍처 구성
- **트러블슈팅**: ping → traceroute → dig → curl → 로그 순서로 좁혀가는 습관
