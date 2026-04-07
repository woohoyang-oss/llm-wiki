# Phase 5: 역할별 대시보드 + 통계 — 상세 개발 계획서 v1

> 작성일: 2026-03-13
> 의존성: Phase 3 (RBAC) 완료 후 착수, Phase 4와 병렬 가능

---

## 1. 현재 상태 분석

### 1.1 대시보드 (`/dashboard`) — 현재 기능

**이미 구현된 항목:**
- KPI 카드 4개 (오늘 수신, 오늘 답장, 평균 응답시간, SLA 위반)
- 상태 분포 바 (10개 상태)
- 퍼포먼스 링 (답장율, 자동답장율)
- 에이전트 워크로드 (상위 5명 오픈 티켓)
- 14일 볼륨 차트 (수신 vs 답장)
- 최근 활동 피드 (15건)
- 빠른 실행 카드 (승인대기, 미답장, 최근 메일, 룰)

**문제점:**
- 모든 사용자가 동일한 뷰 — 역할 구분 없음
- Agent는 전체 메일 통계를 볼 필요 없이 **내 담당 메일만** 봐야 함
- Viewer는 읽기 전용이지만 빠른 실행 카드가 수정 화면으로 연결됨
- 개인 메일함 관련 통계 없음

### 1.2 통계 (`/statistics`) — 현재 기능

**이미 구현된 항목:**
- 도넛 차트 3개 (상태, 우선순위, 발송유형 분포)
- 14일 볼륨 바 차트
- 추세 화살표 (이전 기간 대비)

**문제점:**
- 기간 선택 없음 (30일 고정)
- 에이전트별 필터 없음
- 브랜드별 필터 없음
- 개인 성과 지표 없음
- 내보내기 기능 없음

---

## 2. 역할별 대시보드 뷰 설계

### 2.1 Admin 대시보드

**목적:** 전체 운영 현황 모니터링 + 의사결정

```
┌─────────────────────────────────────────────────────┐
│  대시보드                        [자동갱신 ⟳] [30s] │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ■ KPI 카드 (상단 1줄)                              │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐     │
│  │오늘  │ │오늘  │ │평균   │ │SLA   │ │미배정 │     │
│  │수신  │ │답장  │ │응답   │ │위반  │ │메일  │     │
│  │ 42   │ │ 35   │ │2.3h  │ │ 3    │ │ 12   │     │
│  │+12%↑ │ │-5%↓  │ │-15%↑ │ │+2↑   │ │      │     │
│  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘     │
│                                                     │
│  ■ 상태 분포 바 + 퍼포먼스 링                       │
│  [===새==|===처리중===|=대기=|===완료========]       │
│  답장율: 83%    자동답장율: 45%                      │
│                                                     │
│  ■ 팀 워크로드 (전체 에이전트)                      │
│  ┌────────────────────────────────────────────┐     │
│  │ 김민수    ████████████ 12                  │     │
│  │ 이영희    ██████████ 10                    │     │
│  │ 박서준    ████████ 8                       │     │
│  │ 최하나    ████ 4                           │     │
│  └────────────────────────────────────────────┘     │
│                                                     │
│  ■ 14일 볼륨 차트                                   │
│  ┌────────────────────────────────────────────┐     │
│  │ ▁▂▃▄▅▆▇█▇▆▅▄▃▂  수신                     │     │
│  │ ▁▁▂▃▄▅▆▇▆▅▄▃▂▁  답장                     │     │
│  └────────────────────────────────────────────┘     │
│                                                     │
│  ■ 빠른 실행                                        │
│  [승인대기 5건] [미답장 12건] [SLA위반 3건] [룰관리] │
│                                                     │
│  ■ 최근 활동 피드                                   │
│  10:23 김민수 - 배송문의#1234 답장                   │
│  10:15 시스템 - 자동답장 발송 (confidence: 0.92)    │
│  10:02 이영희 - 교환요청#1233 에스컬레이션          │
│  ...                                                │
└─────────────────────────────────────────────────────┘
```

### 2.2 Agent 대시보드

**목적:** 내 업무 현황 + 개인 성과 + 행동 유도

