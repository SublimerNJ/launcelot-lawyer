# Snippet Protocol — 조문·판례 인용 검증 절차

본 플러그인의 모든 스킬은 조문·판례·규정을 인용할 때 **반드시 본 프로토콜을 따른다**. 모델 지식으로 조문을 단정하거나, 외부 스킬에 검증을 위임하지 않는다. 모든 인용은 원문 fetch에 성공한 텍스트(=스니펫)에서만 만들어진다.

## 핵심 원칙

1. **검증 없는 인용 금지** — 조문 번호·판결 번호·규정 조항은 fetch한 원문 텍스트에서 _그대로 잘라낸_ 스니펫에 한해 출력한다.
2. **위임 금지** — 본 플러그인 내부에서 fetch → 검증 → 인용을 끝낸다. 다른 스킬(예: `launcelot-lawyer`)에 넘기지 않는다.
3. **실패 시 결과물 차단** — 3단 fallback이 모두 실패하면 결과 자체를 만들지 않고 멈춘다. "스니펫 미확보" 라벨로 우회 출력하지 않는다.

## 3단 Fetch Fallback

조문·판례 인용이 필요한 모든 지점에서 다음 순서로 시도한다.

### Step 1 — WebFetch (default)

내장 `WebFetch` 도구로 출처 URL을 직접 호출한다. 출처별 base URL:

- 법령: `https://www.law.go.kr/법령/<법령명>` 또는 `https://www.law.go.kr/LSW/lsInfoP.do?...`
- 대법원 판례: `https://glaw.scourt.go.kr/wsjo/panre/sjo060.do?...`
- 헌재 결정: `https://search.ccourt.go.kr/ths/pr/...`
- 변협 광고규정: `https://www.koreanbar.or.kr` 하위 광고규정·공지 페이지
- 공정위 심결: `https://www.ftc.go.kr/www/selectReportUserList.do?...`
- 개보위 결정: `https://www.pipc.go.kr` 하위 결정·의결 페이지
- 국회의안: `https://likms.assembly.go.kr/bill/billDetail.do?billId=...`

`WebFetch` prompt는 다음 항목을 추출하도록 작성한다.

```
다음을 그대로 추출하라. 요약·해석 금지.
1. 페이지에 명시된 정식 명칭(법령명·판결번호·심결번호 등)
2. 본 인용에 필요한 조항/판시사항의 원문 텍스트 (한 글자도 바꾸지 말 것)
3. 시행일 또는 선고일 (조문이면 현행 시행일, 판례면 선고일자)
4. 페이지 마지막 갱신·확인 표시(있는 경우)
5. 만약 위 정보를 페이지에서 찾을 수 없으면 정확히 "NOT_FOUND_ON_PAGE"를 출력
```

응답이 `NOT_FOUND_ON_PAGE`이거나 HTTP 오류·타임아웃이면 Step 2로.

### Step 2 — firecrawl 스킬

`firecrawl` 스킬을 invoke해 동일 URL을 스크래핑한다. SPA·JS 렌더링·한자 인코딩 깨짐 페이지(특히 glaw·ccourt) 대응. 추출 prompt는 Step 1과 동일.

응답이 비거나 추출 실패면 Step 3으로.

### Step 3 — jina MCP read_url

`mcp__jina__read_url` 도구로 같은 URL을 호출한다. Jina가 markdown으로 정제해서 반환하므로 원문 추출 성공률이 가장 높음.

응답이 비거나 인용에 필요한 조항/판시사항을 포함하지 않으면 Step 4로.

### Step 4 — 결과 생성 차단

3단계 모두 실패하면 다음 형식으로 사용자에게 보고하고 **현재 스킬의 결과물을 만들지 않는다**.

```
⛔ 스니펫 확보 실패 — 결과 생성 불가

대상: <법령명 또는 판례 식별자>
시도한 출처: <Step 1 URL>, <Step 2 URL>, <Step 3 URL>
실패 사유:
- WebFetch: <HTTP 코드 또는 NOT_FOUND_ON_PAGE>
- firecrawl: <에러 메시지>
- jina: <에러 메시지>

조치:
1. 사용자가 URL을 직접 제공하면 같은 절차를 그 URL로 다시 시도한다.
2. 사용자가 원문을 paste하면 그 텍스트를 raw_quote로 받아 진행한다(이 경우 source는 "user-pasted"로 표시).
3. 둘 다 없으면 본 검토 항목을 결과에서 제외한다. 결과의 다른 부분에도 해당 조문을 근거로 한 평가가 있으면 그 항목 역시 제거 또는 보류 처리한다.
```

## 스니펫 객체 스키마

스킬이 인용을 출력할 때마다 다음 필드를 함께 표기한다. yaml·markdown 어느 출력 포맷이든 동일하다.

```yaml
snippet:
  source: "law.go.kr" # 또는 "glaw.scourt.go.kr", "ccourt.go.kr", "koreanbar.or.kr", "ftc.go.kr", "pipc.go.kr", "likms.assembly.go.kr", "user-pasted"
  url: "https://www.law.go.kr/법령/형법/제307조"
  fetched_at: "2026-05-16T12:34:00+09:00"
  fetch_method: "WebFetch" # 또는 "firecrawl", "jina"
  identifier: "형법 제307조 제1항" # 또는 "대법원 2020. 12. 10. 선고 2017도2891 판결" 같은 정식 인용
  effective_date: "2024-02-09" # 조문 시행일 또는 판결 선고일
  raw_quote: |
    공연히 사실을 적시하여 사람의 명예를 훼손한 자는 2년 이하의 징역이나 금고
    또는 500만원 이하의 벌금에 처한다.
```

본문 인용은 항상 `raw_quote` 안의 텍스트만 사용한다. 본문에 다음과 같이 inline으로 표시한다.

```
「형법 제307조 제1항」(law.go.kr, fetched 2026-05-16): "공연히 사실을 적시하여 …"
```

## 캐싱

- 같은 conversation 안에서 동일 `(url, identifier)` 조합에 대해서는 첫 fetch 결과를 재사용해도 된다.
- 새 conversation, 새 스킬 실행에서는 다시 fetch한다. 사용자 turn 사이의 fetch 결과를 무한 캐싱하지 않는다.
- 캐시한 스니펫을 출력할 때도 `fetched_at`은 *최초 fetch 시점*을 그대로 유지한다.

## 무엇을 절대 하지 않는가

- 조문 번호·판결 번호를 모델 지식으로 보충하지 않는다.
- 페이지에서 추출하지 못한 조항의 텍스트를 "일반적으로는 …"으로 풀어 쓰지 않는다.
- 3단 fallback 결과를 "대략 이런 의미일 것"으로 의역해 결과에 넣지 않는다.
- 외부 스킬(`launcelot-lawyer` 포함)에 조문 검증을 위임하지 않는다.
- 스니펫 없이 "이 라인은 형법 307조 위반 소지가 있다"고 결론지지 않는다. 검토 결론을 내려면 그 조문의 스니펫이 결과물 안에 함께 들어 있어야 한다.
