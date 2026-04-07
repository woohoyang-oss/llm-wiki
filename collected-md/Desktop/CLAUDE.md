# GFA 광고 운영 AI — 클로드 CLI 전체 매뉴얼

> 이 문서 하나로 새 클로드 세션에서 GFA 광고 컨트롤을 바로 시작할 수 있음

---

## 1. 텔레그램 세팅

### 봇 매핑 (⚠️ 헷갈리지 말 것!)
- **맥프로14 = @tbcl_gfa_bot** (토큰: `8323306043:AAGAqy7GJ9dORL58bWztGDaIeuVlNDQNKeQ`)
- **맥미니 = @tbminipro_bot** (별도 클로드 CLI 세션)
- **Chat ID (개인)**: `8188602191`
- **Chat ID (GFA 방 그룹)**: `-5109506868`
- 기존 OPS 봇(`8740670708`)과 별도 — EC2에서 별도 가동 중

### ⚠️ 역슬래시 이슈 방지
- 메시지 전송 시 반드시 `--data-urlencode` 사용 (역슬래시/특수문자 깨짐 방지)
- heredoc 대신 `--data-urlencode "text=..."` 패턴 사용

### 최신 offset 확인
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

### 폴링 (30초 long polling)
```bash
curl -s "https://api.telegram.org/bot8323306043:AAGAqy7GJ9dORL58bWztGDaIeuVlNDQNKeQ/getUpdates?offset=OFFSET&timeout=30" | python3 -c "
import sys, json
data = json.load(sys.stdin)
for u in data.get('result', []):
    msg = u.get('message', {})
    chat = msg.get('chat', {})
    print(f'[{u[\"update_id\"]}] type:{chat.get(\"type\")} id:{chat.get(\"id\")} | {msg.get(\"text\",\"\")[:80]}')
"
```
- `OFFSET`에 이전 update_id + 1 넣기
- 30초 타임아웃 후 새 메시지 없으면 빈 결과 → 다시 호출 반복
- 개인/그룹 메시지 모두 수신됨 (`type:private` vs `type:group`)

### 메시지 보내기
```bash
# 개인 메시지
curl -s "https://api.telegram.org/bot8323306043:AAGAqy7GJ9dORL58bWztGDaIeuVlNDQNKeQ/sendMessage" \
  --data-urlencode "chat_id=8188602191" \
  --data-urlencode "text=메시지 내용"

# 그룹 메시지
curl -s "https://api.telegram.org/bot8323306043:AAGAqy7GJ9dORL58bWztGDaIeuVlNDQNKeQ/sendMessage" \
  --data-urlencode "chat_id=-5109506868" \
  --data-urlencode "text=메시지 내용"
```
- 반드시 `--data-urlencode` 사용 (한글/특수문자 이슈 방지)

### 파일 보내기
```bash
curl -s -F "chat_id=8188602191" \
  -F "document=@/파일경로" \
  -F "caption=설명" \
  "https://api.telegram.org/bot8323306043:AAGAqy7GJ9dORL58bWztGDaIeuVlNDQNKeQ/sendDocument"
```

---

## 2. 깃헙 레포 정보

### gfa-auto (POC)
- **URL**: `https://github.com/woohoyang-oss/gfa-auto.git`
- **로컬**: `~/Library/Mobile Documents/com~apple~CloudDocs/claude/gfa-auto/`
- **브랜치**: `main`
- GFA 리포트 다운로드 POC → 기능은 tbnws-ad에 통합됨

### tbnws-ad (메인 프로젝트)
- **로컬**: `~/tbnws-ad/`
- ToBe Networks Marketing Analysis 대시보드 (Flask + Vanilla JS)
- GFA OPS AI 엔진 포함

### 배포 체크리스트 (tbnws-ad)
1. `app/static/data/changelog.json` 배열 맨 앞에 새 항목 추가
2. changelog 포함하여 git add + commit
3. git push
4. SCP → EC2 배포
5. Python 변경 시 `sudo systemctl restart tbnws-ad.service`
- **절대 commit 먼저 하고 changelog 나중에 하지 말 것!**

---

## 3. GFA 광고 관리 방법

### EC2 접속
```bash
SSH_AUTH_SOCK="" ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i ~/Desktop/claude.pem ec2-user@100.103.222.7
```
- SSH 키: `~/Desktop/claude.pem`
- IP: `100.103.222.7` (Tailscale 필수! 공인IP `3.37.188.23` 사용 금지)
- **절대 `pkill -f python` 사용 금지**
- **서버 재시작은 사용자 허락 없이 절대 금지**

### 현재 캠페인 구조 (계정 26343, 2026-03-10 기준)

| Campaign | CampaignNo | adSet(No) | 일예산 | 상태 |
|---|---|---|---|---|
| BOF 0305 장바구니 전환 | 1268005 | 4100170 | ₩50,000 | ON (3/10 증액) |
| BOF 웹사이트 전환 #1 | 1162692 | 1개 | ₩200,000 | ON |
| MOF 0225 SE | 1016343 | 3519687 | ₩44,000 | OFF (3/10 비활성화) |
| TOF 0113 | 1209023 | 3961952(60K), 3965735(45K) | ₩105,000 | ON |
| TOF 0223 | 1255832 | 2개 | ₩90,000 | ON |
| TOF 1127 전환 | 1156362 | 1개 | ₩80,000 | ON |
| TOF 1211 | 1175008 | 3883304(OFF), 3883324(10K), 3883414(20K), 3901558(10K) | ₩50,000 | ON |

