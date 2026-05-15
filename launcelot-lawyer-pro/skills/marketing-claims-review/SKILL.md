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

This skill is the extraction-and-mapping core of Launcelot-Lawyer-Pro. It produces a structured list of every line that _could_ trigger criminal or regulatory exposure, mapped to the relevant 죄목 or 규정, with a draft risk band and a suggested fix.

조문·판례 인용은 본 스킬이 _직접_ 한다. 외부 스킬에 위임하지 않는다. 매 인용은 `references/snippet-protocol.md`에 정의된 3단 fetch fallback(WebFetch → firecrawl → jina)으로 원문을 확보한 뒤 그 텍스트에서만 만들어진다.

## Run order

1. 대본 텍스트를 받는다. paste, 파일 경로, wiki 슬러그 셋 다 허용. 파일이면 Read, wiki 슬러그면 `wiki` MCP 사용. 매터가 활성화돼 있으면 `matters/<slug>/matter.md`도 읽어 사실 컨텍스트로만 사용.
2. 아래 pull bucket 패스를 순차로 돌려 대상 라인을 추출.
3. 각 라인을 죄목·규정 카탈로그에 매핑(라벨 단계 — 아직 조문 텍스트 인용 없음).
4. 각 매핑마다 **`references/snippet-protocol.md` 절차로 조문/판례를 fetch**해 스니펫을 확보한다. 스니펫이 없으면 그 라인은 결과에서 보류 처리(아래 "Snippet 미확보 처리" 참조).
5. risk band 결정 — fetch된 조문·판례 텍스트와 라인을 대조해 명백/가능/회색지대/안전 중 하나.
6. 리포트 발행.

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

Why: 형법 283 (협박), 형법 350 (공갈). Common false positives in legal-channel scripts: hypothetical descriptions of what _a client could do_ to a counterparty. Mark whether the threat is uttered by the speaker or quoted/described.

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

## Snippet 확보 (필수 단계)

각 라인-bucket 매핑에 대해 후보 조문·판례 식별자를 정한 뒤 `references/snippet-protocol.md` 절차로 원문을 확보한다.

- 조문은 시행 중인 현행본을 `law.go.kr`에서 fetch.
- 판례는 `glaw.scourt.go.kr`에서 fetch (인용에 필요한 판결만 — 핵심 판시사항이 명확히 있는 것 위주).
- 변협 광고규정은 `koreanbar.or.kr`에서 fetch.
- 공정위 심결은 `ftc.go.kr`에서 fetch.

3단 fallback이 모두 실패하면 결과 블록 안에 그 조문 인용을 _넣지 않고_ 라인 자체를 "Snippet 미확보 — 보류" 상태로 표시한다(아래 출력 포맷 참조).

## Output format (per line)

For every extracted line, emit a block. Do not collapse blocks even when the same line appears in multiple buckets — emit one block per bucket so the reader can see the full surface area.

````markdown
### Line <N>.<bucket>

**Quote:** "<exact text from the script>"

**Location:** <line range or timestamp if available>

**Bucket:** <A | B | C | D | E | F | G | H | I | J>

**Candidate 죄목·규정:**

- <조문 라벨>
- <…>

**Snippet (확보된 원문):**

```yaml
- identifier: "형법 제307조 제1항"
  source: "law.go.kr"
  url: "https://www.law.go.kr/법령/형법/제307조"
  fetched_at: "<ISO-datetime>"
  fetch_method: "WebFetch"
  effective_date: "2024-02-09"
  raw_quote: |
    공연히 사실을 적시하여 사람의 명예를 훼손한 자는 …
```

**Identifiability / target:**

- <real name | role | inference cue | none>

**Factual or evaluative?** <fact | opinion | mixed>

**Public interest signal:** <present | absent | unclear>
(형법 310 정당행위 / 진실성·공익성 항변의 단서가 되는 표현이 있는지)

**Substantiation on file:** <yes (link) | claimed but unverified | none>

**Risk band:** <명백 | 가능 | 회색지대 | 안전>

**Why this band:** <one sentence. raw_quote의 어느 부분과 라인의 어느 부분이 일치하는지 명시.>

**Suggested fix:** "<rewritten line that keeps the editorial energy where possible>"
````

### Snippet 미확보 라인 블록

3단 fetch fallback이 모두 실패한 라인은 다음과 같이 출력하고 risk band·suggested fix는 비워둔다.

