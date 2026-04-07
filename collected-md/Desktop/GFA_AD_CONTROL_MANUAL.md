# GFA 광고 컨트롤 전체 매뉴얼 (2026-03-10)

---

## 1. EC2 접속

- SSH 키: `~/Desktop/claude.pem`
- IP: `100.103.222.7` (Tailscale 필수, 공인IP `3.37.188.23` 사용 금지)
- 반드시 `SSH_AUTH_SOCK=""` + `-o IdentitiesOnly=yes` 사용

```bash
# SSH
SSH_AUTH_SOCK="" ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i ~/Desktop/claude.pem ec2-user@100.103.222.7

# SCP (파일 업로드)
SSH_AUTH_SOCK="" scp -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i ~/Desktop/claude.pem <로컬파일> ec2-user@100.103.222.7:/home/ec2-user/naver-ad/<파일>
```

- systemd 서비스: tbnws-ad(8087), sabang-dashboard(5050), tbnws-data-api(8100), tobe-ai-v2(3100)
- **서버 재시작은 사용자 허락 없이 절대 금지**
- **절대 `pkill -f python` 사용 금지** — 다른 서비스 죽일 수 있음

---

## 2. 광고 컨트롤 실행 방법

EC2 서버의 `OpsGfaApi`를 통해 네이버 GFA 광고를 직접 제어할 수 있다.

### 실행 패턴
```bash
# 1) 로컬에 스크립트 작성
cat > /tmp/gfa_ops.py << 'PYEOF'
import sys, os
sys.path.insert(0, '/home/ec2-user/naver-ad')
os.chdir('/home/ec2-user/naver-ad')
from app.gfa_ops.ops_state import OpsGfaApi

api = OpsGfaApi()  # 자동으로 세션 로드 + warm_up(XSRF 갱신)

# 예산 변경
result = api.change_budget(4100170, 50000)
# → (True, 50000.0, 50000.0, [adset_detail_dict])

# ON/OFF
result = api.toggle_adset(3519687, False)  # OFF
result = api.toggle_adset(3519687, True)   # ON
# → (True, False) 또는 (True, True)

print(result)
PYEOF

# 2) SCP 업로드
SSH_AUTH_SOCK="" scp -o StrictHostKeyChecking=no -o IdentitiesOnly=yes \
  -i ~/Desktop/claude.pem /tmp/gfa_ops.py \
  ec2-user@100.103.222.7:/home/ec2-user/naver-ad/gfa_ops.py

# 3) SSH 실행
SSH_AUTH_SOCK="" ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes \
  -i ~/Desktop/claude.pem ec2-user@100.103.222.7 \
  "cd /home/ec2-user/naver-ad && python3 gfa_ops.py 2>&1"

# 4) 정리
SSH_AUTH_SOCK="" ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes \
  -i ~/Desktop/claude.pem ec2-user@100.103.222.7 \
  "rm -f /home/ec2-user/naver-ad/gfa_ops.py"
```

### 주의사항
- `OpsGfaApi()` 초기화 시 자동으로 세션 로드 + warm_up (XSRF 갱신)
- 세션 만료 시 Playwright 로그인 필요 → tbe.kr 대시보드에서 수동 갱신
- `change_budget()`의 금액은 자동으로 `round_budget()` 처리됨
- 스케줄러가 30분마다 자동 조정 → 수동 변경 후 되돌릴 수 있으니 주의

---

## 3. GFA API 엔드포인트 (계정 26343)

### 예산 변경
```
POST /apis/gfa/v1/adAccounts/{accountNo}/adSets/edit
  ?adSets[0].no={adSetNo}&adSets[0].budgetAmount={amount}
```
- Spring MVC `@ModelAttribute` 배열 파라미터 형식
- 캠페인 레벨 edit 없음 — 모든 제어는 adSet 레벨

### ON/OFF
```
POST /apis/gfa/v1/adAccounts/{accountNo}/adSets/activation
  ?activated={true|false}&adSetNo={adSetNo}
```
- ⚠️ `adSets/edit`의 `activated` 파라미터는 작동 안 함! 반드시 이 엔드포인트 사용
- 단건 처리 (배치 미지원)

