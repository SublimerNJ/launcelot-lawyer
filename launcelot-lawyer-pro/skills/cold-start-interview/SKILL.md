---
name: cold-start-interview
description: >
  Launcelot-Lawyer-Pro의 단일 콜드스타트 인터뷰. 변호사 신분·채널 프로필·발화 정책·
  위험 감수도·인용 소스·watchlist·materiality threshold·cadence·대본 라이브러리·갭
  트래커 정책·인테그레이션을 한 번에 학습해 ~/.claude/plugins/config/launcelot-lawyer-pro/
  CLAUDE.md와 하위 yaml들을 작성한다. 첫 설치 시, "처음부터 다시 설정", "set up",
  "cold start"라고 말할 때, 또는 CLAUDE.md가 없거나 플레이스홀더가 남아있을 때 실행한다.
argument-hint: "[--redo | --check-integrations | --module <N>]"
---

# /launcelot-lawyer-pro:cold-start-interview

This is the only setup skill. It writes the entire `~/.claude/plugins/config/launcelot-lawyer-pro/` directory: the master profile `CLAUDE.md` and the structured sidecars (`watchlist.yaml`, `materiality.yaml`, `script-library.yaml`, the empty `gap-tracker.yaml`, etc.).

It is the only skill that should run on a fresh install. Every other skill reads what this skill writes.

## When to run

- First install (config directory missing).
- `CLAUDE.md` exists but contains `[PLACEHOLDER]` markers.
- User says "set up", "onboard me", "처음부터 다시", "cold start".
- `--redo`: full re-interview, backup-then-overwrite.
- `--check-integrations`: re-probe MCPs/sister skills and update Integrations only.
- `--module <N>`: jump to a single module (1..8) for partial re-run.

## Output targets

Root: `~/.claude/plugins/config/launcelot-lawyer-pro/`

- `CLAUDE.md` — master profile (sections below).
- `watchlist.yaml` — sources to monitor.
- `materiality.yaml` — Tier 1/2/3 definitions.
- `script-library.yaml` — library locations and indexing policy.
- `gap-tracker.yaml` — empty tracker, ready for use.
- `patterns.yaml` — empty.
- `comments/` — directory only (populated later by `comments` skill).
- `matters/` — directory only (populated later by `matter-workspace`).
- `reviews/`, `digests/`, `diffs/`, `redrafts/` — empty directories.

Create the directory if absent. Back up existing files to `<name>.bak.<ISO-date>` before overwriting on `--redo`.

## Interview structure — 8 modules

Run in order. After each module, mirror back what was captured ("이대로 저장할까?") before moving on. Korean for the questions; bilingual config (English headers + Korean content) so other skills can grep structural section names.

### Module 1 — Attorney identity

1. Bar admission (대한변호사협회). Multiple if applicable.
2. Local bar (서울지방변호사회 등).
3. Firm or sole. Firm name.
4. Stated practice areas on the channel.
5. Titles to feature or disclaim (전직 검사, 전직 판사, 전문 변호사 인증 등). Capture exactly how the channel currently presents them.
6. Admission year.

Write to `CLAUDE.md` → `## Attorney profile`.

### Module 2 — Channel context

1. Channel name(s) and platforms (YouTube long-form / Shorts / 블로그 / SNS / 뉴스레터). List all surfaces.
2. Format mix: 사건해설 / 뉴스반응 / Q&A / 인터뷰 / 쇼츠 / 기타 (percentages).
3. Tone: 차갑게 기술적 / 분석형 / 어그로형 / 사실보도형 / 위로·공감형 / 혼합.
4. Target viewer: 일반 시청자 / 잠재 의뢰인(특정 상황) / 전문가.
5. Typical length and cadence.

Write to `CLAUDE.md` → `## Channel profile`.

### Module 3 — Speech policy (calibration core)

One question at a time. For each, store both the label and the user's raw words.

