---
name: policy-diff
description: >
  한국 법령 조문·판례·변협 공지·공정위 가이드라인 등 규범 텍스트(슬롯 A)와 사용자
  유튜브 대본 또는 대본 라이브러리(슬롯 B)를 대조하여 위반 가능성을 라인 단위로
  추출하고, 매칭 죄목·조문·관련 판례·위험 등급·수정 방향을 제시한다. 사용자가
  "이 조문 vs 우리 대본 diff", "형법 X조 개정됐는데 영향 받는 대본", "이번
  판례가 우리 라이브러리 어디에 걸려?", "policy diff"라고 말하거나 reg-feed-watcher가
  Tier 1·2 변경을 핸드오프할 때 실행한다.
argument-hint: "[규범 슬러그 또는 텍스트] [대본 슬러그 또는 폴더 경로 또는 'library']"
---

# /launcelot-regulatory-legal:policy-diff

This skill performs the actual diff between Korean regulatory texts and the user's script(s). It is the workhorse skill of Launcelot-Regulatory-Legal. It does not verify whether the regulatory text it received is current — that is `launcelot-lawyer`'s job — but it does require launcelot-lawyer verification before any output line carries a statutory or case-law citation.

## Run order

1. Load `~/.claude/plugins/config/launcelot-regulatory-legal/CLAUDE.md`. If missing, redirect to `cold-start-interview`.
2. Resolve slot A — the regulatory input.
3. Resolve slot B — the script input.
4. Verify slot A through `launcelot-lawyer`. Halt the diff if launcelot-lawyer is unavailable; emit a banner and degrade gracefully (see `## Failure handling`).
5. Run the diff. Emit findings.
6. Persist results.
7. Hand off to `gap-surfacer` for any finding the user wants tracked.

## Inputs

### Slot A — Regulatory input

Accept any of:

- A 조문 reference (e.g., `형법 307조`, `정통망법 70조 1항`, `변호사 광고에 관한 규정 4조`). Skill fetches the current text via launcelot-lawyer.
- A 판례 reference (e.g., `대법원 2020. 11. 19. 선고 2020도5813 판결`). Skill fetches the 판시사항·판결요지 via launcelot-lawyer.
- Pasted regulatory text. Treated as user-asserted text; launcelot-lawyer is still asked to confirm currency and authenticity of the citation, but the diff proceeds against the pasted body.
- A reg-feed-watcher item ID (when policy-diff is invoked from a digest handoff). Loads the entry from the most recent `digests/<YYYY-MM-DD>.md`.
- A keyword set (e.g., `명예훼손 신착 판례`). Skill asks launcelot-lawyer for the top N items; user confirms which to diff.

For each, capture and persist:

- `slot_a.id` (slug or citation)
- `slot_a.source_tier` (공식 / 준공식 / 민간 / pasted)
- `slot_a.verification_status` (`verified-current` / `verified-but-superseded` / `unverified` / `not-found-by-launcelot-lawyer`)
- `slot_a.text` (the controlling text used for the diff)
- `slot_a.fetched_at`

### Slot B — Script input

Accept any of:

- A pasted script body.
- A file path (Read).
- A wiki slug (fetch via `njsidian-wiki`).
- A library directive (`library`, `library:published`, `library:tagged:<tag>`) — diffs against every script the config's `## Script library` indexes.

For library mode, iterate: one diff record per script. Concurrency is sequential by default to keep launcelot-lawyer load predictable.

## Diff method

For each script (slot B), walk it line by line. For each line:

1. Pre-filter: does the line plausibly implicate the death's-head categories that slot A addresses? Use the regulatory text's plain text to derive trigger tokens. Skip lines with zero trigger overlap.

2. Element decomposition: extract the elements the regulatory text requires.
   - For statutes: 행위 주체, 행위 객체, 행위 양태, 결과·결과발생 가능성, 위법성 조각 사유.
   - For 판례: 판시사항이 인정한 행위 패턴, 부정한 행위 패턴, 인정 요건.
   - For 광고규정·표시광고법: 금지되는 표현 유형, 허용 조건, 실증 요건.

3. Element matching: for each element, mark `present` / `absent` / `unclear` on the line. A `present-element` cluster sufficient to satisfy the regulatory text's prohibitory side raises a candidate finding. The skill never concludes a violation has occurred — only that the elements appear or do not appear in the line.

4. Defense window: for each candidate finding, list the defenses the regulatory text or surrounding doctrine allows.
   - 진실성·공익성 항변 (형법 310, 정통망법 70 3항)
   - 정당행위 (형법 20)
   - 인용의 정당한 범위 (저작권법 28)
   - 변호사 광고규정의 허용 단서
   - 표시광고법의 실증된 표현 단서

5. Risk band:
   - **명백**: 요건이 라인 안에 모두 갖춰져 있고, 항변 단서가 라인 또는 라인 인접에 보이지 않음.
   - **가능**: 요건이 갖춰져 있으나 항변 단서가 있다(진실성·공익성·정당범위 등).
   - **회색지대**: 요건 일부 결여 또는 적용 자체에 다툼.
   - **안전**: 요건 명백히 결여.

