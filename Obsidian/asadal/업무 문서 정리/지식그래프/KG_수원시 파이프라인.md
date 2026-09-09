> 2026-09-09 기준 **구조 지도**. "어디에 무엇이 있고 어떻게 흐르나"만 담는다.
> 각 층의 상세(검사 139건 · 사람 결정 251건 · 검색 관문 일곱)는 [[KG_수원 선언 구동 파이프라인]] 에 있다.

## 한 장 요약

```
              ontology/schema.yaml · decisions/*.yaml · sites/*.yaml
                            (선언 — 계약서)
                                  │  24개 파일이 읽는다
        ┌──────────┬──────────┬───┴───┬──────────┬──────────┐
        ↓          ↓          ↓       ↓          ↓          ↓
      수집기      적재기     검사기   추론기     임베딩     검색기
   (어휘 검증) (제약 생성)  (대조)  (Cypher 생성) (범위)   (라우팅)
        └──────────┴──────────┴───┬───┴──────────┴──────────┘
                                  ↓
                                Neo4j
                     노드 24,708 · 간선 31,417 · 벡터 10,743
```

```
① 수집   COLLECT · 파이프라인 밖    외부 요청 + LLM. 비싸고 멱등하지 않다
              ↓ data/raw/*.json
② 적재   STEPS 35단계             JSON → Neo4j. 매번 전량, 한 배치(BATCH_ID)
              ↓ 그래프
③ 검색   search/ · api/ · bridge/  질의 시점. 파이프라인 밖
```

---

## 온톨로지는 단계가 아니다 — 계약서다

`ontology/schema.yaml` 은 `STEPS` 어디에도 없다. 대신 **24개 파일이 읽는다** — 수집기·적재기·검사기·검색기 전부.

핵심 논리는 선언 파일 첫머리에 있다.

> Neo4j Community Edition 은 Domain/Range·Disjoint·클래스 계층을 강제하지 못한다.
> 그래서 이 선언은 **DB 가 아니라 적재기가 강제한다.**
> 파이프라인을 거치지 않고 Cypher 로 직접 쓰면 규칙이 무시된다.

**선언은 아무 힘이 없고, 읽는 코드가 힘을 준다.** 그래서 "누가 읽는가"를 세는 것이 곧 "이 선언이 살아 있는가"다 — 2026-09-08 에 그렇게 세어 죽은 선언 넷을 찾았고(`classes` 절 · `entity_labels()` · `slots.default` · `validate/02`), 09-09 에 넷을 더 지웠다(`fee_range` · `org_tokens` · `bureau_levels()` · `VISIT_FOR`).

##### 관계 하나가 다섯 군데에서 일한다

```yaml
HANDLED_BY:
  ko: 담당부서
  desc: 시민이 실제로 가는 창구. 국·실은 결재 라인이라 못 온다
  from: CivilService
  to: OrgUnit
  range: { unit_level: 실무조직 }
  cardinality: "N:M"
  read_by: [search, bridge]
```

```
적재 전   schema.py 가 로드하며 검산한다
          from/to 가 실재 라벨인가 · range 속성이 그 라벨에 있나 · ko/desc 가 있나
          어긋나면 파이프라인이 시작도 못 한다

적재 중   relationships/03 이 실제로 간선 700개를 건다
          (이 단계는 선언을 안 읽는다 — 값을 만드는 방법이라 코드에 있다)

적재 후   validate/01 이 range 를 읽어 "국·실이 담당부서로 걸렸나" Cypher 를 만든다
          validate/06 이 짝(CivilService→OrgUnit)·카디널리티를 그래프와 대조한다

추론      relationships/05 가 inference 선언대로 HANDLED_BY 를 타고
          IN_CHARGE_BUREAU 583개를 합성한다

검색      read_by 는 "누가 읽는가"의 기록이다. 비어 있으면 validate/06 이
          매 실행 "적재만 하고 아무도 안 읽는다"고 띄운다
```

**선언 한 줄을 고치면 검사 수·제약 수·질의 모양이 바뀐다.** 09-08 에 음성 테스트로 여섯 번 확인했다 — `range` 를 뒤집으니 위반 0 → 631, `classes` 의 `requires` 를 비우니 7 → 0.

##### 선언은 세 파일로 갈려 있다

```
ontology/schema.yaml   라벨 14 · 관계 17 · 어휘 · 요건 · 속성 · 추론 규칙
decisions/*.yaml       사람이 개별 건을 판정한 것. why · by 필수 (6파일)
sites/*.yaml           수집 대상. 사이트마다 한 장 (suwon · health · jangan)
                       → 원천 파일 목록도 여기서 나온다 (config.article_sources)
```

