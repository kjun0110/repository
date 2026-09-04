## 개념

[[0825_KG - 법령 계층·표 복원·대상층 축]]까지 만든 그래프를 **기존 챗봇 챗봇에 붙여 하이브리드 RAG를 실측**하는 계획. 프론트에 토글을 달아 "기존 RAG만" vs "기존 RAG + 지식그래프"를 같은 질문으로 비교한다.

**대전제 — Chatty_Project는 한 줄도 수정하지 않는다.** 본 프로젝트라 임시 테스트로 건드리면 안 된다. 기존 챗봇을 라이브러리처럼 import해서 바깥에서 라우트를 얹고, 나중에 `bridge/`·`api/` 폴더만 지우면 흔적이 사라진다.

이 문서는 **코드 작성 전에 기존 챗봇 내부를 읽어서 전제를 검증한 결과**다. 초안 계획의 전제 중 셋이 틀렸고, 예산 제약 셋이 빠져 있었다.

| 초안 전제 | 실제 |
|---|---|
| KG 결과가 `{websearch_result}` 칸에 들어간다 | **틀림.** `{source}` 칸에 들어간다 |
| 웹검색 칸에 값이 있으면 출처가 안 나온다(함정) | **해당 없음.** 그 칸이 항상 비어 있어 규칙이 발동 안 함 |
| 검색 로직 무수정으로 `context_blocks` 전달 | **불가.** `search()`가 밖으로 안 내보냄 |
| 점수를 섞으면 안 된다 | **저절로 지켜짐.** 합치는 코드가 문자열 이어붙이기 |

---

## 1. 주입 지점 — 코드로 확인한 것

### 유일하게 비어 있는 슬롯

```
app.py:2105  stream_ai_response_with_context(..., chatty_quick_result_task, ...)  6번째 인자
app.py:1756  /ask 가 여기 None 을 넘기고 있음                                     비어 있는 확장 슬롯
```

### KG 결과가 실제로 흐르는 경로

```
app.py:2530  _await_chat_public_results(task, timeout=2.0)      결과 수거
app.py:2664  websearch_source_items = filtered_results          "웹검색 자리"에 앉힘
app.py:3161  _limit_items_by_budget(max_items=5, chars=6000)
app.py:3166  _trim_items_text_fields(max_chars=1400)            항목마다 절단
app.py:3193  answer_web_items = websearch_source_items[:3]      3건만 생존
app.py:3246  build_web_evidence_text()  =  "title content url"
app.py:3265  aggregated_content = private_context + web_parts   ★ 합쳐지는 곳
app.py:3449  inputs["source"] = aggregated_content
app.py:3461  chain.astream(inputs)
```

**합치는 로직은 새로 만들 게 없다.** `app.py:3265`가 원래 "기존 RAG + 웹검색"을 합치라고 있던 자리고, 우리는 웹검색 자리에 KG를 밀어넣을 뿐이다. 토글 OFF는 6번째 인자가 `None`이면 되므로 별도 경로가 필요 없다.

### `websearch_answer`는 KG로부터 채워지지 않는다

```
app.py:2613  websearch_answer = ""
app.py:2632  elif use_websearch=="Y" and not is_weather and len(chatty_quick_result)>0:
app.py:2664      websearch_source_items = filtered_results      <- answer 를 안 건드림
app.py:3451  "websearch_result": websearch_answer or ""         <- 항상 ""
```

`prompts/main_prompt.txt:42`의 "`{{websearch_result}}` 에 값이 있을 경우 절대 출처를 표시하지 않습니다"가 발동하지 않는다. **초안의 함정 ②는 존재하지 않는다.**

### 대신 새 리스크 — 칸 이름

```
main_prompt.txt:9   학습된 내용:      {retriever}       <- 답변규칙 34행이 "최우선 참고"로 지목
main_prompt.txt:12  학습된 내용 출처:  {source}          <- KG가 여기 들어감. 지목 없음
```

기존 RAG 청크는 `{retriever}`와 `{source}` **양쪽에** 들어가고 KG는 `{source}`에만 들어간다. 구조상 **KG가 보조 근거 취급**을 받는 배치다.

---

## 2. 두 슬롯 비교

`retriever` 쪽 예산 제한은 `if use_gemma_context_budget:`(`app.py:3111`) 안에만 있다. 일반 모델이면 그 코드가 실행되지 않는다.

```
app.py:3187  answer_rag_chunk_limit = 5 if use_gemma_context_budget else len(retriever)
```

| | `quick_result` 슬롯 | `retriever` 패치 |
|---|---|---|
| 개수 제한 | **3건** (무조건) | 없음 (일반 모델) |
| 글자 제한 | **1400자/건** (무조건) | 없음 |
| 프롬프트 칸 | 학습된 내용 출처 | **학습된 내용** |
| 답변규칙이 지목 | ✗ | ✓ |
| 기존 `/ask` 영향 | **0** (원래 빈 슬롯) | 분기 처리 안 하면 오염 |
| 기존 챗봇 소스 수정 | 없음 | 없음 (런타임 교체) |

