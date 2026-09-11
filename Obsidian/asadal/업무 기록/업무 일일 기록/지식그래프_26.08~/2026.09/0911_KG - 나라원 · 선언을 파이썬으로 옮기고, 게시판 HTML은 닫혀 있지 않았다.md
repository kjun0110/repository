## 개념

- **의존 방향 역전** — 선언(`ontology/`)이 파이프라인(`ingestion/`) 타입을 빌려 쓰던 것을 뒤집는 것. 심판이 선수에게 규칙집을 빌리면 선수를 갈아엎을 때 심판이 같이 무너진다
- **바이트 게이트** — 옮기기 전후에 canonical JSON 바이트가 같은지 대조하는 것. **"동작이 안 바뀌었다"의 증명**이며 오늘 다섯 단계 내내 이걸로 확인했다
- **배관 / 고유 정보** — 라벨 하나를 추가할 때 하는 일의 두 종류. 고유 정보는 "몇 번째 칸이 무슨 속성"이고 줄일 수 없다. 배관은 파일 열기·행 순회·정제·출처 붙이기이며 라벨마다 똑같다
- **골격(shape)** — "이 순서로 읽으면 된다"의 이름표. HTML 이 생긴 모양만 가리키고 내용도 라벨도 모른다. 그래서 재사용된다
- **테이블 주도 추출** — 추출 규칙을 파이썬 자료구조로 적고 해석기 하나가 그것을 도는 방식. 선언 해석기(YAML)와 파서 파일의 중간
- **선언 매체** — 값을 YAML 에 담을지 파이썬에 담을지. 오늘 파이썬으로 바꿨다(D27)

## 요약

[[0910_KG - 나라원 · 근거층은 URL 하나 때문이었고, 분류는 이미 있었다]] 에서 결정만 25건 쌓고 코드는 한 줄도 안 고쳤다. 오늘 **온톨로지 구현을 끝냈다**(S1~S6). 그리고 파서 방식을 정하려다 **내 주장 여섯 개가 틀린 것을 확인했다.**

먼저 폴더 경계를 세웠다. `ontology/` 가 `ingestion/` 을 import 하던 줄을 끊고(타입 7개 이동), 제약·인덱스 생성기를 질의 폴더에서 선언 옆으로 옮겼다. **적재가 질의 폴더를 import 하던 두 줄이 사라져 `ingestion → ontology` 로 직행한다.** 세 단계 다 릴리즈 manifest sha `b5d11ea5…`/`b48de4e4…` 불변으로 확인했다.

그다음 선언 값을 파일 하나로 옮겼다. 마크다운 표 안에만 있던 라벨 9·관계 7·어휘 4 를 기계가 읽을 수 있게 적었더니 **문서 안의 모순 셋이 드러났다** — 같은 문서 두 곳이 `identity_status` 를 반대로 적었고, `client_type` 이 최종안 표에서 빠져 있었다. 그리고 원문을 다시 열어 보니 **어휘 값 6종 중 4종이 사이트와 달랐다**(`재단및협회` ≠ `재단 및 협회`).

파서 방식은 셋을 실측해 정했다. nara1graph 의 선언 해석기(`parse.py` 754줄 + `extract.yaml` 328줄)를 분해하고, **파이썬 테이블 주도 프로토타입을 만들어 실제 스냅샷에 돌렸다.** 해석기 38줄로 라벨 3종·골격 2종이 돌았고, 제일 어려운 연혁이 **훅 7줄로 27건(자사 8·이전 19)** 을 정확히 냈다.

그 과정에서 **나라원 게시판 HTML 이 `</tr>` 를 안 닫는다**는 것을 발견했다 — `<tr>` 14개에 `</tr>` 2개다. 브라우저는 알아서 닫지만 표준 파서는 행을 통째로 잃는다. 골격 함수 **한 곳**을 고쳐 사업 게시판과 공지사항이 동시에 살아났고, 이것이 배관 공유의 값어치를 추측이 아니라 **실제로 일어난 일**로 보여 줬다.

마지막으로 **선언의 매체를 YAML 에서 파이썬으로 바꿨다.** 원래 YAML 을 고른 근거(*"Python 만 두면 근거 주석·타입·어휘를 적을 자리가 없다"*)가 사실이 아니었다. 구운 JSON 바이트 `d48b6746…` 가 불변이라 값이 한 글자도 안 바뀐 것이 증명됐다.

