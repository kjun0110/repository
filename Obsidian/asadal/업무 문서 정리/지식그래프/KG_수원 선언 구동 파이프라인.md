> 2026-09-07 기준. [[KG 수원 - 전체 파이프라인 1차구현]] 은 09-04 상태이고, 이 문서가 그 뒤에 바뀐 구조를 담는다.

## 개념

- **온톨로지 구동** — 선언이 파싱·검증·적재·추론·라우팅을 *만든다*. 선언 한 줄을 고치면 코드를 안 건드려도 검사 수·제약 수·질의 모양이 바뀐다
- **선언(declaration)** — `ontology/schema.yaml` · `sites/suwon.yaml`. 구조와 유효한 값의 집합
- **사람 결정(decisions)** — `decisions/*.yaml`. "이 건은 이렇게 본다"는 판정. 근거(`why`)와 승인자(`by`)가 붙는다
- **규칙 적용기** — 선언한 규칙을 한 바퀴 돌리는 것. 고정점까지 반복하는 추론기와 다르다
- **물질화(materialize)** — 파생 사실을 간선으로 저장할지, 질의 때 풀지
- **조용한 실패** — 실패했는데 결과가 멀쩡해 보이는 것. 검증기가 못 잡는다

---

## 규모

```
선언        1,809줄   schema.yaml 424 + suwon.yaml 333 + decisions 6파일 1,052
코드       17,150줄   scraping 5,230 · search 3,463 · nodes 2,244 · validate 1,679 · ontology 1,078

그래프      노드 24,295 · 간선 31,270 (파생 849) · 벡터 10,743
           제약 14 · 인덱스 22 (풀텍스트 4 · 벡터 1 · 범위 17)

선언 내용   라벨 14 · 관계 18 · 어휘 11종 46값 · 속성 87 · slots 6
           추론 규칙 2 · 전이 선언 3 · 사람 결정 251건
선언이 만드는 것  제약 14개 · 판정 139건 · 파생 849간선 · 검색 어휘 165항목
```

`ontology/schema.yaml` 과 `decisions/` 를 지우면 **적재도 검증도 검색도 안 뜬다.** 실제로 `schema.yaml` 을 숨기고 돌리면 네 단계가 전부 `FileNotFoundError` 로 죽는다.

##### 라벨별 규모

```
Attachment 7,647 · Article 6,285 · Employee 5,136 · Place 2,509 · OrgUnit 940
CivilService 644 · DocumentType 609 · WasteItem 203 · Regulation 192
Domain 76 · Board 20 · TargetGroup 12 · Issuer 12 · Channel 10
```

---

## 전체 흐름

```
scraping (파이프라인 밖 · 주기 선언)
    ↓  data/raw/
[1] validate/07  산출물 순서       선언을 고치고 수집을 안 했나
[2] validate/08  선언·코드 대조     같은 사실이 두 벌인가
[3] schema/01    제약·인덱스        labels[].key 에서 생성
[4~24] nodes/ · relationships/     적재 21단계
[25] validate/01 노드 요건          classes 의 requires · exempt_if
[26] relationships/05  추론         inference 선언에서 Cypher 생성
[27] validate/06 스키마 대조        선언에서 판정 139건
[28] lifecycle/01 폐지 판정
[29~32] embedding/01~04            :Entity · article_searchable
[33] validate/05 품질 게이트        gold 48문항 6축. 나빠지면 exit 1
```

##### 실행 모드 셋

```
python main.py                    적재 33단계 (약 20분)
python main.py --collect=daily    게시글 수집 후 적재
python main.py --check            LLM·쓰기 없이 "값이 안 변했나" 만 (약 1분)
```

---

## 1. 수집 — 선언이 지시한다

**입력** `www.suwon.go.kr` · **출력** `data/raw/`

##### 읽는 선언 — `sites/suwon.yaml`

```
endpoints    board_list · board_view                          2개
selectors    본문·첨부·목록·머리글 …                             9개
crawl        user_agent · delay 0.6 · default_days 365
             refetch_days 30 · seen_stop_pages 2 · max_requests 150000
boards       128개 (take:true 20 · 게시판별 days)
robots       Disallow 경로에 대한 결정 기록
```

