---
name: launch-review
description: >
  한국 변호사 유튜브 대본 1건을 7개 카테고리(변호사법·광고규정 / 명예훼손·모욕 /
  저작권 / 개인정보 / 무자격 자문 경계 / 표시광고법 / 인접 영역 자문)별 체크리스트로
  검토해 종합 메모를 생성한다. marketing-claims-review가 라인별 추출이라면 본 스킬은
  대본 전체에 대한 카테고리 수준의 점검과 게시 권고(게시 가능 / 수정 후 게시 / 비게시
  권고)를 출력한다. 사용자가 "대본 종합 검토", "이 대본 올려도 돼?", "발행 전 점검",
  "launch review"라고 말하거나 게시 직전 대본을 붙여넣을 때 실행한다.
argument-hint: "[대본 텍스트 또는 파일 경로]"
---

# /launcelot-lawyer-pro:launch-review

This skill is the publication gate. It takes a full script, runs seven category-level checklists, and emits a publish-readiness memo. It does not replace `marketing-claims-review` — it consumes its output and adds the macro-level reading.

Use `launch-review` when the script is near-final and the user wants a single "ship / fix / hold" call. Use `marketing-claims-review` earlier in drafting when the user wants every flagged line surfaced.

모든 조문·판례 인용은 본 스킬이 _직접_ `references/snippet-protocol.md` 절차로 fetch한 raw_quote에서만 만들어진다. 외부 스킬에 위임하지 않는다.

## Run order

1. 매터가 활성화돼 있으면 `matters/<slug>/matter.md`만 사실 컨텍스트로 읽는다. 그 외 설정 파일은 로드하지 않는다. 결과 폴더(`reviews/`)는 없으면 자동 생성.
2. Receive the script. Accept paste, file path, or wiki slug.
3. Compute the script identifier (`slug`). Order of preference: explicit `--slug` argument; filename stem if a file was given; wiki slug if a wiki path was given; first 40 characters of the script title (Korean-safe slugification: lowercase ASCII for non-Hangul tokens, Hangul preserved, spaces → hyphen). Append `-<YYYY-MM-DD>` only if the same slug already exists in the reviews directory for the same date.
4. Check `~/.claude/plugins/config/launcelot-lawyer-pro/reviews/<slug>.md`. If it exists, load it. Surface the prior recommendation, prior decisions, and any unresolved items at the top of the new run. Do NOT silently overwrite — the prior memo is appended to as a new revision section.
5. Call `/launcelot-lawyer-pro:marketing-claims-review` on the script. Cache its per-line output (saved to `reviews/<slug>.claims.<ISO-datetime>.md`). marketing-claims-review가 결과 발행 자체를 거부하면(=모든 snippet fetch 실패) 본 스킬도 동일하게 거부하고 종료한다.
6. Run each of the seven category checklists end-to-end across the entire script. 각 체크리스트 항목이 특정 조문·규정에 근거하면 그 조문은 snippet-protocol로 fetch한 raw_quote를 메모에 첨부한다.
7. For each category, classify: `이슈 없음 / 경미 / 중대 / 발행 보류 권고`.
8. Emit the memo to chat AND save it. See `## Persistence` below for exact write rules.

## Categories

Run all seven. Skipping a category requires a written reason in the output.

### Category 1 — 변호사법 23조 및 대한변협 광고규정

Checklist (apply to the script as a whole, not just one line):

1. 직역 표시. 채널이 "변호사", "법무법인" 등 자격 표시를 정확히 하고 있는가. 전문자격 (예: "전문변호사")의 표시는 대한변협 인증 기준을 충족하는가.
2. 비교 광고. 다른 변호사·법무법인·유사 직역에 대한 비교 표현이 있는가. 있다면 광고규정 허용 범위에 있는가.
3. 결과 보장 표현. "무조건 승소", "100% 무죄", "확정적 결과" 등의 표현. 광고규정상 금지에 해당하는 패턴.
4. 사건 유치 직접 유도. 특정 사건에 연관된 시청자를 식별 가능한 방식으로 호명하여 수임을 권유하는가.
5. 비변호사와의 동업·소개료. 비변호사 플랫폼·서비스를 추천하면서 변호사법 34조 저촉 가능성이 있는가.
6. 사실과 다른 경력·실적 표시.

