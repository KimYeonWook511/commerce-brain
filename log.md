# commerce-brain 운영 기록

append-only. 각 항목 형식: `## [YYYY-MM-DD] {ingest|query|lint|setup} | <대상>`

## [2026-07-10] setup | commerce-brain 초기화

- 독립 저장소로 commerce-brain 생성. 구조: `raw/{sessions/<platform>,meetings,specs,images}`, `wiki/{decisions,contracts,topics,incidents,features,knowledge}`.
- 운영 파일: `CLAUDE.md`(schema·정책), `index.md`, `log.md`, `_tag-glossary.md`, `_skipped.md`.
- brain 자체 skill: `ingest`·`query`·`lint`. 코드 repo(side-repo)에 brain 연동 저장·조회 skill.
- 플랫폼 축: backend·frontend·infra (예정: android·ios).
