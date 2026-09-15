# 나라원 온톨로지 기반 KG 적재 파이프라인

nara1.kr 사이트와 직원 명단을 온톨로지 0.4.1 로 Neo4j 에 적재하고, 사업·보유기술 임베딩까지 채우는 운영 적재 경로의 기록.
저장소는 `Ai_Pro_Neo4j_Search`(사내 Gitea), 2026-09-10 ~ 09-15 작업.

[[KG_나라원 온톨로지 기반 KG 실습]]이 프로토타입(`nara1graph`)이었다면, 이 문서는 **운영에 넣는 코드**다.
[[KG_나라원 운영 전환 점검]]에서 막혔던 것(적재 트랜잭션 · 온톨로지 버전 · 개인정보 · 삭제)을 어떻게 풀었는지가 중심이다.

관련: [[KG_나라원 온톨로지 그래프 설계]] · [[KG_온톨로지 기반 KG 생성 시 고려할 점]] · [[KG 수원-나라원 파이프라인 비교]]

---

## 요약

```
온톨로지 nara1-0.4.1 (확정본 sha 38d15d58…)   라벨 10 · 관계 8 · 어휘 10
노드 584 · 관계 761 · 벡터 220 · 제약 10 · 인덱스 22 (벡터 2)
Neo4j 5.26.30 community  ·  문장 gpt-5.6-luna  ·  벡터 text-embedding-ada-002 (1536)
드라이런 매니페스트 fd11c4b5…  (코드 · 수집본 · 결정 표 · 명단 · 온톨로지가 같으면 어디서 돌려도 같다)
```

```
snapshots/ + private/hr/ 명단
   → parsers (추출)  → candidates (후보)  → ontology.validation (검증)
   → release.json (릴리즈)  → writers (Neo4j 쓰기 · 되읽기 확인)  → embed (임베딩)
```

진입점은 하나다.

```bash
uv run --frozen python -m ingestion.load --output .local/r1 --target production --env-file .env --write
```

이 한 줄이 쓰기 → 되읽기 대조 → 임베딩(바뀐 노드만)까지 한다.

---

## 1. 폴더 — "세션을 만지나"로 가른다

```
collector.py → snapshots/ → ingestion/ → Neo4j → graph_rag/ → serving/
                                ↑
                      ontology/ + resolutions/     ← 단계가 아니라 입력
```

| 폴더 | 역할 |
|---|---|
| `ontology/` | 선언(`definition/declaration.py`) → 굽기 → 확정본(`versions/*.json`). `registry` 가 확정본을 읽고, `version.py` 가 버전을 적는 **유일한 곳** |
| `resolutions/` | 사람 결정 CSV — 잘린 발주처 이름 · 사업별 발주처 · 기관유형 동률 · 주요실적 카드 |
| `ingestion/parsers/` | 원문 → 레코드. 추출 규칙 표 11개 + 직원 명단 엑셀 |
| `ingestion/candidates/` | 레코드 → 노드 · 관계 후보 (site · projects · employees · removed) |
| `ingestion/release/` | 후보 → 검증 → 릴리즈 문서. **DB 없이 전부 돈다** |
| `ingestion/writers/` | Neo4j 세션을 만지는 코드 전부. `plan.py` 만 순수(Cypher 생성) |
| `ingestion/embed.py` · `embedding/` | 임베딩 단계 — 재료 · 지시문 · OpenAI 호출 |
| `serving/` · `graph_rag/` | 조회 API (Docker 이미지에는 `serving/` · `graph_rag/` · `ontology/` 만) |

`release/` 와 `writers/` 의 경계가 **드라이런과 섀도우 적재가 확인하는 범위와 정확히 같다.** 경계가 흐리면 "섀도우에서 뭘 확인한 건가"가 흐려진다.

---

## 2. 온톨로지 0.4.1

### 라벨 10 · 노드 584

