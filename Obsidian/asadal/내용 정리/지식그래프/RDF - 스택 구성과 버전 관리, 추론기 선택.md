RDF 계열로 온톨로지 기반 그래프를 구축할 때 **무엇을 설치하고(스택), 어떻게 되돌리고(버전 관리), 어느 추론기를 쓰는가**를 정리한 문서. 조사 시점은 2026-10-06. 전제는 봇 여러 개가 각자 데이터를 쌓는 멀티 테넌시 서비스이고, 조건은 **무료 + 상업적 사용 가능**이다.

결론부터 쓰면 구성은 이렇다.

| 영역            | 선택                                  | 역할                    |
| ------------- | ----------------------------------- | --------------------- |
| DB 서버         | **Jena Fuseki** (+ TDB2)            | 봇별 데이터셋 저장, SPARQL 조회 |
| 운영 추론         | **Jena 추론기** (Fuseki 내장)            | 봇 데이터에 실시간 추론         |
| 온톨로지 작성       | VS Code 또는 **Protégé**              | `.ttl` 파일 작성          |
| 설계 검사         | **HermiT** (Protégé 또는 Python 스크립트) | 모순 검사                 |
| 백엔드           | **Python** (SPARQLWrapper)          | 유일한 입구. 멀티 테넌시와 이력 기록 |
| 데이터 undo/redo | **PostgreSQL `changes` 테이블**        | 변경 이력 직접 기록           |
| 온톨로지 버전 관리    | **git**                             | `.ttl` 파일의 변경 이력      |

---

## 표준과 구현물을 먼저 가른다

가장 많이 헷갈리는 지점이다. **RDF는 데이터를 적는 형식일 뿐이고, 추론은 소프트웨어가 한다.**

| | 성격 | 비유 |
|---|---|---|
| **RDF** | W3C 데이터 모델. `주어 - 술어 - 목적어` 트리플 | 문장을 쓰는 문법 |
| **RDFS / OWL** | RDF 위에 얹는 어휘와 규칙 표준. 의미(semantics)를 정의 | 수학 공리 |
| **SPARQL** | RDF용 질의 언어 | SQL |
| **Apache Jena** | 위 표준들을 다루는 Java 프레임워크 + 저장소 + 서버 | 공리를 보고 실제로 증명하는 프로그램 |

관계형 DB로 치면 **RDF가 "테이블 모델", Jena가 "MySQL 엔진 + 드라이버"** 에 가깝다. 그래서 같은 RDF 데이터라도 **추론기를 붙이느냐에 따라 조회 결과가 달라진다.**

```turtle
# 데이터에 적힌 것
:Happy  rdf:type        :Dog .
:Dog    rdfs:subClassOf :Animal .
```
이 상태로 `:Happy rdf:type :Animal` 은 **어디에도 없다.** RDF 파일은 적힌 것만 갖고 있다. 추론기를 붙여야 이 트리플이 도출된다.

### Jena 홈 화면의 여섯 부품

| 칸 | 부품 | 하는 일 |
|---|---|---|
| **RDF** | RDF API | Java 코드로 트리플을 만들고 읽고 파일로 저장 |
| | **ARQ** | SPARQL 질의 엔진 |
| **Triple store** | TDB / TDB2 | 트리플을 디스크에 영구 저장 |
| | **Fuseki** | SPARQL 엔드포인트를 HTTP 로 제공하는 서버 |
| **OWL** | Ontology API | RDFS · OWL 로 클래스와 관계를 정의 |
| | **Inference API** | 그 규칙대로 새 사실을 추론 |

Python 입장에서 중요한 건 **API 세 개는 Java 라이브러리라 쓸 일이 없고, Fuseki 하나만 설치하면 ARQ · TDB · 추론이 전부 들어 있다**는 것이다. 추론은 Fuseki **설정 파일**에서 켜고, 온톨로지는 `.ttl` 로 올린다.

```
[Python 봇] ──HTTP──▶ Fuseki
                        ├─▶ ARQ (SPARQL 실행)
                        ├─▶ Inference (추론) ← Ontology 정의 사용
                        └─▶ TDB2 (디스크 저장)
```

---

## 추론기 — 역할이 둘로 갈린다

| | **Jena 추론기** (Fuseki 내장) | **HermiT / Openllet** |
|---|---|---|
| 방식 | 규칙 기반 | 논리 증명 기반 (OWL DL 완전 지원) |
| 엄밀함 | OWL 의 **일부**만 처리 | 모든 OWL DL 추론 + 모순 검사 |
| 속도 | 데이터가 많아도 실시간으로 쓸 만함 | 데이터가 많으면 **매우 느려짐** |
| 쓰는 때 | **운영 중** 봇 데이터에 실시간 추론 | **설계할 때** 온톨로지 검사 |