##### 주기도 선언에 있다 — `main.py COLLECT`

```
daily    게시글           증분·조기 종료·LLM 없음 · 약 11분
weekly   조직·민원·추출 12단계  LLM 호출 5개가 전부 여기
monthly  업무편람          40MB · 거의 안 바뀐다
```

STEPS 에 안 넣은 이유: 수집은 멱등하지 않고 비용이 든다. 외부 요청 4개·LLM 호출 5개가 들어 있어 `main.py` 한 번이 몇 시간이 된다. **밖에 두되 이름을 붙였다** — 스케줄러는 주기 이름만 알면 된다.

##### 창(`days`)과 조기 종료는 성격이 반대다

```
조기 종료   "아는 글만 2쪽 연속이면 멈춘다"   매일 도는 증분 수집을 위한 것
창 확대     "옛 글도 받는다"                한 번 하는 소급 수집
```

창만 넓히면 조기 종료가 막는다 — 새로 받아야 할 옛 글은 목록 12쪽 아래에 있다. `--only=<ids>` 로 지정 게시판만 `--full` 로 돈다.

`bcut`(게시판별 기준일)은 걸러진 목록이 아니라 `BOARDS` 전체로 계산한다. 걸러진 목록으로 만들면 안 훑은 게시판의 기준일이 전역 365일로 떨어져 `days: null` 게시판의 옛 글이 병합에서 통째로 탈락한다.

##### 여기서 막는 것

```
max_requests 150000   정상 최대(--full 전량 ≈ 7만)의 두 배. 넘으면 URL 증식 버그다
                      시험: 상한 3으로 낮추면 "[중단] 요청 4건이 상한 3 을 넘었다"
```

---

## 2. 파일을 보는 검사 — 적재 전 두 단계

##### `validate/07` 산출물 순서

```
선언 범위가 바뀌었다     FAIL   "선언을 고치고 수집을 안 했다"
수집 스크립트가 새롭다    WARN   주석만 고쳐도 mtime 이 바뀐다
산출물 나이            리포트  게시글 7일 · 나머지 60일
```

**mtime 이 아니라 범위 지문을 비교한다.** mtime 으로 보면 범위와 무관한 값(`max_requests`·`user_agent`·`delay_sec`)을 고쳐도 재수집을 요구한다 — 실제로 그 오탐이 났다. 지문은 `endpoints`·`selectors`·`boards`·`default_days` 만 해시하고, 수집기가 계산해 산출물에 적는다.

```
delay_sec 만 고치면       FAIL 0   (범위 무관)
default_days 를 고치면    FAIL 1   "수집 범위가 바뀌었다 (7ef965fe -> a4019906)"
```

##### `validate/08` 선언·코드 대조

**`validate/06` 은 선언과 *그래프*를 대조한다. 코드 파일은 안 본다.** 그래서 같은 사실이 선언과 코드에 두 벌 있어도 통과한다.

```
schema.yaml       org_classes -> unit_level 17개
structure.py      UNIT_LEVELS = [...같은 17개...]
```

여기서 하나만 고치면 **적재는 잘 되고 검색만 조용히 틀린다.** 데이터가 멀쩡하니 그래프 검사로는 영원히 안 잡힌다.

```
코드 상수가 늘었다      WARN   기준선 15개 (data/baseline/code_constants.json)
선언 값과 80% 겹친다    WARN   같은 사실이 두 벌인가
```

기준선을 **자동 갱신하지 않는다.** 새로 생긴 것이 조용히 기준이 되면 이 검사가 하려는 일이 무너진다. `quality.json` 과 반대 방침이고 이유가 파일 주석에 있다.

---

## 3. 온톨로지 — 무엇이 선언에 있나

##### 구조