## 1. 폴더 경계를 세웠다 (S1~S3)

##### 문제

`ontology/validation.py` 가 `NodeCandidate`·`ProvenanceRef` 를 `ingestion/models.py` 에서 가져다 썼다. **온톨로지가 "이 후보가 규칙에 맞나"를 판정하는데 후보가 무엇인지는 파이프라인이 정하고 있었다.**

그리고 제약·인덱스를 만드는 `schema_ddl()` 이 `graph_rag/` 에 있는데 쓰는 곳은 적재 두 곳이었다. **적재가 질의 폴더를 import 했다.**

##### 한 일

```
S1  ontology/model.py        선언의 모양 (pydantic).  15종 오류를 로드 시점에 거부
S2  ontology/contracts.py    타입 7개 이동 — ProvenanceRef · EntityKey · NodeCandidate
                             RelationCandidate · Candidate · ValidationIssue · ValidationResult
S3  graph_rag/schema.py  →   ontology/ddl.py
```

`ingestion/models.py` 는 옮긴 7개를 **재수출**해서 `from ingestion.models import NodeCandidate` 를 쓰던 8곳이 한 줄도 안 바뀌었다. `ingestion.models.NodeCandidate is ontology.contracts.NodeCandidate` → `True` 로 복제가 아닌 것도 확인했다.

##### 결과

```
이전   ingestion → graph_rag → ontology
지금   ingestion → ontology

grep "^from ingestion" ontology/*.py     → 없음
grep -rn "graph_rag" ingestion/          → 없음
```

**`ontology/` 는 이제 아무것도 import 하지 않고 나머지가 `ontology/` 를 import 한다.**

##### 게이트 — DDL 은 개수로 세면 안 된다

제약 16개를 개수만 세면 순서가 뒤집혀도 통과한다. 문장 전체를 이어붙여 해시했다(`640a84a0f2911f44`, 불변).

> `git mv` 는 쓰지 않았다. 깃을 건드리는 명령이라 파일만 옮기고 스테이징은 그대로 뒀다.

---

## 2. 선언 값을 파일 하나로 (S4) — 문서 안 모순 셋

##### 문제

선언이 세 군데에 따로 있었고 서로 다른 말을 했다.

```
ontology/registry.py  REGISTRY        0.1.0  라벨 4   ← 런타임이 실제로 읽는 것
                      DOMAIN_DRAFT    0.2.0  라벨 6
                      RELEASE_REGISTRY 0.3.0 라벨 9   ← Assertion·DELIVERED_BY 등 버리기로 한 것
kjundocs/…-decisions.md              라벨 9 관계 7 어휘 4   ← 진짜 결정. 그런데 마크다운 표
```

그리고 옛 구조에는 **타입과 어휘를 적을 자리 자체가 없었다.**

```python
required_properties=("name", "unit_type", "display_order")
#                             ↑ team/division 넷 중 하나라는 걸 적을 칸 없음
#                                          ↑ int 라는 걸 적을 칸 없음
```

##### 한 일

선언 315줄을 적었다. 속성마다 타입·필수·인덱스·어휘를 붙이고, 라벨마다 `note` 에 **왜 그렇게 정했는지를 결정번호로** 남겼다.

##### 드러난 것 — 표로는 일관돼 보이던 모순 셋

```
① identity_status   1-5절 표: 온전한 이름 59 = resolved
                    D19-⑤ : 59 = source_scoped, resolved 는 0건(사람이 확정했을 때의 값)
                    → 같은 문서 두 곳이 반대로 적혀 있었다
② client_type       D21 이 "붙인다"고 철회했는데 1-6 최종안 속성 목록에서 누락
③ name_truncated    identity_status == review_required 와 23건이 정확히 겹침
```

`resolved` 의 뜻이 갈린다 — "이름이 안 잘렸다" 냐 "사람이 확정했다" 냐. **뒤에 쓰인 쪽에 근거가 붙어 있어 그쪽을 택했고, 이후 "문서 내 모순은 나중 결정을 따른다"를 규칙으로 못 박았다**(D26-①).

