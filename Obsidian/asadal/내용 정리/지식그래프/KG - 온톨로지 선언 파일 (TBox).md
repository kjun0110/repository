## 개념

- **TBox** — 클래스·관계·규칙의 선언. "무엇이 무엇인가" ([[Ontology - T-BOX]])
- **ABox** — 실제 노드·관계 인스턴스
- **선언(declaration)** — 규칙을 코드가 아니라 데이터로 적어 둔 것. 적재기가 읽어서 집행한다
- **승격(promotion)** — 속성으로 받던 값을 노드·관계로 올리는 것
- **보류함(quarantine)** — 판정 실패를 그대로 남기는 자리. 승격 후보 목록이 된다

##### 이 문서가 다루는 범위

[[KG - 온톨로지 데이터 투입 액션]]의 A~F 단계가 쓰는 **규칙을 어디에 어떻게 적을 것인가**를 담는다. 파이프라인 자체는 그쪽 문서에 있다.

**2026-09-01 기준 아직 구현하지 않았다.** 규칙은 파이썬 7곳에 흩어져 있고 전부 정상 작동한다. 이 문서는 그것을 한 곳으로 옮기는 안이다.

---

## 선언 파일 (TBox) 예시

```yaml
# ontology/schema.yaml — 단일 진실
classes:
  CivilService:
    identity: [civil_id]
    subclasses:
      Procedure: { if: "content_type == 'procedure'" }
      Reference: { if: "content_type == 'reference'" }
    disjoint_with: [OrgUnit]

  Procedure:
    requires: [HANDLED_BY, REQUIRES]     # 없으면 D1 위반
  Reference:
    requires: [HANDLED_BY]               # 서류·절차 없는 게 정상

routing:                                  # B단계 매핑표
  - { signal: "url:www01-04/*", class: CivilService, layer: 1 }
  - { signal: "page_type:조직도",  class: OrgUnit,     layer: 1 }
  - { signal: "llm:content_type", class: [Procedure, Reference], layer: 3 }

relations:
  PART_OF:    { from: OrgUnit, to: OrgUnit, transitive: true }
  IMPLEMENTS: { from: Regulation, to: Regulation, transitive: true }
  HANDLED_BY: { from: CivilService, to: OrgUnit, cardinality: "1..n", inverse: HANDLES }
  REQUIRES:   { from: CivilService, to: DocumentType }
  ISSUED_AT:  { from: DocumentType, to: Place }

inferred:
  IN_CHARGE_BUREAU: HANDLED_BY / PART_OF+
  VISIT_FOR:        REQUIRES / ISSUED_AT

axioms:
  - name: 외부소관은_수원시조직에_연결금지
    forbid: "c.external_org IS NOT NULL AND (c)-[:HANDLED_BY]->(:OrgUnit)"
  - name: procedure는_절차가_있어야_한다
    warn: "c:Procedure AND c.procedure_text IS NULL"
```

`ontology/`의 기존 6모듈은 없애는 것이 아니라 **역할이 갈린다** — 문자열을 값으로 바꾸는 일은 그대로 파서로 두고, "그 값이 어느 클래스인지 / 무엇이 필수인지 / 무엇과 모순인지"만 선언으로 올린다.

---

## 선언 파일이 실제로 어떻게 도는가

##### 한 줄 — 선언은 "무엇", 코드는 "어떻게"

```
schema.yaml    Procedure 는 HANDLED_BY 가 있어야 한다      ← 무엇이 규칙인가
파이썬          관계 존재 여부를 Cypher 로 세는 코드          ← 어떻게 확인하는가
Neo4j          결과만 담긴다                              ← 라벨·violations·추론 관계
```

**YAML 은 검사하지 않는다.** 글자일 뿐이고, 읽어서 집행하는 것은 적재기다. 지금은 이 둘이 한 파일에 붙어 있다.

```python
# validate/01_requirements.py — '무엇'과 '어떻게'가 섞여 있다
RULES = [("NO_DEPT", "error", "담당부서 없음",
          "NOT (c)-[:HANDLED_BY]->() AND c.external_org IS NULL")]      # 무엇
graph.run(f"MATCH (c:CivilService) WHERE {where} SET c.violations = …")  # 어떻게
```

"무엇"을 선언으로 빼면 파이썬은 **클래스가 몇 개든 상관없는 범용 코드**가 된다. 새 유형이 와도 코드를 고치지 않는다.

##### 세 덩어리

