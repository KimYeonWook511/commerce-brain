---
type: tradeoff
status: open
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [cart, abuse, rate-limit, pagination, capacity]
created: 2026-05-28
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-28-pr-166-cart-review-cycle]]"
---

# 회원당 cart row 개수 상한 — 이 phase 미도입

> [!note] 미결 tradeoff
> 이 phase에서 의도적으로 상한을 두지 않고 트랙으로만 남긴 저울질이다. 이후 raw에서 상한 도입 결정이 나오면 이 노트를 decided로 전환하거나 supersede한다.

## 컨텍스트 — 수량 상한은 있으나 row 개수는 무가드

항목당 수량은 `MIN=1`, `MAX=99`로 도메인이 강제한다. 그러나 회원당 cart row 개수(서로 다른 productId 수)에는 아무 가드가 없다. 무제한으로 서로 다른 상품을 담을 수 있다.

## abuse 시나리오(외부 agent 지적)

외부 agent 검토가 abuse 시나리오를 지적했다: 자동화로 수천 개의 서로 다른 productId를 add하면 (1) cart row가 무한정 누적되고, (2) cart 조회 시 `findAllById(IN ...)`([[cross-aggregate-fk-to-id-참조-컨벤션]]의 조회 방식)의 `IN` 절 binding이 그만큼 커져 조회 부담이 생긴다.

## 이 phase에서 미도입한 이유 — scope·상위 게이트 책임

상한을 두지 않았다. 세 근거다.

- **정상 사용자 부담 없음**: 정상 사용자의 cart 항목 수는 보통 수십 단위라 `IN` 절 부담이 실질적이지 않다.
- **상한 도입이 부르는 결정 누적**: 상한을 도입하면 "어디(도메인/application/DTO)에 둘지, 초과 시 어떤 행위를 거부할지(가장 오래된 항목 자동 제거? 4xx?), invariant를 어떻게 표현할지" 같은 추가 결정이 줄줄이 따라와 이 phase의 scope를 넘는다.
- **abuse는 상위 게이트 책임**: 자동화 abuse는 application 레벨보다 인증·rate limiting 등 상위 게이트의 책임에 더 가깝다.

## 저울질 — (a)row 상한 (b)페이지네이션 (c)rate limiting

운영에서 문제가 관측되면 셋 중 하나를 택한다. 아직 미결이다.

- **(a) 회원당 row 상한**: cart 도메인에 상한 invariant를 넣는다. 초과 시 거부 행위 정의가 필요.
- **(b) GET /cart 페이지네이션**: 조회를 나눠 `IN` 절·응답 크기 부담을 낮춘다. row 누적 자체는 못 막는다.
- **(c) rate limiting 게이트 강화**: add 요청 빈도를 상위에서 제한한다. abuse의 근본을 상위 게이트에서 차단.

## 트리거 — abuse 관측·IN 절 한도 임박

다음 중 하나가 트리거다: (1) 운영에서 abuse가 실제 관측되거나, (2) `IN` 절 binding 한도가 임박할 때. 그 시점에 위 (a)/(b)/(c) 중 택1한다.

## 열린 질문

- 어느 선택지가 최선인지는 트리거 상황(abuse 성격 vs 순수 조회 부하)에 따라 달라진다 — 관측 없이 선택하지 않는다.
- insert race 멱등 흡수 재방문([[cart-동시성-낙관락-processor-분리-retry]])과 함께 cart의 대표 미해결 항목이다.

## 근거

- [[raw/sessions/backend/2026-05-28-pr-166-cart-review-cycle]]
