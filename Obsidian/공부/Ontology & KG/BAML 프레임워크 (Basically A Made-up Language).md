---
수집일: 2026-10-02
tags:
---

## 개념

**BAML(Basically A Made-up Language)** 은 LLM 호출을 **입출력 타입이 있는 함수**로 정의하는 **DSL(도메인 특화 언어)** 입니다.
`.baml` 파일에 타입과 프롬프트를 선언하면, 컴파일러가 파이썬·TypeScript·Go 등의 **타입 안전한 클라이언트 코드를 생성**해 줍니다. BoundaryML 에서 만들었고 **Apache-2.0** 라이선스입니다.

한 줄로 말하면 **"프롬프트를 함수 선언처럼 쓰고, 깨진 출력은 스키마에 맞춰 알아서 복구하는 계층"** 입니다.

---

## 왜 쓰는가 — 기존 방식의 문제

구조화된 출력을 받으려면 보통 아래 코드가 한 파일에 섞입니다.

```python
prompt = f"""다음 글에서 조직을 추출해. JSON 으로만 답해.
{{"name": "...", "parent": "..."}}
글: {text}"""
resp = client.messages.create(...)
raw = resp.content[0].text
raw = raw.strip().removeprefix("```json").removesuffix("```")   # 마크다운 펜스 제거
try:
    data = json.loads(raw)                                      # 깨지면 예외
except json.JSONDecodeError:
    ...                                                          # 재시도? 수동 교정?
obj = Org(**data)                                                # 키 이름 틀리면 또 예외
```

| 흩어지는 것 | 내용 |
|---|---|
| 스키마 | 프롬프트 문자열 안의 예시 JSON **과** 파이썬 클래스에 **두 번** 적는다 |
| 파싱 | 마크다운 펜스·앞뒤 설명·trailing comma 처리 코드를 직접 쓴다 |
| 재시도 | 실패 조건과 백오프를 호출부마다 다시 쓴다 |
| 모델 교체 | 프로바이더별 호출 코드가 호출부에 박힌다 |
| 프롬프트 확인 | **실제로 모델에 간 최종 문자열**을 보기 어렵다 |

BAML 은 이 다섯 개를 선언 하나로 모읍니다. **스키마를 한 곳에만 적고, 프롬프트에는 그 스키마가 자동으로 렌더링**됩니다.

---

## 구조

```
baml_src/                     ← 내가 쓰는 곳 (.baml 소스)
  clients.baml                   모델·재시도 정책 선언
  extract.baml                   타입(class/enum) + 함수(function) + 테스트(test)
  generators.baml                generator 블록
        │
        │  baml-cli generate     ← 코드 생성 (CLI 또는 저장 시 자동)
        ▼
baml_client/                  ← 자동 생성물 (직접 수정하지 않음)
  types.py                       Pydantic 모델
  __init__.py                    b.FunctionName(...) 호출 진입점
        │
        ▼
내 애플리케이션 코드             from baml_client import b
```

**핵심은 "프롬프트 = 함수 선언"** 입니다. 호출부에는 프롬프트 문자열도, 파싱 코드도 남지 않습니다.

---

## 핵심 구성 요소

### 함수(function)와 클래스(class)

```baml
class Organization {
  name        string        @description("조직의 공식 명칭")
  parent      string?                                   // ? = 선택 항목
  level       OrgLevel                                  // enum 참조
  phone       string?       @alias("대표전화")           // 프롬프트에 보일 이름
}

enum OrgLevel {
  Bureau   @description("국")
  Division @description("과")
  Team     @description("팀")
}

function ExtractOrgs(page_text: string) -> Organization[] {
  client ClaudeOpus
  prompt #"
    아래 조직도 페이지에서 조직을 모두 추출해.

    {{ ctx.output_format }}

    {{ _.role("user") }}
    {{ page_text }}
  "#
}
```

| 문법 | 뜻 |
|---|---|
| `string?` | 선택(Optional) |
| `Organization[]` | 배열 |
| `string \| int` | 유니언 |
| `map<string, string>` | 맵 |
| `@description` | 프롬프트에 함께 렌더링되는 필드 설명 |
| `@alias` | 모델에게 보여줄 필드 이름만 바꿈 (코드에서는 원래 이름 유지) |
| `#" ... "#` | 여러 줄 문자열 블록 (안에서 Jinja 문법 사용) |
| `{{ ctx.output_format }}` | 출력 스키마 자동 삽입 |
| `{{ _.role("user") }}` | 이 아래부터 user 메시지로 분리 |

