---
title: Claude Code 계획 파일 요약
tags: [계획, Claude Code, rhkorea, auto-dm, kychr.space, claw-code]
created: 2026-04-06
updated: 2026-04-06
sources: [plan-compiled-soaring-iverson.md, plan-frolicking-whistling-bubble.md, plan-frolicking-whistling-bubble-agent-a6d8333528682b720.md, plan-gleaming-humming-jellyfish.md, plan-happy-tumbling-crown.md, plan-polished-roaming-flame.md, plan-redesign-dashboard-plan.md, plan-sunny-puzzling-aurora.md]
---
# Claude Code 계획 파일 요약

Claude Code 세션에서 생성된 프로젝트 계획 파일 8건을 정리한다. 각 계획은 특정 프로젝트의 구현 로드맵을 담고 있다.

---

## 계획 목록

| 파일명 | 프로젝트 | 핵심 내용 |
|--------|----------|-----------|
| plan-compiled-soaring-iverson | [[rhkorea.com]] 리부트 | Flask+MySQL 커뮤니티에 글쓰기, 댓글, 인증, AI BGM 추천 추가. 6단계 구현 |
| plan-frolicking-whistling-bubble | [[Auto-DM]] 서비스 | ManyChat 대체 자체 DM 자동응답. 키크론 인스타 봇 1단계 MVP |
| plan-frolicking-whistling-bubble-agent | Auto-DM 오픈소스 리서치 | GitHub 15+ 프로젝트 조사. FUMA, SaaS-Instagram-DM-Automations 등 분석 |
| plan-gleaming-humming-jellyfish | [[kychr.space]] API 키 인증 | 리전 매니저용 Bearer 토큰 인증 + API 가이드 추가 |
| plan-happy-tumbling-crown | [[rhkorea.com]] Phase 1 | Next.js 15 리부트. akaxa OTP 인증, 다크 모드, YouTube 임베드, 7단계 구현 |
| plan-polished-roaming-flame | [[Claw Code]] 빌드 | Claude Code Rust 클론을 OpenAI 백엔드로 연결. 환경변수 설정만으로 동작 |
| plan-redesign-dashboard-plan | kychr.space 대시보드 재설계 (상세) | 현재 아키텍처 분석 + 뉴스페이퍼 스타일 재설계. InsightCard, 페이지네이션 상세 |
| plan-sunny-puzzling-aurora | kychr.space 대시보드 재설계 (요약) | 뉴스페이퍼 레이아웃, InsightCard 컴포넌트, 업로드 파이프라인 정리 |

---

## 프로젝트별 요약

### rhkorea.com (2건)
29년간의 브릿팝 커뮤니티 데이터(27,868글 + 70,111댓글)를 기반으로 리부트하는 계획. 초기 계획은 Flask 유지 + DB 확장이었으나, 이후 Next.js 15 전환으로 방향을 수정했다. DC식 익명 + akaxa OTP 이메일 인증의 2층 구조, AI BGM 추천(Claude CLI) 등이 특징이다.

### Auto-DM (2건)
ManyChat을 대체하는 인스타그램 DM 자동응답 시스템 기획. 키크론/지티기어/포스윙 브랜드별 봇 운영 구조. 오픈소스 리서치에서 Graph API 기반 프로젝트가 아직 소수임을 확인하고 시장 기회로 판단했다.

### kychr.space (3건)
시장 인텔리전스 대시보드의 API 키 인증 추가와 뉴스페이퍼 스타일 재설계. 상세 계획에서 현재 아키텍처, 데이터 흐름, Astro 컴포넌트 구조까지 분석했다.

### Claw Code (1건)
Claude Code의 Rust 오픈소스 클론을 빌드하여 OpenAI 모델(gpt-4o)을 백엔드로 사용하는 실험. 코드 수정 없이 환경변수만으로 동작 가능함을 확인했다.
