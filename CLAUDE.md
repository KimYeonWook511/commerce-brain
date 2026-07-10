# commerce-brain — 제품 LLM Wiki

LLM Wiki 패턴으로 운영되는 **commerce 제품 지식 베이스**. 한 제품을 여러 플랫폼(backend/frontend/infra, 예정: android/ios)이 함께 만드는 구조에서, 각 repo의 작업 세션·회의·스펙을 원본으로 삼아 LLM이 구조화된 단일 지식 지도를 만들고 유지한다. 담는 것은 **결정·맥락·트레이드오프·"왜"·미해결 쟁점** — 코드가 아니라 *코드를 그렇게 짠 이유*다. 사람은 원본(raw)을 던지고 질문하고 방향을 잡으며, LLM은 분해·정리·교차참조·일관성 유지를 전담한다.

## 절대 규칙
- `raw/`는 **불변**. 사람만 추가, LLM은 읽기만. 작성 후 수정·이동·삭제 금지.
- `wiki/`는 **100% LLM 소유**. 사람은 직접 편집하지 않는다. 고치려면 raw에 정정 원본을 올리고 재-ingest 한다. wiki는 raw로부터 언제든 재생성 가능한 산출물이다.
- 분류의 진실은 **frontmatter에만**. 폴더·MOC는 navigation 보조다.
- 모든 연결은 wikilink `[[페이지명]]`. 본문은 한국어.

---

## 디렉토리

```
commerce-brain/
├── CLAUDE.md         # 이 파일 — schema·정책
├── index.md          # wiki 카탈로그 (ingest 마다 갱신)
├── log.md            # 운영 기록, append-only
├── _tag-glossary.md  # 표준 태그·동의어
├── _skipped.md       # ingest 가 "가치없음"으로 영구 제외한 raw 목록 (봇 소유)
├── raw/                       # 입력 동선 기준 — 사람이 올리기 쉽게
│   ├── meetings/              # 회의·논의 기록 (결정의 주요 출처)
│   ├── sessions/<platform>/   # 플랫폼별 작업/대화 덤프
│   ├── specs/                 # 요구사항·기획·디자인 핸드오프
│   └── images/
└── wiki/                      # 타입 기준 — 봇이 분해해 작성
    ├── decisions/    # 결정의 "왜" (ADR). 핵심 본체
    ├── contracts/    # 플랫폼 간 계약(API 등) (코드가 정본, companion)
    ├── topics/       # 도메인 모델·일반화된 지식
    ├── incidents/    # 장애·사건 (영향·타임라인·원인·해결·교훈)
    ├── features/     # 기능 단위 목차(MOC) — 봇 자동생성
    └── knowledge/    # 플랫폼 무관 일반 지식
```

raw와 wiki는 폴더를 나누는 목적이 다르다. raw는 *어디 올릴지* 헷갈리지 않게 입력 동선으로 가르고, wiki는 *봇이 분해해 쓰는 출력*이라 타입으로 가른다. 그래서 `sessions/backend/`에 올린 덤프라도 ingest가 그 안의 frontend 얘기를 떼어 `platform: frontend` 노트로 쓸 수 있다 — 실제 플랫폼 귀속은 폴더가 아니라 내용을 읽고 frontmatter에 박는다.

### 플랫폼 축
platform 값: `backend`·`frontend`·`infra` (예정: `android`(주로 React Native)·`ios`). `web`은 `frontend`의 동의어로 glossary에서 정규화한다. 어떤 코드 repo가 어떤 platform인지는 각 repo가 자기 `.brain`에 선언한다(연결·platform 추출 세부는 side-repo skill의 책임). 이 brain 문서는 특정 repo 이름에 의존하지 않는다.

---

## 폴더 운영 원칙 — "검증된 어휘는 미리, 미지는 평탄, 자라면 승격"

- **검증된 어휘 → 미리 폴더**: 위 wiki 타입 폴더는 채울 게 확실한 출발 세트. 빈 폴더 압박 없이 시작.
- **미지 카테고리 → 평탄 시작**: `knowledge/`는 평탄하게 시작해 한 도메인이 5개+ 쌓이면 사후 폴더링.
- **타입에 안 맞는 노트 → 루트 평탄**: 비슷한 성격이 5개+ 쌓이면 lint가 새 폴더(예: `retrospectives/`, `glossary/`) 승격을 제안. 폴더는 고정이 아니라 자란다.
- 분류도 못 하고 정제도 안 된 메모는 wiki로 올리지 말고 raw에 남긴다.

---

## raw 파일 컨벤션

