llmgraph 프로젝트의 두 가지 추출 방식(`use_fibo` 플래그로 갈림) 비교. 언뜻 "온톨로지 vs 라벨"의 단일 축처럼 보이지만, 실제로는 서로 다른 두 축이 동시에 묶인 사례다.

## 축 정리

| 축 | 선택지 |
|---|---|
| A. 타입 표현 | 라벨 하드코딩(계층 불가) / 온톨로지(`:Class`+`SUBCLASS_OF`, 계층 가능) |
| A-1. 온톨로지 출처 | RDF 표준 재사용(FIBO 등) / 직접 설계 (예: [[KG - 지식그래프 구현 시안 내용]]) |
| B. LLM 값 자유도 | 자유 생성(텍스트 그대로) / 목록에서 선택(통제된 어휘) — 필드 구조 자체는 항상 코드에 고정 |

아래 "방식 1/2"는 축 A(+RDF 재사용) + 축 B가 동시에 갈린 조합이다. 순수 "라벨 하드코딩"이나 "온톨로지 직접설계" 사례는 이 프로젝트엔 없다.

---

## 방식 1: 온톨로지(RDF 재사용) + 목록에서 선택

```cypher
MERGE (c:FIBOClass {uri: $uri}) SET c.label = "publicly hel.."   // 타입 노드 (FIBO에서 가져옴)
MERGE (child)-[:SUBCLASS_OF]->(parent)                           // 타입 계층
MATCH (fc:FIBOClass {label: $fibo_class})                        // LLM은 이 목록 중에서 고르기만 함
MERGE (company)-[:IS_A]->(fc)
```

속성(`definition`, `uri`)은 FIBO RDF에서 그대로 가져온 값. `extract_facts_with_fibo` / `create_financial_fact_with_fibo`.

**장점**: 표준화됨, 타입 기준 검색·계층 추론 가능 · **단점**: 온톨로지 사전 준비 필요

## 방식 2: 구조 없음 + 값 자유생성

```cypher
MERGE (c:Company {name: $company})
MERGE (f:FinancialFact {company: $company, metric: $metric, year: $year})
SET f.value = $value
```

`metric` 같은 속성값은 LLM이 텍스트 표현 그대로 채움 → "revenue"/"total revenue"/"net revenues"처럼 흩어질 수 있음. `extract_facts_from_text` / `create_financial_fact`.

**장점**: 사전 준비 없이 바로 씀 · **단점**: 결과가 흩어져 검색·집계가 지저분해짐

---

## 정리

| | 방식 1 | 방식 2 |
|---|---|---|
| 타입 노드 | O (`:FIBOClass`) | X |
| 타입 속성 | 온톨로지 제공 | 없음 |
| 속성값 출처 | 목록에서 선택 | LLM이 즉석 생성 |
| 일관성 | 높음 | 낮음 |

현재 기본값은 `use_fibo=True`(방식 1), 방식 2는 폴백.

## 관련 문서

- [[Property Graph - Neo4j + 온톨로지 개념 정리]]
- [[KG - 지식그래프 구현 시안 내용]]
