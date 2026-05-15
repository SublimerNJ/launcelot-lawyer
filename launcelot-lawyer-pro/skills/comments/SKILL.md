---
name: comments
description: >
  국회의안정보시스템·법제처 입법예고·대한변호사협회 회칙 개정 공청·공정거래위원회
  표시광고 가이드라인 의견수렴 등 한국 입법·규정 개정 과정에서 의견서 제출 기한을
  트래킹하고 제출 여부 결정을 로깅한다. reg-feed-watcher가 입법예고·공청 항목을
  감지하면 자동 등재 후보가 되고, 사용자가 등재 여부와 의견서 제출 의사를 결정한다.
  사용자가 "open한 의견서", "마감 임박", "이 입법예고에 의견 낼래", "의견서 트래커",
  "comments status", "공청 기한"이라고 말할 때 실행한다.
argument-hint: "[list | open | add <bill-id> | decide <cmt-id> | log-submission <cmt-id> | close <cmt-id> | by-source <name>]"
---

# /launcelot-lawyer-pro:comments

This skill tracks public-comment opportunities relevant to Korean lawyer YouTube content — primarily statutory amendments touching defamation, ICNIA, personal data, the lawyer act, advertising regulation, and fair-labeling rules — and records what the user decided to do with each.

## Run order

