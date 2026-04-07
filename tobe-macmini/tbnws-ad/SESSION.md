# 네이버 광고 분석 대시보드 - 세션 정보

> 최종 업데이트: 2026-02-23 (스마트스토어 통합)

## 프로젝트 개요

네이버 검색광고(SA) + 성과형 디스플레이(GFA) + 스마트스토어(Commerce) 통합 분석 대시보드.
키크론(Keychron) + 지티기어(GTGear) 2개 계정의 광고/매출 데이터를 수집/분석/시각화.

## 아키텍처

```
[브라우저] ──→ :8087 ──→ Flask (run.py → app/__init__.py)
                           │ Basic Auth: tobe / tobe1245
                           ├── app/routes/ (Blueprint 18개)
                           ├── app/static/ (ES Module SPA)
                           ├── MySQL (TBNWS_ADMIN DB)
                           ├── 네이버 SA API (키워드/광고그룹 조작)
                           ├── 네이버 DataLab API (검색 트렌드)
                           └── 네이버 Commerce API (스마트스토어 주문/상품/판매자)

[Cron] ── 매시간 ──→ naver_ad_cron.py (SA 광고 통계 수집)
       ── 매시 2분 ──→ smartstore_cron.py (스마트스토어 주문 수집)
       ── 매일 10시 ──→ naver_trend_cron.py (검색 트렌드 수집)
       ── 매일 4시 ──→ ebs_snapshot.sh (EBS 스냅샷 7일 보관)

[GFA] ── 수동 다운로드 (API 없음) ──→ gfa_upload_watcher.py → DB
```

## EC2 서버 정보

- **IP**: 13.125.219.231
- **SSH**: `sshpass -p 'ec2-user' ssh -o StrictHostKeyChecking=no -o PreferredAuthentications=password ec2-user@13.125.219.231`
- **작업 디렉토리**: `/home/ec2-user/naver-ad/`
- **배포 방식**: SCP (git 미설치, 파일 직접 복사)

### 실행 중인 서비스

| 포트 | 프로세스 | 용도 |
|------|---------|------|
| 8087 | `python3 run.py` (Flask) | API + SPA 서빙 (Basic Auth) |

### 서비스 재시작

```bash
SSH_OPTS="-o StrictHostKeyChecking=no -o PreferredAuthentications=password"
EC2="ec2-user@13.125.219.231"

# Flask 재시작
sshpass -p 'ec2-user' ssh $SSH_OPTS $EC2 "pkill -f 'python3 run.py' 2>/dev/null; echo killed"
sshpass -p 'ec2-user' ssh $SSH_OPTS $EC2 "cd /home/ec2-user/naver-ad && nohup python3 run.py >> api.log 2>&1 & echo started"

# 헬스체크
sleep 4 && sshpass -p 'ec2-user' ssh $SSH_OPTS $EC2 "curl -s -u tobe:tobe1245 http://localhost:8087/api/health"
```

### 배포 명령 (SCP)

```bash
SSH_OPTS="-o StrictHostKeyChecking=no -o PreferredAuthentications=password"
EC2="ec2-user@13.125.219.231"

# 프론트엔드만 (Flask 재시작 불필요)
sshpass -p 'ec2-user' scp $SSH_OPTS app/static/index.html $EC2:/home/ec2-user/naver-ad/app/static/index.html
sshpass -p 'ec2-user' scp $SSH_OPTS app/static/js/tabs/파일.js $EC2:/home/ec2-user/naver-ad/app/static/js/tabs/

# 백엔드 (Flask 재시작 필요)
sshpass -p 'ec2-user' scp $SSH_OPTS app/routes/파일.py $EC2:/home/ec2-user/naver-ad/app/routes/
```

### Crontab

```
0 * * * *  cd /home/ec2-user/naver-ad && /usr/bin/python3 naver_ad_cron.py >> cron.log 2>&1
2 * * * *  cd /home/ec2-user/naver-ad && /usr/bin/python3 scripts/smartstore_cron.py >> smartstore_cron.log 2>&1
0 10 * * * cd /home/ec2-user/naver-ad && /usr/bin/python3 naver_trend_cron.py >> cron.log 2>&1
0 10 * * * cd /home/ec2-user/naver-ad && /usr/bin/python3 telegram_ad_report.py --from-db >> telegram_report.log 2>&1
0 6 * * *  /home/ec2-user/cleanup.sh >> /home/ec2-user/cleanup.log 2>&1
0 4 * * *  /home/ec2-user/naver-ad/scripts/ebs_snapshot.sh >> /home/ec2-user/naver-ad/ebs_snapshot.log 2>&1
```

