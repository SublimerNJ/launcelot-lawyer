---
name: policy-redraft
description: >
  policy-diff 또는 gap-surfacer에서 식별된 위반 라인을 한국 형법·정통망법·개보법·표시광고법·
  변호사법·대한변협 광고규정 기준으로 수정한 대안 라인을 생성한다. 원본 톤 유지를 1순위로,
  진실성·공익성 항변 강화 또는 식별성 완화를 2순위로 한다. 사용자가 "이 라인 어떻게
  바꿔?", "수정안 줘", "redraft", "이 갭 닫게 수정해줘"라고 말하거나 policy-diff /
  gap-surfacer에서 핸드오프할 때 실행한다. 본 스킬의 출력은 첫 제안일 뿐이며,
  최종 대본 적용은 사용자 또는 koreanizer / 변호사별 대본 스킬(kmj·kmg·sbh·ldg·wja)의
  소관이다.
argument-hint: "[gap-id | finding-id | pattern-id | --line <slug-B>:<N>]"
---

# /launcelot-regulatory-legal:policy-redraft

This skill drafts marked-up alternative lines that close a gap surfaced by `policy-diff` or `gap-surfacer`. It is a *first draft for review*, not a final rewrite. It does not touch the source script.

## Run order

1. Load `~/.claude/plugins/config/launcelot-regulatory-legal/CLAUDE.md`. Cross-check Launcelot-Product-Legal config for tone and risk calibration if installed.
2. Resolve the input target.
3. Pull the corresponding finding context (the diff record or the gap entry).
4. Generate three candidate redrafts using the strategies below.
5. Emit a marked-up output the user can paste back into their drafting environment.
6. Optionally hand off to koreanizer / per-attorney writing skill for tone smoothing.

## Input target resolution

- `gap-id`: load `gap-tracker.yaml` entry. Use its `slug_b`, `line_n`, `quote`, `slot_a_id`, `defenses`, `suggested_fix_direction`.
- `finding-id` (a policy-diff Finding identifier): load the run file under `diffs/`.
- `pattern-id`: load `patterns.yaml`. Run the strategies once at the pattern level, then again per concrete line, so the user gets one shared fix and N line-specific fixes.
- `--line <slug-B>:<N>`: free-floating mode. Ask for the slot A context (조문·판례 식별자) if not provided. Refuse to proceed without slot A.

## Redraft strategies

Always produce three candidates per line. Label each with which strategy was applied and what it costs.

### Strategy 1 — Tone-preserving narrow fix

Goal: 최소 침습. 원본의 어휘·리듬·말맛을 가능한 한 보존하면서 위법 요건만 결락시킨다.

Tactics (taxonomy):

- 식별성 약화: 실명·직책·날짜·사건번호 등 식별 신호를 일반화. 예: "○○법무법인 김 변호사" → "한 대형 로펌의 형사 전문 변호사".
- 사실 적시 → 의견·평가로 전환: "그 사람은 사기죄로 처벌받을 짓을 했다" → "그 행동이 사기죄 구성요건에 가깝게 보인다는 평가가 가능하다".
- 단정 표현 완화: "무조건 무죄", "100% 승소" → "법리상 가능성이 높다는 견해도 있다".
- 비교 광고 표현 제거: "최고 승소율"·"전국 1등" 등 표시광고법·광고규정 저촉 표현을 제거하거나 사실관계로 환원("작년 사건 수임 N건 중 무죄 M건"처럼 실증 가능한 진술로).
- 카테고리화: "이 변호사는 …다" → "이런 유형의 변호사는 …다".

Output should include the diff: 원본 ↔ 수정.

### Strategy 2 — Defense-strengthening fix

Goal: 본문이 어차피 다루어야 할 내용이라면 진실성·공익성 항변(형법 310)·정당행위(20)·인용의 정당한 범위(저작권법 28)·표시광고법의 실증 표현 요건을 구조적으로 강화한다.

Tactics:

- 사실의 출처를 라인 안에서 명시: "다수 보도에 따르면…" → "조선일보 2024.7.3.자 보도(URL)에 따르면…".
- 공익성 신호 삽입: 이 콘텐츠가 시청자에게 어떤 일반적 주의를 환기시키려는지 한 문장으로 명시.
- 항변의 요건을 의식한 표현: "이 사건의 피고인은 거짓말을 했다" → "기소된 진술 중 사실과 다른 부분이 있다는 검찰 주장이 있다".
- 인용 범위 명시: 판결문·기사 인용 시 인용 분량·출처·인용 목적을 한 줄로 표시.
- 표시광고법: 실증 자료의 존재와 위치를 영상 설명란/자막에 명시할 것을 redraft 옆에 작성자 노트로 부착(라인 자체는 그대로 두되 면책 고지로 보완).

### Strategy 3 — Restructuring fix

Goal: 라인 단위 수정으로는 위법 요건이 해소되지 않는 경우, 단락 또는 섹션 단위 재구성을 제안한다. 예: 한 명의 식별 가능한 인물을 중심으로 사건을 전개하는 구성을, 동일 유형의 사건 패턴 일반을 다루는 구성으로 재편하는 안.

