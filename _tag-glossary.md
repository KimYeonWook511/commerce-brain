---
type: meta
updated: 2026-07-10
---

# 태그 사전 (tag glossary)

표준 태그와 동의어를 관리한다. **빈 상태로 시작해 ingest가 자라게 한다.**

- canonical 형태: **소문자·하이픈·단수형** (`Refresh Tokens` → `refresh-token`).
- 동의어는 canonical로 정규화한다 (`React` → `react`).
- `platform` 태그군은 분류 축이라 아래에 별도로 둔다.

## platform 태그군 (분류 축)

| canonical | 동의어 |
|---|---|
| `backend` | server, 서버, api |
| `frontend` | web, 웹, client |
| `infra` | devops, ops |
| `android` | aos |
| `ios` | — |

> `android`(주로 React Native)·`ios`는 예정 플랫폼이다. 어떤 코드 repo가 어떤 platform인지는 각 repo의 `.brain`이 선언한다(이 glossary는 repo 이름에 의존하지 않는다).

## 일반 태그 (성격·기능)

_(아직 없음 — ingest가 등재. 예: `auth`, `login`, `payment`, `security`, `db` …)_

### 동의어 매핑

_(아직 없음)_

### 상위/하위 관계

_(아직 없음)_
