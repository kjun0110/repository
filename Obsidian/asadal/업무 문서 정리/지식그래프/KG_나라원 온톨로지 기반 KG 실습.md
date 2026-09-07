# 나라원 온톨로지 기반 KG 실습

nara1.kr 을 온톨로지 구동 지식그래프로 만들고 챗봇 API 까지 붙인 기록.
**수원시 홈페이지 생성형 AI 챗봇 사업의 리허설**이다. 여기서 겪은 것을 거기서 다시 겪지 않으려고 남긴다.

관련: [[KG_나라원 온톨로지 그래프 설계]]

---

## 요약

```
선언 1,008줄  ·  코드 3,600줄
노드 1,164  ·  간선 1,549 (파생 80)  ·  벡터 416  ·  검사 48
Neo4j 5 커뮤니티 (로컬 도커)  ·  text-embedding-3-small
```

```
crawler → parse → resolve → infer → validate → embed → load → serve
```

`schema/ontology.yaml` 하나가 파서·검증기·적재기·라우팅·LLM 프롬프트를 전부 구동한다.
지우면 크롤 말고는 아무것도 안 돈다. **문서가 아니라 실행 파일이다.**

---

## 1. 디렉토리 — 엔진과 프로젝트를 가른다

```
kg/          13파일 2,120줄   엔진. 도메인 지식 0. 수원에 그대로 간다
pipeline/     7파일 1,480줄   나라원 전용. 수원은 새로 쓴다
schema/       ontology.yaml   선언 하나
resolutions/  사람 결정 3개
```

| `kg/` | 하는 일 |
|---|---|
| `ontology.py` | 선언 로드·조회·id 생성·훅 로딩 |
| `shapes.py` | 페이지 생김새 8가지 (표·카드·산문·정의목록·이미지목록 등) |
| `clean.py` | 문자열 정리·형변환 |
| `korean.py` | 조사 처리 (은/는, 이/가, 으로/로) |
| `decisions.py` | 승인 판정 · 검토 큐 |
| `freshness.py` | 산출물 순서 검사 |
| `infer.py` | 규칙 해석기 (aggregate · chain · transitive) |
| `schema_card.py` | 선언 → LLM 카드 · Cypher 검증·자동수정 |
| `embed.py` `load.py` `serve.py` | 임베딩 · 적재 · API |

엔진이 프로젝트 코드를 부르는 지점은 선언이 정한다.

```yaml
hooks:
  embed_text: pipeline.embed_text    # 노드 → 임베딩 문장
transforms:
  normalize_org_name: {...}          # 없으면 parse 가 시작을 거부한다
```

---

## 2. 온톨로지에 무엇이 들어 있나

```
라벨 16 · 관계 17 · 규칙 3 · 추출기 14
어휘 81 (닫힌 어휘 값 51 + 라벨 동의어 30)
주의사항 26
```

### 라벨 (도메인 11 + 아카이브 5)

```
Project 206 · Organization 81 · Domain 3 · OrgType 8 · Offering 16
Company 1 · Office 4 · HistoryEvent 27 · Certificate 20
Department 5 · Team 28          ← 조직도가 세로쓰기라 미구현. 노드 0
Resource · Chunk · Notice · Photo · ExternalSite   ← 출처 추적용
```

### 닫힌 어휘 — 여기가 진짜 "어휘"

```yaml
local_gov:
  ko: 지방자치단체
  synonyms: [지자체, 시청, 도청, 구청, 군청]      # 질문 → 코드 (라우팅)
  scheme_codes: {board: "a local government"}    # 사이트 표기 → 코드 (파싱)
```

**어휘가 두 번, 서로 다른 얼굴로 쓰인다.** `scheme_codes` 는 파싱할 때, `synonyms` 는 답할 때.

### 관계 — 검사가 여기서 나온다

```yaml
ORDERED_BY: {from: Project, to: Organization, cardinality: "N:1"}
```

이 세 줄에서 검사 4개가 자동 생성된다 — domain 위반 · range 위반 · 끊어진 끝점 · N:1 위반.
17개 관계 × 3~4 = 검사 48개의 대부분.

### 계층

```
public_sector ⊃ 중앙행정기관 · 지자체 · 공공기관 · 교육 · 입법부
nonprofit     ⊃ 재단
System        ⊃ 통합정보시스템
```