1. Real-name policy: 공인 한정 / 공인+확정 공개판결 당사자 / 언론 보도 인물까지 / 사실 정확하면 광범위 / 절대 실명 사용 안 함.
2. Pending-case policy: 절대 안 함 / 익명화 시 허용 / 보도된 사건 허용 / 광범위 허용.
3. Speculative-statement policy: 엄격(판결문 인정 사실만) / 중도(분석으로 한정자 부착) / 느슨(직접 평가 허용).
4. Absolute-claim policy on outcomes: 절대 안 함 / 일반론으로 학술적일 때 / 광범위.

Write to `CLAUDE.md` → `## Speech policy`.

### Module 4 — Risk calibration

1. Risk appetite: 보수적 / 중도 / 공격적.
2. Past disputes (cease-and-desist, takedown, 변협 진정, 명예훼손 고소/소송, 민사 청구). Each: 일자·요약·결과.
3. Hard "never" lines — 채널이 절대 만들지 않는 콘텐츠 (예: 답변서 자작 가이드, 기획고소 매뉴얼).
4. Hard "must escalate" — 사람이 무조건 확인해야 하는 카테고리.

Write to `CLAUDE.md` → `## Risk calibration` (appetite + past disputes) and `## Hard rules` (never + must-escalate).

### Module 5 — Substantiation sources

1. Acceptable citation sources for legal claims: 대법원 종합법률정보 / 국가법령정보센터 / 헌재 / 학술논문 / 1차 보도 only / 변협 자료.
2. Case citation style: 풀 인용 / 약식.
3. Statute quotation: 전문 인용 + 조항 / 의역 + 조항 / 의역만.
4. Internal substantiation library: 위치(Google Drive / Notion / NJsidian wiki / 로컬).

Write to `CLAUDE.md` → `## Substantiation`.

### Module 6 — Watchlist, Materiality, Cadence (regulatory core)

#### 6a. Watchlist

Read `references/source-catalog.md`. For each catalog source, ask:
- include / include with narrower filter / exclude.
- per-source filter (free text), cadence override (optional).

Allow user-added sources. For each, require: URL, operator, trust tier (공식 / 준공식 / 민간 — 사용자가 명시적으로 선택), reason. Trust tier is never auto-assigned.

Write to `watchlist.yaml`:
```yaml
sources:
  - name: <…>
    url: <…>
    operator: <…>
    trust_tier: <공식 | 준공식 | 민간>
    filter: <…>
    cadence_override: <…>
    reason: <…>
```

#### 6b. Materiality threshold

Three tiers. For each: checklist + raw user text. raw 텍스트 비울 수 없음.

- Tier 1 (즉시 알림): 어느 변경에 cadence를 무시하고 즉시 알릴지.
- Tier 2 (정기 디제스트): 디제스트에 포함.
- Tier 3 (로그만).

Write to `materiality.yaml`:
```yaml
tier1:
  checklist: [...]
  raw: |
    <user text>
tier2: { ... }
tier3: { ... }
```

#### 6c. Cadence

- global: daily / 3-day / weekly / on-demand.
- quiet hours.
- Tier 1 channel: chat / Slack MCP / email draft / file log only. Slack MCP 비가용이면 변경 거부 후 file log 폴백.
- digest output path. 기본 `~/.claude/plugins/config/launcelot-lawyer-pro/digests/<YYYY-MM-DD>.md`.

Write to `CLAUDE.md` → `## Cadence`.

### Module 7 — Script library, Gap tracker policy

#### 7a. Script library

- Locations (복수 가능): 로컬 폴더 / njsidian-wiki prefix / Notion / Google Drive / 기타.
- Indexing policy: 전체 / 발행분 only / 태그된 것 only.
- Slug rule.
- Re-index trigger: Tier 1 변경 시 자동 / 수동.

Write to `script-library.yaml`.

#### 7b. Gap tracker policy

- Auto-ingest from policy-diff: Tier-1 only (기본) / all / manual.
- Aging thresholds (일): 기본 7/14/30.
- Closure rules (checklist): 대본 수정 완료 / 비공개 / 리스크 수용.
- Notification recipient (외부 송신은 항상 사용자 사전 미리보기·yes 후 송신).

Write to `CLAUDE.md` → `## Gap tracker policy`. Initialize empty `gap-tracker.yaml` and `patterns.yaml`.

