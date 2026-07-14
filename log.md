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