```yaml
labels:                          # 14개
  CivilService:
    ko: 민원
    desc: 시민이 신청·문의하는 절차(procedure) 또는 참고 자료(reference) 페이지
    key: civil_id
    entity: true                 # :Entity 가 붙어 벡터 검색의 입구가 된다

relationships:                   # 18종
  PART_OF:
    from: OrgUnit
    to: OrgUnit
    cardinality: "N:1"
    acyclic: true
    transitive: true
    materialize: false           # 일부러 저장 안 한다
    read_by: [search]
```

##### 관계 18종

```
관계                from          -> to            카디널리티  비고
AVAILABLE_VIA      CivilService  -> Channel       N:M
BASED_ON           CivilService  -> Regulation    N:M
DISPOSAL_FEE_IN    WasteItem     -> CivilService  N:1
FOR_TARGET         CivilService  -> TargetGroup   N:M
HANDLED_BY         CivilService  -> OrgUnit       N:M
HAS_FILE           Article       -> Attachment    1:N
IMPLEMENTS         Regulation    -> Regulation    N:1   전이
IN_BOARD           Article       -> Board         N:1
IN_CHARGE_BUREAU   CivilService  -> OrgUnit       N:1   파생
IN_DOMAIN          CivilService  -> Domain        N:1   미사용
ISSUED_AT          DocumentType  -> Issuer        N:M
LISTED_IN          Place         -> CivilService  N:1
PARENT_OF          Domain        -> Domain        1:N   전이 · 미사용
PART_OF            OrgUnit       -> OrgUnit       N:1   전이
POSTED_BY          Article       -> OrgUnit       N:1
REQUIRES           CivilService  -> DocumentType  N:M
VISIT_FOR          CivilService  -> Issuer        N:M   파생 · 미사용
WORKS_AT           Employee      -> OrgUnit       N:1
```

##### 값

```
vocabulary 11종 46값
  content_type 2 · article_type 8 · page_kind 3 · channel 10 · target_group 12
  schedule_kind 4 · external_org 14 · no_issue 3 · facility_suffix 3
  inquiry_labels 3 · inquiry_end 6

slots 6            어느 라벨의 어느 속성이 어느 어휘를 쓰는지
properties 87      type · required(55) · note(7)
article_searchable 벡터 검색에 넣을 게시글 유형 4종
schedule_labels    일정 종류별 답변 문구
page_kind_rule     마커 5개 · 임계 4
```

`slots` 가 필요한 이유: `classes` 가 `content_type == 'procedure'` 라고 쓰면서 **어느 라벨의 속성인지 말하지 않았다.** 이제 `validate/06` 이 "어휘 밖의 값이 그래프에 있나"를 자동으로 본다.

##### 개수는 적지 않는다

09-07에 규모 감각용으로 `n:` 을 넣었다가 **같은 날 안에 32개 중 6개가 틀어졌다**(Article 선언 6,282·실제 6,285). 아무도 안 지키는 숫자는 감각이 아니라 오해다. `coverage` 도 같은 이유로 뺐고, "비어 있는 것이 정상인가"를 설명하는 값만 `note` 로 남겼다.

```
Employee.name             "사이트가 직원 이름을 안 싣는다. 기관장 6명만 있다"
CivilService.external_org "수원시 소관이면 비는 것이 맞다"
DocumentType.issued_at    "발급처를 판정 못 한 서류. 추측하지 않는다"
```

---

## 4. 사람 결정 — 251건, 근거와 승인자 필수

##### 어휘와 무엇이 다른가

```
vocabulary        유효한 값의 집합       "채널은 이 10개뿐"
decisions/*.yaml  개별 건에 대한 판정     "무인민원발급기는 구비서류가 없다"
```

후자에만 근거와 승인자가 붙는다.

##### 여섯 파일

```
search_aliases.yaml  157   시민어 -> 행정어. 묶음마다 실측 근거
places.yaml           35   표 이름 -> (place_type, operator)
org_labels.yaml       29   조직명 -> unit_level 예외
issuers.yaml          13   원문 발급처 문구 -> Issuer
extraction.yaml        9   LLM 오추출 교정 · 페이지 분할
aliases.yaml           7   문서·조직·레벨·건물 별칭
```