```
┌─────────────────────────────────────────────────────┐
│  내 대시보드                     [자동갱신 ⟳] [30s] │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ■ 내 KPI (상단)                                    │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐               │
│  │내 담당│ │오늘  │ │내 평균│ │내 SLA│               │
│  │미처리 │ │처리  │ │응답  │ │위반  │               │
│  │ 8    │ │ 12   │ │1.5h  │ │ 1    │               │
│  └──────┘ └──────┘ └──────┘ └──────┘               │
│                                                     │
│  ■ 내 할 일 (Action Required)                       │
│  ┌────────────────────────────────────────────┐     │
│  │ 🔴 SLA 임박 (1건) — 30분 이내 응답 필요    │     │
│  │   └ [배송 지연 문의] from: hong@gmail.com   │     │
│  │ 🟡 승인 대기 (3건) — 내가 승인해야 할 건    │     │
│  │   └ [키크론 Q1 불량] confidence: 0.78       │     │
│  │   └ [반품 요청] confidence: 0.72            │     │
│  │   └ [결제 오류] confidence: 0.81            │     │
│  │ 🔵 신규 배정 (4건) — 아직 확인 안 한 건     │     │
│  │   └ [제품 문의] [교환 요청] [AS 접수] ...   │     │
│  └────────────────────────────────────────────┘     │
│                                                     │
│  ■ 개인 메일함 요약                                 │
│  ┌────────────────────────────────────────────┐     │
│  │ 📧 읽지 않은 개인 메일: 5건                 │     │
│  │ 📝 AI 초안 대기: 2건                        │     │
│  │ [내 메일함 바로가기 →]                       │     │
│  └────────────────────────────────────────────┘     │
│                                                     │
│  ■ 내 7일 성과 차트                                 │
│  ┌────────────────────────────────────────────┐     │
│  │ 처리건수  ▁▂▃▅▆▇█                          │     │
│  │ 응답시간  █▇▆▅▃▂▁ (낮을수록 좋음)           │     │
│  └────────────────────────────────────────────┘     │
│                                                     │
│  ■ 최근 내 활동                                     │
│  10:23 배송문의#1234에 답장                          │
│  10:02 교환요청#1233 에스컬레이션                   │
│  09:45 제품문의#1230 AI 초안 승인                   │
│  ...                                                │
└─────────────────────────────────────────────────────┘
```

### 2.3 Viewer 대시보드

**목적:** 운영 현황 모니터링 (읽기 전용)

```
┌─────────────────────────────────────────────────────┐
│  운영 현황                       [자동갱신 ⟳] [60s] │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ■ 전체 KPI (Admin과 동일)                          │
│  (단, 빠른 실행 카드 없음 — 수정 화면 연결 제거)    │
│                                                     │
│  ■ 상태 분포 + 퍼포먼스 링                          │
│  ■ 팀 워크로드                                      │
│  ■ 14일 볼륨 차트                                   │
│  ■ 최근 활동 피드 (읽기 전용)                       │
│                                                     │
│  (모든 클릭 가능한 액션 버튼 숨김)                  │
└─────────────────────────────────────────────────────┘
```

---

## 3. 역할별 통계 페이지 설계

### 3.1 Admin 통계

