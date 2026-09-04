## 개념

- **원천(source)** — 페이지 텍스트를 어디서 얻는가. 직접 크롤링 / Chatty pgvector 두 갈래
- **청크(chunk)** — pgvector가 벡터를 붙이는 단위. 페이지를 쪼갠 조각
- **출처층** — 개체가 어느 페이지에서 나왔는지 기록하는 `:Page` 노드
- **승자 규칙** — 두 원천이 같은 개체를 만들었을 때 어느 값을 남길지 정하는 규칙

##### 이 문서가 다루는 범위

[[KG - 온톨로지 데이터 투입 액션]]의 **A1(원천 읽기) 한 자리**를 무엇으로 채울지를 담는다. 그 아래 B~F 단계는 원천과 무관하게 동일하다.

##### 두 원천의 성질

```
직접 크롤링    HTML 을 받는다
              표 병합셀 348 · <br> 구분자 · 담당자 footer · th/td 가 살아 있다
              → 규칙 추출 5패스(표·정규식·명시목록)가 가능하다
              → 민원·조직도는 이 구조가 곧 답의 근거다

pgvector      파싱된 텍스트를 받는다
              구조가 평탄화돼 위 신호가 사라진다
              → LLM 추출 5패스만 남는다
              → 제목·부서·날짜면 되는 것(게시글·공지)에는 충분하다
```

##### 크롤링이 아니면 못 보는 것 — 2026-09-02 실측

민원 안내 페이지의 첨부가 그 예다.

```
civil_service.json 644건의 파일 관련 필드    없음
본문·필드에 파일 흔적                        0건
```

크롤러가 그 자리를 안 봤으니 있는지조차 몰랐다. 페이지를 열어 보니 원인이 둘이었다.

```
① 원본 href 가 /webcontent/ckeditor/2025/3/10/5fb8c09a-… 경로다
② 페이지의 JS 가 실행 중에 그걸 ND_fileDownload.do?id=… 로 바꾼다
   <script> 주석: "본문 첨부파일 다운로드 경로 수정"
```

**정적 HTML 에서 `fileDownload` 만 찾으면 0건이 나온다.** 표본 18건 중 3건이 본문에 파일을 갖고 있었다(개방화장실 현황 2 · 감리원 배치신고 2 · 선정대리인 제도 1).

pgvector 는 파싱된 텍스트라 이 링크가 애초에 없다. **크롤링이어야 볼 수 있고, 그것도 JS 를 알아야 볼 수 있다.**

##### 시민 서식의 본진은 게시판이었다

"서식은 민원 안내 페이지에 있다"고 단정했다가 틀렸다. 실제로는 **서식 전용 게시판 3개**에 있었고 그것을 안 긁고 있었다.

```
1026  민원서식자료실       469건   2018-02-22 ~ 2026-08-30   담당부서 칸 O
1298  민원서식             26건
1275  옥외광고물 관련서식    12건   등록일 칸 없음
                        ─────
                          507건
```

**규모를 재기 전에 원인을 말하지 않는다** 는 것이 여기서 또 확인됐다. 페이지 4개를 열어 보고 "게시판 밖에 있다"고 했는데, 실은 "본 적이 없다"였다.

---

## 원천이 pgvector일 때

Chatty 가 웹사이트를 학습해 넣어 둔 테이블(`td_*_training_data`, `content_type='url'`)을 원천으로 쓰는 경우다. 코드(`edu/url_edus/storage.py`)로 저장 구조를 확인했다.

##### 저장 단위는 페이지가 아니라 청크다

한 행 = 한 청크. 페이지 하나가 여러 행으로 쪼개져 있다.

```
content           원본 URL              같은 페이지면 같음
chunk_num         "1" "2" "3"           문자열이다
text_data         청크 본문
web_title         페이지 제목
subject           주제
content_type      "url"
embedding         vector(1536)
content_metadata  jsonb {
                    source_url, chunk_index,
                    content_hash,        ← 원본 페이지 해시
                    content_length, last_check, update_frequency
                  }
```

`storage.py` 주석에 명시돼 있다 — **"원본 페이지 해시를 모든 청크에 통일 적용(변경감지 목적)"**. 즉 `content`(URL)와 `content_hash` 가 페이지 단위 식별자 노릇을 하므로 청크를 페이지로 되묶을 수 있다.

헤더(source·chunk·title)가 `text_data` 본문에 섞이지 않고 **별도 컬럼**이라, 추출 LLM 이 헤더를 내용으로 오인할 위험은 없다.

##### 청크는 노드가 되지 않는다

