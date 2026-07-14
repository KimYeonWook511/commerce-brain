---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [product, cqrs, service-separation, admin, authorization, ddd]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-product-domain-overview]]"
---

# Product 공개 query 서비스와 관리자 command 서비스 분리

## 컨텍스트·문제(단일 서비스 비대화)

기존에는 단일 `ProductService`에 공개 목록·상세 조회와 관리자 등록/수정/삭제가 한 클래스에 섞여 있었다. DDD 이관 단계에서 이 구조를 그대로 두면 생길 문제를 짚었다.

- **클래스 비대화** — 조회와 쓰기가 계속 한 클래스에 쌓인다.
- **관리자 기능과 일반 기능이 섞이면 흐름이 안 보임** — 관리자 기능을 찾으려고 서비스 클래스를 하나하나 열어봐야 한다.
- **클래스명에서 의도가 안 드러남** — 호출부 가독성을 위해 클래스명부터 "공개인가 관리자인가"를 말해야 한다.

## 결정과 선택 이유(command/query 분리)

단일 `ProductService`를 두 서비스로 쪼갰다.

- `ProductQueryService` — 공개 목록·상세 조회. 비로그인 OK.
- `AdminProductService` — 관리자 등록·수정·soft delete.

command와 query 흐름을 한 서비스에 두지 않는다. 이는 [[ddd-이관-컨벤션-adapter-command-query-네이밍]]가 도메인 공통으로 정한 축(application 서비스는 command/query로 쪼갠다)의 **product 인스턴스**다. 같은 축으로 stock은 관리자/재고/동시성으로 갈리고([[관리자-재고조작-별도api-이력-감사-분리]] · [[stock-도메인-구조-개요]]), order도 유스케이스 단위로 나뉜다([[order-도메인-구조-개요]]).

## 부수적 근거(호출자·트랜잭션·DTO·로깅)

분리하면 코드·설계에서 자연히 갈리는 축들이 서비스 경계와 일치한다.

- **호출자 성격** — 공개 query는 비로그인, 관리자 command는 `ROLE_ADMIN`. 권한 경계가 서비스 단위로 정리된다.
- **트랜잭션 성격** — query는 `@Transactional(readOnly = true)`, 관리자는 쓰기 메서드마다 `@Transactional`.
- **DTO 모양** — query는 외부 노출용 최소 정보(목록은 id·이름·가격), command는 관리자 응답.
- **로깅 정책** — 단순 조회 query엔 INFO 로그가 불필요하고, 도메인 상태 전환을 일으키는 command엔 INFO 로그가 붙는다.

> [!note] 로깅 논거의 지위
> 로깅 대비는 분리를 **정당화하는 설계 논거**로 든 것이지, 이관 시점 코드에서 두 서비스의 로그 유무를 실측해 확정한 축은 아니다. 실측 근거가 아니라 방향 근거로 읽어야 한다.

## 트레이드오프

- 서비스 클래스가 2개가 되고 **둘 다 `ProductRepository`에 의존**한다 — 저장소 경계는 하나인데 서비스만 갈렸다.
- 대신 호출부에서 "공개 흐름인가 관리자 흐름인가"가 한눈에 보이고, 권한·트랜잭션·DTO·로깅이 서비스마다 하나의 성격으로 통일된다.

## 근거

- [[raw/sessions/backend/2026-05-29-product-domain-overview]]
