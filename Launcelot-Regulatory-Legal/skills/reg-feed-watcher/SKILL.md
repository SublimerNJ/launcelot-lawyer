---
name: reg-feed-watcher
description: >
  한국 공식 출처(국가법령정보센터, 대법원 종합법률정보, 헌법재판소, 국회의안정보시스템,
  대한변호사협회, 공정거래위원회, 개인정보보호위원회, 한국저작권위원회)에서 사용자
  watchlist에 등록된 출처의 신착을 launcelot-lawyer를 경유해 끌어와 materiality
  threshold로 필터링한 디제스트를 출력한다. 사용자가 "신착 확인", "오늘 새로운
  거", "regulatory digest", "feed check"라고 말하거나 cadence가 도래했을 때
  실행한다. Tier 1 변경은 즉시 policy-diff 핸드오프를 권유한다.
argument-hint: "[--since <YYYY-MM-DD>] [--source <slug>] [--tier 1|2|3]"
---

# /launcelot-regulatory-legal:reg-feed-watcher

This skill is the periodic eye on Korean regulatory sources. It does not fetch raw HTML on its own — every source query goes through `launcelot-lawyer`, which is responsible for verifying that anything reported actually exists at the cited URL on the date claimed.

## Run order

1. Load `~/.claude/plugins/config/launcelot-regulatory-legal/CLAUDE.md` — `## Watchlist`, `## Materiality threshold`, `## Cadence`, `## Integrations`.
2. Read `references/source-catalog.md` once to know each source's operator and trust tier.
3. Compute the `--since` cutoff. Default: last successful run timestamp from `digests/_index.yaml`, falling back to today minus the cadence window. Honor explicit `--since`.
4. Coverage check (before any fetching): compare the watchlist against the catalog. If a tier-1 category in the user's materiality threshold has zero sources in the watchlist, surface the gap once and ask whether to continue or pause to add the source.
5. For each watchlisted source, hand the query to `launcelot-lawyer` with the cutoff and the source's filter. Collect the structured response (`new_items: []`).
6. Classify each new item into Tier 1 / Tier 2 / Tier 3 / drop, using the materiality threshold definitions stored in the config.
7. Persist the digest.
8. For Tier 1 items, offer immediate `policy-diff` handoff. Wait for user direction; do not auto-run.
9. For Tier 2 items, include them in the digest body. No automatic handoff.
10. For Tier 3 items, write to the log only.

## Item shape

Each item, after launcelot-lawyer verification, is normalized to:

```yaml
item_id: <source-slug>-<YYYY-MM-DD>-<hash6>
source: <name from watchlist>
source_url: <permalink>
source_tier: <공식 | 준공식 | 민간>
title: <…>
type: <조문개정 | 판례 | 결정 | 광고규정개정 | 공지 | 가이드라인 | 의안발의 | 기타>
date: <YYYY-MM-DD>
launcelot_verification: <verified-current | verified-but-superseded | unverified | not-found>
summary: <launcelot-lawyer가 1차 추출한 한 문단 요약. 본 스킬이 모델 지식으로 임의 보충하지 않는다.>
keywords_matched: [<list from filter>]
materiality_tier: <1 | 2 | 3 | drop>
materiality_reason: <Module 3에서 사용자가 정의한 raw 텍스트 인용>
diff_candidate: <yes if 형법·정통망법·개보법·변호사법·광고규정·표시광고법 본문 또는 해석 변경이며 사용자의 대본 라이브러리에 영향 가능 | no>
```

## Digest output

Path: from `## Cadence` config; default `~/.claude/plugins/config/launcelot-regulatory-legal/digests/<YYYY-MM-DD>.md`.

Template:

```markdown
# Regulatory digest — <YYYY-MM-DD>

기간: <since> ~ <until>
실행 트리거: <user | scheduled | reg-feed-watcher --since>
launcelot-lawyer: <available | not available>
범위 제한: <none | …>

---

## Tier 1 — 즉시 검토

(없으면 "해당 없음. 본 cadence 구간에 즉시 알림 사유 없음.")

### <item title>
- 출처: <source name> (<공식 | 준공식 | 민간>)
- 일자: <YYYY-MM-DD>
- 종류: <…>
- 요약: <launcelot 인용>
- 영향 가능 영역: <명예훼손 | 광고규정 | …>
- 권장 후속: `/launcelot-regulatory-legal:policy-diff <item_id> library`
- 검증 상태: <…>
- 원문: <URL>

---

## Tier 2 — 정기 디제스트

| 일자 | 출처 | 종류 | 제목 | 영향 영역 | 검증 |
|---|---|---|---|---|---|
| <…> | <…> | <…> | <…> | <…> | <…> |

---

## Tier 3 — 로그만

(테이블 또는 한 줄 항목)

---

## 메타

- 처리한 출처 수: <N>
- 새 항목 수: <N>
- Tier 1 / 2 / 3 분포: <…>
- 미검증 항목 수 (`launcelot-lawyer unverified`): <N>
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
    counts: { tier1: <n>, tier2: <n>, tier3: <n>, unverified: <n> }
    diff_handoffs: [<item_id>, …]
  ```
- `digests/_seen.yaml` — 모든 본 적 있는 item_id의 해시 집합. 중복 게재 방지.
- `digests/_unresolved.md` — 어느 디제스트에서 Tier 1로 띄웠으나 policy-diff 핸드오프가 아직 실행되지 않은 항목 누적 목록. 매 실행 시 맨 앞에 표시.

## Coverage gap surfacing

Module 2(Watchlist)와 사용자의 Module 3(Materiality threshold)을 cross-check 한다:

- 사용자가 Tier 1에 "변협 광고규정 본문 개정"을 넣었는데 watchlist에 변협이 없다 → 디제스트 상단에 한 번만:
  > ⚠ 커버리지 누락: Tier 1 정의에 "변협 광고규정"이 포함되어 있으나 watchlist에 koreanbar.or.kr이 없다. `/launcelot-regulatory-legal:customize --field "Watchlist"`로 추가하거나, 본 cadence 구간에서 일회성으로 건너뛸 수 있다.
- 같은 누락에 대해 같은 디제스트에서 한 번만 표시한다. 사용자가 명시적으로 "skip [source] for now"라고 했다면 그 결정이 config에 기록된 동안은 다시 묻지 않는다.

## Failure handling

- launcelot-lawyer unavailable: 디제스트 헤더에 ⚠ 배너, 모든 새 항목은 `verification: unverified` 태그. 사용자에게 다음을 묻는다: (1) 사용자가 직접 URL을 열어 확인할 항목을 표시하고 진행, (2) 디제스트 생성을 보류하고 launcelot-lawyer가 복귀한 뒤 재시도. 어느 쪽인가?
- 한 출처에 대한 launcelot-lawyer 응답이 빈손이면 (`new_items: []`): 디제스트의 "메타" 섹션에만 그 출처를 "조회됨, 신착 없음"으로 기록.
- 한 출처에 대한 응답이 오류면: 메타에 "조회 실패 — <reason>"으로 기록하고 다음 cadence로 자동 재시도 큐에 등록.

## Handoff contract

policy-diff로 핸드오프할 때 다음을 함께 전달:

- `slot_a.id = <item_id의 인용>`
- `slot_a.text = <launcelot 요약 + 원문 URL>`
- `triggered_by: reg-feed-watcher:<digest-id>`
- 사용자가 핸드오프 직전 단계에서 "library 전체" / "특정 슬러그 N개" / "검토 보류"를 선택.

gap-surfacer로의 직접 핸드오프는 본 스킬이 하지 않는다(반드시 policy-diff를 거친 결과만 gap이 된다).

## What this skill does NOT do

- 본 스킬은 launcelot-lawyer 없이 어느 한국 공식 사이트의 콘텐츠도 단정하지 않는다. 모델 지식으로 "이 법령은 X로 개정되었다"라고 출력하지 않는다.
- 본 스킬은 자동으로 policy-diff를 실행하지 않는다. Tier 1 항목조차 사용자의 명시적 허가 없이 diff를 돌리지 않는다.
- 본 스킬은 사용자의 대본을 읽지 않는다. 대본 영향 분석은 policy-diff에서 한다.