6. Apply user calibration from `~/.claude/plugins/config/launcelot-product-legal/CLAUDE.md` `## Risk calibration` if that plugin is installed. 보수적 → +1 step. 공격적 → -1 step within 회색지대·가능, never to 안전.

## Output (per line finding)

```markdown
### Finding <slug-B>:<line-N>

**Slot A:** <조문 또는 판례 식별자> <(verification: verified-current | unverified | …)>

**Slot B:** <script slug>, line <N>

**Quote (대본):** "<exact text>"

**Elements present:**
- <element 1>: <evidence in the line>
- <element 2>: <…>
**Elements absent / unclear:**
- <element>: <왜 결여 또는 불분명>

**Available defenses:**
- 진실성·공익성 (형법 310): <적용 가능성 한 줄>
- 정당행위 (형법 20): <…>
- 인용 (저작권법 28): <…>
- 변호사 광고규정 단서: <…>
- 기타: <…>

**Risk band:** <명백 | 가능 | 회색지대 | 안전>
**Calibration applied:** <보수적 +1 | 중도 | 공격적 -1>

**Suggested fix direction:** <한 줄, 본문 톤 유지 가능 여부 포함>
**Hand off to policy-redraft?** <yes | no — reason>
**Hand off to gap-surfacer?** <yes (proposed gap entry below) | no>

**Verification queue:**
- launcelot-lawyer: confirm <조문 또는 판례> currency at fetch time; reverify if older than 7 days.
```

## Output (per script summary)

```markdown
## Diff summary — <slug-B> against <slot_a.id>

- Lines analyzed: <N>
- Findings: 명백 <n> / 가능 <n> / 회색지대 <n>
- Defense-dependent findings: <n>
- Top three lines: <line numbers>
- Suggested next step: <policy-redraft <these line ids> | publish a 수정 후 게시 memo | open gap entries for <these>>
```

## Persistence

Two files per diff run plus index updates.

- `~/.claude/plugins/config/launcelot-regulatory-legal/diffs/<reg-slug>-<YYYY-MM-DD>-<HHMM>.md`
  Full per-line findings + per-script summary for the run. Append-only at the file level (one file per run).

- `~/.claude/plugins/config/launcelot-regulatory-legal/diffs/_index.yaml`
  Run index:
  ```yaml
  - id: <reg-slug>-<YYYY-MM-DD>-<HHMM>
    slot_a: <citation>
    slot_a_verification: <…>
    slot_b_targets: [<slug>, <slug>, …]
    findings_by_band:
      명백: <n>
      가능: <n>
      회색지대: <n>
    handoff_gaps: [<gap-id>, …]
    triggered_by: <user | reg-feed-watcher:<digest-id> | scheduled>
    at: <ISO-datetime>
  ```

For library mode, the same run produces one master file `_library-<YYYY-MM-DD>-<HHMM>.md` linking to per-script sub-files in the same directory.

## Scope integrity

If the user asks to exclude a script, a category of finding, or a defense path:

1. Honor the request.
2. Banner the exclusion at the top of every output and persist it in the index:
   > ⚠️ 범위 제한: 사용자 요청으로 <X> 제외. 본 diff는 전수 점검이 아니며, 제외된 영역의 위반 가능성은 식별되지 않았다.
3. Carry the banner into any handoff to gap-surfacer or policy-redraft.

## Failure handling

- **launcelot-lawyer unavailable**: do not silently proceed. Emit:
  > ⚠️ launcelot-lawyer 비가용. 조문·판례 인용은 미검증 상태. 두 가지 선택지: (1) 사용자가 슬롯 A 텍스트의 출처와 최신성을 직접 보증하고 diff를 진행 (모든 출력에 `[사용자 보증]` 태그 부착), (2) diff 중단. 어느 쪽인가?
  사용자가 (1)을 택해도 결과의 모든 인용 라인에 `[사용자 보증 — 미검증]` 태그를 영구히 부착한다.
- **Slot A text is pasted and launcelot-lawyer says `not-found`**: 중단하고 사용자에게 "이 조문/판례를 찾지 못했다. 출처 URL을 확인하라"라고 보고. diff를 진행하지 않는다.
- **Script library mode and 0 scripts indexed**: 디스크에 아무 것도 쓰지 않고 cold-start의 Module 5(스크립트 라이브러리)를 다시 돌릴 것을 안내.

## What this skill does NOT do

- Does not state the current text of any 조문 or 판례 from model knowledge. All such text is fetched through launcelot-lawyer or pasted by the user.
- Does not conclude that a violation has occurred. Findings are element-level matches with defense windows.
- Does not rewrite any line. That is policy-redraft.
- Does not modify gap-tracker.yaml directly. It hands off candidate gaps to gap-surfacer, which writes the tracker.