##### 형식

```yaml
장안구보건소:
  value: 직속기관
  why: 직속기관(보건소·센터). 이름만으로는 과·팀과 안 갈린다
  decided: 2026-09-07 이전
  by: 미상
```

`why` 나 `by` 가 없으면 **로드 자체가 중단된다.** `by: 미상` 은 정직하게 적었다 — 누가 정했는지 실제로 모르고, 그게 이 작업을 한 이유다.

별칭은 **묶음마다** 근거를 단다. 항목 하나하나가 아니라 범주 전체가 한 번의 판단이었기 때문이다.

```yaml
term_aliases:
  배출 행위:
    value: { 버리는: [폐기물, 배출, 스티커], 버릴: […], … }
    why: >
      시민은 '버린다'고 하고 행정은 '배출/폐기물'이라고 쓴다. 이 범주가 비어 있어서
      '냉장고 버리는 비용 얼마야'의 정답이 어휘검색 top5 밖이었다.
      '대형폐기물'을 붙이는 것은 실측으로 기각했다 — 종량제봉투·재활용 질의의
      정답이 4위로 밀렸다.
```

##### 여기서 막는 것

```
형식 검사    why 를 빼고 로드   -> rc=1 "결정에는 근거(why)와 승인자(by)가 있어야 한다"
대상 존재    '장안구보건서' 오타 -> [주의] "조용히 아무 일도 안 한다"
```

**결정이 잘못 적용된 것은 다른 검사가 잡지만, 아예 적용 안 된 것은 결과가 멀쩡해서 아무도 못 잡는다.**

##### 코드에 남긴 것 15개 — 전부 이유가 있다

```
파싱 로직    조사 18 · 의문사 14 · 법령 접미사 33 · 라벨→종류 매핑 12
도구 스키마   LLM 함수 정의 3종 (출력 파서와 필드가 맞물린다)
표시·실험    답변 문구 · 브리지 비교 질의 27
```

기준은 하나다 — **바꾸려면 그 옆 코드도 같이 바꿔야 하면 코드에 둔다.**

---

## 5. 적재 — 21단계

##### 순서가 의미를 갖는 자리

```
nodes/01 조직  ->  validate/00 원천 모순  ->  nodes/04 민원
relationships/03(HANDLED_BY) ·04(REQUIRES) ·nodes/15(ISSUED_AT)  ->  relationships/05 추론
```

`validate/01`(노드 요건)이 관계 단계 뒤인 이유: "담당부서가 없다"를 판정하려면 `HANDLED_BY` 가 먼저 서야 한다. 앞에 두면 전부 위반으로 잡힌다.

##### 위반을 지우지 않는다

`validate/01` 은 노드를 그대로 두고 `violations` 속성에 코드만 남긴다. 버리면 "왜 이 민원이 검색에 안 나오지"를 나중에 추적할 수 없다.

```
content_type   전체   담당부서없음   구비서류없음   절차없음
procedure      263        5            77          51
reference      381        8           372         375
```

`reference` 가 구비서류·절차 없는 것은 정상이다(381 중 372·375). **같은 빈칸이 클래스에 따라 결함이기도 하고 정상이기도 하다** — 무엇이 필수인지 아는 것이 온톨로지의 일이다.

##### `page_kind` — 목록 페이지 판정

민원 페이지가 단일 절차인가, 여러 서비스를 모은 대문인가.

```
CivilService 644  ->  단일 618 · 목록 26
정상 0~2회    인감증명발급 0 · 여권 신청 0 · 효도수당 2 · 난임부부 2
목록 4회+     복지·문화 50 · 출산 전 지원 20 · 일자리 20 · 출산 후 지원 9
```

대문 페이지의 구비서류·절차·수수료는 **페이지 전체의 것이 아니라 안에 실린 서비스 중 하나의 것**이다. 노드는 지우지 않는다 — `맞춤형 기초생활보장제도`(9회)는 gold 정답이고, 본문은 쓸모가 있고 구조화 필드만 못 믿는다.

