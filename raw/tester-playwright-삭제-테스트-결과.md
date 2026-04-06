# Playwright 담당자 삭제 테스트 결과 (2026-03-23)

## 실행 결과: DELETE_FAILED (백엔드 버그)

## Playwright 테스트 상세
- 타겟: 박혜경 (hyekyung.park@tbnws.com, id=17)
- 카드 발견: OK (agents 페이지에서 정상 렌더링)
- 삭제 버튼 발견: OK (hover 후 visible)
- Confirm 다이얼로그: OK ("담당자 삭제" + 취소/삭제 버튼)
- 삭제 버튼 클릭: OK

## 네트워크 로그
- DELETE /api/admin/agents/17 → HTTP 200
- Response body: {"id":17,"status":"deleted"}
- 콘솔 에러: 없음

## 핵심 발견: 삭제가 실제로 안 됨!
- API가 200 + deleted 응답을 반환하지만 DB에서 삭제 안 됨
- Playwright 후 API 재조회: 여전히 18명, 박혜경 존재
- curl로 직접 DELETE 재시도: 동일 (200 반환, 삭제 안 됨)

## 버그 원인 분석
- 파일: app/api/admin.py:1960-1974 (delete_agent)
- get_db()가 자동 커밋 방식 (database.py:31)
- delete_agent는 return 후 세션 종료 시 commit 시도
- FK 제약 위반으로 commit 실패 → 자동 rollback
- 하지만 FastAPI는 이미 200 응답 전송 완료

### 처리 안 된 FK 참조 (base.py)
- 609행: PersonalWritingGuide.agent_id (CASCADE)
- 629행: TelegramUserLink.agent_id (CASCADE)
- 656행: NotificationPreference.agent_id (CASCADE)
- 747행: ApprovalQueue.agent_id (CASCADE 없음!)
- 804행: ActivityLog.agent_id (CASCADE 없음!)

### 수정 방안
1. delete_agent에서 누락된 FK 참조도 NULL 처리 추가
2. 또는 return 전에 await db.flush()로 제약 위반 조기 탐지
3. 또는 await db.commit()을 명시적으로 호출하고 예외 처리

## 프론트엔드
- 코드 레벨 버그 없음
- 삭제 버튼: hover 시에만 보임 (UX 이슈 가능성, 모바일)
- API 200 응답 후 invalidateQueries로 리스트 갱신 → 삭제 안 됐으므로 다시 보임