Output rows expected: 각 항목별 (해당 없음 / 해당 — 라인 번호 / 해당 — 라인 번호 + 위험 등급).

### Category 2 — 명예훼손 · 모욕 (형법 307·309·311, 정통망법 70)

Checklist:

1. 식별 가능한 제3자 (실명, 직책, 사건번호, 묘사로 특정 가능) 등장 여부.
2. 사실 적시인지, 의견 표현인지, 혼합인지.
3. 적시 사실의 진실성 입증 가능성 (substantiation on file?).
4. 공공의 이익 신호 (형법 310 진실성·공익성 항변의 단서).
5. "출판물에 의한 명예훼손" 가중 (형법 309) 적용 가능성 — 유튜브는 통상 정통망법 70로 가지만, 영상·자막·썸네일 결합으로 309가 함께 거론될 수 있는 패턴인지.
6. 모욕적 평가 표현이 사실 적시와 분리되어 단독으로 존재하는가.

이 카테고리는 marketing-claims-review의 Bucket A·B·C 라인을 모두 모아 카테고리 수준에서 재평가한다.

### Category 3 — 저작권법 (특히 28조 인용)

Checklist:

1. 언론사 기사·사진을 인용하는가. 인용이 공표된 저작물이며, 정당한 범위 안에서, 공정한 관행에 합치되며, 출처 표시가 되어 있는가.
2. 판결문 전문 또는 상당 부분을 화면에 그대로 띄우는가.
3. 타 변호사·전문가의 영상·블로그를 화면에 띄우는가.
4. 음악·이미지·폰트 사용권.

### Category 4 — 개인정보 보호법

Checklist:

1. 실존 인물의 식별 정보 (성명, 주소, 연락처, 직장, 학력, 가족관계, 형사·의료 정보) 노출 여부.
2. 미성년자 식별 정보.
3. 사진·영상·음성에서 모자이크·음성변조 처리 누락.
4. 의뢰인 정보의 비밀유지의무 (변호사법 26, 변호사윤리장전 18조) 저촉 가능성.
5. 사건 당사자가 비공개를 요청한 정보의 노출.

### Category 5 — 무자격 자문 경계

Checklist:

1. 자작 가이드 패턴. 시청자가 변호사 도움 없이 답변서·고소장·내용증명·계약서 등을 직접 완성하도록 안내하는가.
2. 일반 법률 정보와 개별 사건 자문의 경계. 특정 시청자 상황에 대한 직접적 결론 제시인가.
3. 면책 고지의 유무.

### Category 6 — 표시광고법 (공정거래위원회 소관)

Checklist:

1. 실증되지 않은 사실 적시 ("판례 다수 확보", "최고 승소율", "전국 최대" 등).
2. 비교 우위 표시의 실증 가능성.
3. 추천·보증 형식의 자료가 실재하는 후기·평가에 근거하는가.
4. 가격·수임료 표시의 정확성.

### Category 7 — 인접 영역 무자격 자문 경계

Checklist:

1. 의료 (의료법). 진단·치료 방법·예후에 관한 단정.
2. 세무 (세무사법). 신고·환급 절차에 관한 개별 자문.
3. 회계 (공인회계사법).
4. 금융투자 (자본시장법). 특정 종목·상품 매수·매도 권유.
5. 행정사·법무사 직역 침해 또는 역침해.

## Output memo

