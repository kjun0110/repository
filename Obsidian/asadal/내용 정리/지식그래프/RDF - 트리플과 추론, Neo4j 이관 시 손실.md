## RDF 트리플의 제약 — 칸 수가 3개로 고정

RDF의 최소 단위는 정의상 정확히 **3칸(주어, 술어, 목적어)**이다. 테이블처럼 "컬럼을 늘릴 수 있는데 늘리면 부담된다"가 아니라, **애초에 4번째 칸이라는 게 표준 자체에 존재하지 않는다.**

```
주어(Subject) - 술어(Predicate) - 목적어(Object)
    노드      -      관계       -    노드
```

예시:

```turtle
ex:Samsung rdf:type ex:Company, ex:PubliclyHeldCompany ;
    ex:founded "1938" ;
    ex:employees "267937" .
```

위 한 블록이 실제로는 아래 4개의 독립된 트리플로 쪼개진다.

```
(Samsung, rdf:type,     Company)
(Samsung, rdf:type,     PubliclyHeldCompany)
(Samsung, ex:founded,   "1938")
(Samsung, ex:employees, "267937")
```

---

## 추론(Reasoning) — 상속 관계는 온톨로지가 대신 채워준다

`Company`-`PubliclyHeldCompany` 같은 상속(`subClassOf`) 관계는 온톨로지 쪽에 별도로 정의해두면, 데이터에는 가장 구체적인 타입 하나만 넣어도 나머지는 추론기가 채워준다.

```
사람이 직접 입력:
  (Samsung, 타입관계, PubliclyHeldCompany)

온톨로지에 미리 정의됨:
  (PubliclyHeldCompany, subClassOf, Corporation)
  (Corporation, subClassOf, Company)
  ───────────────────────────────────
추론기가 자동 생성:
  (Samsung, 타입관계, Corporation)  ✨
  (Samsung, 타입관계, Company)      ✨
```

추론기가 온톨로지의 상속 규칙을 보고 이 두 트리플을 자동으로 채워 넣는다 — "Samsung은 Company이기도 하다"를 사람이 직접 입력하지 않아도 된다.

---

## Neo4j로 옮기면 이 추론이 사라진다

Protégé/OWL에서 설계한 걸 Neo4j(property graph)로 옮기면, 위에서 본 "자동 추론" 능력이 통째로 빠진다.

| Protégé/OWL에서 정의한 것 | Neo4j로 옮겼을 때 |
|---|---|
| 클래스 이름 (`Actor`) | 라벨 이름 (`:Actor`) — 그대로 옮겨짐 |
| 클래스 계층 (`Actor ⊂ Person`) | ❌ 규칙 자체는 안 옮겨짐 — 사람이 `SET n:Person`으로 흉내내야 함 |
| 관계 이름 (`ACTED_IN`) | 엣지 타입 이름 (`:ACTED_IN`) — 그대로 옮겨짐 |
| Domain/Range 규칙 | ❌ 강제 안 됨 — 사후 쿼리로 검증해야 함 |
| 배타 규칙 | ❌ 강제 안 됨 — 사후 쿼리로 검증해야 함 |

**Neo4j는 "이름"(라벨·관계 타입)은 그대로 받아오지만, RDF/OWL이 갖고 있던 "규칙"(계층 상속, Domain/Range, 배타 제약)은 하나도 안 따라온다.** 앞서 본 `Company`-`PubliclyHeldCompany` 상속 추론도 Neo4j에서는 자동으로 안 되고, `Samsung` 노드에 `:Company` 라벨을 사람이 직접 붙여줘야 한다.

---

## 한 눈에 요약

```
RDF 트리플     →  칸이 정확히 3개(주어-술어-목적어)로 고정된 구조
              →  Turtle 문법 한 블록도 실제로는 여러 개의 독립 트리플로 분해됨

RDF 추론      →  온톨로지에 상속 규칙(subClassOf)을 정의해두면
              →  데이터엔 가장 구체적인 타입 하나만 넣어도
              →  추론기가 상위 타입들을 자동으로 채워줌

Neo4j로 이관  →  이름(라벨/관계 타입)은 그대로 옮겨짐
              →  규칙(클래스 계층·Domain/Range·배타 제약)은 전부 빠짐
              →  상속·제약 검증을 사람이 흉내내거나 사후 쿼리로 대체해야 함
```

## 관련 문서

- [[RDF - neo4j 적용방법]]
