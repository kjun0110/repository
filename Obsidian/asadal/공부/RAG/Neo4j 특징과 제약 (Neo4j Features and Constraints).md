## 개념

Neo4j를 실제 프로젝트(수원시 조직도 그래프)에 쓰면서 부딪힌 **"되는 것 / 안 되는 것"** 제약을 정리합니다. 특히 벡터 인덱스와 노드 속성(property) 쪽은 겉보기엔 유연해 보이지만 실제로는 명확한 하드 제약이 있어서, 스키마 설계 시 미리 알고 있어야 함정을 피할 수 있습니다.

---

# 벡터 인덱스 (Vector Index)

## 지원 알고리즘 — HNSW만 지원

Neo4j 5의 벡터 인덱스는 **HNSW(Hierarchical Navigable Small World) 하나만** 지원합니다. pgvector나 Faiss 같은 다른 라이브러리에 있는 IVFFlat 등의 대안은 Neo4j엔 없습니다.

| 방식 | 특징 |
|---|---|
| **Flat (브루트포스)** | 전수 계산, 100% 정확, 데이터 작을 땐 충분히 빠름 |
| **IVFFlat** | 클러스터로 나눠서 관련 클러스터만 탐색, 사전 클러스터링 필요, Neo4j 미지원 |
| **HNSW** | 그래프 기반 근사 탐색, 속도/정확도 균형 좋음, 대규모(수십만~수백만)에서 강점, **Neo4j 유일 지원** |

> 데이터가 작을 때(예: 수백~수천 개)는 어떤 알고리즘을 쓰든 체감 속도 차이가 거의 없습니다. HNSW의 진짜 장점은 대규모 데이터에서 나타남.

## "인덱스 1개 = 라벨 1개" 제약

**여러 라벨을 하나의 벡터 인덱스로 묶어서 만들 수 없습니다.**

```cypher
-- 라벨 하나를 반드시 지정해야 함
CREATE VECTOR INDEX `과_embedding` IF NOT EXISTS
FOR (n:과) ON (n.embedding)
OPTIONS {indexConfig: {`vector.dimensions`: 1536, `vector.similarity_function`: 'cosine'}}
```

**영향**: 임베딩 대상이 여러 라벨(예: 과/팀/동/관 등 9개)에 걸쳐 있으면, 벡터 인덱스도 라벨 수만큼(9개) 따로 만들어야 합니다. 검색할 때 어느 라벨에 정답이 있을지 미리 알 수 없으므로, 9개 인덱스에 전부 질의(scatter) 후 결과를 병합(gather)해야 함 — 이게 **인덱스 1개(모든 타입을 하나의 라벨로 통합한 구조)보다 느려지는 원인**입니다 (실측: 약 3.17배 차이).

---

# 라벨 매칭 vs 관계 기반 타입 조회

| 방식 | 예시 | 단계 |
|---|---|---|
| 라벨 매칭 | `MATCH (n:과)` | 노드 자체에 라벨 토큰이 저장돼 있어 **1단계**로 즉시 확인 |
| 관계 기반 타입 조회 | `MATCH (n:Instance)-[:TYPE_OF]->(c:Class {name:'과'})` | Class 노드를 찾고 관계를 타고 들어가야 하는 **2단계 이상** |

**Index-free adjacency**: Neo4j는 관계를 포인터로 바로 따라가는 방식이라, 관계 탐색 홉 하나는 그래프 크기와 무관하게 O(1)로 빠릅니다. 그래도 "한 번 더 타야 한다"는 사실 자체가 라벨 매칭보다 한 단계 더 많은 작업이라 상대적으로 느립니다.

---

# 그래프 탐색 (Graph Traversal)

## Index-free Adjacency — 왜 홉이 늘어도 안 느려지는가

RDBMS(관계형 DB)는 노드 간 연결을 외래키(FK)로 표현하고, 연결된 걸 찾으려면 매번 **JOIN**(인덱스 검색 + 매칭)을 해야 합니다. 그래프가 크고 JOIN을 여러 번 해야 할수록(다중 홉) 비용이 기하급수적으로 늘어납니다.

Neo4j는 다릅니다 — 관계를 **포인터로 노드 안에 직접 저장**합니다. "A와 연결된 B"를 찾을 때 JOIN(검색)이 아니라 그냥 **저장된 포인터를 따라가기만** 하면 됩니다. 이걸 **Index-free Adjacency**라고 부릅니다.

