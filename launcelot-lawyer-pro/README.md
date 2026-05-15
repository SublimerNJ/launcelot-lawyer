# Launcelot-Lawyer-Pro

한국 변호사 유튜브 대본의 발행 전 게이트와 발행 후 지속 감시를 한 플러그인으로 통합한 한국법 검증 스튜디오. 모든 조문·판례 인용은 본 플러그인이 직접 fetch한 원문(스니펫)에 닿아 있어야 한다. 모델 지식으로 단정하지 않는다.

## 무엇을 하는가

대본 1건이 들어오면:

1. `marketing-claims-review`로 형법적·행정법적 평가가 가능한 문장을 모두 추출하고 죄목별로 매핑한 위험 등급 표를 만든다.
2. `launch-review`로 7개 카테고리 체크리스트(변호사법·광고규정 / 명예훼손·모욕 / 저작권 / 개인정보 / 무자격 자문 / 표시광고법 / 인접 영역)를 돌려 게시 권고를 낸다.
3. 게시한 뒤에는 `reg-feed-watcher`가 한국 공식 출처(국가법령정보센터·대법원·헌재·국회의안정보·변협·공정위 등)의 신착을 끌어와 디제스트로 알려준다.
4. 신착이 사용자의 대본 라이브러리에 영향을 줄 가능성이 있으면 `policy-diff`로 라이브러리 전체와 대조해 어느 대본의 어느 라인이 걸리는지 갭으로 추출한다.
5. 갭은 `gap-surfacer`/`gaps`가 트래커로 관리하고, `policy-redraft`가 수정 후보 3종을 만든다.
6. 입법예고·공청·변협 회칙 개정 의견서 기한은 `comments`가 추적한다.

조문·판례 인용은 본 플러그인이 직접 한다. `references/snippet-protocol.md`에 정의된 3단 fetch fallback(WebFetch → firecrawl → jina)으로 매번 원문을 확보한 뒤 그 텍스트에서만 인용을 만든다. fetch가 모두 실패하면 결과를 만들지 않고 종료한다.

## 워크플로우

### 단일 진입점 (권장)

```
[대본 작성]
   │
   ▼
/launcelot-lawyer-pro:review-script <대본>
   ├─ --mode quick → is-this-a-problem + marketing-claims-review 요약
   └─ --mode full  → quick + 명백·가능 라인의 feature-risk-assessment + launch-review 게시 권고
```

`review-script`는 발행 전 풀 흐름을 한 마디로 돌리는 wrapper이다. 단계 사이에 체크포인트가 있어 사용자가 중단·재개할 수 있다(`--no-stop`으로 무중단 가능, `--resume <run-id>`로 재개).

### 개별 스킬 (수동 조립)

```
[대본 작성]
   │
   ▼
/launcelot-lawyer-pro:is-this-a-problem  ← 빠른 가부 (선택)
   │
   ▼
/launcelot-lawyer-pro:marketing-claims-review  ← 라인 단위 추출·매핑
   │
   ▼
/launcelot-lawyer-pro:feature-risk-assessment  ← 단일 라인 심층 (필요 시)
   │
   ▼
/launcelot-lawyer-pro:launch-review            ← 게시 권고 (reviews/<slug>.md)
   │
   ▼ [게시]
   │
   ▼ (주기적)
/launcelot-lawyer-pro:reg-feed-watcher         ← digests/<date>.md
   │
   ▼ (Tier 1 영향)
/launcelot-lawyer-pro:policy-diff              ← diffs/<reg>-<date>.md
   │
   ▼
/launcelot-lawyer-pro:gap-surfacer (add)       ← gap-tracker.yaml
   │
   ▼
/launcelot-lawyer-pro:gaps                     ← 사용자 친화 조회
   │
   ▼
/launcelot-lawyer-pro:policy-redraft           ← redrafts/<gap>-<date>.md
   │
   ▼
/launcelot-lawyer-pro:launch-review (재실행)   ← revision <N> append, close gap
```

발행 후 흐름(reg-feed-watcher 이하)은 `review-script` wrapper에 포함되지 않는다. 단일 대본 검토와 라이브러리 단위 모니터링은 의도적으로 분리되어 있다.

## 스킬 12종

### 단일 진입점 (1)

- `review-script` — 발행 전 풀 흐름 wrapper. quick / full 두 모드. 체크포인트·캐시·재개 지원.

### 발행 전 게이트 (4)

- `is-this-a-problem` — "이거 해도 되나?" 즉답. 단발 질의·단발 표현용.
- `marketing-claims-review` — 라인 단위 추출·죄목 매핑·위험 등급. 조문/판례 snippet을 직접 fetch.
- `feature-risk-assessment` — 단일 라인 심층 리스크 분석(무엇이/얼마나/어떻게). 후보 죄목의 snippet 첨부.
- `launch-review` — 7카테고리 체크리스트 + 게시 권고 + revision append.

### 발행 후 모니터링 (4)

- `reg-feed-watcher` — 한국 공식 출처 신착을 직접 fetch해 Tier 1/2/3 디제스트로.
- `policy-diff` — 규범 텍스트와 대본(또는 라이브러리) 라인 단위 대조.
- `policy-redraft` — 위반 라인의 수정 후보 3종.
- `comments` — 입법예고·공청·회칙 개정 의견서 기한 트래킹.

