## 개념

- **`manifest_json`** — `ReleaseHead` 노드에 함께 쓰는 그 릴리즈의 매니페스트(정렬된 JSON, 약 2KB, 개인정보 없음). 다음 적재가 직전 상태를 파일이 아니라 DB 에서 읽는 근거
- **`read_previous`** — DB 그래프와 `ReleaseHead` 기록을 읽어 직전 릴리즈를 되살리는 함수. 기록과 그래프가 서로 맞는지 검증한 뒤에만 돌려준다
- **`unchanged`** — 새로 만든 릴리즈의 `content_sha256` 과 온톨로지 해시가 직전과 같아 DB 에 쓰지 않는 결과. 임베딩 점검은 그대로 돈다
- **`X-Nara1-Mode`** — 관리자 AI 화면이 "Graph 활용" 토글일 때 Chatty `/ask` 요청에 붙이는 헤더. 값은 `hybrid`
- **`nara1.graph_trace`** — Chatty 가 답 스트림에 끼워 보내는 SSE 이벤트. 프론트는 이게 와야 "그래프 적용"을 표시한다
- **`NARA1_GRAPH_CONFIG.askUrl`** — 그래프 모드 요청만 보낼 Chatty 주소. `chatting_graph.htm` 에 하드코딩돼 있다

## 요약

[[0916_KG - 나라원 · 운영 첫 적재 완료, 그래프는 운영 Chatty 에 올라가 있지 않았다]] 에서 릴리즈 보관소로 운영 적재를 마쳤지만, 보관소가 PC 에 있어 다음 적재를 서버에서 할 수 없었다. 릴리즈 때 첫 적재를 붙이는 시도는 서버 호스트에 uv 가 없어 실패해 되돌렸다. 저녁에 "파일 없는 방식"으로 방향을 정했다.

오늘 그 방식(D55)을 구현하고 운영에 전환했다. **직전 릴리즈의 매니페스트를 `ReleaseHead.manifest_json` 에 함께 적고, 다음 적재는 DB 그래프와 이 기록을 읽어 서로 맞는지 증명한 뒤 이어 적재한다.** 보관소 코드(`release/store.py`)는 삭제했다. 온톨로지는 0.4.3 이다. 운영 DB 를 비우고 PC 에서 다시 적재했다. 결과는 노드 584 · 관계 761 · 벡터 220 · 매니페스트 `cace7315…` 다. 같은 명령을 다시 돌리면 `previous: database` · `unchanged` · 임베딩 0 이다. **이제 PC 에서 적재하든 서버에서 적재하든 이어진다.**

프론트 토글이 안 먹히는 원인은 끝까지 추적했다. **토글은 주소를 바꾸지 않고 헤더 하나만 붙인다.** 기존 채팅과 Graph 활용이 모두 운영 Chatty(7000)로 가고, 운영 Chatty 코드는 그 헤더를 읽지 않는다. 그래프 연동 코드는 Chatty `dev` 브랜치에만 있고 main 과 태그에는 없다. 개발 Chatty(9000)에는 코드가 올라가 있다. 하지만 깃의 `.env` 기준으로 `NARA1_GRAPH_ENABLED='false'`, API 주소가 빈 값, 허용 챗봇 목록이 없다. **이 상태로 헤더를 보내면 기존 채팅으로 넘어가지 않고 503 이 난다.**

## 1. 이전 상태를 DB 에 기록 — 파일 없이 이어 적재 (D55 · 0.4.3)

##### 문제 — 보관소가 적재한 곳에 갇힘
보관소 방식에서는 다음 적재가 직전 `release.json` 을 파일에서 읽는다. PC 에서 첫 적재를 하면 파일이 PC 에 남아 서버 적재가 멈춘다. 파일을 옮기거나 서버에 폴더를 두어야 하는데, 서버에는 SSH 경로도 호스트 uv 도 없다.

##### 검토한 안과 기각 사유
- 릴리즈 때 첫 적재 단계 추가(`deploy.yml [LOAD]`) — 기각. 호스트에 uv 가 없어 실패했고(`uv not found for devuser`) 되돌렸다(`2db8568`). 두 번째 적재부터는 여전히 파일 문제가 남는다
- 보관소를 서버 공용 경로로 — 기각. PC 적재와 서버 적재가 서로 다른 보관소를 보게 된다
- 매니페스트 없이 DB 그래프만 읽기 — 기각. "지난 적재 뒤 누가 DB 를 고쳤나"를 증명할 기준이 사라진다. 바깥 기준을 DB 에서 뽑으면 자기 자신과 비교하게 된다