| 라벨 | 건수 | 키 | 주요 속성 |
|---|---|---|---|
| `Company` | 1 | `company_id` | name · founded_date · ceo_name · capital_text · description |
| `OrganizationalUnit` | 33 | `unit_id` | name · unit_type · display_order |
| `HistoryEvent` | 27 | `event_id` | year · month · description · scope |
| `Office` | 4 | `office_id` | name · address · transit |
| `BusinessArea` | 5 | `area_id` | name · description · body_text (분류 없음 1 포함) |
| `Offering` | 12 | `offering_id` | name · description · body_text · **headlines** · embedding_text · embedding |
| `BusinessProject` | 208 | `project_id` | name · contract_date · client_type · public_url · embedding_text · embedding |
| `Client` | 81 | `client_id` | name · identity_status · name_truncated · merged_from · client_type |
| `Employee` | 212 | `employee_id` | name · hire_date · position · rank · contract_type · job_role · 전화 2종 · employment_status |
| `ReleaseHead` | 1 | `head_id` | release_seq · manifest_sha256 · ontology_version |

사이트에서 오는 8개 라벨은 `site_status`(present / removed), 직원은 `employment_status`(active / left) 를 가진다.

### 관계 8 · 761

```
                         (Company)
         ┌──────────┬───────┼────────┬────────────┐
   OPERATES_IN   OFFERS  HAS_EVENT  LOCATED_AT    ↑ MEMBER_OF (회사 직속)
        4          12       27         4
        ↓          ↓        ↓          ↓
 (BusinessArea) (Offering) (HistoryEvent) (Office)
        ↑
     IN_AREA 261
        │
 (BusinessProject) ──ORDERED_BY 208──▶ (Client)

 (Employee) ──MEMBER_OF 212──▶ (OrganizationalUnit | Company)
 (OrganizationalUnit) ──SUB_ORG_OF 33──▶ (OrganizationalUnit | Company)
```

### 어휘 10

```
unit_type · event_scope · identity_status · client_type(사이트 원문 6종)
contract_type · employment_status · job_role · position · rank   ← 직원 명단 기준
site_status                                                      ← 0.4.1
```

### 출처 — 모든 노드·관계에 속성 3개

```
source_uri     원문 페이지 주소 (명단은 file:<파일 이름>)
source_sha256  원문 바이트 해시
observed_at    수집 시각 (명단은 파일 이름 날짜, 한국시간 0시)
```

근거층 노드(Assertion · SourceRecord)나 사실 원장을 두지 않는다. **무엇으로 만들었는지는 릴리즈 매니페스트의 해시들이, 순서는 매니페스트 체인이 남긴다.**

### 릴리즈 밖에서 관리하는 칸 (managed)

`embedding_text` · `embedding` · `embedding_source_sha256` 은 선언에 있지만 **적재기가 쓰지 않는다.**
임베딩 단계만 쓰고, 적재기는 대조에서 빼고 노드를 덮어쓸 때 보존한다.

```cypher
WITH n, row, n {.embedding, .embedding_text, .embedding_source_sha256} AS kept
SET n = row.properties
SET n += kept
```

벡터를 릴리즈에 넣는 안은 버렸다 — 문서가 커지고 실수 표기가 환경마다 달라 해시가 흔들린다.

### 버전 — 한 곳에서, 조회는 "호환 목록"으로

```python
# ontology/version.py
ONTOLOGY_VERSION = "nara1-0.4.1"                 # 적재 · 굽기
SERVING_COMPATIBLE_VERSIONS = ("nara1-0.4.1",)   # 조회 API 가 받아들이는 DB 버전
```

버전이 선언 파일과 registry 두 곳에 있었고 같은지 아무도 검사하지 않았다. 지금은 한 곳이고, 적재 버전이 호환 목록에 없으면 import 에서 멈춘다.