### adset 현황 조회 (OPS AI SQLite)
```bash
SSH_AUTH_SOCK="" ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes \
  -i ~/Desktop/claude.pem ec2-user@100.103.222.7 \
  "cd /home/ec2-user/naver-ad && python3 -c \"
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

### adset_snapshots 스키마 (⚠️ 컬럼명 주의!)
```
id, ts, adset_no, adset_name, campaign_no, campaign_name, funnel,
activated (NOT enabled! NOT is_active!),
budget, spent_ratio, status,
impressions, clicks, cost, conversions, purchases, add_to_cart,
conv_revenue, purchase_revenue, cart_revenue,
ctr, cpc, cpm, roas, adj_roas, cpa
```

### 일별 성과 조회 (MySQL)
```bash
SSH_AUTH_SOCK="" ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes \
  -i ~/Desktop/claude.pem ec2-user@100.103.222.7 \
  "cd /home/ec2-user/naver-ad && python3 -c \"
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
    GROUP BY stat_date, campaign_name ORDER BY campaign_name, stat_date
''')
for r in cur.fetchall():
    print(f'{r[0]} | {r[1]} | cost:{r[4]:,} | ROAS:{r[6]}%')
conn.close()
\""
```

### hourly 테이블 (TBNWS_AD) — ⚠️ 컬럼명 다름!
- DB: `TBNWS_AD` (host=222.122.42.221, user=tbnws22, pass=dnpfzja!)
- 테이블: `naver_ad_gfa_campaign_hourly`
- 시간 컬럼: `hour_of_day` (NOT `hour`!)
- 기타: impressions, clicks, total_cost, total_conv_revenue, purchase_conv_revenue, cart_conv_revenue

---

## 4. GFA 예산 변경 / ON·OFF 방법

### 실행 패턴 (EC2에서 OpsGfaApi 사용)

```bash
# 1) 로컬에 스크립트 작성
cat > /tmp/gfa_ops.py << 'PYEOF'
import sys, os
sys.path.insert(0, '/home/ec2-user/naver-ad')
os.chdir('/home/ec2-user/naver-ad')
from app.gfa_ops.ops_state import OpsGfaApi

api = OpsGfaApi()  # 자동 세션 로드 + warm_up(XSRF 갱신)

# 예산 변경
result = api.change_budget(4100170, 50000)
print(f"Budget change: {result}")
# → (True, 50000.0, 50000.0, [adset_detail_dict])

# ON/OFF
result = api.toggle_adset(3519687, False)  # OFF
print(f"Toggle: {result}")
# → (True, False)
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

### GFA API 엔드포인트 (내부 구조)
```
# 예산 변경
POST /apis/gfa/v1/adAccounts/{accountNo}/adSets/edit
  ?adSets[0].no={adSetNo}&adSets[0].budgetAmount={amount}

# ON/OFF (⚠️ 별도 엔드포인트!)
POST /apis/gfa/v1/adAccounts/{accountNo}/adSets/activation
  ?activated={true|false}&adSetNo={adSetNo}
```
- `adSets/edit`의 `activated` 파라미터는 작동 안 함! 반드시 `/adSets/activation` 사용
- 모든 제어는 adSet 레벨 (캠페인 레벨 edit 없음)

### 주의사항
- `OpsGfaApi()` 초기화 시 자동으로 세션 로드 + warm_up (XSRF 갱신)
- 세션 만료 시 → tbe.kr 대시보드에서 수동 갱신 필요
- `change_budget()` 금액은 자동 `round_budget()` 처리됨
- 스케줄러가 30분마다 자동 조정 → 수동 변경 후 되돌릴 수 있으니 주의

---

## 5. OPS AI 관련 툴

### 프로젝트 구조
- **위치**: `~/tbnws-ad/app/gfa_ops/` (8개 ops_*.py 모듈)
- **라우트**: `app/routes/gfa_ops_ai.py` → `/api/gfa-ops-ai/*`
- **프론트**: `app/static/js/tabs/gfa-ops-ai.js`
- **DB**: SQLite `app/gfa_ops/data/ops.db` (자동생성)
- **세션**: EC2 심볼릭 링크 `data/session.json` → `/data/gfa_sessions/keychron_session.json`

### 주요 모듈
- `ops_state.py` — OpsGfaApi (change_budget, toggle_adset, collect_all)
- `ops_session.py` — OpsSession (세션 관리, XSRF, warm_up)
- `ops_score.py` — 효율 점수 계산
- `ops_decide.py` — 자동 의사결정
- `ops_delegate.py` — 실행 위임
- `ops_learn.py` — 학습 모듈

### 스케줄러
- EC2에서 30분 주기 자동 사이클: collect → score → decide → delegate → learn
- `gfa_ops_ai.py`의 `_auto_start_ops_scheduler()` → 서버 부팅 20초 후 자동 시작
- 수동 변경과 충돌 가능 → 모니터링 필요

### DB 접속 정보
| DB | Host | User | Pass | 용도 |
|---|---|---|---|---|
| TBNWS_ADMIN | 222.122.42.221:3306 | dev | @Tbnws1346 | daily 테이블 |
| TBNWS_AD | 222.122.42.221:3306 | tbnws22 | dnpfzja! | hourly 실시간 |
| OPS SQLite | EC2 로컬 | - | - | ops.db (adset_snapshots 등) |

---

## 6. GFA 운영 핵심 원칙

### 매출 지표 (⚠️ 혼동 금지!)
- **total_conv_revenue**: 전환 기여 매출 → **실매출 아님!** 부풀려진 수치
- **purchase_conv_revenue**: 구매 전환 매출 (실제 구매 기여값)
- **스마트스토어 실매출**: `naver_smartstore_orders` 기준
- 리포트 시 절대 total_conv_revenue를 "매출"로 쓰지 말 것

### 비교 원칙
- 동일 시간 기준 비교 필수 (17시 vs 17시)
- 하루 안 끝났으면 "어제보다 덜 썼다"는 무의미