가르는 기준은 하나다.

```
그래프에 값으로 나타나면        ontology/schema.yaml
사람이 개별 건을 판정한 것이면   decisions/*.yaml
값을 만들어 내는 방법이면       코드
```

09-09 에 코드에서 선언으로 옮긴 것: 어느 라벨을 폐지 판정하나(`labels[].lifecycle`) · `unit_level` 이 어느 어휘를 쓰나(`slots`) · 임베딩 배치·글자 상한(`config`).

---

## ① 수집 — 파이프라인 밖

비용 때문에 밖에 뒀다(외부 요청 4개 + LLM 5개). 대신 "수집 안 하고 적재"를 `validate/07` 이 막는다.

```
daily     scraping/12       게시판 20개 → articles.json            11.3 MB
weekly    scraping/00       조직도 → org_chart · org_teams · org_live
          scraping/01       민원 647p → civil_service_raw + html/ 647개
          02~11             LLM 5 + 규칙 4로 civil_service.json 을 제자리에서 덧칠
                            순서가 의미를 갖는다 — 02 가 만든 위에 03~11 이 붙는다
monthly   scraping/13       업무편람 39 MB (아직 안 씀)

서브도메인  scraping/14 <사이트>  선언 한 장이면 사이트가 추가된다
```

##### 서브도메인 — 확장은 이 층에만 붙는다

2026-09-08 에 둘(health · jangan)을 붙이며 확인했다.

```
사이트마다 다르다   목록 URL · 상세 URL · 본문 selector · 첨부 URL · 응답 속도
선언 안 한다       칸 순서 ← 표 머리(thead) 이름으로 읽으면 저절로 흡수된다
같다              산출물 JSON 모양부터 아래로 전부
                  → nodes/16 과 validate/02 가 sites/*.yaml 에서 원천 목록을 받는다
```

**사이트 하나 = 선언 한 장(20줄).** 구청 사이트는 작성자가 실제 부서명이고 출처가 구를 특정해서, www 게시글에서 못 가르던 동명 부서를 가른다.

---

## ② 적재 — `STEPS` 35단계

단계 수는 온톨로지 크기를 따라간다 — **라벨 14 + 관계 17 = 31, 단계 35.** 거의 1:1이다.

```
── 멈출 수 있을 때 멈춘다 ───────────────────────────────────
 1  validate/07_freshness      원천이 선언보다 오래됐나 (범위 지문)
 2  validate/08_declared       선언 ↔ 코드가 두 벌인가 (AST)
 3  schema/01_constraints      제약 14 · 풀텍스트 4 · 벡터 1  ← labels[].key 에서 생성

── 조직 뼈대 ────────────────────────────────────────────
 4  nodes/01_org_units         OrgUnit 940 + PART_OF
 5  nodes/02_employees         Employee 5,136
 6  nodes/03_legal_dongs       동에 법정동 별칭
 7  validate/00_consistency    원천 JSON 모순 — 노드 만들기 전에 막는다

── 민원과 부속 ──────────────────────────────────────────
 8  nodes/04_civil_service     CivilService 644 + page_kind 판정
 9  nodes/05_document_types    DocumentType 609
10  nodes/06_regulations       Regulation 192 + BASED_ON          needs CivilService
11  nodes/07_channels          Channel 10 + AVAILABLE_VIA         needs CivilService
12  nodes/08_target_groups     TargetGroup 12 + FOR_TARGET        needs CivilService
13  nodes/10_waste_items       WasteItem 203 + DISPOSAL_FEE_IN    needs CivilService
14  nodes/11_places            Place 2,509 + LISTED_IN            needs CivilService
15  nodes/12_domains           Domain 76 + IN_DOMAIN · PARENT_OF  needs CivilService

── 게시글 ──────────────────────────────────────────────
16  validate/02_articles       id 중복 · file 충돌 · 날짜 (원천 3개 대조)
17  nodes/16_articles          Board 23 · Article 6,292 · Attachment   needs OrgUnit
                               담당부서 3단계: 이름 유일 → 전화 → 구로 좁히기
18  nodes/17_body_files        민원 본문 파일 402                      needs CivilService

── 관계 (사이에 노드 보강이 낀다) ────────────────────────
19  relationships/01_part_of   PART_OF 938 + peer_count 심기      needs OrgUnit
20  nodes/09_org_address       주소 상속 · '구청 2층' 해소         needs PART_OF
21  relationships/02_works_at  WORKS_AT 5,136                    needs Employee, OrgUnit
22  nodes/13_refresh_tasks     업무 변경                          needs WORKS_AT
23  nodes/14_sync_employees    입사 · 퇴직                        needs WORKS_AT
24  relationships/03_handled_by  HANDLED_BY 700                  needs CivilService, OrgUnit
25  relationships/04_requires    REQUIRES 890                    needs CivilService, DocumentType
26  nodes/15_issuers           Issuer 12 + ISSUED_AT             needs DocumentType

── 검사와 추론 ─────────────────────────────────────────
27  validate/01_requirements   요건 · 자기모순 → violations 표시   needs HANDLED_BY
28  relationships/05_inferred  IN_CHARGE_BUREAU 583 (VISIT_FOR 는 09-09 삭제)
                               inference 선언 → Cypher 생성       needs HANDLED_BY, REQUIRES, PART_OF
29  validate/06_schema         선언 ↔ 그래프 14종 대조
30  lifecycle/01_mark_stale    폐지 판정 · labels[].lifecycle 셋    needs OrgUnit, Employee
                               (Article 은 일부러 제외) · :Entity 도 뗀다

── 임베딩 → 게이트 ─────────────────────────────────────
31  embedding/01  OrgUnit 190                                    needs OrgUnit
32  embedding/02  CivilService 644                               needs CivilService
33  embedding/03  Employee 5,038                                 needs WORKS_AT
34  embedding/04  Article (검색 대상 4종 · 날짜 창)                needs IN_BOARD
    (넷 다 schema.check_entity 로 시작 — entity: false 면 API 를 쓰기 전에 멈춘다)
35  validate/05_gate  gold 48문항 재고 나빠지면 exit 1
```

