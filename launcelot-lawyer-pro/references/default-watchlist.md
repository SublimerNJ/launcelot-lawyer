# Default Watchlist — 한국 공식 출처 기본값

본 플러그인은 사용자에게 watchlist를 미리 묻지 않는다. `reg-feed-watcher`·`policy-diff`가 watchlist 입력 없이 호출되면 이 문서에 정의된 기본 세트를 그대로 사용한다.

사용자가 출처를 좁히고 싶으면 reg-feed-watcher 첫 실행 후 디제스트를 보고 "이 출처는 빼" / "이 키워드만" 같은 자연어 지시로 한 번에 좁힌다.

## 기본 세트 (10개 출처)

```yaml
sources:
  - id: law_go_kr
    name: 국가법령정보센터
    operator: 법제처
    url: https://www.law.go.kr
    trust_tier: 공식
    content_type: 법령 본문·개정 이력·시행일
    monitoring_keywords:
      - 형법 (특히 307, 309, 311, 156, 283, 347, 314, 350)
      - 정보통신망 이용촉진 및 정보보호 등에 관한 법률 (특히 70조)
      - 개인정보 보호법
      - 변호사법 (특히 23조)
      - 표시·광고의 공정화에 관한 법률
      - 저작권법 (특히 28조)
    cadence_default: weekly

  - id: glaw_scourt
    name: 대법원 종합법률정보
    operator: 대법원 법원도서관
    url: https://glaw.scourt.go.kr
    trust_tier: 공식
    content_type: 대법원·각급 법원 판례 전문, 판시사항
    monitoring_keywords:
      - 명예훼손
      - 정통망법상 명예훼손
      - 모욕
      - 무고
      - 협박
      - 공갈
      - 사기 (사이버)
      - 업무방해
      - 진실성·공익성 항변 (형법 310)
      - 정당행위 (형법 20)
      - 개인정보 보호법 위반
      - 저작권법 28조 인용
    cadence_default: weekly

  - id: ccourt_go_kr
    name: 헌법재판소
    operator: 헌법재판소
    url: https://www.ccourt.go.kr
    trust_tier: 공식
    content_type: 헌법소원·위헌법률심판 결정
    monitoring_keywords:
      - 형법 307 위헌 여부
      - 형법 311 위헌 여부
      - 정통망법 70 위헌 여부
      - 표현의 자유 관련 결정
    cadence_default: weekly

  - id: koreanbar
    name: 대한변호사협회
    operator: 대한변호사협회
    url: https://www.koreanbar.or.kr
    trust_tier: 공식
    content_type: 변호사 광고에 관한 규정, 변호사윤리장전, 광고심의위 심의결과·공지
    monitoring_keywords:
      - 변호사 광고 규정 개정
      - 광고심의위원회 심의결과
      - 결과 보장 표현
      - 비교광고
      - 사건유치 직접 유도
      - 비변호사 동업
      - 전문분야 표시
    cadence_default: weekly

  - id: ftc_go_kr
    name: 공정거래위원회
    operator: 공정거래위원회
    url: https://www.ftc.go.kr
    trust_tier: 공식
    content_type: 표시·광고 심결례, 부당한 표시·광고행위 가이드라인, 비교광고 심사지침
    monitoring_keywords:
      - 변호사 사무소 표시광고
      - 부당한 표시·광고
      - 비교광고 심사지침 개정
    cadence_default: 3-day

  - id: pipc_go_kr
    name: 개인정보보호위원회
    operator: 개인정보보호위원회
    url: https://www.pipc.go.kr
    trust_tier: 공식
    content_type: 결정·의결, 분쟁조정 사례, 가이드라인
    monitoring_keywords:
      - 의뢰인 식별정보 노출
      - 영상 콘텐츠 개인정보 가이드
    cadence_default: weekly

  - id: copyright_or_kr
    name: 한국저작권위원회
    operator: 한국저작권위원회
    url: https://www.copyright.or.kr
    trust_tier: 공식
    content_type: 저작권법 해설, 사례 DB, 분쟁조정
    monitoring_keywords:
      - 인용 적정성
      - 보도자료 인용
      - 판결문 게시 적정성
    cadence_default: weekly

  - id: likms_assembly
    name: 국회의안정보시스템
    operator: 대한민국 국회
    url: https://likms.assembly.go.kr
    trust_tier: 공식
    content_type: 의안(법률안) 검색, 심사 진행, 본회의 통과 여부
    monitoring_keywords:
      - 형법 개정안
      - 정통망법 개정안
      - 변호사법 개정안
      - 개인정보 보호법 개정안
      - 표시·광고의 공정화에 관한 법률 개정안
    cadence_default: weekly

  - id: kcc_go_kr
    name: 방송통신위원회
    operator: 방송통신위원회
    url: https://www.kcc.go.kr
    trust_tier: 공식
    content_type: 정보통신서비스 관련 결정·정책
    monitoring_keywords:
      - 정통망법 시행령 개정
      - 임시조치
    cadence_default: weekly

  - id: seoulbar
    name: 서울지방변호사회
    operator: 서울지방변호사회
    url: https://www.seoulbar.or.kr
    trust_tier: 공식
    content_type: 회원 공지, 광고 가이드라인 보충, 징계 사례
    monitoring_keywords:
      - 서울회 광고 가이드라인
      - 서울회 징계 사례 (공개분)
    cadence_default: weekly
    optional: true # 사용자가 서울회 소속일 때만 실제 fetch. 미설정 시에도 카탈로그에는 둠.
```