```
 ┌─────────────────┐      ┌──────────────┐      ┌────────────┐
 │  schema.yaml    │ 읽음 │   적재기      │ 씀   │   Neo4j    │
 │  (TBox)         │ ───▶ │  (파이썬)     │ ───▶ │   (ABox)   │
 │  사람이 고침      │      │  선언대로 행동  │      │  결과만 담음 │
 └─────────────────┘      └──────────────┘      └────────────┘
```

**TBox 는 DB 에 들어가지 않는다.** Neo4j CE 가 이런 규칙을 강제하지 못하므로 적재기가 대신 지킨다. 그래서 **파이프라인을 거치지 않고 Cypher 로 직접 쓰면 규칙이 무시된다** — "적재는 반드시 파이프라인으로"가 전제다.

##### 어느 단계가 선언의 어느 절을 읽나

```
                                        schema.yaml 에서 읽는 것
──────────────────────────────────────────────────────────────
A. 수용
   원천 읽기                             —
   어휘 검사                             vocabulary
   모순 검사                             axioms
        ↓ 통과한 것만
──────────────────────────────────────────────────────────────
B. 판정
   클래스 결정                            classes.*.if
   계층 깊이 결정                          hierarchy.domain_depth
──────────────────────────────────────────────────────────────
C. 적재
   노드 MERGE                            classes.*.identity
   조직 라벨 부착                          org_classes
   관계 연결                              relations.*.from/to
──────────────────────────────────────────────────────────────
D. 검사   ← 관계가 다 붙은 뒤라야 가능
   요건 검사                              classes.*.requires
   면제 확인                              classes.*.exempt_if
──────────────────────────────────────────────────────────────
E. 파생
   추론 관계 생성                          inferred.*
──────────────────────────────────────────────────────────────
```

같은 파일을 여러 단계가 읽되 **각자 다른 절을 본다.** `main.py` 26단계 중 규칙을 읽는 자리는 7곳이다.

##### 실제 한 건이 통과하는 과정

원천(`등초본 발급`, 실제 레코드):

```
breadcrumb     [종합민원, 민원안내, 주민등록증, 등.초본 발급]
content_type   procedure
dept           혁신민원과
documents      []          ← 비어 있음
procedure_text None        ← 비어 있음
external_org   None
```

```
① 어휘 검사   content_type='procedure' 가 vocabulary 에 있나  → 통과
              (사이트가 'guide' 같은 새 값을 내보내면 여기서 멈춘다)

② 분류        classes 를 훑는다. Procedure.if 가 맞음        → :Procedure

③ 계층        breadcrumb 최상단 '종합민원' → domain_depth 2
              → [종합민원, 민원안내] 까지만 노드로

④ 적재        (:CivilService)-[:IN_DOMAIN]->(:Domain{종합민원 > 민원안내})
              (:CivilService)-[:HANDLED_BY]->(:OrgUnit{혁신민원과})

⑤ 요건 검사   Procedure.requires = [HANDLED_BY, REQUIRES]
              HANDLED_BY 있음 / REQUIRES 없음 / 면제조건 external_org 없음
              → 위반
```

그래프에 남는 것:

```
(:CivilService {name:'등초본 발급', violations:['PROC_EMPTY']})
```

**노드를 지우지 않는다.** 검색에는 그대로 나오고 "왜 불완전한지"만 기록된다.

##### 실패의 세 종류

```
거부(reject)      A에서 걸림      노드를 아예 만들지 않음
                  모르는 어휘·모순  "틀린 사실"이라 들어가면 안 된다

표시(violation)   D에서 걸림      노드는 만들고 딱지만 붙임
                  필수 항목 미달   "불완전"이지 틀린 것은 아니다

보류(quarantine)  B에서 못 정함    판정 실패를 그대로 남김
                  클래스 미상      나중에 규칙을 보탤 근거가 된다
```

**A는 막고 D는 표시한다.** 이 구분이 설계의 요점이다.

##### 순서를 못 바꾸는 이유

```
D가 C 뒤인 이유   "담당부서가 없다"를 판정하려면 HANDLED_BY 가 이미 붙어 있어야 한다.
                 앞에 두면 전부 위반으로 잡힌다.

A가 맨 앞인 이유   모순은 노드가 생기기 전에 막아야 의미가 있다.
                 그리고 A의 검사는 원천 레코드만 봐도 판정된다.
```

##### 지금 그 규칙들이 있는 곳

