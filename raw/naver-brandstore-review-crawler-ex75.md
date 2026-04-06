"""
네이버 브랜드스토어 리뷰 크롤러
상품: Keychron eX75 HE 8K TMR (상품번호: 11995080560)
사용법: pip install openpyxl requests && python naver_review_crawler.py
결과: reviews_11995080560.xlsx + .json

PRODUCT_NO와 BRAND_STORE_ID를 변경하면 다른 상품에도 사용 가능.
merchantNo 자동 감지 → 실패 시 수동 입력.
API 엔드포인트 3개 자동 시도 (brand API v1, v2, smartstore).
"""

import requests
import json
import time
import re
import os
from datetime import datetime

PRODUCT_NO = "11995080560"
BRAND_STORE_ID = "keychron"
OUTPUT_DIR = os.path.dirname(os.path.abspath(__file__))

HEADERS = {
    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) "
                  "AppleWebKit/537.36 (KHTML, like Gecko) "
                  "Chrome/124.0.0.0 Safari/537.36",
    "Referer": f"https://m.brand.naver.com/{BRAND_STORE_ID}/products/{PRODUCT_NO}",
    "Accept": "application/json, text/plain, */*",
}

SESSION = requests.Session()
SESSION.headers.update(HEADERS)

def detect_merchant_no():
    urls = [
        f"https://m.brand.naver.com/{BRAND_STORE_ID}/products/{PRODUCT_NO}",
        f"https://brand.naver.com/{BRAND_STORE_ID}/products/{PRODUCT_NO}",
    ]
    for url in urls:
        try:
            resp = SESSION.get(url, timeout=15)
            if resp.status_code == 200:
                match = re.search(r'"merchantNo"\s*:\s*"?(\d+)"?', resp.text)
                if match:
                    print(f"  ✅ merchantNo 자동 감지: {match.group(1)}")
                    return match.group(1)
        except Exception as e:
            print(f"  [WARN] {url}: {e}")
    try:
        resp = SESSION.get(f"https://m.brand.naver.com/{BRAND_STORE_ID}", timeout=10)
        if resp.status_code == 200:
            match = re.search(r'"merchantNo"\s*:\s*"?(\d+)"?', resp.text)
            if match:
                return match.group(1)
    except: pass
    return None

def get_product_info(merchant_no):
    for url in [
        f"https://m.brand.naver.com/api/product/{PRODUCT_NO}",
        f"https://m.brand.naver.com/api/v1/products/{PRODUCT_NO}?merchantNo={merchant_no}",
    ]:
        try:
            resp = SESSION.get(url, timeout=10)
            if resp.status_code == 200: return resp.json()
        except: continue
    return None

def fetch_reviews(merchant_no, page=1, page_size=20, sort="REVIEW_CREATE_DATE_DESC"):
    endpoints = [
        ("https://m.brand.naver.com/api/review/product", {
            "merchantNo": merchant_no, "originProductNo": PRODUCT_NO,
            "page": page, "pageSize": page_size, "sortType": sort,
        }),
        (f"https://m.brand.naver.com/api/v2/review/product/{PRODUCT_NO}", {
            "merchantNo": merchant_no, "page": page, "pageSize": page_size, "sortType": sort,
        }),
        ("https://smartstore.naver.com/i/v1/reviews/paged-reviews", {
            "originProductNo": PRODUCT_NO, "merchantNo": merchant_no,
            "page": page, "pageSize": page_size, "sortType": sort,
        }),
    ]
    for url, params in endpoints:
        try:
            resp = SESSION.get(url, params=params, timeout=15)
            if resp.status_code == 200:
                data = resp.json()
                if extract_reviews(data) is not None: return data
        except: continue
    return None

def extract_reviews(data):
    if not isinstance(data, dict): return None
    for path in [["contents"],["reviews"],["data","contents"],["data","reviews"],["result","contents"]]:
        obj = data
        for k in path:
            obj = obj.get(k) if isinstance(obj, dict) else None
        if isinstance(obj, list): return obj
    return None

def get_total(data):
    if not isinstance(data, dict): return 0
    for k in ["totalCount","totalElements","total"]:
        if k in data: return data[k]
        if isinstance(data.get("data"), dict) and k in data["data"]: return data["data"][k]
    return 0