### 트래킹 (2)

- `gap-surfacer` — 갭의 add/close/accept-risk/unpublish/snooze. 패턴 갭 검출.
- `gaps` — 사용자 친화 조회 단일 진입점(read 위주).

### 매터 (1)

- `matter-workspace` — 다수 채널·다수 의뢰인 사건을 다룰 때의 컨텍스트 분리. new/list/switch/close/detach. 매터에는 사실 메모만 둔다(평가 규칙·스타일 설정은 받지 않는다).

## 조문·판례 검증 원칙

본 플러그인의 모든 검증성 출력은 `references/snippet-protocol.md`에 정의된 3단 fetch fallback을 거친 raw_quote 위에서만 만들어진다.

1. **WebFetch**로 출처(law.go.kr·glaw.scourt.go.kr·ccourt.go.kr·koreanbar.or.kr·ftc.go.kr·pipc.go.kr·likms.assembly.go.kr 등)를 직접 호출.
2. 실패 시 **firecrawl** 스킬.
3. 실패 시 **jina** MCP `read_url`.
4. 셋 다 실패하면 **결과를 만들지 않고 종료**. "스니펫 미확보" 라벨로 우회 출력하지 않는다.

외부 스킬(`launcelot-lawyer` 포함)에 조문 검증을 위임하지 않는다. 모델 지식으로 조문 텍스트를 단정하지 않는다.

## Agent 3종

- `launch-watcher` — 사용자가 라이브러리에 새 대본을 올리면 launch-review를 권유.
- `reg-feed-watcher-agent` — cadence 도래 시 reg-feed-watcher 실행 권유.
- `gap-aging-agent` — aging threshold 초과 갭을 정기 표면화.

모든 agent는 사용자 동의 없이 외부 송신·자동 결정을 하지 않는다.

## 설정·데이터 위치

루트: `~/.claude/plugins/config/launcelot-lawyer-pro/`

본 플러그인은 _사용자 프로필·스타일·리스크 감수도 같은 사전 설정을 받지 않는다_. 다음 파일들만 사용한다.

- `watchlist.yaml` — (선택) 사용자가 자연어로 좁힌 출처 설정. 없으면 `references/default-watchlist.md`의 10개 출처를 그대로 사용.
- `script-library.yaml` — (선택) 사용자가 등록한 라이브러리 위치. 없으면 처음 사용 시 lazy로 묻고 기록.
- `gap-tracker.yaml` — 갭 본체. 첫 갭 등록 시 자동 생성.
- `gap-tracker.audit.log` — append-only 액션 로그.
- `patterns.yaml` — 패턴 갭 메타.
- `matters/_index.yaml` — 매터 인덱스 (matter-workspace 활성화 시).
- `matters/<slug>/matter.md` — 매터별 사실 메모(콘텐츠 평가 규칙은 두지 않음).
- `reviews/<slug>.md` — launch-review 리비전.
- `reviews/_index.yaml` — 검토 인덱스.
- `digests/<date>.md` — reg-feed-watcher 디제스트.
- `digests/_index.yaml`, `_seen.yaml`, `_unresolved.md` — 디제스트 메타.
- `diffs/<reg>-<date>.md` — policy-diff 결과.
- `diffs/_index.yaml`.
- `redrafts/<id>-<date>.md` — policy-redraft 결과.
- `redrafts/_index.yaml`.
- `comments/<bill-id>.md` — 의견서 트래커.
- `comments/_index.yaml`.

## 의존성

- 내장 `WebFetch` 도구 — 1차 fetch 수단.
- `firecrawl` 스킬 — 2차 fallback. 설치돼 있으면 우선 사용.
- `jina` MCP — 3차 fallback. `mcp__jina__read_url`.
- `njsidian-wiki` MCP — (선택) 대본 라이브러리·과거 검토 이력 접근. 비가용 시 사용자가 수동으로 대본을 붙여넣어야 한다.
- `koreanizer`, `humanizer`, `kmj`, `kmg`, `sbh`, `ldg`, `wja` 스킬 — (선택) policy-redraft가 후보 라인을 만들면 톤 마감과 화자별 보이스 적용을 이 스킬들에 위임할 수 있다.

## 무엇을 하지 않는가

- 사용자에게 변호사 프로필·채널 정보·발화 정책·리스크 감수도·Hard rules·스타일 같은 사전 설정을 묻지 않는다. 설치 즉시 사용 가능.
- 사용자의 원본 스크립트 파일을 수정하지 않는다.
- 외부 송신(Slack·이메일 등)을 사용자 미리보기·yes 없이 보내지 않는다.
- 조문·판례 텍스트를 모델 지식으로 단정하지 않는다. snippet-protocol로 fetch한 raw_quote만 인용한다.
- 외부 스킬(`launcelot-lawyer` 포함)에 조문·판례 검증을 위임하지 않는다.
- 게시·비공개·법적 결정을 자동으로 하지 않는다. 모든 결정은 변호사의 몫이다.
- 미국 법(FTC·NAD·CCPA·GDPR 등) 기반 분석을 하지 않는다. 본 플러그인은 한국법 전용이다.
