---
name: lint
description: Use when wiki health check 가 필요할 때 — "점검해줘", "lint", "wiki 점검", "정리 상태 확인". 모순·중복·낡음·계약 drift·고립·태그 산발·frontmatter·companion 위반 등을 점검하고 수정 제안 후 log.md 에 기록한다.
---

# Lint

commerce-brain wiki 의 health check 운영 절차.

## 전제
- `CLAUDE.md` 의 schema·정책·톤을 기준으로 점검.
- ingest 가 완전 자동으로 돌기 때문에, **lint 가 사람이 주기적으로 검수하는 자리**다. 단, *지적만 하지 말고 수정 제안과 함께* 하고, **사용자가 OK 한 항목만 실제 수정**한다 (wiki 는 봇 소유지만 lint 의 구조 변경은 승인 기반).

## 점검 항목

### 1. 모순된 주장
서로 다른 페이지가 같은 주제에 충돌. `> [!warning] 모순` + 양쪽 보존(임의 삭제 금지). 최근 raw 가 뒤집은 경우 `superseded` 처리.

### 2. 중복 결정 (dedup 보완)
같은 결정이 두 노트로 갈라진 경우 — ingest 의 가벼운 태그·제목 매칭이 놓친 것을 여기서 그물질. 병합하거나, 한쪽을 `superseded`/`duplicate` 로 정리 제안.

### 3. 오래된 주장
`updated:` 가 낡은 페이지, 최근 raw 가 갱신했는데 미반영. `status: outdated`/`superseded` 후보.

### 4. 계약 drift *(신규)*
`sources` 에 `{repo, path}` 가 있는 companion 노트에서, 정본 코드가 바뀐 듯한데 wiki 요약 `updated:` 는 그대로인 경우. 정본을 read-only 로 확인해 재요약 제안.

### 5. 고립 페이지
backlink 0건. **예외**: platform `README`, 폴더 안내용 `README.md`(frontmatter 없는 placeholder), `features/` MOC 는 고립으로 보지 않는다.

### 6. 없는 개념 / MOC 생성·정합성
- 한 태그가 **5개+** 모였는데 `features/<tag>.md` 가 없으면 생성 제안.
- 자주 언급되는데 전용 `topic` 페이지가 없으면 생성 제안.
- 기존 MOC 의 링크 목록이 현재 `tags` query 와 어긋나면(ingest 재생성 누락) 덮어쓰기 재생성.

### 7. 누락된 cross-reference
A 가 B 의 개념을 쓰면서 `[[B]]` 가 빠짐. `sources` 누락 포함.

### 8. 태그 산발
glossary 미등록 태그, 동의어 산발(`React`/`react`/`리액트` → canonical 정규화), 고립 태그. `platform` 태그군은 별도 관리.

### 9. frontmatter 누락·오류
- type별 필수 필드 누락 (특히 `platform`, `sources`).
- `status` 가 enum 범위 밖.
- `api-contract` 인데 `sources` 가 `{repo, path}` 형식이 아님.
- **추적성**: `decision` 에 `author`/`decided_by` 누락 (soft warning).

### 10. companion 위반
companion 노트(또는 `api-contract`)가 정본 계약/ADR 본문을 *복사* 한 흔적. 본문이 정본의 80%+ 길이면 의심 → 요약·인용 모드로 정정.

### 11. 폴더 승격 후보
타입 폴더(또는 루트) 평탄에 비슷한 성격 노트가 **5개+** 쌓이면 새 폴더(예: `retrospectives/`, `glossary/`) 승격 + 이동 제안.

### 12. promote 후보
한 `platform` 노트가 *다른 platform* 에서도 인용되기 시작하면 → `wiki/knowledge/<topic>.md` 로 promote, 원본은 stub + `[[link]]`.

## 결과 보고
- 항목별 발견 list + 각 수정 제안(자동/수동 표시). 사용자 OK 분만 수정.
- 수정이 wiki 를 바꿨으면 `lint: <날짜> <요약>` 형식 커밋으로 제안한다 (CLAUDE.md 커밋 컨벤션). 실제 커밋·push 는 사용자가 확인 후 실행.
- `log.md` 항목:
  ```
  ## [YYYY-MM-DD] lint | <대상>
  - 발견: N건 (모순 a, 중복 b, 계약 drift c, 태그 산발 d, ...)
  - 수정: M건 (사용자 승인)
  - 후속 제안: ...
  ```

## 후속 질문 제안
점검 중 발견한 *질의 가치 있는* 패턴을 query 후보로 보고.
예: "login 결정 6개인데 'why-not-session-cookie' 노트는 없음 — 결정 노트 생성 제안?"