`retriever`는 `app.py:49`에서 모듈 레벨로 import된 이름(`from chat.faiss_process import vector_search`)이라 `app.vector_search`를 런타임에 갈아끼울 수 있다. 소스는 안 바뀌고 메모리에서만 바뀐다.

**양쪽 동시 사용도 가능하다** — 본문은 `retriever`, URL 달린 출처 항목은 `quick_result`. 서로 다른 칸이라 충돌하지 않고, `retriever` 항목이 출처 검증(`is_validated_source_result`: 질문 키워드가 항목에 전부 있어야 함)에 걸려 출처에 안 뜨는 약점을 메운다.

---

## 3. 실측 — 타임아웃이 최대 리스크

기존 챗봇은 KG 결과를 **2초까지만** 기다리고 초과하면 `task.cancel()` 후 빈 리스트로 진행한다(`app.py:2523`, `CHAT_PUBLIC_JOIN_TIMEOUT_SECONDS`).

```
전입신고 어디서 해            vector        3.54s   hits 5
소파 버리려면 스티커 얼마야    vector        5.88s   hits 5
청년 대상 정책 알려줘         civil_filter  0.10s   hits 22
온라인으로 신청할 수 있는 민원 전부  civil_filter  0.03s   hits 88
```

**벡터 경로는 현재 전부 취소된다.** 느린 원인은 `pipeline.py:324`의 `synthesize_answer()` — gpt-4o-mini 호출이다. 기존 챗봇에 넘길 건 완성된 답변이 아니라 원재료(`context_blocks`)라 부를 이유가 없다.

둘 다 기존 챗봇 무수정으로 해결된다.
- `CHAT_PUBLIC_JOIN_TIMEOUT_SECONDS`를 10으로 (환경변수)
- KG 자체 GPT 호출을 끄면 3s 절감

---

## 4. 집계 경로의 두 함정

### URL 없는 항목은 통째로 버려진다

```
app.py:2655  identifier = url or path;  if not identifier: skip
app.py:2955  if not url: continue
```

그런데 집계 경로는 이름만 돌려준다.

```
pipeline.py:84  result["hits"] = [{"name": n} for n in names]     url 없음
```

→ **지금 상태로 붙이면 "청년 대상 정책"은 KG 결과가 0건으로 도착한다.** 토글 on/off가 같은 답을 내서 A/B 비교 자체가 성립하지 않는다. 대표 URL을 붙여야 한다.

### 22건을 22개 항목으로 쪼개면 3건만 남는다

`quick_result` 슬롯의 3건 제한 때문이다. 목록 전체를 **항목 1개**에 담아야 한다. 가장 긴 목록(88건)도 `answer` 기준 1074자라 1400자 안에 들어간다.

### `context_blocks`가 밖으로 안 나온다

```
pipeline.py:314  context_blocks = []                    지역 변수
pipeline.py:324  result["answer"] = synthesize_answer(...)   result 에 안 담김
```

게다가 early return 4곳(`47`/`77`/`85`/`339` — 리더·조직목록·집계·재시도)은 `context_blocks`를 **아예 만들지 않는다.**

---

## 5. 환경 — 확인 결과

서버는 전부 열려 있다.

```
PostgreSQL(main)     110.45.147.64:5432    OPEN
PostgreSQL(chatty)   110.45.147.71:5432    OPEN
MariaDB(.env)        110.45.146.156:3306   OPEN
MariaDB(app_local)   110.45.147.58:3306    OPEN
Milvus search        110.45.147.71:8001    OPEN
Milvus insert        110.45.147.71:8000    OPEN
```

| 항목 | 상태 |
|---|---|
| `HMAC_SECRET_KEY` | 있음 (`.env` 42행, `=` 앞 공백 주의) |
| MariaDB 계정 | 후보 있음 (`.env` `MYSQL_*`). 호스트가 `app_local` 기본값과 다름 — 확인 필요 |
| PostgreSQL 비밀번호 | **없음** ← 유일한 미지수 |
| `config.py` | 없음 (`config.example.py`만. `.gitignore` 대상) |
| 기존 챗봇 의존성 | **미설치** (venv 없음, DB 드라이버 0개) |

### 로컬 인덱스 데이터는 불필요

`FAISS_INDEX/`·`account_settings/`가 없어서 한때 블로커로 봤으나 **오진이었다.** `app_local.py:31`이 RAG 백엔드를 `refactored`로 강제하는데, 이 경로는 로컬 FAISS를 안 쓴다.