우열이 아니라 역할이 다르다. 그래서 **둘 다 쓴다.**

```
[설계]  온톨로지 작성 → HermiT 로 엄밀하게 검사 (Protégé 또는 스크립트)
[운영]  Fuseki 에 올림 → Jena 추론기로 봇 데이터에 실시간 추론
```

HermiT · Pellet 은 **독립 프로그램**이다. Protégé 가 플러그인으로 가져다 쓰는 것뿐이라, Protégé 없이 Python `owlready2` 나 ROBOT 으로도 같은 추론기를 쓸 수 있다.

### 함정 — 설계에서 되던 추론이 운영에서 안 된다

**HermiT 에서 추론되는 게 Fuseki 에서도 추론된다는 보장이 없다.** Jena 추론기가 처리하지 못하는 OWL 표현을 쓰면, 설계할 때는 결과가 나왔는데 운영에서는 안 나온다.

```turtle
# 개수 제약 — Jena 에서 제대로 처리되지 않을 수 있다
:MultiChildParent owl:equivalentClass [
    owl:onProperty :hasChild ; owl:minCardinality 2
] .
```

그래서 **운영용 온톨로지를 Jena 가 잘 처리하는 범위(대략 OWL 2 RL)로 작성**하는 게 안전하다. `subClassOf` · `subPropertyOf` · `domain`/`range` · `inverseOf` · `TransitiveProperty` · `SymmetricProperty` 는 Jena 에서 잘 된다.

---

## OWL 2 프로파일 — EL · QL · RL · DL

OWL 2 전체(DL)는 표현력이 강한 대신 추론이 매우 느려질 수 있다. 그래서 W3C 가 용도별로 기능을 줄인 **프로파일 세 개**를 두었다. (OWL 2 는 2009년 표준, 2012년 개정)

```
              OWL 2 DL (전체, 엄밀, 느릴 수 있음)
           ┌───────────┼───────────┐
        OWL 2 EL    OWL 2 QL    OWL 2 RL
       (큰 분류체계)  (DB 질의)   (규칙 기반) ← RDF DB 가 잘 처리하는 범위
```

| 프로파일 | 이름 | 적합한 용도 | 대표 도구 |
|---|---|---|---|
| **EL** | Existential Language | 클래스가 수십만 개인 거대한 **분류 체계** | ELK, 의료 용어 체계 SNOMED CT |
| **QL** | Query Language | 기존 **관계형 DB를 온톨로지로 조회** | Ontop |
| **RL** | **Rule Language** | **RDF DB 에 쌓인 데이터에 실시간 추론** | **Jena**, GraphDB, RDFox |
| **DL** | Description Logic | 전체 표현력. 설계 검사와 모순 탐지 | HermiT, Pellet/Openllet |

### OWL 2 RL 에서 되는 것과 안 되는 것

| 되는 것 | 제한되는 것 |
|---|---|
| `subClassOf`, `subPropertyOf` | `minCardinality` 같은 **개수 제약** |
| `domain`, `range` | "어떤 X 가 **반드시 존재한다**"를 결론으로 내기 |
| `inverseOf`, `TransitiveProperty`, `SymmetricProperty` | "A **또는** B 다" 같은 합집합 결론 |
| `sameAs`, `equivalentClass` (단순한 경우) | |
| `disjointWith`, `FunctionalProperty` (모순 탐지) | |

핵심 원리는 하나다. RL 은 **있는 데이터로부터 새 사실을 계산**하는 건 잘하지만, **데이터에 없는 개체가 존재한다고 가정**하는 추론은 못 한다.

```turtle
# RL 에서 됨 — 있는 사실로 계산
:Tom :hasParent :John .   →   :John :hasChild :Tom .

# RL 에서 안 됨 — 없는 개체의 존재를 추론
"모든 사람은 엄마가 있다" + :Tom a :Person
→ "Tom 에게는 (이름은 모르지만) 엄마가 존재한다"   ✗
```

**실제 데이터에 대한 추론이 목적이면 RL 로 대부분 충분하다.**

---

## 추론기 목록과 선택

### 엄밀한 OWL DL 추론기 — 설계·검사용