```
┌─────────────────────────────────────────────────────┐
│  통계                                               │
│                                                     │
│  기간: [7일] [30일] [90일] [사용자정의]             │
│  브랜드: [전체 ▼]  에이전트: [전체 ▼]               │
│                              [CSV 내보내기]          │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ■ 핵심 지표 (선택 기간)                            │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐       │
│  │총 수신 │ │총 답장 │ │평균응답│ │자동처리│       │
│  │ 1,234  │ │ 1,100  │ │ 2.3h  │ │ 45%   │       │
│  │이전대비│ │이전대비│ │이전대비│ │이전대비│       │
│  │ +12% ↑ │ │ +8% ↑  │ │-15% ↑ │ │ +5% ↑ │       │
│  └────────┘ └────────┘ └────────┘ └────────┘       │
│                                                     │
│  ■ 분포 차트 (도넛 × 4)                            │
│  [상태분포] [우선순위] [발송유형] [브랜드별]         │
│                                                     │
│  ■ 일별 추이 (라인 차트)                           │
│  ┌────────────────────────────────────────────┐     │
│  │  수신 ── 답장 ── 자동발송                   │     │
│  │  ╱╲ ╱╲                                     │     │
│  │ ╱  ╲╱  ╲                                   │     │
│  └────────────────────────────────────────────┘     │
│                                                     │
│  ■ 에이전트별 성과 (테이블)                         │
│  ┌──────────┬──────┬──────┬──────┬──────┐          │
│  │ 에이전트 │처리건│평균  │SLA  │만족도│          │
│  │          │      │응답  │준수율│      │          │
│  ├──────────┼──────┼──────┼──────┼──────┤          │
│  │ 김민수   │ 45   │1.2h  │ 95% │ 4.2  │          │
│  │ 이영희   │ 38   │1.8h  │ 88% │ 4.5  │          │
│  │ 박서준   │ 32   │2.1h  │ 82% │ 3.9  │          │
│  └──────────┴──────┴──────┴──────┴──────┘          │
│                                                     │
│  ■ 카테고리별 분석 (가로 바 차트)                   │
│  배송: ████████████████ 340건                      │
│  교환: ██████████ 220건                            │
│  AS:   ████████ 180건                              │
│  결제: ██████ 140건                                │
│                                                     │
│  ■ 응답시간 분포 (히스토그램)                       │
│  0-15m: ████████ 32%                               │
│  15-60m: ██████████ 28%                            │
│  1-4h: ████████████ 25%                            │
│  4-24h: ████ 10%                                   │
│  24h+: ██ 5%                                       │
│                                                     │
│  ■ AI 성과 분석                                     │
│  평균 confidence: 0.82                              │
│  자동발송 비율: 45%                                 │
│  승인 후 수정율: 23%                                │
│  지식문서 활용율: 67%                               │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 3.2 Agent 통계

```
┌─────────────────────────────────────────────────────┐
│  내 성과                                            │
│                                                     │
│  기간: [7일] [30일] [90일]                          │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ■ 내 핵심 지표                                     │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐       │
│  │내 처리 │ │평균응답│ │SLA    │ │지식생성│       │
│  │ 45건   │ │ 1.2h  │ │준수95%│ │ 8건   │       │
│  └────────┘ └────────┘ └────────┘ └────────┘       │
│                                                     │
│  ■ 내 일별 처리 추이                                │
│  ┌────────────────────────────────────────────┐     │
│  │  처리건수 ── 응답시간                       │     │
│  └────────────────────────────────────────────┘     │
│                                                     │
│  ■ 내 카테고리별 처리 비율                          │
│  [도넛 차트: 배송 40%, 교환 25%, AS 20%, 기타 15%]  │
│                                                     │
│  ■ 내 AI 활용 현황                                  │
│  AI 초안 수락율: 68%                                │
│  AI 초안 수정율: 22%                                │
│  AI 초안 거부율: 10%                                │
│  지식문서 기여: 8건 (이번 달)                       │
│                                                     │
│  ■ 팀 내 나의 위치 (익명화된 비교)                  │
│  응답시간: 팀 평균 대비 20% 빠름                    │
│  처리량: 팀 상위 25%                                │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 4. 백엔드 API 변경

### 4.1 대시보드 API 확장

**파일:** `app/api/admin.py` — `GET /dashboard` (Line 88)

