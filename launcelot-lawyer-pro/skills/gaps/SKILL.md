---
name: gaps
description: >
  갭 트래커의 사용자 친화 단일 진입점. gap-surfacer가 워크플로우 본체라면 본 스킬은
  "open gap 몇 개야", "이번 주 갭", "긴급한 거 뭐 있어", "가장 오래된 거", "패턴 갭",
  "이 매터 갭만"처럼 빠른 조회·요약 표시에 특화된다. 무거운 변경 액션(close,
  accept-risk, unpublish, snooze)은 gap-surfacer로 라우팅한다.
argument-hint: "[summary | urgent | aging | patterns | by-source <name> | by-matter <slug> | <gap-id>]"
---

# /launcelot-lawyer-pro:gaps

This skill is the front door for the gap tracker. It is read-mostly. It calls into `gap-surfacer` for the heavy lifting and presents the result in a form a busy attorney can scan in 20 seconds.

## Run order

1. 매터 활성 여부 확인. 활성이면 `matters/<slug>/gap-tracker.yaml`을, 비활성이면 프랙티스-레벨 `gap-tracker.yaml`을 사용. 트래커 파일이 없으면 "아직 등록된 갭이 없다"만 출력하고 종료.
2. Resolve the subcommand (default `summary`).
3. Pull only what the subcommand needs. Do not dump the whole tracker unless asked.
4. Format for at-a-glance reading.
5. For any action (`close`, `accept-risk`, `unpublish`, `snooze`), refuse here and redirect to `gap-surfacer`. 본 스킬은 read 위주.

## Subcommands

### summary (기본)

한 페이지 요약. 변호사가 월요일 아침에 보는 형태.

```markdown
# 갭 요약 — <YYYY-MM-DD>

매터: <slug | none>
트래커 마지막 업데이트: <ISO-datetime>

상태 분포: open <n> / 보류 <n> / 수정완료 <n> / 리스크수용 <n> / 비공개결정 <n>

## 지금 봐야 할 5건

1. [<gap-id>] <risk-band> · <죄목·규정 약식> · <slug-B>:line <N>
   "<라인 인용 — first 60 chars>"
   연령 <N일> · <패턴 N건과 묶임 | Tier 1 유래 | 일반>

2. <…>

(나머지는 `/launcelot-lawyer-pro:gaps urgent`·`aging`·`patterns`로 더 보기.)

## 패턴 갭 (있다면)

- <pattern-id>: <공통 출처> — 묶인 갭 <N>건 — 최초 발견 <date>

## 메타

- 가장 오래된 open: <gap-id>, <N일>
- aging breach (임계값 초과): <n건>
- 직전 7일에 close: <n건>
- 직전 7일에 새로 추가: <n건>
```

### urgent

다음 우선순위로 정렬해 출력:

1. Tier 1 변경에서 유래한 갭.
2. risk band 명백.
3. risk band 가능 + aging breach.

각 갭은 summary와 같은 한 블록 형식.

### aging

aging threshold 초과 갭만. 정렬: 가장 오래된 것부터.

각 줄에 다음을 명시:

- 어느 임계값을 넘었는지 (예: "30일 초과").
- 임계값별 액션(기본값 7 / 14 / 30일 — gap-surfacer Aging policy 참조).
- 어떤 알림이 이미 발송됐는지 (`gap-tracker.audit.log` 인용).

### patterns

패턴 갭 전용 뷰. 각 패턴에 대해:

```markdown
### 패턴 <pattern-id>

- 공통 slot A: <…>
- 묶인 갭 (<N>건):
  - <gap-id>: <slug-B>:line <N> — "<라인 인용>"
  - …
- 최초 발견: <date>
- 패턴 형성 시점: <date>
- 권장:
  - policy-redraft <pattern-id>로 공통 수정 방향 일괄.
  - patterns.yaml에 추적 패턴으로 등록할지 사용자에게 한 번 확인.
- 패턴이 close되려면: 묶인 갭 N건이 모두 close 상태가 되어야 함.
```

### by-source <name>

특정 출처(slot A source name)에서 유래한 갭만 필터.

### by-matter <slug>

특정 매터의 갭만 필터. `Active matter`가 그 매터로 설정되어 있지 않아도 본 명령은 동작한다(read 전용).

### \<gap-id\>

단일 갭 상세 뷰. summary 한 블록 포맷에 추가로:

- `discovered_in_diff_runs[]` 전체 이력.
- `defenses`의 풀 텍스트.
- `suggested_fix_direction` 전체.
- 관련 redraft 출력의 경로(`redrafts/<gap-id>-<…>.md`)와 chosen_candidate.
- 관련 launch-review revision 인용(`reviews/<slug-B>.md` 안의 Revision <N>).
- gap-tracker.audit.log 발췌(이 갭의 이력만).
- `slot_a_snippet` 객체 전체 (source, url, fetched_at, raw_quote).

## Read-only contract

본 스킬은 어떤 파일도 수정하지 않는다.

사용자가 `gaps close <gap-id>`처럼 액션 명령을 치면 다음과 같이 응답하고 라우팅:

> 액션은 `/launcelot-lawyer-pro:gap-surfacer close <gap-id>`로 실행하면 된다. 본 스킬은 조회 전용이라 직접 닫지 않는다. (또는 사용자가 명시적으로 본 스킬에서 라우팅을 원한다면 동의 후 한 번만 호출.)

## Persistence

본 스킬은 read 전용이므로 디스크에 큰 흔적을 남기지 않는다. 다만 조회 자체의 로그는 옵션으로:

- `<scope>/gaps-views.log` (사용자가 `--log-views` 옵션을 명시할 때만): append-only `<ISO-datetime>\t<subcommand>\t<filter>`.

기본은 로그 없음. 사용자가 일별 조회 패턴을 추적하고 싶을 때만 활성화.

## What this skill does NOT do

- 갭을 생성·수정·삭제하지 않는다.
- 알림을 송신하지 않는다.
- 다른 매터의 데이터를 cross-read 하지 않는다 — `by-matter <slug>` 명령으로 명시적으로 그 매터를 지정한 경우에만 그 매터 파일을 읽는다.
- 조문·판례 텍스트를 재인용하지 않는다. 트래커에 이미 저장된 `slot_a_snippet.raw_quote`를 표시할 뿐이다.
- 사용자 프로필·스타일·리스크 감수도에 따라 우선순위 표시를 바꾸지 않는다 — 본 플러그인은 그런 설정을 받지 않는다.
