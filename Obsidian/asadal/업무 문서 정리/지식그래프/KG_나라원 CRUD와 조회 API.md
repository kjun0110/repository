# KG_나라원 CRUD와 조회 API

> 나라원 지식그래프에 **지금 구현되어 운영 중인 것**만 정리. 계획이 아니라 현재 동작.
> 기준 2026-09-23 · 온톨로지 `nara1-0.6.1` · 릴리즈 순번 2 · 배포 태그 `v0.1.5-app`

---

## 한눈에

```
수집(collector) ──▶ 데이터 폴더 ──▶ 적재(ingestion) ──▶ Neo4j ──▶ 조회 API(:6060) ──▶ Chatty
   사이트 크롤        원문·회차          릴리즈·검증        그래프      guided 검색
```

| | |
|---|---|
| 운영 DB | Neo4j 5.26 · `110.45.147.64:7687` (서버 안에서는 내부주소) |
| 조회 API | `110.45.147.62:6060` — 컨테이너 `ai_pro_neo4j_search-graph-api` |
| 임베딩 | 사내 BGE-M3 1024차원 (`api-aipro.chatbaram.com/em/bge/v1`) |
| 검색용 LLM | `google/gemma-4-26B-A4B-it` (`api-aipro.chatbaram.com/sllm/v1`) |

---

## 그래프 구조 (현재 실적)

**노드 681 · 관계 1,936 · 벡터 220**

| 라벨 | 건수 | 무엇 |
|---|---|---|
| `Company` | 1 | 회사 — 조직·사무소·연혁이 매달리는 중심 |
| `OrganizationalUnit` | 33 | 조직도의 부서·팀 |
| `Employee` | 212 | 직원 (명단 엑셀에서) |
| `BusinessProject` | 208 | 사업 실적 |
| `Client` | 81 | 발주처 |
| `BusinessArea` | 5 | 사업분야 |
| `Offering` | 12 | 보유기술 |
| `Office` | 4 | 사무소 |
| `HistoryEvent` | 27 | 연혁 |
| `Capture` | 97 | **수집원문** — 모든 사실의 출처 |
| `ReleaseHead` | 1 | 지금 적재된 릴리즈 표식 |

| 관계 | 건수 |
|---|---|
| `SOURCED_FROM` | 1,175 — 노드 → 출처 원문 |
| `IN_AREA` | 261 · `MEMBER_OF` 212 · `ORDERED_BY` 208 |
| `SUB_ORG_OF` 33 · `HAS_EVENT` 27 · `OFFERS` 12 · `LOCATED_AT` 4 · `OPERATES_IN` 4 |

### 출처는 노드다

```
(조직·사업·직원 …) ─[:SOURCED_FROM]→ (:Capture {page_url, sha256, saved_path, snapshot_id, fetched_at})
```

- 노드는 **실제로 실려 있던 원문**을 가리킨다. 페이지당 하나(가장 늦게 받은 것)
- 관계의 출처는 속성 3칸(`source_uri`·`source_sha256`·`observed_at`) — 관계는 노드를 가리킬 수 없어서
- **출처 관계가 없는 노드는 적재되지 않는다**
- **관련도 탐색은 `SOURCED_FROM` 을 타지 않는다** — 같은 원문에 실렸다고 서로 관련 있는 게 아니다

---

## CRUD

### C — 만들기

사람이 노드를 직접 만들지 않는다. **원문에서 나온다.**

```
원문(사이트 HTML·엑셀) → 추출 → 후보 → 검증 → 릴리즈 문서 → MERGE
```

- 키는 **내용 해시**다. `nara1:unit:sha256(상위경로+이름)` 처럼. 같은 사실이면 몇 번 돌려도 같은 노드
- 선언(`declaration.py`)에 없는 라벨·속성·어휘는 **거부**
- 출처 없으면 거부

### R — 읽기

두 갈래다.

```
적재기가 읽기   MATCH (n) RETURN labels(n), properties(n)   — 전체 대조용
조회 API 가 읽기  guided 로 만든 Cypher (읽기 전용만)
```

조회는 **읽기 전용 트랜잭션**이고, `EXPLAIN` 으로 read-only 인지 확인한 뒤에만 실행한다.

### U — 고치기

```
MERGE (n:라벨 {키}) SET n = 새 속성      ← 통째로 맞춘다
```

- 달라진 노드·관계만 문장이 나간다
- **릴리즈 밖(managed) 칸은 보존**: 임베딩 문장·벡터·적재 기록. `SET n =` 전에 담아 뒀다 다시 채운다
- 그래서 재적재해도 **벡터가 안 날아간다**

### D — 지우기

```
릴리즈가 manifest.deletions 에 "지울 것" 을 적는다
  ↓
적재기는 그 목록에 있는 것만 DETACH DELETE (맨 마지막 문장)
  ↓
목록에 없는데 DB 에만 있는 노드가 있으면 → 안 쓰고 멈춤
```

**지우는 규칙은 지금 하나뿐**이다.

```python
DELETION_RULES = ((_orphan_capture, "orphan_capture"),)
```