### 조회
- 캠페인 목록: `GET /apis/gfa/v1/adAccounts/{acct}/campaigns?activated=true`
- 캠페인 상세: `GET /apis/gfa/v1.1/adAccounts/{acct}/campaigns/{campaignNo}` (v1.1!)
- adSet 목록: `GET /apis/gfa/v1.2/adAccounts/{acct}/adSets?campaignNo={no}&page=0&size=200` (v1.2!)
- adSet 상세: `GET /apis/gfa/v1/adAccounts/{acct}/adSets/{adSetNo}`
- 통계: `GET /apis/stats/v1/adAccounts/{acct}/stats/adSetStats?adSetNoList={no}&startDate=...&endDate=...`
- 세션 테스트: `GET /apis/gfa/v1/adAccounts/{acct}/authorities`

---

## 4. 현재 캠페인 구조 (2026-03-10 기준)

| Campaign | CampaignNo | 목적 | adSet(No) | 일예산 | 상태 |
|---|---|---|---|---|---|
| BOF 0305 장바구니 전환 | 1268005 | CONVERSION | 4100170 | ₩50,000 | ON (3/10 증액 20K→50K) |
| BOF 웹사이트 전환 #1 | 1162692 | CONVERSION | 1개 | ₩200,000 | ON |
| MOF 0225 SE | 1016343 | TRAFFIC | 3519687 | ₩44,000 | **OFF** (3/10 비활성화) |
| TOF 0113 | 1209023 | TRAFFIC | 3961952(60K), 3965735(45K) | ₩105,000 | ON, 감축 검토중 |
| TOF 0223 | 1255832 | TRAFFIC | 2개 | ₩90,000 | ON |
| TOF 0530 ADVoost | 932248 | PMAX | 0개 | ₩0 | - |
| TOF 1127 전환 | 1156362 | CONVERSION | 1개 | ₩80,000 | ON |
| TOF 1211 | 1175008 | TRAFFIC | 3883304(OFF), 3883324(10K), 3883414(20K), 3901558(10K) | ₩50,000 | 증액 검토중 |

### 미완료 작업: 0113→1211 예산 이동
사용자 요청: "0113 10만→6만 감축, 1211에 3만 추가"
- 0113: 3961952 60K→30K, 3965735 45K→30K
- 1211: 3901558 10K→30K (ROAS 24,000~52,000%), 3883324 10K→20K
- ⚠️ 1211의 AD부스트(3883414, 20K)는 ROAS 0% 지속 — 유지 또는 감축 논의 필요

---

## 5. adset 조회 (OPS AI SQLite)

DB 위치: EC2 `/home/ec2-user/naver-ad/app/gfa_ops/data/ops.db`

### adset_snapshots 스키마 (⚠️ 컬럼명 주의!)
```
id, ts, adset_no, adset_name, campaign_no, campaign_name, funnel,
activated (NOT enabled! NOT is_active!),
budget, spent_ratio, status,
impressions, clicks, cost, conversions, purchases, add_to_cart,
conv_revenue, purchase_revenue, cart_revenue,
ctr, cpc, cpm, roas, adj_roas, cpa
```

### 최신 스냅샷 조회
```bash
SSH_AUTH_SOCK="" ssh ... "cd /home/ec2-user/naver-ad && python3 -c \"
import sqlite3
conn = sqlite3.connect('app/gfa_ops/data/ops.db')
rows = conn.execute('''
    SELECT adset_no, adset_name, campaign_name, budget, activated, cost, roas
    FROM adset_snapshots
    WHERE ts = (SELECT MAX(ts) FROM adset_snapshots)
    ORDER BY campaign_name
''').fetchall()
for r in rows:
    print(f'{r[2]} | adset:{r[0]} | budget:{r[3]:,} | on:{r[4]} | cost:{r[5]:,} | roas:{r[6]}%')
conn.close()
\""
```

---

## 6. MySQL 성과 조회

### DB 접속 정보
- 외부 MySQL: `222.122.42.221:3306`
- TBNWS_ADMIN: user=`dev`, pass=`@Tbnws1346` (daily 테이블)
- TBNWS_AD: user=`tbnws22`, pass=`dnpfzja!` (hourly 실시간)

