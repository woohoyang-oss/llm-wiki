# 노션 티켓 파이프라인 설계 (2026-03-26)

## 개요
챗봇 "개발자 불러줘" → 노션 티켓 생성 → 커맨드센터 폴링 → 슬랙 팀 배정

## 노션 DB
- URL: https://www.notion.so/tbnws/d3f5daa7d3a848398543da524a120d54
- data_source_id: a0b8418e-81fa-430c-842d-c30fd06eefd4
- 워크스페이스: 투비(TB)

### DB 스키마
- 과제명 (title): [AE-XXX] 제목
- Status (status): 시작 전 / 진행 중 / 완료
- Priority (select): Critical / High / Medium / Low
- Assignee (select): Coder / Reviewer / Tester / PL / TL
- Category (select): Bug / Feature / Refactor / Infra / Docs
- Due Date (date)
- PR Link (url)
- ID (auto_increment)
- 비고 (text)

## 플로우
1. 사용자 → 챗봇: "개발자 불러줘"
2. 챗봇: "개발자가 연결되었습니다. 어떤 개선/버그가 있으신가요?"
3. 사용자가 텍스트 + 이미지로 설명
4. 챗봇이 요약 → MCP로 노션 티켓 생성 (상태: 시작 전)
5. 커맨드센터가 3분마다 폴링 → "시작 전" 티켓 발견
6. 상태를 "진행 중"으로 변경 + 슬랙으로 팀 배정
7. 팀 작업 완료 → "완료"로 변경

## 노션 연동 방식
- **MCP 방식** (현재 POC): Claude MCP 노션 도구로 직접 CRUD
  - mcp__claude_ai_Notion__notion-fetch: DB 조회
  - mcp__claude_ai_Notion__notion-create-pages: 티켓 생성
  - mcp__claude_ai_Notion__notion-update-page: 상태 변경
- **API 토큰 방식 (미완)**: 투비 워크스페이스에 Integration 생성 필요 (관리자 권한 문제)

## 현재 POC 상태
- /loop 3분 폴링으로 동작 중 (세션 종료 시 중단)
- 슬랙 폴링: /loop 5분 (job: a4895109)
- 노션 폴링: /loop 3분 (job: f73dea7e)

## 마이그레이션 시 주의사항
- 세션 전환 시 /loop 재설정 필요
- 노션 MCP 도구 권한 확인
- 슬랙 토큰: PL_TOKEN, TL_TOKEN
- 노션 DB data_source_id 변경 없음
- 노션 API 토큰 (ntn_***MASKED***) → 투비 워크스페이스 Integration 연결 미완 (워크스페이스 권한 문제)

## 영구화 계획
1. 투비 워크스페이스 Integration 문제 해결
2. Python 스크립트 (notion_poller.py) EC2에서 상시 실행
3. 또는 /schedule 원격 에이전트로 크론 등록

## 테스트 티켓
- AE-001: 견적서 PDF 디자인 개선 (진행 중)
- AE-002: 테스트 티켓 — 사용자 요청 시뮬레이션 (진행 중)