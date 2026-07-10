---
name: ingest
description: Use when raw/ 에 추가된 파일을 wiki/ 로 정리할 때 — "정리해줘", "ingest", "raw 정리" 또는 raw/meetings/, raw/sessions/<platform>/, raw/specs/ 에 파일을 추가한 직후. raw 원본을 읽어 플랫폼·타입별로 분해해 wiki 노트를 생성/갱신하고, MOC·index.md·log.md·_tag-glossary.md 를 동기화한다.
---

# Ingest

commerce-brain 의 `raw/` 에 추가된 파일을 `wiki/` 로 정리하는 운영 절차.

## 전제
- 먼저 이 vault 의 `CLAUDE.md` 를 읽어 schema·정책·톤을 따른다.
- `raw/` 는 불변 — 절대 수정·이동·삭제하지 않는다. `wiki/` 만 작성·갱신.
- 현재 **모드 A**: 대표가 직접 실행하되 **사전 질문 없이 완전 자동으로 진행**한다. 사람 개입은 (a) 맥락이 꼭 필요한 극소수 예외 + (b) 끝난 뒤 요약 보고뿐. (아래 "자율 vs 확인" 참조.)

## 자율 vs 확인 (모드 A 상호작용)
**기본은 자율 — 사람 개입 없이 끝까지 진행한다.** 아래 "확인" 항목은 *봇이 raw 작성자의 맥락 없이는 옳게 판단할 수 없을 때만* 예외적으로 묻는다. 판단 기준 한 줄: **"이 raw 를 쓴 사람의 맥락이 있어야만 답할 수 있나?"** — 아니면 자율, 맞으면 확인.

봇이 **자율로 정함** (기본, 묻지 않음): 플랫폼·타입 귀속, frontmatter 채움, 태그 정규화, 폴더 배치, MOC/index/log/glossary 갱신, 그 외 내용만으로 답이 정해지는 모든 것.

예외적으로 **확인** (맥락이 필요한 소수만):
- 한 덤프를 어떻게 쪼갤지 *진짜* 모호할 때
- 새 결정인지 기존 결정의 연장인지 (dedup 판단이 안 설 때)
- 기존 주장과 모순되는데 어느 쪽이 맞는지 (작성자 맥락 필요)
- companion(코드 정본) vs 모드 B(wiki 정본) 경계가 흐릴 때
- 이게 *결정*인지 단순 구현 메모인지

확인할 땐 답하기 쉬운 형태로 — 선택지가 명확하면 옵션으로, 아니면 짧은 질문 한 줄로. (자동화 모드 B 전환 시 이 확인 항목들이 사후 review 큐로 옮겨간다.)

## 절차

1. **대상 raw 식별 — 멱등성**
   - 사용자가 파일을 명시하면 *강제 재처리* (인용·skip 무시).
   - 아니면: `raw/meetings/`, `raw/sessions/<platform>/`, `raw/specs/` 전체를 list 하고, 각 파일이 wiki 에 인용됐는지 grep:
     ```
     grep -r "\[\[raw/sessions/<platform>/<basename>" wiki/
     grep -r "\[\[raw/meetings/<basename>" wiki/
     ```
   - *인용 0건* 이고 `_skipped.md` 에 없는 파일 = 미처리. **사전 확인 없이 미처리 전체를 자동 처리**한다 (특정 파일만 원하면 사용자가 먼저 명시).
   - **단, 폴더 안내용 `README.md`(frontmatter 없는 placeholder)는 후보에서 제외**한다 — raw 가 아니라 폴더 설명이므로. (lint 의 고립 예외와 동일 대상.)
   - **원칙**: wiki `sources:` wikilink 가 "이미 ingest 됨" 의 single source of truth.

2. **읽고 자율 분해**
   - 핵심 결정·검토한 대안·트레이드오프·미해결 쟁점·인물을 추출.
   - `origin` 확인: PR/이슈가 달렸으면 "코드에 반영된 결정" 신호(→ `accepted` 기울임), 없으면 논의 단계 가능성(→ `proposed` 기울임).
   - **분해안은 봇이 자율로 정해 바로 진행**하고, 결과는 끝내기 요약에서 보고한다. 사전 승인을 받지 않는다. 단, 분해가 *진짜* 모호하거나 맥락이 필요하면(아래 예외) 그때만 짧게 묻는다.