### 일별 캠페인 성과 (TBNWS_ADMIN)
```bash
SSH_AUTH_SOCK="" ssh ... "cd /home/ec2-user/naver-ad && python3 -c \"
import pymysql
conn = pymysql.connect(host='222.122.42.221', port=3306, user='dev', password='@Tbnws1346', database='TBNWS_ADMIN', charset='utf8mb4')
cur = conn.cursor()
cur.execute('''
    SELECT stat_date, campaign_name,
           SUM(impressions) imp, SUM(clicks) clk, SUM(total_cost) cost,
           SUM(total_conv_revenue) rev,
           CASE WHEN SUM(total_cost)>0 THEN ROUND(SUM(total_conv_revenue)/SUM(total_cost)*100,1) ELSE 0 END roas
    FROM naver_ad_gfa_campaign_daily
    WHERE stat_date >= DATE_SUB(CURDATE(), INTERVAL 7 DAY)
    GROUP BY stat_date, campaign_name
    ORDER BY campaign_name, stat_date
''')
for r in cur.fetchall():
    print(f'{r[0]} | {r[1]} | cost:{r[4]:,} | ROAS:{r[6]}%')
conn.close()
\""
```

### hourly 테이블 (TBNWS_AD) — ⚠️ 컬럼명 다름!
- 테이블: `naver_ad_gfa_campaign_hourly`
- 시간 컬럼: `hour_of_day` (NOT `hour`!)
- 컬럼: impressions, clicks, total_cost, total_conv_revenue, purchase_conv_revenue, cart_conv_revenue, total_roas, purchase_roas

---

## 7. 텔레그램 봇

- 봇: `@tbcl_gfa_bot`
- 토큰: `8323306043:AAGAqy7GJ9dORL58bWztGDaIeuVlNDQNKeQ`
- Chat ID: `8188602191`
- 기존 OPS 봇(EC2 상시 가동)과 별도

### offset 확인
```bash
curl -s "https://api.telegram.org/bot8323306043:AAGAqy7GJ9dORL58bWztGDaIeuVlNDQNKeQ/getUpdates?offset=-1" | python3 -c "
import sys, json
data = json.load(sys.stdin)
if data.get('result'):
    last = data['result'][-1]
    print(f'Last: {last[\"update_id\"]}, Next offset: {last[\"update_id\"] + 1}')
else:
    print('No updates')
"
```

### 폴링
```bash
curl -s "https://api.telegram.org/bot8323306043:AAGAqy7GJ9dORL58bWztGDaIeuVlNDQNKeQ/getUpdates?offset=OFFSET&timeout=30" | python3 -c "
import sys, json
data = json.load(sys.stdin)
if data.get('ok') and data.get('result'):
    for u in data['result']:
        print(f'[{u[\"update_id\"]}] {u.get(\"message\",{}).get(\"text\",\"\")}')
else:
    print('No messages')
"
```

### 메시지 전송
```bash
curl -s "https://api.telegram.org/bot8323306043:AAGAqy7GJ9dORL58bWztGDaIeuVlNDQNKeQ/sendMessage" \
  --data-urlencode "chat_id=8188602191" \
  --data-urlencode "text=메시지 내용"
```

---

## 8. GFA 운영 핵심 원칙

### 매출 지표 (⚠️ 혼동 금지!)
- **total_conv_revenue**: 전환 기여 매출 (장바구니+구매+기타) → **실매출 아님!** 부풀려진 수치
- **purchase_conv_revenue**: 구매 전환 매출
- **cart_conv_revenue**: 장바구니 전환 매출
- 스마트스토어 실매출: `naver_smartstore_orders` 기준
- 리포트 시 절대 total_conv_revenue를 "매출"로 쓰지 말 것

### 비교 원칙
- 동일 시간 기준 비교 필수 (17시 vs 17시)
- 하루 안 끝났으면 "어제보다 덜 썼다"는 무의미

### 페이싱 분석 결과 (3/10 발견)
- 매출 피크: 9-11시, 14-15시, 22시
- GFA는 SA와 달리 시간대별 입찰 조정 기능 없음
- 현재 노출 시간: 00-02, 08-11, 13-00
- 문제: 활성 시간 내 예산 소진 속도 불균형

---

## 9. 스케줄러

- EC2에서 30분 주기 자동 사이클 (collect→score→decide→delegate→learn)
- `gfa_ops_ai.py`의 `_auto_start_ops_scheduler()` → 서버 부팅 20초 후 자동 시작
- 수동 변경과 충돌 가능 → 수동 변경 후 스케줄러 동작 모니터링 필요