##### 식별자

```
Employee 본체   <부서코드>_<번호>       부서코드가 들어 있어 충돌이 없다
Employee 동기화  live_ + sha1(org_id|직위|전화|업무)
```

동기화 쪽이 원래 **조직 이름**을 썼다. 같은 팀 이름이 여러 구에 있으면 다른 사람이 한 노드로 접혔다(원천 7묶음 35명). `org_id` 로 고쳤다.

이름은 못 쓴다 — 사이트가 직원 이름을 안 싣는다. 저장된 조직 페이지 232개의 표 500개 중 성명 열이 있는 것은 2개(기관장 페이지)뿐이다.

---

## 6. 추론 — 선언에서 Cypher 를 만든다

```yaml
inference:
  IN_CHARGE_BUREAU:
    kind: chain
    via:
      - rel: HANDLED_BY
      - rel: PART_OF
        depth: "0..5"                        가변 깊이
        where: { unit_level: bureau_levels }  노드 필터
        nearest: true                        가장 가까운 하나
        tie_break: org_id                    동점 처리
  VISIT_FOR:
    kind: chain
    via: [{ rel: REQUIRES }, { rel: ISSUED_AT }]
```

`ontology/infer.py` 가 이것을 Cypher 로 바꾼다. 그래프에 안 붙어서 DB 없이 시험할 수 있고, 실제로 그렇게 시험한다.

##### 출처가 붙는다

```
r.derived = true
r.rule    = 'HANDLED_BY / PART_OF*0..5'
```

`IN_CHARGE_BUREAU` 583 · `VISIT_FOR` 266. `validate/06` 이 선언의 `derived: true` 와 간선 표시를 **양방향으로** 대조한다.

##### 물질화하지 않는 것도 선언에 있다

```yaml
PART_OF:   { transitive: true, materialize: false, acyclic: true }
PARENT_OF: { 같음 }
IMPLEMENTS:{ 같음 }
```

`PART_OF` 는 한 칸 938개인데 **2홉 이상 경로가 2,070개**다. 저장하면 3배가 되고 조직 개편마다 재계산해야 한다. 검색은 그 없이도 `PART_OF*` 로 이미 잘 돈다(`context.py` 3곳 · `region.py` 1곳).

`materialize: false` 는 "안 한다"가 아니라 **"일부러 안 한다"는 선언**이다. 없으면 다음 사람이 다시 만든다. `validate/06` 이 "저장 안 하기로 한 폐포가 저장돼 있나"를 검사한다.

##### 선언으로 옮기니 규칙의 빈틈이 드러났다

손으로 쓴 Cypher 와 선언에서 만든 Cypher 가 **18건에서 다른 답**을 냈다. 원인은 구현이 아니라 규칙이었다 — 4개 구보건소가 전부 거리 0·직속기관이라 "가장 가까운 것 하나"로는 답이 안 정해진다(동점 21건). `tie_break: org_id` 를 명시했다.

##### 없는 것

```
규칙 종류      chain 하나. aggregate · transitive 물질화 없음
고정점 반복    없음. 규칙 둘이 원천만 읽어 두 번째 바퀴가 0건
OWL/RDF       아님. 속성그래프다
```

셋 다 **소비자가 없어서 안 만들었다.** 집계는 실측 기각(도메인 트리가 시민 질문과 안 맞는다), 전이 물질화는 손해, 고정점은 쓸 규칙이 없다.

**"추론기"가 아니라 "규칙 적용기"가 정확한 이름이다.**

---

## 7. 스키마 대조 — 선언에서 판정 139건

```
라벨 14   × (존재 · :Entity · 제약)          = 42
관계 18   × (존재 · 짝 · 파생 표시)           = 54
카디널리티 11 · 순환 2 · 어휘 밖 값 6 · 결정 대상 6 · 미사용 관계 18
                                        합계 139건
```

##### 판정 기준

