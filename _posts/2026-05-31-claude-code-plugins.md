---
title: "Claude Code 추천 플러그인 목록"
date: 2026-05-31
categories: [도구, Claude Code]
tags: [mcp, vscode, plugin]
---

Claude Code를 더 강력하게 만들어주는 추천 플러그인 목록입니다.  
MCP 서버, VS Code 확장, 그리고 유용한 설정까지 정리했습니다.

---

## MCP 서버 (Model Context Protocol)

MCP는 Claude Code에 외부 도구를 연결하는 플러그인 시스템입니다.  
`claude mcp add` 명령어로 설치합니다.

### 파일 & 시스템

| 플러그인 | 설명 | 설치 |
|---|---|---|
| **filesystem** | 로컬 파일 시스템 접근 확장 | `claude mcp add filesystem` |
| **desktop-commander** | 터미널 명령 실행, 프로세스 관리 | `npx @wonderwhy-er/desktop-commander setup` |

### 개발 도구

| 플러그인 | 설명 | 설치 |
|---|---|---|
| **github** | PR 생성, 이슈 관리, 코드 리뷰 | `claude mcp add github` |
| **gitlab** | GitLab 연동 | `claude mcp add gitlab` |
| **postgres** | PostgreSQL 데이터베이스 직접 조회 | `claude mcp add postgres` |
| **sqlite** | SQLite 파일 읽기/쓰기 | `claude mcp add sqlite` |

### 검색 & 웹

| 플러그인 | 설명 | 설치 |
|---|---|---|
| **brave-search** | Brave 웹 검색 | `claude mcp add brave-search` |
| **puppeteer** | 브라우저 자동화, 스크린샷 | `claude mcp add puppeteer` |
| **fetch** | URL 콘텐츠 가져오기 | `claude mcp add fetch` |

### 협업 & 생산성

| 플러그인 | 설명 | 설치 |
|---|---|---|
| **slack** | Slack 메시지 읽기/전송 | `claude mcp add slack` |
| **google-drive** | Google Drive 파일 접근 | `claude mcp add google-drive` |
| **memory** | 대화 간 지식 그래프 유지 | `claude mcp add memory` |

---

## VS Code 확장

Claude Code VS Code 확장과 함께 쓰면 시너지가 좋은 확장들입니다.

### 필수

| 확장 | 설명 |
|---|---|
| **Claude Code** (공식) | VS Code 내 Claude Code 통합, 인라인 편집 |
| **GitLens** | 코드 라인별 git blame, 히스토리 시각화 |
| **Error Lens** | 오류/경고를 코드 라인에 바로 표시 |

### 코드 품질

| 확장 | 설명 |
|---|---|
| **ESLint** | JavaScript/TypeScript 린팅 |
| **Prettier** | 코드 자동 포맷터 |
| **SonarLint** | 코드 버그·취약점 실시간 감지 |

### 생산성

| 확장 | 설명 |
|---|---|
| **Todo Tree** | TODO/FIXME 주석 한눈에 관리 |
| **Thunder Client** | 경량 REST API 테스트 클라이언트 |
| **Docker** | 컨테이너 관리 UI |

---

## Claude Code 유용한 설정

### CLAUDE.md 프로젝트 지시파일

프로젝트 루트에 `CLAUDE.md`를 만들면 Claude Code가 항상 참조합니다.

```markdown
# 프로젝트 규칙

- 언어: TypeScript strict mode
- 테스트: Jest, 커버리지 80% 이상 유지
- 커밋: Conventional Commits 형식
- 주석: 한국어로 작성
```

### 자주 쓰는 슬래시 커맨드

| 커맨드 | 설명 |
|---|---|
| `/init` | CLAUDE.md 자동 생성 |
| `/code-review` | 현재 변경사항 코드 리뷰 |
| `/pr` | PR 생성 |
| `/help` | 도움말 |

---

## MCP 설치 예시

```bash
# GitHub MCP 설치 (토큰 필요)
claude mcp add github -e GITHUB_TOKEN=your_token

# 설치된 MCP 목록 확인
claude mcp list
```

---

도움이 됐다면 [GitHub](https://github.com/taengkim)에서 확인해보세요!
