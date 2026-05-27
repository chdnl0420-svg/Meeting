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
    ├─ main.md 초기화
    │
    ├─ [SUBTOPIC LOOP]
    │   ├─ 진행자 호출 → 다음 서브주제 + 토론자 순서 결정
    │   ├─ sub-NN-*.md 초기화
    │   │
    │   ├─ [ROUND LOOP]
    │   │   ├─ 라운드로빈 순서대로 토론자 호출
    │   │   │   → 발언 or 패스
    │   │   ├─ 회의록 append
    │   │   ├─ 진행자 호출 → CONTINUE / END_SUBTOPIC
    │   │   └─ 발전 없음 3라운드 시 강제 END_SUBTOPIC
    │   │
    │   ├─ 진행자 → 서브주제 결론 작성
    │   ├─ main.md 인덱스 추가
    │   └─ 다음 서브주제 있는가? → 진행자 판단
    │
    ├─ 진행자 → "회의 종료 제안" 시
    │   └─ 모든 토론자에게 종료 동의 질문 → 전원 YES 시만 종료
    │
    └─ 최종 출력: 채팅에 종합 요약 + 회의록 경로

⚡ 모든 에이전트 응답(진행자 결정, 토론자 발언, 패스, 진행자 판정)은
   받는 즉시 채팅에 라이브 스트리밍으로 출력됨 (사용자가 진행을 실시간 관전)
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

**② 매 라운드 시작 시:**
```markdown

#### 📍 라운드 {round}
```

**③ 토론자가 발언 시 (SPEAK: YES):**
```markdown
**🤖 {agent}**:
> {발언 내용}
```

**④ 토론자가 패스 시 (SPEAK: NO):**
```markdown
**🤖 {agent}**: _(패스 — {reason})_
```

**⑤ 진행자가 라운드 판정 시:**
```markdown
🧑‍⚖️ **진행자 판정**: `{CONTINUE | END_SUBTOPIC}` — {reason}
```

**⑥ 서브주제 종결 동의 (4-4):**
```markdown
🧑‍⚖️ **진행자**: 이 서브주제 종결 제안 — {reason}

**🤖 claude**: 동의 / 비동의 — {reason}
**🤖 codex**: 동의 / 비동의 — {reason}

(비동의 시) → 추가 논점: "{ADDITIONAL_POINT}" — 라운드 1회 더 진행
```

**⑥-b 서브주제 종결 확정 시 (전원 동의 후):**
```markdown

### ✅ 서브주제 {N} 종결: {title}
**결론**: {conclusion (3-5줄)}
```

**⑦ 회의 종료 제안 / 동의 단계:**
```markdown
🧑‍⚖️ **진행자**: 회의 종료를 제안합니다 — {reason}

**🤖 claude**: 동의 / 비동의 — {reason}
**🤖 codex**: 동의 / 비동의 — {reason}
```

**⑧ 최종 종합은 9단계 형식대로** (아래 참조).

### 정책 원칙

- **풀 텍스트 출력**: 토론자 발언은 요약하지 말고 받은 그대로 출력 (5줄 이내라 부담 없음)
- **메인 라우터의 사고/계획 출력 금지**: 사용자에게 보이는 건 "회의 자체"만. 라우터의 내부 작업(파일 저장, 카운터 관리)은 채팅에 노출 안 함
- **실패도 출력**: codex CLI 실패, 형식 오류, 강제 종결 — 모두 채팅에 표시 (`⚠️` 마커)
- **구분선 일관성**: 서브주제 사이는 `---`, 라운드 사이는 빈 줄

## Workflow (메인 라우터가 따를 단계)

### 0. 권한·CLI 사전 검사 (Codex 포함 시 필수)

토론자 풀에 `codex`가 포함된 경우, 회의 시작 전 다음 두 가지를 검사합니다.

#### 0-1. settings.json 권한 검사

```bash
grep -E '"Bash\(codex exec' ~/.claude/settings.json || echo "MISSING"
```

`MISSING` 이면 **AskUserQuestion 도구**로 사용자에게 승인 요청:

