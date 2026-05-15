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

# /launcelot-lawyer-pro:policy-diff

This skill performs the actual diff between Korean regulatory texts and the user's script(s). It is the workhorse skill of Launcelot-Lawyer-Pro.

규범 텍스트(슬롯 A)는 본 스킬이 _직접_ `references/snippet-protocol.md` 절차로 fetch한 raw_quote를 사용한다. 외부 스킬에 위임하지 않는다. snippet 확보가 안 되면 diff를 시작하지 않는다.

## Run order

1. 매터 활성 여부 확인. 활성이면 `matters/<slug>/matter.md`만 사실 컨텍스트로 사용. 출력 디렉토리(`diffs/`)는 없으면 자동 생성.
2. Resolve slot A — 규범 입력. 본 스킬이 snippet-protocol로 fetch.
3. Resolve slot B — 스크립트 입력.
4. snippet 확보 검증. raw_quote가 비면 diff 시작 불가(아래 Failure handling).
5. Run the diff. Emit findings.
6. Persist results.
7. Hand off to `gap-surfacer` for any finding the user wants tracked.

## Inputs

### Slot A — Regulatory input

Accept any of:

- A 조문 reference (e.g., `형법 307조`, `정통망법 70조 1항`, `변호사 광고에 관한 규정 4조`). 본 스킬이 `references/snippet-protocol.md` 절차로 fetch (default URL: law.go.kr / koreanbar.or.kr / ftc.go.kr).
- A 판례 reference (e.g., `대법원 2020. 11. 19. 선고 2020도5813 판결`). 본 스킬이 glaw.scourt.go.kr 또는 ccourt.go.kr에서 fetch.
- Pasted regulatory text. 이 경우 `slot_a.source = "user-pasted"`로 표시되며, 인용 식별자(법령명·조항 또는 판결번호)는 사용자가 제공한 그대로 기록한다. 본 스킬은 페이스트 텍스트가 진짜 현행본인지 별도로 fetch해 교차 확인하려 시도한다(snippet-protocol). 교차 확인이 성공하면 `slot_a.cross_verified = yes`, 실패하면 `cross_verified = no — user-asserted`로 표시.
- A reg-feed-watcher item ID (when policy-diff is invoked from a digest handoff). reg-feed-watcher가 이미 fetch한 snippet 객체를 그대로 받는다.
- A keyword set (e.g., `명예훼손 신착 판례`). 본 스킬이 glaw.scourt.go.kr 검색에서 top N을 fetch하고 사용자에게 어떤 항목을 diff할지 확인.

각 경우 다음을 기록·보존:

- `slot_a.id` (slug 또는 citation)
- `slot_a.source` (law.go.kr / glaw.scourt.go.kr / ccourt.go.kr / koreanbar.or.kr / ftc.go.kr / user-pasted)
- `slot_a.url`
- `slot_a.fetch_method` (WebFetch / firecrawl / jina / user-pasted)
- `slot_a.fetched_at`
- `slot_a.effective_date`
- `slot_a.raw_quote` (diff에 실제로 사용된 본문)
- `slot_a.cross_verified` (yes / no / n-a)

### Slot B — Script input

Accept any of:

- A pasted script body.
- A file path (Read).
- A wiki slug (fetch via `wiki` MCP if available).
- A library directive (`library`, `library:published`, `library:tagged:<tag>`).
  - `script-library.yaml`이 있으면 거기 기재된 위치를 indexing. 파일이 없으면 본 스킬이 사용자에게 "라이브러리 폴더 또는 wiki prefix를 알려달라"고 한 번 묻고, 응답을 `script-library.yaml`에 lazy로 기록한 뒤 진행. 미응답이면 library 모드 거부, 단일 스크립트 모드로 격하.

For library mode, iterate: one diff record per script. Concurrency is sequential by default to keep fetch load predictable.

## Diff method

For each script (slot B), walk it line by line. For each line:

1. **Pre-filter**: does the line plausibly implicate the categories that slot A addresses? raw_quote에서 trigger 토큰을 추출. trigger 토큰이 라인에 0건이면 skip.

2. **Element decomposition**: raw_quote에서 요건을 추출.
   - 조문: 행위 주체, 행위 객체, 행위 양태, 결과·결과발생 가능성, 위법성 조각 사유.
   - 판례: 판시사항이 인정한 행위 패턴, 부정한 행위 패턴, 인정 요건.
   - 광고규정·표시광고법: 금지되는 표현 유형, 허용 조건, 실증 요건.

3. **Element matching**: 각 요건에 대해 라인의 어느 부분이 `present` / `absent` / `unclear`인지 표시. 본 스킬은 위반이 *발생했다*고 결론짓지 않는다. 단지 라인에 요건이 *보이는지*만 표시.

4. **Defense window**: 후보 항변 — 각 항변에 해당하는 조문/판례도 snippet-protocol로 fetch해 raw_quote를 본 finding에 첨부.
   - 진실성·공익성 항변 (형법 310, 정통망법 70 3항)
   - 정당행위 (형법 20)
   - 인용의 정당한 범위 (저작권법 28)
   - 변호사 광고규정의 허용 단서
   - 표시광고법의 실증된 표현 단서

