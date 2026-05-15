---
name: customize
description: >
  Launcelot-Product-Legal 채널 프로필을 cold-start-interview를 다시 돌리지 않고
  한 항목씩 수정한다. 변호사 신분, 채널 톤, 실명·진행사건·단정 표현 정책, 위험
  감수도, hard rule, 인용 소스 목록, 인테그레이션 상태를 개별 갱신할 수 있다.
  사용자가 "프로필 바꿔", "단정 표현 정책 수정", "리스크 감수도 낮춰", "hard rule
  추가", "customize"라고 말할 때 실행한다.
argument-hint: "[--field <섹션명>] [--show]"
---

# /launcelot-product-legal:customize

This skill edits the practice profile in `~/.claude/plugins/config/launcelot-product-legal/CLAUDE.md` one field at a time. It never re-runs the full cold-start.

## Run order

1. Confirm the config file exists. If not, redirect to `/launcelot-product-legal:cold-start-interview`.
2. If `--show` was passed, print the current value of the requested field (or the whole profile if no field given) and stop.
3. If `--field <name>` was passed, jump straight to that section.
4. Otherwise, list the editable sections and ask which to change. Editable sections (mirror `cold-start-interview` template):
   - Attorney profile
   - Channel profile
   - Speech policy → Real-name policy
   - Speech policy → Pending-case policy
   - Speech policy → Speculative-statement policy
   - Speech policy → Absolute-claim policy
   - Risk calibration → Appetite
   - Risk calibration → Past disputes (append / edit / remove an entry)
   - Hard rules → Hard "never" (add / remove)
   - Hard rules → Hard "must escalate" (add / remove)
   - Substantiation
   - Integrations (re-probe only — equivalent to `cold-start-interview --check-integrations` for the integration section)
5. Read back the current value of the chosen section.
6. Ask the new value, accepting either a label (for the labeled sections) or free text.
7. Back up the file to `CLAUDE.md.bak.<ISO-date>` before writing.
8. Write the edit in place. Update `Last updated:` at the top of the file.
9. Echo a short summary: section, old value (first 80 chars), new value (first 80 chars).

## Edit conventions

- Never edit more than one section per invocation. If the user requests several edits, finish one, confirm, then loop back to step 4.
- For labeled fields (the four speech policies, Appetite), accept only values from the cold-start label set. If the user types something else, ask whether to record it as `Label: 기타` with the raw text preserved.
- For list fields (Past disputes, Hard "never", Hard "must escalate", Acceptable sources), support three operations: `add <item>`, `remove <#>`, `replace <#> <new value>`. Show numbered list before asking.
- For Integrations, do not let the user manually set a probe state to `available`. Re-run the probe and trust the probe result. Only the `fallback note` field is hand-editable.

## Safety

- Refuse to write a value that is empty when the section requires content. Ask again.
- Refuse to delete the last entry of `## Speech policy` subsections (those four labels are required for the validators to run). Suggest replacing the value instead.
- Never touch any file outside `~/.claude/plugins/config/launcelot-product-legal/`.

## What this skill does NOT do

- Does not validate any script.
- Does not change the structure of the config file (no adding new top-level sections).
- Does not migrate the file across plugin versions. That is the upgrade flow's job.
