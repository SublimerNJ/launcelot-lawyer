---
name: gap-surfacer
description: >
  policy-diff에서 추출된 위반 가능성을 영속 트래커로 관리한다. 상태는 open / 수정완료 /
  리스크수용 / 비공개결정 / 보류 다섯 단계. 같은 죄목 패턴이 라이브러리 전반에 반복되거나
  같은 갭이 오래 미해소되면 표면화한다. 사용자가 "갭 트래커 봐줘", "open gap 뭐 있어",
  "이 갭 닫아 — 수정 완료", "리스크 수용으로 처리", "gap status"라고 말하거나
  policy-diff가 핸드오프를 던질 때 실행한다.
argument-hint: "[list | add | close <id> | accept-risk <id> | unpublish <id> | snooze <id> <days>]"
---

# /launcelot-lawyer-pro:gap-surfacer

This skill is the persistent memory of unresolved exposure across the script library. policy-diff finds candidate gaps; gap-surfacer is what keeps them visible until they are deliberately closed.

## Run order

1. 매터 활성 여부 확인. 활성이면 `matters/<slug>/gap-tracker.yaml`을, 비활성이면 프랙티스-레벨 `~/.claude/plugins/config/launcelot-lawyer-pro/gap-tracker.yaml`을 사용. 파일이 없으면 자동 생성(빈 yaml). 별도 정책 파일은 로드하지 않는다 — aging threshold·closure rule 등은 본 스킬에 박혀 있는 기본값 사용.
2. Route by subcommand (default `list`).

## Subcommands

### list

Default action. Print open gaps in priority order:

1. Tier 1 변경(reg-feed-watcher의 Tier 1 디제스트)에서 유래한 갭.
2. 같은 죄목·규정 패턴으로 라이브러리 내 N≥3건이 묶인 갭(패턴 갭).
3. Aging breach 갭 (기본 임계값 7 / 14 / 30일 초과).
4. 나머지 open 갭은 risk band 명백 > 가능 > 회색지대 순.

For each gap row:

```markdown
- [<gap-id>] <risk-band> · <죄목 또는 규정 식별자>
  스크립트: <slug-B>, line <N>
  발견일: <YYYY-MM-DD> · 연령: <N일>
  상태: open
  유래: policy-diff run <run-id>
  slot A: <identifier> · source <…> · fetched_at <…>
  대본 인용: "<exact line>"
  사용자 결정 권고: <…>
```

Footer:

```markdown
요약: open <n> / 수정완료 <n> / 리스크수용 <n> / 비공개결정 <n> / 보류 <n>
패턴 갭: <pattern-id> — <죄목> — <묶인 갭 N건>
Aging breach: <gap-id>들 — 평균 <N일>
```

### add

policy-diff가 핸드오프할 때만 실행되는 경로(사용자가 직접 add를 쳐도 막지 않지만 권장하지 않음).

Input shape (policy-diff의 finding을 그대로 받음):

```yaml
slug_b: <…>
line_n: <…>
quote: <…>
risk_band: <…>
slot_a_id: <…>
slot_a_snippet:
  source: <…>
  url: <…>
  fetched_at: <…>
  fetch_method: <…>
  effective_date: <…>
  raw_quote: |
    <…>
defenses: [<…>]
suggested_fix_direction: <…>
discovered_in_diff_run: <run-id>
```

Behavior:

1. **Reject if `slot_a_snippet.raw_quote` is empty**. snippet이 없는 finding은 갭으로 만들지 않는다(근거 조문이 존재 확인되지 않은 갭은 트래킹 대상 아님).
2. If a gap with the same `(slug_b, line_n, slot_a_id)` triple is already in the tracker:
   - If status is `open`: bump `last_seen_in_run` and append `discovered_in_diff_runs[]`. Do not create a duplicate.
   - If status is `수정완료`: re-open it and append a note that the same line still matches the same slot A. This is a regression signal.
   - If status is `리스크수용`: leave it closed, but record the re-detection in `risk_acceptance_audit[]`.
3. Otherwise create a new gap with status `open`. snippet 객체 전체를 `slot_a_snippet`으로 보존한다.
4. Pattern detection: after writing, re-compute pattern groupings. A pattern requires ≥3 gaps sharing `slot_a_id` (or a normalized form of it for related 죄목들), regardless of which script they live in. Promote the pattern in the tracker.

### close <gap-id>

Close as `수정완료`. Required inputs:

- `revision_ref`: the launch-review revision that recorded the fix (slug-B + revision number — `reviews/<slug>.md`의 Revision <N>). 사용자가 직접 작성한 수정 증빙(diff 또는 변경된 라인 두 줄)도 허용하되 그 텍스트를 `proof_text`로 보존한다.
- `closed_at`: ISO datetime (default: now).
- `closed_by`: user handle.

Refuse to close without `revision_ref` 또는 `proof_text`. The skill's job is to keep gaps alive until there is evidence of fix.

### accept-risk <gap-id>

Close as `리스크수용`. Required inputs:

- `rationale`: free text, mandatory. Stored verbatim and reproduced any time the gap reappears (Module 3 pattern repeat, future diff against new slot A, etc.).
- `accepted_by`: user handle.
- `accepted_at`: ISO datetime.
- `re-review_at`: future date by which the user wants this re-examined. Default: 90일. Re-review surfaces the gap back to `open` on the chosen date with the prior rationale attached.