```
질문: "Codex 토론자를 사용하려면 settings.json 에 'Bash(codex exec:*)' 권한이 필요합니다. 어떻게 진행할까요?"
헤더: "Codex 권한"
옵션:
  1. "내가 직접 settings.json 에 추가하고 새 세션에서 재시도 (권장)"
     설명: 메인 라우터는 보안상 settings.json 을 자가 편집할 수 없음 (자동 모드 분류기 차단). 사용자가 직접 추가 후 재호출.
  2. "Claude(만)로 회의 진행 — Codex 제외"
     설명: 토론자 풀에서 codex 제거하고 claude만으로 회의 진행. 토론 다양성 ↓ 하지만 즉시 진행 가능.
  3. "회의 중단"
     설명: 권한 설정 없이는 codex 호출 시점에서 어차피 차단되므로 미리 중단.
```

- **옵션 1 선택**: 사용자에게 정확한 JSON 패치 제시 후 회의 중단:
  ```diff
  "permissions": {
    "allow": [
      ...,
  +    "Bash(codex exec:*)"
    ]
  }
  ```
- **옵션 2 선택**: 토론자 풀 = `["claude"]` 로 축소 후 회의 진행 (1인 토론은 의미 약함 → 진행자가 "쟁점 정리·자기 반박" 모드로 변경)
- **옵션 3 선택**: 회의 중단, 채팅에 사유 출력

#### 0-2. codex CLI 사전 점검

```bash
codex --version
```

- exit 0 → 진행
- 그 외 → 옵션 2(Claude 단독) 또는 옵션 3(중단)을 사용자에게 다시 묻기

권한·CLI 모두 OK 면 출력 없이 1단계로 진행.

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

#### 2-1. 사용자가 /meeting 인자로 토론자 명시 지정한 경우

`/meeting <주제> --agents claude,codex,game-econ` 형태로 명시되면 진행자 추천 건너뛰고 그 풀 사용 (이름은 `meeting-debater-` 접두 생략 허용 — 메인 라우터가 정규화).

#### 2-2. 명시 지정 없으면 진행자 추천 흐름 (3단계에서)

진행자에게 후보 메타 테이블 전달 → 진행자가 회의 주제 분석 후 `RECOMMENDED_AGENTS: [name1, name2, ...]` 응답. 보통 2~4명. 메인 라우터가 그 추천을 토론자 풀로 확정.

#### 2-3. 후보가 1명 이하면

- 0명: 회의 진행 불가. AskUserQuestion 으로 사용자 통보 + `/meeting-agent` 사용 권유 후 중단.
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
2. 후보 풀에서 이번 회의에 가장 적합한 토론자 2~4명 추천 (RECOMMENDED_AGENTS).
3. 첫 서브주제와 발언 순서 결정.

응답 형식 (반드시):

DECISION: START_SUBTOPIC
RECOMMENDED_AGENTS: [name1, name2, ...]
RECOMMENDED_REASON: <왜 이 조합을 골랐는지, 1-2줄>
SUBTOPIC_TITLE: <한국어 30자 이내>
SUBTOPIC_CONTEXT: <왜 이 서브주제부터, 1-2줄>
SPEAKING_ORDER: [name1, name2, ...] (위 추천 풀의 부분집합, 라운드 순서)
```

입력 (사용자 명시 지정 있을 때): 위 동일하되 `=== 토론자 풀 (사용자 명시) ===` 섹션으로 변경, `RECOMMENDED_AGENTS` 응답은 생략 가능 (있어도 무시).

진행자 응답 파싱:
- `RECOMMENDED_AGENTS` 로 풀 확정 → `participants.md` 작성 (2-4)
- 서브주제 회의록 파일 생성 (`sub-01-{slug}.md`)

### 4. 라운드 루프 (서브주제 내부)

라운드 카운터 `round=1`, 패스 카운터 `pass_streak=0`.

매 라운드:

#### 4-1. 토론자 순환 호출

`SPEAKING_ORDER` 순서대로 각 토론자를 호출. 호출 방식은 그 에이전트 frontmatter 의 `backend` 필드로 결정 — 자세한 절차는 아래 "토론자 호출 디테일 (backend 분기)" 섹션 참조.

| backend | 호출 방식 |
|---------|-----------|
| `claude` | Task 도구로 해당 서브에이전트 호출 (subagent_type=에이전트 name) |
| `codex` | 메인 라우터가 Bash 로 `codex exec` 호출. **첫 호출 = `codex exec` + 풀 부트스트랩 (thread_id 캡처)**, **2회차 이후 = `codex exec resume <thread_id>` + 델타 컨텍스트만**. 세션 컨텍스트가 보존되어 페르소나/큰주제/이전 발언을 매번 재첨부할 필요 없음 |
| `gemini` | (향후) 메인 라우터가 Bash 로 `gemini` CLI 직접 호출 |
| `ollama` | (향후) 메인 라우터가 Bash 로 `ollama run` 직접 호출 |

공통 컨텍스트 패키지 (양쪽 모두 동일):

```
=== 큰 주제 ===
{topic}