| 변경 | 버전 | 호환 목록 | 절차 |
|---|---|---|---|
| 추가만 (칸 · 라벨 · 관계) | 0.4.1 → 0.4.2 | 옛 버전 유지 | 조회 먼저 배포 → 적재 → 다음 배포 때 옛 버전 제거. 끊김 없음 |
| 변경 · 삭제 | 0.4.x → 0.5.0 | 새 버전만 | API 멈춤 → 적재 → 배포 |

확정본(`versions/`)은 고치지 않는다. 운영 적재는 확정본만 받는다(`--allow-draft` 거부).

---

## 3. 단계별

### ① 수집본 — `snapshots/snapshot-full*`

`collector.py` 로 받은 원문과 `manifest.jsonl`(주소 · 수집 시각 · sha256 · 크기). 적재기는 원문 바이트를 매니페스트와 대조하고 다르면 멈춘다.
2026-09-02 전체 수집 + 보강 3벌이 저장소에 있다.

### ② 추출 — 규칙 표 + 해석기

회사 4 · 사무소 · 사업분야 · 연혁 · 보유기술 · 게시판 행 · 카드 · 조직도 = 11개. 직원은 `parsers/roster.py`(표준 라이브러리로 xlsx 읽기).

명단은 조금만 이상해도 멈춘다 — 머리글 15칸 위치 · 병합 셀 · 수식 · 아이디 중복 · 필수 칸 빈칸 · 파일 이름에 날짜 없음.

**보유기술 "큰 글"(`headlines`)** — 본문 앞 150자나 LLM 요약 대신 페이지의 제목 · 소제목 · 슬로건만 모았다.

```
넣음   h3 · h4 · h5 · 클래스에 tit/title · p.txt-style1 · div.txt-style2 · strong.sz_big
뺌     클래스 없는 strong(문장 속 강조) · 본문 문단
```

- `</p>` 가 빠진 문단이 뒤 제목 2개를 삼켰다 → HTML 규칙대로 다음 블록 시작에서 문단을 닫는다
- 클래스 이름만 제목이고 내용은 본문인 문단 → 100자 넘으면 뺀다
- AI 검색 페이지는 본문 없는 검색 앱 화면 → 화면의 기능 이름(검색 분류 · 범위 · 조건 · 결과 구역 제목)을 모은다
- 12쪽 중 하나라도 큰 글이 비면 멈춘다 — 사이트 구조가 바뀐 신호

### ③ 후보 — 사람 결정을 먼저 반영한다

- **발주처** — 게시판이 10자로 잘라 내보낸 이름 23종을 사람이 확정. 우선순위는 사업별 결정 > 발주처 결정 > 원문.
  확정 이름이 같으면 합치고(2쌍), 잘린 이름 하나에 기관이 둘이면 사업별로 나눈다(과학기술정보통신부 → 국립과천과학관 · 중앙전파관리소). 82 → 81.
- **분류 없음** — 사업분야가 없는 사업 5건은 `분류 없음` 사업분야에 붙인다. 회사와는 잇지 않는다(사이트가 그런 분야를 말한 적이 없다).
- **직원 소속** — 팀이 조직도에 있으면 팀, 없으면 부서, 없으면 회사. 조직도에 없는 팀은 알림. 팀 197 · 실 9 · 사업부 3 · 회사 3.
- **직원 키** — `sha256(직원아이디 | 성명)`. 입사일 정정 · 재입사는 덮어쓴다.

사람 결정을 후보에 먼저 넣으므로 속성을 통째로 교체(`SET n =`)해도 다음 적재에서 원문값으로 되돌아가지 않는다.

### ④ 검증 — 틀린 것만 빼고, 기준을 넘으면 전부 멈춘다

선언에서 검사가 나온다(칸 · 타입 · 어휘 · 카디널리티 · 끝점). 틀린 후보만 거부하고 리포트에 남긴다.
선언 · 코드 문제이거나 거부가 10% 를 넘으면 배치 전체를 거부한다.

### ⑤ 릴리즈 — `release.json`

