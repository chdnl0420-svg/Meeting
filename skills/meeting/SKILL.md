---
name: meeting
description: Use when the user invokes `/meeting <주제>` or says "회의 열어줘", "Claude와 Codex 토론시켜", "meeting on X" — runs an automated multi-agent meeting where a Facilitator subagent moderates two (extensible to N) debater subagents (Claude + Codex by default) on the user-given topic, produces hierarchical meeting minutes (main.md + sub-NN-*.md), and returns only a chat summary.
---

# Meeting Skill — 자동 다중 에이전트 회의

## Overview

사용자가 던진 큰 주제를 **진행자 + 토론자 N명**이 자동으로 회의해서 결론을 내고 회의록을 남깁니다.

- **메인 세션 = 라우터** (에이전트 풀 글로빙, 호출 분기, 회의록 누적, 종료 신호 수신)
- **진행자 = 고정 서브에이전트** (`meeting-facilitator`) — 풀에서 제외, 항상 진행자 역할
- **토론자 = `~/.claude/agents/meeting-*.md` 글로빙 후보 풀** (단, `meeting-facilitator.md` 및 frontmatter `role: facilitator` / `participant: false` 명시 파일은 제외)
- 각 에이전트 파일의 **frontmatter `backend` 필드로 호출 방식 분기**:
  - `backend: claude` → Task 도구로 서브에이전트 호출
  - `backend: codex` → 메인 라우터가 Bash 로 `codex exec` 직접 호출 (Claude Code auto-mode classifier 가 서브에이전트의 nested `codex exec` 차단하기 때문. 메인 라우터의 직접 Bash 호출은 `Bash(codex exec:*)` 권한으로 통과)
  - 향후 `gemini`, `ollama` 등 추가 가능
- **모든 에이전트는 페르소나만 다름**. 공통 응답 형식(`roles/common-debater.md`) 은 단일 출처.
- 사용자 개입 0회. 호출 한 번 → **모든 발언이 채팅에 실시간 출력** + 최종 요약.

## When to Use

- `/meeting <주제>` 슬래시 커맨드 호출
- 사용자: "Claude랑 Codex랑 회의 시켜", "두 모델 토론 붙여줘", "X 주제로 다중 에이전트 회의"
- 한국어로 토론·회의록 작성

## When NOT to Use

- 단일 에이전트로 충분한 질문 ("X 가 뭐야?")
- 사용자가 자기 의견을 듣고 싶을 때 (`/grill-me` 사용)
- 코드 리뷰 ("/code-review" 사용)

## Architecture

```
사용자 → /meeting <주제>
         ↓
    메인 라우터 (이 SKILL.md 가 동작 안내)
    │
    ├─ 회의 디렉터리 생성: ./meetings/{ts}-{slug}/
    ├─ main.md 초기화 + agenda.md 초기화 (어젠다 누적 리스트)
    │
    ├─ [어젠다 결정 메타 서브주제 = sub-01]
    │   ├─ 진행자 첫 호출 → 풀 추천 + BACKEND_SELECTION + SPEAKING_ORDER (⛔ INITIAL_AGENDA 등록 금지, 토론자 발언으로만)
    │   ├─ sub-01-어젠다결정.md 초기화
    │   ├─ 라운드로빈 호출 → 각 토론자 어젠다 항목 등록 (SPEAK) + NEST_REQUEST = 어젠다 항목 자동 추가
    │   ├─ 연속 PASS == 활성 수 도달 시 자동 종결
    │   ├─ 진행자 → 어젠다 리스트 확정 + agenda.md 갱신
    │   └─ 어젠다 본 토론 단계로 진입
    │
    ├─ [어젠다 본 토론 LOOP] (어젠다 대기 리스트가 비어있을 때까지)
    │   ├─ 진행자 호출 → AGENDA_PICKED + 풀 재평가 + BACKEND_SELECTION + IDEA_BANK_EVAL
    │   ├─ sub-NN-*.md 초기화
    │   │
    │   ├─ [라운드로빈 PASS-loop]
    │   │   ├─ 활성 토론자 순환 호출 (정해진 backend 로)
    │   │   │   → SPEAK / PASS / NEST_REQUEST (즉시 분기) / ADD_AGENDA (어젠다만 추가)
    │   │   ├─ 회의록 append + 라이브 출력
    │   │   ├─ NEST_REQUEST → 자식 서브주제 즉시 분기 (depth ≤ 5, nest_count ≤ 10)
    │   │   │   ├─ depth 5 진입 시 진행자가 "NEST 무효 한계" 사전 안내
    │   │   │   └─ 자식 종결 후 부모 사이클 재개 (cp 자식 진입 직전 값 유지)
    │   │   ├─ 무한 대화 가드 (speech_count >= 3 × active) → CEO_DECISION
    │   │   └─ 연속 PASS == 활성 수 도달 시 자동 종결
    │   │
    │   ├─ 진행자 → 서브주제 결론 작성 (FINAL_CONCLUSION)
    │   ├─ agenda.md 의 그 어젠다 항목 → ✅ 처리 완료 이동
    │   └─ 어젠다 대기 리스트 ≥ 1 → 다음 어젠다 (LOOP 계속), == 0 → 종합 단계
    │
    ├─ [모든 어젠다 완주] 라우터 자동 진입
    │   ├─ 진행자 → FINAL_SUMMARY 작성 (모든 어젠다 결론 종합)
    │   └─ main.md 종합 섹션 + 종료 시간 기록
    │
    └─ 최종 출력: 채팅에 종합 요약 + 어젠다 완주 리스트 + 회의록 경로

⛔ 회의 종료 투표·진행자 중단 제안 폐지 (2026-05-28). 어젠다 리스트 완주가 유일한 종결 조건.
⚡ 모든 에이전트 응답(진행자 결정, 토론자 발언, 패스)은 받는 즉시 채팅 라이브 스트리밍.
```

## 실시간 채팅 출력 정책 (Live Streaming)

**메인 라우터는 매 에이전트 호출 결과를 받는 즉시 채팅에 출력합니다.** 사용자가 회의 진행을 실시간 관전할 수 있어야 합니다. 파일 저장과 채팅 출력은 동시에 수행 (저장 = 영구 기록, 출력 = 라이브 관전).

### 출력 형식 표준

**① 진행자가 새 서브주제 결정 시:**
```markdown
---
### 🆕 서브주제 {N} 시작: {SUBTOPIC_TITLE}

🧑‍⚖️ **진행자**
- 이유: {SUBTOPIC_CONTEXT}
- 활성 토론자: [{ACTIVE_AGENTS}]
- 발언 순서: {SPEAKING_ORDER}
```

**①-b 토론자 풀 변경이 있을 때 (AGENTS_ADDED 또는 AGENTS_REMOVED 비어있지 않음):**

① 출력 직후 이어서 추가 출력 — 변경이 없으면 이 블록 생략.

```markdown
🧑‍⚖️ **진행자 — 토론자 풀 조정**:
- ➕ 추가: [{AGENTS_ADDED}]   ← 빈 배열이면 줄 생략
- ➖ 제거: [{AGENTS_REMOVED}] ← 빈 배열이면 줄 생략
- 사유: {AGENTS_CHANGE_REASON}
```

**② 라운드 마커 폐지** (2026-05-28). 연속 대화 시퀀스로 표시 — 사이클 구분 없음.

**③ 토론자가 발언 시 (SPEAK: YES):**
```markdown
**🤖 {agent}** _(i={i})_:
> {발언 내용}
```

**④ 토론자가 패스 시 (SPEAK: NO):**
```markdown
**🤖 {agent}** _(i={i})_: _(PASS [{cp}/{N}] — {reason})_
```
(cp = consecutive_pass 현재값, N = 활성 토론자 수. cp == N 도달 시 자동 종결)

**⑤ NEST 즉시 분기 시:**
```markdown
🔀 **NEST 즉시 분기**: "{TITLE}" — 요청자 {requester} 가 자식 서브주제 첫 발언자
(부모 사이클 일시 정지, 자식 종결 후 재개)
```

**⑥ 서브주제 자동 종결 시 (연속 PASS == 활성 수 도달):**
```markdown

### ✅ 서브주제 {N} 자동 종결: {title} (cp={N}/{N})
**결론** (진행자 작성): {conclusion (3-5줄)}
```

**⑦ 어젠다 추가 등록 시 (어젠다 메타에서 NEST_REQUEST 또는 본 토론에서 ADD_AGENDA):**
```markdown
🔀 NEST_REQUEST "{TITLE}" — 어젠다 항목 등록만 (어젠다 메타 모드, 즉시 분기 X)
📌 ADD_AGENDA "{TITLE}" — 어젠다 리스트 등록만 (본 토론 중 추가, 분기 X)
```

**⑧ CEO 호출 시 (4-4 / 4-5 트리거):**
```markdown
⚖️ **CEO 단독 결정 요청** — {트리거 사유}
선택지: A) {옵션A} / B) {옵션B} [/ C) {옵션C}]
**🤖 ceo**: CEO_DECISION = {A|B|C} — {REASON}
→ 라우터: {다음 단계 명시}
```

**⑨ 어젠다 리스트 완주 후 최종 종합** (아래 9단계 형식대로).

⛔ 폐지된 출력 형식: "회의 종료 제안 / END_AGREE 동의" — 7단계 폐지로 자동 소멸.

### 정책 원칙

- **풀 텍스트 출력**: 토론자 발언은 요약하지 말고 받은 그대로 출력 (5줄 이내라 부담 없음)
- **메인 라우터의 사고/계획 출력 금지**: 사용자에게 보이는 건 "회의 자체"만. 라우터의 내부 작업(파일 저장, 카운터 관리)은 채팅에 노출 안 함
- **실패도 출력**: codex CLI 실패, 형식 오류, CEO 트리거 — 모두 채팅에 표시 (`⚠️` 마커)
- **구분선 일관성**: 서브주제 사이는 `---`, 발언 사이는 빈 줄 (라운드 마커 없음)

## Workflow (메인 라우터가 따를 단계)

### 0. 권한·CLI 사전 검사 (자동 fallback, 사용자 질문 금지)

⛔ **AskUserQuestion 절대 금지** ([[meeting-no-user-questions]] 강력 조항). codex 사용 가능 여부에 따라 라우터가 자동 결정하고 채팅에 1회 통보만 한다.

#### 0-1. settings.json 권한 검사

```bash
grep -E '"Bash\(codex exec' ~/.claude/settings.json || echo "MISSING"
```

| 결과 | 처리 (자동) |
|------|------------|
| 권한 있음 | 출력 없이 0-2 진행 |
| `MISSING` | 라우터가 `codex_available = false` 로 설정. 진행자에게 회의 시작 시 "codex 백엔드 사용 불가" 자동 통보 → 진행자가 BACKEND_SELECTION 에서 모든 에이전트를 claude 로 강제. 채팅에 1회 출력: `⚠ codex 권한 부재 — claude 단독 진행. 활성화하려면 settings.json 에 "Bash(codex exec:*)" 추가 후 다음 회의에서 자동 인식` |

#### 0-2. codex CLI 사전 점검

```bash
codex --version
```

| 결과 | 처리 (자동) |
|------|------------|
| exit 0 | 출력 없이 1단계 진행 |
| exit ≠ 0 (CLI 부재·로그인 실패) | `codex_available = false` 설정. 채팅에 1회 출력: `⚠ codex CLI 사용 불가 (exit={code}) — claude 단독 진행`. 회의 계속 |

권한·CLI 둘 다 통과 = `codex_available = true`. 진행자가 BACKEND_SELECTION 에서 자유롭게 codex 선택 가능.
권한·CLI 중 하나 실패 = `codex_available = false`. 진행자에게 회의 시작 컨텍스트에 "codex 사용 불가, BACKEND_SELECTION 은 claude 만 사용" 명시. **회의 중단 안 함**.

### 1. 회의 디렉터리 생성

```bash
TS=$(date +%Y%m%d-%H%M%S)
SLUG=<주제를 20자 이내 한국어 슬러그로 변환>
DIR="./meetings/${TS}-${SLUG}"
mkdir -p "$DIR"
```

`main.md` 를 `templates/main.md` 템플릿으로 초기화 (주제·시작시간·참여자 채움).

### 2. 토론자 후보 풀 글로빙 + 메타 추출

`~/.claude/agents/meeting-*.md` 글로빙. 다음 파일은 **제외**:
- `meeting-facilitator.md` (진행자, 항상 제외)
- frontmatter 의 `role: facilitator` 또는 `participant: false` (옵저버·기록자 등 비-토론자)
- 이름이 `meeting-debater-` 로 시작하지 않으면서 위 조건도 안 맞는 경우는 그대로 후보에 포함 (사용자 정의 페르소나 허용)

각 후보 파일에서 frontmatter 추출 → 메타 테이블 생성:

```
| name                       | backend | model  | persona (요약)             | expertise            |
|----------------------------|---------|--------|----------------------------|----------------------|
| meeting-debater-claude     | claude  | sonnet | 균형·원칙·거버넌스          | 원칙, 분리선, 거버넌스 |
| meeting-debater-codex      | codex   | gpt-5  | 시스템 설계·메커니즘        | 시장 메커니즘, 지표   |
| meeting-debater-game-econ  | claude  | opus   | 게임 경제 디자이너 페르소나 | 게임 경제, MMO 사례   |
```

**호출 방식 분기**는 frontmatter `backend` 필드로 결정 (4-1 참조).

#### 2-0. CEO 강제 참관 (절대 원칙, 모든 회의 공통)

