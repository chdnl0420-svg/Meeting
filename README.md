# Meeting

Claude Code용 `/meeting` 스킬과 토론자 에이전트 모음.

## 구성

```
agents/                   # 회의 진행자 + 토론자 에이전트 (.md)
  meeting-facilitator.md
  meeting-debater-claude.md
  meeting-debater-codex.md
  meeting-game-*.md       # 게임 도메인 전문 토론자들
skills/
  meeting/                # /meeting 슬래시 커맨드 스킬
  meeting-agent/          # /meeting-agent 토론자 생성 스킬
```

## 설치

`agents/` 와 `skills/` 폴더 내용을 각각 다음 위치로 복사:

```
~/.claude/agents/        ← agents/*.md
~/.claude/skills/        ← skills/meeting, skills/meeting-agent
```

## 사용

- `/meeting <주제>` — 자동 다중 에이전트 회의 (Facilitator + 동적 토론자 풀)
- `/meeting-agent <자연어>` — 자연어 한 줄로 새 토론자 에이전트 생성 (3회 딥리서치 보강)

## 제외 항목

- `meetings/` — 회의 산출물 (세션별 회의록)
- `skills/meeting/temp-research/` — 딥리서치 임시 자료