| 추론기 | 특징 | 라이선스 |
|---|---|---|
| **HermiT** | 옥스퍼드 대학. Protégé · owlready2 기본 내장. 가장 표준적 | LGPL |
| **Openllet** | Pellet 을 이어받은 버전. **SWRL 지원**, 모순 원인 설명이 좋음, **Jena 에 끼울 수 있음** | **AGPL** |
| FaCT++ | C++ 로 만들어 빠른 편. 유지보수가 활발하지 않음 | |
| Konclude | 병렬 처리로 매우 빠름. 사용 사례가 적음 | |
| JFact | FaCT++ 의 Java 이식 | |
| (원조) Pellet | 개발이 멈춰 Openllet 으로 대체됨 | |

### 프로파일 전용

| 추론기 | 특징 |
|---|---|
| **ELK** | **OWL 2 EL 전용.** 클래스 수십만 개도 몇 초 만에 분류 |

> ELK 가 의료에서만 빠른 게 아니다. **EL 범위로 작성된 온톨로지면 분야와 무관하게 빠르다.** 의료가 대표 사례인 건 그쪽 온톨로지가 "클래스 수십만 개 + EL 형태"라서다. 다만 **개별 개체에 대한 추론은 약해서** 봇 데이터 실시간 추론에는 맞지 않는다.

### 운영용 — 데이터에 실시간 추론 (대부분 RL 계열)

| 추론기 | 특징 |
|---|---|
| **Jena 추론기** | Fuseki 내장. 무료 |
| GraphDB 추론기 | RL · QL 지원. 저장할 때 미리 계산 |
| RDFox | 상용. 매우 빠르고, 바뀐 부분만 다시 계산 |
| **owlrl** (Python) | rdflib 위에서 OWL 2 RL 추론. 무료. **Fuseki 에 올리기 전 미리 확인용**으로 쓸 만함 |

### 선택

**설계 검사는 HermiT 하나로 충분하다.**

| | HermiT | Openllet |
|---|---|---|
| Protégé | 기본 내장 | 플러그인 |
| Python (owlready2) | 내장 | owlready2 에는 원조 Pellet 이 들어 있음 |
| SWRL | 부분 지원 | **강함** |
| 모순 원인 설명 | 기본 수준 | **좋음** |
| Jena 연동 | 안 됨 | **됨** |
| 라이선스 | **LGPL** | **AGPL** |

- **HermiT 는 LGPL 이라 상업 서비스에 라이브러리로 포함해도 내 코드를 공개할 의무가 없다.** 별도 상업용 유료 라이선스도 없다. HermiT 소스를 수정해 배포할 때만 그 부분을 공개하면 된다. 몇 년째 큰 업데이트 없이 안정화된 상태다
- **Openllet 은 AGPL** 이다. 개발 PC 에서 설계 검사 도구로 쓰는 건 문제없지만, **서비스 서버에 넣어 운영하면 소스 공개 의무가 생길 수 있다.** 상업 서비스라면 주의
- Openllet 을 Jena 에 끼우면 운영에서도 엄밀한 DL 추론이 가능하지만, 데이터가 많아지면 매우 느려서 데이터가 작을 때만 현실적이다

---

## 버전 관리 — 대상이 두 개다

가장 헷갈리는 지점. 관리 대상이 둘이라 방법도 둘이다.

| 대상 | 예시 | 관리 방법 |
|---|---|---|
| **온톨로지** (규칙) | `:Dog rdfs:subClassOf :Animal` | **git** (`.ttl` 파일) |
| **데이터** (봇이 쌓는 사실) | `:Tom :age 11` | **변경 이력 테이블** (PostgreSQL) |

### Fuseki 가 기본으로 주는 것 — undo 는 없다

| 기능 | 가능한 것 | 한계 |
|---|---|---|
| 트랜잭션 롤백 | 작업 도중 실패하면 커밋 전 상태로 | **커밋한 뒤에는 되돌릴 수 없다** |
| 백업(스냅샷) | 특정 시점 전체를 파일로 저장·복원 | 통째로 되돌리는 것이라 **세밀한 undo 불가** |

git 처럼 버전 기록이나 undo/redo 가 **내장돼 있지 않다.** 직접 만들어야 한다.

### 세 가지 방법

| | **1. 앱에서 직접 기록** | 2. RDF Delta | 3. 버전 내장 DB |
|---|---|---|---|
| 기록 주체 | **내 Python 코드** | Fuseki 가 자동 전송 | DB 자체 |
| 기록 위치 | **내가 고른 DB**(Postgres 등) | Patch Server 의 디스크 폴더 (ZooKeeper · S3 도 가능) | DB 내부 |
| undo 구현 | 기록을 꺼내 **반대로 실행** (쉬움) | 역패치를 만들어 적용 (어려움) | `reset` 명령 |
| 추가 운영 | 없음 | **Patch Server 하나 더** | DB 교체 |
| 비고 | | HA(복제)와 이력을 한 번에 해결 | **TerminusDB** — git 개념(commit · branch · diff · 시간 여행) 내장, **대신 OWL 추론이 약함** |