=== 진행된 서브주제 결론 ===
1. {sub1_title} → {sub1_conclusion}
2. ...
(없으면 "없음")

=== 현재 서브주제: {current_title} ===
{current_subtopic_md 전체 내용}

=== 당신의 차례 ===
당신: {agent_name}
직전 발언자: {prev_speaker} (라운드 첫 발언이면 "없음")

이 서브주제에서 새로 할 말이 있습니까?
- 새 근거·반박·관점이 있으면 발언
- 같은 논점 반복이면 패스 (SPEAK: NO)

응답 형식:
SPEAK: YES
CONTENT:
<발언 내용 (한국어, 핵심만 5줄 이내)>

또는:

SPEAK: NO
REASON: <짧게>
```

각 응답을 서브주제 회의록에 append:

```markdown
## 라운드 {round}

**{agent}**: (SPEAK=YES 시 발언 / NO 시 "(패스: {reason})")
```

#### 4-2. 패스 카운터 관리

- 라운드 내 **모든 토론자가 SPEAK: NO** → `pass_streak += 1`
- 하나라도 발언 → `pass_streak = 0`

#### 4-3. 진행자 판정 호출

라운드 끝나면 진행자 재호출. 입력:

```
=== 현재 서브주제: {current_title} ===
{서브주제 회의록 전체}

=== 라운드 {round} 종료 ===
이번 라운드 발언 요약: {round 마지막 발언들}
패스 연속: {pass_streak}회

=== 당신의 임무 ===
이 서브주제를 종결할지, 계속할지 결정.

응답 형식:
DECISION: CONTINUE | END_SUBTOPIC
REASON: <짧게>
(END_SUBTOPIC 시) CONCLUSION:
<서브주제 결론 (한국어, 두 의견 병기 + 종합, 10줄 이내)>
```

#### 4-3-b. 강제 종결 가드 (동의 절차 우선 적용 예외)

`pass_streak >= 3` 이면 진행자 판정 무시하고 **강제 END_SUBTOPIC**.
진행자에게 "강제 종결입니다. CONCLUSION 만 작성하세요" 추가 요청.
**강제 종결 시에는 4-4 동의 절차를 건너뛰고 바로 5단계로 진행** (토론자들이 패스 연속이라 추가 발화 의사가 없음).

#### 4-4. 서브주제 종결 동의 (전원 합의 필요)

진행자가 `END_SUBTOPIC` 결정을 내려도, **각 토론자에게 종결 동의를 묻고 전원 YES** 일 때만 5단계로 진행한다. 한 명이라도 NO 면 그 토론자가 제시한 `ADDITIONAL_POINT` 로 한 라운드 더 진행 후 다시 진행자 판정으로 돌아간다.

##### 4-4-1. 각 토론자에게 종결 동의 질문

각 토론자(claude / codex) 에게 다음 컨텍스트 패키지로 호출:

```
=== 큰 주제 ===
{topic}

=== 진행된 서브주제 결론 (이전) ===
1. ...
(없으면 "없음")

=== 현재 서브주제: {current_title} ===
{서브주제 회의록 전체}

=== 진행자가 이 서브주제 종결을 제안했습니다 ===
사유: {facilitator_reason}
잠정 결론: {facilitator_conclusion}

=== 당신의 응답 ===
응답 형식:
SUBTOPIC_END_AGREE: YES
REASON: <한 줄>

또는:

SUBTOPIC_END_AGREE: NO
REASON: <한 줄>
ADDITIONAL_POINT: <이 서브주제에서 더 다루고 싶은 구체 논점, 30자 이내>
```

호출 방식: claude 는 `meeting-debater-claude` 서브에이전트, codex 는 메인 라우터 직접 호출 (파일 IO 패턴, 모드 SUBTOPIC_END_AGREE).

##### 4-4-2. 결과 분기

- **전원 YES** → 5단계 진행
- **한 명이라도 NO** → 그 토론자의 `ADDITIONAL_POINT` 를 채팅에 출력 + 서브주제 회의록에 명시 → 새 라운드 1회 더 진행 (그 ADDITIONAL_POINT 를 라운드 컨텍스트에 명시) → 다시 4-3 진행자 판정으로 복귀
- **같은 토론자가 같은 ADDITIONAL_POINT 를 2회 연속 제시** → 그 추가 라운드 무시하고 5단계 진행 (무한 거부 방지)
- **강제 종결 케이스** (4-3의 `pass_streak >= 3`) → 동의 절차 생략, 바로 5단계 (강제 종결은 토론자도 패스 연속이라 동의 의사 없음으로 간주)

##### 4-4-3. 채팅 출력 형식

```markdown
🧑‍⚖️ **진행자**: 이 서브주제 종결 제안 — {reason}

**🤖 claude**: 동의 / 비동의 — {reason}
**🤖 codex**: 동의 / 비동의 — {reason}

(전원 동의 시) → 다음 단계로
(비동의 발생 시) → 추가 논점: "{ADDITIONAL_POINT}" — 라운드 1회 더 진행
```

### 5. 서브주제 종결 처리

`END_SUBTOPIC` 받으면:

1. 서브주제 회의록에 `## 결론 (진행자)` 섹션 append
2. `main.md` 의 `## 서브주제 결론` 에 항목 추가:

```markdown
### {N}. {sub_title}
- 결론: {conclusion 요약 한 줄}
- 상세: [./sub-NN-*.md](./sub-NN-*.md)
```

### 6. 다음 서브주제 결정 + 토론자 풀 재평가 — 진행자 호출

매 새 서브주제마다 **진행자에게 후보 풀 전체 + 현재 활성 풀** 을 함께 전달해 토론자 추가/제거를 판단하게 한다.

```
=== 큰 주제 ===
{topic}

=== 토론자 후보 풀 (전체) ===
| name | backend | persona | expertise |
| meeting-debater-claude | claude | 균형·원칙·거버넌스 | 거버넌스 |
| meeting-debater-codex | codex | 시스템 설계 | 메커니즘 |
| meeting-debater-game-econ | claude | 게임 경제 디자이너 | 게임 경제, MMO |
| ... (참여자 미선택 후보 포함 전체) |

=== 현재 활성 토론자 ===
[meeting-debater-claude, meeting-debater-codex]

=== 진행된 서브주제 결론 ===
1. {sub1_title} → {sub1_conclusion}
2. ...

=== 당신의 임무 ===
1. 다음 서브주제가 있는지 판단.
2. 있으면, 그 서브주제에 현재 활성 토론자가 적합한지 분석.
   - 부적합·중복 토론자는 AGENTS_REMOVED
   - 후보 풀에서 더 적합한 토론자는 AGENTS_ADDED
   - 모두 적합하면 변경 없이 진행
3. ACTIVE_AGENTS + SPEAKING_ORDER 확정.

응답 형식 (택1):

DECISION: START_SUBTOPIC
SUBTOPIC_TITLE: <제목>
SUBTOPIC_CONTEXT: <이유>
ACTIVE_AGENTS: [name1, name2, ...]
AGENTS_ADDED: [...]                (없으면 [])
AGENTS_REMOVED: [...]              (없으면 [])
AGENTS_CHANGE_REASON: <왜 추가/제거했는지. 변경 없으면 "변경 없음">
SPEAKING_ORDER: [...]              (ACTIVE_AGENTS 순열)

또는:

DECISION: PROPOSE_END_MEETING
REASON: <짧게>
```

#### 6-1. 응답 검증 (라우터)

진행자 응답 파싱 후 다음 가드를 통과해야 5단계 다음 라운드로 진입:

| 조건 | 처리 |
|------|------|
| `ACTIVE_AGENTS` 비어있음 | 1회 재시도. 다시 실패 시 PROPOSE_END_MEETING 으로 강등 |
| `ACTIVE_AGENTS` ≠ (현재 활성 ∪ ADDED) - REMOVED | 1회 재시도 |
| `AGENTS_ADDED` 항목이 후보 풀에 없는 이름 | 그 이름만 무시하고 진행 |
| `AGENTS_REMOVED` 항목이 현재 활성 풀에 없는 이름 | 그 이름만 무시하고 진행 |
| `SPEAKING_ORDER` 집합 ≠ `ACTIVE_AGENTS` | 라우터가 `ACTIVE_AGENTS` 순서로 자동 보정 |
| `ACTIVE_AGENTS` 가 1명 이하 | 1회 재시도. 다시 실패 시 진행자에게 "최소 2명 유지" 강제 후 재시도. 그래도 1명이면 그대로 진행하되 자기반박 모드 안내 |

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
- 4-4 서브주제 종결 동의 / 7 회의 종료 동의에서도 활성 풀만 질문 (제거된 토론자는 동의 대상 아님)

### 7. 회의 종료 동의 (전원 합의 필요)

`PROPOSE_END_MEETING` 받으면 **모든 토론자에게 종료 동의 질문**:

```
=== 큰 주제 ===
{topic}

=== 진행된 서브주제 결론 ===
1. {sub1} → {conclusion1}
...

=== 진행자가 회의 종료를 제안했습니다 ===
사유: {facilitator_reason}

=== 당신의 응답 ===
END_AGREE: YES | NO
REASON: <짧게>
(END_AGREE: NO 시) ADDITIONAL_SUBTOPIC: <다루고 싶은 서브주제>
```

- 전원 YES → 진행 (8단계)
- 한 명이라도 NO → 그 토론자가 제시한 `ADDITIONAL_SUBTOPIC` 으로 새 서브주제 시작 (4단계 SPEAKING_ORDER 는 진행자에게 다시 결정 요청)

### 8. 최종 종합 — 진행자 호출

```
=== 큰 주제 ===
{topic}

=== 진행된 서브주제 결론 ===
... (전체)

=== 당신의 임무 ===
회의 전체 종합. 두 의견 병기 + 사용자에게 어떤 결정을 권하는지(권고는 선택).

응답 형식:
FINAL_SUMMARY:
<5-10줄 한국어 종합>
```

`main.md` 의 `## 종합 (진행자)` 섹션에 기록 + 종료 시간 기록.

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

이미 모든 발언이 위쪽 채팅에 라이브로 출력되어 있으므로, 9단계는 **종합과 인덱스만** 압축해서 마무리.

## 안전 가드