##### 검사기가 실제로 무는지 확인

일부러 9가지를 틀려 9개 다 거부됐다.

```
IN_AREA target 을 BusinessAria 로     →  endpoint 라벨이 선언에 없습니다
vocab 을 client_types 로              →  선언에 없는 어휘를 가리킵니다
cardinality: N:1                      →  Input should be 'one_to_one', 'one_to_many', …
type: text                            →  Input should be 'string', 'int', 'date', …
배열 source_url 에 indexed: true      →  배열 속성에는 indexed를 쓰지 않습니다
키를 properties 밖 이름으로            →  식별키가 properties에 없습니다
칸 이름을 kye 로 오타                  →  Extra inputs are not permitted
identity_status 속성 삭제              →  아무 속성도 쓰지 않는 어휘입니다
inference_policy: llm_guess           →  Input should be 'direct_only' or 'deterministic'
```

마지막 줄이 중요하다 — **추측을 뜻하는 세 번째 값을 만들 수 없다.** 값이 둘뿐인 것 자체가 계약이다.

##### 내가 임의로 건 required 를 되돌렸다

문서가 명시한 건 `OrganizationalUnit` 3개와 `HistoryEvent` 4개뿐인데 아홉 개를 더 걸었다가 전부 풀었다. 규칙을 파일 머리에 적었다.

```
필수는 셋뿐   ① 식별키  ② 출처 3칸  ③ 문서가 못 박은 것
앞 둘은 우리가 만드는 값이라 비어 있을 수 없다
원문에서 오는 값에 걸면 사이트가 한 칸 비우는 순간 적재 전체가 거부되고,
그 검사는 곧 꺼진다 — D10 에 적은 그 병이다
```

---

## 3. 어휘 값이 사이트와 달랐다

##### 실측

`guest.htm` 의 `h4` 를 다시 읽었다.

```
원문                     문서에 적혀 있던 것
대한민국 중앙행정기관   ≠  중앙행정기관
지방자치단체            =  지방자치단체
공공기관                =  공공기관
재단 및 협회            ≠  재단및협회       ← 공백
교육청 및 학교          ≠  교육청및학교
국회 및 지방의회        ≠  국회및지방의회
```

**6종 중 4종이 달랐다.** D19-④ 가 *"원문 언어 그대로"* 였는데 언어만이 아니라 **문자열 그대로**를 뜻한다.

##### 그 김에 — 사이트에 어휘가 두 벌이다

```
guest.htm h4          대한민국 중앙행정기관 · 지방자치단체 …   ← 어휘로 삼기로 한 쪽
k=cause04 파라미터     Government · Foundations · a …          ← 그래프에 실제로 들어올 값
k=cause03             Homepage · System · IntegratedInformationSystem
```

**그래프에 들어올 값과 어휘에 적은 값이 다르다.** 대응표가 없으면 `client_type` 과 `IN_AREA` 를 채우는 순간 어휘 검증에서 전부 튕긴다. 대응표를 어휘가 아니라 **파서**가 갖기로 했다 — 어휘는 "그래프에 무엇이 있나", 대응표는 "원문을 어떻게 읽나"라 층이 다르다(D26-④).

---

## 4. 파서 방식 — 셋을 재서 정했다

##### 선택지

```
A   파서 파일        페이지마다 파이썬 파일 하나
A'  테이블 주도      규칙을 파이썬 자료구조로, 해석기 하나가 돈다
B   선언 해석기      규칙을 YAML 로, 해석기가 해석한다  (nara1graph 방식)
```

##### nara1graph 분해

```
pipeline/parse.py 754줄
  골격 추출 + 실행기 run_*      310줄  41%
  규칙 해석 (필드·파생·노드화)   142줄  18%
  검사·보고·진입점              157줄  20%
  원문 목록·링크                 96줄  12%
  그래프 조립                    49줄   6%
schema/extract.yaml  328줄 / 추출기 14개 → 평균 13줄
```

##### 실측 ① — "라벨이 늘면 YAML 만 늘고 코드는 그대로"는 조건부다

