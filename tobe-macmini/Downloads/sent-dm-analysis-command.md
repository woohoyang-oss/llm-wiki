# Claude Code 명령: sent.dm 완전 분석 및 개발서버 재현

아래 명령을 Claude Code에 그대로 복붙해서 사용하세요.

---

## 명령 (복붙용)

```
너는 지금부터 sent.dm (https://www.sent.dm/) 사이트의 기능 분석가이자 풀스택 개발자야.

## 미션
sent.dm 웹사이트와 API 문서(https://docs.sent.dm/)를 구석구석 분석해서, 동일한 기능을 가진 서비스를 우리 개발서버에 재현해야 해. 분석 → 설계 → 구현 → 테스트까지 전부 진행해.

## Phase 1: 사이트 기능 분석 (먼저 이것부터 완료해)

### 1-1. 랜딩 페이지 분석
sent.dm 메인 페이지를 방문해서 다음을 분석하고 문서화해:
- Hero 섹션: 카피, CTA 버튼, 코드 프리뷰 (cURL/Python/Node.js 탭 전환)
- 지원 채널 섹션: SMS, WhatsApp, iMessage, RCS 등 어떤 채널을 보여주는지
- Features 섹션: 각 기능 카드의 제목, 설명, 아이콘
- How it Works 섹션: 단계별 플로우
- Pricing 섹션: 가격 구조, 티어, 연락처당 과금 방식
- Developer 섹션: SDK 지원 언어, 코드 예시
- FAQ 섹션: 질문/답변 목록
- Footer: 링크 구조
- 전체적인 UI/UX 패턴: 다크테마, 애니메이션, 스크롤 인터랙션

### 1-2. API 문서 분석
https://docs.sent.dm/ 의 모든 페이지를 순회하면서 분석해:

**Tutorials:**
- Getting Started: 계정 생성 → KYC 인증 → API 키 발급 → 첫 메시지 발송 플로우
- Creating Templates: 템플릿 생성 프로세스, WhatsApp 승인 절차
- Dashboard: 대시보드 기능 목록

**Features & Guides:**
- Messages: 메시지 발송 API (POST /v2/messages/contact, POST /v2/messages/phone), 메시지 상태 라이프사이클 (queued → sent → delivered / failed), 자동 채널 선택 로직, Dashboard Playground
- Contacts: 연락처 관리 API (GET paginated, GET by ID, GET by phone), 연락처 객체 구조
- Templates: 템플릿 CRUD API (POST create, GET list, GET by ID, DELETE), 동적 변수, 카테고리(UTILITY/AUTHENTICATION/MARKETING)

**Concepts:**
- Platform Overview: 아키텍처 다이어그램, 핵심 컴포넌트
- Unified Messaging Intelligence: AI 채널 라우팅 로직
- Channels: 지원 채널별 특성 (SMS, WhatsApp, iMessage, RCS)
- Contacts/Templates 개념 설명

**Webhooks:**
- Setup: 웹훅 엔드포인트 등록
- Security: HMAC-SHA256 서명 검증
- Development & Debugging: 로컬 테스트, 재시도 로직, 멱등성
- Events Reference: message.status 이벤트, template.status 이벤트, 이벤트 페이로드 구조

**API Reference:**
- 전체 엔드포인트 목록, 인증 방식(x-sender-id, x-api-key), 요청/응답 스키마

### 1-3. 분석 결과 저장
분석 결과를 `docs/sent-dm-analysis.md` 파일로 저장해. 구조:
```markdown
# Sent.dm 완전 분석 보고서

## 1. 랜딩 페이지 구조
### 1.1 섹션별 분석
### 1.2 UI/UX 패턴
### 1.3 반응형 브레이크포인트

## 2. API 엔드포인트 전체 목록
### 2.1 Messages API
### 2.2 Contacts API
### 2.3 Templates API
### 2.4 Webhooks

## 3. 핵심 비즈니스 로직
### 3.1 채널 라우팅 알고리즘
### 3.2 폴백 시스템
### 3.3 메시지 상태 머신
### 3.4 비용 계산 로직

## 4. 대시보드 기능 목록
### 4.1 Playground (메시지 테스트)
### 4.2 연락처 관리
### 4.3 템플릿 관리
### 4.4 웹훅 설정
### 4.5 분석/통계

