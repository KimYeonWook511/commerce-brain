---
platform: backend
author: KimYeonWook511
created: 2026-06-03
origin:
  - { type: pr, repo: commerce-backend, ref: 203 }
---

# cross-aggregate FK 5건 일괄 제거 — series 마무리 트랙의 결정 검증과 InnoDB FK/index 비대칭

"신규 도메인은 다른 aggregate 를 객체가 아니라 ID 로만 참조한다"는 원칙을 기존 도메인에 소급 적용해온 4-트랙 series 의 **마지막 트랙**(PR #203). 앞선 세 트랙(Stock #199 / Order #200 / Payment #202)이 JPA 매핑 차원에서 cross-aggregate association 을 모두 해제한 상태였고, DB schema 에만 남아 있던 cross-aggregate FK 5건 — `fk_stock_product_id`, `fk_stock_history_stock_id`, `fk_order_member_id`, `fk_order_item_product_id`, `fk_payment_order_id` — 을 단일 Flyway 마이그레이션(`V4__drop_cross_aggregate_fk_constraints.sql`)으로 일괄 제거했다. 자바 코드 변경 0건, schema 만 변경한 좁은 PR 이다.

작업은 탐색→논의→스텝 설계→워크트리→파일 작성→실행의 6단계 자동화 harness 로 진행했다(phase 1개 `0-drop-fk` + step 3개: 마이그레이션 작성 / 루트 문서 동기 / 회고 작성). 결정이 단순한 series 마무리라 실행기가 세 스텝 + 커밋 + finalize 를 일관되게 자동 처리했다 — 어떤 유형의 트랙이 harness 자동화에 잘 맞는지에 대한 표본으로 남긴다.

## 결정한 것

### 실질 신규 결정은 "FK 제거 후 잔류 KEY index·UNIQUE 를 유지한다" 1건뿐

이 트랙에는 다섯 개의 결정을 기록했지만, 선행 series 의 메타 원칙·정책의 연장이 아닌 **실질 신규 결정은 단 하나** — FK 를 뽑아낸 뒤 같은 테이블에 남는 KEY index 와 UNIQUE 제약을 함께 DROP 하지 않고 유지한다는 것이다.

- **UNIQUE 유지:** `uk_stock_product_id`, `uk_payment_order_id` 는 FK 와 별개 제약으로, "Stock 1:1 Product / Payment 1:1 Order" 라는 도메인 invariant 를 DB 차원에서 보증한다. FK 를 뽑아도 이 불변식은 약해지지 않으므로 유지한다. JPA `@Table(uniqueConstraints = ...)` 매핑도 그대로 둬 코드와 schema 가 일치한다.
- **잔류 KEY index 유지:** FK 컬럼은 application 의 `WHERE`/`JOIN` 조건으로 계속 쓰이므로(예: Order 의 `WHERE member_id`, Payment 의 `WHERE order_id`, StockHistory 의 `WHERE stock_id`) index 가 없으면 full table scan 으로 떨어진다. FK 제약(참조 무결성)과 index(조회 성능 구조)는 서로 다른 레이어라, 무결성 제약을 없애도 index 는 살려둔다.
- **검토한 대안 — FK + 잔류 KEY index 를 함께 DROP:** schema 정리 완결성 관점에선 매력적이나, ALTER 횟수가 늘어 운영 lock 단위가 커지고 조회 성능만 잃는다. 기각. 결과적으로 V4 SQL 은 `DROP FOREIGN KEY` 5건만 담고 `DROP INDEX`/`DROP KEY` 는 0건이다. ALTER 를 최소화하면 향후 운영 배포 시 lock 단위가 작게 유지된다는 부수 효과도 있다.

나머지 네 결정 — 단일 V 파일에 5건을 묶기 / same-aggregate FK 는 범위 밖 / 운영 DB 배포 절차는 별도 결정 / 완료된 task 폴더 문서 불변 — 은 모두 선행 트랙에서 이미 선 정책을 이 트랙에 적용한 것이다. 직전 Payment 트랙 회고에서 관찰했던 **"series 의 마지막 PR 은 새 결정을 만들기보다 선행 원칙이 어디까지 커버하는지를 검증하는 자리"** 라는 패턴이 여기서도 그대로 반복됐다.

- **단일 마이그레이션 vs 도메인별 분리:** 도메인별 V 파일 3개(Stock 2 / Order 2 / Payment 1)로 쪼개면 운영 배포 ALTER 단위를 도메인 경계로 나눌 수 있다. 그러나 이 트랙의 정책 단위가 "series 마무리 한 건"이고, 5건이 모두 초기 스키마 파일 하나에서 함께 정의됐던 origin 도 단일 파일과 일관된다. 운영 배포 절차 자체를 별도 결정으로 뺐으므로 여기서 분리할 실익이 없어 단일 V4 로 묶었다.
- **same-aggregate FK 제외:** `fk_order_item_order_id`(Order↔OrderItem)는 알면서도 건드리지 않았다. cross-aggregate 참조만 ID 로 바꾸는 원칙의 적용 범위가 명시적으로 same-aggregate root-child 관계를 제외하고 객체 참조를 허용하기 때문이다. Order 트랙에서 Order↔OrderItem 을 lifecycle 결합이 강한 same-aggregate 로 판정하고 `@OneToMany`/`@ManyToOne` 객체 참조를 유지했으므로, schema 도 코드 결정과 일관하게 이 FK 를 남긴다.
- **운영 DB 배포 절차는 범위 밖:** 이 PR 은 Flyway 파일 추가 + local/test(Testcontainers integrationTest 로 Hibernate `validate` 통과) 확인까지만 한다. 운영 배포 시점·무중단·롤백 정책은 운영 DB 현황(트래픽/정비 윈도우/replication topology)에 의존하는 다른 축의 결정이라 분리했다 — 한 PR 에 묶으면 "schema 정합성 회복"이라는 단일 메시지가 흐려진다. 참고로 `ALTER TABLE ... DROP FOREIGN KEY` 는 데이터 복사 없이 짧은 metadata lock 만 잡는 DDL 이지만, 동시 트랜잭션 경쟁·정비 윈도우 점검은 운영 배포 단계의 몫이다.
- **완료된 task 폴더 불변:** series 마무리다 보니 선행 sub-PR 문서에 "FK 제거 완료" 후속 노트를 달고 싶은 유혹이 있었으나, "완료된 task 문서는 그 시점 결정의 동결 기록"이라는 프로젝트 컨벤션대로 손대지 않았다. series 종료 사실은 이 트랙 회고와, 최신 상태의 단일 진실 원천인 루트 ADR 문서의 후속 노트로만 표현했다.

### lag 표준 ADR 정립은 표본 1건이라 보류

이 series 진행 중 **과도기 lag** 가 처음 생겼다 — 코드는 cross-aggregate association 을 이미 해제했는데 DB FK 는 아직 남아 있는 불일치 상태. Stock 트랙 머지 이후 Order/Payment 를 거치며 코드 차원 association 이 0건이 됐지만 DB FK 5건은 이 마무리 트랙까지 남아 있었고, 약 하루의 lag 를 의도적으로 단기간 연속 머지로 흡수했다. 이 lag 는 Stock 트랙이 "schema 변경 0건 — 모든 sub-PR 완료 후 일괄 제거"를 메타 원칙으로 못 박은 결과다(부분 제거하면 일부 도메인만 FK 없는 불균형 schema 가 되고, V 파일을 sub-PR 마다 흩으면 schema 변경의 정책 단위가 분산되므로).

향후 다른 cross-aggregate 정리 series 에서 이 lag 가 반복될 수 있어 "허용 기간 표준"을 이 PR ADR 에 박을지 검토했으나 **보류**했다. 표본 1건으로 일반화하는 건 이르다. 회고에는 lag 발생 사실만 기록하고, 표준 정립은 다른 series 에서 패턴이 반복 확인된 뒤로 미뤘다. lag 기간 동안 기술 위험은 낮았다 — Hibernate `validate` 는 FK 존재 여부가 아니라 컬럼 이름·타입·nullable 만 검사하고, FK 가 남아 있는 동안 부모 row 삭제 시도는 DB 가 거부해 공통 안전망 500 으로 위임되는 의도된 동작이었다.

### 별도 이슈를 발행하지 않았다

선행 series 는 sub-PR 묶음을 관리하려고 상위 이슈(#195)를 뒀지만, 이 트랙에는 그런 이슈가 불필요했다. 단일 PR 1건이고 PR 본문에서 series 의 관련 PR 들을 참조로 연결하면 추적성이 확보된다. "이슈가 PR 의 그림자밖에 안 되는 케이스"라 보일러플레이트를 줄이는 판단으로 발행하지 않았다.

## 배운 것

### MySQL InnoDB 의 FK-자동 index 는 5건 중 2건만 — 3건은 기존 index 재사용

이 세션 최대의 기술 교훈은 Gemini 코드 리뷰가 짚은 내 사실관계 오류에서 나왔다. 처음 task 문서에 적은 잘못된 일반화: **"FK 를 만들면 InnoDB 가 항상 자동 index 도 만들고, FK 를 DROP 해도 그 자동 index 는 남는다 — 5건 모두 이 패턴"**. 왜 틀렸는지 인지적으로 분해하면, 결과("FK DROP 후 index 보존 → 조회 성능 영향 0")가 5건 모두 동일했던 탓에 그 결과에 이르는 **원인이 둘로 갈린다는 걸 못 봤다.** 초기 스키마(`V1__init.sql`)를 직접 대조해 실제 매핑을 확인했다:

- **경로 A — FK 가 자동 index 를 만들고, FK DROP 후 그 KEY 가 잔류 (2건):** `tbl_stock_history.stock_id`, `tbl_order_item.product_id`. 초기 스키마에 명시적 `KEY fk_..._id (...)` 라인이 실제로 있다.
- **경로 B — FK 컬럼으로 시작하는 기존 index 가 있어 InnoDB 가 그걸 재사용, 자동 생성 자체가 없음 (3건):** `tbl_stock.product_id`(단일 컬럼 `uk_stock_product_id` 재사용), `tbl_payment.order_id`(단일 컬럼 `uk_payment_order_id` 재사용), `tbl_order.member_id`(복합 `uk_order_member_idempotency (member_id, idempotency_key)` 의 leftmost prefix 재사용). 이 3건은 초기 스키마에 별도 KEY 라인이 없다 — FK 를 뽑아도 잔류할 "자동 index"가 애초에 없다.

핵심 교훈: **InnoDB 에서 FK 는 "인덱스가 없을 때만" 자동 생성하고, 컬럼을 leftmost prefix 로 갖는 index 가 이미 있으면 그걸 재사용한다.** 그래서 FK 와 index 는 서로 다른 레이어이며(무결성 제약 vs 조회 구조), 이 원칙을 잔류 KEY index 유지 결정의 근거로 별도 커밋으로 못 박았다.

### low-priority 리뷰라도 사실관계 검증은 정확 — Gemini 리뷰 대응 패턴

이번 오류를 짚은 지적은 우선순위상 낮았지만 사실관계는 정확했다. 잘못 일반화한 표현을 정확히 집어냈다. 대응 패턴으로 정리하면:

1. **사실관계 1차 확인:** 초기 스키마의 `tbl_order` 정의를 직접 검토해, UNIQUE 가 `(member_id, idempotency_key)` 복합이고 별도 `KEY fk_order_member_id` 라인이 없음을 확인했다.
2. **트랙 톤으로 rephrase:** 리뷰 제안을 그대로 붙여넣지 않고, 이 트랙의 한국어/backtick/영문 식별자 조사 컨벤션에 맞춰 다시 썼다.
3. **같은 사실관계가 여러 문서에 있으면 파일별 별도 커밋:** ADR / architecture / db-schema 세 문서에 걸친 `tbl_order` 잔류 index 표현을 각각 별도 커밋으로 정정해, 리뷰 스레드와 커밋의 1:1 매칭을 유지했다.

### 리뷰 대응의 "4번째 커밋" 패턴 — 대화에서 파생된 결정의 분리

리뷰 대응 흐름 중 사용자 대화에서 일반 원칙("FK 와 index 는 다른 레이어니 그걸 ADR 근거로 박자")이 떠오르면, 그것을 ADR 근거로 못 박는 커밋이 리뷰 응답의 일부로 자연스럽게 **분리**된다. 이번엔 스레드 답변과 1:1 매칭되는 정정 커밋 3건(위 세 문서) + 스레드와 분리된 근거 보강 커밋 1건 = 총 4건이었다. "여러 리뷰 항목을 한 커밋에 묶지 마라"는 리뷰 절차 룰이, 대화에서 파생된 추가 결정까지 자연스럽게 흡수하며 확장된 사례다.

## 미해결·열린 질문

- **lag 허용 기간 표준의 ADR 정립 시점:** 이번 코드-schema lag 는 표본 1건이라 표준화를 보류했다. 다른 cross-aggregate 정리 series(예: Member / Cart 통합 가능성)에서 lag 가 반복 등장하면 그때 표본 2~3건을 모아 ADR 로 일반화하는 게 적절한 타이밍이다. 지금은 lag 허용 근거가 Stock 트랙 ADR 에만 있어, 다음에 같은 상황을 만나면 그 결정을 다시 찾아야 하는 탐색 비용이 남는다.
- **운영 DB 배포 절차의 미결:** 이 PR 머지 후에도 운영 DB 에는 여전히 FK 5건이 남아 있다. 별도 결정으로 뺀 것 자체는 옳으나, 그 "별도 결정"이 담당자·시점이 붙은 실제 action item 이 되어야 기술 부채로 굳지 않는다.
- **결제 시점 가격 snapshot (Issue #201):** Order 트랙(#200)에서 파악된, `addOrderItem` 의 `unitPrice` 가 컬럼으로 저장되지 않는 문제. 이 series 와는 별개의 후속 트랙이며, "결제 시점 단가 snapshot 필수"라는 e-commerce 표준과 어긋나 있다.