| 상황 | 가드 |
|------|------|
| codex 권한 차단 (settings.json 에 `Bash(codex exec:*)` 없음) | 0-1 단계 사전 검사 + AskUserQuestion 실행. 라우터는 settings.json 을 자가 편집할 수 없으므로 사용자가 직접 처리 |
| codex CLI 부재·로그인 안 됨 | 0-2 단계 사전 검사. exit ≠ 0 이면 Claude 단독 모드 또는 회의 중단 선택지 제공 |
| 진행자가 무한 서브주제 생성 | 큰주제 서브주제 수 10개 초과 시 메인 라우터가 PROPOSE_END_MEETING 강제 |
| codex exec 호출 런타임 실패 | exit 2 (로그인) → 회의 중단 + 사용자 안내 / exit 124 (timeout) → 1회 재시도 / 기타 → 그 토론자 라운드 SPEAK: NO 처리 |
| codex exec 가 연속 3회 실패 | 회의 중단 + 사용자 보고 |
| `${MEETING_DIR}/codex-sessions/<agent>.thread` 파일 존재 but resume 호출이 "Session not found" 등으로 실패 | 그 thread 파일 삭제 → 같은 차례 bootstrap 으로 폴백 1회 (페르소나·큰주제·서브주제 다시 박아넣기). 폴백도 실패 시 회의 중단 |
| thread_id 캡처 실패 (bootstrap JSONL 첫 줄에 thread_id 없음) | 1회 재시도. 다시 실패 시 다음 라운드부터 그 토론자는 매번 bootstrap (resume 비활성). 토큰 낭비 발생하므로 사용자에게 경고 출력 |
| 토론자 응답이 형식 어김 | 1회 재시도. 다시 실패 시 그대로 회의록에 raw 기록 + 패스 처리 |
| 진행자 응답이 형식 어김 | 1회 재시도. 실패 시 메인 라우터가 END_SUBTOPIC 강제 |
| 종료 동의에서 무한 NO | 한 토론자가 같은 ADDITIONAL_SUBTOPIC 2회 연속 제시 시 그 제안 무시하고 다시 종료 시도 |
| 서브주제 종결 동의에서 무한 NO | 한 토론자가 같은 ADDITIONAL_POINT 2회 연속 제시 시 추가 라운드 무시하고 5단계 진행 |
| ⚠ deprecated 서브에이전트 사용 시도 | `meeting-debater-codex` 는 더 이상 호출하지 않음 — Task 도구로 그 이름을 부르지 말 것. codex 차례는 항상 메인 라우터 Bash 직접 호출 |
| 진행자가 ACTIVE_AGENTS 를 빈 배열로 반환 | 1회 재시도. 다시 실패 시 PROPOSE_END_MEETING 으로 강등 (토론자 없으면 회의 진행 불가) |
| AGENTS_ADDED 가 후보 풀에 없는 이름 포함 | 그 이름만 제거하고 진행 (조용히 보정). 모두 무효면 변경 무시하고 현재 풀 유지 |
| ACTIVE_AGENTS 가 1명만 남음 | 진행자에게 "최소 2명 유지" 강제 후 1회 재시도. 그래도 1명이면 자기반박 모드 안내 후 진행 |
| AGENTS_REMOVED 된 토론자가 다음 서브주제에서 재합류 | `.thread` 파일이 남아있으면 codex resume + "재합류" 델타 안내. claude 는 매번 풀 컨텍스트라 별도 처리 불필요 |

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

응답은 stdout 직접 반환 (output 파일 미사용). 메인 라우터가 `SPEAK:` / `SUBTOPIC_END_AGREE:` / `END_AGREE:` 키 파싱.

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

`SUBTOPIC_END_AGREE` / `END_AGREE` 단계는 라운드 디렉터리 대신 `sub-${SUB_NN}-end-agree[-${RETRY}]` 사용.

**2-A. MODE=bootstrap (첫 호출) 일 때 INPUT 작성**

```
=== 역할 정의 ===
당신은 /meeting 스킬의 토론자다. 다음 두 파일을 반드시 먼저 읽고 그 정의대로 행동하라:

ROLE_FILE_COMMON: C:\Users\NX3GAMES\.claude\skills\meeting\roles\common-debater.md
PERSONA_FILE: <에이전트 정의 파일 절대 경로, 예: C:\Users\NX3GAMES\.claude\agents\meeting-debater-codex.md>

ROLE_FILE_COMMON 은 공통 응답 형식 (SPEAK_OR_PASS / SUBTOPIC_END_AGREE / END_AGREE).
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
| 같은 서브주제 다음 라운드 | `직전 발언자: {prev}` + 직전 발언 본문 (5줄) + `진행자 판정: {CONTINUE/END_SUBTOPIC} — {reason}` + `이번 라운드 모드: SPEAK_OR_PASS` |
| 서브주제 전환 첫 라운드 | 이전 서브주제 결론 1줄 + `=== 새 서브주제: {title} ===` + `SUBTOPIC_CONTEXT: {ctx}` + `SPEAKING_ORDER: [...]` + `당신 차례` + `모드: SPEAK_OR_PASS` |
| SUBTOPIC_END_AGREE | `=== 진행자 종결 제안 ===` + `사유: {reason}` + `잠정 결론: {conclusion}` + `모드: SUBTOPIC_END_AGREE` |
| END_AGREE | `=== 진행자 회의 종료 제안 ===` + `사유: {reason}` + 진행된 서브주제 결론 인덱스 + `모드: END_AGREE` |
| 비동의 후 추가 라운드 | `이전 라운드 ADDITIONAL_POINT: {point}` + `이번 라운드 모드: SPEAK_OR_PASS — 이 추가 논점에 집중` |

이전 발언/큰주제/페르소나는 절대 재첨부하지 않는다. codex 세션이 기억한다.

**3. codex exec 호출**

MODE 에 따라 명령이 분기되지만 옵션은 동일.

**3-A. MODE=bootstrap (첫 호출):**

```bash
{ cat "$INPUT"; echo; echo "다음 절차로 응답:"; echo "1. 위 컨텍스트의 ROLE_FILE_COMMON·PERSONA_FILE 두 파일을 먼저 읽어라."; echo "2. 모드 지시에 따라 ROLE_FILE_COMMON 의 응답 형식대로 응답 본문을 작성."; echo "3. 응답을 파일에 덮어쓰기 저장: $OUTPUT"; echo "4. 'DONE' 한 단어만 stdout 에 출력하고 종료."; } > "${INPUT}.full"

