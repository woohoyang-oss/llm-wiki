# UI/UX Design System

## Theme System
라이트(기본)/다크 테마 전환. CSS Custom Properties 기반.

### CSS Variables (~50개)
```css
:root {  /* Light theme (default) */
    --bg-primary: #ffffff;
    --bg-secondary: #f9fafb;
    --bg-tertiary: #f3f4f6;
    --bg-elevated: #ffffff;
    --bg-hover: #f3f4f6;
    --text-primary: #111827;
    --text-secondary: #4b5563;
    --text-tertiary: #9ca3af;
    --border-primary: #e5e7eb;
    --border-secondary: #f3f4f6;
    --accent: #4f46e5;  /* indigo */
    --accent-text: #4338ca;
    /* 상태 배지 */
    --status-002-bg: #ecfdf5; --status-002-text: #065f46;  /* 공급중: green */
    --status-003-bg: #fef2f2; --status-003-text: #991b1b;  /* 완전품절: red */
    --status-001-bg: #fefce8; --status-001-text: #854d0e;  /* 일시중지: yellow */
    --status-004-bg: #eff6ff; --status-004-text: #1e40af;  /* 대기중: blue */
}

[data-theme="dark"] {
    --bg-primary: #09090b;
    --bg-secondary: #18181b;
    --bg-tertiary: #27272a;
    --bg-elevated: #18181b;
    --text-primary: #f9fafb;
    --text-secondary: #a1a1aa;
    --text-tertiary: #71717a;
    --accent: #818cf8;
    --accent-text: #a5b4fc;
    /* 상태 배지 (다크) */
    --status-002-bg: #14532d; --status-002-text: #4ade80;
    --status-003-bg: #7f1d1d; --status-003-text: #fca5a5;
    --status-001-bg: #713f12; --status-001-text: #fde047;
    --status-004-bg: #1e3a5f; --status-004-text: #93c5fd;
}
```

### FOUC Prevention
```html
<script>
(function(){
    const saved = localStorage.getItem('sabang-theme');
    if (saved === 'dark') document.documentElement.setAttribute('data-theme','dark');
})();
</script>
```
이 스크립트가 CSS <style> 태그 앞에 위치하여 깜빡임 방지.

## Layout
- 최대 너비: `max-w-4xl` (896px), 가운데 정렬
- 헤더: 타이틀 + 서브타이틀 + 테마토글 + 설정 버튼
- 글로벌 소스 상태바: 활성 소스명 + 스냅샷 시간
- **6탭**: 사용 방법 | 소스데이터 관리 | 소스 미리보기 | 배치 관리 | 수동 관리 | 이력 조회
- 이력 조회 내 서브탭: 📋 이력 조회 | ⚠ 송신실패 분석
- 로그 터미널: 항상 다크 (#0a0a0a 배경, 테마 무관)

## History Table
- 컬럼: ▶(24px) | 구분(56px) | 작업명(auto) | 결과(100px) | 실행자(80px) | 소요(70px) | 실행시간(130px)
- 구분 태그: 좌측 3px 상태 색상 바 (done=green, error=red, running=yellow)
- 결과: 콤팩트 포맷 (`93/160`), white-space: nowrap
- 행 클릭 → 펼침: 메타 정보 바(회색 배경) + 결과 요약 + G-코드 상세(details/summary) + 실행 로그

## Failure Analysis (송신실패 분석 서브탭)
- 쇼핑몰별 실패 카드 (실패 건수 배지, 위험도 색상)
- 실패 사유별 그룹핑 (빈도순 정렬)
- 대응 힌트 자동 생성 (옵션 매핑, API 연동 등)
- 누적 방식: limit=500 별도 API 호출로 전체 배치 이력 수집

## Products Table
- `table-layout: fixed` - 고정 레이아웃
- 컬럼 비율: G-코드 18% | 상품명 28% | 모델명 18% | 쇼핑몰 12% | 상태 8% | 판매가 16%
- td에 `overflow: hidden; text-overflow: ellipsis; white-space: nowrap` 적용
- 원산지(orgpl_ntn): 리스트에서 제거됨, 상세 모달에서만 표시

## Status Badge Colors
- 002 (공급중): green 계열
- 003 (완전품절): red 계열
- 001 (일시중지): yellow 계열
- 004 (대기중): blue 계열

## Batch Card Design
- 좌측 색상 바: green(재입고), red(품절), gray(기타)
- 대상 건수 배지: green/red 색상
- 버튼: "지금 실행" (primary), "확인만" (ghost)
- 하단: 스케줄 시간 + 상시실행 토글