```
faiss_process.py:2042  backend = "refactored" (기본값)
  -> chat/private_rag/facade.py
  -> ai_search/vector/service.py     벡터: Milvus(원격) / 키워드·메타: MariaDB(원격)
```

`Config.FAISS_INDEX_DIR`은 legacy 경로에서만 쓰인다. 코드에 `faiss_score` 같은 이름이 남아 있는 건 계약 유지용 필드명이다. `account_settings`의 SQLite는 없으면 자동 생성된다.

### HMAC

```
security.py:56   message = f"{chat_bot_id}:{timestamp}"
                 헤더 X-HMAC-Signature
```

`timestamp`는 body 필수 필드인데 `AppRequest`에 없다. 다만 `extra="forbid"`가 없어 Pydantic이 무시하므로 그냥 넣으면 된다. 타임스탬프 만료 검사는 주석 처리돼 있다(`security.py:44-53`).

### chat_id

요청에 담는 값이 아니라 DB 조회다.

```
app.py:1303  get_chat_id_from_db(db_name, chat_bot_id)
  -> utils/whoami.py:392 -> execute_query -> PostgreSQL (asyncpg)
```

---

## 6. 작업 순서

**전략: 2단계.** 1단계로 배관만 뚫고 2단계에서 본 측정을 한다. 한 번에 `retriever` 패치로 가면 실패 시 KG API·변환·패치 중 어디가 문제인지 못 가른다. 반대로 1단계에서 끝내면 3건/1400자로 목 졸린 상태라 KG를 불리하게 측정한다.

### 0. 준비 (병렬)

- `venv` + `pip install -r requirements.txt` 백그라운드로 시작 (77줄, 시간 걸림)
- 설치 후 `MYSQL_PASS`로 MariaDB 접속 테스트 → 같은 값으로 PostgreSQL(`chatty_search`)도 시도
- `config.py` 생성 (`config.example.py` 복사)
- **`/ask` 단독 동작 확인** — 이게 되고 나서 KG를 붙여야 나중에 문제 원인을 가릴 수 있다

### 1. `search/pipeline.py` 수정

- `324` 근처: `result["context_blocks"] = context_blocks` (한 줄)
- `47`/`77`/`85`/`339`: early return 4곳은 받는 쪽에서 "없으면 `answer` 사용"으로 처리
- `324`: `synthesize_answer()` 끄는 스위치 — **필수**
- `requirements.txt`에 `fastapi==0.115.9` / `uvicorn==0.34.0` / `httpx==0.28.1`

### 2. `api/server.py` (:9020)

- lifespan으로 GraphClient·OpenAI 재사용
- `POST /kg/search` · `GET /health`
- **`pipeline.search()`는 동기 함수** — `async def`에서 직접 부르면 이벤트 루프가 막힌다. `def` 엔드포인트나 스레드풀
- 응답에 `route`와 소요시간 포함 (디버그 패널용)
- **`curl`로 단독 검증하고 넘어갈 것.** 0단계와 무관하게 여기까지 간다

### 3. `bridge/kg_bridge.py`

경로별 분기:
- **집계(`civil_filter`)**: `answer` 전체를 항목 1개 + **대표 URL 필수**
- **벡터**: `context_blocks`를 항목으로, 1단계에선 최대 3개
- 본문은 `content` 키에 (`app.py:3257`이 이 키를 봄)
- 실패 시 예외 대신 빈 리스트

### 4. `bridge/app_kg.py` — 1단계 (:9010)

```
sys.path.insert -> import app_local -> from app import app, stream_ai_response_with_context, AppRequest
```

- KG 작업을 먼저 띄우고 6번째 인자로 전달
- `CHAT_PUBLIC_JOIN_TIMEOUT_SECONDS = 10`
- body에 `use_websearch:"Y"`, `use_rag:"Y"`, `timestamp`
- 전처리 최소화 — chat_id / 언어감지 / 설정 / 대화이력 4개만. 첨부·유해성·추천질문·인용 생략

**판정:** 로그의 `gather 완료: ... 개수=N`에서 N > 0이면 배관 성공. 답변 품질은 아직 보지 않는다.

### 5. `bridge/static/kg_test.html`

- 토글 → `/ask` 또는 `/ask_kg`, SSE 수신
- 디버그 패널: route / KG 항목 수 / 첫 토큰 지연 / 총 시간
- **`model_name` 고정** — Gemma 폴백으로 넘어가면 예산 조건이 바뀌어 측정이 오염된다

### 6. `retriever` 패치 — 본 측정

- **요청별 분기를 먼저 넣을 것.** 같은 프로세스라 그냥 패치하면 `/ask`(토글 OFF)도 오염돼 A/B가 무너진다. contextvar로 표시하고 없으면 원본 그대로 반환
- **병렬성 유지.** 패치 안에서 KG를 호출하면 FAISS 뒤에 줄 서서 첫 토큰이 늦어진다. 작업을 미리 띄우고 결과만 받아온다

