---
name: marketing-claims-review
description: >
  한국 변호사 유튜브 대본에서 형법적·행정법적 평가가 가능한 문장을 모두 추출해
  대한민국 형법(307·309·311·156·283·347·314·350), 정보통신망법 70조, 개인정보
  보호법 71조 등, 변호사법 23조 및 대한변협 광고규정, 표시광고법, 저작권법
  인용 조항과 대조하여 죄목별로 분류하고 위험 등급(명백 / 가능 / 회색지대 / 안전)을
  매긴 뒤 수정안을 제시한다. 사용자가 "대본 검토", "이거 명예훼손 안돼?", "이
  표현 괜찮아?", "대본에서 형법 위반 찾아줘", "claims review"라고 말하거나
  대본 텍스트·파일을 붙여넣을 때 실행한다.
argument-hint: "[대본 텍스트 또는 파일 경로]"
---

# /launcelot-lawyer-pro:marketing-claims-review

This skill is the extraction-and-mapping core of Launcelot-Lawyer-Pro. It does not decide whether a phrase is criminal — that requires statute and case-law verification, which is delegated to `launcelot-lawyer`. This skill produces a structured list of every line that *could* trigger criminal or regulatory exposure, mapped to the relevant 죄목 or 규정, with a draft risk band and a suggested fix.

## Run order

1. Load `~/.claude/plugins/config/launcelot-lawyer-pro/CLAUDE.md`. If the file does not exist or contains `[PLACEHOLDER]` markers, stop and tell the user to run `/launcelot-lawyer-pro:cold-start-interview` first.
2. Receive the script text. Accept either a paste, a file path, or a wiki search slug. If a file path is given, Read it; if a wiki slug is given, fetch via the `wiki` MCP if available.
3. Run the extraction passes below.
4. For each extracted line, classify against the 죄목·규정 catalog.
5. Hand off every quoted 조문 and 판례 reference to `launcelot-lawyer` for existence verification. If `launcelot-lawyer` is not available (check `## Integrations` in the config), mark every statute and case citation in the output as `[미검증 — launcelot-lawyer 비가용]` and stop short of stating any rule from memory.
6. Apply the user's risk calibration (`## Risk calibration`) and speech policy (`## Speech policy`).
7. Emit the report.

## What gets extracted

Walk the script start to end. For each unit (sentence or clause), decide whether it falls into any of the following pull buckets. A line can land in multiple buckets — record it under each.

### Pull bucket A — Statements of fact about identifiable third parties

Any sentence that asserts a factual proposition about a real, identifiable person or entity. Anonymized hypotheticals do not count; thinly anonymized hypotheticals (e.g., "A씨" with enough surrounding context to identify) do count.

Tag each line with:
- The asserted fact verbatim.
- Identifiability cues (real name, role, time, location, case number, distinctive description).
- Whether the fact is presented as the speaker's claim, as a press report, or as a court finding.

Why: this is the input to defamation analysis (형법 307·309, 정통망법 70).

### Pull bucket B — Insulting expressions

Any sentence containing a value judgment that expresses contempt or scorn directed at an identifiable person or group, separate from any factual claim.

Why: 형법 311 (모욕). Distinct from defamation because no fact is asserted.

### Pull bucket C — Allegations of crime or specific wrongdoing

Any sentence that attributes a crime, regulatory violation, or specific wrongful act to an identifiable person, regardless of whether the speaker labels it an opinion.

Why: 형법 156 (무고) is triggered only by a false report to authorities, not by public speech, so 무고 is rarely the right tag for a YouTube line — but the same content often triggers 형법 309 (출판물 등에 의한 명예훼손) on aggravated terms. Tag both candidates and let `launch-review` route.

### Pull bucket D — Threatening, coercive, or extortive expressions

Any sentence that conveys a threat of harm (physical, reputational, economic, legal) toward an identifiable target.

Why: 형법 283 (협박), 형법 350 (공갈). Common false positives in legal-channel scripts: hypothetical descriptions of what *a client could do* to a counterparty. Mark whether the threat is uttered by the speaker or quoted/described.

### Pull bucket E — Statements that could induce a transaction by deception

Any sentence that promises a result, capacity, or outcome that could induce a viewer to act (retain counsel, send money, take a specific legal step) and that is not literally true or is not substantiated.

Why: 형법 347 (사기), 표시광고법 (허위·과장광고), 변호사법 / 대한변협 광고규정. Note: 사기 requires actual deception causing property disposition, which is rare from a script alone, but the same line is almost always a 표시광고법·광고규정 violation if it overstates capacity.

### Pull bucket F — Statements interfering with another's business or duties

Any sentence that disparages a named competitor (other 변호사·법무법인·전문직), accuses a specific business of misconduct, or could induce viewers to disrupt a specific business's operations.

Why: 형법 314 (업무방해).

### Pull bucket G — Personal data exposure

Any line that exposes identifying information about a real person without a lawful basis: name + address, name + workplace, name + medical/criminal history, name + minor child, contact details, photographs, voice.

Why: 개인정보 보호법 71조 등 (형사처벌), and frequently 형법 307·309 in combination.

### Pull bucket H — Third-party copyrighted material

Any line that quotes verbatim or substantively a press article, judgment text, or other copyrighted work without attribution and without falling within 저작권법 28조 (공표된 저작물의 인용) limits.