Tactics:

- 사람 중심 → 사안 유형 중심.
- 단정 결론 중심 → 절차·구성요건 설명 중심.
- 사건 결과 평가 중심 → 시청자 행동 가이드 중심.
- 변호사 광고 톤 → 공익 정보 톤.

Strategy 3은 라인 단위 redraft가 아니라 단락 또는 섹션 단위의 구성 변경 가이드로 출력한다(시연 가능한 새 라인 1~2개 + 변경 방향 설명).

## Output format

```markdown
# Redraft proposal — <target id>

원본 라인: "<exact quote>"
출처(스크립트): <slug-B>, line <N>
원인(slot A): <조문 또는 판례 식별자> <(검증: …)>
원래 risk band: <…>

---

## Candidate 1 — Tone-preserving narrow fix
**수정안:** "<line>"
**적용 전술:** <식별성 약화 | 사실→의견 | …>
**유지된 톤·말맛:** <한 줄>
**예상 등급 변화:** <명백 → 가능 | 가능 → 회색지대 | …>
**남는 위험:** <없음 | …>
**필요 작성자 노트:** <… | 없음>

## Candidate 2 — Defense-strengthening fix
**수정안:** "<line>"
**적용 전술:** <출처 명시 | 공익성 신호 | …>
**라인 자체 위험 등급 변화:** <…>
**부착 노트:** <영상 설명란 또는 자막에 추가할 한 줄>
**남는 위험:** <…>

## Candidate 3 — Restructuring guidance
**구성 변경 방향:** <한 단락>
**예시 신라인 1:** "<…>"
**예시 신라인 2:** "<…>"
**이 변경이 닫는 갭:** <gap-id 들 또는 finding-id 들>
**이 변경이 만들지 모르는 새 라인 단위 검토 필요:** <yes/no, 사유>

---

## 톤 후속 처리 권고

- koreanizer로 자연스러움 보정 (사용 가능 시).
- 채널 화자 톤(kmj/kmg/sbh/ldg/wja 중 해당) 스킬로 마지막 패스.

## 적용 시 후속 작업

- 사용자가 Candidate를 채택해 본문을 수정한 뒤 launch-review를 재실행하면 해당 revision이 새 launch-review revision으로 기록되며, gap-surfacer의 `close <gap-id>` 흐름에서 `revision_ref`로 인용된다.
- 본 redraft는 사용자가 채택할 때까지 어떤 갭도 close하지 않는다.
```

## Persistence

- `~/.claude/plugins/config/launcelot-regulatory-legal/redrafts/<gap-id-or-finding-id>-<YYYY-MM-DD>-<HHMM>.md`
  Run output.
- `redrafts/_index.yaml`:
  ```yaml
  - target: <gap-id | finding-id | pattern-id>
    at: <ISO-datetime>
    chosen_candidate: <1 | 2 | 3 | none>
    user_decision_note: <…>
    led_to_close: <gap-id | null>
  ```

## Calibration

- Launcelot-Product-Legal `## Risk calibration` if available:
  - 보수적: Candidate 1·2 모두 "+1 등급 안전쪽"으로 작성. Candidate 3을 기본 권장으로 설정.
  - 중도: 세 후보를 균형으로.
  - 공격적: Candidate 2를 기본 권장. 단 명백 → 회색지대 이하로는 redraft하지 않는다(공격적이라도 명백은 명백).
- 채널 톤(`## Channel profile` → tone)에 맞게 후보의 어휘·문장 길이·구두점을 조정.

## Failure handling

- gap-id가 `slot_a_verification: not-found`인 채로 들어옴: redraft를 거부. 사유: "근거 조문·판례가 검증되지 않은 갭은 수정 대상이 아니다. 먼저 policy-diff를 재실행하여 slot A를 확정하라."
- 라인이 너무 길거나 한 라인 안에 두 죄목 이상이 얽혀 있음: Candidate 1·2를 분리(죄목별 별도 redraft)하고 Candidate 3은 단락 재구성으로 한 번에 처리.
- 사용자가 직접 작성한 redraft가 더 낫다고 판단해 본 스킬의 후보를 모두 거부: `chosen_candidate: none`으로 기록하고 사용자가 채택한 자체 라인을 `user_decision_note`에 기록.

## What this skill does NOT do

- 사용자의 원본 스크립트 파일을 수정하지 않는다. 본 스킬의 출력은 항상 제안이며, 적용은 사용자 또는 다른 스킬의 소관.
- 갭을 자동으로 close하지 않는다. close는 gap-surfacer의 `close` 서브커맨드에서 사용자가 명시적으로 실행한다.
- 새로운 조문·판례를 단정하지 않는다. 본 스킬은 policy-diff/gap-surfacer가 이미 launcelot-lawyer로 검증한 slot A만 인용한다.
- 영상 자막·설명란을 직접 편집하지 않는다. 그것은 사용자의 영상 제작 도구가 한다.
