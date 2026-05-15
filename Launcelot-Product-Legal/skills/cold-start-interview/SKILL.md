---
name: cold-start-interview
description: >
  한국 변호사 유튜브 채널의 프로필을 학습한다. 변호사 신분·전문분야, 채널 톤,
  실명·진행사건·제3자 언급 정책, 단정 표현 허용 범위, 과거 분쟁 이력, 시청자층을
  인터뷰로 수집해 검증 임계값을 자동 조정한다. 첫 설치 시, "처음부터 다시 설정"
  요청 시, `~/.claude/plugins/config/launcelot-product-legal/CLAUDE.md`가 없거나
  플레이스홀더가 남아있을 때 실행한다.
argument-hint: "[--redo | --check-integrations]"
---

# /launcelot-product-legal:cold-start-interview

This skill performs a one-time cold-start interview for the Launcelot-Product-Legal plugin. It produces a practice profile saved to `~/.claude/plugins/config/launcelot-product-legal/CLAUDE.md` that every other skill in this plugin will read.

The output is the calibration that lets `marketing-claims-review` and `launch-review` know how strict to be on a given script.

## When to run

- First install: the config file is missing or contains `[PLACEHOLDER]` markers.
- User says "set up the plugin", "onboard me", "처음부터 다시", "cold start".
- With `--redo`: re-run the full interview and overwrite the existing profile.
- With `--check-integrations`: only re-probe MCP availability (`launcelot-lawyer`, `wiki`, `kmj`, `kmg`, `sbh`, `ldg`, `wja`, `koreanizer`, `humanizer`) and update the integration section without re-asking interview questions.

## Output target

`~/.claude/plugins/config/launcelot-product-legal/CLAUDE.md`

If the directory does not exist, create it. If a previous file exists and `--redo` was not passed, ask the user before overwriting; offer to back up to `CLAUDE.md.bak.<ISO-date>`.

## Interview structure

The interview has six modules. Run them in order. After each module, summarize what was captured and confirm before moving on. Never skip a module silently — if a question genuinely does not apply (for example, a solo channel with one host), record `N/A` with the reason rather than omitting the field.

### Module 1 — Attorney identity

Goal: know who is on camera so the validator can apply the right bar advertising rules and avoid wrong-jurisdiction assumptions.

Questions:

1. Bar admission jurisdiction. Confirm Korean bar (대한변호사협회). If multiple, list all.
2. Local bar association membership (서울지방변호사회 etc.).
3. Law firm or sole practice. Firm name if applicable.
4. Stated practice areas on the channel (criminal defense, family, corporate, IP, employment, etc.).
5. Other titles to disclaim or feature (former prosecutor, former judge, certified specialist). Korean bar rules treat these carefully — capture exactly how the channel currently presents them.
6. Year of admission and continuing professional development status if the channel mentions experience claims.

Write to: `## Attorney profile` in the config file.

### Module 2 — Channel context

Goal: understand what kind of content is being made so the reviewer applies the right baseline.

Questions:

1. Channel name(s) and platform(s). YouTube long-form, Shorts, blog, SNS — list all the surfaces this plugin will validate.
2. Format mix. Estimate percentage: case-explainer / news-reaction / Q&A / interview / shorts / other.
3. Tone label. Pick one or describe: 차갑게 기술적 / 분석형 / 어그로형 / 사실보도형 / 위로·공감형 / 혼합.
4. Target viewer. Lay public / potential clients in a specific situation (예: 형사 피의자 가족) / professional peers.
5. Typical script length (minutes) and posting cadence (per week).

Write to: `## Channel profile`.

### Module 3 — Speech policy (the calibration core)

Goal: capture the four policies that drive most criminal exposure decisions. These map directly to thresholds the validator will apply.

Questions (one at a time):

1. **Real-name policy.** When may the channel use a real person's name?
   - Public figures only
   - Public figures plus parties to final, public judgments
   - Anyone whose name has appeared in mainstream press
   - Names allowed broadly if facts are accurate
   - Never use real names; anonymize all third parties
2. **Pending-case policy.** Coverage of cases that are still in progress?
   - Never (only final, fully published judgments)
   - Allowed with anonymization
   - Allowed if the case is already in the press
   - Allowed broadly
3. **Speculative-statement policy.** How far may the channel go on motive, intent, or characterization of someone's conduct when the criminal disposition is not yet final?
   - Strict (only what a published judgment found)
   - Mid (allowed if framed as analysis with qualifiers)
   - Loose (direct characterization permitted)
4. **Absolute-claim policy on outcomes.** Phrases like "100% 승소", "무조건 무죄", "이건 무조건 처벌받습니다".
   - Never (treat all as 변호사 광고규정 / 표시광고법 violations)
   - Allowed when academically generic (no specific case)
   - Allowed broadly

Write to: `## Speech policy`. Store both the policy label and the raw text the user gave, because the raw text is what the validator quotes back when it flags a line.

### Module 4 — Risk calibration

Goal: get the user's deliberate posture so the validator does not waste their time with calls they would always ignore, and so it never silently downgrades a call they would always escalate.

