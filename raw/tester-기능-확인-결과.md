# 기능 확인 결과 (2026-03-23)

## 1. 이메일 스타일 학습 기능

### API 엔드포인트 (admin.py)
- POST /api/admin/personal-learning/start (라인 5755) — 학습 시작
- GET /api/admin/personal-learning/status (라인 5776) — 진행 상황
- GET /api/admin/personal-learning/guide (라인 5798) — 스타일 가이드 조회
- PUT /api/admin/personal-learning/guide (라인 5836) — 스타일 가이드 수정
- GET /api/admin/personal-learning/thread-context/{thread_id} (라인 5857)

### API 호출 결과
- GET /status → {"status":"idle","progress":0} — 정상 응답
- GET /guide → {"status":"not_found"} — 아직 학습 안 됨
- 구현 상태: 완전 구현됨 (stub 아님). personal_learning.py 서비스 존재.

## 2. 담당자 삭제 UI 버그

### 프론트엔드 코드 분석
- 파일: admin/src/app/agents/page.tsx
- 삭제 핸들러: handleDelete (라인 204-212)
  - confirm 다이얼로그로 확인 후 deleteAgent.mutate(agent.id) 호출
- Mutation: admin/src/lib/mutations.ts (라인 451-459)
  - DELETE /admin/agents/${agentId} 호출
  - onSuccess: agents 쿼리 invalidate
- AgentCard 컴포넌트 (라인 58-146)
  - 삭제 버튼: Trash2 아이콘, onClick={onDelete}
  - 버튼은 group-hover:opacity-100 으로 hover 시에만 보임

### 버그 진단
- 코드 레벨에서는 버그 없음. 삭제 로직 정상.
- API 테스트에서도 DELETE /api/admin/agents/{id} 정상 동작 확인 (CRUD 점검에서 검증)
- 가능한 원인:
  1. 삭제 버튼이 hover 시에만 보이므로 (opacity-0 → group-hover:opacity-100) 발견이 어려울 수 있음
  2. 모바일/터치 환경에서 hover가 작동하지 않아 삭제 버튼 접근 불가
  3. 특정 에이전트(자기 자신 등)는 삭제 제한이 있을 수 있으나 프론트엔드에서는 제한 없음
- 결론: 프론트엔드 코드에 명확한 버그는 없음. hover UX 이슈 가능성.

### 관련 파일
- admin/src/app/agents/page.tsx:204-212 (삭제 핸들러)
- admin/src/lib/mutations.ts:451-459 (삭제 뮤테이션)
