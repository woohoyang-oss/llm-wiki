# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## ⚠️ 작업 프로세스 (필수)

**모든 코드 작업 전에 반드시 이 순서를 따른다:**

1. 사용자의 요청에서 **관련 도메인을 식별**한다 (아래 Domain-Package Quick Reference 참조)
2. 해당 도메인의 상세 문서를 **반드시 읽는다**: `.claude/domains/{도메인}.md`
3. 상세 문서에서 정확한 Controller/Service/DAO 클래스명을 확인한 후 코드를 탐색한다

> 예: "휴가 승인 로직 수정" → Quick Reference에서 hr/ 확인 → `.claude/domains/hr.md` 읽기 → VacationService 확인 → 코드 수정

**이 프로세스를 건너뛰고 코드베이스 전체를 탐색하지 않는다.**

## Project Overview

TBNWS Admin Backend - Spring Boot 3.4.8 기반의 관리자 백엔드 시스템. 다중 브랜드(GTGear, Keychron, Aiper) e-commerce 운영을 위한 ERP, CRM, 주문, 재고, HR, 재무 등의 도메인을 관리한다.

- **Java**: 17
- **Build**: Gradle (Groovy DSL)
- **ORM**: MyBatis (XML mapper 기반)
- **DB**: MySQL (9개 DataSource) + PostgreSQL (pgvector)
- **Cache/Session**: Redis (Redisson, Spring Session)
- **Security**: Spring Security + Google OAuth2

## Build & Run Commands

```bash
# 빌드 (테스트 제외)
./gradlew clean build -x test

# 로컬 개발 실행 (dev 프로필, 포트 8080)
./gradlew bootRun --args='--spring.profiles.active=dev'

# 테스트 전체 실행 (Docker 필요 - Testcontainers 사용)
./gradlew test

# 단일 테스트 클래스 실행
./gradlew test --tests "com.tbnws.gtgear.support.tbnws_admin_back.erp.service.ERPServiceTest"

# 단일 테스트 메서드 실행
./gradlew test --tests "com.tbnws.gtgear.support.tbnws_admin_back.erp.service.ERPServiceTest.testMethodName"
```

## Domain-Package Quick Reference

> **기능 요청이 들어오면 아래 표를 먼저 확인하여 해당 도메인 패키지를 찾는다.**
> 상세 내용은 `.claude/domains/{도메인}.md` 참조.

루트 패키지: `com.tbnws.gtgear.support.tbnws_admin_back`

| # | 도메인 | 패키지 | API Prefix | 프론트 Route | 상세 문서 |
|---|--------|--------|------------|-------------|-----------|
| 1 | 통합 ERP | `erp/` | `/api/erp/*`, `/api/serial/*`, `/api/v1/fulfillment/*` | `/erp`, `/fulfillment` | [erp.md](.claude/domains/erp.md) |
| 2 | HR | `hr/` | `/api/hr/*`, `/api/vacation/*`, `/api/overtime/*`, `/api/businessTrip/*`, `/api/v1/hr/*` | `/hr` | [hr.md](.claude/domains/hr.md) |
| 3 | Naver Commerce | `naver/` | `/api/naver/*` | `/naver`, `/brandstore` | [naver.md](.claude/domains/naver.md) |
| 4 | Admin/Dashboard | `admin/` | `/api/settings/*`, `/api/widget/*`, `/api/alert/notice/*`, `/api/biz/*`, `/api/dev/*`, `/api/page-builder/*` | `/`, `/settings`, `/alert`, `/biz`, `/dev` | [admin.md](.claude/domains/admin.md) |
| 5 | B2C CRM | `b2c/` | `/api/b2ccrm/*`, `/api/blog/*` | `/b2ccrm` | [b2c.md](.claude/domains/b2c.md) |
| 6 | B2B CRM | `b2b/` | `/api/v1/b2bcrm/*` | `/b2bcrm` | [b2b.md](.claude/domains/b2b.md) |
| 7 | Message | `message/` | `/api/message/*`, `/api/send/*` | `/message`, `/chatbot` | [message.md](.claude/domains/message.md) |
| 8 | AI | `ai/` | `/api/ai/*` | `/dev` | [ai.md](.claude/domains/ai.md) |
| 9 | Chatbot | `chatbot/` | `/api/send/*` (카카오 챗봇 웹훅) | `/chatbot` | [chatbot.md](.claude/domains/chatbot.md) |
| 10 | Shopify | `shopify/` | `/api/shopify/admin/*`, `/api/open/shopify/*`, `/api/open/keychron/*` | `/shopify` | [shopify.md](.claude/domains/shopify.md) |
| 11 | Event | `event/` | `/api/event/*` | `/event` | [event.md](.claude/domains/event.md) |
| 12 | WMS | `wms/` | `/api/wms/*` | `/wms` | [wms.md](.claude/domains/wms.md) |
| 13 | Finance | `finance/` | `/api/v1/finance/*`, `/api/finance/*` | `/finance` | [finance.md](.claude/domains/finance.md) |
| 14 | API 연동 | `api/` | `/api/open/*` | - | [api.md](.claude/domains/api.md) |
| - | Config | `config/` | - | - | [config.md](.claude/domains/config.md) |
| - | Util | `util/` | - | - | [config.md](.claude/domains/config.md) |