```
청크    검색 단위. 벡터가 붙는 자리        → pgvector 소관
페이지  출처. 변경 감지 단위               → :Page 노드
개체    민원·조직·서류·게시글              → :Entity 노드
```

청크를 노드로 만들면 수만 개가 되고 **pgvector 가 이미 하는 일을 그래프에서 또 하는 것**이 된다. 청크 텍스트는 추출의 재료로만 쓰고 버린다.

##### 출처층을 하나 세운다

```
(:Page {url, content_hash, chunk_count, title, subject, last_check, source})
      :Entity 없음 · 임베딩 없음
   ▲
   │ DERIVED_FROM
   │
(:CivilService) (:DocumentType) (:Place) (:Regulation) …
```

이것이 온톨로지 액션 중 **전파**에 해당한다. 페이지 해시가 바뀌면 거기서 파생된 노드만 골라 다시 추출하면 된다.

```cypher
MATCH (p:Page {url: $url}) WHERE p.content_hash <> $new
MATCH (e)-[:DERIVED_FROM]->(p)
RETURN e
```

지금은 이 연결이 없어서 무엇이 바뀌든 전부 다시 돌린다.

##### 파이프라인

```
1  읽기        SELECT content, chunk_num::int, text_data, web_title,
                      content_metadata->>'content_hash'
               WHERE content_type='url' ORDER BY content, chunk_num::int
                     ※ ::int 캐스팅 필수. 문자열이라 "10" < "2" 로 깨진다
2  페이지 복원   content 로 묶고 chunk_num 순으로 이어붙임          ★ 새 단계
3  변경 감지    page_hash vs :Page.content_hash
                 같음 → 종료(LLM 0회) / 다름 → 재추출 / 없음 → 신규
4  라우팅       URL 패턴 + web_title + subject
                     ※ HTML 이 없으므로 B단계 2층(구조) 신호는 못 쓴다
5  추출         LLM
                     ※ 규칙 추출 5패스(표·정규식)는 여기서 불가
6  검사·적재·추론·계층   기존과 동일
               + (개체)-[:DERIVED_FROM]->(:Page)
               + source='pgvector'
```

##### 합쳐지는 지점 — 두 원천이 같은 개체를 만들면

`civil_id` 가 URL 해시라 크롤링과 pgvector 가 **같은 키를 만든다.** MERGE 이므로 자동으로 한 노드가 되는데, 문제는 어느 값이 남느냐다.

| 필드 | 크롤링 | pgvector |
|---|---|---|
| 구비서류 | 표에서 규칙 추출 | 텍스트에서 LLM |
| 담당부서 | footer 에서 | footer 가 없어 못 뽑음 |
| 표 데이터 | 3,623행 (병합셀 348 복원) | 평탄화돼 사라짐 |

**나중에 돈 쪽이 앞의 값을 덮어쓴다.** `main.py` 가 순차 실행이라 순서에 따라 답이 달라진다. 그래서 승자 규칙이 필요하다.

```
구조가 필요한 필드(구비서류·표·담당부서)   →  크롤링이 이긴다
구조 없이도 되는 필드(제목·본문·분야)      →  최신이 이긴다
한 원천만 갖는 필드                       →  그대로
```

##### 더 나은 길 — 원천을 겹치지 않게 나눈다

```
민원 안내 · 조직도    크롤링만        표·footer·<br> 가 답의 근거
게시글 · 공지         pgvector 가능   제목·부서·날짜면 충분
```

**겹치지 않으면 승자 규칙 자체가 필요 없다.** 제일 단순하고 안전하다.

##### 갱신·폐지

```cypher
-- 테이블에서 사라진 URL 을 폐지할 때
WHERE p.source = 'pgvector' AND p.last_seen <> $batch
```

`source` 가 없으면 pgvector 배치를 돌릴 때 크롤링에서 온 노드까지 폐지된다. **원천이 둘이 되는 순간 `source` 는 필수다** — 단일 원천이라 안 넣기로 한 결정(2026-09-01)이 여기서 뒤집힌다.

##### 아직 확인 못 한 것 (DB 접속 필요)

```
1. 청크 재조립이 무손실인가          경계가 표·문장을 잘랐으면 원문과 다르다
2. 표 구조가 남아 있나               평탄화됐으면 민원·조직은 재현 불가
3. 삭제된 페이지가 테이블에서 사라지나   폐지 감지 가능 여부
4. 페이지당 청크 수                  정렬 문제의 실제 영향 범위
```

**1·2가 되면 pgvector 경유가 열리고, 안 되면 게시글류만 가져오는 것이 맞다.**

---

## 왜 import이 아니라 복사인가

##### `shared.py`가 import 시점에 하는 일

