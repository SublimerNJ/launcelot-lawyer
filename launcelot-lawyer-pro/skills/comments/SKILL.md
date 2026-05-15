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

본 스킬도 외부 출처(국회의안정보·법제처·변협·공정위 등)의 페이지를 인용할 때 `references/snippet-protocol.md` 절차로 _직접_ fetch한다. 외부 스킬에 위임하지 않는다.

## Run order

1. 매터 활성 여부 확인. 활성이면 `matters/<slug>/comments/`을, 비활성이면 프랙티스-레벨 `comments/`을 사용. 폴더가 없으면 자동 생성.
2. Load `comments/_index.yaml` (없으면 빈 인덱스로 시작).
3. Route by subcommand (default `list`).
4. For any submission decision, route the actual drafting away from this skill (this skill records decisions; it does not write opinion letters).

## Subcommands

### list

Print all comment items with status:

```markdown
| cmt-id  | 출처            | 종류            | 제목 | 마감       | 일수 | 상태 | 결정      |
| ------- | --------------- | --------------- | ---- | ---------- | ---- | ---- | --------- |
| CMT-001 | 국회의안정보    | 형법개정안      | …    | 2026-06-15 | D-31 | open | 미정      |
| CMT-002 | 법제처 입법예고 | 정통망법 시행령 | …    | 2026-05-30 | D-15 | open | 제출 예정 |
```

Default ordering: 마감일 임박 순.

### open

`open` 상태 + 마감 30일 이내만. 매일 빠르게 보는 뷰.

### add \<bill-id\>

새 의견서 항목 수동 등재. reg-feed-watcher가 자동으로 등재하지 못한 입법예고·공청을 수동으로 추가할 때.

필수 필드:

- `bill_id` 또는 식별자.
- 제목.
- 출처(예: 국회의안정보·법제처·변협·공정위 등).
- 종류(형법 개정안 / 정통망법 / 개보법 / 변호사법 / 광고규정 / 표시광고 가이드라인 / 저작권법 / 기타).
- 마감일.
- 영향 가능 영역.
- 원문 URL.

원문 URL은 `references/snippet-protocol.md` 절차로 본 스킬이 직접 fetch한다. 페이지에서 마감일과 본문 메타를 추출해 snippet 객체로 보존(`fetched_at`, `raw_quote`). fetch 실패 시 3단 fallback. 모두 실패하면 등재 거부하고 종료(아래 Failure handling).

### decide \<cmt-id\>

사용자 결정 로깅. 결정 enum:

- `제출 예정` — 의견서를 낼 것이다. 누가 작성·검토할지, 언제 초안을 만들지 자유 텍스트로 기록.
- `미제출 결정` — 내지 않기로 함. 사유 필수.
- `참고만` — 제출은 안 하지만 라이브러리 영향 분석은 한다. 자동으로 policy-diff 호출 권유.
- `보류` — 더 보고 결정. 사유 + 재결정 기일.

상태 전이는 `decision_log[]`에 append.

### log-submission \<cmt-id\>

`제출 예정`이었던 항목이 실제로 제출된 시점에 로깅.

필수:

- 제출일.
- 제출 채널(국회 의안 시스템, 법제처 의견 게시판, 변협 회신, 등).
- 제출 본문 보관 경로(사용자 측 파일 위치).
- 핵심 주장 요약 한 단락.
- (선택) 본문에 인용한 조문·판례 — 인용이 있으면 그 조문은 본 스킬이 snippet-protocol로 fetch해 cmt-id 항목에 attach. snippet 미확보면 사용자에게 "이 조문 인용은 검증되지 않았다. 제출 본문에서 인용을 빼거나 사용자가 출처 URL을 제공할지" 묻는다.

본 스킬은 의견서 본문을 작성·저장·송신하지 않는다. 위치만 기록.

### close \<cmt-id\>

의견서 트래커에서 닫기. 닫힘 조건 중 하나가 충족되어야 함:

- `로 제출됨` (`log-submission`이 선행되어야 함).
- `미제출로 종결` (이미 `미제출 결정`을 거쳤거나, 마감일이 지나 자동 종결되는 경우).
- `참고만으로 종결`.
- `법안 폐기·철회로 종결` — 입법안이 무산되어 의견 제출 기회 자체가 사라진 경우.
- `사용자가 무관 판단으로 제거` — 등재 후 다시 본 항목이 라이브러리에 영향 없다고 판단한 경우.

### by-source \<name\>

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

- 출처: <source name> (공식)
- 종류: <…>
- bill_id / 식별자: <…>
- 마감: <YYYY-MM-DD> (D-<N>)
- 영향 가능 영역: <…>
- 원문 snippet:

\`\`\`yaml
source: <…>
url: <…>
fetched_at: <…>
fetch_method: <…>
raw_quote: |
<원문 발췌 — 마감일·핵심 내용 등이 들어있는 부분>
\`\`\`

- 상태: <open | 보류 | closed>
- 결정: <미정 | 제출 예정 | 미제출 결정 | 참고만 | 법안 폐기·철회로 종결>
- 결정 사유 / 메모: <…>

### 결정 로그

- <ISO-datetime>: <상태 전이> · by <user> · 사유: <…>

### 제출 기록 (있다면)

- 제출일: <…>
- 제출 채널: <…>
- 본문 보관 경로: <user-side path>
- 핵심 주장 요약: <한 단락>
- 본문에 인용된 조문·판례 snippet: <…>
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
    snippet_fetched: <yes | no>
  ```

- `comments/_pending_ingest.yaml` — reg-feed-watcher가 던진 후보 큐.
- `comments/_audit.log` — append-only 액션 로그.

## Deadline behavior

- 마감 30일 이내: list 상단 강조.
- 마감 7일 이내: 사용자가 외부 알림 채널을 명시적으로 설정해 둔 경우(예: "이 갭은 Slack으로 알려줘"), 다음 본 스킬 호출 시 알림 송신을 _제안_. 자동 송신하지 않는다.
- 마감 경과: 사용자가 `log-submission` 또는 `close`로 명시적으로 처리할 때까지 `open` 상태 유지. 자동 종결 없음.

## Submission writing — 본 스킬이 하지 않는 것

본 스킬은 의견서 본문을 작성하지 않는다. 사용자가 본문 작성을 원할 때 권장 흐름:

1. `policy-diff <bill-id> library` — 입법안이 라이브러리에 어떻게 영향을 주는지 사실 분석.
2. 사용자가 본문 초안을 자체 작성하거나 koreanizer / 변호사 본인의 보이스 스킬(`kmj`/`kmg`/`sbh`/`ldg`/`wja`)로 톤 마감.
3. `log-submission`으로 결과 기록.

## Failure handling

- `add` 호출 시 URL fetch 3단 fallback 모두 실패: 등재 거부하고 사용자에게 메시지 출력.

  ```
  ⛔ Snippet 확보 실패 — comments 항목 등재 불가.
  대상: <bill_id>
  시도한 출처: <Step 1 URL>, <Step 2 URL>, <Step 3 URL>
  실패 사유: <…>
  조치: 사용자가 다른 URL을 제공하거나 페이지 텍스트를 paste하면 재시도.
  ```

- reg-feed-watcher의 ingest 후보 큐가 손상: 큐 파일을 백업해두고 빈 큐로 재시작. 사용자에게 보고.
- 같은 bill_id가 두 번 등재 시도: 기존 cmt-id를 보여주고 새로 만드는 대신 기존 항목 갱신을 권유.

## What this skill does NOT do

- 의견서 본문을 작성하거나 제출하지 않는다.
- 외부 시스템(국회 의안 시스템, 법제처 의견 게시판 등)에 직접 송신하지 않는다.
- 마감 알림을 사용자 동의 없이 외부 채널로 발송하지 않는다.
- 다른 매터의 comments를 cross-read 하지 않는다.
- 모델 지식으로 입법안 본문이나 조문 인용을 단정하지 않는다. 모든 인용은 snippet-protocol로 본 스킬이 fetch한 raw_quote에서만 만들어진다. 외부 스킬(`launcelot-lawyer` 포함)에 검증을 위임하지 않는다.