아무 노드도 가리키지 않게 된 `Capture` 만 지운다. 재수집으로 페이지가 바뀌면 출처가 새 원문으로 옮겨 가고 옛 원문이 고아가 된다. (실제로 0.6.1 적재 때 96개가 정리됐다.)

**지우지 않는 것 — 상태로 남긴다**

```
사이트에서 사라짐   site_status = "removed"
퇴사                employment_status = "left"
```

---

## 적재 안전장치

| | |
|---|---|
| **적재 전 지문** | DB 가 직전 릴리즈 그래프와 같아야 쓴다. 다르면 "누가 DB 를 고쳤다" 로 보고 멈춤 |
| **한 트랜잭션** | 쓰기 전체가 하나. 되읽어 릴리즈와 통째 비교, 다르면 롤백 |
| **쓰기 잠금** | `ReleaseHead` 에 락 — 동시 적재 방지 |
| **unchanged** | 그래프·온톨로지가 직전과 같으면 아예 안 쓴다 |
| **이전 상태는 DB 에** | `ReleaseHead.manifest_json` — 릴리즈 파일을 보관·전달할 필요 없음 |

---

## 조회 API

### 엔드포인트

| | |
|---|---|
| `GET /health/live` | 떠 있나 |
| `GET /health/ready` | Neo4j 에 붙나 |
| `GET /health/data` | 적재된 그래프를 받아들일 수 있나 (온톨로지 호환·조직 개수) |
| `POST /v1/graph/search` | **템플릿 검색** — 조직 계층 전용 |
| `POST /v1/graph/search-by-question` | **자연어 질문** (지금 주력) |

**인증** — 헤더 `X-API-Key`. 값이 설정돼 있을 때만 검사한다.

### 템플릿 검색 (`/v1/graph/search`)

```json
{"contract_version": "1", "template_key": "ORG_HIERARCHY",
 "params": {"name": "웹개발팀"}, "limit": 1}
```

정해진 Cypher 하나만 돈다. LLM 을 안 쓴다. **지금은 사실상 예비용**이다.

### 자연어 질문 (`/v1/graph/search-by-question`)

```json
{"contract_version": "1", "question": "웹개발팀은 어느 부서 소속이야?", "k": 3}
```

응답:

```json
{"contract_version": "1", "ontology_version": "nara1-0.6.1",
 "request_id": "…", "retrieval_strategy": "guided",
 "outcome": "ok", "reason": "", "items": [ … ],
 "model_calls": 1, "repair_count": 0, "elapsed_ms": 2431.65,
 "schema_fingerprint": "…", "release_fingerprint": "…"}
```

`outcome` 값: `ok` · `no_match` · `ambiguous` · `unavailable` · `timeout` · `unauthorized`

---

## guided — 질문이 답이 되기까지

```
① 질문 받기
② 공개 스키마 만들기      온톨로지 레지스트리에서 자동 생성 (라벨·속성·관계·어휘·동의어)
③ LLM 에게 계획 요청      "이 질문에 맞는 Cypher 를 만들어라" (스키마를 같이 준다)
④ Cypher 검사            쓰기·프로시저·와일드카드·무한 경로 거부, 라벨은 스키마에 있는 것만
⑤ 읽기 전용 실행          EXPLAIN 으로 read-only 확인 → 한 트랜잭션
⑥ 출처 복구              결과 노드들의 SOURCED_FROM → Capture 를 같은 트랜잭션에서
⑦ 근거 묶음               출처가 하나라도 빠지면 source_gap 으로 거부
⑧ 응답 포장
```

**핵심 — LLM 은 Cypher 를 제안할 뿐이고, 쓰기는 구조적으로 불가능하다.**

### 실행 전후로 버전을 두 번 확인한다

`ReleaseHead` 를 쿼리 **앞뒤로** 읽어서 다르면 `data_changed`. 읽는 중에 적재가 끼어드는 것을 막는다.

그리고 **DB 온톨로지 버전이 API 가 아는 버전과 정확히 같아야** 한다. 다르면 `ontology_version_mismatch` (503).

> ⚠️ 이것 때문에 **릴리즈를 먼저 하고 적재를 나중에 하면 그 사이 질문이 전부 막힌다.**
> 온톨로지 버전이 올라가는 배포는 **적재 먼저, 릴리즈 나중**.

### 벡터 검색

`BusinessProject` · `Offering` 두 라벨에만 임베딩이 있다 (벡터 220개, BGE-M3 1024차원).

```cypher
CALL db.index.vector.queryNodes('nara1_businessproject_embedding_vec', 12, $vector)
```

질문 임베딩도 **같은 서버·같은 모델**이어야 한다.

### 근거 3칸 — Chatty 가 요구하는 것

```json
{"source_record_id": "nara1:capture:…", "source_asset_id": "<원문 sha256>",
 "source_location": "raw/인원현황.xlsx 또는 snapshot 경로",
 "source_url": "https://www.nara1.kr/company/group.htm", "source_sha256": "…"}
```