Questions:

1. Risk appetite, three-point scale: 보수적 / 중도 / 공격적. Translate: 보수적 means the channel prefers safer phrasing even when arguably defensible; 공격적 means the channel accepts close-to-the-line phrasing when the position is defensible.
2. Past dispute history (any cease-and-desist letters, takedown notices, bar complaints, defamation actions, civil claims). For each, record short summary + outcome. Past incidents recalibrate where the validator should be strictest.
3. Hard "never" lines — content the channel will not produce regardless of facts (예: 기획고소 가이드, 답변서 자작 가이드 — sample exclusions taken from existing `kmg` skill notes).
4. Hard "must escalate" categories — anything the validator must always flag to a human even when defaults would let it pass.

Write to: `## Risk calibration` and `## Hard rules`.

### Module 5 — Substantiation sources

Goal: know what the channel can legitimately cite when a claim needs support, so the validator can ask "where is the source" with a known set of acceptable answers.

Questions:

1. Acceptable citation sources for legal claims (대법원 종합법률정보 / 국가법령정보센터 / 헌재 / 학술논문 / 1차 보도 only / 변협 자료).
2. Whether the channel cites specific case numbers and how (full citation / 약식).
3. Whether the channel quotes statutory text directly and how (full quote with article number / paraphrase with article number / paraphrase only).
4. Internal substantiation files location (Google Drive folder, Notion DB, NJsidian wiki, etc.).

Write to: `## Substantiation`.

### Module 6 — Integrations probe

Goal: confirm which sister skills and MCPs are actually responding right now. The validator changes behavior based on what is connected.

Probe each of these and record `available` / `not available`:

- `launcelot-lawyer` skill — primary verifier (조문·판례 실존 확인, 3단 논법 포섭).
- `wiki` / `njsidian-wiki` MCP — prior script library.
- `kmj`, `kmg`, `sbh`, `ldg`, `wja` skills — per-attorney script generators (drives speech-policy defaults).
- `koreanizer`, `humanizer` skills — post-validation rewriting.
- `cocounsel-legal:deep-research` — US-only, record as `not applicable` rather than missing.

Write to: `## Integrations`. For anything `not available`, note what the validator will fall back to.

## Config file template

When all six modules are done, write the file using this template. Use the exact section headers — other skills grep for them.

```markdown
# Launcelot-Product-Legal — Practice profile

Last updated: <ISO date>
Cold-start version: 0.1.0

## Attorney profile

- Bar admission: <jurisdictions>
- Local bar: <association>
- Firm: <name or sole>
- Practice areas: <list>
- Featured/disclaimed titles: <list with exact framing>
- Admission year: <YYYY>

## Channel profile

- Channels: <name + platform list>
- Format mix: <percentages>
- Tone: <label + free text>
- Target viewer: <description>
- Length / cadence: <minutes / posts per week>

## Speech policy

### Real-name policy
Label: <selected>
Raw: <user's own words>

### Pending-case policy
Label: <selected>
Raw: <user's own words>

### Speculative-statement policy
Label: <selected>
Raw: <user's own words>

### Absolute-claim policy
Label: <selected>
Raw: <user's own words>

## Risk calibration

- Appetite: <보수적 | 중도 | 공격적>
- Past disputes:
  - <date> — <summary> — <outcome>
- Hard "never":
  - <line>
- Hard "must escalate":
  - <line>

## Substantiation

- Acceptable sources: <list>
- Case citation style: <full | 약식>
- Statute quotation style: <full | paraphrase + article | paraphrase only>
- Internal substantiation library: <path/URL>

## Integrations

- launcelot-lawyer: <available | not available> — <fallback note>
- wiki / njsidian-wiki: <…>
- kmj / kmg / sbh / ldg / wja: <…>
- koreanizer / humanizer: <…>
- cocounsel-legal:deep-research: not applicable (US-only)

## Validator wiring

When `marketing-claims-review` or `launch-review` runs:

1. Load this file.
2. Apply `## Speech policy` thresholds before scoring.
3. Apply `## Risk calibration` to set the floor (보수적 raises everything one step; 공격적 lowers borderline calls one step but never below `회색지대`).
4. Defer to `launcelot-lawyer` for all 조문·판례 lookups. Never assert a statute or judgment from model knowledge.
5. Honor every entry in `## Hard rules`.
```

## Interview style

- One question at a time. Wait for the answer. Do not list all questions at once.
- After each module, mirror back what was captured in two or three sentences. Ask "이대로 저장할까?" before moving on.
- Use Korean for the questions and short framing; the saved config uses the bilingual template above so other skills (which expect English structural headers) can still grep it.
- If the user says "skip" on a question, store `<not provided>` for that field rather than guessing. Other skills are designed to behave conservatively when a field is `<not provided>`.

## What this skill does NOT do

- Does not validate any script. That is `marketing-claims-review` and `launch-review`.
- Does not verify any 조문 or 판례. That is `launcelot-lawyer`.
- Does not change the user's existing scripts.
- Does not write to any path outside `~/.claude/plugins/config/launcelot-product-legal/`.