**원리는 1번과 2번이 같다 — 변경 이력을 별도 저장소에 쌓아 두고 반대로 적용해 되돌린다.** 차이는 누가 어디에 쌓느냐뿐이다. 어느 쪽이든 **Fuseki 에는 현재 상태만 있고, 과거로 돌아갈 정보는 이력 저장소에 있다.**

- **1번은 PostgreSQL 에 쌓을 수 있다.** 저장 위치를 내 코드가 정하기 때문이다
- **2번은 안 된다.** RDF Delta Patch Server 는 디스크 · ZooKeeper · S3 정도만 지원한다. 패치를 꺼내 Postgres 로 옮기는 코드를 짜면 결국 1번과 같아진다
- **3번은 추론이 핵심인 이 구조에 맞지 않는다**

→ **1번 채택.** Fuseki(현재 데이터) + PostgreSQL(변경 이력).

### 1번의 실제 모양

```
                        ┌──▶ Fuseki      /bot1, /bot2   ← 지식 데이터 (RDF)
[Python 백엔드] ────────┤
                        └──▶ PostgreSQL  changes 테이블  ← 변경 이력 (undo/redo)
```

```sql
CREATE TABLE changes (
  id             BIGSERIAL PRIMARY KEY,
  bot            TEXT NOT NULL,     -- 어느 봇의 변경인지
  created_at     TIMESTAMPTZ DEFAULT now(),
  delete_triples JSONB,             -- 삭제한 트리플
  add_triples    JSONB,             -- 추가한 트리플
  status         TEXT NOT NULL      -- 'done' / 'undone'
);
CREATE INDEX ON changes (bot, status, id);
```

| 동작 | 처리 |
|---|---|
| **변경** | 현재 값 조회 → 바뀌는 내용 계산 → SPARQL Update 실행 → `changes` 에 INSERT(`done`) → 같은 봇의 `undone` 기록 삭제(새 변경이 생기면 redo 는 무효) |
| **undo** | 그 봇의 `done` 중 **id 가 가장 큰 줄** → 반대로 실행(add 를 지우고 delete 를 넣음) → `undone` 으로 변경 |
| **redo** | 그 봇의 `undone` 중 **id 가 가장 작은 줄** → 원래대로 재실행 → `done` 으로 변경 |

### 지켜야 할 규칙

1. **실제로 바뀐 것만 기록한다.** 이미 있는 트리플에 추가 요청이 와도 실제로는 안 바뀐다. 이걸 "추가함"으로 기록하면 **undo 할 때 원래 있던 데이터까지 지운다.** 추가 전에 "이미 있나", 삭제 전에 "실제로 있나"를 확인할 것
2. **추론으로 생긴 트리플은 기록하지 않는다.** 원본만 기록하고 되돌리면 추론 결과는 자동으로 따라온다
3. **이력은 서버를 꺼도 남게 저장한다.** 메모리 리스트는 재시작하면 사라진다
4. **저장소가 둘이라 둘 다 성공해야 한다.** Fuseki 수정은 됐는데 Postgres 기록이 실패하면 이력이 어긋난다 → ① Postgres 에 `pending` 으로 먼저 기록 ② Fuseki 수정 ③ 성공하면 `done`, 실패하면 기록 삭제

### 쓰기 경로는 하나로

1번은 **내 코드를 거친 변경만** 기록된다. 그래서 **Fuseki 직접 쓰기를 막고 모든 쓰기를 Python 백엔드로 통일**한다. 멀티 테넌시 격리를 위해서도 어차피 그렇게 해야 한다.

| 요구사항 | 백엔드 한 곳에서 처리 |
|---|---|
| 멀티 테넌시 | 요청한 봇의 데이터셋만 접근하도록 강제 |
| undo/redo | 모든 변경을 빠짐없이 기록 |
| 데이터 검증 | 잘못된 트리플이 들어오기 전에 차단 |
| 로그 · 권한 | 누가 언제 뭘 했는지 한 곳에서 |

**혼합(2번으로 추출해 Postgres 에 적재)은 undo 목적으로는 1번만 못하다.** 운영할 것이 늘고, 기록 시점에 지연이 생기며, 패치에는 **"누가 왜 바꿨는지"와 "사용자 요청 하나"라는 단위**가 없다. undo 는 본래 사용자 행동 하나를 취소하는 것이라 그 맥락이 필요하다. 혼합이 나은 경우는 **백엔드를 우회하는 쓰기(관리자가 Fuseki UI 에서 직접 수정, 다른 시스템의 SPARQL Update, 일괄 업로드)가 있거나 HA 가 필요할 때**뿐이다.

