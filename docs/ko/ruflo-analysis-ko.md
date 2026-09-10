# Ruflo 분석 및 활용 가이드 (한국어)

> 이 문서는 Ruflo 저장소를 직접 열어보고 정리한 분석 노트입니다.
> 공식 저장소: https://github.com/ruvnet/ruflo
> npm: https://www.npmjs.com/package/ruflo
> 라이선스: MIT (상업적 이용 가능, 저작권 표시 필요)
> 분석 시점 기준 버전: 3.40.0

---

## 1. Ruflo가 뭔가요?

한 줄 요약: **Claude Code를 "혼자 일하는 AI"에서 "팀으로 일하는 AI"로 바꿔주는 확장 레이어**입니다.

저장소 README의 핵심 문장:

> **Agent = Model + Harness.**
> 모델은 글을 쓰고, 하네스는 도구·기억·루프·안전장치를 준다. **Ruflo가 그 하네스다.**

### 비유

| 구성 요소 | 비유 |
|---|---|
| Claude (모델) | 머리 좋은 개발자 한 명 |
| Claude Code | 그 사람에게 준 노트북과 터미널 |
| **Ruflo** | 그 사람 주변에 **팀 · 회의실 · 프로젝트 노트 · 업무 프로세스**를 깔아주는 것 |

### 배경

- 원래 이름은 `claude-flow`, 현재는 **Ruflo**로 리브랜딩
- 제작자: `rUv` (ruvnet)
- npm 패키지 3종 배포: `ruflo` (래퍼), `claude-flow` (엄브렐라), `@claude-flow/cli` (구현체)

---

## 2. 폴더 구조 분석 (실측)

```
ruflo/
├── .claude/              # Claude Code에 직접 주입되는 설정
│   ├── agents/           # 에이전트 정의 108개
│   ├── commands/         # 슬래시 커맨드 168개
│   ├── skills/           # 스킬 39개
│   ├── settings.json     # 훅(hooks) + 권한 설정  ★핵심
│   └── helpers/          # 훅 실행 스크립트
│
├── v3/@claude-flow/      # 실제 소스코드 (TypeScript, 25개 패키지)
│   ├── cli/              # CLI 본체 (26개 명령어, 140+ 서브커맨드)
│   ├── memory/           # AgentDB + HNSW 벡터 메모리
│   ├── swarm/            # 멀티 에이전트 오케스트레이션
│   ├── mcp/              # MCP 서버 구현
│   ├── security/         # 입력 검증, CVE 대응
│   ├── neural/           # SONA 학습 시스템
│   └── providers/        # anthropic / openai / google / cohere / ollama
│
├── crates/               # Rust 코드 (성능 필요 구간)
├── plugins/              # 플러그인 40개
├── .claude-plugin/       # 플러그인 마켓플레이스 정의
├── docs/                 # 문서 (ruflo-explained.md 추천)
└── ruflo/                # npx로 실행되는 얇은 래퍼 + 웹 UI 소스
```

### 핵심 기능 3가지

- **스웜(Swarm)** — 아키텍트 → 코더 → 테스터 → 리뷰어 순서로 에이전트가 릴레이하며 작업
- **메모리** — 세션이 끝나도 기억 유지. 벡터 검색으로 과거 결정/맥락 재사용
- **훅(Hooks)** — 파일 수정 시 자동으로 포맷팅 / 학습 / 기록 (`settings.json`의 `PreToolUse`, `PostToolUse`)

---

## 3. 언제 쓰면 좋은가

| 쓰면 좋음 | 쓰지 말 것 |
|---|---|
| 파일 3개 이상 건드리는 기능 개발 | 오타 수정, 1~2줄 버그 |
| 대규모 리팩토링 | 단순 질문 |
| 여러 관점이 필요한 작업(보안 점검 등) | 한 파일만 고치는 작업 |
| 여러 세션에 걸친 장기 프로젝트 | 일회성 작업 |

저장소 문서(`docs/ruflo-explained.md`)의 표현:

> "두 줄 편집이면 그냥 에이전트 하나로 해라."

---

## 4. 설치 및 사용법

### 방법 A — 플러그인 설치 (가장 가벼움, 추천)

Claude Code 안에서:

```
/plugin marketplace add ruvnet/ruflo
/plugin install ruflo-core@ruflo
```

- 슬래시 명령어 + 에이전트 정의 추가
- **프로젝트 폴더에 파일이 생기지 않음**
- `ruflo-core`는 자체 MCP 서버도 함께 등록
- 제거: `/plugin uninstall ruflo-core`

