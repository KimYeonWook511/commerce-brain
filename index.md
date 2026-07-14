---
type: meta
updated: 2026-07-14
---

# commerce-brain — wiki 카탈로그

wiki 페이지를 타입별로 한 줄 요약한다. **ingest 마다 갱신**한다. 분류의 진실은 각 노트 frontmatter이고, 이 카탈로그는 탐색 보조다.

총 101개 노트 · MOC 22개.

## decisions (79)

- [[결과불명-unknown-보존-alreadycomplete-cancel-경로확장]] — 결과 불명 시 UNKNOWN 보존을 AlreadyComplete 이력 재확인·cancel 경로로 확장
- [[결제-도메인-orm-선택과-jpa-오염-격리-실용진영]] — 실용진영 유지, 명시적 save 컨벤션
- [[결제-후처리-대상식별-status중심-재설계]] — 결제 후처리 대상 식별을 실패코드 열거에서 status(UNKNOWN·stale REQUESTED) 중심으로 재설계
- [[결제-escalation-종착통지-escalatedAt-직교필드]] — 새 상태 대신 escalatedAt 직교 필드
- [[결제승인완료-보상-완료우선-이중결제-adapter매핑]] — 완료 우선 + 이중결제 adapter 매핑
- [[관리자-재고조작-별도api-이력-감사-분리]] — 관리자 재고 조작 별도 API + 변경만 이력에 남긴다 (감사 목적 분리)
- [[대사-확정-검증보상-대칭-재승인없음]] — 실시간과 대칭 검증·중복결제 환불 누락 수정
- [[대사-keep-waiting-backoff-next-reconcile-at]] — status-직교 next_reconcile_at 필드로 재조회를 미룬다
- [[도메인-예외-httpstatus-제거-errorcategory]] — HttpStatus 대신 의미 분류(ErrorCategory)
- [[도메인-팩토리-long-id-시그니처-전환과-정책-표면화]] — 부가효과로 드러난 정책과 fixture 침투
- [[미확정차단-대사스캔-정합성-starvation-escalation]] — over-blocking·starvation은 escalation 종착 한 곳에서 풀린다
- [[보상판단-payment-존재-lock-대신-db-unique]] — 락 대신 DB unique 통일 원칙
- [[보상흐름-설계-payment-application-책임-pgcanceller-dispatcher]] — 정책은 payment.application, PG는 cancel 콜백, 의미별 dispatcher 분리
- [[예약-동시소비-가드-version-vs-cas]] — @Version vs CAS, 정확성 동등 시 도메인 표현력 우선
- [[이중결제보상-완료가드-제거-pgpaymentid-무조건취소]] — 보상 대상 pgPaymentId 무조건 취소로 전환
- [[인증-패키지-경계-auth-member-security-분리]] — auth/member/security 3분할과 의존 방향
- [[인프라-일시장애-503-예외-매핑-판단축]] — 예외 상속 전략의 두 직교 판단축
- [[재고복구-동기취소-vs-outbox-비동기만료-비대칭]] — 사용자취소는 동기, 만료는 Outbox+Kafka 비동기
- [[재고차감-동시성-비관락과-productid-정렬]] — 낙관락·Redis락·atomic UPDATE 기각
- [[주문-멱등-캐시-inflight-차단-전용]] — 결과 캐싱 제거
- [[주문-멱등성-캐싱-after-commit-이벤트-분리]] — @TransactionalEventListener(AFTER_COMMIT)로 Redis 분리
- [[주문-멱등성-redis-setnx-plus-rdb-unique-이중방어]] — Redis SETNX 1차 + RDB 복합 unique 2차 이중방어
- [[주문-이중결제-앞단-진입차단-예약조회-단일화]] — 주문 이중결제 앞단 진입 차단 + 예약 조회 단일화·미발견 코드 분리
- [[주문만료-spring-batch-chunk-retry-skip]] — Spring Batch(chunk 100·retry/skip 차등·keyset 페이징)
- [[테스트-태그-2축-모델-ci-잡-분리]] — 테스트 @Tag 2축 모델(환경 요구·격리 분류) + CI 잡 단위슬라이스/통합배치 분리
- [[회원가입-not-supported-트랜잭션-분리]] — REQUIRED/AFTER_COMMIT/NOT_SUPPORTED 3자 비교
- [[auth-redis-장애-도메인예외-매핑-도메인전용-어드바이스]] — Auth Redis 장애를 인프라 도메인 예외 매핑 + 도메인 전용 어드바이스로 통일
- [[cart-도메인-골격-cartitem-단일-aggregate]] — CartItem 단일 entity aggregate
- [[cart-동시성-낙관락-processor-분리-retry]] — 낙관락 + Processor 분리 + retry
- [[cart-회원당-row-상한-미도입]] `open` — 이 phase 미도입
- [[cart-add-product-존재-상태-검증]] — 장바구니 추가 시 Product 존재·상태 검증
- [[cart-delete-미존재-4xx-entity-경유-삭제]] — DELETE 미존재 4xx + entity 경유 삭제
- [[cart-path-id-검증-spec을-코드에-맞춤]] — spec을 코드에 맞춤
- [[cross-aggregate-fetch-join-대체-사용처별-분석과-응답-외부주입]] — 단일 원칙 대신 사용처별 분석 + 응답 DTO 외부 주입
- [[cross-aggregate-fk-to-id-마이그레이션-동기-전략]] — 멀티모듈 선행작업으로서의 동기와 전략
- [[cross-aggregate-fk-to-id-참조-컨벤션]] — 신설 도메인 컨벤션
- [[enum-db-check-미사용-application-layer-위임]] — 유효성 보장을 application layer로 위임
- [[fk-drop-후-잔류-index-unique-유지와-innodb-비대칭]] — 잔류 index·UNIQUE 유지와 InnoDB FK-index 비대칭
- [[flyway-도입-ddl-auto-validate-전환]] — 단일 DB에서도 silent drift는 코드 내부 요인으로 일어난다
- [[jwt-redis-하이브리드-rtr-ttl-근거]] — TTL 근거와 한 회원 한 활성 refresh
- [[mdc-정리-스코프-오너십-2규칙]] — memberId 필터 간 릴레이 제거
- [[multi-column-unique-length-명시-컨벤션]] — enum length 미명시 컨벤션의 좁은 예외
- [[order-concurrency-service-학습코드-격리]] — production과 격리한 동시성 학습 코드
- [[order-version-낙관락-비관락-공존]] — 만료/결제 race를 @Version이 잡는다
- [[orderitem-단가-snapshot-컬럼과-backfill-leftjoin-coalesce]] — LEFT JOIN + COALESCE backfill
- [[paid-order-취소환불-단일tx-의도와-standalone-cancel-대사]] — 단일 tx 환불 의도 + standalone CANCEL 대사로 보장
- [[payment-낙관락-충돌처리-3계층-흡수는-트랜잭션-밖]] — 흡수를 트랜잭션 밖으로 빼는 3계층 구조
- [[payment-낙관적-락-도입-왜-비관-아님]] — 왜 비관 락이 아닌가
- [[payment-동시성-unique-vs-lock-gap-lock회피]] — 왜 lock이 아니라 unique 제약인가 (gap lock 회피)
- [[payment-부분취소-모델만-열고-구현-보류]] `open` — 모델만 열어두고(amount+SUM) 로직은 전액취소만 구현
- [[payment-완료여부-사실조회-hascompletedpayment-srp]] — hasCompletedPayment
- [[payment-이중결제-reserve따닥-mysql-null트릭-unique]] — MySQL NULL 트릭 partial unique
- [[payment-amount-mismatch-이중검증-409-vs-400-분리]] — 재요청 mismatch(409)와 PG 검증(400) 두 검증 분리
- [[payment-append-only-원장과-exists-완료판단]] — 결제 시도 append-only 원장과 EXISTS 기반 완료 판단
- [[payment-attempt-네이밍-정리와-refactor-경계]] — 옛 엔티티 잔재만, 진짜 '시도'는 보존
- [[payment-order-도메인분리와-pg격리]] — 재설계 동기와 식별자 재배치
- [[payment-order-트랜잭션-경계-cross-aggregate-단일tx]] — 외부 호출은 밖, order·payment 두 aggregate 쓰기는 한 트랜잭션
- [[payment-order-facade-결합끊기-tell-dont-ask]] — Tell-Don't-Ask와 facade 조율
- [[payment-reserve-예약테이블-분리-a안-b안]] — 단일 결제 테이블(A안)에서 예약/사건 두 테이블(B안)로
- [[payment-reserve-ready-흐름-재설계-expiresat-재사용만료]] — RESERVED 예약 행 생성·재사용·만료
- [[payment-status-사실만-분류는-정책계산-manual-review-철회]] — MANUAL_REVIEW 새 상태 철회
- [[payment-unknown-결과불명-처리와-예외분류]] — 세 번째 상태와 예외 분류
- [[paymentattempt-상태전이-도메인-검증-defensive]] — 멱등 자기 전이까지 거부
- [[paymentattempt-호출정책-문서-javadoc-archunit-보류]] — ArchUnit 대신 문서·JavaDoc·단일 호출처
- [[persistence-exception-노출-경계-추상수준]] — 영속성 예외 노출 경계를 예외의 추상 수준(추상 vs 구현 구체)으로 다시 긋기
- [[pg-승인-예외-경계-요청전송시점]] — "요청 전송 시점"을 경계로 버그 전파와 UNKNOWN 보존을 가른다
- [[product-공개query-관리자command-서비스-분리]] — Product 공개 query 서비스와 관리자 command 서비스 분리
- [[product-상세조회-stock-의존-재고누락-0-정규화]] — 상품 상세조회 stock 의존 + 재고 누락 시 stockQuantity=0 정규화
- [[product-mvp-범위-imageurl-카테고리-페이지네이션-제외]] — imageUrl 문자열 저장 + 카테고리·페이지네이션·초기재고 제외
- [[product-soft-delete-deletedat-주문이력-보존]] — 주문 이력 보존이 결정을 강제
- [[productstatus-3상태-공개노출-정책]] — ProductStatus 3상태(ON_SALE/SOLD_OUT/STOPPED)와 공개 노출 정책
- [[redis-장애-멱등캐시-fallback-boolean-예외분리]] — boolean을 도메인 예외로 분리
- [[redis-장애-strict-정책-soft-fail-기각]] — soft fail의 함정
- [[refreshtokenstore-delete-제거-로그아웃-미구현]] — 로그아웃 미구현의 흔적
- [[requires-new-격리-제거-보상판단-트랜잭션정책]] — 근거 사라진 방어벨트와 잘못 박힌 정책 문구 정정
- [[schema-무변경-decouple-series-메타원칙과-scope-규율]] — decouple series의 메타원칙과 scope 규율
- [[security-common-leaf-재편과-토큰-포트-네이밍]] — 인증/인가 진입점을 common.security leaf로 재편 + 토큰 포트/어댑터 네이밍
- [[sql-translator-빈-제거-제약명-이중결제-식별]] — 이중결제 식별을 제약명 기반으로 전환
- [[unique-위반-예외번역-infra-adapter-경계와-flush-타이밍]] — infra adapter 경계와 flush 타이밍