## 대시보드 URL

| URL | 설명 |
|-----|------|
| `http://13.125.219.231:8087/` | 통합 대시보드 (Basic Auth) |
| `http://13.125.219.231:8087/?tab=gfa-manual` | GFA 수동 데이터 처리 |
| `http://13.125.219.231:8087/api/health` | API 헬스체크 (인증 면제) |

## 인증 정보

### Basic Auth (대시보드/API)
- **ID**: tobe
- **PW**: tobe1245

### 네이버 검색광고 API (SA)
| 계정 | CUSTOMER_ID | API_KEY | SECRET_KEY |
|------|-------------|---------|------------|
| 키크론 | 3544598 | 0100000000b686d4a0f8cdd33bfde7a803758a4798787903566c774bf6d322f5e7b207ee77 | AQAAAAC2htSg+M3TO/3nqAN1ikeYXTfJS/VsuxOklvpZLwyfdQ== |
| 지티기어 | 901880 | 0100000000c4dfabc24864265711cb997c819e9c409aa825e967b2285e4f70ccee185187d8 | AQAAAADE36vCSGQmVxHLmXyBnpxAsntirNnKeXtNlZjSf8znXQ== |

### 네이버 DataLab API
- **Client ID**: W918q0tkXCSGW2MWq5NE
- **Client Secret**: yhgsPtNzMD

### DB 접속
- **MySQL**: 222.122.42.221:3306 / tbnws22 / dnpfzja! / TBNWS_ADMIN

### GFA (성과형 디스플레이)
- **계정**: 키크론 (Account ID: 26343)
- **로그인**: gfa.naver.com (네이버 계정: demigodsj)
- **API 없음** — CSV 다운로드만 가능

### 네이버 스마트스토어 (Commerce API)
| 계정 | CLIENT_ID | CLIENT_SECRET |
|------|-----------|---------------|
| 키크론 | 4tFLbZlk2DNkWL6BmmCDhy | $2a$04$GyB2FTt0XYFJpb8rVmWeve |
| 지티기어 | 56fryAMxkJDOCnXYxXcQmz | $2a$04$aO/kvEZxjghNtew5N3EXMu |

- **Base URL**: `https://api.commerce.naver.com/external`
- **인증**: OAuth 2.0 (bcrypt 서명)
- **허용 IP**: 112.157.162.131, 3.34.119.173, 13.125.219.231
- **사용 가능 API**: 주문(/v1/pay-order), 상품(/v1/products), 판매자(/v1/seller)
- **사용 불가 API (404)**: 문의, 정산, 판매자 주소록

## 파일 구조