```markdown
### Line <N>.<bucket> — ⛔ Snippet 미확보 (결과 생성 불가)

**Quote:** "<exact text>"
**Bucket:** <…>
**시도한 출처:** <Step 1 URL>, <Step 2 URL>, <Step 3 URL>
**실패 사유:** <Step 1 에러>, <Step 2 에러>, <Step 3 에러>
**조치:** 사용자가 URL을 제공하거나 원문을 paste하면 그 텍스트로 재시도. 둘 다 없으면 본 라인 평가 보류.
```

이 라인은 요약의 band distribution에서 별도 카테고리(`snippet_missing`)로 카운트한다.

## Risk band definitions

Use these definitions consistently. They are tuned to a Korean lawyer YouTube channel, not to a general publisher. **fetch된 조문·판례 raw_quote와 라인을 직접 대조해** 결정한다.

- **명백** — fetch된 조문의 구성요건이 라인과 직접 매칭되고 보호 doctrine(진실성·공익성·정당한 비평)이 라인 자체에서 명백히 부재. 한국 형사변호사라면 출판 전 수정·삭제를 권할 수준.
- **가능** — 구성요건 매칭은 있으나 결과는 검증 불가 사실(진실성, 공익성, 식별성, audience scope)에 달려있음. 사용자 결정 필요.
- **회색지대** — 표면 패턴은 위험으로 잡혔으나 fetch된 판례 doctrine이 라인을 보호할 가능성이 높음(익명화된 사실, 평가성 framing, 공인 대상, 입증 가능).
- **안전** — bucket으로 잡혔지만 면밀히 보면 카탈로그를 건드리지 않음. bucket 분류가 false positive인 경우.

본 스킬은 사용자 risk appetite로 band를 올리거나 내리지 않는다. band는 한국법 자체와 fetch된 raw_quote만 보고 결정한다.

## Summary section

After all per-line blocks, emit:

```markdown
## 요약

- Total lines analyzed: <N>
- Pulled into at least one bucket: <N>
- Band distribution: 명백 <n> / 가능 <n> / 회색지대 <n> / 안전 <n> / snippet_missing <n>
- Top three lines to fix first: <line numbers + one-line reason each>
- Snippet fetch 수: <n attempted> / <n successful> / <n missing>
```

## What this skill does NOT do

- 모델 지식으로 조문·판례를 단정하지 않는다. 모든 인용은 raw_quote(스니펫)에서만 만들어진다.
- 외부 스킬(`launcelot-lawyer` 포함)에 조문 검증을 위임하지 않는다.
- 사용자 risk appetite·speech policy·Hard rules로 결과를 조정하지 않는다. 본 플러그인은 그런 설정을 받지 않는다.
- Full rewriting을 하지 않는다. `Suggested fix`는 한 줄 드래프트이며 전체 리라이트는 `koreanizer` / `humanizer` / 화자별 스킬(`kmj`, `kmg`, 등)에 맡긴다.
- 게시 여부를 결정하지 않는다. 변호사의 몫.

## Failure handling

- 모든 매핑의 스니펫 fetch가 전부 실패한 경우(즉 어느 조문도 raw_quote를 확보하지 못한 경우): 결과 발행을 중단하고 다음을 사용자에게 보고한다.

  ```
  ⛔ Snippet 확보 실패 — marketing-claims-review 결과를 만들 수 없습니다.
  시도한 출처: law.go.kr / glaw.scourt.go.kr / koreanbar.or.kr / ftc.go.kr …
  3단 fallback(WebFetch → firecrawl → jina) 모두 실패. 네트워크 또는 출처
  접근성을 확인하거나, 인용이 필요한 조문/판례의 URL을 직접 알려주십시오.
  ```

- 일부 라인의 스니펫만 실패한 경우: 그 라인만 "Snippet 미확보" 블록으로 표시하고 나머지는 정상 발행.

- 대본에 pull-bucket hit이 0건이면:

  ```
  추출된 검증 대상 라인이 없습니다. 전형적인 위험 패턴(실명 사실 진술, 단정적
  결과 표현, 비교광고, 제3자 사건 묘사 등)이 발견되지 않았으나, 이는 위반이
  없다는 결론이 아닙니다. 확실한 결론을 원하면 /launcelot-lawyer-pro:launch-review로
  카테고리 체크리스트를 한 번 더 돌리시기 바랍니다.
  ```
