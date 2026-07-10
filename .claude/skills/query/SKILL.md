---
name: query
description: Use when wiki 내용에 대해 질문할 때 — "찾아줘", "query", "wiki 에서", "왜 이렇게 결정했지" 또는 wiki 가 답할 수 있는 사실 질문. index.md + frontmatter + wikilink 를 검색해 인용과 함께 답하고, 사람에겐 서술형 / LLM 에이전트에겐 구조화(JSON)로 응답한다.
---

# Query

commerce-brain wiki 에 질문을 던져 답을 찾는 운영 절차. 검색 로직은 소비자와 무관하게 동일하고, **출력 형태만** 호출 방식에 따라 갈린다.

## 전제
- `CLAUDE.md` 의 schema·정책을 따른다.
- 답은 **항상 인용과 함께** — `[[wiki/...]]`, `[[raw/...]]`, `{repo, path}`. 인용 없는 주장 금지.
- 소비자 둘: **사람**(옵시디언, 서술형) / **LLM 에이전트**(의사결정 중 정보 부족 시 호출, 구조화 JSON). 검색 엔진은 공유.

## 절차

1. **질문 분해**
   - 핵심 키워드 · `platform` · `type` 추출.
   - 예: "로그인 토큰 왜 이렇게 했지?" → 키워드 `login`,`token`,`refresh` / type `decision`·`api-contract` 가능성 / platform `backend` 가능성.

2. **후보 좁힘**
   - `index.md` 카탈로그 + `_tag-glossary.md`(동의어·상하위로 키워드 확장)로 후보 list.
   - `platform`/`tags` 로 1차 스코프. (분류 축은 platform 단일 — 레퍼런스의 project/domain 아님.)
   - **규모 폴백**: wiki 가 수백 페이지를 넘어 index.md 통독이 비효율이면, BM25/벡터 검색엔진(또는 grep+리랭커)으로 후보를 뽑는다.

3. **frontmatter + wikilink 그래프로 추리기**
   - 후보의 frontmatter(`type`,`platform`,`tags`,`status`,`superseded_by`,`sources`) 확인.
   - wikilink 그래프(backlink 포함) 따라 확장. 기능 질문이면 `features/<feature>` MOC 가 좋은 진입점.
   - 필요시 `[[raw/...]]` 까지 거슬러 원본 확인.
   - **status 처리**: `superseded`/`deprecated` 는 기본 **하향·제외**하되, 관련되면 결과에 status 와 `superseded_by` 를 **노출**한다 ("이 결정은 [[...]]로 대체됨"). 낡은 결정을 최신인 양 답하지 않는다 — 에이전트가 폐기된 결정으로 코딩하는 것을 막는 핵심.

4. **답변 — 출력 형태 분기**
   - **사람(기본)**: 서술형 + 인라인 `[[페이지명]]` 인용. 긴 답은 `## 근거` 섹션에 출처 모음. 모순 발견 시 양쪽 인용 + `> [!warning]`.
   - **LLM 에이전트(JSON 모드로 호출 시)**: 아래 구조로. 모든 주장에 citation, 각 citation 에 `status`·`platform`·`sources`.
     ```json
     {
       "answer": "refresh token 15분 + rotation. 탈취 시 피해 최소화가 이유.",
       "citations": [
         { "note": "wiki/decisions/login-token-rotation.md",
           "status": "accepted", "platform": "backend",
           "sources": ["raw/sessions/backend/2026-06-24-login-token-rotation"] }
       ],
       "superseded": []
     }
     ```
   - 양쪽 공통: 답이 모드 A(companion) 노트에 의존하면 정본은 `{repo, path}` 로 인용하고 wiki 는 "이해·트레이드오프"만 인용.

5. **새 통찰은 raw 로 (query 는 wiki 에 쓰지 않음)**
   - query 는 **읽기 전용**이다. wiki 에 새 페이지를 직접 생성하지 않는다 (모든 wiki 노트는 raw 에 뿌리를 둬야 재생성 가능 불변식이 유지됨).
   - 단순 조회가 아니라 *여러 페이지를 종합한 새 통찰* 이 나오면: "이 통찰을 `raw/sessions/<platform>/<...>.md` 로 떨궈 ingest 할까요?" 제안. OK 시 **raw 에** 기록하고, 다음 ingest 가 정식 경로로 wiki 에 올린다. (에이전트 JSON 모드에선 반환만.)

6. **`log.md`** — 단순 조회는 생략. 저장·새 통찰이 일어나면 `## [YYYY-MM-DD] query | <질문 요약>`.

## companion 모드 처리
- 답이 코드 정본에 의존하면 `{repo, path}` 로 인용. wiki 페이지 자체는 "이해"만 인용, 정본 본문을 wiki 에서 복사해 답하지 않는다.
- 정본 본문이 필요하면 read-only 로 `repo/<path>`(OpenAPI/proto/config 등) 를 읽어 답에 반영 가능.

## 빈 답 처리
- 후보 0개면 솔직히 알림 + raw 후보 제시: "wiki 엔 없습니다. `raw/sessions/<platform>/<...>.md` 가 아직 미처리인 듯 — ingest 할까요?"
- 추측으로 메우지 않는다.
