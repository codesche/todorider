# Windows NT 서버와 Active Directory — 구조와 동작 원리

Windows NT 커널 위에서 동작하는 서버 OS와, 그 위에 구축되는 디렉터리 서비스인 Active Directory(AD)의 내부 구조를 정리한다.
엔터프라이즈 환경에서 수백~수천 대의 서버와 사용자 계정을 관리할 때 AD가 어떻게 동작하는지 이해하면, 인프라 설계·보안 정책·자동화 스크립트 모두 훨씬 명확해진다.

---

## 1. Windows NT 아키텍처

### 1-1. NT 커널 구조

Windows NT는 **하이브리드 커널** 구조를 채택했다. 모놀리식 커널처럼 대부분의 OS 서비스를 커널 공간에서 실행하지만, 마이크로커널처럼 서버 프로세스를 유저 공간으로 분리하는 설계를 혼합했다.

```
[ 유저 모드 ]
  애플리케이션 프로세스
  Win32 서브시스템 (csrss.exe)
  POSIX / OS/2 서브시스템
  서비스 프로세스 (svchost.exe 등)

[ 커널 모드 ]
  Executive (I/O Manager, Object Manager, Security Reference Monitor, ...)
  NT 커널 (스케줄러, 인터럽트, 동기화)
  HAL (Hardware Abstraction Layer)
```

| 계층 | 역할 |
|---|---|
| HAL | 하드웨어 차이를 추상화. BIOS/UEFI, 인터럽트 컨트롤러 등 |
| NT 커널 | 스레드 스케줄링, 인터럽트 처리, 동기화 프리미티브 |
| Executive | I/O, 메모리, 보안, 객체 관리 서브시스템 |
| 서브시스템 | Win32, POSIX 등 — 유저 공간에서 API 계층 제공 |

### 1-2. NT 계열 서버 OS 계보

```
Windows NT 3.1 (1993)
  └─ NT 4.0 — Active Directory 전신 SAM 도입
       └─ Windows 2000 — AD 정식 도입, Kerberos 채택
            └─ Windows Server 2003/2008/2012/2016/2019/2022
```

모든 Windows Server는 NT 커널 계보다. NT 커널 버전 번호는 Windows 10/11과 공유된다.

### 1-3. SAM vs Active Directory

초기 NT는 **SAM(Security Account Manager)** 이라는 로컬 데이터베이스로 계정을 관리했다. SAM은 단순하지만 도메인 규모로 확장이 불가능하다.

| 항목 | SAM | Active Directory |
|---|---|---|
| 저장 위치 | 각 서버 로컬 레지스트리 | 도메인 컨트롤러 NTDS.DIT |
| 확장성 | 단일 머신 | 수백만 객체 |
| 인증 방식 | NTLM | Kerberos (기본) / NTLM (하위 호환) |
| 관리 단위 | 로컬 그룹 | OU, 그룹, GPO |

---

## 2. Active Directory 기본 개념

### 2-1. 도메인과 트러스트

**도메인(Domain)** 은 AD의 기본 관리 단위다. 동일 도메인 내 사용자·컴퓨터·서비스는 같은 보안 정책과 인증 데이터베이스를 공유한다.

```
example.com (루트 도메인)
  ├─ kr.example.com  (자식 도메인)
  └─ us.example.com  (자식 도메인)
```

**트러스트(Trust)** 는 한 도메인이 다른 도메인의 인증 결과를 신뢰하는 관계다.

| 트러스트 유형 | 설명 |
|---|---|
| 부모-자식 트러스트 | 같은 트리 내 자동 양방향 추이적 트러스트 |
| 트리-루트 트러스트 | 다른 트리 루트 간 양방향 추이적 트러스트 |
| 외부 트러스트 | 다른 포리스트 도메인과의 단방향/비추이적 |
| 포리스트 트러스트 | 포리스트 전체 간 양방향 |

### 2-2. 조직 구성 단위 (OU)

