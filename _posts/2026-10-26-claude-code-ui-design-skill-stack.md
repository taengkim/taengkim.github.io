---
title: "Claude Code UI 디자인 스킬 셋업 — 역할 분담까지"
date: 2026-10-26
categories: [AI, Claude Code]
tags: [claude-code, plugin, skill, ui, design, frontend]
---

## 세 줄 요약

1. AI가 만든 UI의 "AI 티"는 스킬 하나로는 안 잡힌다 — 아트 디렉션·구현·모션을 서로 다른 스킬에 맡기고 순서를 고정해야 한다.
2. 내 셋업은 Taste(방향) → Impeccable(구현) → 에밀 코왈스키(모션), 나머지는 보조다.
3. 겹치는 영역은 CLAUDE.md에 승자를 적어둬야 세션마다 결과가 안 달라진다 — 이 글의 본체는 그 분담표다.

---

AI로 UI를 만들면 묘하게 "AI 티"가 난다. 보라색 그라데이션, 어디서 본 듯한 히어로 섹션, 과한 이모지. 이걸 스킬 하나로 해결하려다 보면 아트 디렉션·구현·모션이 한 스킬 안에서 뭉개진다. 역할별로 스킬을 나누고 충돌 규칙을 CLAUDE.md에 박아두는 게 내가 정착한 셋업이다.

## 스택 한눈에

| 역할 | 스킬 | 담당 |
|---|---|---|
| 아트 디렉션 | Taste (taste-skill) | 룩앤필·무드·레이아웃 방향, "어떻게 보일지" |
| 구현·폴리시 | Impeccable | 방향을 코드로, 디자인 시스템 정렬, a11y·감사 |
| 모션 | 에밀 코왈스키 스킬군 | 애니메이션을 넣을지부터 성능까지 최종 판단 |
| 보조 | ui-ux-pro-max · hallmark · web-design-guidelines · frontend-design | 조회 DB·랜딩 통빌드·가이드라인 리뷰·일반 |

관건은 순서다. 방향(Taste) → 구현(Impeccable) → 모션(에밀). 역순이나 동시 투입은 서로 덮어쓴다.

## 각 스킬이 하는 일

### Taste — 아트 디렉션

One Page Love·Awwwards 큐레이션 사이트 247개를 연구해 만든 방향 라이브러리에서 아이디어·구도·타이포·팔레트·섹션 설계를 "딜"해주고, 엄격한 no-slop 하우스 스타일을 강제한다. 실제 브라우저 인스펙터로 스크린샷을 찍어 look-and-fix 루프를 돈다. 페이지가 템플릿 같다, AI가 만든 것 같다 싶을 때 투입한다.

### Impeccable — 구현·폴리시·감사

`/impeccable craft`(구현) · `polish`(다듬기) · `audit`(감사) · `critique` · `harden`(엣지케이스) · `typeset` · `layout` 커맨드 체계다. 위계·타이포·컬러·반응형·접근성을 코드 레벨에서 잡고, `DESIGN.md`·`PRODUCT.md`를 소유한다. 디자인 시스템 문서의 주인을 한 곳으로 정해두는 것 — 이게 생각보다 중요하다.

### 에밀 코왈스키 스킬군 — 모션 전담

Sonner·Vaul을 만든 디자인 엔지니어 Emil Kowalski의 철학을 담은 스킬 세트다.

| 스킬 | 용도 |
|---|---|
| `animate` | 애니메이션 설계+구현 — 넣을지부터 속성·커브·시간·중단 처리까지 |
| `review-animations` / `improve-animations` | 기존 모션 리뷰 / 코드베이스 전체 모션 감사 |
| `animation-performance` | 프레임 예산 — composite 속성, 메인스레드 밖 모션 |
| `apple-design` | 애플식 물리 기반 모션·제스처를 웹으로 |
| `mobile-native` | 웹앱을 폰에서 네이티브처럼 (100vh 버그, 탭 하이라이트 등) |
| `break-ui` | 최악의 데이터로 UI 스트레스 테스트 |
| `pick-ui-library` / `ask-sonner` | 라이브러리 선택 / Sonner 토스트 |