### Cross-Domain Facade 서비스

복수 도메인에 걸친 트랜잭션 처리:

| Facade | 위치 | 조합 도메인 |
|--------|------|------------|
| `B2BFacadeService` | `b2b/service/facade/` | B2BCRM + ERP + Finance (수주 등록 → 세금계산서 생성) |
| `OrderFacadeService` | `erp/service/facade/` | Order + WMS + ERP (발주 처리, 환율 계산, 재고 동기화) |
| `FinanceFacadeService` | `finance/service/facade/` | Finance 크로스 도메인 처리 |

### API Prefix로 도메인 역추적

```
/api/erp/*                → erp/
/api/serial/*             → erp/ (SerialController)
/api/v1/fulfillment/*     → erp/ (FulfillmentController)
/api/hr/*                 → hr/
/api/vacation/*           → hr/ (VacationController)
/api/overtime/*           → hr/ (OvertimeController)
/api/businessTrip/*       → hr/ (BusinessTripController)
/api/v1/hr/*              → hr/ (WelfareController)
/api/calendar/*           → hr/ (CalendarController)
/api/naver/*              → naver/
/api/settings/*           → admin/ (SettingController)
/api/widget/*             → admin/ (WidgetController)
/api/alert/notice/*       → admin/ (NoticeController)
/api/biz/*                → admin/ (BizController)
/api/dev/*                → admin/ (DevController)
/api/page-builder/*       → admin/ (PageBuilderController)
/api (공통 엔드포인트)     → admin/ (BaseController: memberInfo, menuList 등)
/api/b2ccrm/*             → b2c/ (B2CCRMController)
/api/blog/*               → b2c/ (BlogController)
/api/open/wp/*            → b2c/ (WpController)
/api/v1/b2bcrm/*          → b2b/ (B2BCRMController)
/api/message/*            → message/ (MessageController)
/api/send/*               → message/ (SendController) + chatbot/ (카카오 웹훅)
/api/ai/*                 → ai/ (AiController)
/api/shopify/admin/*      → shopify/ (ShopifyAdminController, ShopifyAdminManageController)
/api/open/shopify/*       → shopify/ (ShopifyController)
/api/open/keychron/*      → shopify/ (KeychronBoardController)
/api/event/*              → event/ (EventController)
/api/wms/*                → wms/ (WMSController)
/api/v1/finance/*         → finance/ (FinanceController)
/api/finance/*            → finance/ (FinanceController - 비버전 엔드포인트)
/api/open/*               → api/ (ExternalApiController)
/api/tpt/*                → chatbot 관련 (message/SendController)
```

## Architecture

### 각 도메인 패키지 내부 구조

```
{domain}/
├── controller/     → REST API 엔드포인트
├── service/        → 비즈니스 로직
│   └── facade/     → 크로스 도메인 조합 (선택적)
├── dao/            → MyBatis DAO (SqlSessionTemplate 사용)
├── dto/            → Request/Response DTO
│   ├── request/
│   └── response/
├── model/          → VO 클래스 (Lombok 미사용)
└── enums/          → 도메인별 enum
```

### 멀티 DataSource 구조

| Bean 접두사 | DB명 | 용도 |
|-------------|------|------|
| `tbnws2` (Primary) | TBNWS_ADMIN | 관리자 시스템 메인 |
| `gtgear` | gtgear | GTGear 쇼핑몰 |
| `wms` | WMS | 창고관리 |
| `ai` | AI_CHATBOT | AI 챗봇 |
| `sone` | attendance | 근태관리 |
| `tbnws` | tbnws | TBNWS 메인 |
| `yourls` | yourls | URL 단축 |
| `tpt` | TPT | TPT 서비스 |
| `shopify` | shopify | Shopify 연동 |

