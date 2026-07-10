---
platform: backend
author: KimYeonWook511
created: 2026-06-03
origin:
  - { type: pr, repo: commerce-backend, ref: 203 }
---

## 한 일

- ADR-020 후속 트랙 (Stock #199 / Order #200 / Payment #202) series 의 **네 번째이자 마지막 트랙** `cross-aggregate-fk-cleanup`. PR #203 — cross-aggregate FK 5건 (`fk_stock_product_id`, `fk_stock_history_stock_id`, `fk_order_member_id`, `fk_order_item_product_id`, `fk_payment_order_id`) 을 단일 Flyway V4 migration 으로 일괄 제거. 코드 변경 0건, schema 만 변경.
- harness skill 로 Explore → Discuss → Step Design → Worktree → File Drafting → Execution 6단계 진행. phase 1개 (`0-drop-fk`) + step 3개 (drop-fk-migration / sync-root-docs / write-retrospective). execute.py 가 step1/2/3 + commit + finalize 자동 수행.
- pr-review-resolve skill 로 Gemini Code Assist review 3건 정정 (task 문서 adr / architecture / db-schema 의 `tbl_order` 잔류 index 표현). 흐름 중 사용자 통찰 "FK 와 index 는 다른 레이어니 ADR 근거로 박자" 를 받아 ADR 결정 2 의 근거 보강 commit 1건 추가 — 총 review 응답 4 commit (3 정정 + 1 보강).

## 결정한 것

정본: `commerce-backend/docs/tasks/cross-aggregate-fk-cleanup/adr.md`

본 세션의 *내 이해*:

- **결정 5개 중 실질 신규는 결정 2 (잔류 KEY index 유지) 1건**. 나머지 4개 (단일 V 파일 / same-aggregate FK 제외 / 운영 DB 별도 / 완료 task 불변) 는 선행 series 의 메타 원칙·정책 연장. payment-jpa-association-decouple 회고에서 관찰한 "마지막 PR 은 새 결정 항목보다 선행 원칙이 어디까지 커버하는가의 검증" 패턴이 본 트랙에서도 그대로 반복됨.
- **lag 표준 ADR 정립은 표본 1건이라 보류**. 본 series 진행 동안 코드 association 해제와 schema FK 제거 사이의 *과도기 lag* 가 발생한 게 처음. 향후 다른 cross-aggregate 정리 series 에서 lag 가 반복될 수 있는데, "허용 기간 표준" 을 본 PR ADR 에 박을지 검토 후 보류 — 표본 1건은 표준 정립에 부족. 회고에 사실 기록만, ADR 정립은 패턴 반복 확인 후로 미룸.
- **Issue 발행 안 함**. 선행 series 의 Issue #195 같은 sub-PR 묶음 issue 가 본 트랙에는 불필요. 본 트랙은 단일 PR 1건이고 PR 본문에서 series PR 들을 Refs 로 연결하면 추적성 확보. "issue 가 PR 의 그림자가 되는 케이스" — 보일러플레이트 줄이기 판단.

다시 본다면:

- harness execute.py worker 가 step1/2/3 을 일관되게 정확히 처리. 본 트랙처럼 결정이 단순한 series 마무리는 harness 자동화의 강점이 부각된다. 다음 단계는 어떤 유형의 트랙이 harness 자동화에 더 잘 맞는지 패턴 누적.
- review 응답 중 사용자 통찰 ("FK 와 index 는 다른 레이어") 이 ADR 결정 2 근거 보강 commit (`23f9f42`) 으로 이어진 흐름이 자연스러움. review skill 룰의 "1:1 매칭 commit 3건 + 별도 보강 1건" 분리가 사용자 대화에서 파생되는 추가 결정을 자연스럽게 흡수.

## 배운 것

### MySQL InnoDB 의 FK 와 index — 5건 중 2건/3건 비대칭

Gemini review 가 발견한 내 사실관계 오류로부터 시작. 처음 task 문서에 적은 잘못된 일반화: "FK 만들면 InnoDB 가 항상 자동 index 도 만든다, FK DROP 해도 그 자동 index 는 남는다" — 5개 FK 모두 이 패턴이라고 일반화.

**잘못 일반화한 이유의 인지 분해**: 결과 (FK DROP 후 index 보존 → 조회 성능 영향 0) 가 5건 모두 동일해서, 원인이 둘로 갈리는 걸 못 봤다.

실제 InnoDB 동작과 본 PR 5건 매핑:

- **경로 A (FK 가 자동 index 생성, FK DROP 시 KEY 잔류)**: 2건 — `tbl_stock_history.stock_id`, `tbl_order_item.product_id`. V1 SQL 에 명시적 `KEY fk_..._id (...)` 라인 있음.
- **경로 B (기존 index 의 prefix 재사용, 자동 생성 자체 없음)**: 3건 — `tbl_stock.product_id` (단일 컬럼 `uk_stock_product_id` 재사용), `tbl_payment.order_id` (단일 컬럼 `uk_payment_order_id` 재사용), `tbl_order.member_id` (복합 `uk_order_member_idempotency (member_id, idempotency_key)` 의 leftmost prefix 재사용). InnoDB 룰: FK 컬럼으로 시작하는 index 가 있으면 그걸 재사용, 자동 생성 X.

**FK 와 index 의 레이어 분리 원칙**: ADR 결정 2 근거에 박음 (커밋 `23f9f42`). 정본 인용으로 충분.

### Gemini review 응답 패턴

low priority 지적이라도 사실관계 검증은 정확. 본 케이스처럼 잘못 일반화한 표현을 정확히 짚어줌. 응답 패턴:
1. **사실관계 1차 확인** — `V1__init.sql` 의 tbl_order 정의 직접 검토. UNIQUE 가 `(member_id, idempotency_key)` 복합이고 별도 `KEY fk_order_member_id` 라인이 없음을 확인.
2. **본 트랙 톤으로 rephrase** — Gemini suggestion 그대로 채택 X. 본 트랙의 한국어 / backtick / 영문 식별자 조사 컨벤션 적용.
3. **같은 사실관계가 여러 문서에 있으면 각 파일별 별도 커밋** — review 와 commit 의 1:1 매칭 유지.

### review skill 의 "4번째 commit" 패턴

review 응답 흐름 중 사용자 대화에서 일반 원칙이 발견되면 그것을 ADR 근거로 박는 commit 이 review 응답의 일부로 자연스럽게 분리된다. 1:1 매칭 commit (3개, thread reply 와 매칭) + 추가 결정 commit (1개, thread reply 와 분리) — review skill 의 "여러 review 항목을 하나의 커밋으로 묶지 마라" 룰의 자연스러운 확장.

## 다음 단계 (지식 가치 있는 미해결)

- **lag 표준 ADR 정립 시점**: 본 series 에서 발생한 코드-schema lag 의 표준화는 표본 1건이라 보류. 향후 다른 cross-aggregate 정리 series (Member / Cart 통합 같은 가능성) 에서 lag 가 반복 등장하면 그때 ADR 정립. 후속 패턴 누적 필요.
- **결제 시점 가격 snapshot 미해결 (Issue #201)**: Order PR #200 에서 `addOrderItem` 의 `unitPrice` 가 컬럼 저장 안 되는 문제. 본 series 와 별개 후속 트랙. e-commerce 표준 (결제 시점 단가 snapshot 필수) 과 어긋남.