##### 순서 제약도 선언이다

`needs` 는 그 단계가 돌기 전에 그래프에 있어야 하는 라벨·관계다. 35단계 중 **23개**에 붙어 있고, `run_step` 이 실행 전에 센다.

2026-09-08 이전에는 이 제약이 **주석에만** 있었다. 그래서 순서를 바꾸면 멈추지 않고 **조용히 덜 적재됐다** — `nodes/17` 을 `nodes/04` 앞에 두면 `MATCH (c:CivilService)` 가 0건 매치해서 본문 파일 0개를 넣고 통과한다.

##### 실행 모드 셋

```
python main.py                   적재 35단계 (약 20분)
python main.py --collect=daily   게시글 수집 후 적재
python main.py --check           LLM·쓰기 없이 "값이 안 변했나"만 (약 1분)
```

---

## ③ 결과 — 지금 그래프

```
노드                                    관계
Attachment  8,050  Article  6,292       HAS_FILE   8,050   IN_BOARD   6,292
Employee    5,136  Place    2,509       WORKS_AT   5,136   POSTED_BY  4,155
OrgUnit       940  CivilService 644     LISTED_IN  2,509   PART_OF      938
DocumentType  609  WasteItem    203     REQUIRES     890   HANDLED_BY   700
Regulation    192  Domain        76     IN_DOMAIN    644   IN_CHARGE_BUREAU 583
Board          23  TargetGroup   12     BASED_ON     402   AVAILABLE_VIA  376
Issuer         12  Channel       10     FOR_TARGET   271   DISPOSAL_FEE_IN 203
                                        ISSUED_AT    156   PARENT_OF     74
:Entity 10,743  ← 벡터 검색 입구         IMPLEMENTS    38   (VISIT_FOR 266 은 09-09 삭제)
```

##### 인덱스 다섯

```
entity_embedding       VECTOR    :Entity(embedding)            의미 검색
article_text_index     FULLTEXT  :Article(title, content)      어휘 검색
civil_text_index       FULLTEXT  :CivilService(name, content, tables_text, …)
employee_task_index    FULLTEXT  :Employee(assigned_task, position)
attachment_name_index  FULLTEXT  :Attachment(name)             서식 질문의 답
```

`:Entity` 가 붙는 라벨은 넷(Article · Employee · CivilService · OrgUnit)뿐이다. `Attachment` 8,050 과 `Place` 2,509 를 넣지 않는 이유는 **후보 자리를 먹어서**다 — 첨부는 `HAS_FILE` 로, 장소는 `LISTED_IN` 으로 따라간다.

`validate/06` 이 이 입구를 두 방향으로 지킨다(09-09): `:Entity` 인데 벡터가 없으면 주의, 폐지됐는데 `:Entity` 면 위반. 반대 방향("벡터 있으면 :Entity")은 **일부러 안 본다** — 범위 밖 게시글 510건이 벡터를 남긴 채 입구만 닫혀 있고, 그건 범위를 오갈 때 재임베딩을 피하려는 설계다.

##### pgvector 는 우리 것이 아니다