##### 택한 안 — 매니페스트를 ReleaseHead 에 싣고 두 해시로 증명
기준을 DB 안에 두되, 그래프와 따로 해시로 묶는다. 이어 적재하려면 두 가지가 모두 같아야 한다.

```
기록 무결성   digest(manifest_json) == ReleaseHead.manifest_sha256
그래프 무결성 digest(DB 그래프 − ReleaseHead) == manifest.content_sha256
헤드 일치     ReleaseHead 속성 == head_properties(manifest, manifest_sha256)
```

하나라도 다르면 `PreviousUnavailable` 로 **쓰기 전에 멈춘다**. 매니페스트는 해시 · 건수 · 입력 파일 해시만 담아 2KB 안팎이고 개인정보가 없다.

##### 구현

| 파일 | 변경 |
|---|---|
| `ontology/definition/declaration.py` | `ReleaseHead.manifest_json`(string, required) |
| `ontology/version.py` | `nara1-0.4.3` · 호환 `("nara1-0.4.2", "nara1-0.4.3")` |
| `ingestion/release/build.py` | `head_properties` · `check_previous` — 이전 상태가 파일이든 DB 든 같은 검증 |
| `ingestion/writers/neo4j.py` | `read_previous` · `PreviousUnavailable`. ReleaseHead 없이 노드만 있으면 "이 적재기로 넣은 DB 가 아님", 기록 없는 옛 헤드면 "비우고 다시 넣거나 `--previous`" 안내 |
| `ingestion/load.py` | `--target` 이면 이전 상태를 DB 에서. `unchanged` 판정. `--output` 선택(안 주면 요약 한 줄만, 서버에 `release.json` 이 쌓이지 않음). `--release-store` 제거. `--verify` 는 "DB 그래프 = DB 기록". `--previous` 는 비상용 |
| `ingestion/release/store.py` | 삭제 |
| `docs/data-load.md` | 보관소 절을 "이전 상태는 DB 에 기록된다"로 교체 |

##### 결과
적재 명령이 첫 적재와 이어 적재에서 같다. `load --target production --env-file .env --write`. 조건은 하나로 줄었다. **적재를 돌리는 곳에 최신 입력(수집본 · 명단 · 결정 표)이 있어야 한다.** 깃은 필수가 아니다. 더 옛 명단이면 안전장치로 멈춘다.

## 2. 운영 전환 — 비우고 0.4.3 으로 다시 적재

##### 택한 안 — 비우고 다시 적재(B안)
0.4.2 헤드에는 `manifest_json` 이 없다. PC 보관소 파일을 `--previous` 로 한 번 넘겨 잇는 방법도 있었지만 쓰지 않았다. 프론트가 아직 그래프에 연결돼 있지 않아 비우는 비용이 없고, 기록이 처음부터 0.4.3 으로 시작하는 편이 깔끔하다.

##### 순서
커밋 `0fe1afd` → main → 릴리즈(API 가 0.4.3 을 호환 목록에 갖도록) → 운영 DB 비움 → PC 에서 적재 → PC 보관소 폴더 삭제.

##### 결과

| 항목 | 값 |
|---|---|
| 순번 · 매니페스트 | 1 · `cace7315…` |
| 노드 · 관계 · 벡터 | 584 · 761 · 220(1024) |
| 임베딩 방법 | 생성 216 · 재시도 2 · 규칙 문장 2 |
| 다시 적재 | `previous: database` · `unchanged: true` · 임베딩 todo 0 |
| `--verify` | `verified: true` |
| 로컬 파일 | `.local` 에 생성 없음 |
| API | `/health/data` ready · 조직 소속 질문 ok |

## 3. 프론트 토글 — 주소는 그대로, 헤더 하나만 다름

##### 문제
관리자 AI 화면에서 "Graph 활용"을 켜도 답이 같고 "Graph 적용 확인 안 됨"이 뜸.