```
shape 10종 중 7종이 딱 한 번씩만 쓰인다
prose 3 · table-rows 2 · card-grid 2  |  span-pair·nested-tree·image-list·
                                          year-blocks·list-items·heading-block·jsonl 각 1
```

**추출기 14개 중 절반은 만들면서 골격을 새로 발명해야 했고, 그때마다 실행기가 따라붙었다.** `parse.py:700` 의 `elif` 사슬이 그 증거다.

##### 실측 ② — 같은 골격의 두 번째부터 1/5 가격

```
board.cause04       26줄   table-rows 를 처음 만들며 추가
board.notice         5줄   같은 table-rows 재사용        ← verified: false, 나중에 추가된 것
highlight           20줄   card-grid 를 처음 만들며 추가
board.companyphoto   5줄   같은 card-grid 재사용          ← verified: false
```

**가장 싼 두 개가 정확히 기존 골격을 재사용한 둘이다.** 이게 "라벨이 늘수록 유리해지는" 실제 기전이다.

##### 실측 ③ — 파이썬 테이블 프로토타입

스크래치패드에 만들어 **실제 스냅샷에 돌렸다.**

```
부품 상자 (clean·cast)     6줄    nbsp_strip · collapse_ws · strip · nfc · to_date · to_int
골격 2종                  30줄    parse_table_rows · parse_label_pairs
선언 테이블 (라벨 3개)     32줄    라벨당 약 10줄
해석기 (11단계 배관)       38줄    ← 고정비. 라벨이 늘어도 안 늘어난다
합계                     113줄

project  BusinessProject  59건   notice  Notice 2건   company  Company 6쌍
```

**해석기가 38줄이다. YAML 판 500줄이 아니라.** 함수를 직접 담기 때문에 `CLEANERS` 레지스트리·`transforms` 레지스트리·`shape → run_*` `elif` 사슬이 전부 사라진다.

##### 실측 ④ — 제일 어려운 것도 훅 7줄

연혁은 한 페이지에 구조가 둘이다.

```html
자사   <li><div class="year">2021</div><ul><li>11월 서울특별시교육청…</li></ul></li>
이전   <li>2000년 04월 모회사 법인 설립</li>
```

```python
def history_hook(sec, head, txt):
    if m := YM_INLINE.match(txt):          # "2000년 04월 …"
        return {"year": int(m[1]), "month": int(m[2]), "description": m[3], "scope": "predecessor"}
    if head and (m := M_ONLY.match(txt)):  # head="2021", "11월 …"
        return {"year": int(head), "month": int(m[1]), "description": m[2], "scope": "own"}
    return None
```

**27건 = 자사 8 · 이전 19.** 문서 실측과 정확히 일치했다. `scope` 판정이 `if` 두 개로 떨어졌다.

##### 손익

```
            고정비    라벨당    9종     20종     30종     같은 버그
A  파서 파일     0     42줄     380     840    1,260    라벨 수만큼 복제
A' 테이블       74     12줄     180     314      434    항상 한 곳
B  YAML        500     13줄     620     760      890    항상 한 곳
```

**A' 로 정했다.** B 가 지불하는 것 대부분이 "YAML 은 함수를 담을 수 없다"의 결과이고, 파이썬이면 안 낸다.

---

## 5. 게시판 HTML 이 닫혀 있지 않았다

##### 증상

프로토타입에서 사업 게시판이 **0건**이 나왔다. 표 머리글과 페이지네이션 행 2개만 잡혔다.

##### 원인

```
<tr> 14개   </tr> 2개
```

```html
<tbody>
<tr class='ListSty2'><td>208</td><td>…용역</td><td>2024-08-20</td><td>대중소기업농어업협력..</td>
<tr class='ListSty1'><td>207</td><td>…유지관리</td><td>2023-03-03</td><td>경기복지재단</td>
                                                                     ↑ </tr> 가 없다
```

닫힌 것은 `<thead>` 의 머리글 행과 페이지네이션 행뿐이다. 브라우저는 다음 `<tr>` 을 만나면 알아서 닫지만 `html.parser` 는 안 닫아 **행을 통째로 잃는다.**

##### 고친 자리 — 한 곳

```python
def _flush(self):
    if self.cur: self.rows.append(self.cur)
    self.cur = None
def handle_starttag(self, t, a):
    if t == "tr": self._flush(); self.cur = []   # 다음 <tr> 에서 이전 행을 닫는다
```

