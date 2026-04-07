# ToBe AI 프로젝트 컨텍스트

## 레포지토리 구조
- `tbnws-ai/` — Python FastAPI + MCP 서버 (AI 데이터 게이트웨이)
- `tobe-ai-v2/` — Next.js 프론트엔드
- `tbnws_admin_back/` — Spring Boot 어드민 백엔드

## GitHub 리모트
- tbnws-ai: `woohoyang-oss/tbnws-ai` (push 대상)
- tobe-ai-v2: `woohoyang-oss/tobe-ai-v2`
- tbnws_admin_back: `TobeNetworksGlobal/tbnws_admin_back` (push 금지!)

## 배포 정보
- **서버**: tbe.kr (AWS EC2)
- **SSH**: `sshpass -p 'ec2-user' ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no ec2-user@tbe.kr`
- **tbnws-ai 배포 경로**: `/home/ec2-user/tbnws-data-api`
- **tbnws-ai 프로세스**: `uvicorn main:app --host 127.0.0.1 --port 8100` (.venv/bin/python3)
- **배포 절차**:
  1. `git push origin main` (로컬 → GitHub)
  2. SSH 접속 후 `cd /home/ec2-user/tbnws-data-api && git pull origin main`
  3. 기존 프로세스 kill 후 `nohup .venv/bin/python3 -m uvicorn main:app --host 127.0.0.1 --port 8100 > /dev/null 2>&1 &`

## 주요 규칙
- `TobeNetworksGlobal` org에는 절대 push 금지
- `erp_purchase_order` 테이블: `order_seq IS NULL` = PENDING(임시발주), `IS NOT NULL` = COMPLETED
- MCP 서버 수정 시 `routers/` REST API와 동기화 필수