`meeting-ceo` 는 **모든 회의에 자동 강제 참관**. 단, CEO 는 회의 본 진행(라운드 발언·투표·동의)에는 **일체 호출되지 않는다**. CEO 의 유일한 역할은 **(b) 단독 결정자**:

- **(b) 단독 결정자만**: 일반 투표·동의가 동률 또는 루프에 빠지면 진행자가 CEO 에게 단독 결정 요청 (CEO_DECISION 모드, 디테일은 7-A 섹션)

회의 본 진행에서 CEO 가 빠지는 항목:
- 4-1 라운드 발언 (SPEAK_OR_PASS) — **호출 안 함** (CEO 는 도메인 의견 안 만드므로 자동 패스. 호출 자체가 토큰 낭비)
- 4-3-b 라운드 종결 투표 — 호출 안 함
- 4-4-1 서브주제 종결 동의 — 호출 안 함
- 7 회의 종료 동의 — 호출 안 함
- NEST_REQUEST 투표 — 호출 안 함

CEO 풀 관리:
- CEO 는 SPEAKING_ORDER · ACTIVE_AGENTS 에 **포함하지 않는다** (활성 토론자 풀과 분리된 "결정자" 슬롯)
- 진행자 추천 (2-2) / 사용자 명시 (2-1) 의 RECOMMENDED_AGENTS · `--agents` 결과와 무관
- 6단계 풀 재평가에서 ADD/REMOVE 무관 — CEO 는 항상 결정자 슬롯에 상주
- 후보 풀 글로빙(2단계) 에 `meeting-ceo.md` 가 부재하면 회의 시작 불가. 채팅에 1회 자동 출력 (`⚠ meeting-ceo.md 부재 — CEO 없는 회의 진행 불가, 회의 종료. 별도 세션에서 /meeting-agent 로 ceo 페르소나 생성 후 재시도`) 후 자동 중단. **사용자 질문 금지**
- `meeting-ceo.md` 의 `backend` 필드는 반드시 `claude` (CEO_DECISION 호출 일관성). `backend: codex` 인 CEO 발견 시 회의 시작 전 거부 + 사용자에게 backend 변경 요청

CEO 호출은 **오직 7-A CEO_DECISION 모드** 만. 라우터가 동률·루프 트리거 발생 시 단독 호출.

#### 2-0-2. Backend 진행자 결정 (절대 원칙, 모든 회의 공통)

**모든 에이전트의 backend (claude / codex) 는 진행자가 서브주제 시작 시 결정한다.** 매 호출마다 재결정 X — 한 서브주제 안에서 그 에이전트는 결정된 backend 로 일관 호출.

기본 규칙:
- 에이전트 frontmatter 의 `backend` 필드는 **default 추천값**. 진행자가 오버라이드 가능.
- 진행자 응답 (A-1·A-2 모드) 에 `BACKEND_SELECTION` 필드 필수 포함 — 각 활성 에이전트의 backend 명시.
- 한 서브주제 안 같은 에이전트는 항상 같은 backend 로 호출됨. 다음 서브주제에서는 진행자가 재결정 가능 (같은 에이전트의 backend 변경 가능).
- 동일 페르소나라도 backend 가 다르면 응답 결이 다름 (codex = 시스템 메커니즘·정량·엣지 케이스 / claude = 균형·원칙·인문학적 분석). 진행자가 서브주제 요구에 맞게 결정.

진행자 backend 선택 기준 (참고):
- **codex 권장**: 시스템 설계 무결성·트랜잭션·멱등성·엣지 케이스·통계 검정·서버 아키텍처가 핵심인 주제
- **claude 권장**: 균형·원칙·합의·인문학적 통찰·정책 결정·UX 체감·법규 해석이 핵심인 주제
- **혼합 권장**: 한 서브주제에서 다른 에이전트끼리 다른 backend 섞기 (양 시각 확보)

⛔ 폐기 (2026-05-28): "Codex 백엔드는 8-A 최종 의견 단계 전용" 정책 폐기. 회의 본 진행에서 codex 호출 가능. 이전 메모리 [[meeting-codex-final-opinion-only]] DEPRECATED.

CEO 예외:
- `meeting-ceo` 는 backend 강제 = claude (CEO_DECISION 호출 일관성)
- 진행자가 CEO 의 backend 를 codex 로 오버라이드 시도하면 라우터가 거부 + claude 강제 적용

#### 2-1. 사용자가 /meeting 인자로 토론자 명시 지정한 경우

`/meeting <주제> --agents claude,codex,game-econ` 형태로 명시되면 진행자 추천 건너뛰고 그 풀 사용 (이름은 `meeting-debater-` 접두 생략 허용 — 메인 라우터가 정규화). 다만:
- 2-0 CEO 강제 참관 자동 적용 (CEO 는 결정자 슬롯, 일반 풀과 무관)
- 2-0-2 적용: `--agents` 에서 backend=codex 이름은 회의 본 진행 풀에서 제거되고 최종 의견 풀로 자동 이동

#### 2-2. 명시 지정 없으면 진행자 추천 흐름 (3단계에서)

진행자에게 후보 메타 테이블 전달 → 진행자가 회의 주제 분석 후 `RECOMMENDED_AGENTS: [name1, name2, ...]` 응답. 보통 2~4명. 메인 라우터가 그 추천을 토론자 풀로 확정. **CEO 누락 시 라우터가 자동 추가 (2-0).**

#### 2-3. 후보가 1명 이하면

- 0명: 회의 진행 불가. 채팅에 1회 자동 출력 (`⚠ 토론자 후보 0명 — 회의 진행 불가, 자동 중단. 별도 세션에서 /meeting-agent 로 토론자 페르소나 생성 후 재시도`) 후 자동 중단. **사용자 질문 금지**
- 1명: 자기 반박/다관점 모드로 진행자가 토론 패턴 변경 (한 토론자가 매 라운드 자기 비판) 또는 사용자 확인 후 진행.

#### 2-4. 선택된 풀 보관 → `participants.md`

회의 디렉터리에 `participants.md` 작성:

```markdown
# 회의 참여 에이전트

- meeting-facilitator (진행자)
- meeting-debater-claude (backend=claude, sonnet) — 균형·원칙·거버넌스
- meeting-debater-codex (backend=codex, gpt-5) — 시스템 설계·메커니즘
- ...

선정 사유: <진행자 RECOMMENDED_AGENTS 응답의 추천 사유 또는 "사용자 명시 지정">
```

다른 작업자(진행자 호출, 토론자 호출, 회의록 작성)는 이 파일을 참조한다.

### 3. 토론자 풀 추천 + 첫 서브주제 결정 — 진행자 호출

`meeting-facilitator` 서브에이전트를 Task 도구로 호출. 사용자 명시 지정이 없는 경우 풀 추천도 같이 받는다.

입력 (사용자 명시 지정 없을 때):

```
=== 큰 주제 ===
{user_topic}

=== 토론자 후보 풀 (글로빙 결과) ===
| name | backend | model | persona | expertise |
|------|---------|-------|---------|-----------|
| meeting-debater-claude | claude | sonnet | 균형·원칙·거버넌스 | 원칙, 거버넌스 |
| meeting-debater-codex | codex | gpt-5 | 시스템 설계·메커니즘 | 시장 메커니즘, 지표 |
| ... |

=== 진행된 서브주제 (지금까지) ===
(없음)

=== 당신의 임무 ===
1. 큰 주제 분석.
2. **6대 도메인 커버리지 점검** (게임 프로젝트 기준): 콘텐츠·내러티브 / 기술 / 아트·비주얼 / 경제·재화 / 수익화·BM / 운영·일정·검증. 각 도메인 최소 1명씩 풀에 포함.
3. 후보 풀에서 이번 회의에 가장 적합한 토론자 추천 (RECOMMENDED_AGENTS). 6 도메인 커버 후 추가 페르소나는 큰 주제 가중치에 따라 선택 — 일반적으로 6~10명.
4. 첫 서브주제(어젠다 결정)와 발언 순서 결정.

⚠️ **사용자 키워드 lock 금지** (memory: meeting-first-subtopic-domain-coverage):
- 사용자가 "비-P2W·인플레만" 명시해도 6 도메인 모두 포함
- 사용자가 "스토리만" 명시해도 6 도메인 모두 포함
- 키워드 가중치는 SPEAKING_ORDER 우선순위·토론 깊이에 반영, 풀 자체는 누락 금지
- 게임 프로젝트가 아니면 (백엔드·SaaS·소설 등) 도메인 정의는 다르되 "핵심 시각 ≥5개" 원칙 유지

응답 형식 (반드시):

DECISION: START_SUBTOPIC
DOMAIN_COVERAGE_CHECK:
1. 콘텐츠·내러티브: ✓ {agent}
2. 기술: ✓ {agent}
3. 아트·비주얼: ✓ {agent}
4. 경제·재화: ✓ {agent}
5. 수익화·BM: ✓ {agent}
6. 운영·일정: ✓ {agent}
RECOMMENDED_AGENTS: [name1, name2, ..., nameN]
RECOMMENDED_REASON: <왜 이 조합을 골랐는지, 1-2줄. 6 도메인 커버 + 큰 주제 가중치 명시>
BACKEND_SELECTION:
- name1: claude
- name2: codex
- name3: claude
- ...                            (RECOMMENDED_AGENTS 전원 명시, meeting-ceo 제외)
BACKEND_SELECTION_REASON: <왜 이 backend 조합, 1-2줄. 시스템 메커니즘 vs 균형/원칙 시각 분배 사유>
IDEA_BANK_EVAL: included | excluded
IDEA_BANK_REASON: <포함/제외 사유 1-2줄. 평가 기준(신호 5개 포함 / 3개 제외) 인용>
INITIAL_AGENDA: []   ⛔ **강력 지침** (2026-05-28 3차 [[meeting-agenda-debaters-only]]): 진행자는 어젠다 항목을 임의 등록할 수 없음. 반드시 `[]` 빈 배열만 허용. 어젠다는 전적으로 토론자 발언(SPEAK/NEST_REQUEST/ADD_AGENDA)으로만 등록. 진행자가 항목을 넣어도 라우터가 무시 + 채팅 경고 출력.
SUBTOPIC_TITLE: 회의 어젠다 결정
SUBTOPIC_CONTEXT: <왜 이 서브주제부터, 1-2줄>
SPEAKING_ORDER: [name1, name2, ...] (RECOMMENDED_AGENTS 순열, 어젠다 결정 메타는 각자 자기 도메인 어젠다 1개 등록 후 PASS 권장)
```

⛔ 폐지된 제약: "SPEAKING_ORDER 는 backend=claude 만". 진행자가 BACKEND_SELECTION 으로 결정하므로 codex 토론자도 SPEAKING_ORDER 에 포함 가능 (회의 본 진행).

> **IDEA_BANK_EVAL 필수 (2-6-A)**: 진행자는 평가 결과에 따라 RECOMMENDED_AGENTS 에 `meeting-idea-bank` 포함 여부 결정. 누락 시 라우터가 자동 `included` 강제.

입력 (사용자 명시 지정 있을 때): 위 동일하되 `=== 토론자 풀 (사용자 명시) ===` 섹션으로 변경, `RECOMMENDED_AGENTS` 응답은 생략 가능 (있어도 무시).

진행자 응답 파싱:
- `RECOMMENDED_AGENTS` 로 풀 확정 → `participants.md` 작성 (2-4)
- 서브주제 회의록 파일 생성 (`sub-01-{slug}.md`)

### 4. 라운드로빈 연속 대화 루프 (서브주제 내부)

**라운드 개념 폐지** (2026-05-28 [[meeting-no-rounds-passall]]). 활성 토론자 순환 호출 + 연속 PASS == 활성 수 시 자동 종결.

#### 4-1. 루프 변수 초기화

```
active_debaters  = SPEAKING_ORDER (CEO 제외, codex 백엔드 제외)
consecutive_pass = 0
i                = 0
speech_count     = 0
nest_count       = 0   # 회의 전체 누적 NEST 수 (서브주제 간 공유)
```

#### 4-2. 순환 호출 (PASS-loop)

```
WHILE consecutive_pass < len(active_debaters):
  agent = active_debaters[i % len(active_debaters)]
  response = call(agent, mode=SPEAK_OR_PASS, context=4-2-a)
  
  IF response.SPEAK == YES:
    consecutive_pass = 0
    append response.CONTENT to subtopic_md
    chat output: 🤖 {agent}: > {content}
    
    IF response.NEST_REQUEST:
      → 4-3 즉시 분기 처리
  ELSE:  # SPEAK: NO (PASS)
    consecutive_pass += 1
    chat output: 🤖 {agent}: _(PASS — {reason})_
  
  speech_count += 1   # SPEAK·PASS 무관 호출당 +1
  i += 1
  
  IF speech_count >= 3 * len(active_debaters):
    → 4-4 무한 대화 가드 (CEO_DECISION 트리거)
  
END LOOP (consecutive_pass == len → 자동 종결)
→ 5단계 (서브주제 종결 처리)
```

#### 4-2-a. 공통 컨텍스트 패키지

호출 방식은 그 에이전트 frontmatter 의 `backend` 필드로 결정 — 자세한 절차는 아래 "토론자 호출 디테일 (backend 분기)" 섹션 참조.

| backend | 호출 방식 |
|---------|-----------|
| `claude` | Task 도구로 해당 서브에이전트 호출 (subagent_type=에이전트 name) |
| `codex` | **회의 본 진행 호출 금지 (2-0-2)**. 8-A 최종 의견 단계 전용 |
| `gemini` | (향후) 메인 라우터가 Bash 로 `gemini` CLI 직접 호출 |
| `ollama` | (향후) 메인 라우터가 Bash 로 `ollama run` 직접 호출 |