**이 한 곳으로 사업 게시판(59건)과 공지사항(2건)이 동시에 살아났다.**

##### 값어치

파서 파일 방식이었으면 같은 처리가 `project.py` 와 `notice.py` 에 각각 있었고, 한 곳만 고치고 다른 데를 놓칠 수 있었다. **배관 공유의 값어치를 추측이 아니라 실제로 일어난 일로 확인했다.**

##### 그 김에 — 다섯 번째 골격을 찾았다

회사개요의 라벨-값 6쌍이 기존 골격 넷 중 어디에도 안 맞는다.

```html
<li><span class="ws4">법 인 명</span><span>㈜나라원시스템</span></li>
<li><span class="ws4">대 표 자</span><span class="txt2Style">심재춘</span></li>
```

`table-rows`(표 없음) · `h4구역+li`(h4 없음) · `prose`(문단 아님) · `평면li`(li 안에서 가로로 짝) 전부 아니다. nara1graph 도 이것만 따로 `span-pair` 로 이름 붙였고 **한 번만 쓴다.**

---

## 6. 선언의 매체를 파이썬으로 (D27)

##### 원래 근거가 사실이 아니었다

Q4 에서 YAML 을 고르며 이렇게 적었다.

> ~~"Python 만 두면 **근거 주석·타입·어휘를 적을 자리가 없고**"~~

파이썬에도 다 적을 자리가 있다. `note`·`example`·`vocab` 은 **`model.py` 가 필드로 갖고 있어서** 적을 수 있었던 것이지 YAML 이라서가 아니었다. **매체의 공으로 잘못 돌렸다.**

##### 바꾼 이유 — 오타를 타이핑하는 순간 잡는다

```
YAML     kye: 조직단위       →  python -m ontology.build 를 돌려야 Extra inputs are not permitted
파이썬    NodeSpec(kye=…)    →  편집기가 즉시. Literal 이 자동완성까지 해준다
```

D16 이 *"선언의 모양은 pydantic 이 강제한다"* 였는데, 파이썬이면 그 보장이 **더 일찍** 온다.

##### 철회한 근거 둘

```
"바이트 게이트 때문에 YAML 이어야 한다"
   → releases.py:137 은 구운 JSON 을 비교한다. 원본 언어와 무관하고, 파이썬에서
     ** 전개를 써도 JSON 바이트는 결정적이라 게이트는 안 깨진다

"반복이 보장이다"
   → 출처 3칸이 8번 반복되는 것보다 **PROV 한 줄이 오히려 읽기 쉽고,
     8곳 중 한 곳만 다르게 쓰는 실수가 불가능해진다
```

대신 **파일 머리에 "로직을 쓰지 않는다"를 명시**하는 규칙으로 대체했다. 매체가 막아 주던 것을 사람이 지킨다.

##### PyYAML 은 안 사라진다

`resolutions/*.yaml`(사람 결정)을 `candidates.py` 가 읽어야 하므로 적재 경로에 그대로 필요하다. 빌드 경로에서만 빠진다. "의존 0" 이라고 했던 것을 정정했다.

##### 손으로 옮기지 않았다

`note` 가 YAML 블록 스칼라라 줄바꿈과 후행 개행이 정확히 보존돼야 한다. 일회용 생성기로 `ontology.yaml` 을 읽어 파이썬 소스를 찍었다.

```
ontology/ontology.yaml 315줄  →  ontology/declaration.py 357줄
구운 JSON  d48b6746217d0686…  불변
```

##### 최종 매체 배치

```
로직이 들어가야 하는 것    →  파이썬   declaration.py · extract.py · shapes.py · clean.py
로직이 들어가면 안 되는 것  →  YAML     resolutions/*.yaml (사람 승인 기록)
```

`resolutions/` 가 YAML 로 남는 이유 — 파이썬이면 **승인이 코드 변경**이 되고, 인증서 21건 전사나 매출 차트 판독처럼 **개발자가 아닌 사람이 채울 수 있는 것**이 여기 들어온다.

---

## 7. 적재 범위는 C·U 뿐이고 D 는 거부로 막힌다