```
PostgreSQL: A → edges 테이블 전체 스캔(또는 인덱스 검색) → B 찾기
             (테이블이 클수록, 홉이 늘수록 느려짐 — JOIN 폭발)

Neo4j:      A → 포인터 따라가면 바로 B
             (그래프 전체 크기와 무관하게 홉 하나는 항상 O(1))
```

**결과**: 1홉이든 5홉이든, 데이터가 100개든 100만 개든, 관계 하나 타는 비용은 거의 동일합니다. 이게 그래프 DB가 다중 홉(multi-hop) 질문에 강한 이유입니다.

## N-홉 탐색 문법

Cypher에서 가변 길이 경로는 `*최소..최대` 문법으로 표현합니다.

```cypher
-- 실제 프로젝트에서 쓴 예시

-- 최상위 조직까지 끝까지 (몇 홉이 될지 미리 모름)
MATCH p = (n:Instance {id: $id})-[:PART_OF*1..]->(root)
WHERE NOT (root)-[:PART_OF]->()

-- 딱 1~2홉만 (직속 직원 + 산하 팀 소속 직원까지만)
MATCH (n:Instance {id: $id})<-[:PART_OF*1..2]-(e:Instance)-[:TYPE_OF]->(:Class {name: '직원'})
```

- `*1..` : 1홉부터 끝까지(경로 길이 상한 없음) — 조직도 최상위(시장)까지 올라갈 때
- `*1..2` : 1~2홉만 — "직속 + 산하 팀"까지만 보고 싶을 때처럼 범위를 제한할 때

## 타입 조회(TYPE_OF)와의 관계

앞서 "라벨 매칭 vs 관계 기반 타입 조회"에서 다룬 `TYPE_OF` 홉도 결국 이 **index-free adjacency 덕분에 비교적 저렴**합니다 (홉 자체는 O(1)). 다만 "1단계 vs 2단계"라는 **횟수 차이**는 여전히 남기 때문에 라벨 매칭보다는 느립니다 — **"각 홉은 싸다"는 것과 "홉을 더 타야 한다"는 것은 서로 다른 얘기**라는 게 포인트입니다.

---
# 스키마 제약 (Constraint) — 에디션에 따라 갈린다

Neo4j는 스키마가 선택사항인 DB지만, 원하면 **제약(constraint)**으로 강제할 수 있습니다. 문제는 **어떤 제약을 쓸 수 있는지가 에디션에 따라 다르다**는 점입니다.

## 제약 3종과 지원 범위

| 제약 종류 | 문법 | Community | Enterprise |
|---|---|---|---|
| **속성 유일성** (uniqueness) | `REQUIRE n.id IS UNIQUE` | ✅ | ✅ |
| **속성 필수** (existence) | `REQUIRE n.name IS NOT NULL` | ❌ | ✅ |
| **속성 타입** (property type) | `REQUIRE n.age IS :: INTEGER` | ❌ | ✅ (5.9+) |
| 노드 키 (node key) | `REQUIRE (n.a, n.b) IS NODE KEY` | ❌ | ✅ |
| 관계 속성 필수 | `FOR ()-[r:REL]-() REQUIRE r.x IS NOT NULL` | ❌ | ✅ |

> [!warning] "Neo4j가 지원한다" ≠ "내 Neo4j에서 된다"
> 공식 문서에는 세 가지 모두 지원한다고 나오지만, **Community Edition에서는 유일성 하나만 걸립니다.** 나머지는 `ConstraintCreationFailed` 에러가 납니다. 도커 이미지 `neo4j:5`는 기본이 Community라 특히 주의해야 합니다.

## 실측 (2026-08-13, `neo4j:5.26.28` Community)

수원시 조직도 프로젝트(`suwon_ontology_opus`)에서 직접 걸어본 결과입니다.

```
Neo4j Kernel ['5.26.28'] — community edition

  [지원] 유일성 (uniqueness)
  [불가] 속성 필수 (existence)      → ConstraintCreationFailed
  [불가] 속성 타입 (property type)  → ConstraintCreationFailed
  [불가] 노드 키 (node key)         → ConstraintCreationFailed
  [불가] 관계 속성 필수              → ConstraintCreationFailed
```

에디션 확인 방법:

```cypher
CALL dbms.components() YIELD name, versions, edition
```

## 문법

