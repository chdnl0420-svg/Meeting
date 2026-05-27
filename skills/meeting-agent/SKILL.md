---
name: meeting-agent
description: Use when the user invokes `/meeting-agent <자연어>` or says "회의용 에이전트 추가/생성", "토론자 만들어줘", "meeting 에이전트 추가" — 자연어 한 줄로 받은 페르소나·전문성·관점을 3회 딥리서치로 보강한 뒤 `/meeting` 스킬용 토론자 에이전트 파일을 `~/.claude/agents/meeting-<slug>.md` 로 생성.
---

# meeting-agent Skill — 회의 토론자 자동 생성

## Overview

사용자가 자연어 한 줄로 원하는 토론자를 묘사하면, 3회 딥리서치로 페르소나·전문성을 보강한 뒤 `/meeting` 스킬과 호환되는 토론자 에이전트 파일을 생성한다.

## When to Use

- `/meeting-agent <자연어>` 슬래시 커맨드 호출
- 사용자: "MMORPG 경제 디자이너 토론자 추가", "보안 전문가 회의 에이전트 만들어줘", "claude-skeptic 페르소나 추가"

## When NOT to Use

- 일반 에이전트 생성 (회의 전용이 아닌 것) — Claude Code 의 일반 에이전트 작성 방식 사용
- 기존 에이전트 페르소나 수정 — 사용자가 직접 파일 편집 또는 별도 명시 요청

## 사전 조건

- `/meeting` 스킬 (`~/.claude/skills/meeting/`) 존재
- `~/.claude/skills/meeting/templates/agent.md` 존재
- `~/.claude/skills/meeting/roles/common-debater.md` 존재
- WebSearch / WebFetch 도구 사용 권한 (딥리서치용)

## Workflow

### 1. 자연어 입력 파싱

사용자 입력에서 다음을 추출 (필요 시 AskUserQuestion 으로 보완):

| 필드 | 예시 | 도출 방식 |
|------|------|-----------|
| `slug` | `game-econ` | 자연어에서 영문 슬러그화. 한글이면 음역 또는 영어 키워드 |
| `display_name` | "MMORPG 경제 디자이너" | 한국어 표시명 |
| `persona_seed` | "MMO 게임 경제 시스템 디자인 전문가 관점" | 자연어 그대로 |
| `expertise_seeds` | [게임_경제, MMO, 인플레이션] | 키워드 추출 |
| `backend` | `claude` (기본) | 자연어에 모델 명시(예: "GPT-5로", "Codex 백엔드") 없으면 사용자에게 1회 질문하거나 기본값 `claude` |
| `model` | `sonnet` (기본) | backend 가 claude 면 sonnet, codex 면 gpt-5 |

### 2. 중복·충돌 검사

- `~/.claude/agents/meeting-<slug>.md` 이미 존재하면 사용자에게 덮어쓸지 / 다른 slug 쓸지 질문 (AskUserQuestion)
- `meeting-facilitator` 와 같은 예약 이름은 거부

### 3. 3회 딥리서치 (필수)

**목표**: 페르소나·전문성을 실제 도메인 지식·실무 관점으로 보강.

각 회차마다 다음 4 단계:
1. **WebSearch**: persona_seed + expertise 키워드 조합으로 검색 (각 회차마다 다른 각도)
2. **WebFetch**: 상위 결과 2~3개 페치
3. **요약 노트 작성**: `~/.claude/skills/meeting/temp-research/<slug>-<round>.md`
4. **다음 회차 검색어 정제**: 1회차 결과 보고 2회차 검색어 좁힘. 2회차 보고 3회차 좁힘 (cross-source 검증)

회차별 각도 예시:
- 1회차: 도메인 전반 (예: "MMORPG 경제 시스템 디자인 주요 원칙")
- 2회차: 실제 사례·실패 사례 (예: "Diablo 3 경매장 실패 인플레이션 원인")
- 3회차: 최근 동향·논쟁점 (예: "2024 MMO 비P2W 수익 모델 사례")

