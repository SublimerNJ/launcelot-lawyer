---
name: reg-feed-watcher
description: >
  한국 공식 출처(국가법령정보센터, 대법원 종합법률정보, 헌법재판소, 국회의안정보시스템,
  대한변호사협회, 공정거래위원회, 개인정보보호위원회, 한국저작권위원회 등)에서
  신착을 끌어와 materiality threshold로 필터링한 디제스트를 출력한다. 사용자가 "신착
  확인", "오늘 새로운 거", "regulatory digest", "feed check"라고 말하거나 cadence가
  도래했을 때 실행한다. Tier 1 변경은 즉시 policy-diff 핸드오프를 권유한다.
argument-hint: "[--since <YYYY-MM-DD>] [--source <slug>] [--tier 1|2|3]"
---

# /launcelot-lawyer-pro:reg-feed-watcher

This skill is the periodic eye on Korean regulatory sources. 각 출처의 신규 게시물을 본 스킬이 _직접_ fetch한다. 외부 스킬에 위임하지 않는다.

모든 fetch는 `references/snippet-protocol.md`의 3단 fallback(WebFetch → firecrawl → jina)을 따른다. 어떤 항목도 모델 지식으로 보충하지 않는다.

## Run order

1. Watchlist 로드:
   - 사용자 watchlist 파일(`~/.claude/plugins/config/launcelot-lawyer-pro/watchlist.yaml`)이 있으면 그것 사용.
   - 없으면 `references/default-watchlist.md`의 10개 출처를 그대로 사용. 디렉토리(`digests/`)는 없으면 자동 생성.
2. Materiality 정의: `references/default-watchlist.md`의 `tier1` / `tier2` / `tier3` 정의를 사용. 사용자가 자연어로 "이 카테고리 빼" / "이 키워드만"이라고 말한 적이 있어 watchlist.yaml에 반영돼 있으면 그것 추가 적용.
3. Cadence 계산: 마지막 성공 실행 시각을 `digests/_index.yaml`에서 읽어 `--since` 기본값으로 사용. 명시적 `--since`가 있으면 그것 우선. quiet hours(KST 23:00~07:00)에는 디제스트 표시 보류.
4. 각 watchlist 출처에 대해 **본 스킬이 snippet-protocol로 직접 fetch**. 신착 인덱스 페이지(예: law.go.kr의 최근 시행 법령, glaw.scourt.go.kr의 최근 판례)에서 since 이후 항목 식별. 후보 항목마다 다시 fetch해 raw_quote 확보.
5. 각 신착 항목을 Tier 1 / Tier 2 / Tier 3 / drop으로 분류.
6. 디제스트 발행.
7. Tier 1 항목은 즉시 `policy-diff` 핸드오프를 *권유*한다(자동 호출 금지). 사용자 명시 후에만 진행.
8. Tier 2 항목은 디제스트 본문에 포함. 자동 핸드오프 없음.
9. Tier 3 항목은 로그만.

## Item shape

각 항목은 다음 형식으로 정규화한다.

```yaml
item_id: <source-slug>-<YYYY-MM-DD>-<hash6>
source: <name from watchlist>
source_url: <permalink>
source_tier: 공식
title: <…>
type: <조문개정 | 판례 | 결정 | 광고규정개정 | 공지 | 가이드라인 | 의안발의 | 기타>
date: <YYYY-MM-DD>
snippet:
  identifier: <…>
  url: <…>
  fetched_at: <ISO-datetime>
  fetch_method: <WebFetch | firecrawl | jina>
  effective_date: <…>
  raw_quote: |
    <원문 텍스트 — 본 스킬이 직접 fetch한 결과>
keywords_matched: [<list from filter>]
materiality_tier: <1 | 2 | 3 | drop>
materiality_reason: <default-watchlist.md의 Tier 정의 중 어느 항목과 매칭됐는지 인용>
diff_candidate: <yes | no> # 형법·정통망법·개보법·변호사법·광고규정·표시광고법 본문 또는 해석 변경이며 사용자 라이브러리에 영향 가능 여부
```

raw_quote가 없는 항목은 디제스트에 _포함하지 않는다_. snippet 미확보 항목은 메타 섹션의 `snippet_missing`에만 카운트.

## Digest output

Path: `~/.claude/plugins/config/launcelot-lawyer-pro/digests/<YYYY-MM-DD>.md` (사용자 별도 지정 있으면 그것).