```cypher
-- 유일성 (Community에서도 됨)
CREATE CONSTRAINT class_name_unique IF NOT EXISTS
FOR (c:Class) REQUIRE c.name IS UNIQUE;

-- 속성 필수 (Enterprise 전용)
CREATE CONSTRAINT instance_name_exists IF NOT EXISTS
FOR (i:Instance) REQUIRE i.name IS NOT NULL;

-- 속성 타입 (Enterprise 전용, 5.9+)
CREATE CONSTRAINT instance_rank_type IF NOT EXISTS
FOR (i:Instance) REQUIRE i.rank IS :: INTEGER;

-- 관계에도 걸 수 있음 (Enterprise 전용)
CREATE CONSTRAINT works_since_exists IF NOT EXISTS
FOR ()-[r:WORKS_IN]-() REQUIRE r.since IS NOT NULL;

-- 조회 / 삭제
SHOW CONSTRAINTS;
DROP CONSTRAINT class_name_unique IF EXISTS;
```

## 알아둘 점

**① 유일성 제약은 인덱스를 자동으로 만듭니다**
`REQUIRE ... IS UNIQUE`를 걸면 그 속성에 RANGE 인덱스가 함께 생깁니다. 따로 `CREATE INDEX`를 할 필요가 없고, 반대로 **그 인덱스만 따로 지우려 하면 실패**합니다.

```
DROP INDEX class_name_unique
→ Unable to drop index: Index belongs to constraint: `class_name_unique`
```

DB를 완전히 비울 때는 **제약을 먼저 지우고 인덱스를 나중에** 지워야 합니다. (실제로 `reset_db.py`에서 이 순서 때문에 한 번 실패했습니다)

**② `IF NOT EXISTS`를 붙이는 게 안전합니다**
적재 스크립트를 반복 실행해도 에러가 안 나고, 멱등성이 보장됩니다.

**③ Community에서는 애플리케이션 레벨로 대신 검증해야 합니다**
속성 필수·타입 제약을 DB가 막아주지 않으므로, 적재 코드에서 직접 확인해야 합니다. 안 그러면 `name`이 `null`인 노드나 숫자여야 할 자리에 문자열이 들어간 노드가 조용히 쌓입니다.

```python
# 적재 후 검증 예시
MATCH (i:Instance) WHERE i.name IS NULL RETURN count(i)
MATCH (i:Instance) WHERE NOT (i)-[:TYPE_OF]->() RETURN count(i)
```

실제로 조직도 프로젝트에서는 이런 것들이 DB 제약 없이 지나갔습니다.
- `PART_OF` 없이 떠 있는 고아 노드 7개
- 원천 데이터에 없는데 이전 적재분이 남은 유령 노드 6개

둘 다 **제약으로 막히지 않는 종류**(관계 존재 여부, 스냅샷 정합성)라, 결국 적재 스크립트에 검증 로직을 넣어서 잡았습니다.
# 노드 속성(Property) 제약

## 가능한 것
- 원시값 하나 (문자열, 숫자, 불리언 등)
- **동일 타입** 원시값의 배열 (`["a", "b", "c"]`, `[1, 2, 3]`)

## 불가능한 것
- **객체(map)의 배열/리스트** — `[{name: "김철수", phone: "031-1"}, {name: "이영희", phone: "031-2"}]` 같은 구조는 속성 하나에 저장 불가. Neo4j는 문서형 DB가 아니라 프로퍼티 그래프라서, 속성은 단순 키-값만 다룸.
- **배열의 특정 인덱스만 부분 업데이트** — `SET n.arr[2] = x` 같은 문법이 없음. 배열 하나를 고치려면 **배열 전체를 다시 만들어서 통째로 덮어써야** 함.

## 우회 방법과 각각의 한계

객체 리스트를 억지로 저장하려면 두 가지 편법이 있는데, 둘 다 심각한 단점이 있습니다.

### (a) 병렬 배열
```
팀노드.names  = ["김철수", "이영희", "박민수"]
팀노드.phones = ["031-1", "031-2", "031-3"]
팀노드.duties = ["여권업무", "여권심사", "청소"]
```
인덱스 0번끼리, 1번끼리 서로 짝이라고 **암묵적으로만 약속**하는 구조. 한 사람 정보를 지우거나 순서가 바뀌면 이름-전화번호 매칭이 어긋나는 **데이터 오염 위험**이 있음. 부분 업데이트가 안 되니 수정도 번거로움.

