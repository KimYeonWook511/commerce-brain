---
type: topic
status: stable
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [code-review, convention, yagni, defensive-programming, design-principle]
created: 2026-06-03
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-03-pr-200-order-jpa-association-decouple]]"
  - "[[raw/sessions/backend/2026-06-03-pr-202-payment-jpa-association-decouple]]"
---

# 단일 ADR보다 코드베이스 실제 패턴 우선 — 미사용 API·방어가드·자동리뷰 판단

## 한 줄 정의

작은 설계 판단(방어 가드를 둘지, 조회 메서드를 신설할지, 자동 리뷰 제안을 받을지)에서, 정적 원칙·단일 ADR보다 **코드베이스의 실제 패턴이 더 큰 정보를 준다.** FK→ID series(Order #200 / Payment #202)에서 반복 확인된 방법론이다.

## 미사용 API 신설 금지 — existsById 회수 사례, 사용처 grep 우선

Order sub-PR에서 초기 사용자 요청("`findById` 말고 `exists` 같은 메서드로 효율을 조금 높여보")을 **사용처 분석 없이** 도메인 시그니처 결정에 그대로 박아 `ProductRepository.existsById`를 신설했다. 구현 → 루트 문서 동기화 → 회고까지 다 끝낸 뒤 리뷰 단계에서야 호출처가 0건임이 드러나 회수했다.

- **회수 이유:** 모든 Order 생성 경로가 `product.getPrice()`를 `addOrderItem`의 unitPrice 인자로 넘긴다. 즉 어차피 객체를 로드해야 하므로 `existsById`로 대체 가능한 사용처가 0건이다. 코드베이스 컨벤션("사용하지 않는 코드 추가 금지")과 정면 충돌.
- **정리 비용:** 코드 파일뿐 아니라 태스크 문서 5개 + 루트 ADR 색인까지 되돌려야 했다. 성격이 다른 변경이라 코드 되돌림(refactor) 커밋과 문서 정리(docs) 커밋으로 분리했다.
- **다시 본다면:** 설계 단계에서 `grep "productRepository\." src/main` 한 줄이면 끝났을 일이다. "`findById`는 모든 컬럼 SELECT라 비효율"이라는 정적 분석만으로 결정하면, 실제 동적 호출 흐름에선 객체 필드가 어차피 필요하다는 사실을 놓친다. **사용자 결정이라도 사용처 확인 뒤에 적용한다.**

같은 함정이 바로 다음 Payment sub-PR에서 다시 나올 뻔했다 — Order 존재 검증용 신규 메서드. 이번엔 신설하지 않았다. 호출처(`PaymentApprovalService.completeApprovedPayment`)가 같은 트랜잭션에서 Order를 `findByMerchantPayKeyForUpdate`로 잠금 로드해 `completePayment()`·`getTotalPrice()`·`getId()`를 함께 쓰므로 Order 객체 로드는 어차피 필요하고, `findBy...`가 없으면 `ORDER_NOT_FOUND`를 던져 존재 검증이 이미 포함된다. 별도 메서드는 사용처 0건이 된다. **grep 한 줄이 ADR·구현·회고를 사후에 되돌리는 것보다 압도적으로 싸다.** 이 판단은 [[도메인-팩토리-long-id-시그니처-전환과-정책-표면화]]에서 존재검증 API를 신설하지 않은 것과 직접 연결되고, "불필요한 추상화(예: sealed interface) 금지"라는 프로젝트 컨벤션과 같은 결이다.

## 방어 가드 vs 정책 — 코드베이스 기존 패턴을 따라 판단

Order sub-PR에서 상품명 조립 `buildProductName`을 LAZY 프록시 자동 로드에서 명시적 `productsById.get(productId)` Map 조회로 바꾸면서 없는 상품이면 `null`이 나올 수 있게 됐고, AI 리뷰가 가드가 필요하다고 지적했다. 결과적으로 `firstProduct == null`이면 `ProductException(PRODUCT_NOT_FOUND)`을 던지는 가드를 **채택(accept)**했다.

- **정상 흐름에선 이 null이 날 수 없다.** soft delete된 상품은 Map에 여전히 포함되고([[product-soft-delete-deletedat-주문이력-보존]]), hard delete는 코드 경로와 FK 제약이 막는다.
- **그럼에도 넣은 이유 — 정책 일관성:** 정상 흐름만 사전 조회로 다루고 예외 상황은 공통 안전망(500)에 위임하되, 예기치 못한 상태를 조용히 통과시키기보다 명시적 예외로 끊는 게 [[find-first-write-not-check-db-unique-멱등]] 방침 위에서 일관된다.
- **결정적 근거 — 코드베이스의 실제 패턴:** `OrderCreateProcessor`가 이미 똑같은 `findAllById` + `Map.get` 패턴에 똑같은 형태의 가드를 쓰고 있었다. 단일 정책 문서(ADR)만으로 모든 사례를 정당화하지 말 것. **정책과 실제 패턴이 충돌하면 후자를 먼저 본다.**

## 자동 코드리뷰 제안 대응 3축 — 방어 검증 reject 사례

Payment sub-PR에서 Gemini Code Assist가 `createCompleted`의 `orderId`/`amount`에 null·범위 방어 검증을 두라고 권했으나 **기각(reject)**했다. 일반론적으로 옳아 보이는 자동 제안에 일관되게 답하는 세 축:

1. **다른 도메인 엔티티의 일관 컨벤션 확인:** 이 코드베이스의 정적 팩토리들(`Order`, `Stock`, `StockHistory`)은 null/range guard로 `IllegalArgumentException`류를 던지지 않는 게 일관된 컨벤션이다.
2. **호출처가 시스템 경계인지 내부인지 판단:** 호출처가 한 곳이고 거기서 영속된 Order 엔티티의 필드를 전달하므로 애초에 발생 불가능한 시나리오 — 시스템 경계가 아니라 내부 호출이다.
3. **프로젝트/시스템 원칙 인용:** "불필요한 추상화를 피한다" + "발생할 수 없는 시나리오에 검증을 넣지 말고, 검증은 시스템 경계에서만 한다"([[도달불가분기-방어금지-불변식테스트-돈정합성-통합테스트]]).

같은 위 두 사례가 보여주듯, 앞의 null 가드는 accept, 이 방어 검증은 reject다 — 코드베이스 패턴(같은 곳에 같은 가드가 이미 있나)과 경계 판단(외부 입력인가 내부 전달인가)이 방향을 가른다. 서로 다른 발견을 주는 두 리뷰가 보완적이라는 점도 관찰됐다: 일반 패턴을 짚는 AI 리뷰(gemini)와 코드베이스 일관성을 파는 subagent 독립 리뷰([[ai-코드리뷰-종합-층위통합과-심각도-도메인보정]])가 각각 null 가드 / 미사용 메서드·단가 비대칭을 끌어올렸다.

## 열린 질문

- **"검증은 시스템 경계에서만"이라는 원칙의 경계 사례.** 내부 호출인데도 명시적 예외를 넣은 null 가드(accept)와, 내부 호출이라 방어 검증을 뺀 케이스(reject)가 공존한다. 둘을 가른 건 "발생 가능성"이 아니라 "코드베이스에 같은 가드가 이미 있나 + LAZY 자동 안전망을 명시 조회로 대체하는 변경 의도"였다. 이 구분선의 형식화는 열려 있다([[persistence-exception-노출-경계-추상수준]]·[[unique-위반-예외번역-infra-adapter-경계와-flush-타이밍]]의 경계 판단과 닿는다).

## 관련 링크

- [[도메인-팩토리-long-id-시그니처-전환과-정책-표면화]] — 존재검증 API 신설 안 함, 같은 회수 학습
- [[cross-aggregate-fetch-join-대체-사용처별-분석과-응답-외부주입]] — "단일 원칙보다 사용처별 분석" 자매 원칙
- [[find-first-write-not-check-db-unique-멱등]] — 정상 흐름 사전 조회 / 예외 안전망 위임 방침
- [[ai-코드리뷰-종합-층위통합과-심각도-도메인보정]] — 자동 리뷰·독립 리뷰의 보완성

## 근거

- [[raw/sessions/backend/2026-06-03-pr-200-order-jpa-association-decouple]] — 미사용 API 신설 안 함·코드베이스 패턴 우선
- [[raw/sessions/backend/2026-06-03-pr-202-payment-jpa-association-decouple]] — 방어 검증 reject 사례(자동 리뷰 대응 3축)