필요한 것만 추가:

```
/plugin install ruflo-swarm@ruflo           # 팀 협업
/plugin install ruflo-rag-memory@ruflo      # 기억력
/plugin install ruflo-security-audit@ruflo  # 보안 검사
/plugin install ruflo-cost-tracker@ruflo    # 비용 추적
```

### 방법 B — CLI 풀 설치 (전체 기능)

```bash
npx ruflo@latest init wizard   # 대화형 마법사 (전 플랫폼 동작)
npx ruflo@latest init          # 비대화형 빠른 설치
npx ruflo@latest doctor        # 건강 체크
```

생성되는 것:

```
내프로젝트/
├── .claude/         # 에이전트, 커맨드, 스킬, 훅
├── .claude-flow/    # 설정
├── CLAUDE.md        # 프로젝트 규칙
└── .mcp.json        # MCP 서버 등록
```

> 주의: 파일이 많이 생성되고 기존 `CLAUDE.md`를 덮어쓸 수 있습니다.
> **반드시 git commit 후 실행하세요.**

### 방법 C — MCP만 연결 (중간)

```bash
claude mcp add ruflo -- npx ruflo@latest mcp start
```

파일 생성 없이 도구만 연결됩니다.

### 자주 쓰는 명령어

```bash
npx ruflo status
npx ruflo doctor

npx ruflo memory store --key "auth-design" --value "JWT + refresh token 방식"
npx ruflo memory search -q "인증 방식"

npx ruflo swarm init --topology hierarchical --max-agents 6

npx ruflo plugins list
```

---

## 5. 플러그인인가, 스킬인가, MCP인가? → 전부 다

| 형태 | 정체 | 하는 일 | 파일 생성 |
|---|---|---|---|
| **스킬** | 설명서 텍스트 | "이럴 땐 이렇게 해" 안내 | 없음 |
| **플러그인** | 스킬 + 명령어 + 에이전트 묶음 | 슬래시 명령어 추가 | 없음 |
| **MCP** | 실제 동작하는 서버 | 메모리 저장, 검색, 실행 | 설정만 |
| **CLI** | 실제 프로그램 | 전부 | 많음 |

비유:

- 스킬 = 레시피 종이
- 플러그인 = 레시피 + 도구 세트 상자
- MCP = 실제 주방 (전기 들어옴)
- CLI = 식당 통째로

```
가벼움 ──────────────────────────────► 무거움
 스킬만    플러그인    MCP연결    CLI풀설치
```

Ruflo는 이 4가지 형태 **모두**로 배포되므로, 필요한 만큼만 선택하면 됩니다.

---

## 6. API 토큰이 필요한가? → 기본은 필요 없음

| 상황 | API 키 필요 여부 |
|---|---|
| Claude Code에서 플러그인으로 사용 | 불필요 (구독으로 커버) |
| CLI로 메모리 / 스웜 사용 | 불필요 |
| `claude -p` 백그라운드 실행 | 불필요 (Claude Code 로그인 사용) |
| OpenAI / Gemini 병행 사용 | 해당 사업자 키 필요 |
| 웹 UI 직접 호스팅 | OpenRouter 등 키 필요 |

Ruflo는 자체적으로 모델을 호출하지 않고, **Claude Code가 이미 인증한 세션을 그대로 사용**합니다.

`v3/@claude-flow/providers/`에 anthropic, openai, google, cohere, ollama 어댑터가 있지만 **전부 선택 사항**입니다.
`ollama`를 쓰면 로컬 모델로 무료 운용도 가능합니다.

---

## 7. 왜 GitHub에서 유명한가

### 실측 데이터

- npm 다운로드: **12개월 기준 약 956만 회** (`data/clone-data.proof.json`)
- 버전 3.40.0까지 릴리스 — 개발이 매우 활발
- 에이전트 108개 / 명령어 168개 / 스킬 39개 / 플러그인 40개

### 유명해진 이유

1. **타이밍** — Claude Code 출시 직후 "확장 방법"을 최초로 체계화 (선점 효과)
2. **컨셉** — "AI 하나 vs AI 팀" 이라는 그림이 SNS에서 확산되기 좋았음
3. **규모감** — "108 에이전트 / 314 MCP 툴 / 40 플러그인" 이라는 숫자가 공유를 유발
4. **MIT 라이선스** — 진입 장벽이 없음
5. **포장 완성도** — 배지, GIF, 랜딩 페이지, 웹 UI까지 갖춘 마케팅

