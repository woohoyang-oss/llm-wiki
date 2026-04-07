# Server Setup Guide (새 환경 / 다른 세션에서 이어서 개발할 때)

## 1. Git Clone / Pull
```bash
git clone https://github.com/tobe-macpro/openclaw-sabang
cd openclaw-sabang
# 또는 기존 저장소에서:
git pull origin main
```

## 2. 로컬 서버 시작
```bash
cd dashboard
SABANG_PASS='@tbnws1245' GOG_KEYRING_PASSWORD='mailbot2026' python3 app.py
```
- 포트: 5050 (변경: `--port 8080`)
- sabangnet.db가 없으면 자동 생성 (빈 DB)
- 배치 설정(재입고 복구, 품절 처리) 자동 seed (`db.py._seed_batch_configs`)

## 3. 초기 데이터 로드 (DB가 비어있을 때)
서버 시작 후 대시보드에서 **"새로고침"** 버튼 클릭
→ 사방넷 API (`getMallProductUpdateLists`) 호출
→ 전체 상품 스냅샷 fetch (2,123 G-codes, 13,386+ products)
→ `sabangnet.db`에 자동 저장

또는 API로 직접:
```bash
curl -X POST http://localhost:5050/api/snapshot/refresh
```

## 4. 브라우저 접속
- `http://localhost:5050/`
- Chrome에서 localhost 접속 불가 시: `http://<로컬IP>:5050/`

## 5. DB 파일은 git에 포함되지 않음
- `.gitignore`에 `*.db` 추가되어 있음
- 새 환경에서는 위 3번 과정으로 DB 생성
- DB는 로컬에서만 존재, 스냅샷 새로고침으로 재생성 가능

## 6. Python 환경
- **필수 패키지**: Flask, requests
- `pip3 install flask requests`
- soldout.py 자동화 (배치 실행): scrapling + patchright 필요 (선택)

## 7. 원격 서버 배포
→ 상세: [deployment.md](deployment.md)
```bash
# 한 줄 배포
./deploy.sh tbe.kr ec2-user ec2-user sb
```