- 파일명: `YYYY-MM-DD-<slug>.md`. **slug는 구체적으로** (`login` ✗ → `login-token-rotation` ✓) — 그러면 충돌이 사실상 없다. 드물게 같은 날 같은 slug가 겹치면 git 머지에서 add/add 충돌로 잡히므로, 그때 한쪽에 `-<짧은해시>` 붙여 재커밋.
- frontmatter (raw도 작성 후 불변):

```yaml
# sessions/<platform>/
platform: backend
author: KimYeonWook511
created: 2026-07-10
origin:                   # 있을 때만 — PR/이슈/회의 등 출처. 없으면 통째 생략
  - { type: pr, repo: <code-repo>, ref: <번호> }
```
```yaml
# meetings/
participants: [KimYeonWook511]
created: 2026-07-10
```

본문은 거친 덤프 OK — 결정 / 검토한 대안 / 트레이드오프 / 막힌 점. `origin`에 PR이 있으면 "코드에 반영된 결정"이라는 강한 신호이고, 없으면 아직 논의 단계일 가능성이 높다 (ingest가 status 판단에 참고).

---

## wiki frontmatter

```yaml
type: decision | tradeoff | topic | incident | api-contract | moc
status: <type별 enum>
platform: backend         # 필수. 여러 플랫폼이면 배열 [backend, frontend, android]
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [login, auth]       # 가로지르는 관심사·기능은 전부 여기
created: 2026-07-10
updated: 2026-07-10
superseded_by: null       # 대체되면 새 결정 링크
sources:                  # 모든 주장의 근거 — 필수
  - "[[raw/sessions/backend/2026-07-10-login-token-rotation]]"
  - { repo: <code-repo>, path: "<정본 파일>#<앵커>" }   # 코드가 정본일 때
```

> **분류 축은 `platform` 하나만 필수.** security·db·devops 같은 "성격"은 별도 필드 없이 전부 `tags`로 단다.

| type | status enum | 필수 |
|---|---|---|
| decision | `proposed \| accepted \| superseded \| deprecated` | type, platform, created, sources |
| tradeoff | `open \| decided` | type, platform, created, sources |
| topic | `draft \| stable \| outdated` | type, created, updated, sources |
| incident | `open \| resolved` | type, platform, created, sources |
| api-contract | `current \| deprecated` | type, platform, created, sources(`{repo,path}`) |
| moc | (생성물) | `features/<feature>.md`, 봇 자동생성 |

---

## 본문 권장 구조

### decision — 두 모드
**기본은 모드 B (wiki가 정본).** 코드에 사는 계약·스펙만 모드 A(companion).

- **모드 B (정본, 대부분)**: ① 컨텍스트·문제 → ② 검토한 대안(각각의 트레이드오프) → ③ 결정 + 선택 이유 → ④ 트레이드오프·리스크·추후 문제 → ⑤ 미해결·후속.
- **모드 A (companion, `sources`에 `{repo,path}` 있을 때)**: ① 정본 위치 인용 → ② 내가 이해한 문제 → ③ 검토한 대안(요약, *본문 복사 금지*) → ④ 지금 다시 본다면.

두 모드 모두 끝에 `## 근거` + `[[raw/...]]` 인용.

> 코드 repo가 `docs/`·스펙 등에 정본을 두면(예: ADR·컨벤션·architecture), 그 repo의 결정은 대체로 **모드 A** — `{repo, path}`로 인용하고 wiki에는 "내 이해·트레이드오프·다시 본다면"만. ADR·OpenAPI 본문 통째 복사 금지.

### 그 외
- **api-contract** — 본질적으로 코드가 정본이라 항상 companion. 계약의 링크·요약·"왜 이 형태인가"만, OpenAPI/스펙 본문 복사 금지.
- **topic** — 한 줄 정의 / 핵심 / 근거 / 관련 링크 / 열린 질문.
- **incident** — 영향 범위·타임라인 / 추적 / 원인 / 해결 / 배운 것.
- **회의·논의(raw) 처리** — 회의는 wiki 타입이 아니다. decision·incident·topic으로 *분해*하고 원본은 sources로 인용. 분해가 무의미한 순수 논의만 `topic`으로 남긴다.

---

## 정본(canonical)

정본은 *repo*가 아니라 *성격*으로 정한다.
- **이유·판단·트레이드오프(왜)** → commerce-brain이 정본(모드 B). 다른 집이 없으니까.
- **코드에 사는 계약·스펙·현재 동작** → repo가 정본(모드 A). `{repo,path}` 인용, 본문 복사 금지. `api-contract`는 항상 이쪽.

---

## MOC (`features/<feature>.md`)

기능이 여러 플랫폼에 흩어질 때 한곳에서 가리키는 목차. **`tags` query의 생성된 뷰**다 — 진실은 노트의 `tags`에만 있고, MOC는 그걸 비추는 거울이라 아무 진실도 소유하지 않는다. 한 태그가 5개+ 모이면 봇이 생성하고, 새 결정마다 링크 목록을 덮어쓰기로 재생성한다. 통째로 지웠다 재생성해도 손실이 없다.