def parse_review(r):
    images = r.get("reviewImages") or r.get("images") or r.get("attachFiles") or []
    img_urls = [u for u in [i.get("url") or i.get("imageUrl") or i.get("filePath","") for i in images if i] if u]
    reply = r.get("sellerReply") or r.get("reply") or {}
    reply_text = reply.get("content","") if isinstance(reply, dict) else ""
    reply_date = (reply.get("createDate") or reply.get("createdDate","")) if isinstance(reply, dict) else ""
    cdate = r.get("createDate") or r.get("createdDate") or r.get("reviewCreatedDate") or ""
    if isinstance(cdate, (int,float)):
        try: cdate = datetime.fromtimestamp(cdate/1000).strftime("%Y-%m-%d %H:%M:%S")
        except: pass
    return {
        "리뷰ID": r.get("id") or r.get("reviewId") or "",
        "닉네임": r.get("writerNickname") or r.get("nickname") or r.get("userName") or "",
        "별점": r.get("reviewScore") or r.get("score") or r.get("starScore") or "",
        "내용": (r.get("reviewContent") or r.get("content") or r.get("body") or "").strip(),
        "작성일": cdate,
        "옵션": r.get("productOptionContent") or r.get("optionText") or "",
        "좋아요": r.get("helpCount") or r.get("sympathyCount") or 0,
        "포토": "Y" if img_urls else "N",
        "이미지수": len(img_urls),
        "이미지URLs": " | ".join(img_urls),
        "판매자답변": reply_text,
        "답변일": reply_date,
    }

def crawl_all(merchant_no):
    all_reviews = []; page, page_size, total = 1, 20, None
    print(f"\n{'='*55}\n  Keychron eX75 HE 8K TMR 리뷰 크롤링\n  상품번호: {PRODUCT_NO} | merchantNo: {merchant_no}\n{'='*55}\n")
    while True:
        print(f"  📥 p.{page}", end=" ", flush=True)
        data = fetch_reviews(merchant_no, page=page, page_size=page_size)
        if not data: print("❌"); break
        reviews = extract_reviews(data)
        if reviews is None:
            print("❌ 파싱실패")
            with open(os.path.join(OUTPUT_DIR,"debug_response.json"),"w",encoding="utf-8") as f:
                json.dump(data, f, ensure_ascii=False, indent=2)
            break
        if total is None:
            total = get_total(data); print(f"(총 {total}건) → {len(reviews)}건")
        else: print(f"→ {len(reviews)}건")
        if not reviews: break
        all_reviews.extend(parse_review(r) for r in reviews)
        if len(reviews) < page_size or (total and len(all_reviews) >= total): break
        page += 1; time.sleep(0.3)
    print(f"\n  ✅ {len(all_reviews)}건 수집 완료\n"); return all_reviews