```markdown
# Regulatory digest — <YYYY-MM-DD>

기간: <since> ~ <until>
실행 트리거: <user | scheduled | reg-feed-watcher --since>
Watchlist: <user-customized | default-watchlist.md>
Snippet fetch: <n attempted> / <n successful>
범위 제한: <none | …>

---

## Tier 1 — 즉시 검토

(없으면 "해당 없음. 본 cadence 구간에 즉시 알림 사유 없음.")

### <item title>

- 출처: <source name> (공식)
- 일자: <YYYY-MM-DD>
- 종류: <…>
- Snippet raw_quote:

  > <원문 인용 — fetched_at, source, url 함께>

- 영향 가능 영역: <명예훼손 | 광고규정 | …>
- 권장 후속: `/launcelot-lawyer-pro:policy-diff <item_id> library`
- 원문 URL: <…>

---

## Tier 2 — 정기 디제스트

| 일자 | 출처 | 종류 | 제목 | 영향 영역 | Snippet URL |
| ---- | ---- | ---- | ---- | --------- | ----------- |
| <…>  | <…>  | <…>  | <…>  | <…>       | <…>         |

각 행의 snippet raw_quote는 본문 하단의 "Snippet 첨부" 섹션에 모아둔다.

---

## Tier 3 — 로그만

(테이블 또는 한 줄 항목. raw_quote는 별도 저장하지 않음.)

---

## Snippet 첨부 (Tier 1·2 본문 인용용)

각 항목의 snippet 객체(yaml)를 그대로 첨부.

---

## 메타

- 처리한 출처 수: <N>
- 새 항목 수: <N>
- Tier 1 / 2 / 3 분포: <…>
- snippet_missing: <N>
- 다음 cadence 예정: <…>
- 범위 제한 / 누락: <…>
```

## Persistence

- `digests/<YYYY-MM-DD>.md` — 본 실행의 디제스트.
- `digests/_index.yaml` — 모든 디제스트의 인덱스. 다음 cadence 계산을 이 인덱스로 한다:

  ```yaml
  - date: <YYYY-MM-DD>
    triggered_by: <user | scheduled>
    since: <…>
    until: <…>
    counts: { tier1: <n>, tier2: <n>, tier3: <n>, snippet_missing: <n> }
    diff_handoffs: [<item_id>, …]
  ```

- `digests/_seen.yaml` — 모든 본 적 있는 item_id의 해시 집합. 중복 게재 방지.
- `digests/_unresolved.md` — 어느 디제스트에서 Tier 1로 띄웠으나 policy-diff 핸드오프가 아직 실행되지 않은 항목 누적 목록. 매 실행 시 맨 앞에 표시.

## Watchlist 자연어 조정

사용자가 디제스트를 보고 "공정위는 빼" / "저작권위는 분기에 한 번만" / "변협만 추적해"라고 말하면, 본 스킬은 그 지시를 `watchlist.yaml`에 반영한다(파일이 없으면 default를 베이스로 새로 생성). 별도 customize 단계 없음.

반영 형식:

```yaml
sources:
  - id: <default-watchlist의 id 또는 새 id>
    name: <…>
    url: <…>
    include: <true | false> # 자연어 "이 출처 빼"
    cadence_override: <weekly | 3-day | monthly | …>
    extra_filter: <…> # "이 키워드만"
    excluded_keywords: [<…>] # "이건 빼"
    reason: <자연어 지시 원문>
    set_at: <ISO-datetime>
```

신뢰등급(`trust_tier`)은 default-watchlist의 10개 출처에 한해 "공식"으로 박혀 있다. 새 출처를 사용자가 추가하면 그 출처에 대해서는 등급을 자동 부여하지 않고, 디제스트 첫 노출 시 한 번만 "공식 / 준공식 / 민간 중 어디인가?"를 묻는다.

## Failure handling

- **모든 출처 fetch 실패** (Step 1·2·3 모두 fail): 디제스트를 만들지 않고 종료. 사용자에게 다음 메시지 출력.

  ```
  ⛔ Snippet 확보 실패 — regulatory digest를 만들 수 없습니다.
  시도한 출처: <list>
  3단 fallback(WebFetch → firecrawl → jina) 모두 실패.
  네트워크 접근성을 확인하거나, 일시적 장애일 수 있으니 잠시 후 재시도.
  ```

- **일부 출처만 실패**: 그 출처를 `메타`의 "조회 실패 — <reason>"에 기록하고 나머지 출처로 디제스트 발행.
- **한 출처 응답이 빈손이면** (`new_items: []`): `메타`에 그 출처를 "조회됨, 신착 없음"으로 기록.
- **한 항목의 snippet 미확보**: 그 항목은 디제스트에서 제외하되 `snippet_missing` 카운트만 메타에 표시.

## Handoff contract

policy-diff로 핸드오프할 때 다음을 함께 전달:

- `slot_a.id = <item_id>`
- `slot_a.snippet = <snippet 객체 그대로>`
- `triggered_by: reg-feed-watcher:<digest-id>`
- 사용자가 핸드오프 직전 단계에서 "library 전체" / "특정 슬러그 N개" / "검토 보류"를 선택.

gap-surfacer로의 직접 핸드오프는 본 스킬이 하지 않는다(반드시 policy-diff를 거친 결과만 gap이 된다).

## What this skill does NOT do

- 모델 지식으로 "이 법령은 X로 개정되었다"라고 출력하지 않는다. 모든 인용은 raw_quote(스니펫)에서만 만들어진다.
- 외부 스킬(`launcelot-lawyer` 포함)에 fetch를 위임하지 않는다.
- 자동으로 policy-diff를 실행하지 않는다. Tier 1 항목조차 사용자의 명시적 허가 없이 diff를 돌리지 않는다.
- 사용자의 대본을 읽지 않는다. 대본 영향 분석은 policy-diff에서 한다.
- 디제스트 첫 실행을 위해 사용자에게 출처 카탈로그를 묻지 않는다. default-watchlist로 바로 시작한다.