##### `chatting_graph.htm` 이 `chatting.htm` 에서 바뀐 곳
- 226줄 `index2.js` → `index2_graph.js`
- 227~230줄 추가 — `graph_chat.css` · `window.NARA1_GRAPH_CONFIG = {askUrl: 'https://dev.chatbaram.com:7000/ask'}` · `graph_ui.js` · `Nara1GraphUI.install(Chatting)`
- 그 밖은 음성 메뉴 CSS 몇 줄 차이

##### 원인 — `graph_ui.js` `prepare()`
`vector` 모드면 원래 요청을 그대로 둔다. `graph` · `hybrid` 모드면 주소만 `askUrl` 로 바꾸고 헤더 `X-Nara1-Mode` 를 추가한다. 본문은 같다.

| | 기존 채팅 | Graph 활용 |
|---|---|---|
| 주소 | `${url1}/ask` (PHP `$url1`, 7000) | `askUrl` (7000) |
| 본문 | 질문 · 모델 · 챗봇 id | 같음 |
| 헤더 | 기본 | 기본 + `X-Nara1-Mode: hybrid` |

두 주소가 같아서 **토글이 실제로 바꾸는 것은 헤더 하나**다. 그래프를 붙일지는 받는 Chatty 가 정한다. `askUrl` 을 따로 둔 목적은 그래프 요청만 다른 Chatty(예: 9000)로 보내 시험하는 것으로 보인다.

## 4. 설계 의도 — 프론트는 Chatty 에만, Chatty 가 Graph API 를 부름

##### 검토한 안 — 프론트가 7000 과 6060 을 동시에 호출 — 기각
- Graph API 키가 브라우저에 노출된다
- `http://IP:6060` 이라 https 페이지에서 막히고 CORS 도 필요하다
- 답이 둘로 오고, 합치는 일을 브라우저가 맡는다

##### 의도한 안 — 서버 간 호출
Chatty 가 기존 벡터 검색과 Graph API 근거를 합쳐 LLM 답을 하나 만든다. 키는 Chatty 서버 안에만 있다. 프론트 변경은 헤더 한 줄뿐이다. 대신 Chatty 가 6060 응답을 기다리는 만큼 느려질 수 있어 짧은 타임아웃과 실패 시 벡터만으로 답하는 처리가 필요하다. 아래 절의 구현이 둘 다 갖추고 있다.

## 5. Chatty 코드 — 7000 · 9000 은 같은 `/ask`, 그래프 연동은 dev 에만

##### 요청을 받는 곳
포트와 상관없이 `Chatty_Project/app.py` 의 `@app.post("/ask")`(main 기준 1294줄)가 받는다. 포트는 코드에 없다. `gunicorn.conf.py` 가 `Config.APP_PORT` 로 바인딩한다.

| 포트 | 서비스 | 서버 폴더 | 배포 |
|---|---|---|---|
| 7000 운영 | `chatty_app.service` | `/home/chat_bot/Chatty_Project` | `deploy-prod.yml` — 태그 `v*.*.*-app` checkout 후 재시작 |
| 9000 개발 | `kss_app.service` | `/home/chat_bot/Chatty_Project_DEV` | `deploy-dev.yml` — `dev` push + 메시지 `app restart` → `reset --hard origin/dev` 후 재시작 |

main 의 `ask()` 인자는 `ask_request` · `session` · HMAC 뿐이다. 선언하지 않은 헤더는 FastAPI 가 버리므로, **7000 은 에러 없이 기존 채팅으로 답한다.**

##### 그래프 연동 커밋
`afb86bc` "feat: Chatty 검색에 나라원 Graph 근거 통합" (9/9). `origin/dev` · `feat/graph-nara1-j` 에 포함, `main` · 태그에는 없음. dev 마지막 배포 `14fcf03`(9/17, `app restart`)이 그 뒤라 9000 에는 코드가 올라가 있다.