```
pgvector   Chatty(기존 RAG)의 저장소. 브리지가 근거만 주고받고 안 건드린다
우리 벡터   Neo4j 안에 있다 — :Entity.embedding + VECTOR 인덱스
```

pgvector 에서 원문을 가져오는 방안은 **보류**다. 우리가 페이지에서 뽑는 것은 텍스트가 아니라 **자리**이기 때문이다 — 담당부서가 어느 칸에, 구비서류가 어느 표에, 본문 파일 링크가 어느 `a` 태그에 있는지. pgvector 에 있는 것은 보통 청크 텍스트지 원본 HTML 이 아니다. 미확인 4가지(청크 재조립 무손실 · 표 생존 · 삭제 감지 · 페이지당 청크 수)를 재기 전에는 조건부다.

---

## ④ 검색 — 파이프라인 밖

```
질문
 ├─ 리더 숏컷            이름이 공개된 6명(시장·부시장·구청장). 제일 싼 경로
 ├─ 지역 감지            구·동(법정동 포함)을 그래프에서 읽는다
 ├─ 구조 라우터          "OO 목록/몇 개" 는 벡터가 아니라 unit_level 필터로
 ├─ 벡터 + 어휘 + 구조   척도가 다른 셋을 RRF(k=60)로 순위 융합
 ├─ 컨텍스트 조립        조상 경로 · 소속 직원 · 구비서류 · 처리 장소 · 첨부
 └─ 답변 생성            컨텍스트 근거로만. 없으면 모른다고 답한다
```

##### 안 하기로 한 것이 선언에 있다

```
page_kind == 목록      구비서류·처리장소·절차·수수료를 감춘다
                       (대문 페이지의 필드는 그 페이지 것이 아니다)
peer_count > 1         특정 지점·소관 기관을 지정하지 않는다
                       (구·동마다 있는 조직은 거주지 관할로 안내한다)
DOC_TRUST_MIN 0.74     그래프에 그 절차가 없으면 구조화 필드를 감춘다
```

---

## 현황 (2026-09-09)

```
미해결 목록   41건 중 23 닫힘 · 18 살아 있음
지표         1위 17/17 · 정밀도 83/227 · 첨부 4/4 · 장소 3/3 · 답변 7/7
검사         자체검사 · 01 · 02 · 06 · 07 · 08 · 게이트 전부 통과
```

##### 남은 18건은 전부 무언가를 기다린다

```
당신 결정 (3)     22 주간수집 검증 · 26 임계 재측정 · 1 게시판 확대
                  → 재수집 하는 날 하나로 묶인다
방아쇠 대기 (4)    27 규칙 종류 · 40 대법원 판단 · 35 문항 두 벌 · 39 죽은 선언 자동검출
습관 (2)          24 답변 읽기(7/48) · 25 DOC_TRUST_MIN 여유 0.013
규모 커지면 (3)    10 청킹 · 16 벡터 중복 · 17 HNSW
결론 남 (6)       12 · 21 · 23-b · 29 · 30 · 32
```

##### 이름을 정직하게 쓰면

```
온톨로지 기반 지식그래프    맞다. 선언이 실제로 집행하고 음성 테스트로 증명된다
선언 구동 파이프라인        맞다. 오히려 이것이 강점이다
추론 기반 검색             지금 수준으로는 과장이다
OWL 추론 · 추론 엔진        아니다
```

추론은 `chain` 1종(`IN_CHARGE_BUREAU`)뿐이다. 있던 `VISIT_FOR` 는 코드 소비처 0곳에 즉석계산과 266/266 완전 중복이라 09-09 에 지웠다. `IN_CHARGE_BUREAU` 도 답변에서는 괄호 한 조각이고 12문항 중 2번 나갔다. **있는 것도 거의 안 쓰인다** — 늘릴지는 제안요청서의 "추론 기반 검색" 이 뭘 뜻하는지 확인한 뒤에 정한다.

---

## 관련 문서

- [[KG_수원 선언 구동 파이프라인]] — 09-07 기준 상세. 검사 139건 · 사람 결정 251건 · 검색 관문 일곱
- [[KG 수원-나라원 파이프라인 비교]] — 같은 철학이 다른 구조로 수렴한 기록 · 교훈 목록
- [[0908_KG - 죽은 선언 넷과 뒤집힌 진단 열둘]] — 이 문서의 09-08 변경분이 나온 날
- [[0907_KG - 수원 · 선언이 검사를 만들면 결함이 드러난다]] — 선언이 검사를 만들게 한 날
- [[KG_나라원 온톨로지 기반 KG 실습]] — 나라원 구현·운영