---

## 운영 (상세 절차는 `.claude/skills/{ingest,query,lint}/SKILL.md`)

- **Ingest** (`raw/`→`wiki/`): 미처리 raw 식별 → 결 단위 분해(한 덤프가 여러 플랫폼이면 여러 노트로) + 중복/뒤집힘 확인(dedup·supersede) → 타입별 노트 생성·갱신(보통 5~15개) → frontmatter + sources 인용 → glossary/index/log 동기 → MOC 재생성.
  현재 **모드 A**: 사람이 실행하되 **사전 질문 없이 완전 자동**으로 진행하고 끝난 뒤 요약 보고. 사람 개입은 맥락이 꼭 필요한 극소수 예외뿐. review·스테이징·크론 없음 (자동화 모드 B는 규모가 커지면).
- **Query**: 키워드·platform·type 추출 → index + glossary로 후보 → frontmatter + wikilink 그래프 → **항상 인용과 함께** 답변. 검색 로직은 하나, **출력만 소비자에 따라 분기** — 사람은 서술형 + `[[링크]]`, LLM 에이전트는 구조화(JSON: `answer` + `citations`(status·platform·sources)). `superseded`/`deprecated`는 하향하되 status 노출. 규모가 커지면 index 통독 대신 검색 엔진(BM25/벡터). **query는 읽기 전용** — 새 통찰은 wiki에 직접 쓰지 않고 raw로 떨궈 ingest가 올린다.
- **Lint** (ingest가 자동인 만큼 **사람의 주기적 검수 자리**): 모순 / 중복(dedup 보완) / 낡음 / 계약 drift / 고립 / 없는 개념·MOC 정합성 / 누락 링크 / 태그 산발 / frontmatter·추적성(author·decided_by) / companion 복사 위반 / 폴더 승격 / promote 후보 점검. 지적과 함께 수정 제안, 승인분만 수정.

### 멱등성
"이미 ingest 됐나"는 wiki에 그 raw의 `[[...]]` 인용이 있는지로 판단(`grep`). 0건이면 후보, 단 `_skipped.md` 등재분은 제외. 사용자가 파일을 명시하면 강제 재처리.

### 모순 처리
새 raw가 기존 주장과 충돌하면 임의 삭제 금지. `> [!warning] 모순` 콜아웃으로 양쪽을 보존하고, 뒤집힌 쪽은 `superseded` + `superseded_by` 링크.

---

## 톤
- 기술 영역(`decisions/`, `contracts/`, `topics/`): 정확함·구체성·기술어 우선.
- 회의·논의: 의도·합의/미합의를 **왜곡 없이** 보존. 합의 안 된 것을 합의된 것처럼 쓰지 않는다.

## tag glossary / log 형식
- `_tag-glossary.md`: 빈 상태로 시작해 자란다. canonical = 소문자·하이픈·단수형, 동의어 정규화(`React`→`react`). `platform` 태그군은 별도로 둔다.
- `log.md` 항목 형식: `## [YYYY-MM-DD] {ingest|query|lint|setup} | <대상>`

## 커밋 컨벤션 (정본)
커밋 메시지는 운영 단위를 prefix로 단다. 이는 스타일이 아니라 **기능**이다 — "누가(사람/봇) 무엇을 했나"를 구분하는 신호이고, hook·추적·`log.md` 대조의 기반이 된다.

- `raw:` — 사람/side-repo skill이 raw 덤프 추가. 예: `raw: backend login-token-rotation`
- `ingest:` — 봇이 ingest로 wiki 갱신. 예: `ingest: 2026-07-10 raw 3건 → 결정 5개`. **`wiki/`를 변경하는 정상 경로는 `ingest:` 커밋뿐이다.** (`_skipped.md` 갱신도 이 커밋에 포함.)
- `lint:` — lint 수정 반영
- `setup:` — 구조·스킬·CLAUDE.md·설정 변경

규칙:
- 커밋 메시지에 `Co-Authored-By` 트레일러를 넣지 않는다 (사람/봇 커밋 모두).
- `wiki/` 변경이 `ingest:`(또는 `lint:`)가 아닌 커밋에 섞이면 사람의 실수 가능성 — hook/리뷰에서 경고 대상.
- prefix 어휘는 `log.md` 형식과 일치시킨다(같은 사건이 git과 log 양쪽에서 같은 단어로 보이게).
- **side-repo skill은 이 컨벤션을 *문서 참조*가 아니라 자기 절차의 동작으로 따른다** — raw 저장 후 반드시 `raw: <platform> <slug>` 형식으로 커밋. (정본은 이 파일, 강제는 각 skill 동작.)