| 단계 | 지금 | 선언으로 가면 |
|---|---|---|
| 어휘 검사 | `validate/00.py` `CONTENT_TYPES` | `vocabulary` |
| 모순 검사 | `validate/00.py` `CHECKS` | `axioms` |
| 클래스 결정 | `content_type` 값을 그대로 사용 | `classes.*.if` |
| 계층 깊이 | `nodes/12.py` `SECTION_DEPTH` | `hierarchy` |
| 조직 부류 | `relationships/03` 의 `IN [...]` | `org_classes` |
| 요건 검사 | `validate/01.py` `RULES` | `classes.*.requires` |
| 추론 | `relationships/05.py` | `inferred` |

**①~⑤는 2026-09-01 기준 이미 전부 돌고 있다.** 규칙이 선언이 아니라 파이썬 안에 있을 뿐이다. 그래서 선언 파일로 옮기는 작업은 **동작을 하나도 바꾸지 않는다** — 회귀 검증이 "차이 0건"이어야 성공이다.

##### 그러면 언제 값어치가 나오나

```
지금   새 유형(게시글·공지·FAQ)이 오면  →  파이썬 7곳 중 어디를 고칠지 찾아야 한다
후     새 유형이 오면                  →  선언에 클래스 한 블록 추가
```

지금은 유형이 `procedure`·`reference` 2종뿐이라 선언할 것이 적다. **진짜로 필요해지는 시점은 수집 범위를 넓혀 새 유형이 들어올 때다.**

##### 2026-09-02 — 실제로 만들었다

`ontology/schema.yaml` + `schema.py`. 예상과 달리 **새 유형을 기다리지 않고도 값이 나왔다.**

```
org_classes             조직 부류 (총괄기관 7종 / 실무조직 10종)
bureau_levels           IN_CHARGE_BUREAU 가 가리킬 수 있는 레벨 7종
body_fallback_levels    본문에서 조직명을 찾을 때 후보로 삼을 레벨 6종
vocabulary              content_type · inquiry_labels · inquiry_end
classes                 Procedure/Reference 는 HANDLED_BY 가 있어야 한다 (external_org 면제)
hierarchy.domain_depth  구역별 분야 계층 깊이 (분야별정보 3 / 종합민원 2)
```

값이 난 자리는 **새 유형이 아니라 중복 제거와 검산**이었다.

```
죽은 코드가 드러났다   ROLLUP_EXCLUDE(classify.py) · FALLBACK_LEVELS(inquiry.py)
                    둘 다 사용처 0회였다. 한 곳에 모으니 보였다
중복 어휘가 합쳐졌다   INQUIRY_LABELS · INQUIRY_END 가 두 곳에 있었다
오타를 잡는다         bureau_levels 에 org_classes 밖의 값이 있으면 로딩 시점에 sys.exit
```

##### 선언과 그것이 만드는 라벨은 다르다

`org_classes` 로 `:총괄기관`·`:실무조직` 라벨까지 만들었다가 **라벨만 걷어냈다.**

```
라벨을 읽는 곳   없음
unit_level      같은 답을 내면서 "국만"·"구만"도 물을 수 있다. 인덱스도 있다
disjoint        선언은 강제하는데 적재기가 SET 만 하고 REMOVE 를 안 했다
```

**선언은 계속 일한다.** `schema.py` 가 그것을 unit_level 닫힌 어휘 대장으로 써서 `bureau_levels` 오타를 잡는다. 선언과 그 선언의 물질화(라벨)를 따로 판단해야 한다는 것이 여기서 나온 교훈이다.

---

## 새 클래스·관계를 언제 어떻게 늘리나

선언을 쓰기로 하면 곧바로 따라오는 질문 — **모든 관계가 미리 정의돼 있어야 하나? 적당한 것이 없으면 새로 추가하기 어려운가?**

##### 답 — 관계·라벨은 닫히고, 속성은 열린다

```
관계 타입    닫힘   질의가 이름을 알아야 쓸 수 있다
노드 라벨    닫힘   :Entity 를 붙일지·임베딩할지 사람이 정해야 한다
속성         열림   모르는 값이 와도 담아둘 수 있다
```

속성이 열려 있으므로 **모르는 것이 와도 파이프라인은 막히지 않는다.** 담아 두었다가 나중에 올린다.

##### 관계를 자동으로 못 늘리는 이유