```
FAIL   그래프에 있는데 선언에 없다        오타 관계·라벨
FAIL   짝이 선언과 다르다               HANDLED_BY 가 CivilService->Employee
FAIL   어휘 밖 값                      Channel.name 에 어휘에 없는 값
FAIL   파생 표시 불일치 · 순환 · 전이 폐포 저장 · 필수 속성 빔
WARN   카디널리티 위반 · 결정 대상 없음 · 유일성 제약 없음 · 선언에만 있음
참고    아무도 안 읽는 관계 (지금 3종 984개)
```

##### 제약이 선언에서 생성된다

```python
for label, spec in schema.labels().items():
    CREATE CONSTRAINT ... FOR (n:{label}) REQUIRE n.{spec['key']} IS UNIQUE
```

기존 이름을 그대로 재현해야 `IF NOT EXISTS` 가 먹으므로 `LEGACY_NAMES` 로 14개를 매핑했다.

##### 검사 비용

```
전체 1.61초(첫 실행) / 0.05초(캐시 후)
속성 NULL 스캔   10노드 0.008초 · 7,647노드 0.007초   노드 수와 거의 무관
관계 짝         간선당 1.7µs                       31만 간선이어도 0.5초
순환 탐색       경로 수에 비례                      깊이를 선언에서 가져온다
```

순환 검사 깊이는 추론 선언의 `depth` 를 읽는다(`PART_OF` 6 · 나머지 8). `*1..12` 는 근거 없는 숫자였다.

---

## 8. 검색 — 관문 일곱

```
질문
 ├─ 0.5 지역 감지        구 이름을 그래프에서 읽는다
 ├─ 구조 라우터           TRIGGER_WORDS 18 · 별칭 118 (전부 선언)
 │    목록 질의면 -> 목록 페이지 제외 · 이름으로 중복 접기
 ├─ 벡터 + 어휘 + 구조 5표 -> RRF (k=60)
 ├─ 자리 보장             어휘 1 · 게시판 2 · 지역 3 · 게시글 3
 ├─ 컨텍스트 블록 조립
 │    page_kind == 목록  -> 구비서류·처리장소·절차·수수료를 감춘다
 │    벡터 최고점 < 0.74  -> 같은 처리 (그래프에 그 절차가 없다)
 ├─ 첨부 블록 (자리 보장)
 └─ 답변 생성
```

##### 검색 어휘 165항목이 선언에 있다

```
term_aliases 60 · target_aliases 33 · trigger_words 18 · channel_aliases 17
attach_triggers 15 · org_aliases 8 · civil_negation_words 5 · board_route_types 1
```

`UNIT_LEVELS` 는 `org_classes` 에서, 구 이름은 그래프에서 나온다. 박아두면 조직 분류나 행정구역이 바뀌었을 때 조용히 틀린다.

##### "없으면 없다고 한다"

```
"출생신고 서류 뭐 필요해"
  -> 후보 1위가 [출산 후 지원 · 목록]
  -> 구조화 필드 전부 감춤
  -> "출생신고에 필요한 구비서류는 수원시 안내에서 찾지 못했습니다"
  -> 출생신고서 서식 링크는 그대로 준다
```

**하나만 막으면 옆으로 샌다.** 구비서류를 막으니 처리 장소로(`주소지 관할 보건소`), 그것을 막으니 절차로(`동 행정복지센터에 신청`) 샜다. 결국 구조화 필드 전체를 한 덩어리로 가린다. 본문·담당부서·전화는 남긴다.

##### 점수 하한

```
구비서류 후보를 가진 질문의 벡터 최고 점수
  그래프에 없는 절차 6개   0.702 ~ 0.734
  ---------- DOC_TRUST_MIN 0.74 ----------
  gold 구비서류 질문       0.747 ~ 0.881
```

`page_kind` 는 "이 노드의 필드가 제 것이 아니다"라는 **구조적 사실**이고 하한은 "그래프에 그 절차가 아예 없다"는 **통계적 추정**이다. 전자가 우선한다. 여유 0.013 으로 얇아서 답변 축이 그것을 지킨다.

---

## 9. 품질 게이트 — gold 48문항 6축