```python
@router.get("/dashboard")
async def dashboard(
    db: AsyncSession = Depends(get_db),
    current_user = Depends(get_current_user_optional)
):
    user_role = current_user.role if current_user else 'viewer'

    if user_role == 'agent':
        return await _agent_dashboard(current_user.id, db)
    elif user_role == 'viewer':
        return await _viewer_dashboard(db)
    else:  # admin
        return await _admin_dashboard(db)

async def _agent_dashboard(agent_id: int, db: AsyncSession):
    """에이전트 전용 대시보드 — 내 담당 메일 기준"""
    return {
        # 내 KPI
        "my_open_count": ...,         # 내 미처리 건수
        "my_today_processed": ...,    # 오늘 내가 처리한 건수
        "my_avg_response_time": ...,  # 내 평균 응답시간
        "my_sla_breached": ...,       # 내 SLA 위반 건수

        # 내 할 일 (Action Required)
        "sla_urgent": [...],          # SLA 임박 메일 목록
        "pending_approvals": [...],   # 내가 승인해야 할 건
        "new_assigned": [...],        # 신규 배정된 건

        # 개인 메일함 요약
        "personal_unread": ...,       # 읽지 않은 개인 메일
        "personal_ai_drafts": ...,    # AI 초안 대기

        # 내 7일 성과
        "my_weekly_stats": [...],     # 일별 처리건수, 응답시간

        # 내 최근 활동
        "my_recent_activities": [...],
    }

async def _viewer_dashboard(db: AsyncSession):
    """Viewer 대시보드 — 읽기 전용, 빠른 실행 없음"""
    data = await _admin_dashboard(db)
    data.pop("quick_actions", None)  # 빠른 실행 제거
    return data
```

### 4.2 통계 API 확장

**파일:** `app/api/admin.py` — `GET /dashboard/statistics` (Line 261)

```python
@router.get("/dashboard/statistics")
async def statistics(
    days: int = Query(30),
    brand: str = Query(None),
    agent_id: int = Query(None),
    db: AsyncSession = Depends(get_db),
    current_user = Depends(get_current_user_optional)
):
    user_role = current_user.role if current_user else 'viewer'

    # Agent는 자신의 통계만 볼 수 있음
    if user_role == 'agent':
        agent_id = current_user.id  # 강제로 본인 ID

    return {
        # 기존 분포 데이터
        "status_distribution": ...,
        "priority_distribution": ...,
        "send_type_distribution": ...,
        "brand_distribution": ...,

        # 추가: 일별 추이 (라인 차트)
        "daily_trend": [
            {"date": "2026-03-01", "received": 42, "replied": 35, "auto_sent": 15},
            ...
        ],

        # 추가: 에이전트별 성과 (admin만)
        "agent_performance": [...] if user_role == 'admin' else None,

        # 추가: 카테고리별 분석
        "category_breakdown": [...],

        # 추가: 응답시간 분포
        "response_time_histogram": [...],

        # 추가: AI 성과
        "ai_stats": {
            "avg_confidence": 0.82,
            "auto_send_rate": 0.45,
            "edit_rate": 0.23,
            "knowledge_usage_rate": 0.67,
        },

        # 추가: 이전 기간 대비
        "comparison": {
            "received_change_pct": 12.3,
            "replied_change_pct": 8.1,
            "response_time_change_pct": -15.2,
        },

        # 내 성과 (agent인 경우)
        "my_stats": {
            "processed_count": 45,
            "avg_response_time": 72,  # minutes
            "sla_compliance": 0.95,
            "ai_acceptance_rate": 0.68,
            "knowledge_contributions": 8,
            "team_percentile": 75,
        } if user_role == 'agent' else None,
    }
```

### 4.3 CSV 내보내기 확장

**파일:** `app/api/admin.py` — `GET /emails/export`

```python
@router.get("/statistics/export")
async def export_statistics(
    days: int = Query(30),
    brand: str = Query(None),
    format: str = Query('csv', regex='^(csv|json)$'),
    current_user = Depends(require_role('admin')),
    db: AsyncSession = Depends(get_db)
):
    """통계 데이터 CSV/JSON 내보내기 (admin만)"""
    pass
```

---

## 5. Frontend 구현

### Task 5-1: 역할별 대시보드 컴포넌트 분리

**파일:** `admin/src/app/dashboard/page.tsx`

```tsx
export default function DashboardPage() {
  const { user } = useAuth();
  const role = user?.role || 'viewer';

  switch (role) {
    case 'admin':
      return <AdminDashboard />;
    case 'agent':
      return <AgentDashboard />;
    case 'viewer':
      return <ViewerDashboard />;
  }
}
```