`-[:BROADER*0..]->` 로 탄다. **`public_sector` 가 직접 붙은 사업 0건, 전이로 세면 180건.**

### 주의사항 26개 — LLM 이 읽는 자리

```
"source_no 는 목록 필터마다 재부여되는 표시용 순번이다. 조인에 쓰면 데이터가 뒤섞인다."
"계약금액·계약기간·수행역할은 원본에 아예 없다."
"한 사업이 분야 2~3개에 걸칠 수 있다(51건). 분야로 세면 합이 206을 넘는다."
"기관유형은 Organization 이 아니라 Project 에 붙는다. 충돌 14곳."
```

**이게 온톨로지의 실제 값이다.** 라벨과 관계만으론 스키마일 뿐이고,
이 26개가 스키마 카드로 LLM 에게 가서 틀린 Cypher 를 막는다.

---

## 3. 단계별 상세

### crawler
온톨로지 안 쓴다. sha256 증분이라 바뀐 것만 다시 받는다.

### parse — 선언 해석
```yaml
extract:       어느 페이지를 어떤 shape 으로, 몇 번째 칸이 무엇
transforms:    값 변환 훅
id_policy:     prj:{sha1(name|contract_date|org_name)[:12]}
scheme_codes:  사이트 표기 → 우리 코드
```
자연키가 없어 대리키를 쓴다. **`clean` 층이 해시 앞에 와야 한다** — 한 글자만 달라도 노드가 갈린다.

### resolve — 사람 결정
```
입력 검사   same_as 대상이 있나 · 못 쓰겠으면 멈춘다
적용        이름 복원 17 · 병합 1
name_site   사이트 표기를 남긴다
```

`name_site` 가 필요한 이유: 결정 파일들이 **복원 전 이름**으로 키를 잡는다(14건 중 5건이 절단 이름).
뒤 단계가 복원된 이름만 봐서는 못 찾는다.

### infer — 규칙 적용

```yaml
rules:
  org_type_rollup:
    kind: aggregate
    materialize: true
    subject: Organization
    via: [{edge: ORDERED_BY, direction: in}, {edge: FOR_ORG_TYPE, direction: out}]
    pick: majority
    on_tie: org-types.yaml
    produce: {edge: HAS_ORG_TYPE, property: org_type,
              conflict_property: org_type_conflicting}

  subsumption:
    kind: transitive
    materialize: false     # 조회 때 BROADER*0.. 로 탄다

  org_type_via_org:
    kind: chain
    enabled: false         # 수원용 견본
```

**`materialize` 가 실행 위치를 정한다.**

| | 적재 시 (true) | 조회 시 (false) |
|---|---|---|
| 계산 | 한 번 | 매 질문 |
| 맞는 것 | 비싼 집계 · 긴 연쇄 | 가변길이 전이 |
| 근거 | 전체를 훑어 세야 함 | Cypher 가 `*0..` 을 네이티브로 함 |

만들어진 간선에는 `derived: true` + `rule` 이름이 붙는다.

### validate — 2단계 게이트

```
1단계 graph/     실측 계약 + 불변식        47건
2단계 inferred/  불변식만 (파생 포함)      48건
```

`infer` 가 `validate` **앞**이어야 한다 — 규칙이 카디널리티를 깨뜨릴 수 있다.

### embed → load → serve

---

## 4. 조회 경로

```
질문
 ├─ 고정 답변 (원본에 없는 항목)          LLM 안 부름
 ├─ 라우팅   AGG 정규식 + 어휘 81개
 ├─ 구조적 → Text2Cypher (최대 3회)
 │    카드 → LLM → repair(방향) → check(스키마) → 실행
 │      행 있음 → 근거
 │      0건    → 근거            ← "없다"의 근거
 │      NO_QUERY + 어휘가 라벨 지목 → 라벨 통째로 (≤30개)
 ├─ 벡터 검색 (항상)              상대점수 필터
 ├─ mask → 답변 생성 → scrub      절단 이름 유출 차단
 └─ 로그
```

---

## 5. 수원에 가져갈 것 — 이 문서의 핵심

### ① 계약을 두 종류로 가른다

```yaml
stage: source      사이트를 세어본 값. 결정을 반영하면 달라지는 게 정상
stage: invariant   결정을 반영해도 깨지면 안 되는 것
```