def save_excel(reviews, product_no):
    try:
        import openpyxl
        from openpyxl.styles import Font, Alignment, PatternFill, Border, Side
    except ImportError:
        print("  ⚠️ pip install openpyxl → CSV fallback"); return save_csv(reviews, product_no)
    fn = os.path.join(OUTPUT_DIR, f"reviews_{product_no}.xlsx")
    wb = openpyxl.Workbook(); ws = wb.active; ws.title = "리뷰"
    if not reviews: wb.save(fn); return fn
    headers = list(reviews[0].keys())
    hf = PatternFill(start_color="1A1A1A",end_color="1A1A1A",fill_type="solid")
    hfont = Font(name="맑은 고딕",bold=True,color="FFFFFF",size=10)
    bd = Border(*(Side(style="thin",color="DDDDDD"),)*4)
    for c,h in enumerate(headers,1):
        cell=ws.cell(1,c,h); cell.fill=hf; cell.font=hfont
        cell.alignment=Alignment(horizontal="center"); cell.border=bd
    for ri,rev in enumerate(reviews,2):
        for ci,k in enumerate(headers,1):
            cell=ws.cell(ri,ci,rev.get(k,""))
            cell.font=Font(name="맑은 고딕",size=9); cell.border=bd
            if k=="내용": cell.alignment=Alignment(wrap_text=True,vertical="top")
    widths={"리뷰ID":12,"닉네임":12,"별점":6,"내용":60,"작성일":18,"옵션":28,"좋아요":7,"포토":5,"이미지수":7,"이미지URLs":40,"판매자답변":40,"답변일":18}
    for c,h in enumerate(headers,1):
        ws.column_dimensions[openpyxl.utils.get_column_letter(c)].width=widths.get(h,14)
    ws.auto_filter.ref=ws.dimensions; ws.freeze_panes="A2"
    ws2=wb.create_sheet("요약")
    scores=[r["별점"] for r in reviews if isinstance(r["별점"],(int,float))]
    avg=f"{sum(scores)/len(scores):.2f}" if scores else "N/A"
    rows=[("상품","Keychron eX75 HE 8K TMR"),("상품번호",product_no),("총 리뷰",len(reviews)),("평균 별점",avg),("",""),
          *[(f"{s}점",scores.count(s)) for s in range(5,0,-1)],("",""),
          ("포토리뷰",sum(1 for r in reviews if r["포토"]=="Y")),
          ("판매자답변",sum(1 for r in reviews if r["판매자답변"])),
          ("수집일시",datetime.now().strftime("%Y-%m-%d %H:%M:%S"))]
    ws2.cell(1,1,"항목").font=hfont; ws2.cell(1,1).fill=hf
    ws2.cell(1,2,"값").font=hfont; ws2.cell(1,2).fill=hf
    for i,(k,v) in enumerate(rows,2):
        ws2.cell(i,1,k).font=Font(name="맑은 고딕",size=10)
        ws2.cell(i,2,v).font=Font(name="맑은 고딕",size=10)
    ws2.column_dimensions["A"].width=14; ws2.column_dimensions["B"].width=30
    wb.save(fn); print(f"  💾 {fn}"); return fn

def save_csv(reviews, product_no):
    import csv
    fn=os.path.join(OUTPUT_DIR,f"reviews_{product_no}.csv")
    if not reviews: return None
    with open(fn,"w",newline="",encoding="utf-8-sig") as f:
        w=csv.DictWriter(f,fieldnames=reviews[0].keys()); w.writeheader(); w.writerows(reviews)
    print(f"  💾 {fn}"); return fn

def save_json(reviews, product_no):
    fn=os.path.join(OUTPUT_DIR,f"reviews_{product_no}.json")
    with open(fn,"w",encoding="utf-8") as f:
        json.dump(reviews,f,ensure_ascii=False,indent=2)
    print(f"  💾 {fn}"); return fn

def print_stats(reviews):
    scores=[r["별점"] for r in reviews if isinstance(r["별점"],(int,float))]
    if not scores: return
    print(f"  📊 별점 분포 ({len(scores)}건)\n  {'─'*40}")
    for s in range(5,0,-1):
        cnt=scores.count(s); pct=cnt/len(scores)*100
        print(f"  {s}점  {'█'*int(pct/2)} {cnt}건 ({pct:.1f}%)")
    print(f"  {'─'*40}\n  평균: {sum(scores)/len(scores):.2f}점 | 포토: {sum(1 for r in reviews if r['포토']=='Y')}건\n")

if __name__ == "__main__":
    print("\n  🔍 merchantNo 감지 중...")
    merchant_no = detect_merchant_no()
    if not merchant_no:
        print(f"\n  ⚠️ 자동 감지 실패!\n  📋 수동 확인:\n     크롬 → https://m.brand.naver.com/{BRAND_STORE_ID}/products/{PRODUCT_NO}\n     F12 → Network → 'review' 검색 → merchantNo 값 복사")
        merchant_no = input("\n  merchantNo 입력: ").strip()
        if not merchant_no: exit(1)
    info = get_product_info(merchant_no)
    if info: print(f"  📦 {info.get('name') or info.get('productName','eX75 HE 8K TMR')}")
    reviews = crawl_all(merchant_no)
    if reviews:
        print_stats(reviews); save_excel(reviews, PRODUCT_NO); save_json(reviews, PRODUCT_NO)
        print(f"  🎉 완료!\n")
    else:
        print("  ⚠️ 수집 실패. merchantNo를 수동 확인해주세요.\n")