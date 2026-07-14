---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [cross-aggregate, fk, id-reference, ddd, aggregate, multi-module, migration, soft-delete]
created: 2026-06-02
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-02-issue-195-fk-to-id-motivation]]"
---

# cross-aggregate FK를 Long ID 참조로 — 멀티모듈 선행작업으로서의 동기와 전략

기존 도메인이 다른 aggregate를 객체로 광범위하게 물던 것을 `Long` ID 참조로 되돌리기로 한 트랙(Issue #195)의 **동기·전략**을 잡은 결정. 코드 작업 전 단계라 구체 PR 경계는 아직 정하지 않은 기획 성격의 기록이다. 실행판(메타 원칙·scope 규율)은 [[schema-무변경-decouple-series-메타원칙과-scope-규율]]로 이어진다.

## 컨텍스트 — 멀티모듈 부담과 cross-aggregate 객체 참조 부채

기존 도메인은 `Order.member`, `OrderItem.product`, `Payment.order`, `Stock.product`처럼 다른 aggregate를 `@ManyToOne`/`@OneToOne` 객체 참조로 광범위하게 물고 있다. 신규 도메인은 이미 "다른 aggregate는 `Long` ID로만 참조한다"는 방침으로 돌아섰지만([[cross-aggregate-fk-to-id-참조-컨벤션]], cart부터 적용), 기존 도메인의 객체 참조를 ID 참조로 되돌리는 트랙은 아직 열리지 않았다.

문제의식은 두 겹이다. (1) 멀티모듈로 가고 싶은데 학습 부담이 커서 바로 도입하기 어렵다. (2) 그런데 멀티모듈로 가려면 어차피 모듈 경계를 넘는 JPA 연관(다른 aggregate를 객체로 무는 `@ManyToOne`/`@OneToOne`)을 끊어야 한다. 즉 **cross-aggregate 참조를 ID로 바꾸는 일은 멀티모듈의 선행 작업**이다.

## 검토한 대안 — Order→Payment 의존 방향 두 갈래

FK를 끊으면 도메인 간 의존이 늘어나 *보이지만*, 실제로는 **JPA lazy loading 뒤에 숨어 있던 의존이 겉으로 드러나는 것**뿐이다. 총량은 비슷하고 대신 의존이 명시적이 되니 통제 가능성이 올라간다. 이 프레임이 없으면 "ID 참조로 가면 코드가 더러워진다"는 표면 반응에 휘말린다. 의존을 어느 방향으로 둘지 두 갈래를 봤다.

- **강한 DIP (처음 떠올린 쪽):** Payment가 Order가 정의한 port를 구현하게 해서 의존 방향을 역전시키는 안. DIP를 세게 적용하는 형태. 트레이드오프는 port/adapter 추상이 늘어 결합 통제는 강하지만 구조가 무거워진다.
- **Order가 Payment를 소비 (택함):** Order가 자기 adapter를 들고 Payment의 공개 facade를 호출한다. 의존 방향은 `Order → Payment` 한 방향이고, Payment는 자기를 누가 쓰는지 모른다. Payment는 "사실을 제공하는 쪽", Order는 "그 사실을 소비하는 쪽"이다.

## 결정 — 멀티모듈 선행작업으로 FK→ID, 'FK 제거'를 세 변경으로 분리, Order→Payment 단방향

**멀티모듈을 건너뛰는 게 아니라 준비 단계를 먼저 끝내둔다.** cross-aggregate 참조를 다 끊어두면, 나중에 멀티모듈로 갈 때 모듈 경계 긋기와 build.gradle 분리 정도만 남는다. 학습 부담을 분산하려고 찾은 우회로가 곧 선행 작업이라는 인식이 이번 트랙의 근거다.

"FK 제거"는 한 덩어리로 부르지만 실제로는 성격이 다른 셋이 섞여 있다. 이걸 갈라둬야 PR 단위를 제대로 자른다.

| 변경 | DB 변경 | 코드 수정 |
|---|---|---|
| DB FK 제약 DROP | 스키마 변경 있음 | **JOIN 쿼리 영향 0** — FK는 참조 무결성 제약일 뿐 JOIN의 전제가 아니다. JOIN은 엔티티 연관/명시적 조인으로 생성되지 FK 유무에 안 걸린다 |
| `@ManyToOne`/`@OneToOne` → `Long fooId` | 없음 | join fetch / QueryDSL path / `entity.getOther()` 다수 수정 |
| cross-aggregate `@OneToMany` 제거 | 없음 | 호출부만 `findByYyyId(...)` 조회로 교체 |

이 분리가 곧 PR 단위 기준이다. 두 번째·세 번째는 DB를 안 건드리니 코드만 바꾸면 되고, 첫 번째(실제 FK DROP)는 스키마가 바뀐다. 그래서 코드 마이그레이션을 먼저 다 끝내고 FK DROP은 마지막에 일괄로 미룰 수 있다(→ 실제 일괄 DROP은 [[fk-drop-후-잔류-index-unique-유지와-innodb-비대칭]]).

**같은 aggregate 내부(예: `Order ↔ OrderItem`)는 풀지 않는다.** cross-aggregate가 아니라 root-child 관계라 객체 참조와 FK를 그대로 둔다. 신규 도메인 방침에서도 same-aggregate는 대상 밖이다.

의존 방향은 `Order → Payment` 단방향으로 두되, **핵심은 어느 안이냐가 아니라 두 불변식**이다: (1) Order가 Payment 엔티티를 직접 import 하지 않는다, (2) 의존이 한 방향이다. 멀티모듈로 갔을 때 모듈 간 cycle을 막는 게 본질이고, 두 안 모두 그걸 만족하는 방식일 뿐이다.

> [!note] 범위 밖 — Order↔Payment 경계 설계
> 이 세션 시점의 판단은 `Order → Payment` 단방향이라는 방향 설정까지다. 실제 Order↔Payment 경계 설계·결합 끊기는 이후 별도 트랙에서 다시 다뤄졌다([[payment-order-facade-결합끊기-tell-dont-ask]], [[payment-order-도메인분리와-pg격리]]).

## FK가 만든 부채 — DDD 'ID로 참조' 원칙과 상품 soft delete 강제 사례

cross-aggregate 참조를 ID로 하기로 한 결정 문서에는 객체 참조의 누적 부채가 정리돼 있다(N+1·fetch join 부담, 도메인 결합도 증가, 단위 테스트에서의 객체 그래프 구성 부담, "다른 aggregate는 ID로만 참조"라는 DDD 정통 원칙 위반). 이번에 다시 인식한 건, 그중 **"DDD 정통 원칙 위반"이라는 한 줄이 사실 최범균 『도메인 주도 개발 시작하기』 3장(애그리거트), 특히 "애그리거트를 ID로 참조" 절 전체를 압축한 것**이라는 점이다. 책이 드는 세 문제가 부채 항목들에 흩어져 걸쳐 있다.

- **편한 탐색의 오용** → 결합도 증가의 *본질*. 직접 객체 참조가 있으면 사실 판단이 객체 traversal 뒤로 숨어 책임 경계가 흐려진다. 보상 정책 판단을 Payment 도메인의 사실 조회로 정리한 PR #192가 이 문제의 실제 사례다([[payment-완료여부-사실조회-hascompletedpayment-srp]]).
- **성능 고민** → N+1 / fetch join 부담.
- **확장 어려움** → 미래 마이크로서비스 분리 경로.

FK가 **비즈니스 정책을 외부에서 결정해버리는** 실물 사례도 있다. `tbl_order_item → tbl_product` FK 때문에 상품을 물리 삭제하면 과거 주문 데이터의 참조 무결성이 깨진다. 그래서 상품 삭제는 `deletedAt` 기반 soft delete로 갈 수밖에 없었다([[product-soft-delete-deletedat-주문이력-보존]]). FK라는 DB 제약이 애플리케이션의 삭제 정책을 바깥에서 못박은 것 — FK를 걷어내면 이런 정책을 애플리케이션이 스스로 정할 자유가 생긴다.

## 진행 방식 — 도메인 경계 단위 점진 PR, DB FK DROP은 맨 마지막 일괄

한 번에 다 풀지 않고 도메인 경계 단위로 쪼개 점진적으로 간다.

- **워밍업 후보:** cross-aggregate `@OneToMany` 제거 — DB를 안 건드리고 호출부만 `findByYyyId(...)`로 교체하면 되니 가장 단순하다.
- **DB FK 실제 DROP은 맨 마지막에 일괄.** 모든 코드 마이그레이션이 끝난 뒤 한 번에 뗀다.
- **운영 DB 적용은 별도 결정으로 분리.** FK의 안전망 가치와 코드 일관성 가치를 그 시점에 다시 저울질한다.

## 미해결·후속

- **첫 PR 경계 미정:** 현재 코드의 `@ManyToOne`/`@OneToMany` 분포를 스캔한 뒤 가장 단순한 경계를 골라야 한다. cross-aggregate `@OneToMany`만 걸린 곳이 있으면 워밍업으로 적합하다는 것까지가 지금의 가설.
- **deferred 트레이드오프 — 운영 DB FK 안전망 vs 코드 일관성:** 사용자가 SQL로 DB를 직접 손대는 환경이 아니라면 참조 정합성 책임을 애플리케이션으로 옮기는 게 멀티모듈 미래까지 일관된다. 다만 코드 마이그레이션이 다 끝난 마지막 시점에 다시 판단한다. 이 저울질은 series 마무리 트랙까지도 action item으로 미결로 남았다([[fk-drop-후-잔류-index-unique-유지와-innodb-비대칭]]).

## 근거

- [[raw/sessions/backend/2026-06-02-issue-195-fk-to-id-motivation]] — Issue #195의 동기·진행 원칙 정리(코드 작업 전 단계). 멀티모듈 선행작업 프레임, 'FK 제거' 세 분리, Order→Payment 단방향 두 불변식, 최범균 DDD 책·상품 soft delete 부채 사례의 출처.
