---
title: Claudex
tags: [오픈소스, Claude Code, GPT, LLM]
created: 2026-04-06
updated: 2026-04-06
sources: [claudex-project-readme.md, plan-sunny-puzzling-aurora.md]
---
# Claudex

## 개요

Claudex는 **OpenClaude + Codex**의 합성어로, Claude Code의 에이전틱 도구 시스템(Bash, Read, Write, Edit, Grep, Glob, Agent, MCP)을 GPT-5.4(Codex) 또는 다른 LLM으로 구동하는 프로젝트다. GitHub에서 `woohoyang-oss/claudex`로 공개되어 있다.

## 아키텍처

도구 호출 흐름은 다음과 같다:

```
Claude Code Tools → openaiShim(포맷 변환) → codexShim(스키마 정규화) → Codex API / Ollama / OpenAI
```

Claude Code의 도구 호출 형식을 OpenAI 호환 포맷으로 변환하는 **openaiShim**과, Codex API의 strict 모드에 맞춰 스키마를 정규화하는 **codexShim** 두 레이어로 구성된다.

## 패치 내역 (2개 파일)

1. **openaiShim.ts**: thinking 모델(qwen3, DeepSeek-R1)의 `reasoning`/`reasoning_content` 필드 처리 추가
2. **codexShim.ts**: `enforceStrictSchema()` 함수로 Codex API strict 모드 대응 — 필수 필드 누락 방지, 금지 키워드 제거, nullable optional 처리

## 지원 모델

| 모델명 | 설명 |
|--------|------|
| `codexplan` | GPT-5.4 (ChatGPT Plus $20/mo로 사용 가능) |
| `codexspark` | GPT-5.3 Codex Spark |
| `qwen3:14b` | Ollama 로컬 실행 (무료) |

## 빌드 배경

원래 Claw Code(Claude Code의 Rust 오픈소스 클론)에서 출발하여, OpenAI 호환 프로바이더가 이미 내장되어 있음을 확인한 뒤 환경변수 설정만으로 Codex 백엔드 연결이 가능하도록 발전시켰다. 셸 에일리어스(`claudex`)를 등록하면 터미널에서 바로 사용할 수 있다.

## 관련 프로젝트

- [[akaxa-space]] — Claudex와 함께 크로스AI 워크플로에서 활용 가능
- [[tobe-mcp]] — MCP 도구를 Claudex에서도 구동 가능

## 관련 소스

- `claudex-project-readme.md` — 프로젝트 README 및 퀵스타트
- `plan-sunny-puzzling-aurora.md` — Claw Code 빌드 + OpenAI 백엔드 연결 계획