섞으면 결정 이후를 **검사할 수 없다.** `Organization 81개` 가 병합 후 80이 되어 항상 실패하므로,
그것 때문에 불변식까지 게이트를 못 건다. **사람이 결정을 덧씌우는 단계가 있는 한 어느 도메인에서나 생긴다.**

### ② 검증은 두 층

```
결정 파일 무결성   resolve/infer 가 자기 입력을 본다   오타 · 없는 코드
그래프 불변식      validate 2단계가 결과를 본다        끊긴 간선 · 카디널리티 · 열거값
```

게이트만으론 부족하다. 결정이 **잘못 적용된 것**은 게이트가 잡지만,
**아예 적용 안 된 것**은 결과가 멀쩡해서 못 잡는다.

### ③ 조용히 실패하지 않는다 — 하루에 네 번 나왔다

```
resolve   same_as 대상 없으면 조용히 병합 안 함
serve     Cypher 0건을 버려 '없다'의 근거가 사라짐
load      group_edges 가 간선 속성을 통째로 버림
infer     모르는 코드면 조용히 간선 안 만듦
```

전부 **"실패했는데 결과는 멀쩡해 보이는"** 형태다. 검증기가 못 잡는다 — 없는 것은 검사할 수 없다.
**못 하겠으면 말한다.** 조용히 넘기면 데이터가 아니라 신뢰가 깨진다.

### ④ 산출물 순서를 검사한다

단계가 늘수록 빠뜨리기 쉽다. `parse` 만 돌리고 `load` 하면 **옛 그래프가 경고 없이 들어간다.**

```
✘ 산출물이 순서대로 새것이 아니다.
    resolved/ 가 뒤졌다  →  python -m pipeline resolve
```

### ⑤ 파생 간선에 출처를 붙인다

`derived: true` + `rule` 이름이 있으면 `materialize: true` 를 써도 된다.
없으면 "사이트에 있던 것"과 "만든 것"이 섞여 **대외 챗봇에서 출처 추적이 끊긴다.**

원래는 `materialize: false` 정책으로 추론을 아예 안 했는데, 표시를 달아 해결했다.

### ⑥ 라벨 동의어는 좁게

`사업` 하나 넣으면 모든 질문이 구조적이 된다 (`Project.label_ko` 가 `사업`).
`Certificate` 에 `인증` 을 넣으면 본문에만 있는 답(누적 399건)을 그래프가 가로채
홈페이지 분야 사업 수(124)를 세는 오답을 낸다.

`회사` 로 묶어야 갈린다:
- `무슨 사업 하는 **회사**예요?` → Offering (사업분야 4개)
- `어떤 사업 **하셨어요**?` → Project (개별 사업)

### ⑦ 프롬프트로 설득하지 말고 답을 쥐여준다

`자본금 얼마예요?` 가 `NO_QUERY` 였다. 카드에 `capital: '9억원'` 이 있고
프롬프트에 `자본금 → Company.capital` 이라고 적어뒀는데도.
**카드 전체의 "돈은 없다" 신호(계약금액·매출)에 눌린 것.** `Company` 만 남긴 최소 카드로는 정상 동작했다.

프롬프트 보강은 부작용이 났다(Cypher 가 `MATCH` 없이 나오고, `계약금액` 이 `NO_QUERY` 대신 산문).
**라벨을 통째로 근거로 주는 폴백**이 해결했다.

### ⑧ 0건도 근거다

`화성에 지사 있나요?` 에 같은 근거로 3번 물어 3번 다른 답이 나왔고 한 번은 사실과 달랐다.
원인은 `if rows:` — Cypher 가 0건이면 근거에 아무것도 안 넣어서
**"화성에 사무소 없음"이 LLM 에게 전달되지 않았다.** 남은 벡터 근거(화성시가 발주처)만 보고 "있다"고 답했다.

---

## 6. 수원 대조

수원 제안요청서(2026.4) 요구사항과의 대조.

