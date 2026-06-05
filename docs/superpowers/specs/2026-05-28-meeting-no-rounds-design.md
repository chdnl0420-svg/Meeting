# Meeting Skill — 라운드 폐지 + 연속 PASS 자동 종결 설계

- 일자: 2026-05-28
- 목적: `/meeting` 스킬의 투표-사유 모순 문제 근본 해결
- 영향 범위: `~/.claude/skills/meeting/SKILL.md`, `roles/common-debater.md`, 메모리 항목
- 우선순위: P0 (현 회의 진행 불가능 수준의 모순 다발)

---

## 1. 문제 정의

### 1-1. 관찰된 현상

`/meeting` 스킬의 다음 3 단계에서 토론자 응답의 **라벨(YES/NO)과 사유(REASON 본문) 의도가 불일치** 하는 패턴이 다발한다:

- 4-3-b `CONTINUE_VOTE` (라운드 종결 투표)
- 4-4 `SUBTOPIC_END_AGREE` (서브주제 종결 동의)
- 7 `END_AGREE` (회의 종료 동의)

### 1-2. 실측 사례 (2026-05-28 회의 `20260528-102807-mmorpg-2d쿼터뷰-비P2W`)

| 라운드 | 단계 | 토론자 | 라벨 | 사유 (의도) | 모순 유형 |
|---|---|---|---|---|---|
| R3 | CONTINUE_VOTE | monetization | NO (종결) | "LTV 설계 논점 누락 위험" | 사유=계속, 라벨=종결 |
| R3 | CONTINUE_VOTE | art-director | NO (종결) | "구체화 후 확정 권장" | 사유=계속, 라벨=종결 |
| R3 | CONTINUE_VOTE | designer | YES (계속) | "수익모델 별도 서브주제로" (이미 어젠다에 등재) | 인지 오류 |
| R4 | CONTINUE_VOTE | producer | YES (계속) | "본인 안 모두 반영, 종결 적합" | 사유=종결, 라벨=계속 |
| R4 | CONTINUE_VOTE | designer | YES (계속) | "현 통합안 채택 지지" | 사유=종결, 라벨=계속 |
| R4 | CONTINUE_VOTE | monetization | YES (계속) | "수익화 리스크 없음, 종결 가능" | 사유=종결, 라벨=계속 |

총 8 모순 / 회의 1건. 어젠다 결정 메타 서브주제가 R4 까지 폭주 후 사용자 강제 중단.

### 1-3. 근본 원인 분석

| # | 원인 | 설명 |
|---|---|---|
| 1 | **이분법 강제** | "조금만 더 다듬고 싶다" 같은 중간 의도를 표현할 슬롯 없음 → 일단 YES 로 안전책 |
| 2 | **YES/NO 매핑 비직관** | `YES = 다음 라운드` 인데 자연어 "yes" 가 "동의/지지" 로 해석돼 종결 의도가 YES 로 흐름 |
| 3 | **비대칭 부담** | 작은 우려라도 NO 던지면 "내가 막은 것" 같아서 안전하게 YES (false continue 편향) |
| 4 | **라벨-사유 검증 부재** | 응답 모순을 라우터도 에이전트도 안 잡음 |
| 5 | **메타 1-round cap 부재** | 어젠다 결정이 본 콘텐츠 흡수해 무한 정밀화 루프 |

---

## 2. 해결 설계

### 2-1. 핵심 원칙

**라운드 개념 자체 제거**. 순차 라운드로빈 연속 대화로 통일. "연속 PASS 수 == 활성 토론자 수" 일 때 자동 종결.

### 2-2. 새 흐름 (서브주제 1개 기준)

