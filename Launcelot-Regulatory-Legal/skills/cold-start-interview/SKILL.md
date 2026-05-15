---
name: cold-start-interview
description: >
  Launcelot-Regulatory-Legal의 watchlist(감시 죄목·규정), materiality threshold
  (어떤 변경을 신호로 볼지), 대본 라이브러리 위치(어디에 사용자 대본이 있는지),
  피드 cadence(언제 검사할지), 갭 트래커 운영 정책을 인터뷰로 학습한다. 첫 설치
  시, "regulatory 처음부터 다시 설정" 요청 시, 또는 ~/.claude/plugins/config/
  launcelot-regulatory-legal/CLAUDE.md가 없거나 플레이스홀더가 남아있을 때 실행한다.
argument-hint: "[--redo | --check-integrations]"
---

# /launcelot-regulatory-legal:cold-start-interview

This skill writes `~/.claude/plugins/config/launcelot-regulatory-legal/CLAUDE.md`, the practice profile every other skill in this plugin reads. It is the only skill that should run on a fresh install.

## When to run

- First install (config missing).
- Config file present but contains `[PLACEHOLDER]` markers.
- User says "set up regulatory", "regulatory 처음부터", "cold start regulatory".
- With `--redo`: full re-interview, backup-then-overwrite.
- With `--check-integrations`: re-probe MCPs and update the integration section only.

## Output target

`~/.claude/plugins/config/launcelot-regulatory-legal/CLAUDE.md`

If the directory does not exist, create it. If a prior file exists, back it up to `CLAUDE.md.bak.<ISO-date>` before writing.

## Interview structure

Seven modules. Run in order. After each, mirror back what was captured and ask "이대로 저장할까?" before moving on.

### Module 1 — Practice context

Goal: tie this plugin to the same attorney/channel context that Launcelot-Product-Legal already learned, so the validators agree on who they are protecting.

1. Read `~/.claude/plugins/config/launcelot-product-legal/CLAUDE.md` if it exists. Pull `## Attorney profile` and `## Channel profile`. Ask the user: "이 정보를 그대로 가져와도 되나? 다르게 등록할 항목이 있나?" Default is to mirror; allow override per field.
2. If Launcelot-Product-Legal config does not exist, ask Module 1 of that plugin's cold-start questions here (attorney identity + channel profile) and save the same answers in this plugin's config under `## Attorney profile` and `## Channel profile`. Note in the file: `Source: 동시 입력 (Launcelot-Product-Legal 미설치)`.

Write to: `## Attorney profile`, `## Channel profile`.

### Module 2 — Watchlist (감시 대상)

Goal: define exactly which 법령, 판례 영역, 행정청 공지를 본 플러그인이 감시할지.