| 요구사항 | 상태 |
|---|---|
| RAG 를 챗봇 UI 와 분리 | ✅ stateless API |
| 향후 API 제공 고려 | ✅ `POST /ask` |
| Vector DB + 지식그래프 구조화 | ✅ 벡터 416 · Neo4j |
| 의미·키워드 결합 검색 | ✅ 벡터 + Text2Cypher |
| 복합 조건 질의 | ✅ 그래프 순회 |
| 환각 방지 가드레일 | ✅ 고정답변 · 스키마 대조 · 0건 근거 |
| 근거 링크 명시 | ✅ `sources` |
| 시맨틱 캐싱 | ❌ |
| Re-ranking | △ 상대점수 필터만 |
| 최신 자료 우선 | ❌ |
| 표·이미지 내부 검색 | ❌ 인증서는 손으로 읽음 |
| 추론 기반 검색 | △ 규칙 3개 |
| 모니터링 대시보드 | ❌ 로그 JSONL |

못 만든 다섯은 **나라원 규모(노드 1,164)로는 연습이 안 된다.**
수원은 민원 644 · 서류 609 · 법령 192 · 조직 940 · 직원 5,136 · 게시글 6,282.

### 수원에서 필요해질 것

`suwon_graph` 에 이미 `derived: true` 간선이 둘 있다 — 추론이 이미 필요해서 손으로 만들어놨다는 뜻.

```
IN_CHARGE_BUREAU  583개   민원 -HANDLED_BY-> 과 -PART_OF-> 국   (연쇄 + 전이)
VISIT_FOR         266개
```

4홉짜리 질문이 나온다.

```
"소파 버리려면 얼마예요?"
  WasteItem -DISPOSAL_FEE_IN-> CivilService -HANDLED_BY-> OrgUnit -PART_OF*-> 구
            └── 연쇄 ──────────────────────┘  └─ 전이 (깊이 가변) ─┘
```

지름길 간선을 미리 만들어두면 LLM 은 **한 홉만 짜면 된다.**
한 홉짜리에서도 방향을 뒤집고 있는 것을 없다고 거절했다. 4홉이면 훨씬 나빠진다.

---

## 7. 아직 안 된 것 — 정직하게

**`parse.py` 결합 27** — 추출기 이름(`board.cause04`)이 코드에 박혀 있다.
일부러 `pipeline/` 에 뒀다. `kg/` 로 옮기면 "엔진"이라는 이름표가 거짓말이 된다.

```
kg/load.py               1    BROADER (계층 간선 고정 이름. 결합 아님)
pipeline/validate.py     1
pipeline/resolve.py      3
pipeline/embed_text.py  12    문장 구조가 도메인 고유. 훅이라 정상
pipeline/parse.py       27    ← 남은 숙제
```

**`chain` 가변 깊이 없음** — `via` 에 `depth` 를 못 받는다. 수원 `PART_OF` 가 그 모양이라 거기서 필요해진다.

**추론기가 아니라 규칙 적용기** — 고정점까지 반복하지 않는다. 규칙 3개에 반복이 필요 없어서.

**"도메인 무관"은 아직 가설** — 도메인이 하나뿐이라 증명 안 됐다. **수원이 첫 시험이다.**

**미승인 5건** — 강남구·과기정통부·한국정보통…·서울시교육청 서…·위버시스템.
사이트 안에 근거가 없다. 나라원 담당자에게 물어야 한다. 지금은 마스킹돼서 답변이 깨지진 않는다.

**동률 1건** — 과학기술정보통신부 `{public_inst: 1, gov_central: 1}`.
산하기관이 둘(중앙전파관리소·국립과천과학관)인데 사이트가 한 기관으로 묶어놨다. 노드를 쪼개야 할 수도 있다.

---

## 8. 다시 돌릴 때

```bash
python -m pipeline parse      # 스키마만 고쳤다
python -m pipeline resolve    # 결정만 채웠다
python -m pipeline infer      # 규칙만 고쳤다
python -m pipeline validate   # 두 단계 게이트
python -m pipeline embed --run
python -m pipeline load
python -m pipeline serve
```

앞 단계를 돌리고 뒤를 빠뜨리면 `freshness` 가 멈춘다.

### 무회귀 확인 — LLM 을 안 부르므로 공짜다

```bash
python -m pipeline embed          # "새로 계산 0" 이어야 한다 (문장 sha1 비교)
python -m pipeline validate       # 47 / 48건 전부 통과
python -m pipeline serve --card   # 카드가 바이트 단위로 같아야 한다
```

임베딩 문장이 한 글자라도 달라지면 `새로 계산` 이 0 이 아니게 된다. **눈으로 볼 필요가 없다.**