**OU(Organizational Unit)** 는 도메인 내부의 계층적 폴더 구조다. 사용자, 컴퓨터, 그룹, 다른 OU를 담을 수 있다.

```
example.com
  └─ OU=Korea
       ├─ OU=개발팀
       │    ├─ CN=홍길동 (사용자)
       │    └─ CN=DEV-PC-01 (컴퓨터)
       └─ OU=운영팀
```

OU의 핵심 역할은 **GPO(그룹 정책 개체) 적용 단위**가 된다는 것이다.

### 2-3. 포리스트와 트리

| 개념 | 설명 |
|---|---|
| **포리스트(Forest)** | AD 최상위 보안 경계. 스키마와 구성 파티션을 공유하는 도메인 트리들의 집합 |
| **트리(Tree)** | 연속된 DNS 네임스페이스를 공유하는 도메인의 계층 구조 |
| **도메인(Domain)** | 인증·정책의 기본 관리 단위 |

포리스트는 스키마를 공유하므로, 스키마 변경(예: Exchange 설치 시 속성 추가)은 전체 포리스트에 영향을 준다.

---

## 3. 도메인 컨트롤러 (DC)

### 3-1. DC의 역할

**도메인 컨트롤러(Domain Controller, DC)** 는 AD 데이터베이스를 호스팅하고 인증 요청을 처리하는 서버다.

- NTDS.DIT 데이터베이스 저장 및 복제
- Kerberos KDC(Key Distribution Center) 서비스 제공
- LDAP 서버로 디렉터리 쿼리 응답
- DNS와 연동해 서비스 레코드(SRV) 게시

### 3-2. FSMO 역할 (단일 마스터 작업)

AD는 기본적으로 **다중 마스터 복제**를 사용하지만, 충돌이 발생하면 안 되는 5가지 작업은 특정 DC에만 단독 부여된다.

| FSMO 역할 | 범위 | 설명 |
|---|---|---|
| Schema Master | 포리스트 | 스키마 변경 권한 |
| Domain Naming Master | 포리스트 | 도메인 추가/제거 권한 |
| PDC Emulator | 도메인 | 암호 변경, 시간 동기화, 하위 호환 |
| RID Master | 도메인 | 보안 ID(RID) 풀 할당 |
| Infrastructure Master | 도메인 | 도메인 간 객체 참조 업데이트 |

### 3-3. AD 데이터베이스 구조 (NTDS.DIT)

NTDS.DIT는 **ESE(Extensible Storage Engine)** 기반의 B-Tree 구조 데이터베이스다.

```
NTDS.DIT
  ├─ 스키마 파티션    — 모든 객체 클래스와 속성 정의 (포리스트 공유)
  ├─ 구성 파티션      — 사이트·서비스·복제 토폴로지 (포리스트 공유)
  └─ 도메인 파티션    — 사용자·컴퓨터·그룹 등 실제 객체 (도메인별)
```

---

## 4. 인증 프로토콜

### 4-1. Kerberos 인증 흐름

Windows 2000 이후 AD 도메인의 기본 인증 프로토콜은 **Kerberos v5**다. 패스워드를 네트워크로 직접 전송하지 않고 티켓 기반으로 동작한다.

```
[클라이언트]          [KDC (DC)]               [서비스 서버]
    │                    │                          │
    │── AS-REQ ─────────▶│  (사전 인증 포함)        │
    │◀─ AS-REP ──────────│  TGT 발급               │
    │                    │                          │
    │── TGS-REQ ────────▶│  (TGT 제시)             │
    │◀─ TGS-REP ─────────│  서비스 티켓 발급        │
    │                    │                          │
    │── AP-REQ ──────────────────────────────────▶│
    │◀─ AP-REP ───────────────────────────────────│
    │                    │                          │
```