##### 실측

코드 전체에 `DELETE`·`DETACH DELETE` 가 **0건**이다. 유일한 `REMOVE` 는 쓰기 잠금 해제다.

```
C  생성   O   MERGE
U  수정   O   SET n = row.properties   ← 속성 통째 교체
D  삭제   X   릴리즈에서 빠진 노드가 DB 에 그대로 남는다
```

##### D 가 없어도 조용히 틀리지는 않는다

```python
nodes = [... for r in tx.run("MATCH (n) RETURN labels(n), properties(n)")]   # DB 전체
if after != release["graph"]:
    raise ValueError("Post-write parity failed; transaction rolled back")
```

`MATCH (n)` 이라 **DB 에 있는데 릴리즈에 없는 노드가 하나라도 있으면 parity 가 깨지고 통째로 롤백된다.** 삭제가 필요한 상황이 오면 적재가 **거부**된다. 잘못 들어가는 게 아니라 멈춘다.

##### 그러나 U 는 지금 손봐야 한다

`SET n =` 통째 교체라, 사람이 `resolutions/clients.yaml` 에 확정한 발주처 이름이 다음 적재에서 원문값으로 **되돌아간다.** `candidates.py` 에 사람 결정이 원문을 이기는 규칙이 필요하다.

##### D 를 나중에 할 때

사라진 것이 셋으로 갈린다 — ① 사이트에서 진짜 없어졌다 ② 크롤이 그 페이지를 못 받았다 ③ 파서 규칙이 바뀌어 못 뽑았다. **스냅샷이 그 페이지를 실제로 받았을 때만 삭제 후보로 본다.** nara1graph 의 `--prune` 은 ②③을 구분하지 않아 크롤 한 번 실패하면 노드가 날아간다.

---

## 검증

```
S2  타입 이동          릴리즈 sha b5d11ea5… / b48de4e4… 불변 · 같은 클래스 True · import 무실패
S3  DDL 이동           문장 16개 이어붙인 해시 640a84a0f2911f44 불변 · 릴리즈 sha 불변
S4  선언 작성          라벨 9 관계 7 어휘 4 · 검사기 9종 전부 거부
S5  build.py          두 번 구워 바이트 동일 d48b6746… · 동결본 덮어쓰기 거부 · --check 무쓰기
                      draft ↔ candidate 차이는 status 한 칸 · 왕복 일치
S6  drafts 정리        0.1.0 중복 삭제(versions/ 와 바이트 동일) · drafts/ 개념 유지
D27 매체 전환          구운 JSON d48b6746… 불변 ← 값 한 글자도 안 바뀐 증명
                      검사기 9종 · 결정성 · import · 릴리즈 sha 전부 재확인
프로토타입             실제 스냅샷 — project 59 · notice 2 · company 6 · history 27(8+19)
원문 재확인            guest.htm h4 6종 · 게시판 th 4칸 · <tr>14:</tr>2 · k/v 파라미터 10종
```

## 재사용 가능한 판단

