---
name: feature-risk-assessment
description: >
  marketing-claims-review·launch-review가 회색지대~명백 사이로 분류한 단일 라인 또는
  단일 표현 패턴에 대해, 한 줄짜리 메모로 정리하기 어려운 경우의 심층 리스크 분석.
  무엇이 잘못될 수 있는지, 얼마나 자주(확률), 얼마나 심각하게(영향), 무엇으로 완화할
  수 있는지를 4축 구조로 분해한다. 사용자가 "이 라인 깊게 분석해줘", "이게 명백인지
  가능인지 모르겠어", "이 표현 어디까지 위험해?", "risk assessment", "심층 분석"이라고
  말하거나 launch-review가 어떤 라인을 핸드오프할 때 실행한다.
argument-hint: "[--line <slug>:<N> | --finding <id> | --gap <gap-id>]"
---

# /launcelot-lawyer-pro:feature-risk-assessment

This skill is the deep-dive complement to `marketing-claims-review` and `launch-review`. Those skills classify quickly; this one slows down on a single line or expression pattern when the quick classification is not enough.

It does not replace the line-level review — it explains the line in four dimensions so the attorney can decide whether to publish, rewrite, or hold.

모든 조문·판례 인용은 본 스킬이 _직접_ `references/snippet-protocol.md` 절차로 fetch한 raw_quote에서만 만들어진다. 외부 스킬에 위임하지 않는다.

## When to run

- launch-review marks a line as `가능` and the suggested fix didn't resolve it.
- marketing-claims-review puts a line into two or more pull buckets and the band differs across buckets.
- A line touches a novel pattern (new case law, new advertising-rule update) and the user wants the analysis written out.
- A pattern gap was promoted by gap-surfacer and the attorney wants the structural risk written up before deciding to add a recurring rule.

## Run order

1. 매터가 활성화돼 있으면 `matters/<slug>/matter.md`만 읽어 사실 컨텍스트로 사용. 다른 설정 파일은 로드하지 않는다.
2. 입력 해석:
   - `--line <slug>:<N>`: read the script line directly.
   - `--finding <id>`: load the marketing-claims-review or policy-diff finding.
   - `--gap <gap-id>`: load the gap entry.
3. 후보 죄목·규정 식별 → 각각에 대해 `references/snippet-protocol.md` 절차로 조문/판례 fetch.
4. 4축 분해 — 각 축의 모든 주장은 fetch된 raw_quote에 닿아 있어야 한다.
5. 메모 발행.
6. 단일 권고(완화책 1종) 또는 "추가 정보 필요" 한 줄.

## Four-axis decomposition

For the target line, work through all four axes. Skipping an axis requires a written reason in the output.

### Axis 1 — What could go wrong

Enumerate the failure modes the line could trigger. For each, name the 죄목·규정 and the elements that would have to be satisfied. Do not assert that elements are satisfied — only that they are _implicated_.

For each candidate failure mode, capture:

- 죄목 또는 규정 식별자 (조문 번호 포함).
- **Snippet**: snippet-protocol로 확보한 조문 raw_quote 객체. 확보 실패 시 본 후보를 *제거*하고 그 사유를 출력한다(후보 자체를 "근거 미확보"로 남기지 않는다).
- 핵심 요건 (3~5개) — raw_quote에서 추출.
- 라인이 어느 요건을 시각적으로 충족하는 듯 보이는가.
- 라인이 어느 요건을 시각적으로 결여하는가.
- 라인이 의존하는 항변(310 진실성·공익성 / 20 정당행위 / 저작권법 28 인용 / 광고규정 단서 / 표시광고법 실증) — 항변 조문도 동일하게 snippet-protocol로 fetch.

후보 죄목 전체의 snippet이 단 한 건도 확보되지 않으면 본 스킬은 결과 발행을 *중단*하고 사용자에게 보고한다(아래 Failure handling).

### Axis 2 — How likely (probability framing)

확률을 숫자로 단언하지 않는다. 한국 변호사 마케팅 컨텍스트의 정성적 등급으로:

