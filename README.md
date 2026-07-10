# commerce-brain

commerce 제품(backend·frontend·infra, 예정: android·ios)의 **결정·맥락·트레이드오프·"왜"·미해결 쟁점**을 담는 LLM Wiki. 코드가 아니라 *코드를 그렇게 짠 이유*를 기록한다.

- **스키마·정책**: [`CLAUDE.md`](CLAUDE.md)
- **입력**: `raw/`(불변) — 사람/`brain-save` skill이 세션·회의·스펙을 올린다.
- **정리된 지식**: `wiki/`(LLM 소유) — `ingest`가 raw를 타입별로 분해해 만든다.
- **카탈로그**: [`index.md`](index.md) · **운영 기록**: [`log.md`](log.md)

## 쓰는 법

- 코드 repo(commerce-backend 등)에서 작업하다가 세션 핵심을 남길 때 → `brain-save` skill (raw로 자동 저장·커밋).
- 과거 결정을 찾을 때 → `brain-query` skill (읽기 전용, 인용과 함께 답).
- brain 자체 세션에서 raw를 wiki로 정리 → `ingest` / 검색 → `query` / 검수 → `lint` (`.claude/skills/`).

## 연결

각 코드 repo 루트의 `.brain` 파일(gitignore)이 `platform`과 이 brain의 상대경로(`brain_path`)를 담아, side-repo skill이 자동으로 이 저장소를 찾는다.