```markdown
[게시 권고 — <게시 가능 | 수정 후 게시 | 발행 보류 권고>]

# Launch Review: <대본 제목 또는 슬러그>

검토 일자: <YYYY-MM-DD>
검토 대상: <대본 식별자 + 길이>
원본 라인별 검토: marketing-claims-review 캐시 <경로 또는 ID>
Snippet fetch 결과: <n attempted> / <n successful>

---

## 종합 권고

<3문장 이내. 권고가 "발행 보류"라면 핵심 사유 한 줄, 핵심 라인 인용 한 줄, 그
한 라인만 고치면 등급이 어디로 가는지 한 줄.>

## 카테고리별 결과

| 카테고리             | 결과 | 핵심 라인 | 비고 |
| -------------------- | ---- | --------- | ---- |
| 1. 변호사법·광고규정 | <…>  | <라인 #>  | <…>  |
| 2. 명예훼손·모욕     | <…>  | <라인 #>  | <…>  |
| 3. 저작권            | <…>  | <라인 #>  | <…>  |
| 4. 개인정보          | <…>  | <라인 #>  | <…>  |
| 5. 무자격 자문 경계  | <…>  | <라인 #>  | <…>  |
| 6. 표시광고법        | <…>  | <라인 #>  | <…>  |
| 7. 인접 영역 자문    | <…>  | <라인 #>  | <…>  |

## 발행 보류 사유 (있을 때만)

<해당 라인 인용 + 카테고리 + 위험 등급 + 수정 방향 한 줄. 각 사유에 근거한 조문/판례의 snippet 객체를 첨부.>

## 수정 후 게시 항목

각 항목에 대해:

- 라인: "<원문>"
- 문제: <카테고리 + 사유>
- 근거 snippet: <조문/판례 identifier + source + fetched_at>
- 수정안: "<대안 표현>"
- 수정 후 예상 등급: <…>

## 인용된 조문·판례 (snippet 목록)

대본 평가에 사용된 모든 조문·판례 raw_quote 객체를 한곳에 모아 제시. 본 메모 안에서 인용된 모든 텍스트는 이 목록의 raw_quote에 닿아야 한다.

## 작성자 결정 메모

<게시 가능>이라도 작성자가 알아야 할 정황 (예: "공공의 이익 항변에 의존하고
있으므로 진실성 입증 자료를 파일로 보관할 것").
```

## Decision rules

- Any `명백` band line in `marketing-claims-review` automatically pushes the overall recommendation to at least `수정 후 게시`. Two or more `명백` lines → `발행 보류 권고` unless every one of them has an accepted suggested fix recorded by the user.
- Any line marked `snippet_missing` in marketing-claims-review → 본 라인은 결과 등급 산정에서 *제외*하되 메모 상단에 `미평가 라인: <list>`로 별도 표시. 이런 라인이 핵심 의문 라인이면 사용자가 URL 또는 원문을 제공할 때까지 종합 권고는 `수정 후 게시`를 넘지 못한다.
- Category-level severity is the max of its constituent line bands, never an average.
- 본 스킬은 사용자 risk appetite·Hard rules·Speech policy로 권고를 조정하지 않는다. 권고는 한국법 자체와 fetch된 raw_quote만 보고 결정한다.

## Persistence

Every run writes to disk. Two files per script slug, plus an index.

### Files

- `~/.claude/plugins/config/launcelot-lawyer-pro/reviews/<slug>.md`
  The canonical review memo. Append-only at the section level. New runs append a new `## Revision <N> — <ISO-datetime>` block; the prior content is preserved verbatim above it. The top of the file maintains a `Current status:` line that reflects the latest revision.

- `~/.claude/plugins/config/launcelot-lawyer-pro/reviews/<slug>.claims.<ISO-datetime>.md`
  The marketing-claims-review output snapshot for that revision. One file per revision. Never overwritten.

