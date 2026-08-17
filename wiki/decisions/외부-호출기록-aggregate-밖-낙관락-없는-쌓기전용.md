---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [payment, refund, aggregate, append-only, optimistic-lock, ledger, pg-gateway, security]
created: 2026-08-16
updated: 2026-08-17
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-08-16-payment-refund-aggregate-boundary]]"
---

# 외부 호출 기록은 aggregate 밖에 둔다 — 낙관 락 없는 쌓기 전용 테이블

## 결정

승인·환불 호출을 한 테이블에 쌓고 **응답 원본을 그대로 담는다.** 자기 저장소로 저장하고 **낙관 락을 갖지 않는다.**

기록을 aggregate 자식으로 두면 순수한 기록을 남기는 일이 부모를 통해야 하고 그 과정에서 낙관 락 충돌을 일으킨다. **기록은 어떤 불변식에도 참여하지 않고 판정에도 쓰이지 않는데** 그렇다.

**판정에 쓰는 값은 이 테이블이 아니라 거래 테이블에 둔다. 쌓기만 하고 읽어서 판단하지 않는다는 것이 이 테이블의 계약이다.**

## 이전 결정과의 관계

결제 재설계 초기에 "PG 응답 원문 전체를 담는 테이블"은 무겁다고 보고 로그로 미뤘고, 승격 트리거(분쟁·CS 증가, 대사 자동화 필요, 결제량 규모 초과)를 정해뒀다([[payment-append-only-원장과-exists-완료판단]]). 이번에 그 트리거가 충족돼 테이블로 올라온 셈이다 — 회수 배치가 결제사 응답을 근거로 결과를 확정해야 하고([[결과회수-상한-폐지와-백오프-표-통지-반복]]), 결과 불명 판정이 전송 계층 실패 유형까지 되짚어야 하기 때문이다([[외부-돈-호출-결과어휘-넷과-전송계층-판정-우선]]).

## 트레이드오프·미해결

- **호출 기록에 마스킹된 카드번호·영수증 주소 같은 것이 원본째 담긴다. 보관 기간과 접근 제한을 정하지 않았다.**
- **호출 기록이 거래 행과 어긋나도 자동으로 드러나지 않는다.** 판정에 안 쓰이기 때문이다. **기록을 남기는 자리를 한 곳으로 묶어 빠뜨리지 않게 하는 것이 지금의 대비 전부다.**
- 낙관 락이 없으므로 같은 건에 대한 동시 기록이 둘 남을 수 있다. 쌓기 전용이라 정합성을 깨지 않지만 중복 행을 읽는 쪽이 생기면 그 전제가 무너진다.

## 근거

- [[raw/sessions/backend/2026-08-16-payment-refund-aggregate-boundary]]
