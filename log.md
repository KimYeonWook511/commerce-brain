# commerce-brain 운영 기록

append-only. 각 항목 형식: `## [YYYY-MM-DD] {ingest|query|lint|setup} | <대상>`

## [2026-07-10] setup | commerce-brain 초기화

- 독립 저장소로 commerce-brain 생성. 구조: `raw/{sessions/<platform>,meetings,specs,images}`, `wiki/{decisions,contracts,topics,incidents,features,knowledge}`.
- 운영 파일: `CLAUDE.md`(schema·정책), `index.md`, `log.md`, `_tag-glossary.md`, `_skipped.md`.
- brain 자체 skill: `ingest`·`query`·`lint`. 코드 repo(side-repo)에 brain 연동 저장·조회 skill.
- 플랫폼 축: backend·frontend·infra (예정: android·ios).

## [2026-07-14] ingest | backend sessions 64건 (최초 대규모 배치)

- **대상**: `raw/sessions/backend/` 미처리 64건 전부 (wiki 인용 0건 확인 → 전량 미처리). meetings/specs 는 README placeholder 뿐이라 대상 없음.
- **방식**: 규모가 커(정상 ingest의 4배) Workflow 병렬 2단계 — ① 9개 클러스터 병렬 매핑(107개 노트 후보 매니페스트) → ② 진짜 중복 6건 병합·granularity 유지로 101개 확정 → ③ 20개 배치 병렬 작성. 시간적 supersede/evolution(payment 05-29 스냅샷 → 06-04 redesign, 주문 멱등 05-29 → pr-180 단순화)은 작성 단계에서 콜아웃으로 보존.
- **신규 101개**: decisions 77 · tradeoff 2 · topics 22(그중 knowledge 폴더 13). 도메인 개요 5개(order/payment/product/stock/auth-member) 를 허브 topic 으로, 나머지는 결정별 decision 으로 분해.
- **주요 병합**: `cross-aggregate-fk-to-id-참조` 3안→2, `재고차감 비관락` 2→1, `find-first/write-not-check` 2→1, AI리뷰 교훈 8→5, 동시성 테스트 2→1.
- **MOC 22개 생성**: payment(38)·order(21)·idempotency(20)·concurrency(18)·exception-handling(17)·compensation·reconciliation·jpa·unique-constraint·redis·optimistic-lock·pessimistic-lock·ddd·product·stock·cart·auth·escalation·reservation·saga·migration·naverpay.
- **glossary**: 일반 태그 등재(도메인/아키텍처/프로세스). 태그 266종으로 산발 → 동의어 통합·싱글턴 정리는 lint 대상으로 명시(노트 frontmatter 소급 수정 안 함).
- **skip 0건**. 64 raw 전부 wiki 에 인용됨(멱등성 확보).
- **검증**: 내부 `[[링크]]` 깨짐 0 · raw 인용 커버리지 64/64 · frontmatter 필수필드 OK.
- **다음 lint 후보**: (1) 태그 산발 통합, (2) payment 05-29 스냅샷 결정들의 supersede 정합 재점검, (3) 대형 MOC(payment 38·order 21) 하위 폴더링/분할 검토, (4) knowledge 13개 — `knowledge/` 하위 폴더링 승격 여부.

## [2026-08-17] ingest | backend sessions 29건 (부분취소 → 결제·환불 모델 재설계 배치)