```
입력:
  active_debaters = [agent1, agent2, ..., agentN]
  subtopic_md     = 빈 파일 (사이클 진행하며 누적)
  consecutive_pass = 0
  i = 0
  speech_count = 0

LOOP:
  agent = active_debaters[i % len(active_debaters)]
  
  response = call(agent, context={
    topic: 큰 주제,
    prev_subtopic_conclusions: 이전 결론들,
    current_subtopic_md: 누적 발언 전체,
    your_turn: agent 정보 + 직전 발언자,
    mode: SPEAK_OR_PASS
  })
  
  if response.SPEAK == YES:
    consecutive_pass = 0
    append response.CONTENT to subtopic_md
    chat output: 🤖 {agent}: > {content}
    
    # NEST_REQUEST 즉시 분기
    if response.NEST_REQUEST:
      if depth < 3 and total_NEST_count < 10:
        # 자식 서브주제 즉시 개시
        child = create_child_subtopic(
          title = response.NEST_REQUEST.TITLE,
          context = response.NEST_REQUEST.REASON,
          active = active_debaters (부모 풀 상속, 진행자가 ADJUST 가능),
          speaking_order = [agent] + active_debaters - [agent] (요청자 먼저)
        )
        recurse into child (depth+1)
        after child ends: append child conclusion to parent_md
      elif total_NEST_count >= 10:
        trigger CEO_DECISION (cap overflow)
      elif depth >= 3:
        log "depth 3 도달, NEST 무효" to parent_md
  else:  # SPEAK: NO (PASS)
    consecutive_pass += 1
    chat output: 🤖 {agent}: _(PASS — {reason})_
  
  speech_count += 1   # SPEAK·PASS 무관하게 호출당 +1
  i += 1
  
  # 종결 조건 체크
  if consecutive_pass >= len(active_debaters):
    break  # 자동 종결
  
  # 무한 대화 가드 (cumulative, SPEAK + PASS 합산)
  if speech_count >= 3 * len(active_debaters):
    trigger CEO_DECISION (loop guard)

END:
  # 진행자 최종 결론 작성 (단 1회)
  facilitator → FINAL_CONCLUSION
  append to subtopic_md
  append to main.md "## 서브주제 결론"
  
  # 다음 서브주제 결정 (facilitator)
  facilitator → next subtopic OR PROPOSE_END_MEETING
```

### 2-3. NEST_REQUEST 즉시 분기 (사용자 명시)

- 토론자가 발언에 `NEST_REQUEST` 포함 → **즉시** 자식 서브주제 개시 (투표 없음)
- 자식 서브주제 첫 발언자 = NEST 요청자 (parent 의 다음 라운드로빈은 자식 종결 후 재개)
- 자식 활성 토론자 풀 = 부모 풀 상속 기본 (진행자가 ADJUST 가능)
- 자식도 동일 규칙 (라운드로빈 + 연속 PASS == 활성 수 → 종결)
- 자식 종결 후 부모 사이클 재개 (요청자 차례의 다음 i 위치부터)

### 2-4. 폐지 항목

| 폐지 | 사유 |
|---|---|
| `CONTINUE_VOTE` 키 (4-3-b) | 라벨 모순 원인. PASS 누적으로 대체 |
| 진행자 잠정 결론 (4-3-a) | 중간 잠정 결론은 모순 유발. 종결 직후 최종 결론만 1회 작성 |
| `SUBTOPIC_END_AGREE` (4-4) | 연속 PASS 가 합의 신호. 별도 폴링은 같은 질문 반복 |
| NEST_REQUEST 투표 (NEST_VOTE) | 즉시 분기로 단순화 |
| 4-3-c 동률 → CEO | 투표 자체 없음 |
| 4-3-d pass_streak >= 3 → CEO | 연속 PASS == 활성 수 = 자동 종결 (별도 CEO 호출 불필요) |
| 4-4-2 같은 ADDITIONAL_POINT 2회 → CEO | 4-4 폐지로 자동 소멸 |

### 2-5. CEO_DECISION 최종 트리거 (3개로 축소)

