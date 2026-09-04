## 개념

`suwon_graph`는 수원시 조직도 지식그래프를 Neo4j로 새로 구현한 프로젝트다(2026-08-18 착수). 기존 `suwon_ontology_opus`(Class 노드 + `TYPE_OF` + `SUBCLASS_OF` 기반, 97% 정확도까지 검증됨)의 설계를 그대로 가져오지 않고, **멀티라벨 직접 부여 방식**으로 처음부터 다시 설계했다 — 타입 개수가 적고 고정적인 도메인이라 타입을 별도 노드로 승격시킬 필요가 없다고 판단했기 때문이다. 최종 목표는 처리부서·민원유형·필요서류 등 복합조건 질문에 답하는 것이고, 이번 단계는 그 기반이 되는 조직도만 구현했다(Layer 2는 스키마 자리만 예약).

---

## 온톨로지 구조

```
:OrgUnit:Entity   조직 계층 전체 (시장/부시장 포함), 임베딩 대상
:Employee         모든 직원 (시장/부시장 개인 포함), 임베딩 안 함

(:OrgUnit)-[:PART_OF]->(:OrgUnit)     조직 계층, 가변길이
(:Employee)-[:WORKS_AT]->(:OrgUnit)   소속, 1홉 고정
```

속성: `OrgUnit(org_id, name, unit_level, main_task, location, fax, embedding, embedding_text, last_seen, status)`, `Employee(employee_id, name, position, phone, duty, last_seen, status)`. `org_id`/`employee_id`는 원본 크롤링 deptPart 코드를 그대로 재사용(재크롤링 시 `MERGE` idempotency 확보 목적).

---

## 핵심 설계 결정

| 결정 | 이유 |
|---|---|
| Class 노드/`TYPE_OF`/`SUBCLASS_OF` 안 씀 | 타입이 적고 고정적 → 멀티라벨이면 그래프 순회 없이 라벨 인덱스로 바로 조회 가능 |
| Employee를 OrgUnit과 분리 | "조직"과 "사람"은 다른 개체. 같이 묶으면 `PART_OF`(가변) vs "소속"(1홉)이 뒤섞여 홉수 하드코딩 버그 재발 |
| Employee는 임베딩 안 함 | 벡터검색 목적은 "어느 부서가 담당인가". 직원 검색은 부서를 먼저 찾은 뒤 풀텍스트로 좁히는 2차 문제 |
| 시장/부시장을 OrgUnit **+** Employee 둘 다로 | 크롤링 원본에 실제로 두 역할이 다 있음(트리 부모 역할 + 리더 본인 직원 레코드). `SUPERVISES` 같은 관계 안 만들고 `PART_OF`에 그대로 둠 — 조상 경로에 낀 부시장은 노이즈가 아니라 유효 정보 |
| `unit_level`은 라벨이 아니라 속성 | "국 목록" 같은 구조질의는 인덱스 건 property 필터로 충분, 별도 계층 라벨/관계 불필요 |
| 최소 시간축 (`last_seen`/`status`만) | opus의 `valid_from`/`valid_to`/부활판정 전부는 과함. 재등장 시 MERGE가 자동으로 `status='active'` SET → 부활 로직 불필요 |

### 이름 분류 예외 (`ontology/classify.py`)

이름 끝 글자 기반 분류로 못 잡는 케이스를 override로 처리:

- **비서실/부속실**: `org_teams.json`의 "팀" 목록에 섞여 들어오고, 이름이 "실"로 끝나 총괄 단위로 잘못 묶일 뻔했으나 실제로는 실무(수행/일정관리) 조직이라 `unit_level="비서실"`로 분리, 임베딩 롤업 제외 목록에서도 뺌.
- **도서관/박물관/미술관**: "관"으로 끝나 담당관으로 잘못 분류될 뻔했으나 시민 대상 시설이라 "시설"로 우선 판정.

---

## 임베딩 롤업 규칙

노드마다 독립적으로 판단(위로 전파 안 됨):

```
1순위: 주요업무(main_task) 있으면 그걸로
2순위: 없고, unit_level이 ROLLUP_EXCLUDE(시장/부시장/실/국/직속기관/구/의회사무국) 아니면
       -> 직속(1홉, WORKS_AT) 활성(status='active') 직원의 duty로 대신 채움
3순위: 둘 다 없으면 스킵 (벡터 인덱스에 안 잡힘)
```

직속 직원만 본다(손자뻘 제외, 팀 자신의 임베딩과 중복 방지). 코드 리뷰에서 "폐지된 직원의 duty가 롤업에 계속 섞이는" 버그를 발견해 `status='active'` 필터를 추가 수정했다.

---

## 구현 현황 (2026-08-18 실행 검증)

| 항목 | 수치 |
|---|---|
| OrgUnit | 939개 |
| Employee | 5,089명 |
| PART_OF | 937개 |
| WORKS_AT | 5,089개 |
| 고아 노드 | 0개 |
| 임베딩 완료 | 914/939개 (총괄 제외 23개, 나머지 데이터 없음으로 스킵) |

검증한 것: `행정지원과 → 기획조정실 → 제1부시장 → 시장` 경로 정상 순회, 비서실/부속실 롤업 임베딩 정상 생성, 국(9개) 전부 임베딩 없음(제외 정상), 시장(Employee)이 시장(OrgUnit)에 `WORKS_AT`으로 연결(이중 표현 정상).

**다음 단계**: 검색 파이프라인(질문 라우팅 + 컨텍스트 조립 + LLM 답변), Opik self-hosted 트레이싱 연동, Layer 2(민원/서류/법령) 확장 — 전부 미착수.

---

## 한 눈에 요약

```
프로젝트     →  suwon_graph (Neo4j + Opik 예정), 조직도만(Phase 1)
설계 방향    →  opus(Class 노드) 대신 멀티라벨 직접 부여로 단순화
라벨         →  OrgUnit(임베딩 O) / Employee(임베딩 X)
관계         →  PART_OF(조직 계층, 가변) / WORKS_AT(소속, 1홉)
시장/부시장   →  OrgUnit + Employee 둘 다 (크롤링 원본 구조 그대로)
시간축       →  최소 버전(last_seen/status), 부활은 MERGE가 자동 처리
임베딩       →  주요업무 우선 -> 직속 활성직원 롤업 -> 스킵
실행 결과    →  OrgUnit 939 / Employee 5,089 / 임베딩 914, 고아노드 0
```

---

## 관련 문서

- [[KG - 지식그래프 구현 시안 내용]]
- [[KG - Layer 정리]]
- [[Property Graph - KG 구축 2가지 방법]]
- [[Graph DB - 구성요소]]
- [[Graph DB - 제약조건(3 Layer)]]