Read `references/source-catalog.md` (this skill's plugin, sibling `references/` folder). For each source in the catalog, ask one of:
- include (감시)
- include with a narrower filter (예: "형법 중 명예훼손·모욕·무고만")
- exclude

For each `include`, capture:
- Source name (catalog entry).
- Filter (free text). Default: 카탈로그가 권장하는 범위.
- Cadence (per-source override). 기본값은 Module 4의 글로벌 cadence를 따른다.

Allow the user to add sources not in the catalog. For each user-added source, require:
- URL
- Operator (공식기관명)
- Trust tier — 공식 / 준공식 / 민간 (사용자가 명시적으로 선택)
- Reason for adding (free text, will be quoted back by reg-feed-watcher in the digest)

Write to: `## Watchlist`.

### Module 3 — Materiality threshold

Goal: filter signal from noise. Most regulatory feed entries do not affect a YouTube script. Decide what does.

Ask three questions:

1. **상시 알림 (Tier 1).** 어떤 변경이 발생하면 즉시(피드 cadence 무시) 사용자에게 알릴 항목인가? 선택지:
   - 변호사법·변협 광고규정의 본문 개정
   - 형법 명예훼손·모욕·정통망법 70 조문 개정 또는 신착 대법원 판례 중 판시사항 변경
   - 헌재의 위헌·합헌 결정 (위 죄목 영역)
   - 사용자가 명시적으로 지정한 키워드
2. **정기 디제스트 (Tier 2).** 즉시 알림은 아니지만 정기 디제스트에 포함할 항목.
3. **로그만 (Tier 3).** 디제스트에는 빼지만 시계열 로그에는 남길 항목.

For each tier, store both a checklist (선택된 항목들)와 자유 텍스트 정의. reg-feed-watcher와 policy-diff는 이 정의를 그대로 인용해 분류 근거로 표시한다.

Write to: `## Materiality threshold`.

### Module 4 — Cadence

Goal: 피드를 언제, 얼마나 자주 끌어올지.

1. Global cadence: daily / 3-day / weekly / on-demand only.
2. Quiet hours: 사용자가 알림을 받지 않을 시간대 (예: 22:00–08:00 KST).
3. Tier 1 escalation channel: 즉시 알림이 어디로 가는가 (chat / Slack MCP / 이메일 draft / 단순 파일 로그).
4. Digest output: 디제스트가 출력될 경로 (기본값: `~/.claude/plugins/config/launcelot-regulatory-legal/digests/<YYYY-MM-DD>.md`).

Write to: `## Cadence`.

### Module 5 — Script library (정책 라이브러리에 대응하는 사용자 영역)

Goal: 대본이 어디에 있는지 알아야 신착 변경분과 대조할 수 있다.

1. Storage location(s). 복수 가능:
   - 로컬 폴더 경로
   - njsidian-wiki 경로(prefix)
   - Notion 페이지/DB ID
   - Google Drive 폴더 ID
   - 기타
2. Indexing policy:
   - 모든 대본을 인덱스
   - 발행된(공개된) 대본만 인덱스
   - 사용자가 명시적으로 태깅한 대본만
3. Slug rule: 한 대본을 식별하는 키 (예: 파일명, wiki path, 영상 URL).
4. Re-index trigger: Tier 1 변경 발생 시 자동 재인덱스 / 사용자 수동 재인덱스만.

If MCPs to access the library are not connected, record the location anyway and mark as `<not yet accessible>` so reg-feed-watcher's digest can ask the user to connect.

Write to: `## Script library`.

### Module 6 — Gap tracker policy

Goal: policy-diff에서 발견된 위반 가능성이 어떻게 트래커에 들어오고 어떻게 닫히는지 정의.

1. Auto-ingest. policy-diff 결과를 자동으로 gap-tracker.yaml에 적재할 것인가? (기본: 예. 단 Tier 1만 자동, Tier 2·3는 사용자 확인.)
2. Aging thresholds. open gap이 며칠 이상 미해소면 reminder를 띄울 것인가 (예: 7일·14일·30일).
3. Closure rules. 다음 중 어느 것이 닫힘 조건인가:
   - 대본 수정 완료 (revision 기록)
   - 대본 비공개 처리
   - 리스크 수용 결정 (사유 기록 필수)
4. Notification policy. 알림 채널이 있다면 (Slack MCP 등), 누가 알림을 받는가. 기본은 사용자 본인.

Write to: `## Gap tracker policy`.

### Module 7 — Integrations probe

Probe and record `available` / `not available`:

- `launcelot-lawyer` skill — 조문·판례 실존 검증. 핵심 위임 대상.
- `wiki` / `njsidian-wiki` MCP — 대본 라이브러리 접근.
- `Launcelot-Product-Legal` config 파일 존재 여부 — Module 1 미러링 가능성.
- Slack MCP — 알림 채널.
- Google Drive / Notion MCP — 대본 라이브러리 접근.
- 한국 공식 출처에 대한 직접 접근 MCP가 있는지 (없으면 launcelot-lawyer 웹 검증 경유).

Write to: `## Integrations`. For each `not available` item, record the fallback (no auto-notification / manual paste of digest items / etc.).

## Config file template

```markdown
# Launcelot-Regulatory-Legal — Practice profile

Last updated: <ISO date>
Cold-start version: 0.1.0

## Attorney profile
<…mirrored from Launcelot-Product-Legal or freshly captured…>

## Channel profile
<…>

## Watchlist

### Source: <name>
- URL: <…>
- Operator: <…>
- Trust tier: <공식 | 준공식 | 민간>
- Filter: <…>
- Cadence override: <…>
- Reason: <…>
(반복)

## Materiality threshold

### Tier 1 — 즉시 알림
- Checklist: <selected items>
- Raw: <user's own words>

### Tier 2 — 정기 디제스트
- Checklist: <…>
- Raw: <…>

### Tier 3 — 로그만
- Checklist: <…>
- Raw: <…>

## Cadence

- Global: <daily | 3-day | weekly | on-demand>
- Quiet hours: <…>
- Tier 1 channel: <…>
- Digest output path: <…>

## Script library

- Locations:
  - <type>: <path or ID> <(accessible | not yet accessible)>
- Indexing policy: <…>
- Slug rule: <…>
- Re-index trigger: <…>

## Gap tracker policy

- Auto-ingest: <Tier-1 only | all | manual>
- Aging thresholds: <e.g., 7 / 14 / 30>
- Closure rules: <list>
- Notification recipient: <…>

## Integrations

- launcelot-lawyer: <available | not available> — <fallback note>
- njsidian-wiki: <…>
- Launcelot-Product-Legal config: <found | not found>
- Slack MCP: <…>
- Google Drive / Notion MCP: <…>
- Direct Korean-regulator MCP: <…>

## Validator wiring

When `reg-feed-watcher`, `policy-diff`, `gap-surfacer`, or `policy-redraft` runs:

1. Load this file first.
2. Apply `## Materiality threshold` to filter feed entries.
3. Defer every 조문·판례 lookup to `launcelot-lawyer`. Never assert a statute or judgment from model knowledge.
4. Read `## Watchlist` to know which sources to pull and how to label trust tier.
5. Honor `## Gap tracker policy` for any new gap.
```

## Interview style

- One question per turn. Korean for questions, bilingual config file (English structural headers + Korean content) so other skills can grep.
- "Skip" → `<not provided>`. Never guess.
- Module 1 mirroring: never silently mirror Launcelot-Product-Legal. Always show the values and ask.

## What this skill does NOT do

- Does not pull feeds. That is `reg-feed-watcher`.
- Does not check anything against statute. That is `policy-diff` (and via it, `launcelot-lawyer`).
- Does not touch any file outside `~/.claude/plugins/config/launcelot-regulatory-legal/`.