> CEO (`meeting-ceo`) 는 호출에서 자동 제외 (2-0). SPEAKING_ORDER 에 포함하지 않으며, 라우터가 호출 대상 리스트에서 자동 제거.

컨텍스트 패키지:

```
=== 큰 주제 ===
{topic}

=== 진행된 서브주제 결론 ===
1. {sub1_title} → {sub1_conclusion}
2. ...
(없으면 "없음")

=== 현재 서브주제: {current_title} ===
{current_subtopic_md 전체 — 누적된 모든 발언 포함}

=== 당신의 차례 ===
당신: {agent_name}
직전 발언자: {prev_speaker} (첫 발언이면 "없음")
사이클 위치: i={i}, consecutive_pass={consecutive_pass}/{len(active)}

== PASS 의미 ==
- PASS = "현재까지의 누적 결론 수용 + 종결 의사 표명"
- 연속 PASS 수 == 활성 토론자 수 시 자동 종결 (별도 투표 없음)
- 같은 논점 반복할 거면 PASS
- 새 근거·반박·관점·구체 제안 있을 때만 SPEAK
- 모호하면 PASS (안전 기본값)

== NEST_REQUEST ==
- 자식 서브주제 분기가 필요하면 SPEAK 응답에 NEST_REQUEST 블록 추가
- 라우터가 즉시 자식 서브주제 개시 (투표 없음). 본인이 첫 발언자가 됨
- 부모 사이클은 자식 종결 후 재개됨

응답 형식 (common-debater.md 의 모드 1 SPEAK_OR_PASS 참조)
```

#### 4-2-b. 회의록 누적

각 응답을 서브주제 회의록에 append (라운드 헤더 없음, 평탄한 시퀀스):

```markdown
## 대화 시퀀스

**{agent}** (i={i}): {SPEAK 시 발언 본문 / PASS 시 "(패스: {reason})"}
```

#### 4-3. NEST_REQUEST 분기 정책 — 어젠다 메타 vs 일반 서브주제 분기점

토론자 응답에 `NEST_REQUEST` 가 포함되면 **현재 서브주제가 어젠다 결정 메타 단계인지 일반 본 토론 서브주제인지에 따라** 처리 분기:

##### 4-3-A. 어젠다 결정 메타 서브주제 (sub-01) — NEST = 어젠다 항목 자동 등록

⚠ **즉시 분기 금지** (2026-05-28). 어젠다 결정 메타에서 NEST_REQUEST 는 **어젠다 누적 리스트에 항목 추가 요청** 으로 해석한다:

```
IF current_subtopic == 어젠다 결정 메타 (sub-01):
  IF response.NEST_REQUEST:
    # 어젠다 리스트에 항목 추가
    agenda_list.append({
      title: response.NEST_REQUEST.TITLE,
      context: response.NEST_REQUEST.REASON,
      proposer: requester_agent,
      registered_at: "어젠다 메타 (NEST 변환)"
    })
    # 어젠다 메타는 계속 진행 (다음 i)
    # 자식 분기 안 함
    chat output: 🔀 NEST_REQUEST "{TITLE}" — 어젠다 항목 등록만 (어젠다 메타 모드)
```

이 정책은 advisor 권고·실제 사례 (어젠다 메타 NEST 즉시 분기로 어젠다 정렬 무산) 의 결과. NEST 가 어젠다 메타에서 호출돼도 본 토론은 후속 서브주제에서 진행.

##### 4-3-B. 일반 본 토론 서브주제 (sub-02 이후) — NEST = 1 라운드 가드 + 즉시 분기 (2026-05-28 5차)

일반 본 토론에서 NEST_REQUEST 는 다음 가드를 모두 통과해야 즉시 분기. 미통과 시 자동 ADD_AGENDA 변환:

**1 라운드 가드 (2026-05-28 5차 신규)**: 부모 사이클에서 활성 토론자 N명이 **각자 1회 이상 발언 또는 PASS** 한 후에만 NEST_REQUEST 즉시 분기 허용. 그 전(speech_count < N) 의 NEST_REQUEST 는 자동 ADD_AGENDA 로 변환 — 어젠다 누적 리스트에 항목 추가만 되고 자식 분기 안 함. 부모 사이클 계속.

**목적**: 첫 발언자가 곧바로 NEST 로 자식 깊이 들어가서 다른 토론자 의견을 묻어버리는 패턴 차단. 부모 서브주제에서 1 라운드 의견 수렴 후 NEST 결정.

**예외**: 부모 서브주제 풀 = 1명 (자기반박 모드) 인 경우 1 라운드 가드 무효 (의미 없음). 활성 풀 2명 이상에서만 적용.

**depth 5 보류 정책 (2026-05-28 6차 신규, 7차 cap 3→5 확장)**: depth 5 NEST 시도는 분기 무효 + 어젠다 추가 안 함이지만, **보류 리스트(`deferred_nests`)** 에 항목 등록 + 고손자(현재) 종결 후 상위(증손/손자/자식/루트) 사이클 재개 시 토론자 컨텍스트에 **"보류된 NEST 주제 — 다시 신청 권장"** 안내 추가. 사용자 정책 ([[meeting-nest-depth3-deferred]]): 아예 논의 안 되는 건 금지, 자연스럽게 적절한 depth 에서 재신청 유도.

```
IF current_subtopic == 일반 본 토론 서브주제 (sub-02 이후):
  IF response.NEST_REQUEST:
    IF current_depth >= 5:
      # depth 5 보류 (6차 신규, 7차 확장 5→5) — 무효 + 보류 리스트 등록
      deferred_nests.append({
        title: response.NEST_REQUEST.TITLE,
        reason: response.NEST_REQUEST.REASON,
        requester: requester_agent,
        deferred_at_sub: current_sub_id,
        deferred_at_depth: 5
      })
      append to subtopic_md:
        ## NEST 보류 (i={i}, depth=5 한계)
        - 요청자: {requester_agent}
        - 보류된 주제: "{TITLE}"
        - 사유: {REASON}
        - 안내: 이번 고손자 종결 후 상위(증손/손자/자식/루트) 사이클에서 다시 NEST_REQUEST 신청 가능
      chat output: ⚠️ NEST_REQUEST "{TITLE}" 보류 — depth 5 한계. 상위 사이클 재개 시 다시 신청 가능
      # 부모 사이클 계속 (다음 i)
    
    ELIF nest_count >= 10:
      → 4-4 NEST cap overflow 가드 (CEO_DECISION 트리거)
    
    ELIF speech_count < len(active_debaters) AND len(active_debaters) >= 2:
      # 1 라운드 가드 (5차 신규) — ADD_AGENDA 변환
      agenda_list.append({
        title: response.NEST_REQUEST.TITLE,
        context: response.NEST_REQUEST.REASON,
        proposer: requester_agent,
        registered_at: f"{current_sub_id} 본 토론 중 1라운드 미완 NEST→ADD_AGENDA"
      })
      append to parent_md:
        ## NEST→ADD_AGENDA 변환 (i={i}, speech_count={N}/{len(active)})
        - 요청자: {requester_agent}
        - 변환 사유: 1 라운드 가드 (다른 토론자 의견 수렴 전 분기 차단)
        - 어젠다 리스트 추가: "{TITLE}"
      chat output: ⚠️ NEST→ADD_AGENDA "{TITLE}" — 1 라운드 가드(speech_count={N}/{len(active)}), 어젠다 리스트 등록만, 분기 X
      # 부모 사이클 계속 (다음 i)
    
    ELSE:
      nest_count += 1
      
      # 자식 서브주제 즉시 개시 (진행자 6단계 호출로 풀·backend 결정)
      child_md = create_child_subtopic(
        title           = response.NEST_REQUEST.TITLE,
        context         = response.NEST_REQUEST.REASON,
        active_debaters = parent_active_debaters,  # 부모 풀 상속 (진행자 조정 가능)
        backend_map     = parent_backend_map,      # 부모 backend 상속 (진행자 조정 가능)
        speaking_order  = [requester_agent] + parent_active_debaters - [requester_agent]
      )
      
      append to parent_md:
        ## NEST 분기 (i={i}, depth={parent_depth}→{child_depth})
        - 요청자: {requester_agent}
        - 자식 서브주제: [./sub-{child_path}.md](./sub-{child_path}.md)
        - 분기 사유: {NEST_REQUEST.REASON}
      
      # 자식 사이클 진입 (재귀, 진행자 6단계 호출로 BACKEND_SELECTION 재결정)
      recurse 4-2 with child_md (depth+1)
      
      # 자식 종결 후 부모 md 에 자식 결론 append
      append to parent_md:
        ## 자식 서브주제 결론 ({child_path})
        {child_conclusion}
      
      # 부모 사이클 재개 (consecutive_pass 자식 진입 직전 값 유지)
```

**하위 트리 종결 시 보류 NEST 재신청 안내 (6차 신규, 7차 depth 5 확장)**: 자식·손자·증손·고손 등 모든 하위 서브주제 종결 후 부모 사이클이 재개될 때, 라우터가 그 하위 트리에서 발생한 `deferred_nests` 항목들을 부모 컨텍스트에 안내. 부모 사이클의 다음 호출되는 토론자(들) 컨텍스트 패키지에 다음 섹션 추가:

```
=== 보류된 NEST 주제 (재신청 권장) ===
종결된 하위 트리에서 depth 5 한계로 보류된 NEST 주제 N개:
1. "{title}" — 요청자: {requester}, 사유: {reason} (보류 위치: {deferred_at_sub})
2. ...

당신이 이 주제 중 하나(또는 새 주제) 를 본 사이클에서 다시 NEST_REQUEST 로 신청하면 정상 분기 가능 (현재 depth={parent_depth}, 자식은 depth={parent_depth+1} 가 되어 추가 분기 여유 확보).
강제 사항 아님 — 의사 있는 토론자만 발의. 신청 안 되면 보류 항목은 그대로 회의록 raw 기록만 남고 어젠다 리스트에는 추가 안 됨.
```

이렇게 보류 항목은 사라지지 않고 토론자에게 노출되어 자연스럽게 적절한 depth 에서 재신청될 수 있다. 단, 어젠다 리스트 자동 추가는 안 함 — 사용자 정책 ([[meeting-nest-depth3-deferred]]): "흐름은 자식 주제에서 자연스럽게."

##### 4-3-C. 본 토론 중 어젠다 추가 (ADD_AGENDA 모드)

본 토론 서브주제에서 토론자가 발언에 `ADD_AGENDA: <항목>` 명시 시 라우터가 어젠다 리스트에 추가만 수행 (분기 안 함). NEST_REQUEST 와 다름 — ADD_AGENDA 는 분기 의도 없음 신호:

```
IF response.ADD_AGENDA:
  agenda_list.append({
    title: response.ADD_AGENDA,
    proposer: requester_agent,
    registered_at: "{current_sub_id} 본 토론 중 추가"
  })
  chat output: 📌 ADD_AGENDA "{title}" — 어젠다 리스트 등록만, 분기 안 함
  # 현재 사이클 계속
```

> 단일 응답에 NEST_REQUEST + ADD_AGENDA 가 둘 이상이면 모드별 첫 번째만 처리, 나머지 무시 + 회의록에 raw 기록.

#### 4-4. 무한 대화 가드 (CEO_DECISION 트리거)

`speech_count >= 3 × len(active_debaters)` 도달 시 CEO_DECISION 호출 (7-A):

- 트리거 사유: "한 서브주제 발언 수 > 3 × 활성 (PASS 자연 수렴 실패)"
- CEO 선택지:
  - A) 즉시 종결 (현재까지 발언으로 결론 작성)
  - B) 강제 PASS 사이클 1회 (모든 토론자 강제 PASS 호출 → 결론으로)
  - C) 자식 서브주제 분기 (CEO 가 핵심 쟁점 명시 → 자식 서브주제 개시)

CEO 결정 결과 그대로 채택 후 5단계 또는 자식 분기 진행.

#### 4-5. NEST cap overflow 가드 (CEO_DECISION 트리거)

`nest_count >= 10` 인데 새 NEST_REQUEST 도달 시 CEO_DECISION 호출 (7-A):

- 트리거 사유: "회의 누적 NEST 수 11 (cap 10 초과)"
- CEO 선택지:
  - A) 분기 강행 (cap +1, 이번만 허용)
  - B) 흡수 (NEST_REQUEST 기각, 현재 서브주제에 노트로만 남김)
  - C) 회의 종결 강제 (남은 NEST 모두 거부 + PROPOSE_END_MEETING)

### 5. 서브주제 종결 처리

`END_SUBTOPIC` 받으면:

1. 서브주제 회의록에 `## 결론 (진행자)` 섹션 append
2. `main.md` 의 `## 서브주제 결론` 에 항목 추가:

```markdown
### {N}. {sub_title}
- 결론: {conclusion 요약 한 줄}
- 상세: [./sub-NN-*.md](./sub-NN-*.md)
```

### 6. 다음 서브주제 결정 + 토론자 풀 재평가 + Backend 결정 — 진행자 호출

매 새 서브주제마다 **진행자에게 후보 풀 전체 + 현재 활성 풀 + 어젠다 누적 리스트** 를 함께 전달해 토론자 추가/제거·backend 결정을 판단하게 한다.

⛔ **PROPOSE_END_MEETING 모드 폐지** (7단계 폐지로). 진행자는 회의 종결 권한 없음 — 어젠다 리스트가 0개 됐을 때 라우터가 자동 8단계 진입.