- **드문 패턴**: 같은 패턴이 사용자 라이브러리·동종 채널·과거 판례에서 명백한 분쟁으로 비화한 사례가 거의 없음.
- **드물지 않음**: 동종 채널에서 비공식 클레임·삭제 요청이 발생한 사례가 있음. 공식 분쟁은 미확정.
- **반복적**: 같은 패턴이 명예훼손 또는 광고규정 위반으로 다툼이 된 사례가 다수 있음.
- **고위험 알려진 패턴**: 동종 패턴에 대한 형사 또는 행정 조치 사례가 snippet으로 확인됨.

확률 라벨의 근거는 반드시 *증거*에 닿아야 한다(snippet 객체에 담긴 판례 또는 심결, 또는 본 라이브러리의 과거 분쟁 이력, 또는 명확히 "근거 없음 — 정성 판단"으로 명시). 근거 없는 라벨은 출력하지 않는다.

### Axis 3 — How bad (severity)

만약 위반으로 확정되면 어디까지 갈 수 있는지. 한국법 컨텍스트의 단계:

- **민사 청구 가능성**: 손해배상·정정보도·사과문 청구.
- **임시조치·삭제 가능성**: 정통망법 44조의2 또는 플랫폼 자체 정책에 따른 게시물 차단.
- **변협 진정·징계 가능성**: 변호사법·광고규정 위반 시.
- **행정 처분 가능성**: 표시광고법 위반 시 공정위 조사·시정 명령·과징금.
- **형사 입건 가능성**: 명예훼손·모욕·개인정보 보호법·저작권법 형사조항.
- **사후 채널 운영 영향**: 동영상 비공개·채널 평판 하락·플랫폼 정책 충돌.

각 단계에 대해 라인 단위의 노출 정도를 정성 평가하되, "확실하다"라는 단정은 하지 않는다. 각 단계의 법정형·처분 범위를 인용할 경우 그 조문의 snippet이 Axis 1에 들어있어야 한다.

### Axis 4 — Mitigations

각 후보 완화책에 대해:

- 변경 종류 (라인 수정 / 단락 재구성 / 자막 보완 / 면책 고지 / 영상 보류 / 비공개 처리 / 패턴 등록).
- 변경이 닫는 위반 모드 (Axis 1 항목과 연결).
- 변경이 남기는 잔존 위험.
- 변경의 채널 톤·메시지 손실 비용 (낮음/중간/높음).
- 변경 후 예상 위험 등급 (Axis 1 모드별).

## Output memo

````markdown
# Feature risk assessment — <target id>

작성 일자: <YYYY-MM-DD>
대상 라인: "<exact quote>"
출처: <slug-B>, line <N> · 매터: <slug | none>
Snippet fetch 결과: <n attempted> / <n successful>

---

## Axis 1 — 무엇이 잘못될 수 있나

### 후보 1: <죄목 또는 규정>

**Snippet:**

```yaml
- identifier: "형법 제307조 제1항"
  source: "law.go.kr"
  url: "<…>"
  fetched_at: "<…>"
  fetch_method: "WebFetch"
  effective_date: "<…>"
  raw_quote: |
    <원문 텍스트>
```

- 요건 (raw_quote에서 추출): <…>
- 라인이 충족하는 듯 보이는 요건: <…>
- 라인이 결여하는 듯 보이는 요건: <…>
- 라인이 의존하는 항변: <항변 조문 식별자 + snippet>
- 항변의 약점: <…>

### 후보 2: <…>

…

## Axis 2 — 얼마나 자주 일어날 수 있나

- 라벨: <드문 패턴 | 드물지 않음 | 반복적 | 고위험 알려진 패턴>
- 근거: <snippet 객체로 인용된 판례/심결 또는 "근거 없음 — 정성 판단">

## Axis 3 — 얼마나 심각해질 수 있나

| 단계           | 노출 정도 | 비고 |
| -------------- | --------- | ---- |
| 민사 청구      | <…>       | <…>  |
| 임시조치·삭제  | <…>       | <…>  |
| 변협 진정·징계 | <…>       | <…>  |
| 행정 처분      | <…>       | <…>  |
| 형사 입건      | <…>       | <…>  |
| 채널 운영 영향 | <…>       | <…>  |

## Axis 4 — 완화책

### 완화 A