```
manifest   release_seq · previous_manifest_sha256 · expected_before_fingerprint
           ontology {version, status, sha256} · snapshots · resolutions · roster
           extractors {이름: 버전} · content_sha256 · report_sha256 · counts
graph      검증 통과 노드 · 관계 + ReleaseHead
report     halted · rejected · review_required · pending_decisions · notes
```

- 결정 표 · 명단 · 추출 규칙 버전까지 해시로 묶는다 → CSV 한 칸을 채워도 다른 릴리즈가 된다
- 이전 릴리즈를 먼저 자기 검사한다(매니페스트 · 그래프 · 리포트 · 헤드 해시)
- **드라이런 매니페스트 해시가 회귀 기준**이다. 리팩토링 네 번 모두 해시가 같아 "넣을 데이터가 한 글자도 안 바뀌었다"를 증명했다

### ⑥ 쓰기 — 지금 DB 가 내가 예상한 그 상태인가

```
preflight       스키마 DDL 전에 — DB 가 이 릴리즈의 이전 상태(처음이면 빈 DB)거나 이미 같은 상태인가
apply_schema    제약 · 인덱스 · 벡터 인덱스
apply_release   한 트랜잭션: ReleaseHead 잠금 → 되읽기 → 지문 확인 → 달라진 것만 쓰기(건수 확인)
                → 되읽어 릴리즈와 대조 → 다르면 예외 → 전체 되돌림
verify_release  그래프 대조 + 제약 · 인덱스 이름 · ONLINE
```

**삭제 정책**

| 대상 | 처리 |
|---|---|
| 관계 | 릴리즈에 없으면 지운다 — 팀 이동 · 퇴사가 있는 두 번째 적재가 항상 실패하므로 |
| 퇴사자 | 노드는 남기고 `left`, 소속 관계만 지운다. 직전 재직자의 10% 넘게 퇴사면 멈춤 |
| 사이트에서 사라진 노드 | 지우지 않고 `removed`, 관계만 지운다. 라벨별 10% 넘게 사라지면 멈춤 |
| 노드 | 지우지 않는다. 릴리즈에 없는 노드가 DB 에 있으면 멈춘다 |

사라진 것은 셋으로 갈린다 — ① 사이트에서 진짜 없어졌다 ② 수집이 그 페이지를 못 받았다 ③ 파서 규칙이 바뀌어 못 뽑았다.
②③ 에서 지우면 데이터 유실이라 **상태만 바꾸고 대량이면 사람에게 넘긴다.**

### ⑦ 임베딩 — 바뀐 노드만

| 라벨 | 문장 재료 |
|---|---|
| BusinessProject 208 | 사업명 · 계약일 · 기관유형 · 운영 주소 · 발주처 이름 · 유형 · 사업분야 (그래프 이웃을 붙인다) |
| Offering 12 | 이름 · 소개 · 큰 글 |

```
재료 해시 = 라벨 · 사실 · 생성 모델 · 지시문 버전 · 임베딩 모델
저장된 embedding_source_sha256 과 다른 노드만 → gpt-5.6-luna 로 검색 문장 → ada-002 벡터
바뀐 게 없으면 OpenAI 호출 0
```

- **이름 검사** — 문장에 사업명 · 발주처 · 계약일(보유기술은 이름)이 글자 그대로 없으면 빠진 말을 짚어 한 번 더 부르고, 그래도 없으면 사실만 이은 규칙 문장(받침 따라 은/는)
- **쓰기** — 한 트랜잭션. 시작 때와 ReleaseHead 가 같고 재료 해시가 그대로일 때만. 되읽어 해시 · 차원 · 벡터 인덱스 ONLINE 확인
- 문장은 파일이 아니라 DB 에 둔다(`embedding_text`)

지시문 v1 로 220건을 만들어 읽어 보니 발주처를 줄여 쓰고(대한무역투자진흥**공사** → 진흥**사** 2건), 말투가 섞이고, "검색할 수 있습니다" 같은 군더더기가 붙었다.
v2 + 이름 검사로 사업명 · 발주처 · 계약일 208/208 · 합니다체 아닌 문장 0.