## 5. 재현 시 필수 구현 항목 (우선순위별)
```

---

## Phase 2: 기술 스택 결정 및 프로젝트 셋업

분석이 완료되면 다음 스택으로 프로젝트를 셋업해:

### 프론트엔드 (랜딩 + 대시보드)
- Next.js 14+ (App Router)
- TypeScript
- Tailwind CSS
- Framer Motion (애니메이션)
- shadcn/ui (대시보드 컴포넌트)

### 백엔드 (API 서버)
- Next.js API Routes 또는 별도 Express/Fastify 서버
- TypeScript
- Prisma (ORM)
- PostgreSQL (DB)
- Redis (큐잉, 캐시)

### 프로젝트 구조
```
project/
├── apps/
│   ├── web/                 # 랜딩 페이지
│   ├── dashboard/           # 대시보드 앱
│   └── api/                 # API 서버
├── packages/
│   ├── db/                  # Prisma 스키마 & 클라이언트
│   ├── types/               # 공유 타입 정의
│   └── utils/               # 공유 유틸리티
├── docs/
│   ├── sent-dm-analysis.md  # 분석 보고서
│   └── api-spec.md          # 우리 API 스펙
└── docker-compose.yml       # PostgreSQL + Redis
```

---

## Phase 3: 핵심 기능 구현 (우선순위 순)

### P0 - 반드시 구현
1. **랜딩 페이지**: sent.dm과 동일한 섹션 구조, 다크 테마, 코드 프리뷰
2. **인증 시스템**: API Key 발급 (x-sender-id, x-api-key)
3. **Messages API**:
   - `POST /v2/messages/contact` - contactId로 메시지 발송
   - `POST /v2/messages/phone` - 전화번호로 메시지 발송 (자동 연락처 생성)
   - 메시지 상태 관리: queued → sent → delivered / failed
   - 자동 채널 선택 (채널 미지정 시 WhatsApp 우선 → SMS 폴백)
4. **Contacts API**:
   - `GET /v2/contacts` - 페이지네이션 연락처 목록
   - `GET /v2/contacts/:id` - ID로 조회
   - `GET /v2/contacts/phone/:number` - 전화번호로 조회
5. **Templates API**:
   - `POST /v2/templates` - 템플릿 생성
   - `GET /v2/templates` - 목록 조회
   - `GET /v2/templates/:id` - 상세 조회
   - `DELETE /v2/templates/:id` - 삭제

### P1 - 필수이지만 P0 이후
6. **Webhooks 시스템**:
   - 웹훅 엔드포인트 등록/관리
   - HMAC-SHA256 서명 생성
   - 이벤트 발송 (message.status, template.status)
   - 재시도 로직 (지수 백오프)
7. **Dashboard**:
   - Playground (메시지 테스트 발송)
   - 연락처 관리 UI
   - 템플릿 관리 UI (시각적 빌더)
   - API Keys 관리
8. **채널 라우팅 엔진**: 
   - 연락처별 채널 가용성 감지
   - 비용 최적화 라우팅
   - 폴백 체인 설정

### P2 - 있으면 좋은 것
9. 실시간 분석 대시보드
10. SDK 생성 (TypeScript, Python)
11. 가격 계산기
12. KYC 인증 플로우

---

## Phase 4: 테스트

### 4-1. API 테스트 (필수)
각 엔드포인트별 테스트를 작성해:
```
tests/
├── api/
│   ├── messages.test.ts      # 메시지 발송/조회 테스트
│   ├── contacts.test.ts      # 연락처 CRUD 테스트
│   ├── templates.test.ts     # 템플릿 CRUD 테스트
│   ├── webhooks.test.ts      # 웹훅 발송/서명검증 테스트
│   └── auth.test.ts          # API 키 인증 테스트
├── integration/
│   ├── message-flow.test.ts  # 메시지 발송 → 상태변경 → 웹훅 전체 플로우
│   ├── channel-routing.test.ts # 채널 자동선택 & 폴백 테스트
│   └── template-approval.test.ts # 템플릿 생성 → 승인 플로우
└── e2e/
    ├── landing.test.ts       # 랜딩 페이지 렌더링 테스트
    └── dashboard.test.ts     # 대시보드 주요 기능 테스트
```

### 4-2. 테스트 시나리오 (반드시 커버)
1. **메시지 발송 성공**: 유효한 contactId로 메시지 발송 → 200 + messageId 반환
2. **전화번호 발송**: 신규 번호로 발송 → 자동 연락처 생성 + 메시지 발송
3. **채널 자동선택**: 채널 미지정 → WhatsApp 가용 시 WhatsApp, 불가 시 SMS
4. **폴백 동작**: WhatsApp 실패 → SMS로 자동 전환 → 웹훅 이벤트 발송
5. **인증 실패**: 잘못된 API Key → 401 Unauthorized
6. **웹훅 서명 검증**: 올바른/잘못된 HMAC 서명으로 검증 테스트
7. **템플릿 변수 치환**: 동적 필드 {{customerName}} 등이 올바르게 치환되는지
8. **연락처 페이지네이션**: 대량 데이터에서 cursor 기반 페이지네이션 동작
9. **Rate Limiting**: API 요청 제한 초과 시 429 반환
10. **메시지 상태 머신**: queued → sent → delivered 정상 전이 확인

### 4-3. 테스트 실행
```bash
# 유닛 테스트
npm run test

# 통합 테스트 (DB 필요)
npm run test:integration

# E2E 테스트
npm run test:e2e

# 전체 테스트 + 커버리지
npm run test:coverage
```

---

## 진행 규칙

1. **Phase 1을 완료한 후** Phase 2로 넘어가. 분석 없이 코드부터 쓰지 마.
2. 각 Phase 완료 시 진행상황을 알려줘.
3. 구현 중 sent.dm 원본과 다르게 가야 할 부분이 있으면 미리 알려줘.
4. 모든 API는 sent.dm과 동일한 엔드포인트 경로와 요청/응답 형식을 따라.
5. 한국 시장 특화 기능(카카오톡 알림톡/친구톡)은 별도 채널로 추가해.
6. 테스트는 반드시 통과하는 상태에서 커밋해.
7. 에러 핸들링은 sent.dm API 문서의 에러 코드 체계를 따라.

지금 바로 Phase 1 분석부터 시작해.
```

---

## 사용법

1. Claude Code 터미널에서 기존 프로젝트 디렉토리로 이동
2. `claude` 입력해서 Claude Code 시작
3. 위 명령 전체를 복붙
4. Phase 1 분석이 끝나면 결과를 확인하고 Phase 2로 진행 허용
5. 필요시 중간에 "카카오톡 채널 우선순위를 높여줘" 같은 추가 지시 가능