---

## 대안을 왜 안 골랐나 — LPG(Postgres + AGE)

| | **Fuseki + PostgreSQL** (선택) | PostgreSQL + **Apache AGE** |
|---|---|---|
| 데이터 모델 | RDF | **LPG** (Neo4j 와 같은 방식) |
| 질의 | SPARQL | openCypher + SQL |
| **OWL 추론** | **된다** | **안 된다** |
| 온톨로지 | 표준 언어로 선언 → DB 가 이해 | 데이터로 저장 → DB 는 의미를 모름 |
| 운영할 DB | 2개 | **1개** |
| undo 이력과 데이터의 일관성 | 저장소가 2개라 신경 써야 함 | **한 트랜잭션으로 처리** |
| 멀티 테넌시 · 권한 · HA | Fuseki + RDF Delta | **Postgres 기능 그대로, 전부 무료** |
| 라이선스 | Apache 2.0 | Apache 2.0 |
| 성숙도 | Fuseki 는 오래되고 안정적 | 상대적으로 신생. 지원 Postgres 버전 확인 필요 |

AGE 에서도 온톨로지를 노드·관계로 **저장**하고 `SUBCLASS_OF*0..` 같은 가변 길이 경로로 계층 탐색 정도는 **쿼리로 흉내** 낼 수 있다. 하지만 **규칙을 선언하면 알아서 추론해 주는 기능이 없어서**, 규칙이 바뀌면 관련 쿼리와 코드를 전부 고쳐야 한다. `.ttl` 을 가져오는 도구도 없다(Neo4j 에는 n10s 가 있다).

**판단 기준은 질문 하나다 — "봇이 추론으로 새 사실을 알아내야 하나?"**
추론이 서비스의 핵심 가치면 Fuseki, "있으면 좋은 것" 정도면 AGE 가 운영 부담이 훨씬 적다. **온톨로지 기반 GraphRAG 가 목표라 Fuseki 를 택했다.** (벡터 검색은 Fuseki 가 약하므로 같은 PostgreSQL 에 pgvector 를 붙이는 조합)

---

## 지킬 원칙

1. **온톨로지는 OWL 2 RL 범위로 작성한다** — HermiT 검사 결과와 Fuseki 운영 추론 결과를 일치시키기 위해
2. **쓰기는 반드시 Python 백엔드를 거친다** — Fuseki 포트는 외부에 공개하지 않는다
3. **이력에는 실제로 바뀐 원본 트리플만 기록한다** — 추론 결과는 기록하지 않는다
4. **봇마다 데이터셋을 분리한다**

라이선스는 전부 무료이고 상업적 사용에 문제가 없다 — Fuseki **Apache 2.0**, HermiT **LGPL**, PostgreSQL · Apache AGE **무료/Apache 2.0**. 단 Openllet 은 **AGPL** 이라 서버 운영에 넣을 때 주의.

---

## 관련 문서

- [[1006_KG - 나라원 · 설계도가 통째여야 증분이 안전하고, 되돌리기는 버전이 아니라 출처 단위여야 한다]] — 이 조사를 한 날의 기록. 같은 날 나라원 스택에 대보고 내린 결론(현행 유지)이 거기 있다
- [[Graph DB - 2가지 비교 (RDF, LPG)]] — RDF 진영과 LPG 진영의 갈림. 이 문서는 RDF 쪽을 골랐을 때의 구체적 구성이다
- [[RDF - 트리플과 추론, Neo4j 이관 시 손실]] — 추론이 무엇을 채워 주는지와, Neo4j 로 옮길 때 그게 왜 사라지는지
- [[RDF - neo4j 적용방법]] — 반대 방향. LPG 에서 RDF/OWL 을 흉내 낼 때의 한계
- [[KG - 온톨로지 선언 파일 (TBox)]] — `.ttl` 로 무엇을 선언하는지. git 으로 관리할 대상이 이 파일이다
- [[Ontology - T-BOX]] — 규칙(TBox)과 데이터(ABox)의 구분. 버전 관리 대상이 둘로 갈리는 이유와 같은 구분이다
- [[Graph DB - 종류 조사]] — Jena · GraphDB · Stardog · Virtuoso · Neptune 등 제품 비교
- [[KG - Hybrid RAG]] — GraphRAG 에 벡터 검색이 함께 필요한 이유
