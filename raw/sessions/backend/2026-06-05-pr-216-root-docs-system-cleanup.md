---
platform: backend
author: KimYeonWook511
created: 2026-06-05
origin:
  - { type: pr, repo: commerce-backend, ref: 216 }
---

## 한 일
- 루트 `docs/` 문서 역할 경계 재정의: PRD=제품 기능 상위 인덱스(각 task 상세는 흡수 않고 링크), ADR=프로젝트의 유일한 결정 타임라인, architecture=서비스·컴포넌트 전수 나열 표를 버리고 "코드/로깅 컨벤션/task 문서가 단일 출처"인 개념도로 슬림화.
- CLAUDE.md 재구성: 핵심 규칙을 최상단에, "시점별 규칙" 표 두 개 도입 — (1) 명령 실행 전 컨벤션 확인, (2) **코드 변경 후 루트 문서 동기화**(변경 종류별로 갱신할 루트 문서를 대조하는 표). PR 템플릿의 "문서/하네스 변경" 체크도 같은 대조 방식으로 교체.
- 참조가 전부 사라진 `docs/claude-harness.md` 삭제(CLAUDE.md의 자동 import와 참고문서 링크를 둘 다 제거하자 orphan화됨).
- 파일명 소문자 컨벤션 통일: `docs/PRD.md→prd.md`, `docs/ADR.md→adr.md`, `.github/ISSUE_TEMPLATE.md→issue_template.md`. 참조하는 모든 곳 함께 갱신.

## 결정한 것
- **문서 역할 분리 원칙**: 루트 문서는 "현재 상태의 개념"만, 코드 심볼(메서드·클래스명) 전수 나열은 코드가 단일 출처라 최소화. 중복되면 리팩터 때 문서가 stale해지므로. (루트 ADR=유일 타임라인, task adr=staging 결정은 파일 1의 template staging 개편과 짝.)
- **동기화를 "판단"이 아니라 "대조"로**: 코드 변경 후 루트 문서 갱신 여부를 그때그때 판단하지 말고, 변경 종류(API 계약/DB 스키마/구조/설계 결정/내부 구현)별 대응 문서를 표로 박아 대조만. CLAUDE.md와 PR 템플릿에 같은 표.
- **파일명 소문자화에서 codex 제외**: codex 하네스(`.codex/`)도 이 문서들을 참조하지만 codex 전면 개편이 별도 이슈(#103)로 예정돼 이번엔 의도적 제외. 단 루트 codex 진입점 가이드(AGENTS.md)는 `.codex/` 바깥이라 포함해 갱신.
- **완료 task 문서 불변 원칙의 예외**: 머지된 task 폴더 문서는 불변이 원칙인데, 파일명을 바꾸면 안의 `docs/ADR.md` 참조가 깨진 링크가 된다. 불변 고수(broken link 수용) vs 예외적 일괄 수정 갈림길에서 후자 선택. 약 120개 task 문서 경로 참조 일괄 치환.

## 트레이드오프
- **macOS ↔ GitHub case 함정**: macOS 기본 파일시스템은 대소문자를 안 구분해 `docs/ADR.md` 링크가 `docs/adr.md`를 가리켜도 로컬에선 멀쩡히 열린다. 하지만 GitHub(Linux, 대소문자 구분)에선 깨진다. 로컬 확인만으론 안 보이고 GitHub 웹에서만 드러나는 broken link라, 잔재를 로컬 눈이 아니라 grep 전수 검색으로 검증.
- 완료 task 불변 원칙을 한 번 깬 것. 파일명 정합성을 우선했고, 별도 커밋으로 분리해 "왜 불변 문서를 건드렸나"를 기록으로 남김.

## 정본 위치
- `docs/prd.md`, `docs/adr.md`, `docs/architecture.md`, `CLAUDE.md`, `.github/pull_request_template.md`
- PR #216 (open, 미머지. 이슈 #214 해결). 파일 1(harness 9-stage 개편, 같은 세션의 다른 메모)과 짝.
