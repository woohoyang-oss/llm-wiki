---
name: sabang-soldout
description: "사방넷 품절 처리 자동화. Use when: user asks to check out-of-stock items, process soldout on sabangnet, or sync inventory status to shopping malls. Reads Google Spreadsheet for stock data, then automates sabangnet admin to send soldout status to all connected malls."
metadata: { "openclaw": { "emoji": "📦", "requires": { "bins": ["python3", "gog"] } } }
---

# 사방넷 품절 처리 스킬

구글 스프레드시트 재고 데이터 → 사방넷 품절/판매중지 자동화

## When to Use

- "품절 처리해줘"
- "재고 없는 상품 품절 송신"
- "키크론 품절 확인"
- "사방넷 품절 상태 업데이트"
- "재고 N개 이하 제품 확인"
- "판매중지 처리"

## 환경

- Python venv: `~/.openclaw/scrapling-env/bin/python3`
- Google Sheets: `gog sheets` CLI (ai.login@tbnws.com)
- 사방넷: Scrapling StealthyFetcher/Camoufox (sbadmin13.sabangnet.co.kr)
- 스프레드시트 ID: `1ItUgaLRuxnQWN7hZRS_SjaZSbZ9WEEnshgkZQEMAXvU`
- 시트 탭: `#키크론`, `#지티기어`, `#해피독_강아지사료`, `#키크론_2`, `#가습기`, `#Xtrfy`

## Commands

### 품절 대상 조회 (사방넷 접속 불필요)

```bash
GOG_KEYRING_PASSWORD="mailbot2026" ~/.openclaw/scrapling-env/bin/python3 \
  ~/.openclaw/workspace/skills/sabang-soldout/soldout.py list-soldout \
  --sheet="#키크론" --threshold=0
```

### 1단계: 재고 부족 + 사방넷 등록 확인 (check)

```bash
SABANG_PASS="@tbnws1245" GOG_KEYRING_PASSWORD="mailbot2026" \
  ~/.openclaw/scrapling-env/bin/python3 \
  ~/.openclaw/workspace/skills/sabang-soldout/soldout.py check \
  --sheet="#키크론" --threshold=0 --date-range=3개월
```

### 특정 G-코드 검색

```bash
SABANG_PASS="@tbnws1245" ~/.openclaw/scrapling-env/bin/python3 \
  ~/.openclaw/workspace/skills/sabang-soldout/soldout.py search \
  G-0029-0362-0002 --date-range=3개월
```

### 품절 처리 실행

```bash
SABANG_PASS="@tbnws1245" ~/.openclaw/scrapling-env/bin/python3 \
  ~/.openclaw/workspace/skills/sabang-soldout/soldout.py execute \
  G-0029-0362-0002 --date-range=3개월
```

### 시트 기반 전체 실행

```bash
# dry-run (확인만)
SABANG_PASS="@tbnws1245" GOG_KEYRING_PASSWORD="mailbot2026" \
  ~/.openclaw/scrapling-env/bin/python3 \
  ~/.openclaw/workspace/skills/sabang-soldout/soldout.py run \
  --sheet="#키크론" --threshold=0 --date-range=3개월 --dry-run

# 실제 실행
SABANG_PASS="@tbnws1245" GOG_KEYRING_PASSWORD="mailbot2026" \
  ~/.openclaw/scrapling-env/bin/python3 \
  ~/.openclaw/workspace/skills/sabang-soldout/soldout.py run \
  --sheet="#키크론" --threshold=0 --date-range=3개월
```

## 옵션

| 옵션 | 기본값 | 설명 |
|------|--------|------|
| `--sheet` | `#키크론` | 스프레드시트 탭 이름 |
| `--threshold` | `0` | 재고 N개 이하를 품절 대상으로 (0=재고없음, 5=5개이하) |
| `--date-range` | `3개월` | 사방넷 송신일 필터 (1주일, 1개월, 3개월, 6개월, 1년) |
| `--dry-run` | off | 실행 없이 확인만 |

## 프로세스

### 1단계: 품절 대상 확인 (check)
1. `gog sheets get` → 스프레드시트에서 재고 threshold 이하 G-코드 추출
2. Scrapling(Camoufox) → 사방넷 로그인 → sbadmin13 진입
3. 쇼핑몰상품수정 → 날짜범위 설정 → 자체상품코드 검색
4. 사방넷에 등록된 제품 리스트 출력

### 2단계: 품절 송신 (run/execute)
1. 1단계와 동일한 검색
2. 검색 결과 전체 선택 → 상품수정 송신
3. 웹 팝업 내 송신 완료 대기

## 주의사항

- 비밀번호는 환경변수 `SABANG_PASS`로 전달
- GOG_KEYRING_PASSWORD="mailbot2026" 필요 (non-TTY 환경)
- 날짜범위 기본 "1주일" → 반드시 "3개월" 이상 설정 (키크론 제품은 최근 송신이 드묾)
- 1회 500건 이하 권장
- 노란색 행(변환 불가)은 자동 건너뜀
- sbadmin13 페이지 구조 변경 시 셀렉터 조정 필요
