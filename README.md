# commerce-brain

commerce 제품의 **결정 지식 베이스**. 여러 플랫폼(backend / frontend / infra, 예정: android / ios)이 한 제품을 만들며 흩어뜨리는 **결정·맥락·트레이드오프·"왜"·미해결 쟁점**을 한곳에 모은다. [LLM Wiki 패턴](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)으로 운영되며 — 사람은 원본(raw)을 떨구기만 하고, LLM이 그것을 읽어 구조화된 위키를 만들고 유지한다. 위키는 **Obsidian 형식**으로 저장된다 — 평범한 마크다운 + frontmatter + `[[wikilink]]` 그래프라, 별도 DB 없이 Obsidian으로 바로 열어 링크를 따라 탐색할 수 있다.

> 담는 것은 *코드를 그렇게 짠 이유*다. 코드 자체나 API 스펙(코드가 정본)은 담지 않고, 그 결정의 배경만 담는다.

## 왜 필요한가
"이거 왜 이렇게 결정했더라?"의 답은 보통 PR·회의·대화에 흩어져 사라진다. commerce-brain은 그 휘발성 맥락을 한 repo에 모아, 사람이든 AI 에이전트든 **출처와 함께** 다시 꺼내볼 수 있게 한다.

## 구조

```
commerce-brain/
├── CLAUDE.md            # 운영 헌법 (schema·정책·절차 기준)
├── index.md             # wiki 카탈로그
├── log.md               # 운영 기록
├── _tag-glossary.md     # 표준 태그 사전
├── _skipped.md          # ingest 가 제외한 raw 목록
├── raw/                 # 원본 (불변) — 사람/skill 이 떨구는 곳
│   ├── meetings/        # 회의·논의 기록
│   ├── sessions/<platform>/  # 플랫폼별 작업·대화 덤프
│   ├── specs/           # 요구사항·기획·디자인
│   └── images/
├── wiki/                # LLM 이 정리한 결과 (봇 소유)
│   ├── decisions/       # 결정의 "왜" (핵심)
│   ├── contracts/       # 플랫폼 간 계약(API 등) (코드 정본 참조)
│   ├── topics/          # 도메인 모델·일반 지식
│   ├── incidents/       # 장애·사건
│   ├── features/        # 기능 단위 목차 (자동생성)
│   └── knowledge/       # 플랫폼 무관 지식
└── .claude/skills/      # ingest · query · lint
```

세 가지 동작으로 돈다 — **ingest**(쌓기) · **query**(찾기) · **lint**(유지보수).

### 플랫폼 축
분류 축인 platform 값: `backend` · `frontend` · `infra` (예정: `android`(주로 React Native) · `ios`). `web`은 `frontend`의 동의어다. 어떤 코드 repo가 어떤 platform인지는 각 repo가 자기 `.brain`에 선언한다 — 이 brain은 특정 repo 이름에 의존하지 않는다.

## 어떻게 쌓이나

**1) 원본을 `raw/`에 떨군다** — 두 경로:

- **repo 작업물** → 각 코드 repo에 둔 *brain 연동 저장 skill*이, 작업·세션 결과를 frontmatter까지 맞춰 `raw/sessions/<platform>/`에 자동 저장한다. 사람은 작업만 하면 된다.
- **repo 밖에서 생긴 것**(회의·스펙) → 사람이 `raw/meetings/` · `raw/specs/`에 직접 올린다. 파일명은 `YYYY-MM-DD-<slug>.md`.

**2) `ingest`로 정리한다** — `ingest`를 돌리면 미처리 raw를 자동으로 읽어 플랫폼·타입별 wiki 노트로 분해·정리한다. 사전 질문 없이 진행하고 끝나면 요약을 보고한다.

## 어떻게 찾나

brain 세션 안에서는 `query`로, 코드 repo에서는 그 repo의 brain 연동 조회 skill로 묻는다. 답은 **항상 출처(`[[wiki/...]]`·`[[raw/...]]`)와 함께** 온다.
- **사람** — Obsidian에서 직접 보거나, 서술형 답 + 링크.
- **AI 에이전트** — 의사결정 중 맥락이 부족하면 query를 호출, 구조화된(JSON) 답 + 인용으로 받는다.

## 어떻게 유지되나

`lint`가 주기적으로 위키 건강을 점검한다 — 모순·낡음·고립·계약 drift·태그 산발 등. 봇이 점검해 수정을 *제안*하고, **사람이 승인한 것만** 반영한다. ingest가 자동으로 도는 만큼, lint가 위키를 시간이 지나도 안 썩게 하는 안전망이다.

## 연결

각 코드 repo 루트의 `.brain` 파일(gitignore)이 `platform`과 이 brain의 상대경로(`brain_path`)를 담는다. side-repo skill은 `git rev-parse --git-common-dir` 기준으로 메인 repo 루트의 `.brain` 하나를 찾으므로, worktree 어디서 실행해도 같은 연결을 본다.

## 핵심 규칙
- **`raw/`는 불변** — 작성 후 수정·삭제하지 않는다. 정정도 새 raw로.
- **`wiki/`는 LLM이 소유** — 사람이 직접 편집하지 않는다. 고치려면 raw에 정정 원본을 올리고 재-ingest. wiki는 raw로부터 언제든 재생성 가능한 산출물이다.
- **분류의 진실은 frontmatter에만** — 폴더는 navigation 보조.
- **커밋 컨벤션** — `raw:` / `ingest:` / `lint:` / `setup:` prefix. `wiki/` 변경의 정상 경로는 `ingest:`(또는 `lint:`) 커밋뿐이다.

## 더 보기
- 운영 규칙·schema 전체: [`CLAUDE.md`](./CLAUDE.md)
- 절차 상세: [`.claude/skills/`](./.claude/skills/) (ingest · query · lint)