timeout 300 codex exec --json --skip-git-repo-check \
  -s workspace-write \
  -C "$MEETING_DIR" \
  -c model_reasoning_effort=medium \
  - < "${INPUT}.full" 2>&1 | tee "${IO_DIR}/codex-${AGENT_NAME}.jsonl" | tail -5
EXIT=$?

# thread_id 캡처 (JSONL 첫 줄)
THREAD_ID=$(head -1 "${IO_DIR}/codex-${AGENT_NAME}.jsonl" | grep -oP '"thread_id":"\K[^"]+')
if [ -n "$THREAD_ID" ]; then
  echo "$THREAD_ID" > "$THREAD_FILE"
fi
```

**3-B. MODE=resume (2회차 이후):**

```bash
{ cat "$INPUT"; echo; echo "다음 절차로 응답:"; echo "1. 모드 지시에 따라 ROLE_FILE_COMMON 의 응답 형식대로 응답 본문을 작성. (이미 세션에 로드돼 있음)"; echo "2. 응답을 파일에 덮어쓰기 저장: $OUTPUT"; echo "3. 'DONE' 한 단어만 stdout 에 출력하고 종료."; } > "${INPUT}.full"

timeout 300 codex exec resume --json --skip-git-repo-check \
  -s workspace-write \
  -C "$MEETING_DIR" \
  -c model_reasoning_effort=medium \
  "$THREAD_ID" \
  - < "${INPUT}.full" 2>&1 | tee "${IO_DIR}/codex-${AGENT_NAME}.jsonl" | tail -5
EXIT=$?
```

옵션 메모:
- `--json` : 첫 줄 `thread.started` 이벤트에서 thread_id 캡처용 (bootstrap) / resume 시에도 유지하면 디버깅 용이
- `-s workspace-write` + `-C "$MEETING_DIR"` : $OUTPUT 파일 쓰기 권한, 작업 디렉터리 한정 (기존과 동일)
- `-c model_reasoning_effort=medium` : low 는 페르소나 무시 위험, high 는 과사고. medium 권장
- `- < "${INPUT}.full"` : stdin 으로 프롬프트 전달 (인자 길이 제한 회피, `</dev/null` 불필요 — stdin 이 명시적 EOF 로 닫힘)
- `timeout 300` : 5분 hard cap
- `tee "${IO_DIR}/codex-${AGENT_NAME}.jsonl"` : 전체 이벤트 스트림을 디버깅용으로 영구 보존

**4. exit code + 결과 파일 검증**

| 조건 | 처리 |
|------|------|
| EXIT=0 AND $OUTPUT 존재 AND 내용이 `SPEAK:` / `SUBTOPIC_END_AGREE:` / `END_AGREE:` 로 시작 | 정상. $OUTPUT 그대로 사용 → 5단계 |
| EXIT=0 BUT $OUTPUT 없음/형식 어김 | 1회 재시도 (같은 MODE 로). 다시 실패 시 회의록에 raw 기록 + SPEAK: NO 처리 |
| EXIT=2 | 로그인 필요. 회의 중단 + "별도 터미널에서 `codex login` 후 재시도" 안내 |
| EXIT=124 | timeout. 1회 재시도. 다시 timeout 이면 SPEAK: NO 처리 |
| MODE=resume AND `Session not found` 류 stderr | thread 파일 삭제 후 bootstrap 으로 1회 폴백 (페르소나·큰주제 다시 박아넣기). 폴백도 실패 시 회의 중단 |
| 기타 | stdout 마지막 줄을 회의록에 raw 기록 + SPEAK: NO 처리. 연속 3회 발생 시 회의 중단 |

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

회의 중에는 라이브로 다음과 같이 흘러갑니다:

```markdown
---
### 🆕 서브주제 1 시작: 데이터 모델
🧑‍⚖️ **진행자**
- 이유: 다른 결정의 기반이 됨
- 발언 순서: [claude, codex]

