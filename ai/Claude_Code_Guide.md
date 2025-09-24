# 🤖 Claude Code 완벽 가이드

> Claude Code를 효율적으로 활용하기 위한 종합 가이드


## 📥 설치 및 초기 설정

### 1️⃣ **설치**
```bash
npm install -g @anthropic-ai/claude-code
```

### 2️⃣ **프로젝트 시작**
```bash
cd your-project
claude
```

### 3️⃣ **권한 스킵 (선택사항)**
```bash
claude --dangerously-skip-permissions
```
> ⚠️ **주의**: 권한 확인 없이 모든 작업을 허용합니다. 신뢰할 수 있는 환경에서만 사용해야 한다.

<br/>

## 🎯 모델 선택 가이드

### 🌟 **Default 모드 (추천)**
- **특징**: 작업 복잡도에 따라 자동으로 모델 선택
- **비용 효율성**: 50% 사용량까지 Opus → 이후 Sonnet 전환
- **추천 대상**: 처음 사용자, 비용 절약 중시

### 💰 **요금제별 모델 지원**

| 요금제 | Opus 4 | Sonnet 4 | 비고 |
|--------|--------|----------|------|
| **Pro** ($20/월) | ❌ | ✅ | Sonnet 4만 사용 가능 |
| **Max** ($100-200/월) | ✅ | ✅ | 모든 모델 사용 가능 |

> 💡 **팁**: 복잡한 작업이 필요하면 Max 플랜 또는 API 크레딧 별도 구매

<br/>

## 🎮 필수 슬래시 명령어

### 🧹 **세션 관리**
```bash
/clear          # 대화 컨텍스트 완전 초기화 (⭐ 중요!)
/compact        # 대화 기록 요약 후 보존
/resume         # 과거 기록 복원하여 수정 가능
```

### 📊 **상태 확인**
```bash
/status         # Claude Code 상태 (버전, 모델, 계정 등)
/cost           # 현재 세션 토큰 사용량과 비용 확인
/help           # 모든 슬래시 명령어 목록
```

### 📝 **프로젝트 문서화**
```bash
/claude-md      # CLAUDE.md 파일 자동 생성
```
> 또는 "이 프로젝트에 대한 정보를 CLAUDE.md 파일로 만들어줘"라고 직접 요청 가능

<br/>

## 🐙 GitHub 연동 기능

### 🔧 **Git 작업**
```bash
/diff           # Git diff 보기
/commit         # Git 커밋 자동 생성
```

### 🤝 **GitHub 연동**
```bash
/install-github-app  # GitHub PR 자동 리뷰 설정
/bug                 # 버그 리포트 전송
```

<br/>

## 🚀 고급 활용 팁

### 📂 **파일 지정**
```bash
@ 기호 사용 시 특정 파일 지정 가능
```

### ⚠️ **자주 하는 실수**
1. **`/clear` 안 쓰기**: 토큰 낭비의 주범! 자주 사용하기
2. **Pro 플랜에서 Opus 4 기대**: Pro는 Sonnet 4만 지원
3. **권한 스킵 남용**: 보안상 위험할 수 있음

<br/>

## 🔌 MCP (Model Context Protocol) 확장

### 📚 **추천 MCP 서버들**

| 이름 | 용도 | 링크 |
|------|------|------|
| **Awesome MCP** | 종합 서버 모음 | [GitHub](https://github.com/punkpeye/awesome-mcp-servers) |
| **Official MCP** | 공식 서버들 | [GitHub](https://github.com/modelcontextprotocol/servers) |
| **Thinking** | 사고 과정 추적 | - |
| **Awesome Spring** | Spring 관련 최신 정보 | - |
| **Awesome Python** | Python 관련 최신 정보 | - |

> 💡 **팁**: "Awesome" 키워드가 붙은 MCP를 설치하면 해당 분야의 최신 파일과 정보까지 인식 가능!

<br/>

## 🎯 실전 사용 시나리오

### 📋 **프로젝트 초기 설정**
1. `claude` 명령어로 시작
2. `/claude-md`로 프로젝트 문서 자동 생성
3. 필요한 MCP 서버 설치
4. `/status`로 환경 확인

### 🔄 **개발 중 워크플로우**
1. 작업 시작 전 `/clear`로 컨텍스트 초기화
2. `@filename`으로 특정 파일 작업
3. 정기적으로 `/cost`로 사용량 체크
4. `/commit`으로 깔끔한 커밋 생성

### 🐛 **디버깅 및 리뷰**
1. `/diff`로 변경사항 확인
2. GitHub App 연동으로 자동 PR 리뷰
3. `/bug`로 이슈 리포팅

<br/>

## 🔗 참고 자료

### 📖 **공식 문서 및 가이드**
- [Claude Code 꿀팁 블로그](https://velog.io/@nara04040/Claude-Code-꿀팁)
- [Claude 확장 가이드](https://goddaehee.tistory.com/372)
- [Awesome MCP Servers](https://github.com/punkpeye/awesome-mcp-servers)
- [Official MCP Servers](https://github.com/modelcontextprotocol/servers)

<br/>

## ⚡ Quick Reference

### 🔥 **매일 사용하는 필수 명령어**
```bash
/clear    # 🧹 컨텍스트 초기화 (가장 중요!)
/status   # 📊 현재 상태 확인
/cost     # 💰 비용 확인
@file     # 📂 파일 지정
```

### 🎯 **프로젝트 관리**
```bash
/claude-md    # 📝 프로젝트 문서화
/commit       # 📦 Git 커밋
/diff         # 🔍 변경사항 확인
```
<br/>

> 💡 **Pro Tip**: `/clear`는 사용하는 게 좋기는 하지만 
`/compact` 를 활용해서 진행했던 내용을 요약하는 방향으로 사용하는 게 더 효율적이다.
>
> 🚀 **Advanced**: MCP 서버를 적극 활용하면 Claude Code의 진가를 경험할 수 있다! (단, 토큰 비용이 든다는 점을 감안해야 한다)