질의가 이름을 모르면 아무도 쓰지 않는다. `:SUPPORTS_FOR` 같은 관계가 자동으로 생겨도 `search/` 가 그 이름을 모르면 조회할 코드가 없다 — **노드는 늘고 답은 그대로**다.

그리고 어휘가 갈라진다.

```
HANDLED_BY · MANAGED_BY · IN_CHARGE_OF · RESPONSIBLE_FOR
   → 같은 뜻인데 LLM 이 페이지마다 다르게 짓는다
   → "담당부서" 질의가 넷 중 하나만 잡는다
```

0828에 `Task` 노드 11,494개를 만들었다가 전량 기각한 적이 있다. 자동으로 늘렸다면 **그 판단 자체를 못 한다.**

##### 새 개념이 들어오는 길 — 3단계 승격

```
1단계  속성으로 받는다        새 정보는 일단 문자열 속성
         ↓ 값이 쌓이고 반복되면
2단계  닫힌 어휘인지 확인한다   값 종류가 몇 개인지 센다
         ↓ 공유되면
3단계  노드·관계로 승격한다    선언 추가 + 마이그레이션
```

##### 판단 기준은 하나 — 값이 공유되는가

```
여러 개체가 같은 값을 가리키고 그것으로 묶어 묻고 싶다   →  노드
개체 하나에만 붙고 조건 검색만 한다                     →  속성
```

`의료기관`은 서류 35종이 공유하므로 노드다. `수수료 600원`은 민원 하나만 가지므로 속성이다.

##### 이 프로젝트의 실제 선례 넷

| 대상 | 시작 | 판단 근거 | 결말 |
|---|---|---|---|
| `issued_at` | 문자열 속성 387건 | 12종 닫힌 어휘 | `:Issuer` + `ISSUED_AT` 승격 (2026-09-01) |
| `domain` | 문자열 속성 12종 | breadcrumb 에 계층 존재 | `:Domain` + `PARENT_OF` 승격 (2026-09-01) |
| 장소 92종 | 노드화 검토 | 공유가 1종뿐 | 속성 4종으로 유지 (0827) |
| 담당업무 | `:Task` 11,494개 | 90%가 조직 1곳 전용 | 전량 제거 (0828) |

**두 번은 올렸고 두 번은 안 올렸다.** 기준이 같아서 결론이 갈렸을 뿐이다.

##### 파이프라인이 막히지 않게 하는 자리

```
알려진 클래스   → 정상 적재
모르는 유형     → 보류함(quarantine). 거부가 아니다
모르는 필드     → extra 속성에 담아둔다
```

**보류함이 곧 승격 후보 목록이다.** 쌓이면 "이건 몇 건이나 되니 노드로 올릴까"를 데이터로 판단할 수 있다. 지금은 이 자리가 없어서 애매하면 어느 한쪽으로 밀어 넣어야 한다(`본문 없으면 보수적으로 reference`).

##### 자동 추가는 '제안'까지만

```
LLM 이 새 관계를 제안    →  선언이 아니라 제안 목록에 기록
사람이 검토             →  이름 통일 · 기존 관계와 중복 확인
승인되면 선언에 추가      →  그때부터 질의가 쓸 수 있다
```

**발견은 기계가 잘하고, 이름을 정하고 질의에 연결하는 것은 사람이 해야 한다.**

##### 비용

2026-09-01에 `Issuer`·`Domain`·`Article`·`Attachment` 네 클래스를 새로 세웠고 각각 반나절~하루였다. 선언 파일이 있으면 **그중 절반이 YAML 편집으로 줄지만, 올릴지 말지의 판단은 그대로 사람 몫이다.**

---

## 클래스 계층이 왜 필요한가

같은 그룹핑이 지금 코드 두 곳에 리스트로 박혀 있다.

```python
# ontology/classify.py — 임베딩 롤업에서 뺄 것
ROLLUP_EXCLUDE = {"시장","부시장","실","국","직속기관","구","의회사무국"}

# relationships/03_civil_handled_by.py — 민원 담당부서가 될 수 있는 것
WHERE o.unit_level IN ['과','팀','관','센터','사업소','직속기관','시설']
```

둘 다 **이름 없는 클래스**다. 라벨로 세우면 `WHERE o:실무조직` 한 줄이 되고, 새 조직 유형이 생겼을 때 리스트 두 곳을 찾아다니며 고칠 필요가 없어진다.

합치는 순간 두 가지가 드러난다.