---

## 4. 조회 설계 — 라벨마다 찾는 방법이 다르다

질문 흐름: 벡터 검색으로 시작 노드 → 그래프 관계로 넓힘 → text2cypher 보조 → LLM 답변.

| 방법 | 대상 | 이유 |
|---|---|---|
| **벡터** | BusinessProject 208 · Offering 12 | 뜻으로 묻고, 다 줄 수 없는 양 |
| 고정 컨텍스트 | Company · Office · HistoryEvent · BusinessArea (약 3,600자) | 작고 거의 안 바뀜 — 찾기 실패가 없다 |
| 이름 검색 + 그래프 | OrganizationalUnit 33 · Client 81 | 이름이 짧고 비슷해 벡터가 잘 못 가린다 (웹사업팀 · 웹수행팀) |
| 정해진 조회(템플릿) | Employee 212 | 개인정보 · 정확한 조건. 전화번호는 text2cypher 로 답하지 않는다 |
| text2cypher (보조) | 개수 · 조건 · 목록 질문 | 확정본 JSON 을 LLM 에 준다. 읽기 전용 · LIMIT · 시간 제한 |

버린 것 — 회사 · 사무소 벡터(1 · 4개라 찾는 단계가 무의미), 보유기술 조각 노드(새 라벨을 늘리지 않기로), 노드 한 칸에 벡터 여러 개(벡터 인덱스는 노드당 하나), 본문 앞 150자 · LLM 요약(뒤쪽 유실 · 재현 안 됨).

질문 임베딩도 같은 `text-embedding-ada-002` 로 해야 한다.

---

## 5. 운영 적재

### 서버 구성

| 서버 | 역할 |
|---|---|
| `.62` | 운영 서버 — 코드 checkout · 조회 API Docker(운영 6060 / 개발 16060) · 적재 명령 실행 |
| `.64` (내부 `10.20.20.12`) | PostgreSQL + **Neo4j** (bolt 7687). DNS `milvus.chatbaram.com` 이 여기를 가리키지만 이름과 달리 Milvus 서버가 아니다 |
| `.71` · `.58` · `.57` | Milvus · MariaDB · Gitea |

`.env` 의 Neo4j 주소는 내부망이라 작업 PC 에서 닿지 않는다. 외부 주소로는 닿지만 bolt 평문이라 직원 데이터가 인터넷을 지난다 → **서버에서 적재한다.**

### 순서

```bash
cd /home/chat_bot/Ai_Pro_Neo4j_Search
git pull && uv sync --frozen --no-dev

uv run --frozen python -m ingestion.load --output .local/r1-review          # 1. 드라이런 (DB 접속 없음)
docker compose --env-file .env -f docker-compose.yml down                  # 2. API 멈춤
uv run --frozen python -m ingestion.load --output .local/r1 \
    --target production --env-file .env --write                           # 3. 적재 · 확인 · 임베딩
docker compose --env-file .env -f docker-compose.yml up -d --build \
    --wait --wait-timeout 90                                               # 4. API 새 코드로 빌드 · 실행
```

두 번째 적재부터 `--previous <직전 release.json>`. `--output` 은 매번 새 폴더.

**준비** — Neo4j 5.23 이상(조회가 `CALL () {}` 문법) · 나라원 전용 빈 DB · `.env` 에 Neo4j 접속과 `NARA1_OPENAI_API_KEY` · 서버에서 OpenAI 접속.

**확인** — 드라이런 `report.json` 의 `halted` 비었는지 · 적재 출력 `verified: true` · `embedding.verified: true` · API `/health/data` ready.

### 멈췄을 때