## 기본 cadence

- 전역: weekly
- Tier 1 (즉시 알림) 검출 시: 그 디제스트는 cadence 무시하고 즉시 사용자에게 노출
- Quiet hours: 23:00~07:00 (KST) — Tier 1도 이 시간대에 디제스트 안 띄움, 다음 활성 시간에 노출

사용자 별도 지정이 있으면 그것만 override.

## 기본 materiality (Tier 정의)

```yaml
tier1: # 즉시 알림 — cadence 무시
  checklist:
    - watchlist의 monitoring_keywords와 직접 매칭되는 조문/판례/규정 신설·개정·폐지
    - 사용자 라이브러리 대본의 *명백/가능* 위험 라인 평가 결과에 직접 영향을 줄 수 있는 변경
    - 변호사 광고규정의 결과 보장·비교광고·전문분야 표시 관련 항목 변경
    - 형법 307·311 또는 정통망법 70 위헌 결정의 결론 변경

tier2: # 정기 디제스트에 포함
  checklist:
    - watchlist 출처의 일반 신규 게시물(가이드라인·정책 자료·심결례)
    - 사용자 라이브러리에 직접 영향은 없지만 인접 영역에 적용될 수 있는 변경
    - 의안 발의·소관위 회부 단계 (통과 전)

tier3: # 로그만
  checklist:
    - 단순 페이지 리뉴얼, 표기 수정
    - 사용자 라이브러리·관심 키워드와 무관한 출처 변경
```

## 디제스트 출력 경로

기본값: `~/.claude/plugins/config/launcelot-lawyer-pro/digests/<YYYY-MM-DD>.md`

폴더가 없으면 첫 실행 시 자동 생성. 사용자가 별도 경로 지정 시 그것 사용.

## 사용자가 출처를 좁히는 방법

watchlist는 **사용자가 처음부터 일일이 정하지 않는다**. 다음 흐름으로 자연어 조정한다.

1. `/launcelot-lawyer-pro:reg-feed-watcher` 첫 실행 → 기본 10개 출처로 디제스트 생성.
2. 디제스트 보고 "공정위는 빼" / "저작권위는 분기에 한 번만" 같이 자연어 지시.
3. 그 지시를 watchlist.yaml에 반영(없으면 생성). 다음 실행부터 반영된 watchlist 사용.

처음부터 카탈로그를 사용자에게 열어두고 일일이 선택받지 않는다.