- **대상**: `raw/sessions/backend/` 미처리 29건 전부 (wiki 인용 0건 확인). 기간 2026-07-15 ~ 2026-08-16, 세 국면 — ① 부분취소 1차 설계(issue #291, 07-15~07-23) ② 재설계 + 트랜잭션 경계 수정(#298/#299) + 문서 번호 정리(#301/#302, 08-08) ③ 네이버페이 실측(#304) → 결제·환불 모델 전면 재설계(PR #305, 08-10~08-16). meetings/specs는 README placeholder뿐이라 대상 없음.
- **신규 53개**: decisions 49(그중 tradeoff 3) · topics 2 · knowledge 12. 큰 raw 하나가 결정 14개를 담아(`2026-08-08-partial-cancel-design-decisions`) 결 단위로 6개 노트로 쪼갰고, PR #305 raw 4건(총 82KB)은 25개 결정 노트로 분해했다.
- **supersede 6건** — 뒤집힌 결정을 양쪽 보존하고 `superseded_by`로 연결:
  - `결제-부분환불-도입-현행한계-4가지와-테이블분리` → `결제사건-테이블분리-기각과-유일제약-문자열-단일컬럼-교체` (실제 필드 수를 세니 응집도 근거가 사라졌다)
  - `payment-reserve-예약테이블-분리-a안-b안` / `payment-reserve-ready-흐름-재설계-expiresat-재사용만료` → `예약테이블-폐지-결제행-활성슬롯-단일화와-사라지는-방어`
  - `대사-keep-waiting-backoff-next-reconcile-at` / `결제-escalation-종착통지-escalatedAt-직교필드` → `결과회수-상한-폐지와-백오프-표-통지-반복` (상한이 폐지되면서 고정 간격 근거가 사라졌다)
  - `이중환불-최종방어선-잔액대조-도입` → `잔액대조-옵션-미사용-공급자-권장의-전제-확인` (실측으로 도입했다가 비동기 구조와 안 맞아 기각)
- **기존 노트 진화 콜아웃 12건**: `paid-order-취소환불-단일tx-…`(단정한 단일 tx가 코드와 어긋나 있었음), `payment-status-사실만-…`(적용 범위를 너무 넓게 읽었던 정정), `unique-위반-예외번역-…`(제약 이름 기반으로 세분화), `payment-append-only-…`(승격 조건 둘이 충족됨), `payment-unknown-…`(어휘 셋→넷), `payment-부분취소-모델만-…`(open→decided, 맞은 것/틀린 것), `payment-이중결제-reserve따닥-…`, `payment-낙관적-락-…`(예약해둔 재판단이 닫힘), `payment-동시성-unique-vs-lock-…`, `미확정차단-대사스캔-…`, `orderitem-단가-snapshot-…`, `주문-멱등성-redis-setnx-…`(#171이 결제 쪽에서 답을 얻음), `payment-도메인-구조-개요`(2차 재설계 반영), knowledge 5건에 역참조.
- **MOC 32개**(신규 10): `refund`(30)·`partial-cancel`(18)·`pg-gateway`(15)·`mysql`(13)·`error-code`(11)·`transaction-boundary`(11)·`double-payment`(10)·`aggregate`(10)·`adapter`(10)·`unknown-status`(8). 기존 22개도 전량 재생성하며 `superseded`/`open`/`draft` status를 목록에 노출. **프로세스·메타 축(`convention` 21·`process` 12·`verification` 8)은 MOC로 만들지 않았다** — MOC는 기능 목차이고 그 축은 `knowledge/`가 담당한다(기존 ingest와 같은 기준).
- **glossary**: 개수 갱신(태그 279종) + 신규 canonical 등재(`refund`·`partial-cancel`·`aggregate`·`invariant`·`unknown-status`·`verification`·`operations` 등). 미통합 동의어에 `transaction`·`transaction-boundary`, `jpa`·`hibernate`, `spec-review`·`design-review` 추가.
- **skip 0건**. 29 raw 전부 wiki에 인용됨(멱등성 확보).
- **검증**: 내부 `[[링크]]` 깨짐 0 · raw 인용 커버리지 29/29 · index ↔ 실제 노트 1:1 대조 0 diff · frontmatter 필수필드 OK.
- **다음 lint 후보**:
  1. **`payment` MOC가 79개** — 이제 분할이 실질적으로 필요하다. `refund`·`partial-cancel`·`reconciliation`으로 이미 갈라지므로 `payment` MOC를 하위 축 인덱스로 재구성할 여지.
  2. **태그 산발 심화(279종)** — 위 미통합 목록 승인 후 일괄 정규화.
  3. **`decisions/` 118개 평탄** — 폴더 승격 검토(index에서는 이미 8개 주제군으로 갈라 쓰고 있다).
  4. **`knowledge/` 25개** — 하위 폴더링 승격(index에서 방법론/테스트/문서운영/구현패턴 4군으로 갈라 쓰는 중).
  5. **`payment-도메인-구조-개요`가 두 차례 재설계로 사실상 무효** — 현재 구조를 반영한 새 허브 topic 작성 여부.
  6. **`contracts/` 여전히 빈 폴더** — 멱등키 헤더 계약·취소 요청/응답 형태가 api-contract 후보로 보인다(코드가 정본이므로 companion).

## [2026-08-17] lint | wiki 전체 154 노트 · 33 MOC

- **발견 20건**: 모순 2 · 낡음 5 · 누락 cross-ref 8쌍 + sources 2 · 태그 산발(279종·싱글턴 134) 1 · 폴더 승격 후보 2. **통과**: 고립 0 · MOC drift 0 · frontmatter/status enum 0 · companion 본문 복사 위반 0 · 진짜 중복 결정 0 · promote 대상 0(전 노트 단일 platform).
- **수정 4묶음 (사용자 승인)**:
  1. **모순 2건** — `대사-확정-검증보상-대칭-재승인없음`을 `superseded`로 내리고 뒤집힘 콜아웃 추가(**제목의 "재승인은 없다"가 더는 참이 아니다**; 살아남은 것 셋을 명시). `결제-후처리-대상식별-status중심-재설계`의 **임계 상수 정본 선언**(1분/15분/6시간)이 무효가 된 것을 절 단위로 콜아웃.
  2. **낡음 콜아웃 4건** — `payment-낙관락-충돌처리-3계층`(14곳 인용) · `payment-order-도메인분리와-pg격리`(허브) · `주문-이중결제-앞단-진입차단`(그 조회에 결함이 있었음) · `결제승인완료-보상-완료우선`.
  3. **cross-ref 8쌍 + sources 2건** — 형제 결정 간 양방향 링크 보강, 본문에서 인용하던 `{repo,path}` 2건을 frontmatter `sources`로 승격(`docs/prd.md`, `docs/adr/20260614-pr248-application-role-suffix.md`).
  4. **태그 정규화** — 동의어 8쌍 통합(27개 노트 frontmatter 소급 수정): `transaction`→`transaction-boundary`(20) · `duplicate-payment`·`double-charge`→`double-payment`(16) · `schema-design`→`schema`(11) · `authentication`→`auth`(10) · `reconcile`→`reconciliation`(25) · `naming`→`naming-convention`(8) · `refactoring`→`refactor`(8) · `save-and-flush`·`save-flush`→`flush`(4). 미등록 태그 14종 glossary 등재. **279→269종, 싱글턴 134→131.**
- **MOC 33개** — 태그 병합 반영해 전량 재생성 + `schema`(11) 신규.
- **판단 근거로 남긴 것**:
  - **`security`(7)를 `auth`(10)에 합치지 않았다** — 결제 노트에서 `security`는 **접근 제어**를 뜻하고 인증 도메인과 다른 축이다. 합치면 두 관심사가 한 태그에 섞인다.
  - **태그 겹침 기반 dedup이 무력화됐다** — `payment`(79)·`refund`(30)가 지배해 4개+ 공유 쌍이 101건 걸리고 그중 진짜 중복은 0건이었다. **dedup 신호로 쓰려면 지배 태그를 제외하고 봐야 한다.**
  - **계약 drift는 판정 불가로 남겼다** — companion 7개 중 3개가 07-14 기준인데 정본이 `commerce-backend`에 있어 이 저장소에서 read-only 확인이 안 된다.
- **보류(사용자 판단)**: `decisions/` 118개 평탄 → 하위 폴더링. 파일 이동은 전체 wikilink 재검증이 필요해 별도 회차로 미룸.
- **다음 lint 후보**:
  1. **`payment` MOC 79개** — 분할이 실질적으로 필요. `refund`·`partial-cancel`·`reconciliation`으로 이미 갈라지므로 하위 축 인덱스로 재구성.
  2. **`payment-도메인-구조-개요`가 두 차례 재설계로 사실상 무효** — 콜아웃으로 "여기가 아니라 저기를 봐라"만 걸어둔 상태. 현재 구조를 반영한 새 허브 topic 작성 여부.
  3. **남은 동의어** — `infra-adapter`→`adapter`, `hibernate`를 `jpa` 하위로, `spec-review`·`design-review`, `batch`·`spring-batch`, `pg`→`pg-gateway`, `cancel`→`partial-cancel`.
  4. **싱글턴 131종** — 절반이 여전히 1회용. 3회 미만 태그를 일괄 정리할지 판단 필요.
  5. **`decisions/`·`knowledge/` 폴더 승격** (이번 보류분).
  6. **`contracts/` 빈 폴더** — 멱등키 헤더 계약·취소 요청/응답 형태가 api-contract 후보(코드가 정본이므로 companion).