#### 📍 라운드 1
**🤖 claude**:
> notifications 테이블 + user_preferences 분리. RLS는 user_id 기준으로 ...
**🤖 codex**:
> 동의. 단 payload는 JSONB로 두고 인덱스는 type+created_at 복합 ...

🧑‍⚖️ **진행자 판정**: `CONTINUE` — 인덱스 전략 추가 논의 여지

#### 📍 라운드 2
**🤖 claude**: _(패스 — Codex 인덱스 안에 동의)_
**🤖 codex**: _(패스 — 추가 쟁점 없음)_

🧑‍⚖️ **진행자 판정**: `END_SUBTOPIC` — 합의 도달

### ✅ 서브주제 1 종결: 데이터 모델
**결론**: notifications 테이블 + user_preferences 분리, RLS 적용, payload는 JSONB, 인덱스는 (user_id, type, created_at) 복합. 양측 합의.

---
### 🆕 서브주제 2 시작: 전송 채널
... (이하 동일 패턴) ...

---

## 🗣 회의 종료

**주제**: 실시간 알림 시스템 설계
**서브주제 3개** · 총 11라운드

### 서브주제별 결론
1. **데이터 모델** → 양측 합의
2. **전송 채널** → Claude: 인앱 우선 / Codex: 이메일 병행. 단계적 도입 권고
3. **실패 처리** → BullMQ 큐 + 지수 백오프 (양측 합의)

### 종합 (진행자)
... (5-10줄) ...

📁 회의록: `./meetings/20260527-115342-실시간알림/main.md`
```

## Red Flags — 메인 라우터가 멈춰야 할 신호

- 서브주제 5개 넘었는데 진행자가 계속 새 주제 생성 → 메인 라우터가 사용자에게 "진행 너무 길어짐, 계속할지" 확인
- 토론자 둘 다 5라운드 연속 패스 → 그 서브주제 강제 종결 (안전 가드보다 더 빠른 종결)
- codex CLI 가 연속 3회 실패 → 회의 중단 후 사용자 보고

## 호출 예시

```
User: /meeting 실시간 알림 시스템 설계
Main: (디렉터리 생성 → 진행자 호출 → ... → 출력)
User: (요약만 봄, 필요 시 main.md 열람)
```

## Related

- 진행자 정의: `~/.claude/agents/meeting-facilitator.md` (풀에서 항상 제외)
- 토론자 후보 풀: `~/.claude/agents/meeting-*.md` (단 facilitator 및 `role: facilitator` / `participant: false` 제외). 매 회의 시작 시 동적 글로빙.
- 기본 제공 토론자:
  - `meeting-debater-claude.md` (backend=claude, sonnet, 균형·원칙·거버넌스)
  - `meeting-debater-codex.md` (backend=codex, gpt-5, 시스템 설계·메커니즘)
- 공통 응답 형식 (모든 백엔드 공통): `~/.claude/skills/meeting/roles/common-debater.md`
- 신규 에이전트 생성 스킬: `/meeting-agent <자연어>` — 자동으로 페르소나·전문성 정의 후 `meeting-*.md` 파일 생성 (3회 딥리서치 포함)
- 에이전트 템플릿: `~/.claude/skills/meeting/templates/agent.md`
- 슬래시 커맨드: `~/.claude/commands/meeting.md` (회의 진입), `~/.claude/commands/meeting-agent.md` (에이전트 생성)
- 회의 디렉터리 템플릿: `templates/main.md`, `templates/sub.md`
- 회의별 참여자 기록: `<meeting_dir>/participants.md`