### 보조 스킬

- ui-ux-pro-max — 검색형 DB. 스타일 79종, 제품 팔레트 192개, 폰트 페어링 74개, UX 가이드라인 119개. 직접 디자인시키기보다 조회용으로 쓴다
- hallmark — 랜딩·마케팅 페이지 통째 빌드/리디자인 전용. 이름 부를 때만
- web-design-guidelines — 접근성·웹 인터페이스 가이드라인 준수 리뷰
- frontend-design (공식) — 위 셋이 안 다루는 일반 프론트엔드 디자인

## 셋팅

### 1. 플러그인 설치

```bash
# Claude Code 안에서
/plugin marketplace add Leonxlnx/taste-skill
/plugin install taste-skill@taste-skill

/plugin marketplace add pbakaus/impeccable
/plugin install impeccable@impeccable

/plugin marketplace add nextlevelbuilder/ui-ux-pro-max-skill
/plugin install ui-ux-pro-max@ui-ux-pro-max-skill

# 공식 마켓플레이스
/plugin install frontend-design@claude-plugins-official
```

에밀 코왈스키 스킬군·hallmark·web-design-guidelines는 플러그인이 아니라 개별 스킬이라 `~/.claude/skills/`에 설치된다 (skills CLI `npx skills add <owner>/<repo>` 방식 또는 claude.ai 스킬 동기화).

### 2. 역할 분담을 CLAUDE.md에

스킬이 겹치면 — 예를 들어 Impeccable에도 `animate` 커맨드가 있다 — 에이전트가 아무거나 집는다. 같은 요청인데 세션마다 다른 스킬이 잡히면 결과도 널뛴다. `~/.claude/CLAUDE.md`에 분담표를 박아두면 끝나는 문제다.

```markdown
## 디자인 스킬 역할 분담

| 역할 | 스킬 | 쓰는 때 |
|---|---|---|
| 아트 디렉션 | Taste | 룩앤필·무드·레이아웃 방향·테마·레퍼런스 결정 |
| 구현 폴리시·감사 | Impeccable | 방향을 코드로, 디자인 시스템 정렬, a11y·반응형, 출시 전 감사 |
| 모션 판단 | 에밀 코왈스키 스킬 | 애니메이션 여부·속성·커브·시간·성능. 모션 리뷰 |

규칙:
- 순서는 Taste → Impeccable → 에밀 코왈스키. 모션 최종 판단은
  Impeccable의 animate가 아니라 에밀 코왈스키 스킬이 한다.
- DESIGN.md·PRODUCT.md는 Impeccable이 소유한다.
- 나머지는 보조: hallmark는 이름 부를 때만, ui-ux-pro-max는 조회 DB로,
  web-design-guidelines는 가이드라인 리뷰로.
```

## 실전 워크플로 — 새 화면 하나 만들 때

1. Taste — "대시보드 방향 잡아줘" → 무드·레이아웃·팔레트·레퍼런스 확정
2. Impeccable — `/impeccable craft`로 구현 → `polish`로 다듬기 (DESIGN.md 갱신)
3. 에밀 — `find-animation-opportunities`로 모션 넣을 곳 찾고 `animate`로 구현
4. 출시 전 — `/impeccable audit` + `web-design-guidelines` 리뷰 + `break-ui`로 최악의 데이터 스트레스

## 마무리

스킬을 많이 까는 것보다 겹치는 영역의 승자를 미리 정해두는 게 품질을 좌우한다. 분담표 없이 쓰던 때는 같은 프롬프트에도 결과가 널뛰었는데, CLAUDE.md 한 블록 넣고 나서는 적어도 "누가 결정하는지"는 흔들리지 않는다.