| 위치 | 역할 |
|---|---|
| `app.py` `ask()` | `X-Nara1-Mode` 헤더 인자 → `prepare_request()` |
| `app.py` `stream_ai_response_with_context` | 벡터 검색 뒤 `search()` 로 Graph API 호출, `answer_chunks()` 로 근거 추가. 주석 "Graph supplements the existing retrieval; it never replaces that flow" |
| `app.py` 스트리밍 | `graph_request.event()` — `nara1.graph_trace`, 타임아웃에도 보냄 |
| `chat/nara1_graph.py:82` `prepare_request` | `vector` · 첨부 있음 → 기존 흐름. 그 밖에 조건 불충족이면 에러 응답 |
| `chat/nara1_graph.py:161` `search` | `POST {NARA1_GRAPH_API_URL}/v1/graph/search-by-question`, 헤더 `X-API-Key`, 본문 `contract_version "1"` · `question` · `k 3`. 타임아웃 5초, 재시도 없음, 응답 256KB 제한 |
| `chat/nara1_graph.py:119` `_validated_items` | 근거 항목 필수 필드 · 출처 URL · sha256 형식 검증 |
| `chat/nara1_graph.py:19` `HYBRID_INSTRUCTIONS` | 조직 소속을 사업 담당으로 확대 해석하지 말 것 등 |

호출 경로와 계약(`/v1/graph/search-by-question`, `contract_version "1"`)은 우리 `serving/api.py:166` 과 맞는다. 응답 필드 전체 대조는 하지 않았다.

##### 6060 주소가 적힌 곳
Chatty 코드에는 없다. 주소는 환경변수에서만 읽는다. 값이 적힌 곳은 우리 `docs/deployment.md:75` 안내문뿐이다.

##### 9000 이 지금 그래프를 못 붙이는 이유 — 설정이 꺼져 있음
Chatty `dev` 의 `.env`(커밋 `afb86bc` 에서 추가):

```
NARA1_GRAPH_API_KEY=(값 있음)
NARA1_GRAPH_API_URL=''
NARA1_GRAPH_ENABLED='false'
```

`NARA1_GRAPH_CHATBOT_IDS` 는 없다. `prepare_request` 는 다음 순서로 막는다.

| 조건 | 응답 |
|---|---|
| `NARA1_GRAPH_ENABLED` 가 true 아님 | 503 "아직 활성화되지 않았습니다" |
| `db_name != "naraone"` 또는 챗봇 id 가 허용 목록 밖 | 403 |
| URL · 키 형식 불량 | 503 "연결 설정이 필요합니다" |
| 질문 1~1000자 밖 | 422 |

**헤더가 있는데 조건이 안 맞으면 기존 채팅으로 넘어가지 않고 에러로 끝난다.** 반대로 Graph API 호출 자체가 실패하거나 타임아웃이면 `outcome` 만 `unavailable` · `timeout` 이 되고 채팅은 계속된다.

##### 결론 — 프론트 한 줄로는 안 됨
9000 으로 시험하려면 다음이 모두 필요하다.

```
프론트        chatting_graph.htm askUrl → https://dev.chatbaram.com:9000/ask
Chatty dev   NARA1_GRAPH_ENABLED='true'
.env         NARA1_GRAPH_API_URL='http://110.45.147.62:6060'   (16060 은 미기동)
             NARA1_GRAPH_API_KEY=우리 Graph API 키와 같은 값
             NARA1_GRAPH_CHATBOT_IDS='76bc1dfa-9a10-4d5e-850d-ca044e5de5b9'
배포          dev push + app restart
```

운영(7000)은 이 커밋을 main 에 합치고 운영 `.env` 를 채운 뒤 `v*.*.*-app` 태그로 배포해야 한다. Chatty 담당 영역이다.

## 6. 곁가지 — 운영 Neo4j 브라우저 인증 실패

`http://110.45.147.64:7474/browser/` · `bolt://110.45.147.64:7687` · 사용자 `neo4j` 로 접속 시 `authentication failure`. 주소와 사용자 이름은 `.env` 와 같다. 같은 비밀번호로 오늘 적재 · `--verify` 가 통과했으므로 입력한 비밀번호 문제다(따옴표 포함 · 공백). 여러 번 틀리면 잠시 잠긴다.

## 검증

```
섀도우 게이트   gate_055_shadow 12절 실패 0 — 첫 적재 기록 저장 · 재적재 unchanged · 변경 시 순번 2 ·
               별도 프로세스에서 파일 없이 이어 적재 · DB 손대면 멈춤 · 옛 헤드 안내 · verify · 임베딩 220 · 조회 ready
운영 적재      verified · embedding.verified true · 생성 220 (생성 216 · 재시도 2 · 규칙 2)
운영 재적재    previous=database · unchanged · 임베딩 todo 0 · .local 파일 생성 없음
운영 --verify  verified true
API           /health/data ready (0.4.3) · 조직 소속 질문 ok
프론트        chatting_graph.htm · chatting.htm 대조, graph_ui.js · index2_graph.js 요청 경로 확인
Chatty        git log --all -S "X-Nara1-Mode" → afb86bc 1건 · branch --contains → dev 계열만 · merge-base → main 미포함
              origin/dev .env 의 NARA1_GRAPH_* 3줄 확인 (키 값은 보지 않음)
```