```
top1       후보 1위가 맞나            17/17
relevant   후보 5개 중 몇 개가 정답     83/227
attach     첨부를 찾았나              4/4
places     장소가 지역으로 좁혀졌나     3/3
aggregate  집계 질문(채점 제외)
answer     생성된 답변 문장            6/6   ← 유일하게 답변을 본다
```

앞의 다섯은 **전부 후보만** 본다. `validate/04` 가 `make_answer=False` 로 부르니 검증 중에는 답변이 생성조차 안 됐다. 그래서 `출생신고` 가 정밀도 0/5 로 잡히면서도 오답이 나갔다 — "후보가 나쁘다"는 말했고 "나쁜 후보로 답을 지어냈다"는 못 말했다.

##### 답변 축

`answer_must` / `answer_not` 이 붙은 문항만 `make_answer=True` 로 한 번 더 돈다. 판정은 **문자열 포함으로만** 한다 — 의미 판정을 넣으면 판정기 자체가 흔들려 회귀 검사가 성립하지 않는다.

```json
"출생신고 서류 뭐 필요해": {
  "answer_not": ["관할 보건소", "진료비 영수증", "입·퇴원 확인서", "진단서"]
}
```

`answer_must` 항목의 `|` 는 "이 중 아무거나"다. `전입신고` 답변이 "거주지 관할 동에서 처리합니다"라고 옳게 말하는데 `행정복지센터` 라는 글자가 없어 실패로 잡힌 적이 있다.

##### 판정 기준

```
1위 적중 감소       FAIL
첨부·장소·답변 감소  FAIL
정밀도 3건 초과 하락 FAIL
gold_sha 변경       비교를 건너뛰고 이번 값을 새 기준으로 (자를 바꾼 것은 회귀가 아니다)
```

실패하면 `exit 1` 이고 **기준선을 갱신하지 않는다.**

---

## 10. 무회귀 확인 — `main.py --check`

```
산출물 순서 통과 (주의 3)
선언·코드 대조 통과 (주의 0)
스키마 대조 통과 (주의 0 · 참고 4)
    [dry-run] 새로 계산할 것 0개  <- 변화 없음   (OrgUnit)
    [dry-run] 새로 계산할 것 0개  <- 변화 없음   (CivilService)
    [dry-run] 새로 계산할 것 0개  <- 변화 없음   (Employee)
    [dry-run] 새로 계산할 것 0개  <- 변화 없음   (Article)
```

LLM 0회 · 그래프 쓰기 0회 · 약 1분.

`embedding_text` 가 노드에 저장돼 있어 **API 없이도 "몇 개가 달라졌나"를 셀 수 있다.** 임베딩 문장이 한 글자라도 달라지면 대상 수가 0이 아니게 된다 — 눈으로 대조할 필요가 없다.

```
build_text() 에 공백 한 칸 추가   ->  새로 계산할 것 4,834개
되돌림                          ->  0개
```

---

## 검증

```
파이프라인    33단계 · 26회 실행 · 전부 게이트 통과
지표         1위 17/17 · 정밀도 83/227 · 첨부 4/4 · 장소 3/3 · 답변 6/6
검사         스키마 대조 주의 0 · 선언·코드 대조 주의 0 · 산출물 순서 주의 3
값 대조       코드에서 선언으로 옮긴 상수 14건 전부 원본과 동일 (AST 로 읽어 비교)

선언을 고치면 무엇이 따라 움직이나 — 코드 수정 0줄
  labels 에 라벨 1줄 추가        제약 14 -> 15개
  relationships 카디널리티 수정   판정 139 -> 138건
  inference depth 0..5 -> 0..2  Cypher 가 PART_OF*0..2 로
  vocabulary.external_org +1줄   EXTERNAL_ORGS 14 -> 15
  article_type 에서 값 제거       FAIL "어휘 밖 값"
  PART_OF 의 acyclic 제거        [중단] "transitive 인데 acyclic 이 없다"
  schema.yaml 을 숨김            적재·검증·검색이 전부 FileNotFoundError

검사 발동 시험 — 인위적 위반으로 전부 확인
  카디널리티 · 순환 · 파생 표시 · 어휘 밖 값 · 필수 속성 · 전이 폐포 저장
  결정 형식 · 결정 대상 · 코드 상수 신설 · 요청 상한 · 범위 지문
```