| 단계 | 설명 |
|---|---|
| AS-REQ/AS-REP | 클라이언트가 KDC에서 TGT(Ticket Granting Ticket) 발급 |
| TGS-REQ/TGS-REP | TGT를 제시하고 특정 서비스의 서비스 티켓 발급 |
| AP-REQ/AP-REP | 서비스 서버에 서비스 티켓 제시, 상호 인증 |

Kerberos의 핵심은 **KDC만 신뢰하면 클라이언트와 서버가 서로를 검증**할 수 있다는 점이다.

### 4-2. NTLM vs Kerberos

| 항목 | NTLM | Kerberos |
|---|---|---|
| 방식 | Challenge-Response (해시 전송) | 티켓 기반 (비밀키 불전송) |
| DC 관여 | 매 인증마다 DC 검증 | TGT 유효 시간 내 DC 불필요 |
| 상호 인증 | 지원 안 함 | 지원 |
| 사용 상황 | 로컬 계정, 도메인 외부, IP 직접 접속 | 도메인 계정, DNS 이름 사용 시 |
| 취약점 | Pass-the-Hash, NTLM Relay 공격 | Golden Ticket, Kerberoasting |

> **실무 포인트**: IP 주소로 서버에 접속하면 Kerberos를 사용할 수 없어 NTLM으로 폴백된다. 가능하면 항상 FQDN(DNS 이름)으로 접속해야 보안이 강화된다.

---

## 5. 그룹 정책 (GPO)

### 5-1. GPO 동작 방식

**GPO(Group Policy Object)** 는 도메인 내 컴퓨터와 사용자에게 설정을 일괄 배포하는 메커니즘이다.

```
[ GPO 적용 순서 — LSDOU ]

Local GPO
  └─ Site GPO
       └─ Domain GPO
            └─ OU GPO (계층 순서대로, 하위 OU가 마지막)
```

나중에 적용된 GPO가 충돌 시 우선한다. OU GPO > Domain GPO 순이다.

### 5-2. GPO 저장 구조

| 구성요소 | 위치 | 역할 |
|---|---|---|
| GPC (Group Policy Container) | AD 내부 (NTDS.DIT) | GPO 메타데이터, 버전 정보 |
| GPT (Group Policy Template) | SYSVOL 공유 폴더 | 실제 설정 파일 (레지스트리, 스크립트 등) |

클라이언트는 로그인 시 SYSVOL에서 GPT를 다운로드해 적용한다. SYSVOL은 모든 DC에 복제된다.

### 5-3. 자주 쓰이는 GPO 설정 예시

```
컴퓨터 구성 > 정책 > Windows 설정 > 보안 설정
  ├─ 암호 정책: 최소 길이 12자, 복잡성 조건 필수
  ├─ 계정 잠금 정책: 5회 실패 시 30분 잠금
  └─ 감사 정책: 로그인 성공/실패 이벤트 기록

사용자 구성 > 정책 > 관리 템플릿
  └─ 제어판, 시작 메뉴, 작업 표시줄 제한
```

---

## 6. AD 복제

### 6-1. 복제 토폴로지

AD는 **사이트(Site)** 개념으로 네트워크를 분리한다. 사이트 내 복제는 자동이고 빠르며, 사이트 간 복제는 **사이트 링크** 설정을 통해 스케줄과 대역폭을 제어한다.

```
서울 사이트 (DC-SEO-01, DC-SEO-02)
  │
  │ 사이트 링크 (야간 복제)
  │
부산 사이트 (DC-PUS-01)
```

### 6-2. USN 기반 복제

AD 복제는 **USN(Update Sequence Number)** 으로 변경 사항을 추적한다. 각 DC는 자신의 변경마다 USN을 증가시키고, 다른 DC는 마지막으로 동기화한 USN 이후의 변경만 요청한다.

충돌이 발생하면 **타임스탬프와 버전 번호**로 최신 값을 선택한다. 이를 **LWW(Last Writer Wins)** 정책이라 한다.

---