### 7. 비교 측정

기존 32문항 + 추가분. 토글 on/off로 같은 질문.

- **집계** — "청년 대상 22건"이 22건으로 나오나 *(1단계에선 구조상 안 나옴. 2단계에서만 의미 있음)*
- **지시문 생존** — 전입신고에서 "거주지 관할 구" 안내가 살아남나
- **출처** — `source_url`이 표시되나
- **첫 토큰 지연** — on/off 차이

---

## 7. 파일 목록

```
suwon_graph/
  search/pipeline.py          수정  (context_blocks 노출 + synthesize 스위치)
  requirements.txt            수정  (fastapi/uvicorn/httpx)
  api/__init__.py             신규
  api/server.py               신규  :9020
  bridge/__init__.py          신규
  bridge/kg_bridge.py         신규
  bridge/app_kg.py            신규  :9010
  bridge/static/kg_test.html  신규

Chatty_Project/
  config.py                   신규  (gitignore 대상, 기존 파일 수정 0)
```

초안은 신규 4개였으나 실제로는 8개다. "`bridge/` 폴더만 지우면 흔적이 사라진다"는 정확히는 `bridge/` + `api/`이고, `pipeline.py`의 몇 줄은 남는다 — 기존 동작을 안 바꾸는 추가라 무해하다. **Chatty_Project는 계획대로 완전히 원상태다.**

---

## 재사용 가능한 판단

- **"어느 슬롯에 넣느냐"가 예산과 프롬프트 위상을 동시에 결정한다.** 같은 데이터를 `{source}`에 넣으면 3건/1400자에 보조 근거 취급이고, `{retriever}`에 넣으면 무제한에 주 근거다. 붙이기 전에 받는 쪽 예산을 먼저 읽어야 한다.
- **호출 계약만 보고 데이터가 어디로 흐르는지 단정하지 말 것.** `chatty_quick_result`를 "웹검색 결과"에 넣는 분기가 있길래 `{websearch_result}`로 간다고 봤으나, 그 분기는 `source_items`만 채우고 `answer`는 `""`로 남긴다. 변수를 끝까지 따라가야 했다.
- **초안의 제약이 실제 제약이 아닐 수 있다.** 넷 중 둘(출처 함정, 점수 혼합)은 코드를 읽으니 존재하지 않았고, 대신 초안에 없던 셋(3건 제한, URL 필수, 2초 타임아웃)이 진짜 제약이었다.
- **"없다"고 판단하기 전에 이름 표기를 의심할 것.** `HMAC_SECRET_KEY`는 `.env`에 있었는데 `=` 앞 공백 때문에 `^KEY=` 검색이 놓쳤다. dotenv는 정상적으로 읽는다.
- **파일이 없다고 데이터가 없는 건 아니다.** `FAISS_INDEX/` 부재를 블로커로 봤으나 실제 백엔드는 원격 Milvus였다. 코드에 남은 옛 이름(`faiss_score`)에 속았다.
- **미리 우회로를 만들지 않는다는 원칙과, 측정된 벽에 맞추는 것은 다르다.** 집계 경로의 URL·1항목 압축은 우회로가 아니라 실측된 제약(URL 없으면 폐기, 3건 컷)에 대한 대응이다. 안 넣으면 KG 결과가 0건이라 비교 대상 자체가 없다.

---

## 아직 확인 안 된 것

- **PostgreSQL 접속정보** — 유일한 진짜 블로커. `MYSQL_PASS`와 같을 가능성이 있으나 드라이버 미설치로 검증 못 함
- **MariaDB 호스트 불일치** — `.env`는 `110.45.146.156`, `app_local.py:72` 기본값은 `110.45.147.58`. 같은 계정이 양쪽에 통하는지 미확인
- **집계 질의가 실제로 살아남는지** — 3건 제한을 우회해도 `{source}` 칸이 프롬프트상 지목받지 못하는 문제가 남는다. 6단계 이후에야 판정 가능
- **인용(citation) 동작** — `retriever` 경로로 옮기면 KG 블록에 각주가 붙는다. 근거 추적이 되는 이득인지 노이즈인지 실측 필요
- **출처 검증 통과율** — `retriever` 항목은 질문 키워드가 전부 들어 있어야 출처에 뜬다. KG 블록이 길어 대체로 통과할 것으로 보이나 미측정

---

## 관련 문서

- [[0825_KG - 법령 계층·표 복원·대상층 축]]
- [[0821_KG - Layer 2 전 분야 확장 및 검색·데이터 구조 정리]]
- [[0821_KG - suwon_graph 계획 (RFP 대조 및 데이터소스 조사)]]
- [[0818_KG - suwon_graph 검색 파이프라인 및 Opik 연동]]