### 냉정한 평가

- **좋은 점**: 에이전트 프롬프트 품질이 높고, ADR(아키텍처 결정 기록) 문서가 380건 이상으로 매우 충실
- **아쉬운 점**: 기능이 지나치게 많아 진입 난이도가 높음. 과거 "150배 빠름" 같은 과장 수치를 이후 실측치로 스스로 정정한 이력이 있음
- **현실**: 대부분의 사용자는 전체 기능의 10%도 사용하지 않음

참고로 저장소 스스로 이렇게 적어두었습니다:

> README의 기능 목록은 "조사해볼 지도"이지, 당신의 설치 환경에서 전부 동작한다는 주장이 아니다.

측정치 정정 사례:

| 항목 | 과거 주장 | 현재 실측 |
|---|---|---|
| HNSW 검색 | 150x ~ 12,500x | 1.9x (N=20k), 3.2~4.7x (N=5k) |
| Flash Attention | 2.49x ~ 7.47x | **측정 불가로 삭제** |

---

## 8. 로컬 에이전트 구축에 도움이 되는가 → 매우 그렇다

이 저장소의 **가장 큰 실질적 가치**입니다.

### 보물 1 — 에이전트 프롬프트 108개

```bash
cat .claude/agents/core/coder.md
cat .claude/agents/core/tester.md
```

"AI에게 역할을 부여하는 방법"의 실전 예시집입니다. MIT 라이선스이므로 참고·복제·수정이 모두 가능합니다.

### 보물 2 — 훅(Hook) 시스템

`.claude/settings.json`:

```json
"hooks": {
  "PreToolUse":  [ /* 실행 전 개입 */ ],
  "PostToolUse": [ /* 실행 후 자동 처리 */ ]
}
```

응용 예시:
- 파일 저장 시 자동 포맷팅
- 커밋 전 자동 테스트
- 위험한 명령어 자동 차단

### 보물 3 — 아키텍처 참고처

| 배울 것 | 위치 |
|---|---|
| MCP 서버 구현 | `v3/@claude-flow/mcp/` |
| 벡터 메모리 구현 | `v3/@claude-flow/memory/` |
| 에이전트 오케스트레이션 | `v3/@claude-flow/swarm/` |
| 플러그인 제작 | `plugins/ruflo-plugin-creator/` |

### 추천 학습 순서

1. `.claude/agents/` 읽기 — 프롬프트 감각 익히기
2. `settings.json` 훅 따라 만들어보기
3. 작은 MCP 서버 하나 직접 만들기
4. 플러그인으로 패키징

> 통째로 도입하기보다 **필요한 부분만 뜯어서 학습**하는 편이 효율적입니다.

---

## 9. 수익화 아이디어

MIT 라이선스이므로 상업적 이용이 자유롭습니다 (저작권 고지 필요).

### 단기 — 지금 바로 가능

**A. 한국어 특화 에이전트 팩**
- 한국식 코드 리뷰 에이전트 (존댓말, 문화 반영)
- 국내 서비스 API 연동 에이전트
- 한국 스타트업 문서 템플릿
- 판매처: Gumroad, 크몽, 인프런

**B. 강의 / 전자책**
- "Claude Code 200% 활용법 — AI 팀 만들기"
- 국내에 해당 주제 자료가 거의 없어 선점 가능
- 플랫폼: 인프런, 유데미, 클래스101

**C. 세팅 대행 서비스**
- 중소기업 대상 사내 AI 개발환경 구축
- Ruflo 커스터마이징 + 사내 문서 학습

### 중기 — 2~3개월

**D. 도메인 특화 플러그인** (Ruflo에 없는 영역)
- 커머스 (쇼핑몰 개발 자동화)
- 공공기관 (전자정부 표준 프레임워크)
- 게임 개발 (Unity 연동)
- 모델: 월 구독

**E. SaaS 대시보드**
- 팀 AI 사용량 / 비용 모니터링 서비스
- `ruflo-cost-tracker`를 웹 서비스화
- AI 비용 관리 수요는 실제로 매우 큼

### 장기

**F. 완전 오프라인 로컬 버전**
- 보안상 외부 AI를 못 쓰는 조직 대상
- Ollama + Ruflo = 사내망 전용 AI 팀
- 금융 / 의료 / 공공 시장, 단가가 높음

### 주의점

