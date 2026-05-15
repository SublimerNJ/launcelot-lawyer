---
name: matter-workspace
description: >
  다수 채널·다수 의뢰인 사건 콘텐츠를 한 변호사가 다룰 때의 매터 컨텍스트 분리.
  new/list/switch/close/detach 서브커맨드로 한 매터의 검토 메모·갭·리뷰가 다른
  매터에 누설되지 않도록 격리한다. 사용자가 "새 매터", "이번 의뢰인 건 분리해줘",
  "지금 매터 뭐야", "매터 닫아", "프랙티스 레벨로", "matter switch"라고 말할 때
  실행한다.
argument-hint: "[new <slug> | list | switch <slug> | close <slug> | detach | show]"
---

# /launcelot-lawyer-pro:matter-workspace

This skill manages matter-scoped context. A "matter" is a discrete engagement: one client's case, one channel's playlist, one campaign. It exists so that two clients' contexts (their scripts, their gap trackers, their launch reviews) never bleed into each other.

By default, `matters` are off. A solo channel without multi-client load can ignore this skill entirely. Turning it on is a deliberate act and every other skill reads `matters/_index.yaml`의 `active`를 보고 매터-스코프 경로에 쓸지 결정한다.

## Run order

1. `~/.claude/plugins/config/launcelot-lawyer-pro/matters/_index.yaml`을 읽는다. 없으면 빈 인덱스(`enabled: false`, `active: null`, `matters: []`)로 초기화.
2. 첫 사용이면(인덱스가 없었으면) 사용자에게 매터 활성화 여부를 묻는다. "no"면 `enabled: false`로 저장하고 종료. "yes"면 `enabled: true`. 매터 활성화 자체에는 변호사 프로필·스타일 같은 추가 설정을 묻지 않는다.
3. Otherwise, route by subcommand.

## Subcommands

### new \<slug\>

Create a new matter.

- Reject the slug if it conflicts with an existing matter or with reserved words (`practice-level`, `none`, `default`, `archive`, `_index`, `_archive`).
- Korean-safe slugification: lowercase ASCII for Roman tokens, Hangul preserved, spaces → hyphen.
- Create `matters/<slug>/matter.md` from the template (below).
- 사실 메모만 받는다(아래 matter.md 템플릿 참고). Speech policy·Hard rule 같은 *콘텐츠 평가 규칙*은 매터 단위로도 받지 않는다 — 본 플러그인은 그런 설정을 받지 않는다.
- Update `matters/_index.yaml`: append the new matter, set `active: <slug>`, set `enabled: true`.
- Initialize matter-scoped data directories under `matters/<slug>/`:
  - `reviews/`, `digests/`, `diffs/`, `redrafts/`, `comments/`, `feature-risk/`, `review-script/`
  - `gap-tracker.yaml` (matter-scoped, separate from practice-level tracker)
  - `gap-tracker.audit.log`
- Echo: "매터 `<slug>` 생성됨. 활성 매터로 전환됨."

### list

List all matters with status.

```markdown
| 매터   | 상태     | 시작일 | 마지막 활동 | 미해소 갭 | 노트 |
| ------ | -------- | ------ | ----------- | --------- | ---- |
| <slug> | active   | <date> | <date>      | <n>       | <…>  |
| <slug> | archived | <date> | <date>      | <n>       | <…>  |
```

Note the currently active matter in the footer.

### switch \<slug\>

Switch the active matter.

- Refuse if slug not found.
- Write `active: <slug>` to `matters/_index.yaml`.
- Echo: 이전 매터 → 새 매터. 새 매터의 미해소 갭·미결정 launch-review 항목 요약 한 줄.

`switch practice-level` is a synonym for `detach`.

### close \<slug\>

Archive a matter.

- Ask for reason (사건 종결 / 캠페인 종료 / 의뢰인 이탈 / 기타) and outcome 한 줄.
- Confirm: 미해소 갭 N건이 있다면 어떻게 처리할지(매터 닫기 전 모두 close / accept-risk / 미해소 상태 그대로 archive). 자동으로 닫지 않는다.
- Move `matters/<slug>/` to `matters/_archive/<slug>/`. (filesystem-level move, no data loss.)
- If the closed matter was active, set `active: null` in `matters/_index.yaml`.

### detach

Switch to practice-level context (no active matter). Useful when working on general channel content not tied to any client.