```
tbnws-ad/
├── run.py                          # 엔트리포인트
├── requirements.txt
├── SESSION.md                      # 이 파일
├── GFA_PENDING_DOWNLOADS.md        # GFA 미완료 다운로드 기록
│
├── app/
│   ├── __init__.py                 # create_app() 팩토리
│   ├── config.py                   # DB/SA/Telegram/DataLab 설정
│   ├── auth.py                     # Basic Auth 미들웨어
│   ├── db.py                       # get_conn, fix_types, api_error
│   ├── sa_client.py                # 네이버 SA API 클라이언트
│   ├── scoring.py                  # 파워링크 스코어링 엔진
│   ├── funnel_engine.py            # GFA 퍼널 분석 엔진 (~1850줄)
│   │   #  FunnelClassifier: 4-signal 확률적 분류
│   │   #  BetaBinomialEstimator: 베이지안 CTR/CVR
│   │   #  HillSaturation: 예산 vs 전환 체감곡선
│   │   #  AdstockDecay: 광고 이월효과 감쇠
│   │   #  BudgetOptimizer: 등한계 원칙 예산 재배분
│   │   #  FunnelHealthScorer: 퍼널 건강도 100점
│   │   #  ActionItemGenerator: 액션아이템 자동 생성
│   │   #  CampaignMatrixScorer: 캠페인 종합 점수
│   │
│   ├── smartstore_client.py         # 네이버 Commerce API 클라이언트 (OAuth 2.0 + bcrypt)
│   │
│   ├── routes/                     # Flask Blueprints (18개)
│   │   ├── health.py               # /api/health
│   │   ├── bizmoney.py             # /api/bizmoney
│   │   ├── campaigns.py            # /api/campaigns
│   │   ├── adgroups.py             # /api/adgroups
│   │   ├── keywords.py             # /api/keywords
│   │   ├── daily.py                # /api/daily, daily-compare
│   │   ├── overview.py             # /api/overview (+ SA 비효율 드릴다운)
│   │   ├── scoring_routes.py       # /api/scoring
│   │   ├── diagnosis.py            # /api/diagnosis (액션아이템 + 요일패턴분석)
│   │   ├── trends.py               # /api/trends
│   │   ├── actions.py              # /api/action/execute, logs
│   │   ├── telegram.py             # /api/telegram/*
│   │   ├── gfa.py                  # /api/gfa/* (대시보드/DOW히트맵/비효율드릴다운/누락진단/업로드)
│   │   ├── funnel.py               # /api/funnel/* (퍼널분석/adstock)
│   │   ├── budget.py               # /api/budget/*
│   │   ├── strategy_config.py      # /api/strategy/*
│   │   ├── smartstore.py           # /api/smartstore/* (주문/상품/판매자/정산)
│   │   └── static_routes.py        # / SPA 서빙
│   │
│   └── static/                     # 프론트엔드 (Vanilla JS + ES Modules)
│       ├── index.html              # HTML 쉘
│       ├── styles/main.css
│       └── js/
│           ├── config.js           # API_BASE, COLORS
│           ├── utils.js            # fmt, apiFetch, destroyChart
│           ├── router.js           # URL 쿼리파라미터 라우팅
│           ├── debug.js            # ?debug=1 디버그 패널
│           ├── app.js              # 탭 오케스트레이터
│           ├── components/
│           │   ├── table-utils.js  # 테이블정렬, ON/OFF토글, 입찰편집
│           │   └── funnel-chart.js # 퍼널 차트 컴포넌트
│           └── tabs/               # 탭별 독립 모듈
│               ├── overview.js
│               ├── dailycompare.js
│               ├── campaigns.js
│               ├── adgroups.js
│               ├── keywords.js
│               ├── trends.js
│               ├── combined.js
│               ├── scoring.js
│               ├── actions.js
│               ├── logs.js
│               ├── strategy.js
│               ├── telegram.js
│               ├── gfa.js          # GFA 대시보드 (KPI/DOW히트맵/비효율드릴다운)
│               ├── gfa-manual.js   # GFA 수동 데이터 처리
│               ├── funnel.js       # GFA 퍼널 분석
│               ├── batch-history.js # 배치 수집 이력
│               ├── smartstore.js   # 스마트스토어 대시보드
│               ├── ss-orders.js    # 주문분석
│               ├── ss-products.js  # 상품분석
│               ├── ss-inquiries.js # 문의분석 (클레임 기반)
│               ├── ss-seller.js    # 판매자정보
│               └── ss-settlement.js # 정산분석
│
├── scripts/
│   ├── naver_ad_cron.py            # SA 데이터 수집 크론
│   ├── naver_trend_cron.py         # 트렌드 수집 크론
│   ├── smartstore_cron.py          # 스마트스토어 주문 수집 크론
│   ├── ebs_snapshot.sh             # EBS 스냅샷 자동화 (7일 보관)
│   ├── telegram_ad_report.py       # 텔레그램 리포트 발송
│   └── gfa_upload_watcher.py       # GFA ZIP 자동 업로드
│
└── prototypes/                     # 원본 아카이브
```

## 대시보드 탭 구성

### SA 검색광고
| 탭 | 설명 | API |
|----|------|-----|
| 📊 개요 | KPI 카드 + 일별 차트 + 캠페인 파이 + SA 비효율 드릴다운 | `/api/overview` |
| 📅 일별비교 | 기간 대비 일별 비교 (요일 표시) | `/api/daily-compare` |
| 🎯 캠페인 | 캠페인별 테이블 + 차트 | `/api/campaigns/enhanced` |
| 📦 광고그룹 | TOP 10 + ROAS 분포 + ON/OFF 토글 | `/api/adgroups/enhanced` |
| 🔑 키워드 | TOP 15 + 스캐터 + 테이블 | `/api/keywords/enhanced` |
| 📈 검색트렌드 | 트렌드 라인 (앵커 정규화 v2) + 전체선택 토글 | `/api/trends` |
| 🔗 통합분석 | 광고-트렌드 상관관계 | `/api/combined` |
| ⚡ 스코어링 | 파워링크 입찰 스코어링 | `/api/scoring` |