### (b) JSON 문자열
```
팀노드.employees_json = '[{"name":"김철수","phone":"031-1"}, ...]'
```
- Cypher가 문자열 내부를 못 들여다봄 → `WHERE e.name = 'X'` 같은 네이티브 조회 불가
- 매번 애플리케이션 코드에서 전체 문자열을 가져와 파싱해야 함 → DB의 쿼리 능력을 포기하는 셈
- 풀텍스트 인덱싱 불가 (아래 항목 참고)

---

# 풀텍스트 인덱스(Fulltext Index)와 배열 속성

배열 속성(`n.duties = [...]`)에 풀텍스트 인덱스를 거는 것 자체는 가능합니다. 하지만:

**매칭 결과는 "이 노드가 매치됐다"까지만 알려주고, 배열의 몇 번째 요소가 매치됐는지는 알려주지 않습니다.**

그래서 "이 팀 직원 중 누구의 업무가 검색어와 관련있는지" 처럼 **개별 식별이 필요한 경우**, 각 대상(직원)이 독립된 노드로 존재해야만 개별 점수(`score`)를 매기고 "누가 매치됐는지" 특정할 수 있습니다. 배열/JSON으로 합치는 순간 이 개별 식별·랭킹 기능은 구현이 불가능해집니다.

---

# 설계 선택에 따른 영향 — 예시: "직원을 노드로 둘까, 상위 노드 속성으로 합칠까"

| 필요 기능 | 직원 = 별도 노드 | 직원 = 상위 노드 속성 |
|---|---|---|
| 전화번호 한 명만 수정 | `SET e.phone = 'x'` 한 줄 | 배열/JSON 전체를 읽어서 수정 후 통째로 재작성 |
| "이름이 있는 리더만" 조직 전체에서 찾기 | `MATCH (e:직원) WHERE e.name IS NOT NULL` 한 번 | 모든 상위 노드를 열어서 속성 파싱 후 확인 (훨씬 무거움) |
| 팀 소속 직원 중 관련도 높은 사람만 랭킹 | 풀텍스트 인덱스로 개별 점수 가능 | 배열 매치 여부만 알 수 있고 "누구"인지 특정 불가 |
| 데이터 정합성 | 노드 하나 = 한 사람, 오염 위험 없음 | 병렬 배열 동기화 실패 시 데이터 오염 위험 |

---

## 한 눈에 요약

```
벡터 인덱스
├── 지원 알고리즘: HNSW만 지원 (IVFFlat 등 대안 없음)
└── 제약: 인덱스 1개 = 라벨 1개 (여러 라벨 묶어서 인덱스 불가)
    └── 영향: 라벨 여러 개에 걸친 검색은 인덱스 수만큼 왕복(scatter-gather) 필요

스키마 제약 (constraint)
├── Community: 유일성(IS UNIQUE) 하나만 가능
├── Enterprise: + 속성 필수(IS NOT NULL), 속성 타입(IS :: INTEGER), 노드 키, 관계 제약
├── 유일성 제약은 인덱스를 자동 생성 → 그 인덱스만 따로 DROP 불가
│                                  → DB 초기화 시 제약 먼저, 인덱스 나중
└── Community에선 애플리케이션 레벨 검증으로 대체해야 함
    └── 관계 존재 여부·스냅샷 정합성은 애초에 제약으로 못 막음

라벨 매칭          →  노드 자체에 토큰 저장, 1단계, 네이티브 인덱스라 빠름
관계 기반 타입 조회  →  Class 찾고 관계 타야 함, 2단계 이상, 상대적으로 느림
Index-free adjacency →  관계 탐색은 포인터 기반이라 홉 1개는 O(1) (그래도 0단계보단 느림)

노드 속성 (property)
├── 가능: 원시값 하나, 동일 타입 원시값 배열
├── 불가능: 객체(map)의 배열, 배열 특정 인덱스 부분 업데이트
└── 우회 방법의 한계:
    ├── 병렬 배열 → 인덱스 정합성 깨질 위험, 부분 수정 번거로움
    └── JSON 문자열 → Cypher가 내부 조회 불가, 풀텍스트 인덱싱 불가, 매번 파싱 필요

풀텍스트 인덱스 + 배열 속성 →  매치 여부는 알아도 "배열의 몇 번째"인지는 모름
                            →  개별 식별/랭킹이 필요하면 반드시 독립 노드여야 함
```

---

## 관련 문서

- [[그래프DB 라벨 vs 온톨로지 구조 벡터검색 비교정리 (Graph DB Label vs Ontology Vector Search Comparison)]]
- [[지식그래프와 RAG 종류 (Knowledge Graph & RAG)]]