```
=== 큰 주제 ===
{topic}

=== 토론자 후보 풀 (전체) ===
| name | default_backend | persona | expertise |
| meeting-debater-claude | claude | 균형·원칙·거버넌스 | 거버넌스 |
| meeting-debater-codex | codex | 시스템 설계 | 메커니즘 |
| meeting-debater-game-econ | claude | 게임 경제 디자이너 | 게임 경제, MMO |
| ... (참여자 미선택 후보 포함 전체) |

(default_backend 는 에이전트 frontmatter 의 추천값. 진행자가 BACKEND_SELECTION 에서 오버라이드 가능.)

=== 현재 활성 토론자 + backend ===
- monetization (claude)
- economy-designer (codex)
- ...

=== 어젠다 누적 리스트 ===
처리 대기: ① 코어 루프 ② 2D 쿼터뷰 아트 ③ DoD ...
처리 완료: (1) 인플레/Sink (sub-02), (2) 결제 퍼널 (sub-03)

=== 진행된 서브주제 결론 ===
1. {sub1_title} → {sub1_conclusion}
2. ...

=== 당신의 임무 ===
1. 어젠다 대기 리스트에서 **다음 처리할 어젠다 항목 1개** 선택 (우선순위 판단)
2. 그 어젠다를 본 토론할 서브주제 정의 (제목·컨텍스트)
3. 현재 활성 토론자 중 부적합·중복 → AGENTS_REMOVED. 후보 풀에서 더 적합 → AGENTS_ADDED
4. 각 활성 에이전트의 **BACKEND_SELECTION** 결정 (claude / codex). 기준은 SKILL 2-0-2 참조
5. ACTIVE_AGENTS + SPEAKING_ORDER + BACKEND_SELECTION + IDEA_BANK_EVAL 확정

응답 형식 (모드 START_SUBTOPIC 만):

DECISION: START_SUBTOPIC
AGENDA_PICKED: <어젠다 항목 번호·제목> (어젠다 대기 리스트에서 선택한 항목 명시)
SUBTOPIC_TITLE: <제목>
SUBTOPIC_CONTEXT: <이유>
ACTIVE_AGENTS: [name1, name2, ...]
AGENTS_ADDED: [...]                (없으면 [])
AGENTS_REMOVED: [...]              (없으면 [])
AGENTS_CHANGE_REASON: <왜 추가/제거했는지. 변경 없으면 "변경 없음">
BACKEND_SELECTION:
- name1: claude
- name2: codex
- name3: claude
- ...                              (ACTIVE_AGENTS 전원 — meeting-ceo 제외)
BACKEND_SELECTION_REASON: <왜 이 조합인지 1-2줄. 시스템 메커니즘/인문학 시각 분배 사유>
IDEA_BANK_EVAL: included | excluded
IDEA_BANK_REASON: <포함/제외 사유 1-2줄. 이 서브주제 기준 평가>
SPEAKING_ORDER: [...]              (ACTIVE_AGENTS 순열)
```

> **IDEA_BANK_EVAL 필수**: 매 서브주제마다 재평가. 누락 시 라우터가 자동 `included` 강제.
> **BACKEND_SELECTION 필수**: 매 서브주제마다 모든 ACTIVE_AGENTS 의 backend 명시. 누락된 에이전트가 있으면 라우터가 그 에이전트의 frontmatter default_backend 로 자동 채움 + 경고.
> **AGENDA_PICKED 필수**: 진행자는 어젠다 대기 리스트에서 명시적으로 1개 선택. 빈 항목 픽이거나 대기 리스트에 없는 항목 시 1회 재시도, 다시 실패 시 라우터가 대기 리스트 첫 항목 자동 채택.

#### 6-1. 응답 검증 (라우터)

진행자 응답 파싱 후 다음 가드를 통과해야 5단계 다음 라운드로 진입:

| 조건 | 처리 |
|------|------|
| `ACTIVE_AGENTS` 비어있음 | 1회 재시도. 다시 실패 시 라우터가 후보 풀에서 어젠다 항목과 가장 매칭 잘 되는 5명 자동 선정 (회의 중단 권한 없음) |
| `ACTIVE_AGENTS` ≠ (현재 활성 ∪ ADDED) - REMOVED | 1회 재시도 |
| `AGENTS_ADDED` 항목이 후보 풀에 없는 이름 | 그 이름만 무시하고 진행 |
| `AGENTS_REMOVED` 항목이 현재 활성 풀에 없는 이름 | 그 이름만 무시하고 진행 |
| `SPEAKING_ORDER` 집합 ≠ `ACTIVE_AGENTS` | 라우터가 `ACTIVE_AGENTS` 순서로 자동 보정 |
| `ACTIVE_AGENTS` 가 1명 이하 | 1회 재시도. 다시 실패 시 진행자에게 "최소 2명 유지" 강제 후 재시도. 그래도 1명이면 그대로 진행하되 자기반박 모드 안내 |
| `meeting-ceo` 가 `ACTIVE_AGENTS` 또는 `SPEAKING_ORDER` 에 잘못 포함됨 | 라우터가 그 항목 자동 제거 (CEO 는 SPEAK_OR_PASS 호출 대상 아님. 결정자 슬롯에 항상 상주, 2-0) |
| `meeting-ceo` 가 `AGENTS_REMOVED` 에 포함됨 | 라우터가 그 항목 무시 (CEO 는 활성 풀과 분리된 결정자 슬롯, 2-0) |
| `meeting-ceo` 가 `BACKEND_SELECTION` 에서 codex 로 명시됨 | 라우터가 claude 로 강제 오버라이드 (CEO backend 강제, 2-0-2) |
| `BACKEND_SELECTION` 항목 누락 (ACTIVE_AGENTS 의 일부 에이전트 미명시) | 라우터가 누락 에이전트의 frontmatter `default_backend` 로 자동 채움 + 채팅 경고 |
| `BACKEND_SELECTION` 항목이 ACTIVE_AGENTS 가 아닌 이름 포함 | 그 항목 무시 (오타·잘못된 참조) |
| `BACKEND_SELECTION` 값이 claude/codex 외 | 그 항목을 default_backend 로 보정 |
| `AGENDA_PICKED` 누락 또는 대기 리스트에 없는 항목 | 1회 재시도. 다시 실패 시 라우터가 대기 리스트 첫 항목 자동 채택 |
| `IDEA_BANK_EVAL` 필드 누락 | 1회 재시도. 재시도도 누락 시 라우터가 자동 `included` 강제 + `meeting-idea-bank` 를 `ACTIVE_AGENTS` 끝에 자동 추가 |
| `IDEA_BANK_EVAL: included` 인데 `meeting-idea-bank` 가 `ACTIVE_AGENTS` 에 없음 | 라우터가 자동 추가 + `AGENTS_ADDED` 에 자동 등록 |
| `IDEA_BANK_EVAL: excluded` 인데 `meeting-idea-bank` 가 `ACTIVE_AGENTS` 에 있음 | 라우터가 자동 제거 + `AGENTS_REMOVED` 에 자동 등록 |
| `INITIAL_AGENDA` 가 비어있지 않음 (진행자 3단계 응답) | ⛔ **강력 지침** ([[meeting-agenda-debaters-only]]). 라우터가 자동 `[]` 으로 보정 + 채팅 경고 1회 출력: `⚠ 진행자 INITIAL_AGENDA 무시 — 어젠다는 토론자 발언으로만 등록`. 회의 진행은 계속, agenda.md 는 빈 ⏳ 처리 대기로 초기화 |

#### 6-2. 풀 변경 반영

- 메인 라우터의 "현재 활성 풀" 변수를 `ACTIVE_AGENTS` 로 교체
- `participants.md` 끝에 변경 이력 append:
  ```markdown
  ## 서브주제 {N}: {title} — 풀 변경
  - ➕ 추가: {AGENTS_ADDED} 
  - ➖ 제거: {AGENTS_REMOVED}
  - 사유: {AGENTS_CHANGE_REASON}
  - 최종 활성: {ACTIVE_AGENTS}
  ```
- 변경이 `[]/[]` 면 이 섹션 생략

#### 6-3. 새로 합류한 토론자 컨텍스트 보강

`AGENTS_ADDED` 의 토론자는 이 서브주제 첫 호출 시 컨텍스트 패키지에 다음을 추가:

```
=== 회의 합류 안내 ===
당신은 이번 서브주제부터 회의에 합류합니다. 위 "진행된 서브주제 결론" 을 읽고 큰 그림을 파악한 뒤 이번 서브주제 토론에 참여하세요.
```

backend 별 구체 동작:
- `backend: claude` → 매 라운드 풀 컨텍스트 재첨부 구조이므로 자동으로 진행된 결론이 포함됨. 위 합류 안내 한 줄만 prepend.
- `backend: codex` → `.thread` 파일이 있으면 (= 이전 서브주제에서 참여한 적 있다가 빠진 후 재합류) resume + 델타에 "재합류" 안내. `.thread` 가 없으면 (= 회의 첫 합류) bootstrap 절차로 진행하되 "진행된 서브주제 결론" 전체를 부트스트랩 컨텍스트에 포함.

#### 6-4. 풀에서 빠진 토론자 처리

`AGENTS_REMOVED` 의 토론자는 이번 서브주제 라운드에서 호출되지 않을 뿐, 세션 자체는 보존한다:
- `.thread` 파일 유지 (다음 서브주제에서 재합류 가능)

### 7. 회의 종결 — 어젠다 누적 리스트 완주 시 자동 진입 (투표·진행자 중단 폐지)

⛔ **회의 종료 투표·진행자 중단 제안 폐지** (2026-05-28). PROPOSE_END_MEETING / END_AGREE / ADDITIONAL_SUBTOPIC 모드 전부 제거. **모든 어젠다 항목이 본 토론을 거쳐 자동 종결될 때까지 회의 진행**.

#### 7-1. 어젠다 누적 리스트

회의는 **단일 누적 어젠다 리스트** 를 유지한다:
- 첫 서브주제 (sub-01) = 어젠다 결정 메타. 진행자 초기 등록 + 토론자 SPEAK 응답에 등록된 어젠다 항목 + NEST_REQUEST 가 변환된 어젠다 항목 (어젠다 모드 NEST 정책, 아래 4-3-A 참조) 모두 누적
- 어젠다 메타 자동 종결 후 → 진행자가 어젠다 리스트에서 다음 항목 선택 → 새 서브주제 (sub-02 ~ sub-N) 본 토론
- 본 토론 서브주제에서 NEST_REQUEST = 일반 즉시 분기 (어젠다 리스트와 무관, 자식 서브주제)
- 본 토론 도중 토론자가 새 어젠다 항목을 제기하고 싶으면 발언에 `ADD_AGENDA: <항목>` 명시 (라우터가 어젠다 리스트에 추가, 분기는 안 함)

#### 7-2. 회의 자동 종결 조건

다음 두 조건 모두 만족 시 라우터가 8단계 (FINAL_SUMMARY) 자동 진입:
1. 어젠다 리스트의 모든 항목이 본 토론 서브주제로 다뤄지고 자동 종결됨 (cp == active)
2. 본 토론 중 추가된 어젠다 항목도 모두 처리됨 (= 어젠다 리스트 = 처리 완료 리스트)

진행자는 회의를 임의로 중단할 권한 없음. 어젠다 리스트가 0개 되어야 종결.

#### 7-3. 어젠다 리스트 영속화

라우터는 회의 디렉터리에 `agenda.md` 파일 유지:

```markdown
# 어젠다 누적 리스트

## ⏳ 처리 대기
- [ ] (3) 코어 루프 설계 (designer 등록, sub-01 메타)
- [ ] (5) 2D 쿼터뷰 아트 Bible (art-director 등록)

## ✅ 처리 완료
- [x] (1) 인플레이션/Sink → sub-02 → 결론
- [x] (2) 결제 퍼널 → sub-03 → 결론
```

매 어젠다 항목 추가·종결 시 라우터가 갱신. 회의 종결 시 처리 완료 100% 검증.

### 7-A. CEO 단독 결정 (CEO_DECISION) — 2개 트리거 (2026-05-28 축소·재축소)

다음 2개 상황에서 메인 라우터가 `meeting-ceo` 를 단독 호출해 결정을 받는다. CEO 의 응답은 그대로 채택되며 추가 투표·동의 없이 라우터가 다음 단계로 진행:

| 트리거 위치 | 조건 | CEO 선택지 |
|------------|------|------------|
| 4-4 | 무한 대화 가드 (`speech_count >= 3 × len(active)`) | A) 즉시 종결 (현재까지 발언으로 결론) / B) 강제 PASS 사이클 1회 / C) 자식 서브주제 분기 (CEO 가 핵심 쟁점 명시) |
| 4-5 | NEST cap overflow (`nest_count >= 10` 인데 새 NEST_REQUEST, 일반 서브주제만) | A) 분기 강행 (cap +1, 이번만) / B) 흡수 (NEST 기각, 어젠다 항목으로만 등록) |

> ⚠ **추가 폐지 (2026-05-28 2차)**: "7 회의 종료 단계 ADDITIONAL_SUBTOPIC 2회 연속" CEO 트리거 폐기 — END_AGREE 자체 폐지로 자동 소멸. NEST cap overflow 의 "C) 회의 종결 강제" 옵션도 폐기 (진행자가 회의 중단 불가).

#### 7-A-1. 호출 컨텍스트 패키지

