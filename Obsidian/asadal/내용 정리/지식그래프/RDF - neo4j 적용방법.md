### 방법 1: Neo4j에 RDF를 그대로 임포트 (neosemantics, n10s)

Neo4j 공식 플러그인인 **neosemantics(n10s)**를 쓰면 RDF/Turtle 파일(ORG, vCard 등)을 그대로 읽어서 Neo4j 그래프로 변환할 수 있습니다.

- `n10s.rdf.import.fetch()` 또는 `n10s.rdf.import.inline()` 같은 프로시저로 RDF 트리플을 노드/관계로 매핑
- 온톨로지의 클래스(`org:OrganizationalUnit`)는 노드 레이블로, 속성(`org:subOrganizationOf`)은 관계 타입으로 자동 변환됨
- 즉, ORG/vCard RDF 파일을 그대로 가져다가 Neo4j 그래프 DB에 로드하는 게 가능합니다

**장점**: 표준 온톨로지(ORG, vCard)를 그대로 재사용, 다른 시스템과 상호운용성 유지  
**단점**: RDF의 트리플 구조가 그대로 그래프에 반영되다 보니 스키마가 좀 장황해질 수 있음 (예: 리터럴 속성도 각각 별도 처리 필요)

### 방법 2: RDF는 "설계 참고용"으로만 쓰고, Neo4j는 직접 스키마 설계

실무에서 더 흔한 방식입니다.

- ORG/vCard 온톨로지의 **개념 모델**(어떤 클래스가 있고, 어떤 관계가 있는지)만 참고해서
- Neo4j 라벨/관계 타입/속성으로 직접 재설계
- 예: `(:Department)-[:SUB_ORG_OF]->(:Department)`, `(:Department)-[:HAS_CONTACT]->(:Phone {number: "02-1234-5678"})`

**장점**: Cypher 쿼리하기 훨씬 자연스럽고 성능도 좋음, 불필요한 RDF 오버헤드 없음  
**단점**: 표준 RDF 네임스페이스와의 직접 호환성은 사라짐 (하지만 속성 이름을 온톨로지 용어에서 따오면 의미적 연결성은 유지 가능)