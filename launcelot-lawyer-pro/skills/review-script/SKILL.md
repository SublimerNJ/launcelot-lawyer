---
name: review-script
description: >
  Launcelot-Lawyer-Pro의 단일 진입점 wrapper. "대본 검토해", "이 대본 법률 검토", "발행 전
  검토", "review script" 한 마디로 발행 전 풀 흐름을 자동 체이닝한다. 두 모드. quick은
  is-this-a-problem(대본 전체 게시 가부 메타 점검) + marketing-claims-review 라인 단위
  추출의 종합 요약. full은 quick 위에 명백·가능 라인에 대한 feature-risk-assessment +
  launch-review 게시 권고까지 완주. 사용자가 대본을 붙여넣거나 파일·wiki 슬러그를 함께
  주면서 검토를 요청할 때 실행한다.
argument-hint: "[대본 텍스트 또는 파일 경로 또는 wiki 슬러그] [--mode quick|full] [--no-stop]"
---

# /launcelot-lawyer-pro:review-script

This skill is the single front door for the pre-publication flow. It chains the relevant skills in a defined order, surfaces a checkpoint between phases so the user can stop, and emits one consolidated report at the end.

It does NOT replace the underlying skills. It composes them. Every line of substantive output is produced by the underlying skill (`is-this-a-problem`, `marketing-claims-review`, `feature-risk-assessment`, `launch-review`). This skill's job is sequencing, deduplication, and one unified header.

It does NOT trigger any post-publication flow (`reg-feed-watcher`, `policy-diff`, `gap-surfacer`, `policy-redraft`, `comments`). Those stay user-driven by design — the post-publication flow operates on the library at scale and chaining it in here would burn context for a single-script review.

## Modes

### quick

Goal: 30초~2분 안에 대본의 위험 표면을 보여준다.

순서:

1. `is-this-a-problem`을 *대본 전체에 대한 메타 질문*으로 호출. 입력: "이 대본을 지금 그대로 게시해도 되나?" + 대본 전체. 결과는 즉시 가능 / 한 번 더 봐야 함 / 보류 한 줄 + 사유.
2. `marketing-claims-review`를 호출. 라인 단위 pull-bucket 추출 + 죄목 매핑 + 위험 등급. 캐시.
3. 두 결과를 한 페이지로 종합.
4. 다음 단계 한 줄 제안 (full로 더 깊게 / launch-review로 게시 권고 / 종료).

### full

Goal: 게시 권고까지 한 번에 완주.

순서:

1. quick 모드 1·2 단계 수행. 결과 캐시.
2. 캐시된 `marketing-claims-review`에서 등급이 `명백` 또는 `가능`인 라인들을 추출.
3. 각 라인에 대해 `feature-risk-assessment` 호출. 입력: 라인 + 죄목·규정 식별자. 결과 캐시.
4. `launch-review` 호출. **launch-review가 내부에서 다시 `marketing-claims-review`를 돌리지 않도록**, 본 단계에서 캐시된 결과를 launch-review에 전달한다(launch-review SKILL.md의 Run order 3·5단계 입력 슬롯에 캐시 경로를 명시). 캐시 재사용 불가 시(파일 손상 등)에만 다시 돌린다.
5. launch-review의 7카테고리 점검 + 게시 권고 메모 생성. revision으로 reviews/<slug>.md에 append.
6. 종합 출력: 본 wrapper의 헤더 + is-this-a-problem 결과 + marketing-claims-review 요약 + 명백·가능 라인의 feature-risk-assessment 압축본 + launch-review 메모 + 결정 로깅 프롬프트.

기본 모드: `--mode`가 생략되면 quick. 사용자가 "발행 전 풀 검토", "게시 권고까지", "full review" 같은 표현을 쓰면 full로 자동 추정하되, 자동 추정 결과를 헤더에 명시하고 사용자가 "아니 quick으로"라고 정정할 수 있게 한다.

## Run order