- **`직속기관`을 두 리스트가 서로 반대로 쓴다.** 롤업 쪽은 "총괄만 하니 빼라", 담당부서 쪽은 "실무 조직이니 포함". 보건소 4곳·농업기술센터가 걸린다. 목적이 달라 정당한 차이일 수 있지만 지금은 물어볼 자리조차 없다
- **`'센터'`는 죽은 값이다.** 원천 232건 어디서도 `classify()`가 `'센터'`를 만들지 않는다(반려동물센터·도시안전통합센터는 override로 `사업소`)

##### 실측 분포 (원천 232건)

```
과 124 · 동 44 · 사업소 10 · 국 9 · 관 8 · 전문위원 7 · 단 6 · 시설 6
직속기관 5 · 구 4 · 부시장 2 · 실 2 · 비서실 2 · 시장 1 · 의회사무국 1 · 지방의회 1

기타 0건 · override 25건 전부 적중
```

판정 자체는 정확하다. 그래서 클래스 계층은 **검색 품질을 올리는 작업이 아니라 정리 작업**이다.

---

## Neo4j CE에서 어디까지 강제되나

[[Ontology - T-BOX]]에서 정리한 대로, 선언한 규칙이 그래프DB로 그대로 넘어가지 않는다.

| 선언 | Neo4j CE |
|---|---|
| 클래스 이름 | 라벨로 그대로 |
| 관계 이름 | 엣지 타입으로 그대로 |
| 유일 키 | `CREATE CONSTRAINT ... IS UNIQUE` 로 강제됨 |
| 속성 존재(NOT NULL) | **Enterprise 전용.** CE에서는 불가 |
| Domain/Range | 강제 안 됨 |
| Disjoint | 강제 안 됨 |
| 클래스 계층 | 강제 안 됨 |

그래서 이 안의 전제는 **온톨로지를 DB가 아니라 적재기가 강제한다**는 것이다. 선언 파일이 진실이고, 적재기가 그 선언을 읽어 A2·B·D·F를 수행한다. DB는 결과만 담는다.

이것은 우회가 아니라 기존 방침과 같다 — `schema/01_constraints.py` 주석에 이미 "필수값 검증은 적재 함수에서 애플리케이션 레벨로 한다"고 적혀 있다. 달라지는 것은 그 검증이 **어디에 적혀 있느냐**다(코드 → 선언).

---

## 작업 규모와 순서

| 순서 | 단계 | 규모 | 상태 |
|---|---|---|---|
| 1 | F1 추론 | 하루 | **완료 2026-09-01** — IN_CHARGE_BUREAU 583 · VISIT_FOR 266 |
| 2 | D3 일관성 | 반나절 | **완료 2026-09-01** — 3규칙, 위반 0 |
| 3 | A3 동일성 | 반나절 | 남음. content_hash 원천 확인 |
| 4 | D1·D2 검사 | 하루 | **완료 2026-09-01** — 위반 128건 (error 14 / warn 114) |
| 5 | F2 계층 | 반나절 | **완료 2026-09-01** — Domain 76 · IN_DOMAIN 644 |
| 6 | A2·B 게이트·라우팅 | 하루 | 남음. 원천 데이터 형태 확인 |
| 7 | 선언 파일 자체 | 1.5일 | **완료 2026-09-02** — schema.yaml + 로더 |
| 8 | D4 관계 Range·방향 | 1시간 | **완료 2026-09-02** — 도입 시점 위반 0/700, 0/938 |

**F1(추론)이 먼저였다.** 기존 데이터를 건드리지 않고 선언만 올려도 새 질의가 열린다. A2·B는 원천 데이터를 실제로 봐야 매핑표를 짤 수 있어서 뒤에 뒀고, 아직 남아 있다.

---

## 관련 문서

- [[지식그래프 진행 상황]] — 전체 진행 상황과 남은 단계

- [[KG - 온톨로지 데이터 투입 액션]] — 이 선언을 읽어 도는 파이프라인 A~F
- [[KG - 원천별 적재 (크롤링 vs pgvector)]] — 원천이 둘이 되면 선언에 source·승자 규칙이 추가된다
- [[Ontology - T-BOX]] — 온톨로지가 정의하는 4가지와 Neo4j 이관 시 손실
- [[Graph DB - 2가지 비교 (RDF, LPG)]] — 역관계를 선언하지 않는 이유
- [[RDF - 트리플과 추론, Neo4j 이관 시 손실]] — 선언 형식을 OWL로 갈 때의 대가
