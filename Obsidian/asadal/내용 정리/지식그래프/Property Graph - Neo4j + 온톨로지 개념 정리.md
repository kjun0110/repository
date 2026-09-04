> llmgraph 프로젝트 진행하며 정리한 그래프DB/온톨로지 핵심 개념
> 작성일: 2026-08-06

## 1. Cypher 노드 문법

```cypher
CREATE (c:Company {name: "Cboe"})
```

`c`=변수명(쿼리 안 임시 별명) · `Company`=라벨(노드 종류 태그) · `{name:...}`=속성(실제 값)

## 2. `MERGE` vs `CREATE`

- `CREATE`: 무조건 새로 만듦 → 반복 실행 시 중복 생김
- `MERGE`: 있으면 재사용, 없으면 생성 (find-or-create / upsert) → 반복 실행되는 적재 코드는 항상 `MERGE`

## 3. 라벨은 태그일 뿐, 노드는 각자 따로 있다

```
○ Cboe [Company]   ○ NVIDIA [Company]   ○ Apple [Company]
```

셋 다 독립된 노드. `Company`는 그냥 공통으로 붙은 이름표(철수/영희/민수가 다 "사람"인 것과 동일) — `[Company]`라는 그릇 안에 담겨있는 게 아님.

## 4. 타입을 표현하는 두 가지 방식

| | A. 라벨 직접 부여 | B. 타입을 별도 노드로 (`IS_A`/`SUBCLASS_OF`) |
|---|---|---|
| 문법 | `(:Company:Corporation {...})` | `(dog:Type{name:"개"})-[:SUBCLASS_OF]->(animal:Type{name:"동물"})`<br>`(pet:Instance{name:"멍멍이"})-[:IS_A]->(dog)` |
| 장점 | 조회 빠름(라벨 인덱스), 단순 | 타입에 `definition` 등 속성 부여 가능, 타입끼리 계층(관계) 형성 가능 |
| 단점 | 라벨끼리 관계 불가 → 계층/상속 불가능, 상위 라벨을 노드 생성 시점에 전부 미리 붙여둬야 함 | 조회 시 관계 순회 필요(`SUBCLASS_OF*0..`), 문법 복잡 |

**핵심: 라벨은 서로 관계를 못 맺는다.** 계층이 필요하면 타입을 노드로 승격시켜야 하고, 그 노드가 인스턴스에게 "라벨 역할"을 `IS_A`로 대신 해준다.

## 5. 온톨로지가 성립하는 지점

```
"동물"
  ↑ SUBCLASS_OF
"개"      "고양이"
  ↑ IS_A     ↑ IS_A
멍멍이     나비
```

라벨을 붙인 사실 자체는 온톨로지가 아니라 그냥 분류표. **온톨로지는 타입 노드끼리의 관계(`SUBCLASS_OF`)에서 성립.** "동물 전체 조회" 시 이 계층을 타고 올라가면 멍멍이/나비가 직접 연결 안 돼 있어도 찾아짐 — 단, Neo4j가 자동 추론해주는 게 아니라 쿼리에서 직접 순회 문법(`*0..`)을 써야 함.

## 6. LLM 추출 — 필드 구조는 고정, "값"만 자유 vs 선택

LLM은 라벨/필드를 만들지 않고, **정해진 필드 안의 값만** 채운다 (`company`, `metric` 등 필드명은 프롬프트에 개발자가 하드코딩).

| | 값을 자유롭게 | 값을 목록에서 선택 |
|---|---|---|
| 프롬프트 | 예시만 힌트로 (`...등`) | 전체 목록 삽입 + "이 중에서" 명시 |
| 검증 | 없음, 그대로 저장 (`MERGE`) | `MATCH`라서 목록 밖 값은 반영 안 됨 |
| 결과 | 같은 개념도 "revenue"/"headcount"로 흩어짐 | 표현이 달라도 같은 타입 노드로 수렴 |
| 타입 노드화 | 어려움 (값이 매번 달라 매칭 불가) | 가능 (목록 = 타입 노드 목록) |

→ **일반 추출 vs 온톨로지 기반 추출을 가르는 지점**은 이 "값을 목록에서 고르게 강제하는가"뿐이다.

## 7. 수치 값도 타입화하기 (Fact 패턴)

회사 타입뿐 아니라 **수치 값(지표)도 같은 방식으로 타입 노드에 연결**할 수 있다.

```cypher
CREATE (emp:Type {name: "EmployeeCount"})-[:SUBCLASS_OF]->(:Type {name: "FinancialMetric"})

CREATE (fact:Fact {value: 1647, year: 2023})-[:IS_TYPE_OF]->(emp)
CREATE (cboe:Company)-[:HAS_FACT]->(fact)
```

`"직원수 1647명"`이라는 값 자체가 `Fact` 노드가 되고, 그 노드가 `EmployeeCount` 타입에 연결된다. 값과 타입을 분리해두면 "이 회사의 모든 재무지표"나 "EmployeeCount 타입인 Fact 전부" 같은 조회가 관계 순회만으로 가능해진다.

## 8. 실무 메모

- **성능**: Neo4j 관계 순회는 포인터 추적(index-free adjacency)이라 JOIN보다 가벼움. 계층 3~10단계, 데이터 수만 개 규모에선 라벨 방식이든 타입-노드 방식이든 체감 차이 없음. 느려지는 경우는 계층이 매우 깊거나, 역방향 서브트리 전체 조회, 고빈도 반복 호출일 때. `SUBCLASS_OF*0..`처럼 상한 없는 순회는 `*0..5`로 상한 걸어두면 안전.
- **스키마**: Neo4j는 스키마 선택사항 — `CREATE TABLE` 같은 사전 선언 불필요, 쿼리에서 라벨/관계 쓰는 순간이 곧 정의. 유니크 제약(`CREATE CONSTRAINT ... IS UNIQUE`)은 선택적으로 걸 수 있음 (현재 llmgraph/fibo-rag 둘 다 안 걸어둔 상태, 데이터 커지면 고려).
- **CRUD 설계 원칙** (`neo4j_client.py`): Create는 `MERGE`+`SET`, Read는 양방향 `CONTAINS`(괄호로 AND/OR 우선순위 버그 방지), Update/Delete는 `RETURN count(...)`로 실제 영향받은 건수 확인, Delete는 `DETACH DELETE`로 관계까지 함께 제거.