- Write `active: null` to `matters/_index.yaml`.
- Do NOT set `enabled: false`. The matters facility remains on; only the active scope changes.

### show

Print the current matter's `matter.md` plus a one-line summary (미해소 갭 수, 마지막 launch-review revision 일자).

## matter.md template

`matters/<slug>/matter.md`:

```markdown
# Matter — <slug>

Created: <ISO date>
Status: active | archived
Last activity: <ISO date>

## Matter identification

- 사건번호: <…>
- 의뢰인 약식: <…> (실명 보호 컨벤션에 따른 표기)
- 법원·연도: <…>
- 기타 식별: <…>

## Matter scope

- 적용 채널: <…>
- 캠페인 식별자: <…>
- 기간: <…>

## Notes

<자유 텍스트. 사실 메모만. 예: "본 매터의 의뢰인은 익명화 필요", "법원 비공개 결정으로 사건명 노출 금지", "공판 기일 전까지 보도자료 인용 금지" 같은 사실관계 메모. 콘텐츠 평가 규칙은 여기 적지 않는다 — 본 플러그인은 사용자별 평가 규칙을 받지 않는다.>

## Closing summary (close 후 자동 채워짐)

- Closed: <ISO date>
- Reason: <…>
- Outcome: <…>
- 마지막 미해소 갭: <…>
```

## matters/\_index.yaml 형식

```yaml
enabled: <true | false>
active: <slug | null>
matters:
  - slug: <…>
    status: <active | archived>
    created: <ISO date>
    last_activity: <ISO date>
    open_gaps: <n>
    archive_path: <… | null> # archived인 경우만
```

## How other skills consume `active matter`

Every skill that writes data files reads `matters/_index.yaml` first:

| enabled | active   | 동작                                                                                            |
| ------- | -------- | ----------------------------------------------------------------------------------------------- |
| false   | null     | 프랙티스-레벨 경로에 쓴다. 매터 디렉토리 무시.                                                  |
| true    | null     | 프랙티스-레벨 경로에 쓰되, 어디에 쓰는지 출력 헤더에 명시.                                      |
| true    | \<slug\> | `matters/<slug>/` 하위 동등 경로에 쓴다. 프랙티스-레벨 데이터와 매터 데이터를 절대 섞지 않는다. |

`matter.md`는 매터-스코프 스킬 실행 시 *사실 컨텍스트*로만 읽힌다. 사용자별 평가 규칙으로 작용하지 않는다.

매터 간 cross-read는 본 스킬이 명시적으로 허용하기 전까지 막혀 있다. 어느 스킬도 `matters/A/`의 파일을 보면서 `matters/B/`의 결정에 영향을 주지 않는다. 패턴 갭 검출 같은 분석에서 cross-read가 필요하면 그 스킬이 사용자에게 명시적 동의를 받은 뒤에만 진행한다.

## Active-matter contract for skill outputs

매터가 활성화되어 있을 때 모든 출력의 첫 줄에 다음을 표시:

```
[Matter: <slug>]  ·  [Source: matter-scoped]
```

프랙티스-레벨로 떨어진 출력은:

```
[Matter: (none — practice-level)]  ·  [Source: practice-level]
```

이 표시는 사용자가 어느 컨텍스트의 결과를 읽고 있는지 헷갈리지 않게 한다.

## Safety

- 절대 다른 매터의 파일을 읽지 않는다(명시적 cross-read 동의가 있을 때 제외).
- `close <slug>` 후의 archive는 데이터를 삭제하지 않는다. archive 하위에서 그대로 보존.
- `enabled: false`인 상태에서 사용자가 매터 관련 명령(`switch`, `close` 등)을 치면 활성화 여부를 먼저 묻고 진행.

## What this skill does NOT do

- 매터 안의 갭·리뷰 데이터를 직접 편집하지 않는다. 그 작업은 각 영역의 스킬(gap-surfacer / launch-review 등)에서.
- 매터를 자동 생성·자동 close 하지 않는다.
- 매터 식별자(사건번호 등)의 정확성을 검증하지 않는다. 입력 그대로 저장한다.
- 매터별 Speech policy·Hard rule·Risk calibration·콘텐츠 평가 규칙을 받지 않는다. 본 플러그인은 그런 설정을 받지 않는다(매터 안에서도 마찬가지). 매터에는 *사실 메모*만 둔다.
