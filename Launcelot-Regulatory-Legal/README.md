# Launcelot-Regulatory-Legal

한국 변호사 유튜브 대본 라이브러리에 영향을 줄 수 있는 한국 법령·판례·변협 공지의 신착을 감시하고, 현행 규범과 사용자 대본을 대조해 위반 가능성을 갭(gap)으로 추적·해소한다.

## Launcelot-Product-Legal와의 역할 분리

| | Launcelot-Product-Legal | Launcelot-Regulatory-Legal |
|---|---|---|
| 입력 단위 | 대본 1건 | 규범 변경 1건 또는 대본 라이브러리 |
| 시점 | 발행 전 게이트 | 발행 후 지속 감시 |
| 출력 | 게시 권고 + 라인별 수정안 | 신착 디제스트 / 갭 트래커 / 영향받는 대본 목록 |
| 본 플러그인이 하지 않는 것 | 대본 발행 후 모니터링 | 라인 단위 즉시 게이트 |

두 플러그인 모두 조문·판례 실존 검증은 `launcelot-lawyer`에 위임한다.

## 감시 대상 (기본값, cold-start-interview에서 조정)

- 국가법령정보센터 (law.go.kr) — 형법, 정보통신망법, 개인정보 보호법, 변호사법, 표시·광고의 공정화에 관한 법률, 저작권법 개정 이력
- 대법원 종합법률정보 (glaw.scourt.go.kr) — 명예훼손·모욕·무고·공갈·협박·사기·업무방해·개인정보·저작권·변호사법 관련 신착 판결
- 헌법재판소 (ccourt.go.kr) — 위 영역의 위헌·합헌 결정
- 국회의안정보시스템 (likms.assembly.go.kr) — 형법·정통망법·개보법·변호사법·표시광고법 개정안
- 대한변호사협회 공지 — 변호사 광고에 관한 규정 개정, 윤리장전 개정, 광고심의 사례
- 공정거래위원회 — 표시·광고 가이드라인, 변호사 광고 관련 심결례

## 스킬

- `cold-start-interview`: watchlist, materiality threshold, 대본 라이브러리 위치, 피드 cadence 학습.
- `customize`: 위 항목들을 한 항목씩 수정.
- `reg-feed-watcher`: 한국 공식 출처에서 신착을 수집하고 materiality threshold로 필터링한 디제스트 출력.
- `policy-diff`: 특정 규범 변경 또는 현행 규범 텍스트와 사용자 대본(또는 대본 라이브러리)을 대조해 위반 가능성을 추출.
- `gap-surfacer`: 추출된 위반 가능성을 트래커로 관리(open / 수정완료 / 리스크수용 / 비공개결정).
- `policy-redraft`: 위반 라인의 수정 대안을 생성. 톤 유지 + 진실성·공익성 항변 강화 + 식별성 완화.

## 설정 파일 위치

`~/.claude/plugins/config/launcelot-regulatory-legal/CLAUDE.md`

## 데이터 파일 위치

- `~/.claude/plugins/config/launcelot-regulatory-legal/gap-tracker.yaml` — 갭 트래커
- `~/.claude/plugins/config/launcelot-regulatory-legal/digests/<YYYY-MM-DD>.md` — 피드 디제스트
- `~/.claude/plugins/config/launcelot-regulatory-legal/diffs/<reg-slug>-<YYYY-MM-DD>.md` — diff 결과
