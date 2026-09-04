### 노드 (Node) 예시

```
노드 = ID + 라벨(들) + 속성 딕셔너리
```

**예시 노드 1**

```
ID: 1
라벨: Person, Actor
속성: {
  name: "아이유",
  birth_year: 1993,
  agency: "EDAM엔터테인먼트"
}
```

**예시 노드 2**

```
ID: 2
라벨: Drama
속성: {
  name: "나의아저씨",
  year: 2018,
  episode_count: 16
}
```

Cypher로 만들면 이렇게 돼요:

cypher

```cypher
CREATE (:Person:Actor {name: "아이유", birth_year: 1993, agency: "EDAM엔터테인먼트"})
CREATE (:Drama {name: "나의아저씨", year: 2018, episode_count: 16})
```

여기서 보이듯:

- **ID**: Neo4j가 내부적으로 자동 부여 (사람이 신경 안 써도 됨)
- **라벨**: `:Person:Actor`처럼 콜론으로 여러 개 붙일 수 있음 (지난번 얘기한 "클래스 계층 흉내내기"가 이거였죠)
- **속성 딕셔너리**: `{key: value, ...}` 형태, 개수 제한 없이 자유롭게 추가 가능

### 엣지 (Relationship) 예시

```
엣지 = ID + 시작노드 + 끝노드 + 타입 + 속성 딕셔너리
```

**예시 엣지**

```
ID: 501
시작노드: 1 (아이유)
끝노드: 2 (나의아저씨)
타입: ACTED_IN
속성: {
  기간: "2018",
  배역: "지안",
  source_doc: "doc_003",
  confidence: 0.9
}
```

Cypher로 만들면:

cypher

```cypher
MATCH (a:Person {name: "아이유"}), (d:Drama {name: "나의아저씨"})
CREATE (a)-[:ACTED_IN {기간: "2018", 배역: "지안", source_doc: "doc_003", confidence: 0.9}]->(d)
```

### 전체를 그림으로 합치면

```
┌─────────────────────────────┐         ┌─────────────────────────────┐
│ 노드 ID: 1                    │         │ 노드 ID: 2                    │
│ 라벨: Person, Actor            │         │ 라벨: Drama                  │
│ 속성:                          │         │ 속성:                        │
│   name: "아이유"                │         │   name: "나의아저씨"           │
│   birth_year: 1993             │         │   year: 2018                │
│   agency: "EDAM엔터테인먼트"     │         │   episode_count: 16         │
└──────────────┬────────────────┘         └───────────────┬─────────────┘
               │                                           │
               │        엣지 ID: 501                        │
               │        타입: ACTED_IN                       │
               └───────► 속성:                        ◄────┘
                          기간: "2018"
                          배역: "지안"
                          source_doc: "doc_003"
                          confidence: 0.9
```

### 속성을 나중에 추가하는 예시 (자유도 확인)

이미 만든 엣지에 새 속성을 그냥 이어붙이면 됩니다:

cypher

```cypher
MATCH (a:Person {name: "아이유"})-[r:ACTED_IN]->(d:Drama {name: "나의아저씨"})
SET r.awards = "백상예술대상 후보"
```

이렇게 실행하면 501번 엣지의 속성 딕셔너리가:

```
{
  기간: "2018",
  배역: "지안",
  source_doc: "doc_003",
  confidence: 0.9,
  awards: "백상예술대상 후보"   ← 그냥 추가됨, 다른 노드/엣지에 영향 없음
}
```

**이게 RDF와 결정적으로 다른 지점이에요** — RDF였다면 이 정보를 붙이기 위해 별도 트리플들(reification)을 새로 만들어야 했는데, Neo4j는 그냥 기존 엣지 속성 딕셔너리에 한 줄 추가하면 끝이에요.

### 라벨 추가하기

아까 만든 아이유 노드가 처음엔 `Person`, `Actor`만 있었다고 해봐요:

cypher

```cypher
MATCH (p:Person {name: "아이유"})
RETURN labels(p)
// 결과: ["Person", "Actor"]
```

여기에 "가수 활동도 한다"는 걸 나중에 알게 되면, `Singer` 라벨을 그냥 추가하면 됩니다:

cypher

```cypher
MATCH (p:Person {name: "아이유"})
SET p:Singer
```

cypher

```cypher
MATCH (p:Person {name: "아이유"})
RETURN labels(p)
// 결과: ["Person", "Actor", "Singer"]
```

한 번에 여러 개도 가능해요:

cypher

```cypher
MATCH (p:Person {name: "아이유"})
SET p:Singer:Songwriter
```

### 라벨 제거하기

cypher

```cypher
MATCH (p:Person {name: "아이유"})
REMOVE p:Actor
```