```
=== CEO 단독 결정 요청 ===
큰 주제: {topic}
현재 단계: {4-4 무한 대화 가드 / 4-5 NEST cap overflow / 7 회의 종료 루프}
트리거 사유: {speech_count={N} >= 3 × {len(active)} / nest_count={N} 초과 / 같은 ADDITIONAL_SUBTOPIC 2회 연속}

=== 진행된 서브주제 결론 (이전) ===
{없으면 "없음"}

=== 현재 서브주제 / NEST_REQUEST / 종료 안건 ===
{컨텍스트 전체 — 회의록·잠정 결론·반복된 ADDITIONAL_POINT 등}

=== 선택지 ===
A) {옵션 A 설명}
B) {옵션 B 설명}
[C) {옵션 C 설명}]   (해당 단계에 따라)

=== 모드 ===
CEO_DECISION

=== 응답 형식 ===
CEO_DECISION: A | B | C | ADVISORY
REASON: <한 줄, 왜 그 선택인지 — ADVISORY 면 왜 자문이 필요한지>
[FORCED_SPEAKER: <옵션 B/C 시 강제 발언자 또는 NESTED_TITLE>]
[ADVISORY_REQUEST: [agent1, agent2, ...]   — CEO_DECISION = ADVISORY 일 때만, 최소 1명 최대 5명, advisory pool 에서 선택]
```

#### 7-A-2. 호출 방식

- backend=claude 인 CEO → Task 도구로 `meeting-ceo` 서브에이전트 호출 (`subagent_type: "meeting-ceo"`)
- backend=codex 인 CEO → 메인 라우터 Bash 직접 호출 (resume 패턴, 입력 모드: CEO_DECISION) — 단, CEO 는 backend=claude 강제(2-0) 라 이 경로는 사실상 미사용
- 응답 파싱: `CEO_DECISION: A|B|C|ADVISORY` + `REASON` + 옵션 부가 필드
- 형식 어김 시 1회 재시도. 다시 실패 시 기본값 (가장 보수적 옵션: "종결" 또는 "흡수") 으로 라우터 결정 + 회의록에 raw 기록
- `CEO_DECISION = ADVISORY` 면 7-A-5 자문 사이클 진입

#### 7-A-3. 채팅 출력 형식 (1차 호출, A/B/C 직접 결정)

```markdown
⚖️ **CEO 단독 결정 요청** — {트리거 사유}
선택지: A) {옵션A} / B) {옵션B} [/ C) {옵션C}]

**🤖 ceo**: CEO_DECISION = {A|B|C} — {REASON}
→ 라우터: {다음 단계 명시}
```

#### 7-A-4. 회의록 기록

서브주제 회의록 (또는 main.md, 회의 종료 단계) 에 다음 섹션 append:

```markdown
## CEO 결정 (round {N})
- 트리거: {조건}
- 선택지: A) ... / B) ... [/ C) ...]
- [자문 받은 advisor: [agent1, agent2, ...]   ← ADVISORY 사이클 있었을 때만]
- 결정: {A|B|C}
- 사유: {REASON}
```

#### 7-A-5. 자문 사이클 (CEO_DECISION = ADVISORY 시)

CEO 가 결정 전 Director 급 + 아이디어 뱅크에게 의견을 구할 수 있다. **중립적 결정** 보장 — 자문은 의견 수렴일 뿐 결정권은 CEO 단독.

##### Advisory pool 자동 식별

후보 풀 글로빙 결과(`~/.claude/agents/meeting-*.md`) 중 다음 조건 모두 만족하는 에이전트 자동 등록:

- **name 패턴 매칭** (다음 중 하나):
  - `meeting-(.*-)?director(-.*)?$` (예: `meeting-game-product-director`, `meeting-game-art-director`)
  - `meeting-(.*-)?td(-.*)?$` (Technical Director, 예: `meeting-game-td`)
  - `meeting-(.*-)?producer(-.*)?$` (예: `meeting-game-producer`)
  - 정확히 `meeting-idea-bank`
- `backend: claude` (codex 백엔드는 2-0-2 본 진행 금지)
- `meeting-ceo` 본인은 제외 (자기 자문 무의미)

advisory pool 이 0명이면 CEO 의 ADVISORY 선택을 라우터가 무효화 + CEO 1차 응답을 A/B/C 중 가장 보수 옵션으로 자동 변환 + 채팅에 `⚠ 자문 풀 부재, A 자동 채택` 안내.

##### 자문 호출 절차

1. 라우터가 CEO 응답의 `ADVISORY_REQUEST: [...]` 검증:
   - advisory pool 에 있는 이름만 통과. 무효 이름은 그 항목 제거 + 경고
   - 검증 후 0명이면 advisory pool 부재 처리와 동일 (보수 옵션 강제)
2. 검증 통과 에이전트들에게 **동시(병렬) ADVISORY 모드 호출**. 호출 컨텍스트:

   ```
   === CEO 자문 요청 ===
   당신은 회의의 결정자(CEO)로부터 단독 자문을 요청받았습니다. 의견은 수렴되고 최종 결정은 CEO 단독으로 내립니다 (당신의 의견이 결정을 뒤집지 않음). 중립적 시각·자기 expertise 관점에서 5줄 이내로 답하세요.

   === 큰 주제 ===
   {topic}

   === 트리거 단계 / 사유 ===
   {4-3-c 동률 / 4-3-d pass_streak / 4-4-2 ADDITIONAL_POINT 루프 / NEST_VOTE 동률 / 7 종료 루프}

   === 트리거 안건 컨텍스트 ===
   {회의록·잠정 결론·반복된 안건 등}

   === CEO 가 검토 중인 선택지 ===
   A) {옵션A 설명}
   B) {옵션B 설명}
   [C) {옵션C 설명}]

   === CEO 의 자문 사유 ===
   {CEO 1차 응답의 REASON 그대로}

   === 모드 ===
   ADVISORY

   === 응답 형식 ===
   ADVISORY_OPINION:
   <5줄 이내 한국어. 본인 expertise 관점에서 A/B/C 중 어느 쪽을 권하는지 또는 제3안 메모 + 핵심 근거 1~2개. 결정을 뒤집으려는 시도 금지 — 시각 제공만.>
   [PREFERRED_OPTION: A | B | C   ← 선호 옵션 명시 (선택). 명시 안 해도 됨]
   ```

3. 응답 수집 (병렬). 형식 어김 → 1회 재시도. 실패 시 raw 기록 + 다음 advisor 진행
4. 모은 자문 의견 + 원 트리거 컨텍스트로 **CEO 에게 2차 CEO_DECISION 호출**:

   ```
   === CEO 2차 결정 (자문 완료) ===
   [원 1차 컨텍스트 그대로]

   === 받은 자문 의견 ===
   - 🤖 advisor1 (PREFERRED: B): {ADVISORY_OPINION}
   - 🤖 advisor2 (선호 미명시): {ADVISORY_OPINION}
   ...

   === 응답 형식 ===
   CEO_DECISION: A | B | C   ← ADVISORY 옵션 제외, 이번엔 반드시 직접 결정
   REASON: <한 줄, 자문 의견을 어떻게 반영했는지>
   [FORCED_SPEAKER: ...]
   ```

5. 2차 응답에서 또 ADVISORY 가 들어오면 무한 사이클 방지 — 자동으로 가장 보수 옵션 채택 + 채팅에 `⚠ 자문 사이클 2회 시도, 보수 옵션 강제 채택`
6. 자문 받은 의견 전체는 회의록 7-A-4 의 "자문 받은 advisor" 라인 + 별도 섹션:

   ```markdown
   ### CEO 자문 의견 (round {N} 결정 전)
   - 🤖 advisor1: {ADVISORY_OPINION}
   - 🤖 advisor2: {ADVISORY_OPINION}
   ```

##### 채팅 출력 (자문 사이클)

```markdown
⚖️ **CEO 단독 결정 요청** — {트리거 사유}
선택지: A) {옵션A} / B) {옵션B} [/ C) {옵션C}]

**🤖 ceo (1차)**: CEO_DECISION = ADVISORY — {REASON}
자문 대상: [advisor1, advisor2, ...]

💬 **CEO 자문 의견**:
- 🤖 advisor1 [선호: B]: > {ADVISORY_OPINION}
- 🤖 advisor2: > {ADVISORY_OPINION}

**🤖 ceo (최종)**: CEO_DECISION = {A|B|C} — {REASON}
→ 라우터: {다음 단계 명시}
```

##### 중립성 보장 가드

- advisor 의 PREFERRED_OPTION 은 참고용. CEO 의 최종 결정에 동률·과반수 규칙 적용 X (advisor 가 모두 B 선호해도 CEO 가 A 결정 가능)
- CEO 2차 응답의 REASON 은 "advisor N명 다수가 B 선호이나 1-way door 비가역성으로 A 채택" 같은 **자문 반영의 명시적 사유** 권장 (강제 아님)
- 자문 사이클은 1 트리거당 최대 1회 (위 5번 가드)

### 8. 최종 종합 — 진행자 호출 (어젠다 리스트 완주 시 라우터 자동 진입)

**진입 조건**: 7-2 자동 종결 조건 충족 시 (어젠다 누적 리스트 = 처리 완료 100%) 라우터가 자동 호출. 진행자가 호출 시점을 결정하지 않음 — 어젠다 리스트가 0개 됐을 때 라우터가 결정.

```
=== 큰 주제 ===
{topic}

=== 처리 완료 어젠다 리스트 ===
1. {agenda1} → {sub-NN} → {결론 요약}
2. {agenda2} → {sub-NN} → {결론 요약}
...

=== 진행된 서브주제 결론 (자식·손자 포함 인덱스) ===
... (전체)

=== 당신의 임무 ===
회의 전체 종합. 양측 의견 병기 + 사용자에게 어떤 결정을 권하는지(권고는 선택). 미터치 어젠다는 0이어야 정상 (이번 회의에서 모두 처리됨).

응답 형식:
FINAL_SUMMARY:
<5-10줄 한국어 종합. 큰 주제 축별 결론 정리 + 합의/이견 분리 + 권고>
```

`main.md` 의 `## 종합 (진행자)` 섹션에 기록 + 종료 시간 기록. **본 단계 직후 9단계 (최종 출력) 로 직행** (Codex 최종 의견 수렴 단계 폐지 — 회의 본 진행에서 진행자가 codex 백엔드 토론자를 이미 활용했음).

### 9. 최종 종합 출력

라이브 스트리밍의 마무리로 채팅에 다음 형식 출력:

```markdown
---

## 🗣 회의 종료

**주제**: {topic}
**서브주제 {N}개** · 총 {총_라운드}라운드

### 서브주제별 결론
1. **{sub1}** → {sub1_conclusion_1line}
2. **{sub2}** → {sub2_conclusion_1line}
...

### 종합 (진행자)
{final_summary}

📁 회의록: `{dir_path}/main.md`
```

이미 모든 발언이 위쪽 채팅에 라이브로 출력되어 있으므로, 9단계는 **종합과 인덱스만** 압축해서 마무리. 8-A Codex 최종 의견은 이미 8-A-4 에서 출력됐으므로 9단계에서 재출력하지 않음 (인덱스 한 줄로 "Codex 의견 N건 main.md 참조" 정도만).

## 중첩 서브주제 (Nested Subtopics — NEST 즉시 분기, 최대 depth 5)

### 개요

토론자가 발언 중 자식 서브주제로 분기가 필요한 쟁점을 인지하면 SPEAK 응답에 `NEST_REQUEST` 블록 포함 → **라우터가 즉시 자식 서브주제 개시** (투표·진행자 정리 없음). 부모 사이클은 자식 종결 후 재개.

- **Depth 정의**: main.md = depth 0. main 의 직속 서브주제 = depth 1 (sub). 그 자식 = depth 2 (자식). 손자 = depth 3. 증손자 = depth 4. 고손자 = depth 5. **depth 6 절대 금지** (2026-05-28 7차 확장).
- **파일명 규칙** (점 구분 인덱스):
  - depth 1: `sub-01-{slug}.md`, `sub-02-{slug}.md`
  - depth 2: `sub-01.01-{slug}.md`, `sub-01.02-{slug}.md`, `sub-02.01-{slug}.md`
  - depth 3: `sub-01.01.01-{slug}.md`, `sub-01.01.02-{slug}.md`
  - depth 4: `sub-01.01.01.01-{slug}.md`
  - depth 5: `sub-01.01.01.01.01-{slug}.md`

### 즉시 분기 메커니즘 (2026-05-28 [[meeting-nest-immediate-branch]])

#### 경로: 토론자 발언 중 NEST_REQUEST 포함

토론자가 발언 끝에 `NEST_REQUEST` 블록 포함 (common-debater.md 모드 1 참조):

```
SPEAK: YES
CONTENT:
<발언 본문>
NEST_REQUEST:
TITLE: <자식 서브주제 제목, 30자 이내>
REASON: <왜 자식으로 — 다른 도메인·5줄 한계·시간 절약 1줄>
```

#### 라우터 처리 (4-3 참조)

1. CONTENT 를 부모 서브주제 md 에 append
2. NEST 가드 체크:
   - `current_depth >= 5` → NEST 보류 (deferred_nests 등록 + 부모 사이클 계속, [[meeting-nest-depth3-deferred]] 6차 정책)
   - `nest_count >= 10` → CEO_DECISION 호출 (4-5)
3. 통과 시 즉시 자식 서브주제 개시:
   - 제목 = `NEST_REQUEST.TITLE`
   - 활성 풀 = **부모 풀 상속** (진행자가 자식 진입 직후 6단계 호출로 조정 가능)
   - 발언 순서 = **요청자가 첫 발언자** + 나머지 부모 풀 순서대로
