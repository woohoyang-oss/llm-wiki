# Database Schema

## SQLite: sabangnet.db (WAL mode)
Path: `/Users/tobe/claude/openclaw/sabang/dashboard/sabangnet.db`

### products (15,389 rows)
핵심 상품 데이터. 사방넷 API 스냅샷에서 import.
```sql
CREATE TABLE products (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    gcode TEXT,           -- G-코드 (예: G-0029-0338-0001)
    prd_nm TEXT,          -- 상품명
    modl_nm TEXT,         -- 모델명
    brnd_nm TEXT,         -- 브랜드명
    orgpl_ntn TEXT,       -- 원산지
    status_cd TEXT,       -- 상태코드 (001=일시중지, 002=공급중, 003=완전품절, 004=대기중)
    status_nm TEXT,       -- 상태명
    sepr TEXT,            -- 판매가
    shma_nm TEXT,         -- 쇼핑몰명
    reg_date TEXT,        -- 등록일
    mod_date TEXT,        -- 수정일
    shop_prd_cd TEXT,     -- 쇼핑몰 상품코드
    settle_price TEXT,    -- 정산가
    snapshot_date TEXT,   -- 스냅샷 날짜
    raw_json TEXT         -- 원본 JSON
);
```

### 통계
- 총 15,389 rows
- G-코드 있는 것: 13,386 rows (2,123 unique G-codes)
- G-코드 없는 것: 2,003 rows (API 기본 필터로 제외)
- 상태 분포: 일시중지 7,532 / 공급중 5,648 / 완전품절 173 / 대기중 33
- 쇼핑몰: 20개

### batch_configs (2 rows)
```sql
-- 재입고 복구: 재고 20개 초과, 일시중지→공급중
-- 품절 처리: 재고 0개 이하, 공급중→일시중지
```

### batch_history
배치 실행 이력. batch_id, started_at, completed_at, total/success/failed count 등.

### snapshot_meta
스냅샷 메타정보 (total_gcodes, total_products, updated_at 등).

### tasks
비동기 태스크 (배치 실행, 스냅샷 등). status, result_json, logs 포함.