| 상황 | 대응 |
|---|---|
| `halted` · 종료코드 1 | `report.json` · `summary.json` 사유 |
| 직전 재직자 10% 넘게 퇴사 | 명단이 맞으면 `--allow-many-left` |
| 사이트 노드 라벨별 10% 넘게 사라짐 | 수집이 맞으면 `--allow-many-removed` |
| 임베딩만 실패 (`embedding.error`) | 같은 명령을 새 `--output` 으로 다시 — 적재는 `already_applied`, 남은 임베딩만 |
| OpenAI 키 없음 | DB 접속 전에 멈춘다. 적재만 할 때 `--skip-embed` |
| `ontology_version_mismatch` (503) | DB 버전이 `SERVING_COMPATIBLE_VERSIONS` 에 없다 — API 를 새 코드로 빌드했는지 |

### 조회 API 점검 (`deployment.md`)

- Compose `network_mode: host` — API 가 6060 / 16060 에서 직접 받는다. `restart: unless-stopped`
- Compose healthcheck 는 **TCP 포트만** 본다 — 통과해도 DB 연결 · 데이터 준비를 보장하지 않는다
- `/health/live` 프로세스 · `/health/ready` DB 인증 후 `RETURN 1` · `/health/data` 조직 노드 존재 · 온톨로지 버전

| reason | 의미 | 다음 확인 |
|---|---|---|
| `neo4j_unreachable` | 연결 경로 없음 | DB IP · Bolt 포트 · 방화벽 |
| `neo4j_auth_failed` | 인증 실패 | 계정 · 비밀번호 |
| `empty_graph` | 조직 노드 없음 | 대상 DB · 적재 |
| `ontology_version_mismatch` | 버전 불일치 | 적재 릴리즈 버전 · 호환 목록 |
| `neo4j_query_failed` | 그 밖의 조회 실패 | DB 권한 · 데이터베이스 이름 · 서버 로그 |

Chatty 는 `NARA1_GRAPH_API_URL` · `NARA1_GRAPH_API_KEY` 로 이 API 를 부르고, Neo4j 주소는 모른다.

### 보관

`release.json` 은 다음 적재의 `--previous` 로 **반드시** 필요하다. 직원 전화번호가 들어 있어 저장소에 넣지 않고 권한을 제한한 곳에 둔다.

---

## 6. 섀도우 확인

로컬 섀도우 Neo4j(5.26.30, 볼륨 없는 전용 컨테이너)를 비우고 운영과 같은 명령으로 돌렸다.

| 확인 | 결과 |
|---|---|
| 빈 DB 에서 한 명령 | 76초 — 노드 584 · 관계 761 · 임베딩 220 (바로 217 / 다시 2 / 규칙 문장 1) · 확인 통과 |
| 같은 명령 다시 | 적재 `already_applied` · OpenAI 호출 0 |
| 키 없음 | DB 접속 전 멈춤 · 스키마도 안 생김 |
| 임베딩만 실패 (가짜 끊김) | 종료 1 · 적재 결과 남음 · 쓴 것 없음 · 다시 치면 남은 1개만 |
| 릴리즈 체인 | 퇴사 1 · 팀 이동 1 → 관계 2개 삭제 · 노드 유지 / 같은 키 복귀 / 옛 릴리즈 재적재 거부 |
| 롤백 | 쓰기 문장 누락 · 없는 노드를 가리키는 관계 · 삭제 뒤 건수 불일치 → 먼저 쓴 ReleaseHead 까지 전부 되돌림 |
| 벡터 보존 | 속성이 바뀐 사업을 다시 써도 벡터가 남음 · 발주처 이름이 바뀌면 그 발주처 사업만 다시 임베딩 |
| 사라진 사업 | `removed` · 관계 0 · 임베딩 대상에서 빠짐 |
| 조회 | 데이터 준비 ready · 조직 소속 질문 정상 |

---

## 7. 가져갈 것

### ① 삭제를 미뤄도 되는 조건 — 조용히 틀리지 않고 멈출 것