### unpublish <gap-id>

Close as `비공개결정`. Means the underlying script was taken down or set unlisted to resolve the gap. Required: `unpublish_action` (delete / unlist / set-private) and `acted_at`. The skill checks `reviews/_index.yaml`'s `current_recommendation` for that slug-B; if the slug-B's status does not match (e.g., still `게시함`), flag a mismatch and refuse to mark `비공개결정` until the user reconciles.

### snooze <gap-id> <days>

Move the gap to status `보류` for N days. Required: `snooze_reason`. Snoozed gaps are excluded from `list` defaults but appear in `list --include-snoozed` and on the snooze expiry date automatically revert to `open` with an aging counter that does not skip the snoozed days.

## Aging policy (기본값)

기본 임계값 7 / 14 / 30일. 사용자가 자연어로 "갭 알림을 더 자주" / "30일까지 기다려"라고 말하면 `gap-tracker.yaml` 최상단의 `aging_thresholds:` 필드에 그 값을 lazy로 기록한 뒤 그 이후 실행부터 적용.

- 7일: 다음 list 실행 시 `Aging breach`로 표시.
- 14일: 사용자에게 직접 "이 갭에 대해 결정을 내려달라" 알림.
- 30일: 사용자가 명시적 결정을 내릴 때까지 매 실행 시 알림. 어느 알림도 자동 종결을 만들지 않는다.

## Notifications (외부 채널 — opt-in)

알림은 기본적으로 *비활성*이다. 사용자가 "Slack MCP에 알림 보내" / "이 갭을 메일로" 같이 명시한 경우에만 작동한다.

1. 모든 외부 메시지는 보내기 전에 사용자에게 미리보기를 보여주고 명시적 yes를 받는다.
2. 메시지 본문에 조문·판례 인용이 들어가면 "본 인용은 <fetched_at> 시점의 스냅샷이다. 외부 송신 전에 사용자가 재확인할 것"이라는 한 줄 면책을 raw_quote와 함께 첨부하고 추가 동의를 받는다.
3. 일괄 송신을 한 번에 묶지 않는다. 한 갭, 한 송신, 한 확인.

자동 송신(cadence·시간 기반)은 본 스킬의 기본 동작이 아니다. 어떤 cadence가 도달해도 사용자 확인 없이는 외부로 한 글자도 나가지 않는다.

## Pattern surfacing

패턴 갭(같은 slot_a_id에 묶인 ≥3건)이 만들어지면 다음을 자동으로 본문에 포함:

```markdown
### 패턴 갭 <pattern-id>

- 공통 출처(slot A): <slot_a_id>
- 묶인 갭: <gap-id 목록>
- 최초 발견: <date>
- 권장 후속:
  - 단발 수정이 아니라 dataset-level 점검을 권장한다.
  - `/launcelot-lawyer-pro:policy-redraft <pattern-id>`로 공통 수정 방향을 한 번에 받은 뒤,
    `/launcelot-lawyer-pro:policy-diff <slot_a_id> library`를 다시 돌려 누락을 확인하라.
  - 같은 슬롯 A에 반복적으로 걸리는 라인 패턴이면 `patterns.yaml`에 패턴 메타로 등록할지 사용자에게 한 번 확인한다.
```

## Persistence

- `gap-tracker.yaml`: 본체. 모든 갭의 상태·이력 및 `slot_a_snippet` 객체.
- `gap-tracker.audit.log`: append-only. 모든 add/close/accept-risk/unpublish/snooze 액션의 시점·행위자·이유·이전 상태·이후 상태. 본 로그는 사용자가 직접 편집하지 않는다.
- `patterns.yaml`: 패턴 갭의 메타. 패턴이 close되는 조건(모든 묶인 갭이 close 상태가 되면 패턴도 close)을 본 파일이 관리한다.

## Failure handling

- `gap-tracker.yaml` 손상 시: 마지막 정상 백업(`gap-tracker.yaml.bak.<…>`)을 제시하고 사용자에게 복원 vs 새로 시작을 묻는다. 자동 복원하지 않는다.
- policy-diff handoff에 `slot_a_snippet.raw_quote`가 비어있으면: 갭으로 받지 않는다(위 add 동작 1번). 사용자에게 "이 finding은 snippet이 없어 트래커에 등록할 수 없다. policy-diff를 재실행해 slot A의 snippet을 확보하라"라고 회신.

## What this skill does NOT do

- 사용자 대본을 수정하지 않는다.
- 외부 메시지를 자동 송신하지 않는다.
- 조문·판례 인용의 현행성을 자체적으로 판단하지 않는다(policy-diff가 snippet-protocol로 fetch한 결과를 그대로 보존). 외부 스킬(`launcelot-lawyer` 포함)에 위임하지 않는다.
- `launch-review` 결정 로그를 변경하지 않는다. 두 트래커는 분리되어 있고 cross-reference만 한다.
- 사용자 프로필·스타일·리스크 감수도·Hard rules로 우선순위를 바꾸지 않는다 — 본 플러그인은 그런 설정을 받지 않는다.