## contracts

_(없음)_

## topics (9)

- [[동시성-테스트-작성-규칙과-단언-전략]] — "타이밍을 운에 맡기지 않는다"
- [[서블릿-필터-예외-처리와-에러-디스패치]] — 전역 핸들러 경계와 OncePerRequestFilter
- [[auth-member-security-도메인-구조-개요]] — 3패키지 분리와 인증 데이터 흐름
- [[jwt-예외-catch-footgun-bare-securityexception]] — bare SecurityException이 java.lang으로 조용히 바인딩
- [[order-도메인-구조-개요]] — 엔티티·상태·서비스 경계 지도
- [[payment-도메인-구조-개요]] `draft` — 두 Aggregate와 PG 연동 경계
- [[product-도메인-구조-개요]] — 공개조회/관리자관리 분리와 기반 도메인 위치
- [[silent-schema-drift-패턴]] — Hibernate 자동 추론 의존 + 엔티티 의도 미명시
- [[stock-도메인-구조-개요]] — 차감·복구·이력·동시성 격리의 세 축

## incidents

_(없음)_

## knowledge (13)

- [[도달불가분기-방어금지-불변식테스트-돈정합성-통합테스트]] — 돈 정합성은 통합 테스트로
- [[문서-코드-정합성-개념정본-심볼최소화]] — 문서는 개념·정책·왜의 정본, 현재 코드 형상·심볼은 코드가 단일 출처
- [[보상-catch-2차예외-평탄화-원칙]] — 예외 안 던지는 의도 캡슐화 메서드
- [[설계단계-검증-정본·선례-확인실패와-전제의-적대적검증]] `draft` — 기존 결정·컨벤션·선례 확인 실패 패턴과 전제의 적대적 검증
- [[스코프-규율-한-pr-한-목적-인접부채-별도이슈-분리]] — 한 PR 한 목적, 발견한 근본 개선·인접 부채는 즉석 패치 말고 처리처로 분리
- [[코드베이스-패턴-우선-설계판단-미사용api-방어가드-자동리뷰]] — 미사용 API·방어가드·자동리뷰 판단
- [[하네스-프로세스-무게와-문서-동작-정합]] — 규칙은 기억이 아니라 정본으로 확인
- [[하네스-ac-검증범위-격리테스트와-위험경로-결정적강제]] `draft` — 격리 테스트를 AC에 넣고 위험 경로를 결정적으로 강제
- [[ai-코드리뷰-종합-층위통합과-심각도-도메인보정]] — 상반돼 보이면 층위로 통합, 심각도는 도메인으로 보정, blind accept 금지
- [[backend-완료된-task-문서-불변-원칙]] — commerce-backend 문서 운영 원칙
- [[ddd-이관-컨벤션-adapter-command-query-네이밍]] — adapter·command/query·의도기반 메서드명·legacy 분리
- [[find-first-write-not-check-db-unique-멱등]] — 사전 find로 정상 멱등 흡수, unique 위반 race만 안전망 500
- [[jpa-메커니즘-이점-한계와-ddd-괴리-트레이드오프]] — 실무 트레이드오프 정리

## features (MOC, 22)

- [[auth]] (7)
- [[cart]] (6)
- [[compensation]] (11)
- [[concurrency]] (18)
- [[ddd]] (9)
- [[escalation]] (7)
- [[exception-handling]] (17)
- [[idempotency]] (20)
- [[jpa]] (11)
- [[migration]] (6)
- [[naverpay]] (7)
- [[optimistic-lock]] (9)
- [[order]] (21)
- [[payment]] (38)
- [[pessimistic-lock]] (6)
- [[product]] (7)
- [[reconciliation]] (9)
- [[redis]] (9)
- [[reservation]] (6)
- [[saga]] (5)
- [[stock]] (6)
- [[unique-constraint]] (10)
