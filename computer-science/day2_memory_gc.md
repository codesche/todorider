# Day 2 — OS: 메모리와 Garbage Collection

## 1. 핵심 요약
- **메모리 구조**
  - 코드(Code): 실행 명령 저장
  - 데이터(Data): 전역/정적 변수 저장
  - 힙(Heap): 동적 할당, GC 관리
  - 스택(Stack): 함수 호출 시 지역 변수/매개변수 저장

- **가상 메모리 (Virtual Memory)**
  - 실제 메모리(RAM)를 추상화하여 더 큰 메모리 공간 제공
  - 페이징(Paging): 프로세스를 고정 크기 페이지 단위로 나눠 관리
  - 페이지 폴트(Page Fault): 필요한 페이지가 메모리에 없어 디스크에서 로드해야 할 때 발생

- **Java/JVM 메모리 구조**
  - Heap: 객체 저장 (Young/Old 영역)
  - Stack: 메서드 호출 스택, 지역 변수
  - Metaspace: 클래스 메타데이터 저장 (Java 8 이후 PermGen 대체)
  - PC Register, Native Method Stack 포함

- **Garbage Collection (GC)**
  - 불필요한 객체를 자동 회수
  - Stop-the-world: GC 동작 중 애플리케이션 멈춤
  - Minor GC: Young 영역 정리 (빠름)
  - Major/Full GC: Old 영역 정리 (느림, STW 길어짐)
  - 알고리즘: Mark-Sweep, Mark-Compact, Copying
  - 최신: G1 GC, ZGC → STW 최소화

- **.NET 비교**
  - .NET도 세대별 GC(Gen0, Gen1, Gen2) → Java Young/Old와 유사
  - 차이는 메모리 관리 구현 방식, 개발자 관점에서 개념은 거의 동일

---

## 2. 면접형 Q&A 예시

**Q1. 가상 메모리가 필요한 이유는 무엇인가요?**  
👉 실제 메모리보다 큰 주소 공간을 제공하고, 프로세스 간 메모리 보호를 보장하기 위해 사용합니다. 이를 통해 프로그램은 독립적으로 동작하며 하드웨어 자원을 효율적으로 활용할 수 있습니다.

**Q2. Java에서 GC의 기본 동작 원리를 설명해주세요.**  
👉 GC는 더 이상 참조되지 않는 객체를 찾아 제거합니다. Young 영역에서 Minor GC, Old 영역에서 Major GC가 실행되며, Stop-the-world가 발생할 수 있습니다. 최신 GC 알고리즘(G1, ZGC)은 STW 시간을 줄이는 데 초점을 맞춥니다.

**Q3. Stop-the-world란 무엇인가요?**  
👉 GC가 실행될 때 모든 애플리케이션 스레드를 멈추는 현상입니다. 성능 저하의 주요 원인으로, 튜닝 및 최신 GC 알고리즘을 통해 최소화합니다.
