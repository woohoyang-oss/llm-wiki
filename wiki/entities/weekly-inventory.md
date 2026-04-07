---
title: 주간 재고 리포트 자동화
tags: [keychron, inventory, automation, excel]
created: 2026-04-06
updated: 2026-04-06
sources: []
---
# 주간 재고 리포트 자동화

## 개요

매주 월요일 키크론 재고 현황을 자동으로 집계하여 Excel 리포트를 생성하고 Gmail로 발송하는 자동화 파이프라인이다.

## 파이프라인

```
fetch_data.sh (EC2)
    ↓
build_keychron_full.py (JSON → Excel)
    ↓
send_keychron_email.py (Gmail 발송)
```

1. **fetch_data.sh**: EC2에서 재고 데이터를 수집하여 JSON으로 저장
2. **build_keychron_full.py**: JSON 데이터를 Excel 파일로 변환
3. **send_keychron_email.py**: 생성된 Excel 파일을 Gmail API로 발송

## Excel 구성

- **Summary 시트**: 카테고리별 재고 집계 (키보드, 스위치, 키캡 등)
- **Inventory 시트**: SKU 단위 상세 재고. 재고 수준에 따라 색상 알림 적용 (품절 임박 빨강, 저재고 노랑 등)

## 관련 시스템

- [[keychron-supply-tracker]] - 재고/발주 관리 시스템