| 트리거 | 조건 | CEO 선택지 |
|---|---|---|
| **NEST cap overflow** | 11번째 NEST_REQUEST 시 | A) 분기 강행 (cap +1) / B) 흡수 / C) 종결 강제 |
| **무한 대화 가드** | 한 서브주제 발언 수 > 3 × 활성 토론자 수 | A) 즉시 종결 (현재까지 발언으로 결론) / B) 강제 PASS 라운드 (모두 PASS 강제) / C) 자식 서브주제 분기 |
| **회의 종료 단계 루프** | 같은 ADDITIONAL_SUBTOPIC 2회 연속 (7단계) | A) 채택 / B) 기각 후 종료 |

기존 CEO Advisory 메커니즘 (Director + idea-bank 자문) 은 그대로 유지 (모든 CEO_DECISION 에서 가용).

### 2-6. CEO 역할 정책 (옵션 1 — 최소 유지)

- 모든 회의 강제 참관 (결정자 슬롯)
- SPEAK_OR_PASS / NEST 분기 / 종결 자연 수렴에 일체 개입 안 함
- 위 3개 트리거 발생 시만 단독 호출
- 대부분 회의에서 CEO 호출 0회 — "비상시 소화기"

### 2-6-A. 진행자의 idea-bank 적극 검토 규칙 (사용자 명시)

진행자는 **모든 서브주제 시작 시 (3단계 RECOMMENDED_AGENTS) 와 풀 재평가 시 (6단계 AGENTS_ADDED/REMOVED)** 에 `meeting-idea-bank` 참여 여부를 **명시적으로 평가** 해야 한다. 수동적 누락 금지.

#### 평가 기준 (포함 권장 신호)
다음 중 1개 이상 충족 시 idea-bank 포함:
1. **참신성·혁신 비중**: 큰 주제·서브주제에 "새로운", "차별화", "기존 X 와 다른" 같은 키워드 또는 의도
2. **컨벤셔널 수렴 위험**: 도메인 전문가만 모이면 업계 표준안으로 빠르게 수렴할 가능성 (예: MMORPG 코어루프 = 또 던전·일퀘·레이드)
3. **장르·메카닉 융합**: 둘 이상의 게임 장르·시스템을 결합하는 결정 (예: MMORPG + 로그라이크, F2P + 1회성 결제)
4. **메카닉 ↔ 세계관 연동**: 루도내러티브 설계 (메카닉이 스토리·세계관과 어떻게 결합하는가)
5. **규제·플랫폼 제약 우회**: 한국 확률공시·EU DFA·중국 판호 등 제약 안에서 창의적 BM 설계 필요

#### 평가 기준 (제외 권장 신호)
다음 중 모두 충족 시 idea-bank 제외 가능:
1. 순수 기술적 결정 (Netcode tick rate, DB 스키마, ACID 보장 등)
2. 명확히 경계된 엔지니어링 문제 (성능 최적화, 빌드 파이프라인 등)
3. 데이터 분석·KPI 산출 같은 정량 작업 (참신성보다 정확성)

#### 응답 형식 추가
진행자 응답 (3단계 / 6단계) 에 다음 필드 **필수 추가**:

```
IDEA_BANK_EVAL: included | excluded
IDEA_BANK_REASON: <포함/제외 사유 1-2줄. 평가 기준 인용 권장>
```

#### 라우터 가드
- `IDEA_BANK_EVAL` 필드 누락 → 1회 재시도 강제. 재시도도 누락 시 라우터가 자동으로 `included` 로 강제 (적극 참여 default)
- `IDEA_BANK_EVAL: excluded` 인데 사유가 모호 (제외 신호 3가지 모두 명시 안 됨) → 1회 재시도. 그래도 모호하면 `included` 강제

#### 효과
- idea-bank 가 도메인 전문가 풀에 섞여 "novelty challenger" 역할 수행
- 동일 페르소나 풀이 반복 패턴에 갇히는 것 방지
- "참신성은 항상 누군가 의식해야 잡힌다" 원칙

### 2-7. 어젠다 결정 메타 서브주제 특수 규칙

