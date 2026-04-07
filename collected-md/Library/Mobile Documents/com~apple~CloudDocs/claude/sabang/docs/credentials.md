# Credentials & Environment

## Server Environment Variables
- `SABANG_PASS='@tbnws1245'` - 사방넷 로그인 비밀번호
- `GOG_KEYRING_PASSWORD='mailbot2026'` - GOG keyring 비밀번호

## Python Environment
- Dashboard: system Python 3 (`python3`)
- Automation (soldout.py): `~/.openclaw/scrapling-env/bin/python3` (Python 3.9)
  - scrapling, patchright 패키지 필요

## Network
- Flask server: `0.0.0.0:5050`
- Chrome에서 localhost 접속 불가 → `192.168.1.215:5050` 사용
- curl로는 `localhost:5050` 정상 접속