### GFA 디스플레이
| 탭 | 설명 | API |
|----|------|-----|
| 📊 대시보드 | GFA KPI 12개 + DOW 히트맵 + 비효율 드릴다운 (캠페인→광고그룹→소재) | `/api/gfa/dashboard`, `/api/gfa/dow-heatmap`, `/api/gfa/waste-drilldown` |
| 🎯 퍼널 분석 | 노출→클릭→전환 퍼널 + 베이지안 분석 + Adstock + VT보정 | `/api/funnel/*` |
| 📥 수동 데이터 처리 | 누락 진단 + GFA 다운로드 요청/다운로드/업로드 | `/api/gfa/missing-dates`, `/api/gfa/upload` |

### 스마트스토어
| 탭 | 설명 | API |
|----|------|-----|
| 📊 대시보드 | KPI + 매출트렌드 + 상품비중 + 광고-매출 연계 | `/api/smartstore/overview`, `/api/smartstore/daily` |
| 📋 주문분석 | 상태분포 + 시간대/요일분석 + 가격대분포 + 클레임 | `/api/smartstore/orders/analysis` |
| 📦 상품분석 | TOP15 + 파레토 + 카탈로그(API) | `/api/smartstore/products/analysis`, `/api/smartstore/products/catalog` |
| 💬 문의분석 | 클레임 유형 + 일별추이 + 상품별 클레임률 | `/api/smartstore/claims/analysis` |
| 🏪 판매자정보 | 셀러정보(API) + 채널정보 + DB통계 | `/api/smartstore/seller-info` |
| 💵 정산분석 | 예상 정산 (수수료 5.5%) + 일별/월별 정산 | `/api/smartstore/settlement/analysis` |

### 인사이트
| 탭 | 설명 | API |
|----|------|-----|
| 🚨 액션 아이템 | 자동 진단 (요일패턴 포함) + 실행 버튼 | `/api/diagnosis`, `/api/action/execute` |
| 📋 실행 로그 | 액션 실행 이력 | `/api/action/logs` |
| 📊 배치 이력 | 크론 배치 수집 이력 조회 | `/api/batch-history` |

### 설정
| 탭 | 설명 | API |
|----|------|-----|
| ⚙️ 전략설정 | 스코어링 전략 가중치 설정 | `/api/strategy/*` |
| 🔔 알림설정 | 텔레그램 푸시 알림 설정 | `/api/telegram/*` |

## DB 테이블

### SA (검색광고) 테이블
| 테이블 | UK | 주요 컬럼 |
|--------|-----|----------|
| naver_ad_campaign_daily | account, campaign_id, stat_date | imp_cnt, clk_cnt, sales_amt, ccnt, conv_amt, ctr, cpc, ror |
| naver_ad_adgroup_daily | account, adgroup_id, stat_date | + bid_amt |
| naver_ad_keyword_daily | account, keyword_id, stat_date | + keyword, adgroup_name |
| naver_ad_bizmoney | - | account, bizmoney, collected_at |
| naver_trend_daily | group_name, stat_date | ratio, keywords |
| naver_ad_action_log | seq (PK, auto) | action_type, status, result_msg, executed_at |

### GFA (디스플레이) 테이블
| 테이블 | UK | 주요 컬럼 |
|--------|-----|----------|
| naver_ad_gfa_campaign_daily | account, campaign_id, stat_date | impressions, clicks, cost, conversions, revenue, roas |
| naver_ad_gfa_adgroup_daily | account, adgroup_id, stat_date | + reach, frequency |
| naver_ad_gfa_creative_daily | account, creative_id, stat_date | + creative_name |

### 스마트스토어 테이블
| 테이블 | UK | 주요 컬럼 |
|--------|-----|----------|
| naver_smartstore_orders | product_order_id | account, order_id, product_no, product_name, quantity, total_price, product_order_status, order_date, claim_type |
| naver_smartstore_order_daily | account, stat_date | order_cnt, paid_cnt, cancel_cnt, return_cnt, revenue, cancel_amt, avg_order_value |
| naver_smartstore_product_daily | account, stat_date, product_no | order_cnt, quantity, revenue, cancel_cnt, cancel_amt |