## 재사용 가능한 판단

- **기준 상태를 옮길 수 없으면 데이터 옆에 싣는다.** 대신 데이터와 별도 해시로 묶어야 "DB 가 자기 자신과 비교"하는 함정을 피한다
- **이어 적재 가능 여부는 두 해시로 증명한다.** 기록 해시(기록이 온전한가)와 그래프 해시(기록 뒤 DB 가 안 바뀌었나)는 서로 다른 질문이다
- **바뀐 게 없으면 쓰지 않는다.** 같은 명령을 여러 번 돌려도 안전해야 서버 · PC · 사람 누구나 돌릴 수 있다
- **스키마 전환 직후, 소비자가 붙기 전이 비우고 다시 넣기 가장 싸다**
- **토글이 안 먹힐 때 요청 차이를 표로 뽑는다.** 주소 · 본문 · 헤더 중 무엇이 바뀌는지 보면 받는 쪽 책임인지 바로 갈린다
- **코드 존재 여부는 브랜치 전체에서 찾는다.** main 만 보고 "없다"고 판단했다가 `git log --all -S` 로 dev 에서 찾았다
- **코드가 올라가 있어도 설정이 꺼져 있으면 동작하지 않는다.** 배포 확인은 코드와 환경변수 두 층으로 본다
- **안전장치의 실패 방향을 확인한다.** 조건 불충족이 "기존 흐름"이 아니라 "에러"로 가면, 설정 전에 트래픽을 보내는 순간 채팅이 깨진다
- **외부 서비스 주소는 호출하는 쪽 환경변수에만 둔다.** 코드와 프론트에 IP 가 없어야 운영 · 개발 전환이 설정만으로 된다

## 한계

- **Chatty 서버의 실제 환경변수는 보지 못했다.** 깃의 dev `.env` 를 근거로 "꺼져 있다"고 판단했다. systemd 등에서 따로 주입하면 다를 수 있다
- **Chatty 저장소는 PC 에 받아 둔 원격 기록 기준이다.** 원격 main 기록은 9/17 것까지 있다
- **9000 에 실제 요청을 보내 보지 않았다.** HMAC 비밀값이 7000 과 같은지, `.56` → `.62:6060` 통신이 열려 있는지 모른다
- **Chatty 연동 코드와 우리 API 응답의 필드 단위 대조는 하지 않았다.** 경로 · `contract_version` 만 맞춰 봤다
- **조회는 여전히 조직 소속 질문 하나뿐이다.** 벡터 220개를 읽는 코드가 없다
- **개발 Graph API(16060)는 떠 있지 않다.** `dev` 브랜치가 옛 워크플로다. main 을 dev 에 합쳐 push 해야 한다
- **서버에서 적재하는 실행 경로는 아직 없다.** 호스트에 uv 가 없어 API 이미지로 일회용 컨테이너를 띄우는 방식이 유력하다
- **재수집 시 "최신 수집본만 읽기"가 없다.** 새 수집본 폴더를 넣으면 같은 주소가 두 번 나와 멈춘다
- **PC 적재는 여전히 bolt 평문으로 외부 주소를 거쳤다**

## 관련 문서

- [[0916_KG - 나라원 · 운영 첫 적재 완료, 그래프는 운영 Chatty 에 올라가 있지 않았다]] — 어제. 보관소를 오늘 DB 기록으로 대체했고, "어느 브랜치에 있는지 확인하지 못했다"던 연동 위치를 오늘 찾았다
- [[KG_나라원 운영 전환 점검]] — 운영 적재 0.4.3 전환 완료, 프론트 연결 조건(9000 설정 · 운영 Chatty 반영)으로 갱신할 대상
- [[0825_KG - 기존 챗봇 연동 구현 및 KG 버그 수정]] — 수원 쪽 기존 챗봇 연동. 서버 간 호출 구조를 비교할 때 참고