4. 자식 사이클 진입 (재귀, depth+1) — 자식도 동일 규칙 (라운드로빈 + 연속 PASS 종결)
5. 자식 종결 후:
   - 자식 결론을 부모 md 의 "## 자식 서브주제 결론" 섹션에 append
   - 부모 사이클 재개 (다음 i 위치부터)

#### 진행자의 NEST 권한 (단순화)

| 권한 | 허용? |
|------|------|
| NEST_REQUEST 판단 (APPROVE/REJECT) | ❌ 폐지 (자동 분기) |
| 자식 진입 직후 풀 조정 (6단계 호출로 ADD/REMOVE) | ✅ |
| 자식 종결 시 결론 작성 | ✅ |
| NEST 트리거 조건 가이드라인 제시 (도메인 교차·5줄 한계·시간 절약) | ✅ (강제 아님, 토론자 자율 판단용) |

### NEST cap 정책

- **회의 전체 누적 cap = 10** (전체 회의에서 11번째 NEST_REQUEST 시 CEO_DECISION)
- **depth cap = 5** (depth 5 에서 NEST_REQUEST 자동 보류, 2026-05-28 7차 확장 3→5)
- 두 cap 모두 라우터 자동 적용 (진행자 판단 없음)

### 부모 일시 정지 → 자식 진행 → 부모 재개 (회의록 기록)

부모 md 에 자동 추가:

```markdown
## NEST 분기 (i={i}, depth={parent_depth}→{child_depth})
- 요청자: {requester_agent}
- 자식 서브주제: [./sub-{child_path}.md](./sub-{child_path}.md)
- 분기 사유: {NEST_REQUEST.REASON}
- 자식 종결 후 부모 사이클 재개 예정
```

자식 종결 후 부모 md 에 자동 append:

```markdown
## 자식 서브주제 결론 ({child_path})
{child_conclusion}
```

부모 사이클 재개 시 다음 호출되는 토론자는 자식 결론을 누적 서브주제 md 컨텍스트로 자동 수용 (별도 안내 불필요 — 컨텍스트 패키지가 부모 md 전체를 포함).

### main.md 인덱스 트리

서브주제 결론은 depth 별 들여쓰기로 표시:

```markdown
## 서브주제 결론

### 1. 어젠다 결정
- 결론: ...

### 2. 수직 슬라이스 + KPI
- 결론: ...
  #### 2.1. (자식, depth 2) 멀티플레이 검증 범위 정의
  - 결론: ...
    ##### 2.1.1. (손자, depth 3) Netcode 동기화 모델 선택
    - 결론: ...
      ###### 2.1.1.1. (증손, depth 4) tick rate 결정
      - 결론: ...
        ####### 2.1.1.1.1. (고손, depth 5) AOI 알고리즘 선택
        - 결론: ...
  #### 2.2. (자식, depth 2) 30분 세션 KPI 컬럼 표
  - 결론: ...

### 3. 코어 루프
- 결론: ...
```

### 자식 서브주제 토론자 풀

자식은 **부모 풀 기본 상속**. 자식 진입 직후 진행자가 6단계 호출 (ACTIVE_AGENTS / AGENTS_ADDED / AGENTS_REMOVED + IDEA_BANK_EVAL) 로 조정 가능. 새로 합류한 토론자는 부모 컨텍스트 + 자식 진입 사유 같이 받음.

### 자식 서브주제 종결

자식도 부모와 동일 규칙: 연속 PASS == 자식 활성 수 시 자동 종결. 별도 투표·동의 없음.

### 강제 종결 가드 (자식 depth 무관 동일 적용)

- 자식 `speech_count >= 3 × len(active)` → CEO_DECISION 호출 (4-4 무한 대화 가드)
- 자식 진입 시 `current_depth >= 5` 인데 NEST_REQUEST → 자동 보류 + deferred_nests 등록 + 회의록 기록 (depth 5 보류 정책)

## 안전 가드

| 상황 | 가드 |
|------|------|
| **CRITICAL — `codex exec resume` 에 `-s` 또는 `-C` 옵션 사용** | **즉시 중단·재작성**. resume 는 두 옵션 모두 미지원, CLI 가 "tip: to pass '-s' as a value, use '-- -s'" 로 거부하고 응답 0바이트. 3-B 블록의 정확한 옵션 (`-m -c -o --json --skip-git-repo-check` + thread_id + stdin) 만 사용. 응답 캡처는 항상 `-o $OUTPUT`. **이 실수가 반복되면 회의 자체 중단** |
| 진행자가 depth 5 에서 START_NESTED_SUBTOPIC 시도 | 거부 + "동일 depth 형제 서브주제로 분기 또는 부모 종결 후 다음 depth-1 서브주제로 처리" 안내 + 1회 재시도. 재시도도 NESTED 면 강제 CONTINUE 처리 |
| 진행자가 필요한 페르소나가 후보 풀(글로빙 결과)에 부재 | **자동 fallback** (사용자 질문 금지). 진행자가 가장 비슷한 expertise 의 후보로 자동 대체 + 결론·main.md 에 "⚠ 미커버 도메인: {도메인 명} (사유: 후보 풀에 페르소나 부재, 자동 대체)" 메모 추가. `/meeting-agent` 자동 호출 안 함 — 사용자가 회의 종료 후 후속 회의로 보강 가능 |
| 자식 서브주제가 부모 컨텍스트 없이 시작 | 자식 시작 시 컨텍스트 패키지에 부모 서브주제 md 전체 + 분기 사유 (NESTED_CONTEXT) 포함 필수 |
| **CRITICAL — `grep -P` 로 thread_id 캡처** | MSYS/Windows 로케일에서 "supports only unibyte and UTF-8 locales" 로 실패. 반드시 `sed 's/.*"thread_id":"\([^"]*\)".*/\1/'` 사용 |
| codex 권한 차단 (settings.json 에 `Bash(codex exec:*)` 없음) | 0-1 자동 fallback: `codex_available=false` 설정, claude 단독 자동 진행 + 채팅에 1회 안내. 사용자 질문 금지 |
| codex CLI 부재·로그인 안 됨 | 0-2 자동 fallback: `codex_available=false` 설정, claude 단독 자동 진행 + 채팅에 1회 안내. 사용자 질문 금지 |
| 진행자가 무한 어젠다 등록 | **무제한 허용** (2026-05-28). 사용자 정책: 토큰 과다보다 품질 우선. 어젠다 리스트 완주까지 회의 진행. |
| codex exec 호출 런타임 실패 | exit 2 (로그인) → 그 에이전트만 claude 로 자동 fallback + 채팅 1회 안내 / exit 124 (timeout) → 1회 재시도 / 기타 → 그 토론자 라운드 SPEAK: NO 처리. **회의 중단 안 함** |
| codex exec 가 연속 3회 실패 | 그 에이전트만 claude 로 자동 fallback + 채팅 1회 안내. **회의 중단 안 함** (다른 에이전트는 정상 진행) |
| `${MEETING_DIR}/codex-sessions/<agent>.thread` 파일 존재 but resume 호출이 "Session not found" 등으로 실패 | 그 thread 파일 삭제 → 같은 차례 bootstrap 으로 폴백 1회. 폴백도 실패 시 그 에이전트만 claude 로 자동 fallback |
| thread_id 캡처 실패 (bootstrap JSONL 첫 줄에 thread_id 없음) | 1회 재시도. 다시 실패 시 다음 라운드부터 그 토론자는 매번 bootstrap (resume 비활성). 사용자에게 경고 출력 (토큰 비용 무시 정책이라 회의는 계속) |
| 토론자 응답이 형식 어김 | 1회 재시도. 다시 실패 시 그대로 회의록에 raw 기록 + 패스 처리 |
| 진행자 응답이 형식 어김 | 1회 재시도. 실패 시 메인 라우터가 안전 default 적용 (FINAL_CONCLUSION 강제 작성 또는 대기 리스트 첫 항목 자동 채택) |
| 서브주제 무한 대화 (speech_count >= 3 × 활성) | **CEO_DECISION 호출 (4-4)** — CEO 가 즉시 종결/강제 PASS 사이클/자식 분기 단독 결정 |
| NEST cap overflow (nest_count >= 10 인데 새 NEST_REQUEST, 일반 본 토론만) | **CEO_DECISION 호출 (4-5)** — CEO 가 분기 강행/흡수(어젠다 등록) 단독 결정. "회의 종결 강제" 옵션 폐기 (7 단계 폐지) |
| depth 5 에서 NEST_REQUEST 도달 (일반 본 토론) | 자동 보류 + `deferred_nests` 등록 + 회의록에 "depth 5 한계 보류" 기록 + 부모 사이클 계속. 하위 종결 후 상위 사이클 재개 시 다음 토론자 컨텍스트에 "보류된 NEST — 재신청 권장" 안내. **진행자에게 사전 안내** (자식 진입 직후 6단계 호출 시 "depth 5 진입, NEST 보류 모드" 채팅 출력) |
| 어젠다 결정 메타에서 NEST_REQUEST 도달 | **즉시 분기 X**, 어젠다 누적 리스트에 항목 등록만 (4-3-A 정책) |
| `IDEA_BANK_EVAL` 필드 누락 (진행자 3·6단계 응답) | 1회 재시도. 재시도도 누락 시 라우터가 자동 `included` 강제 (적극 default) |
| `IDEA_BANK_EVAL: excluded` 인데 사유가 제외 신호 3가지 모두 명시 안 됨 | 1회 재시도 (보정 안내). 그래도 모호하면 `included` 강제 |
| `BACKEND_SELECTION` 필드 누락 (진행자 3·6단계 응답) | 1회 재시도. 재시도도 누락 시 라우터가 ACTIVE_AGENTS 전원을 frontmatter `default_backend` 로 자동 채움 + 채팅 경고 |
| `BACKEND_SELECTION` 에 `meeting-ceo` 포함 | 라우터가 그 항목 무시 (CEO backend 강제 = claude) |
| `meeting-ceo` 가 활성 풀에 없음 | 라우터 자동 추가 (2-0 CEO 강제 참관). 진행자 응답 무시하고 ACTIVE_AGENTS + SPEAKING_ORDER 끝에 자동 append |
| 후보 풀에 `meeting-ceo.md` 자체가 부재 | 회의 시작 전 채팅에 1회 자동 출력 후 자동 중단 (사용자 질문 금지). 별도 세션 안내. |
| CEO_DECISION 응답 형식 어김 | 1회 재시도. 다시 실패 시 가장 보수적 옵션(종결/흡수) 자동 채택 + raw 회의록 기록 |
| CEO 가 일반 투표·동의·SPEAK_OR_PASS 호출에 잘못 포함됨 | 라우터가 호출 대상 리스트에서 자동 제외 (2-0 규칙) |
| `meeting-ceo.md` frontmatter 가 `backend: codex` | 회의 시작 전 거부 + 사용자에게 `backend: claude` 변경 요청 (2-0 규칙) |
| 활성 풀 0명 됨 | 채팅에 1회 자동 출력 후 자동 중단 (사용자 질문 금지). 별도 세션 안내. |
| CEO 1차 응답이 `CEO_DECISION: ADVISORY` 인데 ADVISORY_REQUEST 비어있거나 무효 이름만 | 1회 재시도 (보정 안내 포함). 실패 시 가장 보수 옵션(종결/흡수/기각) 자동 채택 (7-A-5) |
| Advisory pool 자체가 0명 (`*-director` · `*-td` · `*-producer` · `idea-bank` 없음) | CEO 의 ADVISORY 선택 무효화 + A/B/C 중 가장 보수 옵션 자동 채택 + `⚠ 자문 풀 부재` 안내 |
| CEO 2차 응답에서 또 `ADVISORY` | 무한 사이클 방지. 자동으로 가장 보수 옵션 채택 + `⚠ 자문 사이클 2회 시도` 안내 |
| ADVISORY 호출 받은 advisor 응답 형식 어김 | 1회 재시도. 실패 시 raw 기록 + 그 advisor 의견 제외하고 진행. 모든 advisor 실패 시 CEO 1차 응답을 가장 보수 옵션으로 자동 변환 |
| ADVISORY_REQUEST 에 `meeting-ceo` 자기 자신 포함 | 그 항목 제거 (자기 자문 무의미, 7-A-5) |
| ADVISORY 응답에 NEST_REQUEST 블록 포함 | 라우터가 NEST_REQUEST 무시. ADVISORY_OPINION 본문만 사용 (ADVISORY 는 의견 수렴 단계, 분기는 일반 라운드 발언에서만) |
| ADVISORY 응답이 SPEAK 키 사용 (또는 폐지된 END_AGREE/SUBTOPIC_END_AGREE 키 사용) | 그 키 무시. ADVISORY_OPINION + PREFERRED_OPTION 외 모두 raw 기록만 |
| 진행자가 ACTIVE_AGENTS 를 빈 배열로 반환 | 1회 재시도. 다시 실패 시 라우터가 후보 풀에서 어젠다 매칭 5명 자동 선정 (회의 중단 권한 없음) |
| AGENTS_ADDED 가 후보 풀에 없는 이름 포함 | 그 이름만 제거하고 진행 (조용히 보정). 모두 무효면 변경 무시하고 현재 풀 유지 |
| ACTIVE_AGENTS 가 1명만 남음 | 진행자에게 "최소 2명 유지" 강제 후 1회 재시도. 그래도 1명이면 자기반박 모드 안내 후 진행 |
| AGENTS_REMOVED 된 토론자가 다음 서브주제에서 재합류 | `.thread` 파일이 남아있으면 codex resume + "재합류" 델타 안내. claude 는 매번 풀 컨텍스트라 별도 처리 불필요 |
| 진행자가 BACKEND_SELECTION 에서 같은 에이전트의 backend 를 서브주제마다 바꿈 | 허용 (페르소나 동일, 호출 방식만 변경). 단 같은 서브주제 안에서는 불변 |