5. **Risk band**:
   - **명백**: 요건이 라인 안에 모두 갖춰져 있고, 항변 단서가 라인 또는 라인 인접에 보이지 않음.
   - **가능**: 요건이 갖춰져 있으나 항변 단서가 있다(진실성·공익성·정당범위 등).
   - **회색지대**: 요건 일부 결여 또는 적용 자체에 다툼.
   - **안전**: 요건 명백히 결여.

본 스킬은 사용자 risk appetite·Hard rules·Speech policy로 band를 조정하지 않는다. band는 한국법 자체와 fetch된 raw_quote만으로 결정한다.

## Output (per line finding)

```markdown
### Finding <slug-B>:<line-N>

**Slot A snippet:**

\`\`\`yaml

- identifier: <…>
  source: <…>
  url: <…>
  fetched_at: <…>
  fetch_method: <…>
  effective_date: <…>
  cross_verified: <yes | no | n-a>
  raw_quote: |
  <…>
  \`\`\`

**Slot B:** <script slug>, line <N>

**Quote (대본):** "<exact text>"

**Elements present:**

- <element 1>: <evidence in the line>
- <element 2>: <…>

**Elements absent / unclear:**

- <element>: <왜 결여 또는 불분명>

**Available defenses (snippet 첨부):**

- 진실성·공익성 (형법 310): <적용 가능성 한 줄 + snippet 객체>
- 정당행위 (형법 20): <…>
- 인용 (저작권법 28): <…>
- 변호사 광고규정 단서: <…>

**Risk band:** <명백 | 가능 | 회색지대 | 안전>

**Suggested fix direction:** <한 줄, 본문 톤 유지 가능 여부 포함>

**Hand off to policy-redraft?** <yes | no — reason>
**Hand off to gap-surfacer?** <yes (proposed gap entry below) | no>
```

## Output (per script summary)

```markdown
## Diff summary — <slug-B> against <slot_a.id>

- Lines analyzed: <N>
- Findings: 명백 <n> / 가능 <n> / 회색지대 <n>
- Defense-dependent findings: <n>
- Snippet fetch: <n attempted> / <n successful>
- Top three lines: <line numbers>
- Suggested next step: <policy-redraft <these line ids> | publish a 수정 후 게시 memo | open gap entries for <these>>
```

## Persistence

Two files per diff run plus index updates.

- `~/.claude/plugins/config/launcelot-lawyer-pro/diffs/<reg-slug>-<YYYY-MM-DD>-<HHMM>.md`
  Full per-line findings + per-script summary for the run. Append-only at the file level (one file per run).

- `~/.claude/plugins/config/launcelot-lawyer-pro/diffs/_index.yaml`
  Run index:

  ```yaml
  - id: <reg-slug>-<YYYY-MM-DD>-<HHMM>
    slot_a:
      id: <citation>
      source: <…>
      cross_verified: <…>
      fetched_at: <…>
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

- **슬롯 A snippet 확보 실패**: diff를 시작하지 않는다. 사용자에게 다음 메시지 출력 후 종료.

  ```
  ⛔ Snippet 확보 실패 — policy-diff 시작 불가.
  대상: <slot_a.id>
  시도한 출처: <Step 1 URL>, <Step 2 URL>, <Step 3 URL>
  실패 사유: <Step 1>, <Step 2>, <Step 3>
  조치: 사용자가 URL을 제공하거나 원문을 paste하면 그 텍스트로 재시도.
  ```

- **슬롯 A가 사용자 paste이며 cross-verification 실패**: 사용자에게 "이 텍스트의 출처를 cross-verify하지 못했다. (1) 사용자가 보증하고 진행 — 결과 라인에 `[cross-verify 실패]` 태그 영구 부착, (2) 출처 URL을 직접 제공해 다시 시도, (3) diff 중단"을 묻는다.
- **항변 조문 snippet 일부 실패**: 그 항변 조문은 finding에서 제거하고 사유를 명시. 핵심 항변(형법 310, 정통망법 70 3항 등)이 실패하면 band를 결정하지 않고 그 finding을 `보류`로 표시.
- **Script library mode and 0 scripts indexed**: 사용자에게 "라이브러리 폴더 또는 wiki prefix를 알려달라" 한 번 묻고 응답을 `script-library.yaml`에 lazy로 기록. 응답이 없으면 library 모드를 거부하고 단일 스크립트 모드로만 진행.

## What this skill does NOT do

- Does not state the current text of any 조문 or 판례 from model knowledge. All such text is `references/snippet-protocol.md` 절차로 본 스킬이 직접 fetch하거나 사용자가 paste한 것에서만 만들어진다.
- Does not delegate fetch or verification to any external skill (`launcelot-lawyer` included).
- Does not conclude that a violation has occurred. Findings are element-level matches with defense windows.
- Does not rewrite any line. That is policy-redraft.
- Does not modify gap-tracker.yaml directly. It hands off candidate gaps to gap-surfacer, which writes the tracker.
- Does not adjust band by user risk appetite — 본 플러그인은 그런 설정을 받지 않는다.