- **옮기기만 한 변경은 해시로 증명한다.** S2·S3·D27 세 번 다 릴리즈 sha 또는 구운 JSON 바이트로 무회귀를 확인했다. "고쳤는데 안 바뀌었다"는 말이 아니라 숫자다
- **개수로 세는 게이트는 순서 뒤바뀜을 못 잡는다.** DDL 16개를 개수만 세면 통과한다. 문장 전체를 이어붙여 해시해야 한다
- **표를 기계가 읽는 형태로 옮기면 모순이 드러난다.** 마크다운 표 안에서는 일관돼 보이던 `identity_status` 가 같은 문서 두 곳에서 반대였다
- **문서 안의 모순은 나중 결정을 따른다.** 표는 갱신을 빠뜨리지만 결정은 근거를 달고 쓰인다
- **"원문 그대로"는 언어가 아니라 문자열이다.** 어휘 6종 중 4종이 공백 때문에 달랐다
- **어느 한쪽이 유리하다고 말하려면 그쪽 값을 먼저 재야 한다.** "해석기는 예외를 못 받는다"고 물렸다가, 저쪽 `transforms` 4개 + `hooks` 2개를 보고 철회했다
- **고정비가 큰 설계는 회수 구간을 세어 본다.** shape 10종 중 7종이 한 번만 쓰였다 = 절반은 회수가 없었다
- **매체를 고를 때 "그 매체라서 되는 것"과 "모델이 있어서 되는 것"을 가른다.** `note`·`example` 은 YAML 덕이 아니라 `model.py` 필드 덕이었다
- **버그가 한 곳에서 고쳐지는지가 공유 구조의 값어치다.** `</tr>` 미닫힘을 골격 함수 한 곳에서 고쳐 두 추출기가 동시에 살아났다
- **HTML 이 문법에 맞다고 가정하지 않는다.** 게시판이 `</tr>` 를 안 닫는다. 정규식으로 훑으면 한 행에 12건이 뭉쳐 나온다
- **필수는 우리가 만드는 값에만 건다.** 원문에서 오는 값에 걸면 사이트가 한 칸 비우는 순간 적재 전체가 멎고, 그 검사는 곧 꺼진다
- **삭제가 없어도 parity 가 있으면 안전하다.** 잘못 들어가는 게 아니라 멈춘다 — D 를 미룰 수 있는 근거다
- **승인 기록은 코드가 아니어야 한다.** 파이썬 파일이면 승인이 코드 변경이 되고, 개발자 아닌 사람이 못 채운다

## 한계

- **파서는 아직 한 줄도 없다.** 프로토타입 둘은 스크래치패드에만 있고 프로젝트에 안 넣었다. 실제 라벨 9종을 뽑는 코드는 S7 부터다
- **`registry.py` 는 여전히 0.1.0 하드코딩이다.** 새 선언(`declaration.py`)을 아직 아무도 읽지 않는다. 교체는 S16 이며 서빙(버전 불일치로 근거 거부)과 릴리즈(바이트 게이트)가 동시에 걸려 3단계로 쪼개야 한다
- **영문 코드 대응표가 없다.** `Government → 대한민국 중앙행정기관`, `Homepage → nara1:area:website` 가 없으면 `client_type` 과 `IN_AREA` 를 채우는 순간 어휘 검증에서 전부 튕긴다
- **`resolutions/` 폴더가 아직 없다.** 잘린 발주처 23 · 동률 4종이 갈 곳이 없다
- **테이블 주도가 나머지 넷에서도 되는지는 모른다.** 연혁(훅 7줄)·회사개요(1줄)는 쟀지만 `offering`(h3 없는 2건)·`project`(주요실적 12 병합)는 안 해봤다. 훅이 30줄 넘는 블록이 셋 이상 나오면 그 블록만 손으로 쓴 파서로 뺀다
- **`declaration.py` 의 "로직 금지"는 사람이 지켜야 한다.** YAML 은 물리적으로 못 썼지만 파이썬은 규칙일 뿐이다
- **`name_truncated` 가 `identity_status` 와 중복인 채로 남아 있다.** 23건이 정확히 겹치고, 이것 때문에 `bool` 이 세 번째 사용 타입이 됐다
- **`read_by` 가 9종 중 7종에서 비어 있다.** 그래프를 읽는 코드가 계층 질의 하나뿐이라 지금은 정직한 상태지만, 파이프라인을 다 만든 뒤에도 비어 있으면 뺄 후보다
- **`<tr>` 미닫힘이 다른 페이지에도 있는지 안 봤다.** 게시판 한 장만 확인했다

## 관련 문서

- [[0910_KG - 나라원 · 근거층은 URL 하나 때문이었고, 분류는 이미 있었다]] — 어제. 결정 25건을 오늘 코드로 옮겼고, 그 과정에서 결정 안의 모순 셋이 드러났다
- [[0909_KG - 목록도 가설이었다, 그리고 모르는 것에 딱지를 붙였다]] — "원천을 한 장 열면 5분". 오늘은 `guest.htm` 과 게시판 HTML 에서 같은 일이 일어났다
- [[0907_KG - 수원 · 선언이 검사를 만들면 결함이 드러난다]] — 선언을 기계가 읽게 만들면 결함이 드러난다는 같은 패턴
- [[KG 수원-나라원 파이프라인 비교]] — 두 구현 대조. 오늘 파서 층에서 갈리는 지점을 수치로 확인했다