### 스마트스토어 데이터 현황 (2026-02-23 기준)
| 계정 | 주문 건수 | 데이터 범위 | 비고 |
|------|----------|------------|------|
| 키크론 | 764건 | 2023-12-14 ~ 2026-01-24 | 1/24 이후 신규주문 없음 |
| 지티기어 | 67건 | 2023-12-21 ~ 2026-02-13 | |

> Commerce API lastChangedStatuses는 최근 90일 상태변경만 조회 가능. 커서 기반 페이징 필수(moreFrom/moreSequence).

### GFA 데이터 현황 (2026-02-22 기준)
| 분석 단위 | 건수 | 데이터 범위 |
|-----------|------|------------|
| 캠페인 | 5,620 | 2024-03-04 ~ 2026-02-22 |
| 광고그룹 | 1,345 | 2024-10-18 ~ 2026-02-22 |
| 소재 | 28,775 | 2024-03-04 ~ 2026-02-22 |

> 광고그룹 과거 데이터(2024-03-04~2024-10-17) 미수집 — GFA_PENDING_DOWNLOADS.md 참조

## GFA 수동 데이터 수집 워크플로우

GFA는 API가 없어서 수동 다운로드가 필요합니다.

### 방법 1: 대시보드 UI
1. `http://13.125.219.231:8087/?tab=gfa-manual` 접속
2. "데이터 현황"에서 누락 확인
3. "요청하기" 버튼 클릭 → GFA 새 창 열림
4. GFA에서 "다운로드 요청" 클릭
5. "다운로드 목록 열기" → ZIP 다운로드
6. "업로드" 폼에서 ZIP 업로드

### 방법 2: CLI 스크립트
```bash
# 누락 확인
python3 scripts/gfa_upload_watcher.py --check

# Downloads 폴더 ZIP 자동 업로드
python3 scripts/gfa_upload_watcher.py

# 실시간 감시 (파일 떨어지면 즉시 업로드)
python3 scripts/gfa_upload_watcher.py --watch
```

## 개발 시 주의사항

1. **DB 컬럼명**: SA는 `imp_cnt, clk_cnt, sales_amt, ccnt`, GFA는 `impressions, clicks, cost, conversions`
2. **DictCursor**: `get_conn()`이 DictCursor 반환. `row['컬럼명']` 사용 (row[0] 불가)
3. **API_AUTH**: `config.js`에 `btoa('tobe:tobe1245')` 하드코딩
4. **ES Modules**: 프론트엔드는 빌드 도구 없이 ES Module import/export 사용
5. **EC2 SSH**: `PreferredAuthentications=password` 옵션 필수 (키 인증 불가)
6. **EC2 배포**: git 미설치 → SCP로 파일 복사. 정적 파일은 재시작 불필요, 백엔드는 재시작 필요
7. **GFA 자동화**: `navigate` 도구 사용 불가 (보안 차단). `javascript_tool`로 `window.location.href` 사용
8. **GFA 확인 버튼**: `.ant-modal-footer` 셀렉터 사용 불가. 좌표 클릭 또는 `.ant-btn-primary.ant-btn-lg` 사용
9. **GFA View-through 어트리뷰션 보정**: ROAS를 단계별 할인 (`TOF×0.3, MOF×0.5, BOF×0.8`)
10. **CSS 특이성 주의**: 미디어쿼리는 특이성을 높이지 않음. `.parent .child` (0,2,0) > `@media .child` (0,1,0)
11. **Flex 퍼센트 + gap**: `flex:0 0 45%` + `flex:0 0 55%` + `gap:16px` = 오버플로우. `flex:45 1 0`/`flex:55 1 0` 사용

## 세션 로그

| 날짜 | 파일 | 주요 내용 |
|------|------|----------|
| 2026-02-21 | `docs/session-2026-02-21-gfa-sidebar.md` | GFA 대시보드 구축, 사이드바 트리 네비게이션, 예산 예측 |
| 2026-02-22~23 | `docs/session-2026-02-22-funnel-optimization.md` | 퍼널 엔진 교정, 모바일 반응형, To-Be 설계 |
| 2026-02-23 | `docs/session-2026-02-23-smartstore.md` | 스마트스토어 6탭 구축, Commerce API 연동, EBS 스냅샷 자동화, 트렌드 Y축 개선 |