### 클라이언트(client)와 재시도 정책(retry_policy)

```baml
retry_policy Exponential {
  max_retries 3
  strategy {
    type exponential_backoff
    delay_ms 200
  }
}

client<llm> ClaudeOpus {
  provider anthropic
  retry_policy Exponential
  options {
    model "claude-opus-5-5"
    api_key env.ANTHROPIC_API_KEY
  }
}
```

`provider` 로 `anthropic` · `openai` · `google-ai` · `vertex-ai` · `aws-bedrock` · `azure-openai` · `ollama` · `openai-generic` 등을 지정합니다. **모델을 바꿀 때 애플리케이션 코드는 손대지 않습니다.**

### 테스트(test)

```baml
test 조직도_1페이지 {
  functions [ExtractOrgs]
  args {
    page_text #"
      기획조정실
        ├ 기획예산과
        └ 법무담당관
    "#
  }
}
```

에디터 확장(VS Code 등)의 플레이그라운드에서 **최종 렌더링된 프롬프트를 눈으로 보고** 바로 실행할 수 있습니다. 프롬프트를 추측하지 않아도 되는 게 실무에서 가장 큰 차이입니다.

### 생성기(generator)

```baml
generator target {
  output_type "python/pydantic"      // typescript, go, ruby, rest/openapi 등
  output_dir "../"
  version "0.xx.x"                   // 설치된 BAML 패키지 버전과 일치해야 함
  default_client_mode "sync"         // sync | async
}
```

### 호출

```python
from baml_client import b

orgs = b.ExtractOrgs(page_text=text)   # -> list[Organization], 타입 완성 지원
print(orgs[0].name, orgs[0].level)
```

---

## 스키마 정렬 파싱 (Schema-Aligned Parsing, SAP)

BAML 의 가장 중요한 기술입니다. **JSON 모드나 함수 호출로 모델을 강제하는 대신, 모델이 자유롭게 답하게 하고 받은 쪽에서 스키마에 맞춰 복구**합니다.

```
[모델 출력]
"음, 찾아보니 이런 것 같습니다:
```json
{ name: '기획예산과', parent: "기획조정실", level: Division, }   ← 키 따옴표 없음,
```                                                              작은따옴표, trailing comma
이상입니다."
        │
        ▼ SAP — "스키마에 맞게 만들려면 최소 몇 번 고쳐야 하나"를 계산 (편집 거리 발상)
        ▼
Organization(name="기획예산과", parent="기획조정실", level=OrgLevel.Division)
```

고쳐 주는 오류의 종류는 이렇습니다.

| 분류 | 예 |
|---|---|
| 문법 오류 | 주석, trailing comma, 따옴표·콜론 누락, 이스케이프 안 된 줄바꿈 |
| 타입 불일치 | 배열이어야 하는데 문자열, 소수여야 하는데 분수 |
| 구조 문제 | 괄호 누락, 키 이름 오타, 스트리밍 중간의 미완성 출력 |
| 군더더기 | 앞뒤로 붙은 설명 문장("yapping") |
| 후보 여럿 | 모델이 여러 안을 내놓았을 때 가장 맞는 것 선택 |