- 변경: <라인 수정 / 단락 재구성 / 자막 보완 / 면책 고지 / 영상 보류 / 비공개 / 패턴 등록>
- 닫는 위반 모드: <Axis 1의 후보 N>
- 잔존 위험: <…>
- 채널 톤 비용: <낮음 | 중간 | 높음>
- 변경 후 예상 등급: <…>

### 완화 B

…

## 단일 권고

(완화책 중 하나만, 또는 "추가 정보 필요" 한 줄. 두 개를 동시에 권하지 않는다.)

## 다음 단계

- 본 평가를 어떤 갭/리뷰에 연결할지: <gap-id | finding-id | launch-review revision>
- 작성자가 알아둬야 할 단 한 가지: <한 줄>
````

## Persistence

- `~/.claude/plugins/config/launcelot-lawyer-pro/<scope>/feature-risk/<target-id>-<YYYY-MM-DD>-<HHMM>.md`
  `<scope>`은 활성 매터에 따라 `matters/<slug>` 또는 프랙티스-레벨.
- `<scope>/feature-risk/_index.yaml`:

  ```yaml
  - target: <target-id>
    at: <ISO-datetime>
    probability_label: <…>
    severity_max: <…>
    recommended_mitigation: <A | B | … | none>
    led_to: <gap close | rewrite | hold | pattern add | none>
    snippets_fetched: <n>
  ```

## 권고 결정 로직

본 스킬은 사용자 risk appetite로 권고를 바꾸지 않는다. 권고는 다음 룰로만 결정한다.

- Axis 3에 `형사 입건 가능성: 명백`이 있으면 → 영상 보류 또는 단락 재구성 권고.
- Axis 1의 후보 중 항변 가능성이 모두 약하면 → 라인 수정 권고.
- Axis 1의 후보 중 일부에 강한 항변이 있지만 라인이 항변 단서를 결여하면 → 자막/단서 보완 권고.
- Axis 2가 `고위험 알려진 패턴`인데 라인이 그 패턴과 직접 매칭되면 → 영상 보류 권고.
- 그 외 → 라인 수정 또는 면책 고지 권고.

여러 룰이 동시에 적용되면 가장 강한 권고 하나만 선택. 두 개 권고를 동시에 내지 않는다.

## Failure handling

- **모든 후보 snippet fetch 실패**: 본 스킬은 결과를 만들지 않는다. 사용자에게 다음 메시지 출력 후 종료.

  ```
  ⛔ Snippet 확보 실패 — feature-risk-assessment 결과를 만들 수 없습니다.
  후보 죄목/규정: <list>
  시도한 출처: <…>
  3단 fallback(WebFetch → firecrawl → jina) 모두 실패.
  사용자가 URL 또는 원문을 직접 제공하면 그 텍스트로 다시 시도합니다.
  ```

- **일부 후보 snippet 실패**: 그 후보를 Axis 1에서 *제거*하고 사유를 별도 섹션에 명시. 남은 후보로만 4축 분해 진행. 단, 라인의 핵심 위험 후보(예: 명백 라인에서 형법 307)가 빠지면 결과를 발행하지 않고 종료(위 항목과 동일 처리).

- **입력 라인이 너무 짧거나 맥락이 부족**: 사용자에게 추가 맥락(전후 2~3 라인) 요청. 맥락 없이 진행하지 않는다.

- **동일 라인에 대한 직전 평가가 7일 이내에 존재**: 새 평가 작성 전, 직전 평가와의 변화 한 줄을 추가. 동일 결론이면 새 파일 없이 직전 평가에 `Revisited at: <date>` 라인을 append.

## What this skill does NOT do

- 라인을 수정하지 않는다. 수정안은 policy-redraft.
- 갭을 만들지 않는다. 갭 생성은 policy-diff → gap-surfacer.
- 게시 권고를 내지 않는다. 게시 권고는 launch-review.
- 새로운 조문·판례를 단정하지 않는다. raw_quote에 없는 텍스트는 인용하지 않는다.
- 사용자 프로필·스타일·리스크 감수도로 권고를 조정하지 않는다.
- 외부 스킬(`launcelot-lawyer` 포함)에 조문 검증을 위임하지 않는다.