1. Load `~/.claude/plugins/config/launcelot-lawyer-pro/CLAUDE.md`. config 없으면 cold-start-interview로 리다이렉트.
2. Resolve `--mode` (quick / full / 자동 추정).
3. Resolve script input — paste / file path / wiki slug. 슬러그 계산은 launch-review와 같은 규칙(launch-review SKILL.md `## Run order` 3단계 참조). 매터 활성 여부 확인.
4. `## Integrations` → `launcelot-lawyer`의 가용성을 한 번 점검. 비가용이면 본 wrapper 출력 헤더에 미검증 배너를 한 번에 부착하고, 하위 스킬들의 동일 배너 출력은 중복 표시되지 않게 압축한다(하위 스킬 호출 시 옵션 `--banner-suppress`를 부여하고 본 wrapper가 단일 책임으로 배너를 단다).
5. 모드별 단계 실행. 단계 사이 체크포인트(아래) 적용.
6. 종합 메모 출력 + 결정 로깅 프롬프트.
7. 본 wrapper 실행 자체를 로그.

## 체크포인트

기본 동작은 각 단계가 끝나면 사용자에게 한 줄 요약과 함께 "계속할까?" 묻고 yes를 받은 뒤에만 다음 단계로 간다. 사용자가 `--no-stop`을 명시하면 단계 사이의 확인을 건너뛰고 끝까지 한 번에 간다.

quick 모드 체크포인트:

- C-Q1: is-this-a-problem 결과 표시 후 → marketing-claims-review로 계속할까?
- C-Q2: marketing-claims-review 요약 표시 후 → 종합 출력으로 진행할까?

full 모드 체크포인트:

- C-F1: is-this-a-problem 결과 표시 후 → marketing-claims-review로 계속할까?
- C-F2: marketing-claims-review 요약 표시 후 → 명백·가능 라인의 feature-risk-assessment로 계속할까?
- C-F3: feature-risk-assessment 결과 압축본 표시 후 → launch-review로 계속할까?
- C-F4: launch-review 메모 표시 후 → revision append + 결정 로깅을 진행할까?

체크포인트 직전에 본 wrapper는 *지금까지 본 것*만 보여주고 *앞으로 할 일*은 한 줄 미리보기로만 명시한다. 사용자가 중단하면 그 시점까지의 캐시를 디스크에 보존하고, 나중에 `review-script --resume <run-id>`로 재개할 수 있다.

## 캐시·재개

본 wrapper의 모든 단계 결과는 캐시 디렉토리에 저장된다:

`<scope>/review-script/<run-id>/`
- `meta.yaml` — 실행 메타(slug, mode, started_at, last_checkpoint, status).
- `is-this-a-problem.md`
- `marketing-claims-review.md` (라인 단위 풀 출력)
- `feature-risk-assessment/<line-id>.md`
- `launch-review.md` (메모 사본; launch-review가 reviews/<slug>.md에 append한 revision의 인용)

`<scope>`은 활성 매터에 따라 `matters/<slug>/review-script/` 또는 `review-script/`.

재개: `--resume <run-id>`. 마지막 체크포인트 이후부터 재개. 캐시가 손상된 단계가 있으면 그 단계부터 다시.

## 출력 종합 메모

```markdown
[게시 권고 — <게시 가능 | 수정 후 게시 | 발행 보류 권고 | 보류(추가 확인 필요)>]
[Mode: <quick | full>]  ·  [Matter: <slug | practice-level>]
[launcelot-lawyer: <available | not available>]

# Review — <slug>

Run ID: <run-id>
Started: <ISO datetime>
Completed: <ISO datetime>

---

## 1단계 결과 — is-this-a-problem (메타)

판정: <…>
사유: <…>

## 2단계 결과 — marketing-claims-review (라인 단위)

- 추출된 라인: <N>
- 등급 분포: 명백 <n> / 가능 <n> / 회색지대 <n> / 안전 <n>
- Hard-rule 히트: <n>
- 상위 3건:
  1. <…>
  2. <…>
  3. <…>

(전체 라인은 `<scope>/review-script/<run-id>/marketing-claims-review.md`)

## 3단계 결과 — feature-risk-assessment (full 모드 한정)

명백·가능 라인 <N>건 심층 분석.

| 라인 | 죄목·규정 | Axis 2 라벨 | Axis 3 최고 단계 | 권고 완화책 |
|---|---|---|---|---|
| <…> | <…> | <…> | <…> | <…> |

(전체는 `<scope>/review-script/<run-id>/feature-risk-assessment/`)

## 4단계 결과 — launch-review (full 모드 한정)

(launch-review 메모 본문 인용 또는 압축. 권고·카테고리 표·수정안 항목 그대로.)

연결된 revision: reviews/<slug>.md → Revision <N>

## 종합 결정

본 검토가 권고하는 단일 행동: <…>

(게시 가능 / 수정 후 게시 / 발행 보류 권고 중 하나, 또는 보류(추가 확인 필요))

## 결정 로깅 프롬프트

이 리비전에 대한 결정을 기록할까? (게시함 / 수정함 / 보류 / 비공개결정 / 나중에)
```