핵심 발상은 **"보낼 때는 엄격하게, 받을 때는 관대하게"(Postel's Law)** 입니다. 비용 함수가 문자 유사도가 아니라 **스키마를 아는** 상태로 계산된다는 점이 일반 JSON 복구 라이브러리와 다릅니다.

### 벤더 벤치마크 (Berkeley Function Calling Leaderboard, n=1000)

| 모델 | 함수 호출 | Python AST | **SAP** |
|---|---|---|---|
| gpt-3.5-turbo | 87.5% | 75.8% | **92%** |
| gpt-4o | 87.4% | 82.1% | **93%** |
| claude-3-haiku | 57.3% | 82.6% | **91.7%** |
| gpt-4o-mini | 19.8% | 51.8% | **92.4%** |
| claude-3-5-sonnet | 78.1% | 93.8% | **94.4%** |

> BoundaryML 이 직접 낸 수치이고 측정 모델도 구세대입니다. 숫자 자체보다 **"네이티브 구조화 출력이 없는 모델에서도 성공률이 평탄해진다"** 는 경향을 보는 것이 맞습니다. 로컬 모델(ollama 등)을 쓸 때 이 성질이 특히 크게 작용합니다.

---

## 출력 스키마 렌더링 (ctx.output_format)

`{{ ctx.output_format }}` 는 프롬프트 안에 **JSON Schema 가 아니라 타입 정의문**을 넣습니다.

```
Answer in JSON using this schema:
{
  name: string
  education: [
    {
      school: string
      graduation_year: string
    }
  ]
}
```

BAML 문서는 JSON Schema 를 쓰지 않는 이유를 **"타입 정의보다 4배 비효율적이고, 사람(따라서 모델)이 읽기 매우 어렵다"** 고 밝히고 있습니다. 중첩이 깊거나 작은 모델일 때 차이가 커집니다.

| 옵션 | 역할 |
|---|---|
| `prefix` | 스키마 앞 안내 문구 교체 |
| `always_hoist_enums` | enum 정의를 스키마 위로 분리 |
| `or_splitter` | 유니언·옵셔널 표기 문자 (기본 `or`) |
| `hoist_classes` | 어떤 클래스를 위로 뺄지 (`auto` / true / false / 클래스명) |
| `hoisted_class_prefix` | 분리된 클래스 앞에 붙일 라벨 (예: `interface`) |

---

## 런타임 타입 추가 (@@dynamic + TypeBuilder)

`.baml` 에 박아 둔 타입만으로는 **온톨로지가 버전마다 바뀌는 작업**을 감당할 수 없습니다. 이때 쓰는 장치입니다.

```baml
class Organization {
  name string
  @@dynamic          // 런타임에 필드를 더할 수 있게 표시
}
```

```python
from baml_client.type_builder import TypeBuilder
from baml_client import b

tb = TypeBuilder()
tb.Organization.add_property('email', tb.string()).description("대표 이메일")

levels = tb.add_enum("OrgLevel")          # 클래스·enum 자체를 새로 만들 수도 있음
for v in load_levels_from_ontology():     # 온톨로지에서 읽어온 값으로 채우기
    levels.add_value(v)
tb.Organization.add_property('level', levels.type().optional())

res = b.ExtractOrgs("...", {"tb": tb})
```

**온톨로지 정의(YAML·DB)에서 읽어온 클래스와 통제 어휘를 그대로 추출 스키마로 바꿀 수 있다**는 뜻입니다.

---

## 온톨로지·지식그래프 작업에서의 쓸모

| 온톨로지 쪽 | BAML 쪽 |
|---|---|
| 노드 라벨(Label) | `class` |
| 노드 속성(Property) | 클래스 필드 + `@description` |
| 통제 어휘 / 분류 체계 | `enum` (오타·유사어를 enum 값으로 흡수) |
| 관계(Relationship) | 양 끝 식별자와 관계 타입을 담은 `class` 를 반환하는 함수 |
| 필수/선택 속성 | `string` vs `string?` |
| 온톨로지 버전 변경 | `@@dynamic` + TypeBuilder |

적재 파이프라인에서의 위치는 이렇습니다.

```
원문(HTML·PDF·게시글)
    ↓ 청킹
    ↓ ★ BAML 함수 — 텍스트 → 타입 있는 객체        ← 여기만 담당
    ↓ 검증 (식별자 존재? 관계 양끝 존재? 중복?)      ← BAML 이 해주지 않는 부분
    ↓ Cypher MERGE 적재
Neo4j
```

**BAML 은 "타입에 맞는 객체"까지만 보장합니다.** 값이 **사실인지**, 그래프에 **실제로 존재하는 노드를 가리키는지**는 별도의 검증 단계가 그대로 필요합니다. SAP 가 관대하다는 것은 **틀린 값도 그럴듯하게 스키마에 맞춰 들어온다**는 뜻이기도 해서, 오히려 검증을 느슨하게 하면 더 위험합니다.

---

## 비교

| | 방식 | 스키마 선언 위치 | 구조 강제 방법 | 특징 |
|---|---|---|---|---|
| **BAML** | DSL + 코드 생성 | `.baml` 한 곳 | **SAP**(관대한 파싱) | 프롬프트·타입·모델·재시도·테스트가 한 파일. 빌드 단계가 늘어남 |
| **Instructor** | 파이썬 라이브러리 | Pydantic 모델 | 네이티브 함수 호출 + 검증 실패 시 재시도 | 기존 코드에 가장 적게 침습. 프롬프트는 여전히 코드 안 문자열 |
| **네이티브 구조화 출력** | API 기능 | API 요청의 스키마 | 제약 디코딩 / 스키마 강제 | 가장 확실하지만 **지원 모델에서만** 가능 |
| **LangChain 출력 파서** | 체인 프레임워크 | 파서 객체 | 포맷 지시문 + 파서(+ 교정 파서) | 체인 생태계가 필요할 때. 추상화 층이 두꺼움 |

고르는 기준은 **"모델을 바꿔 가며 쓸 것인가"** 입니다. 한 프로바이더의 최신 모델만 쓴다면 네이티브 구조화 출력이 가장 단순하고, 로컬·구세대·여러 프로바이더를 섞는다면 BAML 의 SAP 가 값을 합니다.

---

## 한계와 도입 비용

| 항목 | 내용 |
|---|---|
| 새 언어 | `.baml` 문법과 Jinja 템플릿을 새로 배워야 한다 |
| 빌드 단계 | `baml-cli generate` 가 빌드·CI 에 들어가고, 생성물 관리 규칙(커밋 여부)을 정해야 한다 |
| 버전 고정 | `generator` 의 `version` 과 설치된 런타임 버전이 **맞아야** 한다 |
| 동적 프롬프트 | 프롬프트가 `.baml` 안에 있어, 런타임에 문자열을 조립하는 방식은 제약이 있다 (TypeBuilder·ClientRegistry 로 일부 해소) |
| 에디터 의존 | 플레이그라운드 이점은 에디터 확장 설치가 전제다 |
| 벤더 의존 | 상대적으로 젊은 스타트업 프로젝트다 (Apache-2.0 이라 포크는 가능) |
| **관대함의 역설** | 파싱이 관대하다는 것은 **틀린 값도 통과한다**는 뜻이다. 사실 검증은 반드시 따로 둔다 |

---

## 한 눈에 요약

```
BAML        →  LLM 호출을 '타입 있는 함수'로 선언하는 DSL (Apache-2.0)
흐름        →  baml_src/*.baml  →  baml-cli generate  →  baml_client  →  내 코드
핵심 기술   →  SAP: 모델을 강제하지 않고, 받은 출력을 스키마에 맞춰 복구
프롬프트    →  {{ ctx.output_format }} 가 타입 정의문을 삽입 (JSON Schema 대비 1/4 토큰)
온톨로지    →  class=라벨 / enum=통제어휘 / @@dynamic=버전 변경 대응
담당 범위   →  "타입에 맞는 객체"까지. 사실 검증과 적재는 여전히 내 몫
쓸 때       →  여러 모델·로컬 모델을 섞고, 추출 스키마가 자주 바뀔 때
안 쓸 때    →  한 프로바이더 최신 모델만 쓰고 스키마가 단순할 때
```

---

## 관련 문서

- [[S03. 전통 구축 방법론 2 - 텍스트 기반 온톨로지 자동 추출]]
- [[프로퍼티 그래프 온톨로지 모델링 기준 (Property Graph Ontology Modeling Criteria)]]
- [[온톨로지 레이어 구조와 Layer 2 확장 (Ontology Layer Structure and Layer 2 Expansion)]]
- [[지식그래프와 RAG 종류 (Knowledge Graph & RAG)]]
- [[청킹 종류 (Chunking Types)]]

---

##### 참고문헌

- BAML 공식 문서 — https://docs.boundaryml.com
- Schema-Aligned Parsing 소개 글 — https://www.boundaryml.com/blog/schema-aligned-parsing
- 저장소 — https://github.com/BoundaryML/baml