Why: 저작권법.

### Pull bucket I — Lawyer advertising / professional-rules-implicating language

Any line that:
- Promises a specific outcome ("무조건 이깁니다", "100% 승소", "반드시 무죄").
- Claims comparative superiority over named or strongly implied other lawyers/firms.
- Solicits a specific identifiable potential client.
- Implies expertise or specialty without the qualifying basis (예: 전문 자격이 없는데 "전문 변호사").
- Could be read as splitting fees with non-lawyers or as steering to a non-lawyer service.
- States or implies a guarantee.

Why: 변호사법 23조, 대한변호사협회 변호사 광고에 관한 규정, 변호사윤리장전, 표시광고법.

### Pull bucket J — Adjacent-domain unauthorized advice

Any line giving specific medical, tax, accounting, financial-investment, or other regulated-profession advice beyond legal commentary.

Why: 의료법, 세무사법, 공인회계사법, 자본시장법 등 (각 직역의 무자격 자문 형사처벌).

## Output format (per line)

For every extracted line, emit a block. Do not collapse blocks even when the same line appears in multiple buckets — emit one block per bucket so the reader can see the full surface area.

```markdown
### Line <N>.<bucket>

**Quote:** "<exact text from the script>"

**Location:** <line range or timestamp if available>

**Bucket:** <A | B | C | D | E | F | G | H | I | J>

**Candidate 죄목·규정:**
- <조문 번호 + 명칭>
- <…>

**Identifiability / target:**
- <real name | role | inference cue | none>

**Factual or evaluative?** <fact | opinion | mixed>

**Public interest signal:** <present | absent | unclear>
(form 310 정당행위 / 진실성·공익성 항변의 단서가 되는 표현이 있는지)

**Substantiation on file:** <yes (link) | claimed but unverified | none>

**Risk band (draft):** <명백 | 가능 | 회색지대 | 안전>

**Why this band:** <one sentence>

**Suggested fix:** "<rewritten line that keeps the editorial energy where possible>"

**Verification handoff:**
- launcelot-lawyer: confirm currency of <조문>, retrieve representative 판례 on <쟁점>, flag if interpretation has shifted.
```

## Risk band definitions

Use these definitions consistently. They are tuned to a Korean lawyer YouTube channel, not to a general publisher.

- **명백** — A reasonable Korean criminal lawyer would advise removing or rewriting before publication. Either the line crosses a settled rule or the protective doctrine (진실성·공익성·정당한 비평) is plainly unavailable.
- **가능** — Plausible criminal or regulatory exposure exists. Outcome depends on facts the validator cannot verify (truth, public interest, identifiability, audience size). Requires user decision with eyes open.
- **회색지대** — Surface pattern looks risky but doctrine likely shields it given the channel's profile (anonymized facts, evaluative framing, public-figure target, substantiated). Note for log; not actionable on its own.
- **안전** — Pattern does not implicate the catalog above. Emitted only when the line was pulled into a bucket but on closer review does not raise the issue.

Apply the user's `## Risk calibration`:
- 보수적 → raise every band one step (안전 stays 안전; 회색지대 → 가능; 가능 → 명백). Never lower.
- 중도 → as scored.
- 공격적 → lower borderline calls one step within `회색지대` and `가능`, never lower than 회색지대, never affect 명백.

## Summary section

After all per-line blocks, emit:

```markdown
## 요약

- Total lines analyzed: <N>
- Pulled into at least one bucket: <N>
- Band distribution: 명백 <n> / 가능 <n> / 회색지대 <n> / 안전 <n>
- Hard-rule violations (from `## Hard rules`): <list of line numbers, or "none">
- Top three lines to fix first: <line numbers + one-line reason each>
- Verification queue handed to launcelot-lawyer: <count of items>
```

## What this skill does NOT do

- Does not assert that a specific 조문 currently reads any particular way. Hand 조문 quotation off to `launcelot-lawyer`.
- Does not assert that a specific 판례 exists or held what is described. Same handoff.
- Does not rewrite the script in finished form. The `Suggested fix` field is a one-line draft; full rewriting belongs to `koreanizer` / `humanizer` / the relevant per-attorney writing skill (`kmj`, `kmg`, etc.).
- Does not decide whether to publish. That is the attorney's call.

## Failure handling

- If `launcelot-lawyer` is unavailable, append a banner at the top of the output:

  ```
  ⚠️ launcelot-lawyer 비가용 — 본 결과의 조문·판례 인용은 미검증 상태이며,
  사용자가 직접 국가법령정보센터(law.go.kr) / 대법원 종합법률정보
  (glaw.scourt.go.kr) / 헌재(ccourt.go.kr)에서 실존과 최신성을 확인해야 한다.
  ```

- If the script contains zero pull-bucket hits, emit:

  ```
  추출된 검증 대상 라인이 없습니다. 전형적인 위험 패턴(실명 사실 진술, 단정적
  결과 표현, 비교광고, 제3자 사건 묘사 등)이 발견되지 않았으나, 이는 위반이
  없다는 결론이 아닙니다. 확실한 결론을 원하면 /launcelot-lawyer-pro:launch-review로
  카테고리 체크리스트를 한 번 더 돌리시기 바랍니다.
  ```