지금 코드에 노드 `DELETE` 가 없다. 대신 **적재 전 지문 · 트랜잭션 안 되읽기 · DeletionRequired** 세 겹이 삭제가 필요한 상황에서 적재를 거부한다.
잘못 들어가는 게 아니라 멈추니까 삭제 기능을 나중으로 미룰 수 있었다.

### ② 삭제 대신 상태, 대량이면 사람

퇴사(`left`) · 사이트에서 사라짐(`removed`) 모두 같은 모양 — 노드는 두고 관계만 지우고, 10% 를 넘으면 멈춘다.
명단 파일을 잘못 넣으면 전원이 퇴사 처리되고, 수집이 한 번 실패하면 사업이 통째로 사라진다.

### ③ 릴리즈 밖 칸을 선언에 적는다

벡터처럼 적재 뒤에 따로 채우는 값이 있으면, 적재기의 "DB 가 직전 릴리즈와 같은가" 검사와 통째 교체가 둘 다 깨진다.
선언에 `managed` 로 표시하고 적재기가 **대조에서 빼고 보존**하게 했다.

### ④ 회귀 기준은 매니페스트 해시 하나

매니페스트가 온톨로지 · 수집본 · 결정 표 · 명단 · 추출 규칙 · 그래프 · 리포트를 전부 묶으므로, 드라이런 해시가 같으면 적재할 것이 하나도 안 바뀐 것이다.
리팩토링 · 버전 정리 · 폴더 이사를 할 때마다 이 해시로 확인했다. **운영 서버 드라이런도 같은 해시가 나와야 한다.**

### ⑤ LLM 이 만든 문장은 검사한다

"이름을 바꾸지 마라"를 지시문에 적어도 220건 중 2건이 발주처를 줄였다. 지시문을 고치는 것만으로는 확률이 줄 뿐이다.
**필수 말을 글자 그대로 검사 → 짚어서 다시 → 그래도 안 되면 규칙 문장.** 방법별 개수를 결과에 남겨 규칙 문장이 늘면 보이게 했다.

### ⑥ 한 명령, 단 비싼 실패는 앞에서

적재와 임베딩을 두 명령으로 두면 임베딩을 빠뜨린다(벡터가 비면 반쪽 적재). 합치되, OpenAI 키가 없으면 **DB 접속 전에** 멈추게 했다.
적재만 되고 임베딩이 빈 상태를 만들지 않으려는 것이다. 임베딩만 실패하면 같은 명령을 다시 치면 된다.

### ⑦ 버전은 한 곳, 조회는 목록으로

온톨로지를 올린 조회 배포와 적재 사이에 조회가 503 으로 멈추는 틈이 생긴다. 호환 목록으로 "추가만"인 변경은 끊김 없이 넘긴다.

### ⑧ 개인정보는 출력하지 않는다

명단 확인은 개수 · 모양 · 분류 값으로만 했다(212명 · 휴대폰 212 고유 · 직장번호 형식 분포). 오류 메시지에도 행 번호와 칸 이름만.
`release.json` 은 전화번호가 들어가 저장소 밖에 둔다.

---

## 8. 남은 것

| 항목 | 내용 |
|---|---|
| 운영 첫 적재 | 서버에서 드라이런 → 적재 → API 재빌드 |
| 수집 세대 | 지금은 모든 `snapshot-full*` 을 합쳐 읽어, 새 수집본을 추가하면 같은 주소 다른 내용으로 멈춘다 → 최신 수집본만 읽기 |
| 자동화 | 스케줄 · 버튼 트리거로 수집 → 적재 → 임베딩. 락 · 알림 · 건수 변화량 검사 · 릴리즈 보관소 |
| 노드 삭제 | removed · left 가 오래된 노드 정리 기준 (필요하면 0.4.2) |
| 조회 API | 벡터 검색 · 직원 템플릿 조회 · text2cypher (다른 담당) |
| 보류 데이터 | 인증 21건 · 매출 5개년 |