| 주의 | 이유 |
|---|---|
| "Ruflo 기반" 명시 | MIT 라이선스 의무 |
| 이름을 그대로 사용하지 않기 | 상표 이슈 가능성 |
| 과장 광고 답습 금지 | 소비자 신뢰 |
| 차별점 확보 | 단순 재포장은 시장성이 없음 |

---

## 10. React나 PHP로 만들 수 있는가

결론: **부분적으로 가능**합니다. 계층을 나눠보면 명확해집니다.

```
┌──────────────────────────────┐
│  3층: UI (대시보드, 채팅)      │  React ○   PHP ○
├──────────────────────────────┤
│  2층: API / 오케스트레이션     │  React ×   PHP △
├──────────────────────────────┤
│  1층: MCP 서버 + Claude 연동   │  React ×   PHP △
└──────────────────────────────┘
```

### React

**가능**
- 에이전트 상태 대시보드
- 채팅 UI (`flo.ruv.io`가 실제 사례)
- 스웜 시각화, 비용 / 토큰 모니터링

**불가능**
- 파일 시스템 접근 (브라우저 보안 제약)
- 프로세스 실행
- MCP stdio 통신

→ React는 **프론트엔드 담당**. 백엔드는 별도로 필요합니다.

### PHP

**가능**
- 웹 대시보드 / 관리자 페이지
- **MCP 서버 (HTTP / SSE 전송 방식)**
- 에이전트 작업 큐 관리
- 세션 / 메모리 DB 저장 (MySQL)
- Claude API 호출 (단순 HTTP 요청)

**어려움**
- **Claude Code 플러그인 제작** — Node.js 생태계 전용이라 PHP로는 불가
- 실시간 스트리밍 (PHP의 약점)
- 벡터 검색 (직접 구현 필요, 또는 MySQL 9.0 벡터 지원 / Qdrant 등 외부 연동)

### 추천 조합

**시나리오 A — 웹 서비스**
```
React (프론트)
   ↓
PHP / Laravel (백엔드 API)
   ↓
Claude API 직접 호출
   ↓
MySQL (메모리 저장)
```
Ruflo의 **아이디어만 차용**하고 구현은 직접. 충분히 가능합니다.

**시나리오 B — Claude Code 확장**
```
Node.js / TypeScript 필수
```
PHP로는 불가능합니다.

**시나리오 C — 하이브리드 (추천)**
```
Node.js  → MCP 서버 (얇게, 100줄 수준)
PHP      → 실제 비즈니스 로직
React    → 대시보드
```
MCP 서버를 **PHP API를 호출하는 얇은 껍데기**로 만들면,
익숙한 PHP로 대부분을 구현하고 Node는 최소한만 사용할 수 있습니다.

---

## 11. 요약 및 권장 행동

| 질문 | 답 |
|---|---|
| 이게 뭐야? | Claude Code를 AI 팀으로 확장하는 하네스 |
| 플러그인/스킬/MCP? | 전부 다 — 원하는 형태만 골라 쓰면 됨 |
| API 키 필요? | 기본은 불필요 (Claude Code 로그인 재사용) |
| 왜 유명해? | 타이밍 + 컨셉 + 규모감 + MIT + 마케팅 |
| 로컬 에이전트에 도움? | 매우 그렇다 — 프롬프트/훅/아키텍처 참고처로 최상 |
| 수익화 가능? | MIT라 자유. 강의·에이전트 팩·세팅 대행이 현실적 |
| React/PHP 가능? | UI는 자유, MCP는 하이브리드 구성 권장 |

### 다음 단계 권장

1. `docs/ruflo-explained.md` 읽기 — 저장소에서 가장 정직하고 읽기 쉬운 문서
2. `.claude/agents/` 안의 에이전트 정의 몇 개 열어보기
3. 관심 있으면 플러그인 방식(방법 A)으로 가볍게 체험
4. 처음부터 `init` 풀 설치는 권장하지 않음 (파일이 많이 생성됨)

---

## 참고 링크

- 공식 저장소: https://github.com/ruvnet/ruflo
- 이슈: https://github.com/ruvnet/ruflo/issues
- npm (ruflo): https://www.npmjs.com/package/ruflo
- npm (claude-flow): https://www.npmjs.com/package/claude-flow
- 제작자: https://github.com/ruvnet
- 웹 UI 데모: https://flo.ruv.io/
- Goal Planner 데모: https://goal.ruv.io/
- 저장소 내 추천 문서: `docs/ruflo-explained.md` (14장 구성 가이드)
- 성능 실측 근거: `docs/reviews/intelligence-system-audit-2026-05-29.md`
