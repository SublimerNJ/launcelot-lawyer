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

This skill manages matter-scoped context. A "matter" is a discrete engagement: one client's case, one channel's playlist, one campaign. It exists so that two clients' contexts (their scripts, their hard rules, their gap trackers) never bleed into each other.

By default, `matters` are off. A solo channel without multi-client load can ignore this skill entirely. Turning it on is a deliberate act and every other skill reads `Active matter` to know whether to scope reads/writes.

## Run order

1. Load `~/.claude/plugins/config/launcelot-lawyer-pro/CLAUDE.md`. Check the existence of the `## Matter workspaces` block.
2. If the block is missing, treat as first use: write an initial block:
   ```markdown
   ## Matter workspaces

   Enabled: ✗
   Active matter: (none — practice-level)
   Cross-matter context: off
   Matters root: ~/.claude/plugins/config/launcelot-lawyer-pro/matters/
   ```
   Ask the user whether to enable matters. If "no", leave Enabled `✗` and stop.
3. Otherwise, route by subcommand.

## Subcommands

### new <slug>

Create a new matter.

- Reject the slug if it conflicts with an existing matter or with reserved words (`practice-level`, `none`, `default`, `archive`).
- Korean-safe slugification: lowercase ASCII for Roman tokens, Hangul preserved, spaces → hyphen.
- Create `matters/<slug>/matter.md` from the template (below).
- Interview the user for matter-specific overrides:
  - 매터 식별자: 사건번호 / 의뢰인 약식 (실명 보호 컨벤션) / 법원·연도 / 기타.
  - 매터의 채널·캠페인 식별자.
  - 매터별 Speech policy 오버라이드 (예: 본 사건에서는 실명 절대 금지).
  - 매터별 Hard rule 추가 (사건 특화 금지선).
  - 매터별 Script library 한정 (이 매터의 대본만 별도 폴더에 있는 경우).
- Update `CLAUDE.md` → `## Matter workspaces` → `Enabled: ✓`, `Active matter: <slug>`.
- Initialize matter-scoped data directories under `matters/<slug>/`:
  - `reviews/`, `digests/`, `diffs/`, `redrafts/`, `comments/`
  - `gap-tracker.yaml` (matter-scoped, separate from practice-level tracker)
  - `gap-tracker.audit.log`
- Echo: "매터 `<slug>` 생성됨. 활성 매터로 전환됨."

### list

List all matters with status.

```markdown
| 매터 | 상태 | 시작일 | 마지막 활동 | 미해소 갭 | 노트 |
|---|---|---|---|---|---|
| <slug> | active | <date> | <date> | <n> | <…> |
| <slug> | archived | <date> | <date> | <n> | <…> |
```

Note the currently active matter in the footer.

### switch <slug>

Switch the active matter.

- Refuse if slug not found.
- Write `Active matter: <slug>` to `CLAUDE.md` → `## Matter workspaces`.
- Echo: 이전 매터 → 새 매터. 새 매터의 미해소 갭·미결정 launch-review 항목 요약 한 줄.

`switch practice-level` is a synonym for `detach`.

### close <slug>

Archive a matter.

- Ask for reason (사건 종결 / 캠페인 종료 / 의뢰인 이탈 / 기타) and outcome 한 줄.
- Confirm: 미해소 갭 N건이 있다면 어떻게 처리할지(매터 닫기 전 모두 close / accept-risk / 미해소 상태 그대로 archive). 자동으로 닫지 않는다.
- Move `matters/<slug>/` to `matters/_archive/<slug>/`. (filesystem-level move, no data loss.)
- If the closed matter was active, set `Active matter: (none — practice-level)`.

### detach

Switch to practice-level context (no active matter). Useful when working on general channel content not tied to any client.

- Write `Active matter: (none — practice-level)`.
- Do NOT set `Enabled: ✗`. The matters facility remains on; only the active scope changes.

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
- 의뢰인 약식: <…>  (실명 보호 컨벤션에 따른 표기)
- 법원·연도: <…>
- 기타: <…>

## Matter scope

- 적용 채널: <…>
- 캠페인 식별자: <…>
- 기간: <…>

## Speech policy overrides (이 매터에 한정)

(없으면 "오버라이드 없음 — 프랙티스-레벨 Speech policy 그대로 적용".)

### Real-name policy override
<…>

### Pending-case policy override
<…>

### Speculative-statement policy override
<…>

### Absolute-claim policy override
<…>

## Hard rule additions (이 매터에 한정)

- Never (matter-scoped):
  - <line>
- Must escalate (matter-scoped):
  - <line>

## Matter-scoped script library

- Locations: <…>
- 메모: <…>

## Notes

<자유 텍스트. 의뢰인과 채널 사이의 비밀유지 의무 컨텍스트, 외부 노출 금지 사실 등.>

## Closing summary (close 후 자동 채워짐)

- Closed: <ISO date>
- Reason: <…>
- Outcome: <…>
- 마지막 미해소 갭: <…>
```

## How other skills consume `Active matter`

Every skill that writes data files reads `## Matter workspaces` first:

| Enabled | Active matter | 동작 |
|---|---|---|
| ✗ | (none) | 프랙티스-레벨 경로에 쓴다. 매터 디렉토리 무시. |
| ✓ | (none — practice-level) | 프랙티스-레벨 경로에 쓰되, 어디에 쓰는지 출력 헤더에 명시. |
| ✓ | <slug> | `matters/<slug>/` 하위 동등 경로에 쓴다. 프랙티스-레벨 데이터와 매터 데이터를 절대 섞지 않는다. |

`Cross-matter context` 옵션:
- off (기본): 다른 매터의 파일을 절대 읽지 않는다. 같은 변호사라도 매터 A의 갭이 매터 B의 검토에 영향을 주지 않는다.
- on: 패턴 갭 검출, 가용 인용 자료 검색 등 일부 분석에서 다른 매터의 파일을 읽을 수 있다. on으로 바꿀 때 사용자에게 매번 명시적 동의를 받는다.

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

- 절대 다른 매터의 파일을 읽지 않는다(`Cross-matter context: on`이고 사용자가 명시적으로 허락한 경우 제외).
- `close <slug>` 후의 archive는 데이터를 삭제하지 않는다. archive 하위에서 그대로 보존.
- `Enabled: ✗`인 상태에서 사용자가 매터 관련 명령을 치면 활성화 여부를 먼저 묻고 진행.
- 활성 매터를 가진 채로 cold-start-interview `--redo`를 실행하려고 하면 한 번 경고하고 `Matter workspaces` 섹션은 보존한다.

## What this skill does NOT do

- 매터 안의 갭·리뷰 데이터를 직접 편집하지 않는다. 그 작업은 각 영역의 스킬(gap-surfacer / launch-review / customize 등)에서.
- 매터를 자동 생성·자동 close 하지 않는다.
- 매터 식별자(사건번호 등)의 정확성을 검증하지 않는다. 입력 그대로 저장한다.