리서치 실패 (검색 결과 부족, WebFetch 실패) 시 1회 재시도. 다시 실패하면 사용자에게 통보 + 리서치 없이 진행할지 질문.

### 4. 에이전트 정의 합성

`~/.claude/skills/meeting/templates/agent.md` 템플릿 로드 + 다음 필드 채움:

- `name`: `meeting-<slug>`
- `description`: 한 줄 (자연어 시드 + 리서치 핵심 키워드)
- `role: debater`
- `backend`: 1단계 결정값
- `model`: 1단계 결정값
- `expertise`: 리서치로 보강된 5~7개 키워드
- `persona`: 1-2줄 핵심 요약 (페르소나 관점·말투·강조점)
- 본문 페르소나 섹션: 3-5줄 (리서치에서 도출된 도메인 관점)
- 전문성 섹션: 3-5개 도메인 (각 1줄 구체 사례)
- 행동 원칙: 템플릿 기본값 + persona 별 추가 1~2줄 (차별화 포인트)

응답 형식 섹션은 템플릿 그대로 (`common-debater.md` 참조).

### 5. 파일 생성

`~/.claude/agents/meeting-<slug>.md` 에 Write. 기존 파일 있으면 (2단계에서 사용자 동의 받은 경우) 덮어쓰기.

### 6. 사용자 확인 출력

생성된 파일 전체 본문 + 다음 안내:

```markdown
✅ 에이전트 생성 완료: `meeting-<slug>` (backend=<backend>, model=<model>)

**다음 회의부터 후보 풀에 자동 등록됩니다** (`meeting-*.md` 글로빙).

활용:
- `/meeting <주제>` → 진행자가 이 에이전트를 자동 추천할 수 있음
- `/meeting <주제> --agents <slug>,claude,codex` → 명시 지정

검토 권장:
- 페르소나·전문성이 기대와 다르면 `~/.claude/agents/meeting-<slug>.md` 직접 편집
- backend 변경하려면 frontmatter `backend` 필드 수정 + 그에 맞는 model 도 조정

리서치 노트 (참고): `~/.claude/skills/meeting/temp-research/<slug>-*.md`
```

### 7. 리서치 노트 정리

리서치 노트(`temp-research/`) 는 사용자가 검토할 수 있게 영구 보존. 단 30일 이상 된 파일은 다음 호출 시 자동 정리 권장 (cleanup 옵션).

## 호출 예시

```
User: /meeting-agent MMORPG 경제 디자이너 토론자 추가
Main: (자연어 파싱 → 3회 딥리서치 → 합성 → meeting-mmorpg-econ.md 생성 → 출력)

User: /meeting-agent GPT-5로 보안 전문가
Main: (backend=codex 인식 → 보안 도메인 리서치 → meeting-security.md 생성)

User: /meeting-agent claude-skeptic 일관되게 비판하는 페르소나
Main: (페르소나 위주, 리서치는 "devil's advocate 토론 기법" 등으로 → meeting-claude-skeptic.md)
```

## 안전 가드

| 상황 | 가드 |
|------|------|
| 자연어가 너무 모호 | AskUserQuestion 으로 페르소나·도메인·backend 명시 받기 |
| 리서치 결과 부족 | 1회 재시도 → 사용자에게 진행 여부 질문 |
| 슬러그 충돌 | 덮어쓰기 vs 새 슬러그 선택 |
| `meeting-facilitator` 같은 예약 이름 | 거부 + 다른 이름 요청 |
| 외부 backend (gemini, ollama 등) 명시 | CLI 설치·권한 사전 확인. 없으면 사용자 안내 후 backend=claude 권장 |
| 부적절·악의적 페르소나 요청 | 거부 + 사용자 통보 |

## Related

- `/meeting` 스킬: `~/.claude/skills/meeting/SKILL.md`
- 에이전트 템플릿: `~/.claude/skills/meeting/templates/agent.md`
- 공통 응답 형식: `~/.claude/skills/meeting/roles/common-debater.md`
- 슬래시 커맨드: `~/.claude/commands/meeting-agent.md`
- 리서치 노트 보관: `~/.claude/skills/meeting/temp-research/`