**신규 컴포넌트:**
- `components/dashboard/admin-dashboard.tsx` — 기존 대시보드 리팩토링
- `components/dashboard/agent-dashboard.tsx` — 내 업무 중심
- `components/dashboard/viewer-dashboard.tsx` — 읽기 전용
- `components/dashboard/action-required-card.tsx` — Agent 할일 카드
- `components/dashboard/personal-inbox-summary.tsx` — 개인 메일 요약
- `components/dashboard/my-weekly-chart.tsx` — 개인 7일 성과

### Task 5-2: 통계 페이지 확장

**파일:** `admin/src/app/statistics/page.tsx`

**추가 컴포넌트:**
- `components/stats/period-selector.tsx` — 기간 선택기 (7/30/90/사용자정의)
- `components/stats/brand-filter.tsx` — 브랜드 필터
- `components/stats/agent-filter.tsx` — 에이전트 필터 (admin만)
- `components/stats/daily-trend-chart.tsx` — 일별 추이 라인 차트
- `components/stats/agent-performance-table.tsx` — 에이전트별 성과 테이블
- `components/stats/category-breakdown.tsx` — 카테고리 바 차트
- `components/stats/response-time-histogram.tsx` — 응답시간 분포
- `components/stats/ai-performance-card.tsx` — AI 성과 지표
- `components/stats/export-button.tsx` — CSV 내보내기 버튼
- `components/stats/my-stats-card.tsx` — 개인 성과 (agent용)

### Task 5-3: 대시보드 API 쿼리 훅

**파일:** `admin/src/lib/queries.ts`

```tsx
// 역할별 대시보드
export function useDashboard() {
  // 기존 — role에 따라 서버가 다른 데이터 반환
}

// 확장된 통계
export function useStatistics(params: {
  days?: number;
  brand?: string;
  agentId?: number;
}) {
  return useQuery({
    queryKey: ['statistics', params],
    queryFn: () => apiClient.get('/admin/dashboard/statistics', { params }),
  });
}

// 통계 내보내기
export function useExportStatistics() {
  return useMutation({
    mutationFn: (params) => apiClient.get('/admin/statistics/export', {
      params, responseType: 'blob'
    }),
  });
}
```

---

## 6. 의존성 그래프

```
Task 5-1 (역할별 대시보드 분리)
  ├─ 4.1 _agent_dashboard API
  ├─ AgentDashboard 컴포넌트
  ├─ ActionRequiredCard 컴포넌트
  └─ PersonalInboxSummary 컴포넌트

Task 5-2 (통계 페이지 확장)
  ├─ 4.2 statistics API 확장
  ├─ PeriodSelector, BrandFilter, AgentFilter
  ├─ DailyTrendChart, AgentPerformanceTable
  ├─ CategoryBreakdown, ResponseTimeHistogram
  └─ AIPerformanceCard, ExportButton

Task 5-3 (API 쿼리 훅)
  └─ queries.ts 확장
```

Phase 3 RBAC 완료 후 착수. Phase 4와 병렬 가능 (공유 의존성 없음).

---

## 7. 테스트 체크리스트

### 대시보드
- [ ] Admin 로그인 → 전체 KPI + 팀 워크로드 + 빠른 실행 표시
- [ ] Agent 로그인 → 내 KPI + 내 할일 + 개인 메일 요약 표시
- [ ] Viewer 로그인 → 전체 KPI 표시, 빠른 실행 숨김
- [ ] Agent 대시보드의 "내 할 일" 클릭 → 해당 메일로 이동
- [ ] 자동 갱신 동작 확인

### 통계
- [ ] 기간 선택 (7/30/90일) → 데이터 갱신
- [ ] 브랜드 필터 → 해당 브랜드만 집계
- [ ] Agent는 agent_id 필터 본인으로 고정
- [ ] CSV 내보내기 → 파일 다운로드
- [ ] Agent 통계에 "팀 내 나의 위치" 표시

---

## 8. 구현 순서 (3일)

```
Day 1: 대시보드 API 분기 (admin/agent/viewer) + Agent 대시보드 컴포넌트
Day 2: 통계 API 확장 (필터, 추이, 에이전트별) + 통계 페이지 확장
Day 3: CSV 내보내기 + 테스트 + 자동갱신 최적화
```