### Module 8 — Integrations probe

Probe and record `available` / `not available` + fallback note:

- `launcelot-lawyer` skill — 조문·판례 실존 검증. **이 한 가지는 비가용 시 본 플러그인의 모든 검증성 출력이 미검증 태그로 마킹된다는 점을 사용자에게 명확히 고지한다.**
- `wiki` / `njsidian-wiki` MCP — 라이브러리·과거 검토 접근.
- `kmj`, `kmg`, `sbh`, `ldg`, `wja` 스킬 — 화자별 보이스, redraft 후속.
- `koreanizer`, `humanizer` 스킬 — 톤 마감.
- Slack MCP, Google Drive MCP, Notion MCP — 라이브러리·알림.
- 한국 공식 출처 직접 MCP가 있는지 — 없으면 reg-feed-watcher가 launcelot-lawyer 경유.

Write to `CLAUDE.md` → `## Integrations`.

## CLAUDE.md template

```markdown
# Launcelot-Lawyer-Pro — Practice profile

Last updated: <ISO date>
Cold-start version: 1.0.0
Config root: ~/.claude/plugins/config/launcelot-lawyer-pro/

## Attorney profile
<…>

## Channel profile
<…>

## Speech policy

### Real-name policy
Label: <…>  ·  Raw: <…>

### Pending-case policy
Label: <…>  ·  Raw: <…>

### Speculative-statement policy
Label: <…>  ·  Raw: <…>

### Absolute-claim policy
Label: <…>  ·  Raw: <…>

## Risk calibration

- Appetite: <보수적 | 중도 | 공격적>
- Past disputes:
  - <date> — <summary> — <outcome>

## Hard rules

- Never:
  - <line>
- Must escalate:
  - <line>

## Substantiation

- Acceptable sources: <list>
- Case citation style: <full | 약식>
- Statute quotation style: <full | paraphrase + article | paraphrase only>
- Internal library: <path/URL>

## Cadence

- Global: <…>
- Quiet hours: <…>
- Tier 1 channel: <…>
- Digest output path: <…>

## Gap tracker policy

- Auto-ingest: <…>
- Aging thresholds: <…>
- Closure rules: <…>
- Notification recipient: <…>

## Integrations

- launcelot-lawyer: <available | not available> — <fallback note>
- njsidian-wiki: <…>
- kmj / kmg / sbh / ldg / wja: <…>
- koreanizer / humanizer: <…>
- Slack MCP: <…>
- Google Drive / Notion MCP: <…>
- Direct Korean-regulator MCP: <…>

## Validator wiring

When any skill runs:

1. Load this file plus the relevant sidecar (watchlist.yaml / materiality.yaml / script-library.yaml).
2. Apply `## Speech policy` thresholds before scoring.
3. Apply `## Risk calibration` to set the band floor (보수적 raises everything one step; 공격적 lowers borderline within 회색지대·가능 only, never to 안전, never affects 명백).
4. Defer to `launcelot-lawyer` for all 조문·판례 lookups. Never assert a statute or judgment from model knowledge.
5. Honor every entry in `## Hard rules`.
6. Read matters from `matters/` if a matter is active (see matter-workspace).
```

## Interview style

- One question per turn. Korean.
- Never skip silently. `<not provided>` is the only allowed empty value, and downstream skills are designed to behave conservatively when they see it.
- Module 1·2 mirror prompt: if a previous Launcelot-Product-Legal or Launcelot-Regulatory-Legal config is found (legacy), offer to import the Attorney/Channel sections and confirm field by field.
- Module 8 probe results: never let the user manually set anything to `available`. Trust the probe.

## What this skill does NOT do

- Does not validate any script. That is `marketing-claims-review`, `feature-risk-assessment`, `launch-review`, `is-this-a-problem`.
- Does not verify any 조문·판례. That is `launcelot-lawyer`.
- Does not pull feeds, build a digest, or run a diff. Those are `reg-feed-watcher`, `policy-diff`, `gap-surfacer`.
- Does not touch any file outside `~/.claude/plugins/config/launcelot-lawyer-pro/`.
