# CLAUDE.md — Multi-Agent 코드 리뷰 워크플로우

## 프로젝트 개요

이 프로젝트는 GitHub PR이 열리면 세 가지 전문 에이전트가 **병렬로** 코드를 분석하고
리뷰 결과를 PR 코멘트로 자동 게시하는 Multi-Agent 코드 리뷰 시스템입니다.

---

## 에이전트 구성

에이전트 정의 파일은 `.claude/agents/` 디렉토리에 있습니다.

| 에이전트 | 파일 | 담당 영역 |
|---|---|---|
| security-reviewer | `.claude/agents/security-reviewer.md` | SQL Injection, XSS, 인증 취약점, 하드코딩 시크릿 |
| performance-reviewer | `.claude/agents/performance-reviewer.md` | N+1 쿼리, 알고리즘 복잡도, 불필요한 재렌더링 |
| style-reviewer | `.claude/agents/style-reviewer.md` | 네이밍 컨벤션, 데드 코드, 주석 품질, 복잡도 지표 |

---

## 워크플로우 실행 규칙

### PR 리뷰 트리거
- GitHub Actions(`claude-review.yml`)가 `pull_request` 이벤트에서 워크플로우를 실행합니다.
- 오케스트레이터는 `gh pr diff {pr_number}`로 diff를 수신합니다.

### 병렬 실행 및 격리
- 세 에이전트는 `parallel()` 로 동시에 실행됩니다.
- 각 에이전트는 `isolation: 'worktree'` 로 독립된 git worktree에서 동작합니다.
- 에이전트가 파일을 수정하지 않으면 worktree는 자동으로 삭제됩니다.

### 결과 취합 및 코멘트

#### 1. 중복 이슈 제거
- 동일 파일·라인(`파일경로:라인번호`)을 지적하는 이슈가 여러 에이전트에서 중복 보고된 경우, **심각도가 가장 높은 이슈 하나만 유지**합니다.
- 심각도가 동일하면 security > performance > style 에이전트 순으로 우선합니다.

#### 2. 우선순위 매트릭스
중복 제거 후 남은 이슈 각각에 **수정 난이도**와 **영향도**를 부여하고 아래 매트릭스로 우선순위를 산정합니다.

| 수정 난이도 \ 영향도 | 높음 | 중간 | 낮음 |
|---|---|---|---|
| **쉬움** | P1 (즉시 수정) | P2 | P3 |
| **중간** | P1 (즉시 수정) | P2 | P3 |
| **어려움** | P2 | P3 | P4 (백로그) |

- **수정 난이도 기준**
  - 쉬움: 한 줄 수정, 상수 추출, 파일명 변경 등 5분 이내 처리 가능
  - 중간: 함수 분리, 쿼리 리팩터링 등 30분 이내 처리 가능
  - 어려움: 아키텍처 변경, 외부 의존성 도입 등 1시간 이상 소요
- **영향도 기준**
  - 높음: 보안 취약점, 앱 동작 불가, 데이터 손실 위험
  - 중간: 성능 저하, 유지보수 부채 누적
  - 낮음: 코드 스타일, 가독성, 선호도 문제

#### 3. 높은 심각도 요약 섹션
`final.md` 최상단에 `## 즉시 수정 필요 (HIGH)` 섹션을 별도로 추가합니다.
- 높음 심각도 이슈만 추출하여 나열합니다.
- 이슈가 없으면 `높은 심각도 이슈가 없습니다.`를 기재합니다.
- HIGH severity 발견 시 `gh pr review --request-changes`를 자동 실행합니다.

### 결과 파일 저장 규칙

각 에이전트는 분석 완료 후 결과를 아래 경로에 마크다운 파일로 저장합니다.

| 에이전트 | 저장 경로 |
|---|---|
| security-reviewer | `/tmp/review/security.md` |
| performance-reviewer | `/tmp/review/performance.md` |
| style-reviewer | `/tmp/review/style.md` |

오케스트레이터는 세 파일을 모두 읽은 뒤 severity 순으로 취합하여 최종 리뷰를 `/tmp/review/final.md`에 작성합니다.

```
/tmp/review/
├── security.md      # security-reviewer 결과
├── performance.md   # performance-reviewer 결과
├── style.md         # style-reviewer 결과
└── final.md         # 오케스트레이터 최종 취합본
```

**저장 형식 규칙:**
- 각 에이전트 결과 파일은 `[높음/중간/낮음] 파일명:라인번호 - 제목` 형식을 유지합니다.
- `/tmp/review/` 디렉토리가 없으면 저장 전에 생성합니다(`mkdir -p /tmp/review`).
- `final.md`는 세 파일이 모두 존재할 때만 작성합니다. 누락된 파일이 있으면 오케스트레이터가 오류를 기록합니다.

**`final.md` 출력 구조:**
```markdown
## 즉시 수정 필요 (HIGH)
<!-- 높은 심각도 이슈만 추출한 별도 요약 -->

## 우선순위 매트릭스
<!-- P1~P4 분류표: 수정 난이도 × 영향도 -->

## 전체 이슈 목록
<!-- 중복 제거 후 severity 순(높음→중간→낮음) 정렬 -->
```

---

## 파일 구조

```
my-project/
├── .claude/
│   ├── settings.json          # Claude Code 권한 및 환경 설정
│   └── agents/
│       ├── security-reviewer.md    # 보안 에이전트 시스템 프롬프트
│       ├── performance-reviewer.md # 성능 에이전트 시스템 프롬프트
│       └── style-reviewer.md       # 스타일 에이전트 시스템 프롬프트
├── .github/
│   └── workflows/
│       └── claude-review.yml  # GitHub Actions CI 정의
└── CLAUDE.md                  # 이 파일 — 워크플로우 지시 및 프로젝트 개요
```

---

## 개발 가이드라인

- 에이전트 프롬프트를 수정할 때는 `.claude/agents/` 내 해당 파일만 편집합니다.
- 새 에이전트 추가 시 이 파일의 에이전트 구성 표도 함께 업데이트합니다.
- `settings.json`에서 에이전트별 허용 tool 목록을 관리합니다.
- 리뷰 결과 스키마 변경은 워크플로우 스크립트와 이 문서에 동시에 반영합니다.