- `~/.claude/plugins/config/launcelot-lawyer-pro/reviews/_index.yaml`
  Index of all reviewed scripts. One entry per slug:

  ```yaml
  - slug: <slug>
    title: <script title>
    first_reviewed: <ISO-date>
    last_reviewed: <ISO-date>
    revisions: <N>
    current_recommendation: <게시 가능 | 수정 후 게시 | 발행 보류 권고>
    current_band_max: <명백 | 가능 | 회색지대 | 안전>
    unresolved_lines: <N>
    snippet_missing_lines: <N>
    decision_log:
      - revision: 1
        at: <ISO-datetime>
        recommendation: <…>
        decided_by: <user handle or 'pending'>
        action: <게시함 | 수정함 | 보류 | 비공개결정>
        note: <free text, optional>
  ```

### Memo template (revision block)

Each new revision appends this block to `reviews/<slug>.md`:

```markdown
## Revision <N> — <ISO-datetime>

[게시 권고 — <게시 가능 | 수정 후 게시 | 발행 보류 권고>]

검토자: <user>
입력 본문 식별자: <file path | wiki slug | paste hash>
이전 리비전과의 차이: <첫 리비전이면 "최초 검토". 이후 리비전이면 변경된 라인
수와 추가/해소된 카테고리 요약. 같은 카테고리에서 위험 등급이 어떻게 이동했는지
한 줄.>
Snippet fetch 결과: <n attempted> / <n successful>
연결 파일: claims = reviews/<slug>.claims.<ISO-datetime>.md

(이하 원래 메모 본문 — 종합 권고, 카테고리 표, 발행 보류 사유, 수정 후 게시
항목, snippet 목록, 작성자 결정 메모)
```

### Read-before-write

Before every run, read the existing `reviews/<slug>.md` if present. The chat
output must lead with one of:

- `최초 검토` — slug 신규.
- `이전 리비전 N건 발견 — 직전 권고: <…>, 미해소 라인: <N>` — slug 기존.

If the user explicitly says "이거 처음부터 다시 검토" (or `--fresh`), do NOT
overwrite. Instead, archive the existing file to
`reviews/_archive/<slug>.<ISO-datetime>.md` and start a fresh `reviews/<slug>.md`
with revision 1. Update `_index.yaml` with a new `archived:` reference.

### Decision logging

After emitting the memo, prompt the user once:

> 이 리비전에 대한 결정을 기록할까? (게시함 / 수정함 / 보류 / 비공개결정 / 나중에)

If the user gives a decision, append it to the `decision_log:` in
`_index.yaml`. If "나중에", set `decided_by: pending` and move on. Never assume
a decision.

## Failure handling

- **marketing-claims-review가 결과 발행 거부** (모든 라인의 snippet 확보 실패): 본 스킬도 동일 사유로 결과를 만들지 않는다. 사용자에게 동일 메시지를 그대로 전달하고 종료.
- **일부 라인 snippet 미확보**: 그 라인을 `미평가 라인`으로 별도 표시. 종합 권고는 `수정 후 게시` 이하로 유지.
- **marketing-claims-review가 pull-bucket hit 0건**: 7개 카테고리 체크리스트는 그대로 실행한다. 라인 추출에 잡히지 않는 publication-stage 이슈(저작권 인용 형식, 면책 고지 누락, 표시광고 실증 자료 미보관 등)는 카테고리 수준에서만 보인다.

## What this skill does NOT do

- Does not generate full rewritten scripts. Hand off to `koreanizer` / per-attorney writing skill for production rewrites.
- Does not approve publication. The recommendation is the validator's read; the attorney makes the publish call.
- Does not modify any source script file. Persistence is to the plugin config directory only; the user's original script files are never edited.
- Does not delegate 조문·판례 검증 to any external skill (`launcelot-lawyer` included). 본 스킬과 marketing-claims-review가 snippet-protocol로 직접 fetch한다.
- Does not consult any "Hard rules" / "Speech policy" / "Risk calibration" — 본 플러그인은 그런 설정을 받지 않는다.
