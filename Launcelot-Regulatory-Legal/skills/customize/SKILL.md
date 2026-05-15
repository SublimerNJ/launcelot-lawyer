---
name: customize
description: >
  Launcelot-Regulatory-Legal의 watchlist, materiality threshold, 대본 라이브러리 경로,
  cadence, gap 트래커 운영 정책, 인테그레이션을 cold-start-interview를 다시 돌리지
  않고 한 항목씩 수정한다. 사용자가 "regulatory 프로필 바꿔", "watchlist에 변협 추가",
  "Tier 1 정의 수정", "라이브러리 경로 변경", "cadence 매일에서 주간으로", "customize
  regulatory"라고 말할 때 실행한다.
argument-hint: "[--field <섹션명>] [--show]"
---

# /launcelot-regulatory-legal:customize

This skill edits the practice profile in `~/.claude/plugins/config/launcelot-regulatory-legal/CLAUDE.md` one field at a time. It never re-runs the full cold-start.

## Run order

1. Confirm the config file exists. If not, redirect to `/launcelot-regulatory-legal:cold-start-interview`.
2. If `--show` was passed, print the current value of the requested field (or the whole profile) and stop.
3. If `--field <name>` was passed, jump straight to that section.
4. Otherwise, list the editable sections and ask which to change. Editable sections:
   - Attorney profile (mirror sync from Launcelot-Product-Legal if installed)
   - Channel profile (mirror sync)
   - Watchlist → 출처 add / remove / replace / filter 변경 / cadence override
   - Materiality threshold → Tier 1 / Tier 2 / Tier 3 정의 수정
   - Cadence → global / quiet hours / Tier 1 channel / digest output path
   - Script library → 위치 add/remove, indexing policy, slug rule, re-index trigger
   - Gap tracker policy → auto-ingest, aging thresholds, closure rules, notification recipient
   - Integrations (re-probe only, equivalent to `cold-start-interview --check-integrations`)
5. Read back the current value.
6. Ask the new value.
7. Back up the config file to `CLAUDE.md.bak.<ISO-date>`.
8. Apply the edit. Update `Last updated:` at top.
9. Echo a short summary: section, old (first 80 chars), new (first 80 chars).

## Edit conventions per section

### Watchlist

- `add <source>`: ask URL, operator, trust tier (공식 / 준공식 / 민간 — 사용자가 명시적으로 선택), filter, cadence override, reason. New entry appended.
- `remove <source>`: confirm. On removal, scan recent digests for items from this source; if any are still in `_unresolved.md`, warn the user and require explicit `--force` to remove anyway.
- `replace <source-field> <value>`: e.g., 필터만 좁히기.
- Coverage interaction: 사용자가 Tier 1 정의에 포함된 카테고리의 마지막 출처를 제거하면 한 번 경고하고 진행한다.

### Materiality threshold

- 각 Tier는 (checklist, raw text) 쌍. checklist 항목은 enum, raw는 자유 텍스트.
- raw 텍스트를 비울 수 없음. 비우려는 시도는 거부하고 새 텍스트를 요구한다.
- Tier 1 정의를 좁히면 `_unresolved.md`의 Tier 1 항목 중 새 정의에 더 이상 해당하지 않는 항목들의 처리 방식을 묻는다 (Tier 2로 강등 / 그대로 둠 / 제거).

### Cadence

- `global`: enum (daily / 3-day / weekly / on-demand). 변경 즉시 `_index.yaml`의 다음 cadence 예정 재계산.
- `quiet hours`: 자유 텍스트(예: "22:00–08:00 KST").
- `Tier 1 channel`: enum (chat / Slack MCP / email draft / file log only). Slack MCP를 선택하는 경우 인테그레이션 가용성 확인. 비가용이면 변경 거부하고 file log로 폴백.
- `digest output path`: 절대 경로 또는 `~/...` 형식. 존재하지 않는 디렉토리면 생성할지 묻는다.

### Script library

- 위치 추가 시 그 위치가 현재 접근 가능한지 즉시 확인(MCP 가용성). 비가용이면 `<not yet accessible>`로 등록.
- `indexing policy`, `slug rule`, `re-index trigger`는 enum.
- 라이브러리 위치 변경 시: 기존 슬러그가 새 위치에서 더 이상 유효하지 않을 수 있음. 영향 가능 슬러그 수를 보고하고 진행 여부 확인.

### Gap tracker policy

- `auto-ingest`: enum (Tier-1 only / all / manual). 변경 즉시 발효.
- `aging thresholds`: 정수 일 수 (콤마 구분 또는 단일). 임계값 변경 시, 현재 open 갭의 aging breach 재계산.
- `closure rules`: checklist (대본 수정 완료 / 비공개 처리 / 리스크 수용). 항목 제거 시 그 closure 경로로 닫힌 과거 갭을 어떻게 처리할지 묻는다(닫힌 상태 유지 권장).
- `notification recipient`: 자유 텍스트. 외부 송신 정책은 그대로(사전 미리보기 + 명시적 yes).

### Integrations

- 사용자가 수동으로 가용/비가용 플래그를 변경할 수 없음. 본 섹션 수정은 `--check-integrations`와 동일하게 자동 probe.
- 단 `fallback note`는 사용자가 직접 편집 가능.

## Edit safety

- 본 스킬은 한 번에 한 섹션만 수정한다. 사용자가 동시 변경을 요청하면 첫 변경을 마치고 확인 후 두 번째로 진행.
- 빈 값으로 비울 수 없는 필수 필드: Watchlist의 trust tier·URL, Materiality threshold 각 Tier의 raw, Cadence의 global·Tier 1 channel, Script library의 slug rule, Gap tracker policy의 모든 필드. 빈 값 시도는 거부하고 새 값을 요구한다.
- 본 스킬은 `~/.claude/plugins/config/launcelot-regulatory-legal/` 밖의 어떤 파일도 만지지 않는다.

## Cross-plugin sync

`## Attorney profile` 또는 `## Channel profile`을 본 스킬에서 수정하면, Launcelot-Product-Legal의 같은 섹션과 어긋날 수 있다. 본 스킬은 변경을 적용한 뒤 다음을 묻는다:

> 같은 변경을 Launcelot-Product-Legal config(`~/.claude/plugins/config/launcelot-product-legal/CLAUDE.md`)에도 반영할까? (반영 / 본 플러그인에만 적용 / 직접 결정 안 함)

자동 반영하지 않는다. 두 플러그인의 프로필이 갈라질 수 있음을 사용자가 의식적으로 알게 한다.

## What this skill does NOT do

- 피드를 끌어오지 않는다.
- 어떤 갭도 만들거나 닫지 않는다.
- 어떤 디제스트 파일도 수정하지 않는다(편집은 config 한정).
- 플러그인 버전 마이그레이션을 하지 않는다.