1. Load `~/.claude/plugins/config/launcelot-lawyer-pro/CLAUDE.md`. Read `## Cadence` and `## Integrations`.
2. Load the comments index at `comments/_index.yaml` (matter-scoped if `Active matter` is set, else practice-level).
3. Route by subcommand (default `list`).
4. For any submission decision, route the actual drafting away from this skill (this skill records decisions; it does not write opinion letters). Refer the user to their preferred drafting flow (often koreanizer + per-attorney voice skill for tone, plus user's own substantive content).

## Subcommands

### list

Print all comment items with status:

```markdown
| cmt-id | 출처 | 종류 | 제목 | 마감 | 일수 | 상태 | 결정 |
|---|---|---|---|---|---|---|---|
| CMT-001 | 국회의안정보 | 형법개정안 | … | 2026-06-15 | D-31 | open | 미정 |
| CMT-002 | 법제처 입법예고 | 정통망법 시행령 | … | 2026-05-30 | D-15 | open | 제출 예정 |
```

Default ordering: 마감일 임박 순.

### open

`open` 상태 + 마감 30일 이내만. 매일 빠르게 보는 뷰.

### add <bill-id>

새 의견서 항목 수동 등재. reg-feed-watcher가 자동으로 등재하지 못한 입법예고·공청을 수동으로 추가할 때.

필수 필드:
- `bill_id` 또는 식별자.
- 제목.
- 출처(watchlist source name).
- 종류(형법 개정안 / 정통망법 / 개보법 / 변호사법 / 광고규정 / 표시광고 가이드라인 / 저작권법 / 기타).
- 마감일.
- 영향 가능 영역(사용자가 정의한 Tier 1·2 카테고리 매핑).
- 원문 URL.

검증: launcelot-lawyer로 URL 실존 확인. 비가용이면 `[미검증 — 수동 등재]` 태그.

### decide <cmt-id>

사용자 결정 로깅. 결정 enum:

- `제출 예정` — 의견서를 낼 것이다. 누가 작성·검토할지, 언제 초안을 만들지 자유 텍스트로 기록.
- `미제출 결정` — 내지 않기로 함. 사유 필수.
- `참고만` — 제출은 안 하지만 라이브러리 영향 분석은 한다. 자동으로 policy-diff 호출 권유.
- `보류` — 더 보고 결정. 사유 + 재결정 기일.

상태 전이는 `decision_log[]`에 append.

### log-submission <cmt-id>

`제출 예정`이었던 항목이 실제로 제출된 시점에 로깅.

필수:
- 제출일.
- 제출 채널(국회 의안 시스템, 법제처 의견 게시판, 변협 회신, 등).
- 제출 본문 보관 경로(사용자 측 파일 위치).
- 핵심 주장 요약 한 단락.
- (선택) 검증 위임 결과: 의견서 본문에 인용한 조문·판례를 launcelot-lawyer로 검증한 결과.

본 스킬은 의견서 본문을 작성·저장·송신하지 않는다. 위치만 기록.

### close <cmt-id>

의견서 트래커에서 닫기. 닫힘 조건 중 하나가 충족되어야 함:

- `로 제출됨` (`log-submission`이 선행되어야 함).
- `미제출로 종결` (이미 `미제출 결정`을 거쳤거나, 마감일이 지나 자동 종결되는 경우).
- `참고만으로 종결`.
- `법안 폐기·철회로 종결` — 입법안이 무산되어 의견 제출 기회 자체가 사라진 경우.
- `사용자가 무관 판단으로 제거` — 등재 후 다시 본 항목이 라이브러리에 영향 없다고 판단한 경우.

### by-source <name>

특정 출처에서 유래한 의견서 항목만 필터.

## Auto-ingestion contract (reg-feed-watcher 핸드오프)

reg-feed-watcher가 Tier 1·2 디제스트에서 다음 조건을 만족하는 항목을 발견하면:

- `type` ∈ {조문개정안 발의, 입법예고, 시행령·시행규칙 개정안, 광고규정 개정안, 회칙 개정안 공청, 공정위 가이드라인 의견수렴}.
- 마감일(comment deadline)이 명시되어 있음.

해당 항목은 `comments/_pending_ingest.yaml`에 후보로 적재되고, 다음 본 스킬 호출 시(`list`·`open` 등) 사용자에게:

> reg-feed-watcher가 의견서 후보 N건을 잡았다. 각각 등재할지 묻겠다.

후보별로 사용자가 `등재 / 제외 / 보류` 결정. 자동 등재하지 않는다.

## Output template — 단일 cmt 항목 상세

```markdown
### [CMT-<NNN>] <제목>

- 출처: <watchlist source name> (<공식 | 준공식 | 민간>)
- 종류: <…>
- bill_id / 식별자: <…>
- 마감: <YYYY-MM-DD>  (D-<N>)
- 영향 가능 영역: <…>
- 검증 상태: <verified-current | unverified | not-found>
- 원문: <URL>
- 상태: <open | 보류 | closed>
- 결정: <미정 | 제출 예정 | 미제출 결정 | 참고만 | 법안 폐기·철회로 종결>
- 결정 사유 / 메모: <…>

### 결정 로그

- <ISO-datetime>: <상태 전이>  ·  by <user>  ·  사유: <…>

### 제출 기록 (있다면)

- 제출일: <…>
- 제출 채널: <…>
- 본문 보관 경로: <user-side path>
- 핵심 주장 요약: <한 단락>
- launcelot-lawyer 검증 결과: <…>
```

## Persistence

루트는 `<scope>/comments/` (활성 매터에 따라 `matters/<slug>/comments/` 또는 `comments/`).

- `comments/<cmt-id>.md` — 항목별 상세.
- `comments/_index.yaml` — 인덱스:
  ```yaml
  - id: CMT-<NNN>
    bill_id: <…>
    source: <…>
    type: <…>
    deadline: <ISO date>
    status: <open | 보류 | closed>
    decision: <…>
    decided_at: <…>
    submitted_at: <… | null>
    led_to_diff_run: <run-id | null>
    matter: <slug | practice-level>
  ```
- `comments/_pending_ingest.yaml` — reg-feed-watcher가 던진 후보 큐.
- `comments/_audit.log` — append-only 액션 로그.

## Deadline behavior

- 마감 30일 이내: list 상단 강조.
- 마감 7일 이내: 외부 알림 채널이 활성화돼 있고 `## Gap tracker policy → Notification recipient`이 설정돼 있으면, 다음 본 스킬 호출 시 알림 송신을 *제안*. 자동 송신하지 않는다.
- 마감 경과: 사용자가 `log-submission` 또는 `close`로 명시적으로 처리할 때까지 `open` 상태 유지. 자동 종결 없음.

## Submission writing — 본 스킬이 하지 않는 것

본 스킬은 의견서 본문을 작성하지 않는다. 사용자가 본문 작성을 원할 때 권장 흐름:

1. `policy-diff <bill-id> library` — 입법안이 라이브러리에 어떻게 영향을 주는지 사실 분석.
2. 사용자가 본문 초안을 자체 작성하거나 koreanizer / 변호사 본인의 보이스 스킬(`kmj`/`kmg`/`sbh`/`ldg`/`wja`)로 톤 마감.
3. `log-submission`으로 결과 기록.

## Failure handling

- launcelot-lawyer 비가용: `add` 시 URL 검증 실패면 `[미검증 — 수동 등재]` 태그 부착하고 진행 옵션 제공. 마감일 추출은 사용자가 직접 입력.
- reg-feed-watcher의 ingest 후보 큐가 손상: 큐 파일을 백업해두고 빈 큐로 재시작. 사용자에게 보고.
- 같은 bill_id가 두 번 등재 시도: 기존 cmt-id를 보여주고 새로 만드는 대신 기존 항목 갱신을 권유.

## What this skill does NOT do

- 의견서 본문을 작성하거나 제출하지 않는다.
- 외부 시스템(국회 의안 시스템, 법제처 의견 게시판 등)에 직접 송신하지 않는다.
- 마감 알림을 사용자 동의 없이 외부 채널로 발송하지 않는다.
- 다른 매터의 comments를 cross-read 하지 않는다.