## 자동 추정 규칙

사용자가 `--mode`를 생략했을 때 본 wrapper가 모드를 추정하는 규칙:

- 메시지에 "풀 검토", "게시 권고", "full review", "전체 검토", "발행 결정"이 들어있으면 → full.
- 그 외에는 → quick.
- 추정 결과를 헤더에 명시: "Mode (자동 추정): quick. 풀 검토가 필요하면 `--mode full`로 다시 호출하라."

본 wrapper는 추정을 silently 적용하지 않는다.

## Calibration

본 wrapper는 calibration을 직접 적용하지 않는다. 하위 스킬들이 각자 calibration을 적용한다. 본 wrapper의 종합 권고는 launch-review의 권고를 그대로 사용하며, full 모드에서는 launch-review가 자동으로 marketing-claims-review와 feature-risk-assessment의 결과를 입력으로 받았으므로 calibration이 이미 반영된 상태다.

본 wrapper가 추가하는 calibration 동작은 단 하나: 사용자 risk appetite가 `보수적`인데 quick 모드 결과가 `명백` 또는 hard-rule 히트를 1건 이상 포함하면, 종합 권고에 "본 모드는 quick이므로 명백·가능 라인의 심층 분석이 빠져 있다. 보수적 calibration에서는 full 모드 권장."이라는 한 줄을 추가.

## Failure handling

- launcelot-lawyer 비가용: 본 wrapper 헤더에 한 번 배너. 하위 스킬 호출 시 `--banner-suppress` 또는 동등 옵션으로 중복 출력 방지. 단 미검증 인용은 각 단계 본문에 그대로 표시(인용 줄별 `[미검증]` 태그).
- 한 단계에서 캐시 작성 실패: 사용자에게 보고하고 본 wrapper 중단. 이미 작성된 캐시는 보존. 사용자가 `--resume <run-id>`로 재개.
- 사용자가 체크포인트에서 중단: `status: paused`로 캐시 저장. 재개 가능.
- full 모드에서 명백·가능 라인이 0건: feature-risk-assessment 단계를 건너뛰고 launch-review로 직행. 출력에 "심층 분석 건너뜀 — 명백·가능 라인 없음" 표시.
- 동일 slug에 대한 직전 review-script 실행이 같은 날 이미 있음: 새 run-id를 부여하되 사용자에게 직전 run의 결과를 먼저 보여주고 진행 여부 확인.

## 매터 컨텍스트

- 활성 매터가 있으면 본 wrapper의 모든 캐시·출력 경로가 매터-스코프로. 하위 스킬 호출도 그 매터 스코프로 전파.
- 매터가 비활성이면 프랙티스-레벨.

## Read-before-write

본 wrapper 실행 시작 시, `reviews/_index.yaml`에서 동일 slug의 직전 launch-review 결정 로그가 있는지 확인. 있으면 헤더에 "직전 검토: <date>, 권고 <…>, 결정 <…>"으로 표시.

## What this skill does NOT do

- 발행 후 흐름(reg-feed-watcher / policy-diff / gap-surfacer / policy-redraft / comments)을 자동으로 호출하지 않는다.
- 사용자 결정(게시함 / 보류 등)을 자동으로 기록하지 않는다. 결정은 launch-review의 결정 로깅 프롬프트 또는 본 wrapper의 종합 메모 끝의 프롬프트에서 사용자가 명시적으로 답해야 기록된다.
- 사용자의 원본 대본을 수정하지 않는다.
- 새로운 조문·판례를 단정하지 않는다(하위 스킬들이 launcelot-lawyer 위임 원칙을 그대로 따른다).
- 하위 스킬들의 출력 포맷을 임의로 바꾸지 않는다. 종합 페이지에 압축하긴 하지만, 원본 풀 출력은 캐시 디렉토리에 그대로 보존된다.