어젠다 결정 서브주제는 본 콘텐츠를 흡수하지 않도록 다음 프롬프트 추가:

```
=== 어젠다 결정 모드 ===
당신은 어젠다 (회의에서 다룰 서브주제 목록) 를 정하는 메타 단계에 있다.
규칙:
1. 본 라운드로빈 첫 차례에 본인 도메인 우선순위 서브주제 1개만 제안 (5줄 이내)
2. 본인 제안의 NEST_REQUEST 는 허용 (즉시 분기 가능)
3. 본인 제안 후 다음 차례부터는 PASS 권장 — 단, 본인 제안이 다른 토론자에 의해
   심각하게 왜곡되었을 때만 SPEAK 로 교정
4. 본 콘텐츠 (예: D3 RMAH, BP 단가, GDD 섹션 디테일) 는 해당 서브주제 진입 후 다룸
```

자연스럽게 1 사이클 후 연속 PASS 가 누적되어 자동 종결.

### 2-8. 변경 영향 받는 SKILL.md 섹션

1. `## Workflow` 의 4번 (라운드 루프) → 2-2 의사 코드로 전면 재작성
2. `## Workflow` 의 3·6번 (진행자 호출) → 응답 형식에 `IDEA_BANK_EVAL` + `IDEA_BANK_REASON` 필드 필수 추가
3. `## 토론자 호출 디테일 > Backend: claude` 컨텍스트 패키지 → `=== 누적 발언 ===` 항목 강조
4. `## 안전 가드` 표 → 동률/루프 행 4건 삭제, CEO 트리거 3개로 재작성, `IDEA_BANK_EVAL` 누락 가드 추가
5. `## 출력 사례` → 라운드 마커 제거, 연속 PASS 표시 형식 추가, 진행자 풀 결정 출력에 idea-bank 평가 표시
6. `## 중첩 서브주제` 섹션 → NEST 투표 부분 제거, 즉시 분기로 재작성
7. (신규 섹션) `## 진행자의 idea-bank 적극 검토 규칙` 추가
8. (신규 섹션) `## 어젠다 결정 메타 서브주제 특수 규칙` 추가
9. `~/.claude/agents/meeting-facilitator.md` 의 페르소나 정의에 "idea-bank 적극 검토" 책임 명시 추가
10. `## When NOT to Use` 등 도입부 텍스트는 영향 없음

### 2-9. 영향 받는 메모리 항목

| 메모 | 조치 |
|---|---|
| `meeting-round-continue-vote` | **DEPRECATED** 마킹. 신규 `meeting-no-rounds-passall` 로 대체 |
| `meeting-pass-normalized` | 강화 (PASS 가 적극 수렴 신호로 격상) |
| `meeting-no-parallel-except-vote` | 폐지 가능 (투표 자체 없음). 단 END_AGREE (7단계) 는 유지하므로 그것만 예외로 축소 |
| `meeting-nested-subtopics` | NEST 투표 부분 제거 (즉시 분기) |
| `meeting-nest-no-unilateral-reject` | **DEPRECATED** (NEST 투표 없음 → 단독 판단/투표 양쪽 모두 무의미) |
| `meeting-vote-reason-label-mismatch` | 본 spec 으로 해결됨. **CLOSED** 마킹 |
| `meeting-ceo-observer-decider` | 트리거 3개로 재정의 (메모 본문 업데이트) |

### 2-10. 신규 메모리 항목

- `meeting-no-rounds-passall.md` — 라운드 폐지 + 연속 PASS 종결 + 어젠다 1-cycle cap
- `meeting-nest-immediate-branch.md` — NEST_REQUEST 즉시 분기 + 요청자 첫 발언
- `meeting-idea-bank-active-eval.md` — 진행자가 모든 서브주제 풀 결정 시 idea-bank 참여 명시 평가 + 평가 기준 5개 (포함) / 3개 (제외)

---

## 3. 검증 시나리오

### 3-1. 시나리오 A — 정상 종결 (모든 토론자가 합의)