**DAO 사용 패턴:**
```java
public AdminDAO(
  @Qualifier("sqlSessionTemplate") SqlSessionTemplate sqlSessionTemplate,        // TBNWS_ADMIN
  @Qualifier("sqlSessionTemplateGTGEAR") SqlSessionTemplate sqlSessionTemplateGTGEAR  // gtgear
) { ... }
```

### MyBatis Mapper

```
src/main/resources/mapper/
├── erp/            → ERP 도메인 (24개 XML)
├── naver/          → 네이버 연동
├── shopify/        → Shopify 연동
└── *.xml           → 도메인별 (ai, order, finance, hr, wms, b2bcrm 등)
```

### 인증/세션

- Google OAuth2 → `config/security/CustomOAuth2UserService`
- `@IfLogin` 어노테이션 → `config/resolver/IfLoginArgumentResolver` → `SessionUser` 주입
- Redis 세션 (namespace: `tbnws_session`, 3일 유지)
- 쿠키명: `TBNWS_JSESSIONID`

### 로깅

- `@Logging(action = "...")` AOP 기반 관리자 액션 로깅 (`config/log/`)
- MDC 필터: `transactionId`, `clientIp`, `name`

### 모니터링

- Actuator 베이스: `/tbnwsSecret` (health, metrics, prometheus)

## Spring Profiles

| Profile | Port | 용도 | 스케줄러 |
|---------|------|------|----------|
| `dev` | 8080 | 로컬 개발 | OFF |
| `blue` | 8081 | 프로덕션 (Blue-Green) | ON |
| `green` | 8082 | 프로덕션 (Blue-Green) | ON |
| `staging` | 8080 | 스테이징 | OFF |
| `test` | 9000 | 테스트 | OFF |

## Testing

- JUnit 5 + Instancio + Testcontainers (MySQL 8.0 + Redis)
- 기반 클래스: `ServiceContext` (@ActiveProfiles("test"), @Transactional, @SpringBootTest)
- Docker 필수 (Testcontainers)
- Fixture 패턴: `src/test/java/.../fixture/`

## Deployment

- AWS CodeDeploy + Blue-Green (Nginx 프록시 전환)
- CI: GitHub Actions → Gradle → S3 (`s3://mugeonbucket/`) → CodeDeploy
- 버전 태그: `admin-back/vX.Y.Z`
- 헬스체크: `/tbnwsSecret/health`

## PR Convention

- PR 제목: `[TB-번호] 설명`
- 본문 템플릿: `.github/pull_request_template.md`
  - `## 📌 Summary` / `## 💬 Review Points` / `## 📎 References`

## Code Review Response Template

### 심각도 태그

- CRITICAL: 즉시 수정 필요 (보안, 데이터 손실)
- HIGH: 머지 전 수정 (버그, 트랜잭션 오류)
- MEDIUM: 개선 권장 (잠재적 문제)

### 이슈 형식

1. **[심각도] 한 줄 요약**
2. **파일**: `파일경로:라인번호`
3. **문제**: 1-2문장
4. **이유**: 영향/결과
5. **수정**: 구체적 코드 제시

### 규칙

- 한국어 리뷰, 최대 7개 이슈 (심각도 순), 각 이슈에 수정 코드 포함

### 이 프로젝트의 의도적 패턴 (리뷰 제외)

- VO에 Lombok 미사용
- DAO에 인터페이스 없이 구현 클래스만 사용
- ModelMapper로 DTO-VO 변환
- @IfLogin 커스텀 어노테이션으로 세션 사용자 주입

## Key External Integrations

| 서비스 | 용도 |
|--------|------|
| 네이버 커머스 API | 상품 동기화, 주문 관리 |
| Shopify Admin API | 주문, 상품, 재고 동기화 |
| POPBiLL | 전자세금계산서 |
| Aligo SMS / 카카오 알림톡 | 메시징 |
| OpenAI + AWS Bedrock + Spring AI | AI 기능 |
| Google APIs | Gmail, Calendar |
| Elasticsearch | 검색 |
| CJ/LG/LT 배송 API | 배송 추적 (Strategy 패턴) |
| Jenkins | 빌드 트리거 |
| 사방넷 | 마켓플레이스 주문 동기화 |