3. **분해 — 플랫폼 × 타입** *(레퍼런스 대비 핵심 변경)*
   - 한 덤프가 여러 플랫폼을 담으면 **플랫폼별로 쪼개고**, 각 조각을 타입으로 분류한다.
   - 예: 로그인 덤프 하나 →
     - 백엔드 인증 API 결정 → `decision`, `platform: backend`
     - 안드로이드 토큰 저장 → `decision`, `platform: android`
     - 웹 로그인 폼 → `decision`, `platform: frontend`
     - 세션 스토어(infra) → `decision`, `platform: infra`
     - 여러 플랫폼이 공유하는 토큰 계약 → `api-contract`, `platform: [backend, frontend, android]`
   - frontmatter `type` 은 노트당 1개. `platform` 은 단일 또는 배열.
   - **dedup/supersede**: 새 노트 쓰기 전, 같은 결정을 다룬 기존 노트가 있는지 **`tags`·제목 매칭**(grep)으로 확인. 표현이 달라 매칭이 애매하면 사용자에게 "기존 [[...]]와 같은 결정인가요?" 한 줄로 확인. (의미 검색은 query 검색엔진 도입 시 함께 업그레이드 — 지금은 가벼운 매칭, 놓친 중복은 lint 가 보완.)
     - 연장·보강이면 → 기존 노트 갱신 + `sources` 에 이번 raw 추가.
     - 뒤집은 결정이면 → 새 노트 `status: accepted`, 기존은 `status: superseded` + `superseded_by` 링크 (양쪽 보존).

4. **노트 생성·갱신 (라우팅)**
   - `raw/meetings/` → 보통 여러 `decision`/`incident`/`topic` 으로 *분해*, 회의 원본은 각 노트 `sources` 로 인용.
   - `raw/sessions/<platform>/` → 내용에 따라 `decisions/` / `topics/` / `incidents/` / `contracts/`.
   - `raw/specs/` → `topics/` (도메인 정리) 또는 관련 결정의 컨텍스트로.
   - 본문 구조는 CLAUDE.md "본문 권장 구조" 를 따른다.
   - 한 ingest 가 보통 **5~15개** wiki 페이지를 건드린다. 너무 적으면 cross-reference 를 빠뜨리고 있는 것.

5. **정본 모드 분기**
   - `sources` 에 `{repo, path}` 가 있거나 `type: api-contract` → **모드 A(companion)**: 계약/스펙 본문 *복사 금지*, 링크·요약·"왜 이 형태인가" 만.
   - 그 외 → **모드 B**: wiki 가 정본, 4섹션 ADR.

6. **frontmatter 표준화**
   - schema 는 CLAUDE.md 참조. `platform` 필수, `author`·`decided_by` 채움.
   - `sources:` 에 `[[raw/...]]` 또는 `{repo, path}` 인용 추가 — **이 wikilink 가 다음 ingest 의 skip 신호**.

7. **MOC 재생성** *(신규)*
   - 이번에 건드린 `tags` 각각에 대해, 그 태그를 단 노트가 **5개+** 모였으면 `wiki/features/<tag>.md` 를 생성/갱신.
   - MOC 의 링크 목록은 `tags: [<tag>]` query 결과로 **덮어쓰기 재생성** (사람 손 안 댐, SSOT 는 노트의 tags).

8. **`_tag-glossary.md` 갱신** — 새 태그는 **자율로 등재**하고 동의어 정규화(`React`→`react`). `platform` 태그군은 별도. 등재 내역은 끝내기 요약에 보고(사전 승인 없음).

9. **`index.md` 갱신** — 신규/변경 페이지를 카테고리(타입)별로 등록.

10. **`log.md` 항목** — `## [YYYY-MM-DD] ingest | <대상>` + 신규/업데이트 페이지 목록.

## Skip 처리
완전 자동이므로 사전 후보 확인이 없다. ingest 가 **자율로** 판단해, 추출할 결정이 없는 raw(잡담·중복·무가치)는 `_skipped.md` 에 경로를 등재한다 (raw 는 불변이므로 봇 소유 ledger 에). 다음 ingest 부터 후보에서 제외되며, 등재 내역은 끝내기 요약에 보고한다. (자동화 모드 B 전환 시엔 review 큐에서 사람이 확인.)

## 모순 처리
새 raw 가 기존 wiki 주장과 충돌하면 `> [!warning] 모순` 콜아웃 + 두 입장 보존. 임의 삭제 금지. 뒤집힌 쪽은 `superseded`.

## 톤
- 기술 영역(`decisions/`, `contracts/`, `topics/`): 정확함·구체성·기술어 보존.
- 회의록: 발언자 의도·합의/미합의를 왜곡 없이 보존. 합의 안 된 것을 합의된 것처럼 쓰지 않는다.

## 끝내기
- 요약 보고: 신규 N개 / 업데이트 M개 / 새 태그 K개 / MOC 갱신 / skip L개.
- wiki 변경은 `ingest: <날짜> <요약>` 형식 커밋으로 제안한다 (CLAUDE.md 커밋 컨벤션 — `wiki/` 를 쓰는 정상 경로는 `ingest:` 커밋뿐). 실제 커밋·push 는 사용자가 확인 후 실행.
- 다음 lint 후보가 보이면 한 줄 제안 (예: "login 태그 6개 — MOC·cross-link 점검 추천").
