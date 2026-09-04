### 그룹 1: 완전 오픈소스 (PostgreSQL급 자유도, 상업적 제약 없음)

| DB                  | 라이선스                       | 특징                                                           |
| ------------------- | -------------------------- | ------------------------------------------------------------ |
| **JanusGraph**      | Apache 2.0                 | Linux Foundation 관리, IBM/Google 등이 지원, 대규모 분산 그래프에 강함        |
| **Apache AGE**      | Apache 2.0 (PostgreSQL 확장) | PostgreSQL 위에 Cypher 그래프 기능을 얹는 확장. PostgreSQL 자체가 무료라 완전 자유 |
| **Dgraph** (독립 실행형) | Apache 2.0                 | GraphQL 네이티브 인터페이스, 분산 처리                                    |

### 그룹 2: 오픈코어 (무료지만 상업적 제약 있음, Neo4j와 비슷한 구조)

| DB           | 라이선스                                        | 제약 내용                                    |
| ------------ | ------------------------------------------- | ---------------------------------------- |
| **Memgraph** | Business Source License (BSL)               | 개발/일반 프로덕션 사용은 무료, 다만 OSI 기준 정식 오픈소스는 아님 |
| **ArangoDB** | Business Source License + Community License | 상업적 사용 제약 있음, 단일 클러스터당 100GB 데이터 한도      |
| **FalkorDB** | Source-available 라이선스                       | 소스는 공개되지만 정식 오픈소스는 아님, GraphRAG 특화       |
| **Neo4j**    | GPL(CE) / 상업(EE)                            | (기준점, 이미 자세히 얘기함)                        |

### 그룹 3: 완전관리형 클라우드 서비스 (설치 없이 가입만)

|DB|성격|
|---|---|
|**Amazon Neptune**|AWS 운영, 종량제|
|**Azure Cosmos DB (Gremlin API)**|MS 운영, 종량제|
|**Neo4j AuraDB**|Neo4j사 운영, 종량제 (무료 티어 있음)|

----
# 자유도
라이선스 자유도 순으로 줄 세우면 
**JanusGraph(Gremlin) / Apache AGE(Cypher) / Neo4j CE(Cypher) (완전 자유) > Memgraph(Cypher) / ArangoDB(AQL) / FalkorDB(Cypher) (조건부 무료) > Neo4j EE(Cypher) (완전 유료)** 순

- **라이선스 완전 자유를 제일 중요하게 본다면** → JanusGraph나 Apache AGE가 더 깔끔함 (Neo4j는 CE/EE 구분 있고 조건이 있음)
- **속도/성능만 최우선이라면** → 용도에 따라 Memgraph(인메모리, 초저지연)나 FalkorDB(GraphRAG 특화)가 더 나을 수 있음


### Neo4j Community Edition의 제약

| 항목                  | CE                                              | EE에만 있는 기능                            |
| ------------------- | ----------------------------------------------- | ------------------------------------- |
| **단일/다중 서버**        | 단일 인스턴스만 가능                                     | 클러스터링(여러 서버로 분산) 가능                   |
| **백업**              | 오프라인 백업만 (서비스 중단 필요)                            | 온라인 백업(무중단)                           |
| **장애조치(failover)**  | 없음 — 서버 하나 죽으면 그냥 멈춤                            | 자동 장애조치 지원                            |
| **접근 제어**           | 기본적인 수준만                                        | 세밀한 역할 기반 권한 관리(RBAC)                 |
| **읽기 확장**           | 안 됨                                             | 수평 읽기 확장, Change Data Capture, 병렬 런타임 |
| **GDS(알고리즘 라이브러리)** | 전부 사용 가능하지만 **CPU 최대 4코어 제한**, 모델 카탈로그 최대 3개 제한 | 코어 수 무제한                              |

가능한 것
- Cypher 쿼리 전체
- ACID 트랜잭션
- 벡터 인덱스
- GDS 알고리즘 전부 (PageRank, Louvain 커뮤니티 탐지 등 — 4코어 제한 내에서)
- 단일 서버 기준으로는 데이터 규모/기능 제약 없음