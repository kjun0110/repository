## 개념

`suwon_graph`의 조직도 그래프(939 OrgUnit, 5,089 Employee) 위에 실제 질문에
답하는 검색 파이프라인(하이브리드 RAG)을 구현하고, Opik self-hosted를 붙여
질문 하나가 어떤 경로를 탔는지 trace로 확인할 수 있게 만들었다. ([[0818_KG - suwon_graph 조직도 구현 정리]]의 후속)

**설계 원칙**: opus의 정규화(LLM 별칭 확장)/지역 감지 단계는 처음부터 넣지
않았다. 그건 opus가 baseline 실패를 실측한 뒤 추가한 것들이라, 우리도 실패
사례 없이 미리 만들면 자기충족적이다. 최소 버전으로 시작해서 실측하며 추가.

---

## 파이프라인 구조

```
질문
  ├─(0) 리더 숏컷        position 문자열 직접 매치 -> LLM/벡터검색 전부 생략
  ├─(1) 구조 질의 라우터   "OO 목록/몇 개" -> unit_level 필터 직접 조회
  ├─(2) 벡터 검색         정규화 없이 원문 그대로 임베딩 -> entity_embedding
  ├─(3) 컨텍스트 조립      PART_OF 조상경로 + WORKS_AT 직원(많으면 풀텍스트로 좁힘)
  └─(4) LLM 답변 합성      컨텍스트 근거만 사용, 없으면 "모른다"
```

`search/steps/{leader,structure,vector,context,synthesize}.py` + 오케스트레이션
`search/pipeline.py`. 실행: `python search/pipeline.py "질문"`.

---

## 리더 숏컷이 왜 이렇게 단순한가

실측: 이름이 공개된 직원은 전체 5,089명 중 **6명뿐**이고 전부 position이
"장"으로 끝나는 최상위 리더(수원시장/제1부시장/구청장 4명)다. 그래서
"이름 공개 + position이 장으로 끝남" 조건만으로 리더 후보를 뽑고, 질문에
position 문자열이 그대로 들어있는지 확인하는 걸로 충분했다 — 별도 사전이나
분류 로직 불필요.

---

## Opik 연동

`track_openai(client)`로 OpenAI 호출을 자동 계측하고, `search()`(루트)와
각 step 함수에 `@track(type="tool")`을 달아서 하나의 질문 = 하나의 trace,
그 안에 어떤 경로(리더/구조/벡터)를 탔는지 스팬으로 남게 했다.

### 실제 겪은 이슈 2개 (self-hosted 기동 중)

| 이슈 | 원인 | 해결 |
|---|---|---|
| `opik.ps1` 실행 시 `Missing closing ')' in expression` | Windows용 런처 스크립트 자체의 인코딩/파싱 버그 | 우회 — `docker compose --profile opik up -d`를 직접 실행 (README의 공식 프로파일 명령) |
| 첫 trace 전송 시 `404 Workspace not found` | 이 머신에 **다른 프로젝트**(`fibo-rag`)용 `~/.opik.config`가 이미 있었고 `workspace="k-jun"`(Opik Cloud 값)이 박혀있었음. self-hosted는 `"default"` 워크스페이스만 지원 | 전역 설정 파일은 안 건드리고, `config.py`에서 `OPIK_WORKSPACE`/`OPIK_URL_OVERRIDE`를 env var로 `setdefault` — 이 프로젝트 실행 범위 안에서만 격리 |

### 검증 (REST API로 서버 상태 직접 확인)

```
리더 숏컷 질문  ->  span_count=2, llm_span_count=0, has_tool_spans=true
벡터검색 질문   ->  span_count=7, llm_span_count=1, providers=['openai']
```

trace 이름/입출력/소요시간까지 서버에 정상 저장된 것을 `curl .../api/v1/private/traces`로 직접 확인했다.

---

## 실측 결과 (테스트 질문 4건)

| 질문 | 경로 | 결과 |
|---|---|---|
| "수원시장 연락처 알려주세요" | 리더 숏컷 | 정확 |
| "수원시 구청 목록 알려줘" | 구조 라우터 | 정확 (4개 구 전부) |
| "노인 일자리 신청하고 싶어요" | 벡터 검색 | 정확 (부서/경로/연락처까지) |
| "쓰레기 무단투기 신고하려면 어디로" | 벡터 검색 | **"확실한 답을 찾을 수 없습니다"** |

마지막 케이스는 실패가 아니라 **설계가 의도대로 작동한 증거**다 — top-3
후보(건축팀/사업장폐기물관리팀/광고물관리팀)가 실제 담당 부서(환경위생과/
청소자원과)가 아니어서, LLM이 "지어내지 마라" 프롬프트에 따라 억지로
답하지 않고 정직하게 모른다고 했다. 동시에 이건 opus가 별칭 사전을 도입한
계기가 됐던 것과 **똑같은 패턴**(시민 어휘 "무단투기" ≠ 행정 문서 어휘)이라,
별칭 사전이 필요하다는 첫 실측 근거가 됐다. 지금 바로 만들지 않고 실패
사례를 더 모아서 카테고리 단위로 착수할 예정 — 개별 문항 대응이 아니라
일반화되는 사전을 만들기 위함.

---

## 한 눈에 요약

```
파이프라인       →  리더숏컷 -> 구조라우터 -> 벡터검색 -> 컨텍스트조립 -> LLM합성
설계 원칙        →  정규화/별칭/지역 감지 없이 최소로 시작, 실패 실측 후 추가
리더 숏컷        →  이름 공개 직원 6명뿐(전부 최상위 리더) -> 문자열 매치로 충분
Opik 연동        →  track_openai(자동 llm 스팬) + @track(tool 스팬) 조합
겪은 이슈        →  opik.ps1 파싱버그(우회) / 전역 설정파일 workspace 충돌(env var 격리)
검증             →  REST API로 trace 서버 저장 직접 확인(span_count/llm_span_count)
첫 실패 실측     →  "무단투기" 벡터검색 실패 -> 별칭 사전 필요성 증명, 착수는 보류
```

---

## 관련 문서

- [[0818_KG - suwon_graph 조직도 구현 정리]]
- [[KG - Hybrid RAG]]
- [[KG - 지식그래프 구현 시안 내용]]
- [[KG - 홉 탐색 전략 (Hop Depth Strategy)]]
