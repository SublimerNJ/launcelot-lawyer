---
name: is-this-a-problem
description: >
  "이거 해도 되나?" 형 즉답 스킬. 짧은 질문·짧은 표현·짧은 시나리오를 사용자의 calibration과
  Hard rule에 대조해 30초 안에 한 줄로 판정한다. 즉시 가능 / 한 번 더 봐야 함 / 보류
  세 가지 중 하나. 사용자가 "이거 해도 돼?", "이 표현 가능?", "이 사건 직후 영상
  올려도 돼?", "이거 사건수임 광고로 보일까?", "is this a problem", "sanity check"라고
  말할 때 실행한다. 본 스킬의 출력은 게이트가 아니며, 보류 또는 한 번 더 봐야 함이면
  marketing-claims-review·launch-review·feature-risk-assessment로 핸드오프한다.
argument-hint: "[질문 또는 짧은 표현]"
---

# /launcelot-lawyer-pro:is-this-a-problem

This skill answers a Slack-grade question: "can we do this?". It produces one of three calls in under 30 seconds: 즉시 가능 / 한 번 더 봐야 함 / 보류. It is not the launch gate — launch-review is. It is the early Slack-style sanity check.

## Run order

1. Load `~/.claude/plugins/config/launcelot-lawyer-pro/CLAUDE.md`. Read `## Speech policy`, `## Hard rules`, `## Risk calibration`. Load matter-scoped overrides if `Active matter` is set.
2. Read the user's question. If it is longer than ~5 sentences or includes a full script, refuse this skill and redirect to `marketing-claims-review`: 본 스킬은 단발 질의용.
3. Pattern-match against the calibration:
   - Hard rule hit → `보류`.
   - Tier 1 materiality category 직격 → `보류` 또는 `한 번 더 봐야 함`.
   - Speech policy 라벨 위반 직격 → `한 번 더 봐야 함`.
   - 위 어디에도 명백히 해당하지 않으면 → `즉시 가능` (단 조건 있을 수 있음).
4. 한 줄 판정 + 사유 1~2문장 + 다음 단계 한 줄 출력.
5. `보류` 또는 `한 번 더 봐야 함`이면 핸드오프 제안.

## Inputs the skill accepts

| 유형 | 예시 | 처리 |
|---|---|---|
| 단발 표현 | "100% 무죄 받을 수 있습니다" | 단정 표현 정책과 광고규정에 대조 |
| 단발 시나리오 | "이 사건 1심 끝났는데 영상 올려도 돼?" | Pending-case policy + 식별성 + 매터 정책 |
| 단발 인물 언급 가부 | "○○ 회장 이름 그대로 써도 돼?" | Real-name policy + 공인 여부 + 사건 단계 |
| 단발 비교 광고 | "다른 로펌보다 우리가 빠르다고 써도 돼?" | 광고규정 + 표시광고법 비교광고 |
| 단발 자작 가이드 | "답변서 자작 가이드 영상 만들어도 돼?" | Hard rule (must-escalate / never) |

## Pattern matching rules

### 즉시 보류 (`보류`)

다음 중 하나라도 명백히 해당:
- `## Hard rules` → Never에 등록된 패턴과 직접 매칭.
- 사용자가 명시적으로 "절대 금지"로 정의한 Speech policy 카테고리에 정면 충돌 (예: real-name = "절대 실명 사용 안 함"인데 실명 사용 가부 질의).
- 명백한 무자격 자문 영역 침해 (의료 진단·세무 환급 절차 등) 직접 질의.
- 진행 중 사건 + 사용자 pending-case policy = "절대 안 함" 조합.

출력은 한 줄 + 사유 + 후속.

### 한 번 더 봐야 함 (`한 번 더 봐야 함`)

다음 중 하나라도 해당:
- `## Hard rules` → Must escalate에 등록된 패턴과 매칭.
- Speech policy 라벨이 회색지대에 있음 (예: speculative-statement = "중도"인데 의견·평가 표현이 한정자 없이 들어감).
- Tier 1 materiality 영역의 신착 변경(reg-feed-watcher digests/_unresolved.md)에 묻혀 있어 영향 가능성이 있음.
- 실명·사건 단계 등 식별성 신호가 있는데 substantiation을 확인하지 못함.

출력: 한 줄 + 무엇을 확인해야 다음 단계로 갈 수 있는지 한 줄 + 권장 후속 스킬.

### 즉시 가능 (`즉시 가능`)

세 가지 모두 만족:
- Hard rule 미해당.
- Speech policy 카테고리 모두 라벨의 허용 범위 안.
- Tier 1 materiality 직격 영역 신호 없음.

출력: "즉시 가능 — <짧은 이유>." 단 조건이 있으면 "단, <조건>"을 한 문장 더 부착.

## Output template

```markdown
[은행 — <보류 | 한 번 더 봐야 함 | 즉시 가능>]

질문: "<inline quote — first 80 chars>"
매터: <slug | practice-level>

판정: <라벨>
사유: <1~2문장. 어느 규범·정책·룰에 닿는지 인용.>
조건: <"단, …" 형식. 없으면 한 줄 생략.>

다음 단계: <한 줄 — `marketing-claims-review` / `feature-risk-assessment` / `launch-review` / `policy-diff` / "지금은 추가 액션 없음" 중 하나.>
```

조문·판례 인용이 들어가면 `launcelot-lawyer`로 실존 확인 후. 비가용이면 인용 자체를 생략하고 사유를 룰 라벨 수준에서만 표현.

## Calibration

- 사용자 risk appetite가 `보수적`이면 `한 번 더 봐야 함`을 `보류`로 한 단계 끌어올린다. `즉시 가능`은 그대로.
- `공격적`이면 `한 번 더 봐야 함` 중 일부를 `즉시 가능 — 단, …` 형태로 내릴 수 있다. 단 Hard rule 매칭은 절대 내리지 않는다.

## Persistence

- 본 스킬은 디스크에 큰 흔적을 남기지 않는다. 다만 모든 호출은 한 줄로 로그된다:
  - `<scope>/is-this-a-problem.log` — append-only.
  - 형식: `<ISO-datetime>\t<call>\t<question first 80>\t<handoff target or '-'>`
- `<scope>`는 활성 매터에 따라 `matters/<slug>` 또는 프랙티스-레벨.

## Handoff contract

`한 번 더 봐야 함` 또는 `보류`로 끝났을 때 후속 스킬을 *제안*만 한다. 자동 호출하지 않는다. 사용자가 "그럼 launch-review 돌려" 같은 명시적 지시를 줘야 다음 스킬이 작동.

## Self-limits

- 본 스킬은 한 번에 한 질문만 처리한다. 사용자가 두 가지를 동시에 묻으면 하나만 처리하고 나머지는 별도 호출을 권한다.
- 본 스킬은 변호사 자신의 행동 판단(예: 외근 가부·수임 가부)에는 답하지 않는다. 콘텐츠 표현·게시 가부에 한정.
- 본 스킬은 결정의 근거가 calibration·Hard rule·Speech policy일 때만 즉답한다. 그 밖의 영역(예: 새로운 죄목·새로운 광고규정 개정 적용)이 필요하면 자동으로 `한 번 더 봐야 함`으로 격하한다.

## What this skill does NOT do

- 어떤 라인도 수정·생성·삭제하지 않는다.
- 어떤 갭도 만들지 않는다.
- 어떤 launch-review revision도 만들지 않는다.
- 모델 지식으로 조문·판례를 단정하지 않는다.