## 토론자 호출 디테일 (backend 분기)

각 호출 방식은 그 토론자 에이전트 파일 frontmatter `backend` 필드로 결정.

### Backend: `claude` (서브에이전트)

Task 도구로 호출. `subagent_type` = 그 에이전트의 `name` (예: `meeting-debater-claude`, `meeting-debater-game-econ`).

> **현재 한계 (환경 의존)**: 이 환경의 `Agent` 도구에는 spawn 한 서브에이전트를 ID 로 이어 부르는 `SendMessage` 가 노출돼 있지 않음. 따라서 Claude 토론자는 **매 라운드마다 새 Task 호출 + 풀 컨텍스트 재첨부** 가 현재로선 불가피. (Codex 토론자처럼 세션 영속화 안 됨.) 향후 `SendMessage` 가 가용해지면 codex 패턴과 동일하게 `${MEETING_DIR}/claude-sessions/<agent>.id` 에 agent ID 저장 + 이후 라운드 SendMessage 로 델타 전달 구조로 전환할 것.

입력 컨텍스트 패키지(4-1 본문) 의 서두에 반드시 다음 prepend:

```
ROLE_FILE_COMMON: C:\Users\NX3GAMES\.claude\skills\meeting\roles\common-debater.md

(아래 컨텍스트와 별개로, 위 ROLE_FILE_COMMON 을 반드시 먼저 읽고 그 응답 형식을 준수하라. 본인의 페르소나는 서브에이전트 정의 파일에서 자동 로드된다.)

[4-1 컨텍스트 패키지 그대로]
```

응답은 stdout 직접 반환 (output 파일 미사용). 메인 라우터가 `SPEAK:` 키 파싱 (NEST_REQUEST / ADD_AGENDA 는 SPEAK 응답의 하위 블록).

### Backend: `codex` — 메인 라우터 직접 호출 + thread 영속 세션 + 파일 IO

⛔ **`meeting-debater-codex` (혹은 다른 `backend: codex` 에이전트) 를 Task 도구의 서브에이전트로 호출 금지.** Claude Code auto-mode classifier 가 서브에이전트의 nested `codex exec` 호출을 "중첩 자율 에이전트 루프"로 분류해 차단한다.

**핵심 원칙 — 세션 영속화 + 델타 전송**:
- **첫 호출 (회의에서 그 토론자의 첫 차례)** : `codex exec` 로 풀 부트스트랩 (역할/페르소나/큰주제/현재 서브주제 컨텍스트) → 응답 첫 줄 `{"type":"thread.started","thread_id":"<UUID>"}` 에서 `thread_id` 캡처 → `${MEETING_DIR}/codex-sessions/<agent>.thread` 에 저장
- **2회차 이후 모든 호출** : `codex exec resume <thread_id>` + **델타 컨텍스트만** (직전 라운드 발언, 진행자 판정, 이번 라운드 모드 지시). 페르소나·큰주제·이전 회의록은 codex 세션이 기억하므로 재첨부 금지
- 토큰 캐시 히트율 70%+ (검증값), 페르소나 일관성 자동 보장

#### 세션 파일 구조

```
${MEETING_DIR}/
  codex-sessions/
    meeting-debater-codex.thread       # 단일 줄: thread_id UUID
    meeting-debater-game-econ.thread   # 토론자별 한 파일
    ...
  codex-io/
    sub-01-round-1/
      input.txt
      output.txt
    sub-01-round-2/
      ...
```

`.thread` 파일이 없으면 = 그 토론자의 첫 호출. 있으면 = resume.

#### 호출 절차

**0. 세션 파일 체크 (분기 결정)**

```bash
SESSION_DIR="${MEETING_DIR}/codex-sessions"
mkdir -p "$SESSION_DIR"
THREAD_FILE="${SESSION_DIR}/${AGENT_NAME}.thread"

if [ -f "$THREAD_FILE" ]; then
  MODE="resume"
  THREAD_ID=$(cat "$THREAD_FILE")
else
  MODE="bootstrap"
fi
```

**1. IO 파일 준비**

```bash
IO_DIR="${MEETING_DIR}/codex-io/sub-${SUB_NN}-round-${ROUND}"
mkdir -p "$IO_DIR"
INPUT="${IO_DIR}/input-${AGENT_NAME}.txt"
OUTPUT="${IO_DIR}/output-${AGENT_NAME}.txt"
rm -f "$OUTPUT"
```

(폐지) `SUBTOPIC_END_AGREE` / `END_AGREE` 단계 — 7단계 폐지로 자동 소멸.

**2-A. MODE=bootstrap (첫 호출) 일 때 INPUT 작성**

```
=== 역할 정의 ===
당신은 /meeting 스킬의 토론자다. 다음 두 파일을 반드시 먼저 읽고 그 정의대로 행동하라:

ROLE_FILE_COMMON: C:\Users\NX3GAMES\.claude\skills\meeting\roles\common-debater.md
PERSONA_FILE: <에이전트 정의 파일 절대 경로, 예: C:\Users\NX3GAMES\.claude\agents\meeting-debater-codex.md>

ROLE_FILE_COMMON 은 공통 응답 형식 (SPEAK_OR_PASS — NEST_REQUEST / ADD_AGENDA 는 SPEAK 응답의 하위 블록).
PERSONA_FILE 은 당신의 페르소나·전문성·행동 원칙.
이 두 파일의 정의는 이후 모든 후속 호출에서도 유지된다 (resume).

=== 큰 주제 ===
{topic}

=== 진행된 서브주제 결론 (이전) ===
{없으면 "없음"}

=== 현재 서브주제: {current_title} ===
{current_subtopic_md 전체}

=== 당신의 차례 ===
당신: {agent_name}
직전 발언자: {prev_speaker} (없으면 "없음")
모드: SPEAK_OR_PASS

응답 형식은 ROLE_FILE_COMMON 의 모드 1 정의를 그대로 따른다.
```

**2-B. MODE=resume (2회차 이후) 일 때 INPUT 작성 — 델타만**

```
=== 라운드 {round} 업데이트 ===
{변경/추가된 컨텍스트 — 아래 케이스별 패턴}
```

케이스별 델타:

| 케이스 | INPUT 본문 |
|--------|------------|
| 같은 서브주제 다음 호출 | `직전 발언자: {prev}` + 직전 발언 본문 (5줄) + `현재 cp: {N}/{len(active)}` + `이번 차례 모드: SPEAK_OR_PASS` |
| 서브주제 전환 첫 호출 | 이전 서브주제 결론 1줄 + `=== 새 서브주제: {title} ===` + `AGENDA_PICKED: {agenda_item}` + `SUBTOPIC_CONTEXT: {ctx}` + `SPEAKING_ORDER: [...]` + `당신 차례` + `모드: SPEAK_OR_PASS` |
| backend 변경 케이스 | 진행자가 BACKEND_SELECTION 에서 같은 에이전트의 backend 를 변경한 경우, 새 backend 첫 호출은 bootstrap (이전 thread 무관) |

이전 발언/큰주제/페르소나는 절대 재첨부하지 않는다. codex 세션이 기억한다.

**3. codex exec 호출**

⚠️ **CLI 옵션 차이 절대 주의**: `codex exec` (bootstrap) 와 `codex exec resume` 는 옵션 집합이 다르다.

| 옵션 | `codex exec` (bootstrap) | `codex exec resume` |
|------|--------------------------|---------------------|
| `-s workspace-write` (sandbox) | ✅ 지원 | ❌ **미지원 — 사용 금지** |
| `-C <dir>` (cwd) | ✅ 지원 | ❌ **미지원 — 사용 금지** |
| `-m <model>` | ✅ 지원 | ✅ 지원 |
| `-c <key=val>` | ✅ 지원 | ✅ 지원 |
| `-o <file>` (output-last-message) | ✅ 지원 | ✅ 지원 |
| `--json` | ✅ 지원 | ✅ 지원 |
| `--skip-git-repo-check` | ✅ 지원 | ✅ 지원 |

**원칙**: 응답 캡처는 양쪽 모두 **`-o $OUTPUT`** (output-last-message) 로 통일. agent 가 파일 쓰기 명령을 받지 않으므로 sandbox/cwd 옵션 자체가 불필요. bootstrap 도 `-o` 로 통일하면 resume 와 동일 패턴 유지 → 디버깅·재현 일관성.

(과거 호환: 만약 더 복잡한 파일 IO 가 필요해 bootstrap 에서 sandbox 가 필요한 경우에만 `-s workspace-write -C "$MEETING_DIR"` 추가 가능. resume 에는 절대 금지.)

**3-A. MODE=bootstrap (첫 호출):**

```bash
PROMPT_FILE="${INPUT}.full"
{ cat "$INPUT"; echo; echo "위 컨텍스트의 ROLE_FILE_COMMON·PERSONA_FILE 두 파일을 먼저 읽고, 모드 지시에 따라 ROLE_FILE_COMMON 의 응답 형식대로 응답하라. 응답이 곧 마지막 메시지로 캡처된다."; } > "$PROMPT_FILE"

timeout 300 codex exec --json --skip-git-repo-check \
  -m gpt-5.5 \
  -c model_reasoning_effort=medium \
  -o "$OUTPUT" \
  - < "$PROMPT_FILE" > "${IO_DIR}/codex-${AGENT_NAME}.jsonl" 2>&1
EXIT=$?

# thread_id 캡처 (JSONL 첫 줄. grep -P 는 Windows MSYS 로케일에서 깨지므로 sed 사용)
THREAD_ID=$(head -1 "${IO_DIR}/codex-${AGENT_NAME}.jsonl" | sed 's/.*"thread_id":"\([^"]*\)".*/\1/')
if [ -n "$THREAD_ID" ] && [ "$THREAD_ID" != "$(head -1 "${IO_DIR}/codex-${AGENT_NAME}.jsonl")" ]; then
  echo "$THREAD_ID" > "$THREAD_FILE"
fi
```

**3-B. MODE=resume (2회차 이후):**

```bash
PROMPT_FILE="${INPUT}.full"
{ cat "$INPUT"; echo; echo "모드 지시에 따라 ROLE_FILE_COMMON 의 응답 형식대로 응답하라 (페르소나·이전 컨텍스트는 세션에 이미 로드됨). 응답이 곧 마지막 메시지로 캡처된다."; } > "$PROMPT_FILE"

# resume 에는 -s, -C 금지 (CLI 가 거부함)
timeout 300 codex exec resume --json --skip-git-repo-check \
  -m gpt-5.5 \
  -c model_reasoning_effort=medium \
  -o "$OUTPUT" \
  "$THREAD_ID" \
  - < "$PROMPT_FILE" > "${IO_DIR}/codex-${AGENT_NAME}.jsonl" 2>&1
EXIT=$?
```

옵션 메모:
- `-o "$OUTPUT"` : agent 의 마지막 메시지를 그 파일에 직접 저장. agent 가 파일 쓰기 명령을 받을 필요 없음 → sandbox 옵션 불필요
- `--json` : 이벤트 스트림. bootstrap 에서 thread_id 캡처용, resume 도 디버깅용 유지
- `-m gpt-5.5` : 모델 강제. 두 명령 모두 동일 위치
- `-c model_reasoning_effort=medium` : low 는 페르소나 무시 위험, high 는 과사고. medium 권장. 투표·짧은 응답은 low 도 무방
- `- < "$PROMPT_FILE"` : stdin 으로 프롬프트 전달 (인자 길이 제한 회피)
- `timeout 300` : 5분 hard cap
- stdout/stderr 는 `${IO_DIR}/codex-${AGENT_NAME}.jsonl` 로 일괄 리다이렉트 (tee 불필요 — agent 응답은 `-o` 가 캡처)
- sed thread_id 캡처: `grep -P` 는 MSYS/Windows 로케일 (`grep: -P supports only unibyte and UTF-8 locales`) 에서 실패하므로 절대 사용 금지. `sed 's/.*"thread_id":"\([^"]*\)".*/\1/'` 패턴 사용

**4. exit code + 결과 파일 검증**

| 조건 | 처리 |
|------|------|
| EXIT=0 AND $OUTPUT 존재 AND 내용이 `SPEAK:` 로 시작 | 정상. $OUTPUT 그대로 사용 → 5단계 |
| EXIT=0 BUT $OUTPUT 없음/형식 어김 | 1회 재시도 (같은 MODE 로). 다시 실패 시 회의록에 raw 기록 + SPEAK: NO 처리 |
| EXIT=2 | 로그인 필요. 그 에이전트만 claude 로 자동 fallback + 채팅 1회 안내 (`⚠ codex 로그인 필요 — 별도 터미널에서 codex login 후 다음 회의에서 자동 인식. 본 회의는 claude 단독 진행`). **회의 중단 안 함** |
| EXIT=124 | timeout. 1회 재시도. 다시 timeout 이면 SPEAK: NO 처리 |
| MODE=resume AND `Session not found` 류 stderr | thread 파일 삭제 후 bootstrap 으로 1회 폴백. 폴백도 실패 시 그 에이전트만 claude 로 자동 fallback (회의 중단 안 함) |
| 기타 | stdout 마지막 줄을 회의록에 raw 기록 + SPEAK: NO 처리. 연속 3회 발생 시 그 에이전트만 claude 로 자동 fallback (회의 중단 안 함) |

**5. 응답 사용 + 출력**