---

## 재사용 가능한 판단

- **선언이 검사를 만들면 선언을 고칠 때 검사 수가 변한다.** 그것이 온톨로지 구동의 증거다. 말이 아니라 숫자로 확인된다
- **판단을 코드에서 선언으로 옮기면 판단의 빈틈이 드러난다.** 동점 처리가 없다는 것을 옮기기 전에는 아무도 몰랐다
- **노드가 자기 것이 아닌 값을 주장하면 읽는 쪽을 아무리 단속해도 소용없다.** 프롬프트를 네 번 고쳐 네 번 실패했다
- **하나만 막으면 옆으로 샌다.** 구비서류 -> 처리 장소 -> 절차
- **아무도 안 지키는 숫자는 감각이 아니라 오해다.** 넣은 지 몇 시간 만에 32개 중 6개가 틀어졌다
- **오탐이 쌓이면 검사가 무시당한다.** mtime 대신 범위 지문을 비교하도록 고친 이유다
- **기준선을 자동 갱신하면 그 검사가 무너진다.** `code_constants.json` 은 사람이 손으로 더한다
- **쓸 데가 확인 안 된 규칙 종류는 만들지 않는다.** 만들어도 검증할 방법이 없다
- **검증기는 *없는 것*을 검사하지 못한다.** 대신 선언과 실제를 대조하면 잡힌다

---

## 한계

- **답변 축이 6문항뿐이다**(48문항 중). 나머지 42문항의 답변은 아무도 안 본다
- **`DOC_TRUST_MIN = 0.74` 의 여유가 0.013 이다**(0.734 vs 0.747)
- **`page_kind` 임계 4는 실측 한 번으로 정했다**(임계 2 -> 127개 · 3 -> 49개 · 4 -> 26개)
- **`COLLECT[weekly]` 12단계·`[monthly]` 1단계를 한 번도 안 돌렸다.** `scraping/02~11` 이 같은 파일을 제자리에서 덧칠해 순서가 의미를 갖는데 확인이 안 됐다
- **`VISIT_FOR` 266 · `IN_DOMAIN` 644 · `PARENT_OF` 74 를 아무도 안 읽는다**(합 984)
- **규칙 종류가 `chain` 하나다.** 고정점 반복 없음. 제안요청서의 "추론 기반 검색" 요구 수준을 모르는 상태다
- **`observed` 구분이 없다.** 계층이 관측된 것인지 사람이 정한 것인지 선언에 없다
- **중간 산출물이 없다.** `BATCH_ID` 일관성 때문에 일부러 그렇게 했고, 전체가 20분이라 지금은 안 아프다. 데이터가 10배가 되면 아파진다
- **`resolve` 단계가 없어 `stage: source/invariant` 구분도 없다**
- **수집이 여전히 `www` 하나다.** `*.suwon.go.kr` 서브도메인 12개 미착수, 게시판 128개 중 20개
- **`article_type unknown` 이 0건이다.** 격리·승격 장치를 만들어 놓고 한 번도 안 돌았다
- **코드에 도메인 지식이 없다는 주장은 하지 않는다.** `scraping/` 5,230줄에 사이트 구조가 그대로 박혀 있고, 선언으로 덜어낸 것은 게시판 쪽뿐이다

---

## 관련 문서

- [[KG 수원-나라원 파이프라인 비교]] — 같은 철학이 다른 구조로 수렴한 기록
- [[KG_나라원 온톨로지 기반 KG 실습]] — 나라원 구현·운영
- [[KG 수원 - 전체 파이프라인 1차구현]] — 09-04 상태. 이 문서 이전
- [[0907_KG - 수원 · 선언이 검사를 만들면 결함이 드러난다]] — 이 구조를 만든 날의 기록
- [[KG 범용 계획]]