전부 `Capture` 노드에서 나온다. 명단처럼 웹 주소가 없는 것은 **"내부 파일 자료"** 로 표시된다.

---

## 실제로 되는 질문 / 안 되는 질문

**된다** (2026-09-23 운영에서 확인)

```
웹개발팀은 어느 부서 소속이야?          조직 상향
○○○ 어느 팀이야?                      직원 소속   (출처: raw/인원현황.xlsx)
웹개발팀에 몇 명 있어?                  인원 수
경기도 수원시에서 발주한 사업 알려줘     발주처 → 사업
한국사능력검정시험 … 어디서 발주했어?    사업 → 발주처
홈페이지구축 및 운영 분야 실적 알려줘    사업분야
나라원 보유기술 뭐 있어?                보유기술
나라원 사무소 어디 있어?                사무소
나라원 대표자가 누구야?                 회사 정보
나라원이 수행한 사업이 몇 건이야?        집계
```

**안 된다** — 둘 다 `query_generation_failed`

| 질문 | 원인 |
|---|---|
| `시스템사업부 밑에 무슨 팀 있어?` | LLM 이 `SUB_ORG_OF` **방향을 거꾸로** 씀 → 결과 0건 |
| `나라원 연혁 알려줘` | LLM 이 **문법 틀린 Cypher** 생성 (`ORDER BY` 뒤에 `collect()` 를 이어 붙임) |

**둘 다 그래프·적재는 정상**이다. 연혁 27건도 조직 33개도 멀쩡히 들어 있다. 조회 쪽 프롬프트·스키마 문제.

고칠 방향: ① 스키마에 **관계 방향 설명** 추가 ② 재시도 횟수 `NARA1_GUIDED_MAX_REPAIRS=2` 로

---

## 운영 명령

```sh
# 드라이런 — DB 에 안 씀
uv run --frozen python -m ingestion.load --target production --env-file .env

# 적재 — 쓰기·확인·임베딩
uv run --frozen python -m ingestion.load --target production --env-file .env --write

# 확인만
uv run --frozen python -m ingestion.load --target production --env-file .env --verify

# 재수집 (서버 데이터 폴더에 회차 생성)
NARA1_DATA_DIR=/home/chat_bot/nara1-data uv run --frozen python collector.py refresh

# 수집할 때가 됐나 — 0이면 수집, 1이면 건너뜀
python collector.py due --days 7
```

**서버(.62)에서 컨테이너로 돌릴 때** — 저장소를 `/app` 에 마운트하면 이미지의 `.venv` 를 가려서 실패한다.

```sh
docker run --rm --network host --user "$(id -u devuser):$(id -g devuser)" \
  -e NARA1_DATA_DIR=/data -e PYTHONPATH=/src -e PYTHONIOENCODING=utf-8 \
  -v /home/chat_bot/Ai_Pro_Neo4j_Search:/src:ro \
  -v /home/chat_bot/nara1-data:/data -w /src \
  ai_pro_neo4j_search-graph-api \
  /app/.venv/bin/python -m ingestion.load --target production --env-file .env --write
```

### 적재 결과 보는 법

```json
{"release_seq": 2, "unchanged": false, "database_written": true, "verified": true,
 "deleted": 96, "recorded": "2026-09-23T06:25:55Z",
 "retention": {"runs_removed": [], "blobs_removed": 0, "cache_removed": 0},
 "embedding": {"generated": 0, "verified": true}}
```

| | |
|---|---|
| `verified` | 되읽기 대조 통과 |
| `deleted` | **지운 노드 수** — 적재에서 유일하게 되돌릴 수 없는 동작 |
| `embedding.generated` | 0이면 벡터 재사용 (내용이 안 바뀜) |
| `recorded` | `ReleaseHead.last_success_at` — 며칠째 안 바뀌면 고장 |

---

## 파일 관리

```
저장소            snapshots/snapshot-full*  첫 수집본(기준점) · uploads/  명단
                  → 배포가 git checkout -f 로 덮어써서 여기에 쌓으면 안 된다
데이터 폴더        <NARA1_DATA_DIR>/snapshots/{runs,blobs} · uploads · cache/extract
   보관 규칙       완주 회차 4개 · 끊긴 회차 7일 · 업로드 kind 당 3개
                  원문은 남은 회차나 그래프가 가리키는 것만 · 캐시는 파싱에 쓰이면 남김
                  적재가 끝까지 성공한 직후에만 정리 · 저장소는 안 건드림
```

---

## 아직 안 된 것

```
스케줄 자동 실행        (수동 실행까지만 확인)
서버에서 수집 실행       (PC 에서만 해봄)
조직 하향 · 연혁 질문    조회 쪽 문제
퇴사자·사라진 노드 삭제  보관 기간 규칙 미정
```

---

## 관련

- [[KG_나라원 온톨로지 그래프 설계]]
- [[KG_나라원 온톨로지 기반 KG 적재 파이프라인]]
- [[KG 수원-나라원 파이프라인 비교]]
- [[KG_온톨로지 기반 KG 생성 시 고려할 점]]