```bash
RESPONSE=$(cat "$OUTPUT")
```

stdout 파싱 불필요. 회의 종료 후 `codex-io/` `codex-sessions/` 둘 다 영구 보존 (디버깅·재현).

라이브 스트리밍 정책에 따라 채팅에 `**🤖 {agent}**: > ...` 출력 + 서브주제 회의록 파일에 append.

## 파일 구조

```
./meetings/
  20260527-115342-실시간알림/
    main.md                       # 큰주제 인덱스 + 종합
    participants.md               # 참여 에이전트 명단
    sub-01-데이터모델.md          # 서브주제 회의록 1
    sub-02-전송채널.md            # 서브주제 회의록 2
    sub-03-실패처리.md
    codex-sessions/               # backend=codex 토론자별 thread_id
      meeting-debater-codex.thread       # 한 줄: UUID
      meeting-debater-game-econ.thread
    codex-io/                     # 라운드별 codex IO 디버깅 보존
      sub-01-round-1/
        input-meeting-debater-codex.txt
        input-meeting-debater-codex.txt.full
        output-meeting-debater-codex.txt
        codex-meeting-debater-codex.jsonl   # 전체 이벤트 스트림
      sub-01-round-2/
        ...
      sub-01-end-agree/
        ...
```

템플릿: `templates/main.md`, `templates/sub.md` 참조.

## 출력 사례 (사용자가 보는 채팅)

라운드 마커 없이 연속 대화 시퀀스로 흐릅니다 (cp = consecutive_pass / N = 활성 수):

```markdown
---
### 🆕 서브주제 1 시작: 데이터 모델
🧑‍⚖️ **진행자**
- 이유: 다른 결정의 기반이 됨
- 활성 토론자: [backend-dev, architect, security] (N=3)
- 발언 순서: [backend-dev, architect, security]
- idea-bank 평가: excluded — 순수 기술 결정 + 명확히 경계된 엔지 문제

**🤖 backend-dev** _(i=0)_:
> notifications 테이블 + user_preferences 분리. RLS는 user_id 기준으로 ...

**🤖 architect** _(i=1)_:
> 동의. 단 payload는 JSONB로 두고 인덱스는 type+created_at 복합 ...

**🤖 security** _(i=2)_:
> RLS 정책에 service_role 우회 차단 명시 필요. 또한 토큰 로테이션 ...

**🤖 backend-dev** _(i=3)_: _(PASS [1/3] — security 안 흡수 충분)_
**🤖 architect** _(i=4)_: _(PASS [2/3] — 추가 쟁점 없음)_
**🤖 security** _(i=5)_: _(PASS [3/3] — 합의 도달)_

### ✅ 서브주제 1 자동 종결: 데이터 모델 (cp=3/3)
**결론** (진행자 작성): notifications 테이블 + user_preferences 분리, RLS 적용 (service_role 우회 차단), payload는 JSONB, 인덱스는 (user_id, type, created_at) 복합. 전원 합의.

---
### 🆕 서브주제 2 시작: 전송 채널
... (NEST 분기 사례)

**🤖 backend-dev** _(i=0)_:
> SMTP 직접 vs SES/SendGrid 추상화 — 운영 비용·신뢰성·디버깅 셋 다 평가 필요
> 🔀 NEST_REQUEST: "이메일 전송 추상화 레이어 선택"

🔀 **NEST 즉시 분기**: "이메일 전송 추상화 레이어 선택" — 요청자 backend-dev 가 자식 서브주제 첫 발언자
(부모 사이클 일시 정지, 자식 종결 후 재개)

  --- 자식 서브주제 (depth 2) ---
  **🤖 backend-dev** _(i=0, depth=2)_: SES Lambda 트리거 ...
  ... (자식 사이클 자동 종결)
  
  ### ✅ 자식 서브주제 자동 종결: 이메일 전송 추상화 (cp=3/3)

--- (부모 재개) ---
**🤖 architect** _(i=1)_: 자식 결론 수용. 추가로 ...
... (이하 동일 패턴)

---

## ⚖️ CEO 단독 결정 요청 — 무한 대화 가드 (speech_count=9 >= 3 × 3)
선택지: A) 즉시 종결 / B) 강제 PASS 사이클 / C) 자식 분기
**🤖 ceo**: CEO_DECISION = A — 핵심 결정 이미 합의, 디테일은 구현 단계로 미룸
→ 라우터: 서브주제 즉시 종결

---

## 🗣 회의 종료

**주제**: 실시간 알림 시스템 설계
**서브주제 3개** (NEST 1개 포함) · 총 발언 18회 (SPEAK 12 + PASS 6)

### 서브주제별 결론
1. **데이터 모델** → 전원 합의
2. **전송 채널** → 단계적 도입 (자식: 이메일 SES 추상화)
3. **실패 처리** → BullMQ 큐 + 지수 백오프 (CEO_DECISION 종결)

### 종합 (진행자)
... (5-10줄) ...

📁 회의록: `./meetings/20260527-115342-실시간알림/main.md`
```

## 어젠다 결정 메타 서브주제 특수 규칙 (2026-05-28 4차 — 자유 등록 중립 규칙)

어젠다 결정 (첫 서브주제 "회의 어젠다 결정") 은 본 콘텐츠를 흡수하지 않도록 다음 프롬프트 추가:

```
=== 어젠다 결정 모드 (중립 규칙 — 자유 등록) ===
당신은 어젠다 (회의에서 다룰 서브주제 목록) 를 정하는 메타 단계에 있다.

규칙 (중립적):
1. **개수 제한 없음** — 본인 도메인 핵심 어젠다를 자유롭게 등록할 수 있다. 1개·다수·0개(PASS) 모두 가능. 본인 판단으로 결정 (강제 등록 의무 없음, 강제 PASS 의무 없음).
2. SPEAK 응답의 CONTENT 본문에 어젠다 1개 (5줄 이내 요약). 추가 어젠다는 NEST_REQUEST 블록(들)로 등록.
3. **다중 NEST_REQUEST 허용** (어젠다 메타 한정) — 한 응답에 NEST_REQUEST 블록을 여러 개 적어도 라우터가 어젠다 리스트에 모두 추가한다. 본 토론 서브주제(sub-02 이후)에서는 NEST_REQUEST 다중 시 첫 번째만 처리 (기존 정책 유지).
4. **NEST_REQUEST = 어젠다 항목 추가 요청** (즉시 분기 X). 본 토론은 후속 서브주제에서 진행됨.
5. **PASS 조건 (중립)** — 다음 중 하나라도 해당하면 PASS:
   - 추가 등록할 어젠다가 없음
   - 다른 토론자가 본인 영역을 이미 충분히 커버함
   - 본인 도메인이 이번 큰 주제와 관련성이 낮음
   - 사이클을 계속할 명분이 없음
6. 본인 어젠다가 다른 토론자에 의해 심각히 왜곡되면 SPEAK 로 교정 가능.
7. 본 콘텐츠 (수치·구체 메카닉·실행 디테일) 는 본 토론 진입 후 다룸 — 어젠다 메타에서는 5줄 요약만.
```

자연스럽게 등록 욕구 소진 시점에 연속 PASS 가 누적되어 자동 종결 (cp == active). 1 사이클 강제 등록 없음 — 첫 차례에서도 본인이 등록할 이유가 없다고 판단하면 PASS 가능.

진행자가 어젠다 결정 종결 후 작성하는 FINAL_CONCLUSION 은 "확정된 어젠다 누적 리스트 + 본 토론 우선순위" 형식.

### 어젠다 리스트 영속화 (어젠다 메타 종결 후)

라우터는 어젠다 메타 종결 즉시 `${MEETING_DIR}/agenda.md` 파일에 어젠다 리스트 초기화:

```markdown
# 어젠다 누적 리스트

## ⏳ 처리 대기
- [ ] (1) {등록자 1 의 어젠다} (등록자: {agent1}, sub-01 SPEAK)
- [ ] (2) {등록자 2 의 어젠다} (등록자: {agent2}, sub-01 SPEAK)
- [ ] (3) {NEST 변환 어젠다} (등록자: {agent3}, sub-01 NEST_REQUEST)
...

## ✅ 처리 완료
(어젠다 메타 자체는 처리 완료로 표시 안 함 — 본 토론 어젠다만 표시)
```

본 토론 진행 중 ADD_AGENDA 로 추가되는 항목도 ⏳ 처리 대기 섹션에 append. 어젠다 처리 시 ⏳ → ✅ 이동.

## 진행자의 idea-bank 적극 검토 규칙 (2026-05-28)

진행자는 **모든 서브주제 시작 시 (3단계 RECOMMENDED_AGENTS) 와 풀 재평가 시 (6단계 AGENTS_ADDED/REMOVED)** 에 `meeting-idea-bank` 참여 여부를 **명시적으로 평가** 해야 한다. 수동적 누락 금지.

### 포함 권장 신호 (1개 이상 충족 시 idea-bank 포함)
1. **참신성·혁신 비중**: 큰 주제·서브주제에 "새로운", "차별화", "기존 X 와 다른" 같은 키워드·의도
2. **컨벤셔널 수렴 위험**: 도메인 전문가만 모이면 업계 표준안으로 빠르게 수렴할 가능성 (예: MMORPG 코어루프 = 또 던전·일퀘·레이드)
3. **장르·메카닉 융합**: 둘 이상의 게임 장르·시스템을 결합하는 결정 (예: MMORPG + 로그라이크)
4. **메카닉 ↔ 세계관 연동**: 루도내러티브 설계
5. **규제·플랫폼 제약 우회**: 한국 확률공시·EU DFA·중국 판호 등 제약 안에서 창의적 BM 설계

### 제외 권장 신호 (3가지 모두 충족 시 idea-bank 제외 가능)
1. 순수 기술적 결정 (Netcode tick rate, DB 스키마, ACID 보장 등)
2. 명확히 경계된 엔지니어링 문제 (성능 최적화, 빌드 파이프라인 등)
3. 데이터 분석·KPI 산출 같은 정량 작업

### 응답 형식 추가 (진행자 3·6단계 응답에 필수)

```
IDEA_BANK_EVAL: included | excluded
IDEA_BANK_REASON: <포함/제외 사유 1-2줄. 평가 기준 인용 권장>
```

### 라우터 가드
- `IDEA_BANK_EVAL` 필드 누락 → 1회 재시도 강제. 재시도도 누락 시 라우터가 자동으로 `included` 강제 (적극 default)
- `IDEA_BANK_EVAL: excluded` 인데 사유에 제외 신호 3가지 모두 명시 안 됨 → 1회 재시도. 그래도 모호하면 `included` 강제

## Red Flags — 메인 라우터의 자동 처리 신호

⛔ **회의 분량 임의 중단 폐지** (사용자 정책 4·5). 토큰 과다 무시 + 진행자 회의 중단 금지 + 어젠다 완주가 유일 종결 조건.
⛔ **사용자 질문 금지** ([[meeting-no-user-questions]] 강력 조항). 모든 예외 상황은 자동 fallback 으로 처리.

- 한 서브주제 발언 수 > 3 × 활성 → CEO_DECISION (무한 대화 가드, 4-4). 회의는 계속
- 일반 본 토론에서 NEST cap overflow (nest_count >= 10) → CEO_DECISION (4-5). 분기 강행 또는 어젠다 흡수, 회의 종결 옵션 없음
- codex 호출 실패 (권한·CLI·런타임 · 연속 3회 실패 포함) → **그 에이전트만 claude 로 자동 fallback + 채팅 1회 안내**. 회의 중단 안 함
- 필요 페르소나 후보 풀 부재 → 진행자가 가장 비슷한 expertise 후보로 자동 대체 + 결론에 "⚠ 미커버 도메인" 자동 메모
- 진행자가 어젠다 리스트에 무한 항목 추가 → 그대로 진행 (토큰 비용 무시 정책)
- meeting-ceo.md 또는 활성 풀이 0명 시에만 자동 중단 (채팅에 사유 출력 후 종료, 사용자 질문 X)

## 호출 예시

```
User: /meeting 실시간 알림 시스템 설계
Main: (디렉터리 생성 → 진행자 호출 → ... → 출력)
User: (요약만 봄, 필요 시 main.md 열람)
```

## Related

- 진행자 정의: `~/.claude/agents/meeting-facilitator.md` (풀에서 항상 제외)
- 토론자 후보 풀: `~/.claude/agents/meeting-*.md` (단 facilitator 및 `role: facilitator` / `participant: false` 제외). 매 회의 시작 시 동적 글로빙.
- 기본 제공 토론자 (frontmatter `backend` = default 추천값. 진행자가 BACKEND_SELECTION 오버라이드 가능):
  - `meeting-debater-claude.md` (default_backend=claude, sonnet, 균형·원칙·거버넌스)
  - `meeting-debater-codex.md` (default_backend=codex, gpt-5, 시스템 설계·메커니즘)
- 공통 응답 형식 (모든 백엔드 공통): `~/.claude/skills/meeting/roles/common-debater.md`
- 신규 에이전트 생성 스킬: `/meeting-agent <자연어>` — 자동으로 페르소나·전문성 정의 후 `meeting-*.md` 파일 생성 (3회 딥리서치 포함)
- 에이전트 템플릿: `~/.claude/skills/meeting/templates/agent.md`
- 슬래시 커맨드: `~/.claude/commands/meeting.md` (회의 진입), `~/.claude/commands/meeting-agent.md` (에이전트 생성)
- 회의 디렉터리 템플릿: `templates/main.md`, `templates/sub.md`
- 회의별 참여자 기록: `<meeting_dir>/participants.md`