```python
ssl._create_default_https_context = ssl._create_unverified_context
os.environ["PYTHONHTTPSVERIFY"] = "0"
aiohttp.ClientSession._build_ssl_context = lambda self, *a, **k: False

logger = LoggerSingleton.get_logger(...)
embedding_model = OpenAIEmbeddings(openai_api_key=Config.OPENAI_API_KEY)
```

**SSL 검증이 프로세스 전역에서 꺼진다.** suwon_graph의 requests·Neo4j·OpenAI 호출까지 영향을 받고 조용하다. 기능 문제가 아니라 보안 문제다.

##### 모듈별 shared 의존

| 모듈 | 줄수 | shared |
|---|---|---|
| content_hash.py | 22 | 0곳 |
| url_normalizer.py | 648 | 0곳 |
| page_classifier.py | 1,535 | 0곳 |
| learn_list_id.py | 461 | 0곳 |
| category.py | 380 | 0곳 |
| rejection.py | 276 | 0곳 |
| url_utils.py | 339 | 1곳 |
| change_detection.py | 288 | 1곳 |
| breadcrumb.py | 102 | 1곳 |

##### 결정적 사실
가져올 파일들이 `shared`에서 실제로 쓰는 심볼은 **`logger` 하나뿐**이다. `embedding_model`·`DB_BULK_SIZE` 같은 무거운 심볼은 `storage.py`·`batch.py`만 쓰는데 둘 다 가져오지 않는다. **복사 비용은 `import logger` 세 줄이다.**

##### 함정
`page_classifier`는 `shared`를 안 쓰지만 `url_utils`를 import하고, `url_utils`가 `shared`를 쓴다. import 경로로는 결국 딸려온다.

```
page_classifier → url_utils → shared → Config · OpenAI · SSL 무력화
```

##### DB 의존 모듈은 3개뿐
`storage.py` · `batch.py` · `batch_with_embedding.py`. 나머지 20개는 DB에 붙지 않는다. `change_detection`은 `get_url_metadata()` 안에서만 `db_config`를 부르므로 그 함수만 갈아끼우면 된다.

##### 출처 표기
이식한 파일 맨 위에 원본 경로 · 이식 날짜 · 원본 대비 변경점을 적는다. 복사의 유일한 단점(원본 갱신을 못 따라감)을 문서로 메운다. 이 프로젝트가 이미 그렇게 하고 있다 — `ontology/classify.py`도 "suwon_label에서 실측으로 검증된 분류 규칙을 그대로 이식한다"로 시작한다.

---

---

## Chatty 수집 계층을 가져온다면 — 복사 대상

##### 복사 대상 (3,108줄)

```
crawl/
  content_hash.py       22줄   그대로     sha256(공백정규화 파싱텍스트)
  url_normalizer.py    648줄   그대로
  url_utils.py         339줄   logger 1줄
  page_classifier.py  1535줄   도메인 예외 → suwon.go.kr
  change_detection.py  288줄   logger + get_url_metadata → Neo4j
  rejection.py         276줄   그대로     에러 페이지 판별
```

##### 손대는 곳 5군데

1. `logger` 3파일 — `from .shared import logger` → 자체 logger
2. `page_classifier`의 도메인 예외 — `chatty.kr` → `suwon.go.kr`
3. `change_detection.get_url_metadata()` — PostgreSQL → Neo4j `content_hash`. **이 함수 하나만.** NEW/NO_CHANGE/CHANGED 판정 로직은 그대로 산다
4. `change_detection`이 부르는 `parsing.build_structured_page_result` — 우리 파서로 연결. **여기가 접합부**
5. import 상대 경로 정리

##### 증분 임베딩이 여기 딸려온다
지금은 매번 5,909개를 전부 재계산한다. `embedding/*`의 `WHERE n.embedding IS NOT NULL`은 개수를 세는 용도일 뿐 건너뛰는 로직이 없다.

##### 안 가져오는 것

```
parsings/* 7종     우리 파싱 유지
storage · batch*   pgvector/Milvus 의존
shared             SSL 전역 무력화
playwright_fetcher · browser_pool   JS 페이지를 만나면 그때
```

##### 검증
644건 재수집 → 기존 JSON과 필드 단위 **불일치 0** · 2회 연속 실행 시 CHANGED 0건 · 검색 20문항 부작용 0

---

---

## 관련 문서

- [[KG - 온톨로지 데이터 투입 액션]] — 원천 뒤에 이어지는 A2~F 단계
- [[KG - 온톨로지 선언 파일 (TBox)]] — 원천이 둘이 되면 선언에 source·승자 규칙이 들어간다
- [[지식그래프 진행 상황]] — 전체 진행 상황과 남은 단계