```
활성: [A, B, C, D] (N=4)
i=0: A SPEAK     (cp=0)
i=1: B SPEAK     (cp=0)
i=2: C SPEAK     (cp=0)
i=3: D SPEAK     (cp=0)
i=4: A PASS      (cp=1)
i=5: B PASS      (cp=2)
i=6: C PASS      (cp=3)
i=7: D PASS      (cp=4 == N → 종결)
→ 진행자 최종 결론 작성, 다음 서브주제로
```

총 호출: 8 (발언 4 + 패스 4) + 진행자 1 = 9. **모순 라벨 0**.

### 3-2. 시나리오 B — 부분 합의 후 한 명이 추가 의견

```
활성: [A, B, C, D]
i=0: A SPEAK     (cp=0)
i=1: B SPEAK     (cp=0)
i=2: C PASS      (cp=1)
i=3: D PASS      (cp=2)
i=4: A PASS      (cp=3)
i=5: B SPEAK     (cp=0)  # 추가 의견
i=6: C PASS      (cp=1)
i=7: D PASS      (cp=2)
i=8: A PASS      (cp=3)
i=9: B PASS      (cp=4 == N → 종결)
```

추가 의견 자연스럽게 흡수. 별도 ADDITIONAL_POINT 슬롯 불필요.

### 3-3. 시나리오 C — NEST 즉시 분기

```
부모 (depth=1) 활성: [A, B, C]
i=0: A SPEAK + NEST_REQUEST("X")
  → 자식 (depth=2) 활성: [A, B, C], 발언 순서 [A, B, C]
  → 자식 사이클 진행
  → 자식 종결 → 자식 결론 부모 md 에 append
i=1 (부모 재개): B SPEAK
i=2: C SPEAK
... 부모 사이클 계속
```

NEST 투표 호출 없음. 분기/흡수 결정 없음.

### 3-4. 시나리오 D — 무한 대화 가드 발동

```
활성: [A, B, C, D] (N=4)
호출 11번까지 cp 가 0~3 사이 진동, 4 도달 안 함
i=11: speech_count == 12 >= 3 × 4 → CEO_DECISION 트리거
  → CEO 선택: A) 즉시 종결 / B) 강제 PASS 라운드 / C) 자식 분기
```

극단적 케이스만 CEO 개입. (SPEAK 12개 / PASS 0~11 혼합 / 둘 다 가능)

---

## 4. 마이그레이션 계획

### 4-1. SKILL.md 패치 순서

1. (`Workflow > 4`) 의사 코드 교체
2. (`안전 가드`) 표 수정
3. (`중첩 서브주제`) NEST 즉시 분기 재작성
4. (`출력 사례`) 채팅 출력 형식 갱신
5. (신규) 어젠다 결정 메타 규칙 섹션
6. `roles/common-debater.md` 의 SUBTOPIC_END_AGREE / CONTINUE_VOTE / NEST_VOTE 모드 정의 삭제 + SPEAK_OR_PASS / END_AGREE 만 유지

### 4-2. 메모리 정리

- DEPRECATED 마킹 3건: `meeting-round-continue-vote`, `meeting-nest-no-unilateral-reject`, `meeting-vote-reason-label-mismatch` (closed)
- 본문 업데이트 3건: `meeting-pass-normalized`, `meeting-no-parallel-except-vote`, `meeting-ceo-observer-decider`
- 신규 2건: `meeting-no-rounds-passall`, `meeting-nest-immediate-branch`
- MEMORY.md 인덱스 정리

### 4-3. 회귀 테스트

- 직전 회의 (`20260528-102807-mmorpg-2d쿼터뷰-비P2W`) 의 어젠다 결정 단계를 새 설계로 재실행 가정 시뮬레이션
- 예상 결과: 1 사이클 (8 발언) + 일부 PASS 누적 → 자동 종결 (호출 ~12회)
- 비교: 기존 호출 ~70회 → -83% 절감 + 모순 0건

---