## 7. 실무에서 자주 마주치는 문제들

### 7-1. 시간 동기화 문제

Kerberos는 클라이언트와 DC 간 시각 차이가 **5분 이내**여야 작동한다. 시간이 어긋나면 인증 실패가 발생한다.

```powershell
# 도메인 컨트롤러와 시간 동기화 강제
w32tm /resync /force

# 시간 동기화 상태 확인
w32tm /query /status
```

### 7-2. SPN 중복 문제

**SPN(Service Principal Name)** 은 Kerberos가 서비스를 식별하는 이름이다. SPN이 중복 등록되면 Kerberos 인증이 실패하고 NTLM으로 폴백된다.

```powershell
# SPN 중복 조회
setspn -X

# 특정 계정의 SPN 목록 확인
setspn -L DOMAIN\서비스계정명
```

### 7-3. 복제 충돌 확인

```powershell
# 복제 상태 점검
repadmin /replsummary

# 복제 오류 목록
repadmin /showrepl
```

### 7-4. 계정 잠금 원인 추적

계정이 반복 잠금될 때 원인 머신을 찾는 방법이다.

```powershell
# PDC Emulator DC에서 이벤트 로그 검색
# 이벤트 ID 4740 = 계정 잠금 발생
Get-WinEvent -ComputerName PDC명 -FilterHashtable @{
    LogName = 'Security'
    Id = 4740
} | Select-Object TimeCreated, Message
```

---

## 8. 보안 관점에서 AD 이해하기

### 8-1. AD 공격 주요 기법

| 공격 기법 | 설명 | 대응 |
|---|---|---|
| Pass-the-Hash | NTLM 해시를 탈취해 인증 우회 | Credential Guard 활성화, NTLM 제한 |
| Kerberoasting | 서비스 계정 티켓을 오프라인 크래킹 | 서비스 계정에 강한 패스워드, Managed Service Account 사용 |
| Golden Ticket | KRBTGT 계정 해시 탈취 → 임의 TGT 생성 | KRBTGT 패스워드 정기 갱신, 비정상 티켓 수명 모니터링 |
| DCSync | DC 복제 권한 악용해 해시 덤프 | 복제 권한 최소화, AD 감사 로그 |

### 8-2. 권한 최소화 원칙

```
나쁜 예: 모든 서비스를 Domain Admins 계정으로 실행
  → 서비스 하나 침해 = 도메인 전체 장악

좋은 예: 서비스별 전용 서비스 계정 (gMSA 사용)
  → 침해 범위를 해당 서비스로 제한
```

**gMSA(group Managed Service Account)** 는 패스워드를 DC가 자동 관리하므로, 서비스 계정 패스워드 노출 위험이 없다.

---

## 9. 정리

| 개념 | 핵심 요약 |
|---|---|
| Windows NT 커널 | 하이브리드 커널. HAL → NT 커널 → Executive → 서브시스템 계층 |
| Active Directory | LDAP 기반 디렉터리 서비스. 도메인·OU·GPO·복제로 구성 |
| 도메인 컨트롤러 | NTDS.DIT 저장, Kerberos KDC, LDAP 서버 역할 |
| Kerberos | 티켓 기반 인증. TGT → 서비스 티켓 → 서비스 접근 |
| NTLM | 해시 기반 레거시 인증. IP 접속·로컬 계정 시 사용. 보안 취약 |
| GPO | LSDOU 순서로 적용. SYSVOL 통해 배포 |
| AD 복제 | USN 기반 다중 마스터. 사이트로 WAN 대역폭 제어 |
| 보안 | Kerberoasting, Golden Ticket, DCSync 주요 공격 벡터 |

Active Directory는 단순한 사용자 관리 도구가 아니다. 기업 인프라 전체의 인증·권한·정책 허브다. 이 구조를 이해하면 SSO 연동, LDAP 쿼리 최적화, 인프라 자동화 모두 훨씬 명확해진다.
