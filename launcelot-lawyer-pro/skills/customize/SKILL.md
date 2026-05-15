---
name: customize
description: >
  Launcelot-Lawyer-Pro의 전 영역(변호사·채널 프로필, 발화 정책, 위험 감수도, Hard
  rule, substantiation, watchlist, materiality threshold, cadence, 대본 라이브러리,
  갭 트래커 정책, 인테그레이션)을 cold-start-interview를 다시 돌리지 않고 한 항목씩
  수정한다. 사용자가 "프로필 바꿔", "단정 표현 정책 수정", "위험 감수도 낮춰",
  "watchlist에 변협 추가", "Tier 1 정의 수정", "Hard rule 추가", "라이브러리 경로
  변경", "cadence 매일에서 주간으로", "customize"라고 말할 때 실행한다.
argument-hint: "[--field <섹션명>] [--show] [--module <N>]"
---

# /launcelot-lawyer-pro:customize

This skill edits the practice profile in `~/.claude/plugins/config/launcelot-lawyer-pro/` one field at a time. It never re-runs the full cold-start.

## Run order

1. Confirm the config directory exists. If not, redirect to `/launcelot-lawyer-pro:cold-start-interview`.
2. `--show`: print the current value of the requested field (or the whole profile if no field) and stop.
3. `--field <name>`: jump straight to that section.
4. `--module <N>`: jump to a module (mirror of cold-start modules 1..8).
5. Otherwise, list the editable sections and ask which to change.
6. Read back the current value before any edit.
7. Ask the new value.
8. Back up the affected file(s) to `<name>.bak.<ISO-date>` before writing.
9. Apply the edit. Update `Last updated:` in CLAUDE.md if any field there changed.
10. Echo a short summary: file, section, old (first 80 chars), new (first 80 chars).

## Editable sections

Mapped to the underlying file. Edits are scoped to one section at a time.

### CLAUDE.md
- Attorney profile
- Channel profile
- Speech policy → Real-name / Pending-case / Speculative-statement / Absolute-claim
- Risk calibration → Appetite (label) / Past disputes (list ops)
- Hard rules → Never (list ops) / Must escalate (list ops)
- Substantiation → Acceptable sources / Case citation style / Statute quotation style / Internal library
- Cadence → Global / Quiet hours / Tier 1 channel / Digest output path
- Gap tracker policy → Auto-ingest / Aging thresholds / Closure rules / Notification recipient
- Integrations → re-probe only (equivalent to `cold-start-interview --check-integrations`); fallback notes are hand-editable.

### watchlist.yaml
- 출처 list ops: `add <source>` / `remove <source>` / `replace <source-field> <value>`.

### materiality.yaml
- Tier 1 / Tier 2 / Tier 3 — checklist edit + raw text edit.

### script-library.yaml
- Locations list ops, Indexing policy, Slug rule, Re-index trigger.

## Edit conventions

### Labeled fields (Speech policy 4종, Risk appetite)

Accept only the enum values defined in cold-start. If the user types something else, ask whether to record it as `기타` with the raw text preserved as evidence.

### List fields (Past disputes, Hard never, Hard must-escalate, Acceptable sources, watchlist, script-library locations)

Three operations: `add <item>` / `remove <#>` / `replace <#> <new value>`. Show numbered list before asking.

### Tier definitions (materiality.yaml)

Each tier is `(checklist, raw text)`. raw 텍스트는 비울 수 없음. 비우려는 시도는 거부하고 새 값을 요구한다.

### Cadence

- `global`: enum (daily / 3-day / weekly / on-demand). 변경 즉시 디제스트 다음 cadence 예정 재계산.
- `Tier 1 channel`: enum. Slack MCP를 선택할 때 가용성 확인. 비가용이면 변경 거부, file log로 폴백 제안.
- `digest output path`: 디렉토리 미존재 시 생성 여부 묻기.

### Gap tracker policy

- `auto-ingest`: enum (Tier-1 only / all / manual).
- `aging thresholds`: 정수 일 수.
- `closure rules` checklist: 항목 제거 시 그 경로로 닫힌 과거 갭 처리 옵션 묻기(닫힘 상태 유지 권장).
- `notification recipient`: 자유 텍스트. 외부 송신 정책(사용자 사전 미리보기 + yes)은 본 스킬에서 변경 불가.

### Integrations

수동으로 가용/비가용 플래그를 바꿀 수 없음. 본 섹션 변경은 자동 probe(`--check-integrations`와 동일). 단 `fallback note`는 사용자가 직접 편집 가능.

## Safety

- 한 번에 한 섹션만 수정. 사용자가 여러 변경을 요청하면 첫 변경을 마치고 확인 후 다음으로.
- 필수 필드를 비우려는 시도는 거부:
  - Speech policy의 네 라벨
  - Materiality threshold 각 Tier의 raw
  - Cadence의 global·Tier 1 channel
  - Script library의 slug rule
  - Gap tracker policy의 모든 필드
  - Watchlist 항목의 trust tier·URL
- `~/.claude/plugins/config/launcelot-lawyer-pro/` 밖의 파일은 일체 만지지 않음.

## Hard rule audit interaction

`Hard rules → Never` 또는 `Hard rules → Must escalate`를 수정하면 `reviews/_hard_rule_log.yaml`에 해당 rule이 과거에 몇 번 매칭됐는지 보여준 뒤 변경을 진행한다. 사용자가 rule을 제거하려 할 때 매칭 이력이 있으면 마지막에 한 번 더 확인.

## Coverage interaction

- Tier 1 정의를 좁히면 `digests/_unresolved.md`의 Tier 1 항목 중 새 정의에서 빠지는 항목들의 처리(Tier 2 강등 / 그대로 / 제거)를 묻는다.
- Watchlist에서 어떤 출처를 지우려는데 그 출처가 현재 사용자의 Tier 1 정의에 포함된 카테고리의 유일한 출처면, 한 번 경고하고 진행.

## What this skill does NOT do

- 피드를 끌어오지 않는다.
- 어떤 갭도 만들거나 닫지 않는다.
- 어떤 디제스트 파일·diff 파일·redraft 파일·리뷰 파일도 직접 수정하지 않는다. 편집은 config 한정.
- 플러그인 버전 마이그레이션을 하지 않는다.
