---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [fk, flyway, innodb, index, unique-constraint, schema, migration]
created: 2026-06-03
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-03-pr-203-cross-aggregate-fk-cleanup]]"
  - "[[raw/sessions/backend/2026-06-03-pr-202-payment-jpa-association-decouple]]"
---

# cross-aggregate FK 5건 일괄 DROP — 잔류 index·UNIQUE 유지와 InnoDB FK-index 비대칭

cross-aggregate 참조를 ID로 되돌리는 4-트랙 series([[cross-aggregate-fk-to-id-마이그레이션-동기-전략]], [[schema-무변경-decouple-series-메타원칙과-scope-규율]])의 **마지막 트랙**(PR #203). 앞선 세 트랙(Stock #199 / Order #200 / Payment #202)이 JPA 매핑 차원에서 cross-aggregate association을 모두 해제한 상태였고, DB schema에만 남아 있던 cross-aggregate FK 5건을 단일 Flyway로 일괄 제거했다.

## 컨텍스트 — 코드 association 해제 완료 후 남은 FK 5건, 단일 Flyway V4

series의 "schema 변경 0건" 메타 원칙에 따라, 앞선 세 sub-PR은 코드 association만 끊고 DB FK는 그대로 뒀다. 그 결과 DB schema에 남은 cross-aggregate FK 5건 — `fk_stock_product_id`, `fk_stock_history_stock_id`, `fk_order_member_id`, `fk_order_item_product_id`, `fk_payment_order_id` — 을 단일 Flyway 마이그레이션(`V4__drop_cross_aggregate_fk_constraints.sql`)으로 일괄 제거했다. **자바 코드 변경 0건**, schema만 변경한 좁은 PR이다([[flyway-도입-ddl-auto-validate-전환]]).

직전 Payment 트랙 회고에서 관찰했던 **"series의 마지막 PR은 새 결정을 만들기보다 선행 원칙이 어디까지 커버하는지를 검증하는 자리"**라는 패턴이 여기서도 반복됐다. 그래서 이 트랙에 기록한 다섯 결정 중 네 개는 선행 정책의 적용이고, 실질 신규 결정은 하나뿐이다.

## 결정 — FK DROP 후 잔류 KEY index·UNIQUE 유지 (실질 신규 결정)

FK를 뽑아낸 뒤 같은 테이블에 남는 KEY index와 UNIQUE 제약을 함께 DROP하지 않고 **유지**한다.

- **UNIQUE 유지:** `uk_stock_product_id`, `uk_payment_order_id`는 FK와 별개 제약으로 "Stock 1:1 Product / Payment 1:1 Order"라는 도메인 invariant를 DB 차원에서 보증한다. FK를 뽑아도 이 불변식은 약해지지 않으므로 유지한다. JPA `@Table(uniqueConstraints = ...)` 매핑도 그대로 둬 코드와 schema가 일치한다([[multi-column-unique-length-명시-컨벤션]]).
- **잔류 KEY index 유지:** FK 컬럼은 application의 `WHERE`/`JOIN` 조건으로 계속 쓰인다(Order의 `WHERE member_id`, Payment의 `WHERE order_id`, StockHistory의 `WHERE stock_id`). index가 없으면 full table scan으로 떨어진다. FK 제약(참조 무결성)과 index(조회 성능 구조)는 서로 다른 레이어라, 무결성 제약을 없애도 index는 살려둔다.

**검토한 대안 — FK + 잔류 KEY index 함께 DROP (기각):** schema 정리 완결성 관점에선 매력적이나, ALTER 횟수가 늘어 운영 lock 단위가 커지고 조회 성능만 잃는다. 결과적으로 V4 SQL은 `DROP FOREIGN KEY` 5건만 담고 `DROP INDEX`/`DROP KEY`는 0건이다. ALTER를 최소화하면 향후 운영 배포 시 lock 단위가 작게 유지된다는 부수 효과도 있다.

## InnoDB FK-자동index 비대칭 — 경로 A vs 경로 B

이 세션 최대의 기술 교훈은 Gemini 코드 리뷰가 짚은 사실관계 오류에서 나왔다. 처음 적은 잘못된 일반화: **"FK를 만들면 InnoDB가 항상 자동 index도 만들고, FK를 DROP해도 그 자동 index는 남는다 — 5건 모두 이 패턴."** 왜 틀렸는지 분해하면, 결과("FK DROP 후 index 보존 → 조회 성능 영향 0")가 5건 모두 동일했던 탓에 그 결과에 이르는 **원인이 둘로 갈린다는 걸 못 봤다.** 초기 스키마(`V1__init.sql`)를 직접 대조해 실제 매핑을 확인했다.

- **경로 A — FK가 자동 index를 만들고, FK DROP 후 그 KEY가 잔류 (2건):** `tbl_stock_history.stock_id`, `tbl_order_item.product_id`. 초기 스키마에 명시적 `KEY fk_..._id (...)` 라인이 실제로 있다.
- **경로 B — FK 컬럼으로 시작하는 기존 index가 있어 InnoDB가 그걸 재사용, 자동 생성 자체가 없음 (3건):** `tbl_stock.product_id`(단일 컬럼 `uk_stock_product_id` 재사용), `tbl_payment.order_id`(단일 컬럼 `uk_payment_order_id` 재사용), `tbl_order.member_id`(복합 `uk_order_member_idempotency (member_id, idempotency_key)`의 leftmost prefix 재사용). 이 3건은 초기 스키마에 별도 KEY 라인이 없다 — FK를 뽑아도 잔류할 "자동 index"가 애초에 없다.

핵심 교훈: **InnoDB에서 FK는 "인덱스가 없을 때만" 자동 생성하고, 컬럼을 leftmost prefix로 갖는 index가 이미 있으면 그걸 재사용한다.** 이 원칙을 잔류 KEY index 유지 결정의 근거로 별도 커밋으로 못 박았다. 리뷰 사실관계 대응 패턴 자체는 [[코드베이스-패턴-우선-설계판단-미사용api-방어가드-자동리뷰]]와 인접하나 초점이 다르다(사실 정정 vs 방어가드 판단).

## 선행 정책의 적용 — 신규 결정이 아닌 네 건

- **단일 V파일 vs 도메인별 분리:** 도메인별 V 파일 3개로 쪼개면 운영 배포 ALTER 단위를 도메인 경계로 나눌 수 있으나, 이 트랙의 정책 단위가 "series 마무리 한 건"이고 5건이 모두 초기 스키마 파일 하나에서 함께 정의됐던 origin도 단일 파일과 일관된다. 운영 배포 절차 자체를 별도 결정으로 뺐으므로 여기서 분리할 실익이 없어 단일 V4로 묶었다.
- **same-aggregate FK 제외:** `fk_order_item_order_id`(Order↔OrderItem)는 알면서도 건드리지 않았다. Order 트랙이 Order↔OrderItem을 lifecycle 결합이 강한 same-aggregate로 판정하고 객체 참조를 유지했으므로([[cross-aggregate-fk-to-id-참조-컨벤션]]), schema도 코드 결정과 일관하게 이 FK를 남긴다.
- **운영 DB 배포 절차는 범위 밖:** 이 PR은 Flyway 파일 추가 + local/test(Testcontainers integrationTest로 Hibernate `validate` 통과) 확인까지만 한다. 운영 배포 시점·무중단·롤백 정책은 운영 DB 현황(트래픽/정비 윈도우/replication topology)에 의존하는 다른 축이라 분리했다. `ALTER TABLE ... DROP FOREIGN KEY`는 데이터 복사 없이 짧은 metadata lock만 잡는 DDL이지만, 동시 트랜잭션 경쟁·정비 윈도우 점검은 운영 배포 단계의 몫이다.
- **완료된 task 폴더 불변:** 선행 sub-PR 문서에 "FK 제거 완료" 후속 노트를 달고 싶었으나 [[backend-완료된-task-문서-불변-원칙]]대로 손대지 않았다. series 종료 사실은 이 트랙 회고와 루트 ADR 문서의 후속 노트로만 표현했다.

별도 이슈도 발행하지 않았다. 선행 series는 상위 이슈(#195)로 sub-PR 묶음을 관리했지만, 단일 PR 1건이고 PR 본문에서 관련 PR들을 참조로 연결하면 추적성이 확보된다 — "이슈가 PR의 그림자밖에 안 되는 케이스"라 보일러플레이트를 줄였다.

## code-schema lag — 과도기 불일치, 표준화는 표본 1건이라 보류

series 진행 중 **과도기 lag**가 처음 생겼다 — 코드는 cross-aggregate association을 이미 해제했는데 DB FK는 아직 남아 있는 불일치 상태. Stock 트랙 머지 이후 Order/Payment를 거치며 코드 차원 association이 0건이 됐지만 DB FK 5건은 이 마무리 트랙까지 남았고, 약 하루의 lag를 의도적으로 단기간 연속 머지로 흡수했다. 이 lag는 Stock 트랙이 "schema 변경 0건 — 모든 sub-PR 완료 후 일괄 제거"를 메타 원칙으로 못 박은 결과다(부분 제거하면 일부 도메인만 FK 없는 불균형 schema가 되므로). 이는 코드가 정본이고 schema가 뒤따르는, 통제된 형태의 [[silent-schema-drift-패턴]]의 역방향 케이스다.

lag 기간 기술 위험은 낮았다 — Hibernate `validate`는 FK 존재 여부가 아니라 컬럼 이름·타입·nullable만 검사하고, FK가 남아 있는 동안 부모 row 삭제 시도는 DB가 거부해 공통 안전망 500으로 위임되는 의도된 동작이었다.

"허용 기간 표준"을 이 PR ADR에 박을지 검토했으나 **보류**했다. 표본 1건으로 일반화하는 건 이르다. 다른 cross-aggregate 정리 series(예: Member/Cart 통합 가능성)에서 lag가 반복 등장하면 그때 표본 2~3건을 모아 ADR로 일반화하는 게 적절한 타이밍이다. 지금은 lag 허용 근거가 Stock 트랙 ADR에만 있어, 다음에 같은 상황을 만나면 그 결정을 다시 찾아야 하는 탐색 비용이 남는다.

## 미해결·후속

- **lag 허용 기간 표준의 ADR 정립 시점:** 표본 1건이라 보류. 다른 series에서 반복 확인된 뒤 ADR화 — 미래 트랙.
- **운영 DB 배포 절차의 미결:** 이 PR 머지 후에도 운영 DB에는 여전히 FK 5건이 남아 있다. 별도 결정으로 뺀 것 자체는 옳으나, 그 "별도 결정"이 담당자·시점이 붙은 실제 action item이 되어야 기술 부채로 굳지 않는다.

## 근거

- [[raw/sessions/backend/2026-06-03-pr-203-cross-aggregate-fk-cleanup]] — PR #203 마무리 트랙. 실질 신규 결정(잔류 index·UNIQUE 유지), InnoDB FK-자동index 비대칭 5건 분해, 단일 V4·same-aggregate 제외·운영 배포 별도·task 불변, lag 발생과 표준화 보류의 출처.
- [[raw/sessions/backend/2026-06-03-pr-202-payment-jpa-association-decouple]] — 5개 FK 목록과 코드-schema 과도기 lag를 처음 명시한 선행 Payment 트랙 미해결 항목.
