---
title: kychr.space - 키크론 마켓 인텔리전스 대시보드
tags: [kychr-space, 대시보드, Astro, 인사이트, 키크론]
created: 2026-04-06
updated: 2026-04-06
sources: [kychr-space-project.md, kychr-space-CLAUDE.md, plan-redesign-dashboard-plan.md, plan-polished-roaming-flame.md]
---
# kychr.space - 키크론 마켓 인텔리전스 대시보드

## 개요

kychr.space는 키크론 글로벌 디스트리뷰터를 위한 마켓 인텔리전스 대시보드이다. 디스트리뷰터가 시장 데이터(가격 분석, 경쟁사 동향, VOC 등)를 업로드하면 Claude CLI가 자동으로 분석 리포트를 생성하고, 챗봇이 축적된 분석 데이터를 기반으로 질의응답을 제공한다.

GitHub: github.com/woohoyang-oss/kychr.space (private)
URL: https://kychr.space

## 기술 스택

- **프레임워크**: Astro 5.9 SSR + React + Tailwind CSS
- **서버**: EC2 t3.micro (ap-northeast-2, 13.124.196.73) + nginx + pm2
- **배포**: GitHub Actions (main push -> build -> rsync dist -> pm2 restart)
- **인증**: 이중 방식 - 브라우저 쿠키(kychr-auth) + API 키(Bearer kychr_...)
- **데이터 저장**: 파일시스템 (data/insights/) + index.json 메타데이터
- **분석 엔진**: Claude CLI (-p - stdin 방식)

## 인사이트 업로드 및 분석 파이프라인

1. **업로드**: POST /api/insights/upload으로 파일 저장 (HTML, XLSX, CSV, TSV, MD 지원, 최대 5MB)
2. **MD 분석**: Claude CLI가 {slug}.analysis.md 생성 (영문 상세 분석 리포트)
3. **인덱스 업데이트**: MD에서 영문 제목, 요약, 핵심 발견, 태그 추출 -> index.json 갱신
4. **챗봇 연동**: 모든 .analysis.md 파일을 지식 베이스로 자동 반영

### 언어 규칙
- 모든 생성 콘텐츠(제목, MD 분석, 요약, 태그): 영문 필수
- 한국어/일본어/중국어 원본 데이터는 영문으로 번역하여 분석
- 브랜드명은 원본 유지 (예: "키크론" -> "Keychron")
- 챗봇은 사용자 언어로 응답하되 저장 데이터는 항상 영문

## 주요 페이지

- `/` - 대시보드 (인사이트 카드 + GitHub 활동)
- `/insights` - 업로드 및 목록 페이지
- `/insights/[date]/[slug]` - 인사이트 상세 뷰어 (AI Brief + 원본 데이터)
- `/rapid-trigger` - 정적 시장 분석 페이지
- `/api/chat` - 챗봇 엔드포인트

## 디자인 시스템

라이트 웜 배경(#fcfcf9) 기반, 다크 모드 없음, 이너 스크롤 금지(풀페이지 스크롤만). rounded-[1.5rem] 카드, shadcn 스타일 KPI 카드, Chart.js 차트(브랜드별 컬러: Keychron #6366f1, Preflow #ec4899, Hansung #eab308 등).

## 리디자인 계획: 뉴스페이퍼 스타일

대시보드를 "Keychron Newspaper" 느낌으로 재구성하는 계획이 수립되어 있다.

- **Featured Insight**: 첫 번째 인사이트를 큰 카드로 표시 (요약 전문 + 핵심 발견)
- **그리드 카드**: 나머지 인사이트를 3열 그리드로 배치 (제목 + 요약 1줄 + 태그 + 날짜)
- **페이지네이션**: 페이지당 6개 (1 Featured + 5 Grid), ?page= 쿼리 파라미터
- **새 컴포넌트**: InsightCard.astro, PaginationControls.astro

## 현재 상태 (2026-04-06)

- 6개 인사이트 업로드 및 분석 완료
- API 키 인증 정상 작동
- iframe srcdoc로 원본 HTML 보존
- 한국어 -> 영문 자동 번역 파이프라인 동작 중
- 일괄 재분석 엔드포인트 구현

## 관련 시스템

- [[keychron-supply-tracker]] - 키크론 발주 추적 시스템

## 관련 소스

- [kychr-space-project.md](../../raw/kychr-space-project.md)
- [kychr-space-CLAUDE.md](../../raw/kychr-space-CLAUDE.md)
- [plan-redesign-dashboard-plan.md](../../raw/plan-redesign-dashboard-plan.md)
- [plan-polished-roaming-flame.md](../../raw/plan-polished-roaming-flame.md)