## 5. 위험 및 완화

| 위험 | 완화 |
|---|---|
| **무한 대화** (계속 SPEAK 만 누적) | `speech_count > 3 × N` 가드 + CEO_DECISION |
| **즉시 PASS 폭주** (첫 사이클 만에 종결) | 어젠다 결정 외 서브주제는 진행자가 SUBTOPIC_CONTEXT 에 "최소 1 사이클 발언 권장" 명시 (강제 아님) |
| **NEST 남발** | depth 3 하드 캡 + 누적 NEST cap 10 + 트리거 조건 3가지 가이드라인 (강제 아님) |
| **CEO 미활용** | 의도된 설계. CEO 는 "소화기". 매 회의 호출은 과잉 |
| **PASS 의미 혼동** | common-debater.md 에 "PASS = 잠정 결론 수용 + 종결 의사" 명시 |

---

## 6. 비기능 요구

- **호출 회수**: 현 회의 대비 -80% 이상 (어젠다 기준 ~70 → ~12)
- **모순 발생률**: 8건/회의 → 0건/회의 (라벨 자체 제거)
- **사용자 개입**: 회의 정상 흐름에서 0회 (현재와 동일)
- **호환성**: 기존 `meetings/<ts>-<slug>/` 디렉터리 구조·파일명 규칙 유지

---

## 7. 미해결 / 후속 작업

- (후속) NEST 즉시 분기의 cap 10 도달 시점에 사용자 참고 정보 어떻게 노출할지 (CEO_DECISION 시점에 모든 NEST 트리 출력 권장)
- (후속) 어젠다 결정 메타 규칙을 다른 메타 서브주제 (예: 회의 종료 검토) 에도 적용할지 검토
- (후속) 다중 NEST_REQUEST 단일 발언에 포함 시 처리 정책 (현 설계: 단일 NEST 만 허용, 두 번째는 무시)

---

## 부록 A — common-debater.md 변경 사양 (요약)

### 폐지 모드
- `CONTINUE_VOTE` (4-3-b)
- `SUBTOPIC_END_AGREE` (4-4)
- `NEST_VOTE` (NEST 투표)

### 유지 모드
- `SPEAK_OR_PASS` (라운드로빈 발언)
- `END_AGREE` (회의 종료 동의, 7단계)
- `CEO_DECISION` (CEO 단독 호출)
- `ADVISORY` (CEO 자문)

### SPEAK_OR_PASS 응답 형식 (구조 변경 없음, 동작만 변경)
```
SPEAK: YES
CONTENT:
<5줄 이내>
[NEST_REQUEST:
TITLE: <제목>
REASON: <이유>]

또는:

SPEAK: NO
REASON: <한 줄 — 패스 사유>
```

### 동작 변경 사항 (구조는 동일하나 라우터 처리가 달라짐)
- `SPEAK: NO` 의미가 격상: "이번 라운드 할 말 없음" → "잠정 결론 수용 + 종결 의사 표명". 연속 PASS 가 활성 수와 같으면 자동 종결
- `NEST_REQUEST` 포함된 SPEAK 응답: 라우터가 응답 받자마자 **즉시 자식 서브주제 개시** (투표·승인 없음). 단일 응답에 NEST_REQUEST 가 둘 이상이면 첫 번째만 처리, 나머지 무시
- 단일 발언에 `CONTENT` + `NEST_REQUEST` 둘 다 있으면: CONTENT 는 부모 md 에 기록, 자식 서브주제 즉시 진입 (발언자 = 자식 첫 발언자)

### 신규 가이드 (프롬프트에 명시)
- PASS 의 의미: "잠정 결론 수용 + 종결 의사 표명"
- 같은 논점 반복할 거면 PASS
- 새 근거·반박·관점이 있을 때만 SPEAK
- 모호하면 PASS (안전 기본값)
- NEST_REQUEST 는 신중히 — 즉시 자식 서브주제로 분기되며 부모 진행이 일시 정지됨
