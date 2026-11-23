# A5 脚本模板：入选文献 PDF 下载

复制到项目目录（如 `code/a5_download.py`）后，只改配置区再运行。

| 配置项 | 说明 |
|--------|------|
| `WORK_DIR` | A3/A4 产出目录 `output/downloads/{theme}_{timestamp}` |
| `OUTPUT_BASE` | 输出根目录（标记文件位置），默认 `output` |
| `SMOKE_TEST` | 冒烟：只下载入选第 1 篇 |

流程：逐篇 `detail_url` 直达详情页（无 href 则标题检索降级）→ `a#pdfDown` 下载 → 三道校验 → 重命名 `{year}-{journal}-{title}.pdf` → 回写 `selected.jsonl`。不需要 LLM。

参考：[phase3_pipeline.md](../crawler/phase3_pipeline.md) § A5、[cnki_session.md](../crawler/cnki_session.md)、[cnki_locators.md](../crawler/cnki_locators.md)。

```python
# -*- coding: utf-8 -*-
"""
A5: 入选文献 PDF 下载(详情页 a#pdfDown; CAJ-only 记 caj_skipped.md)
输入: WORK_DIR/selected.jsonl
产出: WORK_DIR/papers/*.pdf, download_log.csv, caj_skipped.md, selected.jsonl 回写 downloaded_filename
"""
import sys, os, re, json, time, random, csv
sys.stdout.reconfigure(encoding='utf-8')
sys.stderr.reconfigure(encoding='utf-8')

import fitz  # PyMuPDF
from playwright.sync_api import sync_playwright
from playwright.sync_api import TimeoutError as PlaywrightTimeout

# ============================================================
# 配置（★ 每次运行前修改）
# ============================================================
WORK_DIR = "output/downloads/<theme_timestamp>"
OUTPUT_BASE = "output"
SMOKE_TEST = False
SMOKE_LIMIT = 1
DOWNLOAD_TIMEOUT = 60000  # 毫秒

PROFILE_DIR = os.path.join(os.path.dirname(os.path.abspath(__file__)), "edge_profile")

def polite_sleep(lo=3, hi=6):
    time.sleep(random.uniform(lo, hi))

def wait_marker_cleared(marker_path, timeout=600, label="登录"):
    open(marker_path, "w").write("waiting")
    print(f"等待用户{label}... 标记: {marker_path}", flush=True)
    t0 = time.time()
    while os.path.exists(marker_path):
        if time.time() - t0 > timeout:
            return False
        time.sleep(2)
    return True

def detect_captcha(page):
    """两类验证码(均实测): 腾讯拼图(容器常驻,opacity>0 才算) + 点选文字弹窗(可见且带尺寸)。
    纯文本匹配会命中常驻模板,必须配合可见性/尺寸/透明度判断。"""
    try:
        return page.evaluate("""() => {
          const t = document.querySelector('#tcaptcha_transform_dy');
          if (t) {
            const cs = getComputedStyle(t), r = t.getBoundingClientRect();
            if (cs.display !== 'none' && parseFloat(cs.opacity) > 0.01 && r.width > 50) return true;
          }
          const els = Array.from(document.querySelectorAll('div, p, span'));
          return els.some(e => {
            const tx = (e.innerText || '').trim();
            if (!/请完成安全验证|请依次点击/.test(tx) || tx.length > 60) return false;
            const r = e.getBoundingClientRect();
            return e.offsetParent !== null && r.width > 50 && r.height > 10;
          });
        }""")
    except Exception:
        return False

def safe_filename(s, maxlen=80):
    s = re.sub(r'[\\/:*?"<>|\r\n]+', "_", s or "").strip().strip(".")
    return s[:maxlen] or "untitled"

def validate_pdf(path):
    """三道校验: %PDF 头 / 大小 >10KB / 页数 >=2"""
    try:
        with open(path, "rb") as f:
            if f.read(4) != b"%PDF":
                return False, "非PDF(可能CAJ或错误页)"
        if os.path.getsize(path) < 10 * 1024:
            return False, "文件过小"
        with fitz.open(path) as doc:
            if doc.page_count < 2:
                return False, "页数<2"
        return True, "ok"
    except Exception as e:
        return False, f"校验异常:{e}"

def ensure_login(page):
    page.goto("https://kns.cnki.net/kns8s/AdvSearch", wait_until="domcontentloaded", timeout=30000)
    time.sleep(4)
    if page.locator("input.btn-search").count():
        print("[A5] 高级检索页可用,视为具备权限", flush=True)
        return True
    return wait_marker_cleared(os.path.join(OUTPUT_BASE, ".wait_login"), label="登录")

def goto_detail_by_title(page, title):
    """降级路径: 高级检索篇名精确检索 → 第一行标题链接 href"""
    page.goto("https://kns.cnki.net/kns8s/AdvSearch", wait_until="domcontentloaded", timeout=30000)
    time.sleep(4)
    # 第一行字段: 主题 -> 篇名(行内属性选择器,篇名=TI)
    row = page.locator("dl.inputs-list:visible > dd").first
    cur = row.locator("div.sort.reopt .sort-default span").first.get_attribute("value")
    if cur != "TI":
        row.locator("div.sort.reopt").first.click(timeout=5000)
        time.sleep(1.5)
        row.locator("li[data-val='TI'] a").first.click(timeout=5000)
        time.sleep(1)
    row.get_by_role("textbox").first.fill(title)  # textbox 角色排除 hidden input
    page.locator("input.btn-search").click(timeout=5000)
    page.wait_for_selector("table.result-table-list tbody tr", timeout=20000)
    a = page.locator("table.result-table-list tbody tr td:nth-child(2) a").first
    found_title = a.inner_text().strip()
    if found_title.replace(" ", "") != title.replace(" ", ""):
        print(f"  [A5] 标题不完全一致: 期望 {title[:30]!r} 实得 {found_title[:30]!r},仍采用首个结果", flush=True)
    return a.get_attribute("href")

def download_one(ctx, page, rec, papers_dir):
    """返回 (status, detail)。status: ok / caj_only / fail"""
    url = rec.get("detail_url") or ""
    if not url:
        try:
            url = goto_detail_by_title(page, rec["title"])
        except Exception as e:
            return "fail", f"标题检索失败:{type(e).__name__}"
    try:
        page.goto(url, wait_until="domcontentloaded", timeout=30000)
        time.sleep(3)
    except Exception as e:
        return "fail", f"详情页打开失败:{type(e).__name__}"

    if detect_captcha(page):
        if not wait_marker_cleared(os.path.join(OUTPUT_BASE, ".wait_captcha"), label="过验证码"):
            return "fail", "验证码超时"
        page.reload(wait_until="domcontentloaded")
        time.sleep(3)

    if page.locator("a#pdfDown").count() == 0:
        if page.locator("a#cajDown").count():
            return "caj_only", url
        return "fail", "详情页无 PDF/CAJ 按钮"

    fname = safe_filename(f"{rec.get('year') or '0000'}-{rec.get('journal') or '未知'}-{rec.get('title','')}") + ".pdf"
    dst = os.path.join(papers_dir, fname)
    for attempt in range(2):  # 失败重走完整流程重试 1 次
        try:
            with page.expect_download(timeout=DOWNLOAD_TIMEOUT) as dlinfo:
                page.locator("a#pdfDown").first.click()
            dl = dlinfo.value
            dl.save_as(dst)
            ok, why = validate_pdf(dst)
            if ok:
                return "ok", fname
            if os.path.exists(dst):
                os.remove(dst)
            print(f"  [A5] 校验失败({why}),{'重试' if attempt == 0 else '放弃'}", flush=True)
            if attempt == 0:
                page.goto(url, wait_until="domcontentloaded", timeout=30000)
                time.sleep(3)
        except Exception as e:
            print(f"  [A5] 下载异常({type(e).__name__})", flush=True)
    return "fail", "下载/校验失败"

def main():
    selected_path = os.path.join(WORK_DIR, "selected.jsonl")
    papers_dir = os.path.join(WORK_DIR, "papers")
    os.makedirs(papers_dir, exist_ok=True)
    records = [json.loads(l) for l in open(selected_path, encoding="utf-8")]
    todo = [r for r in records if not r.get("downloaded_filename")]
    if SMOKE_TEST:
        todo = todo[:SMOKE_LIMIT]
    print(f"[A5] 入选 {len(records)},待下载 {len(todo)}(断点续跑跳过已下载)", flush=True)

    stats = {"ok": 0, "caj_only": 0, "fail": 0}
    log_path = os.path.join(WORK_DIR, "download_log.csv")
    caj_path = os.path.join(WORK_DIR, "caj_skipped.md")

    with sync_playwright() as pw:
        ctx = pw.chromium.launch_persistent_context(
            PROFILE_DIR, channel="msedge", headless=False,
            viewport={"width": 1920, "height": 1080}, accept_downloads=True,
        )
        page = ctx.pages[0] if ctx.pages else ctx.new_page()
        try:
            if not ensure_login(page):
                print("[A5] 登录等待超时,退出", flush=True)
                return 1
            for i, rec in enumerate(todo, 1):
                print(f"\n[A5] [{i}/{len(todo)}] {rec.get('title','')[:40]}", flush=True)
                status, detail = download_one(ctx, page, rec, papers_dir)
                stats[status] = stats.get(status, 0) + 1
                print(f"  -> {status}: {detail}", flush=True)
                # 回写
                rec["download_status"] = status
                if status == "ok":
                    rec["downloaded_filename"] = detail
                with open(log_path, "a", encoding="utf-8", newline="") as f:
                    csv.writer(f).writerow([rec.get("id"), rec.get("title"), status, detail, time.strftime("%F %T")])
                if status == "caj_only":
                    # 去重后追加(断点续跑/多轮运行可能重复遇到同一篇)
                    existing = open(caj_path, encoding="utf-8").read() if os.path.exists(caj_path) else ""
                    if rec.get("title") not in existing:
                        with open(caj_path, "a", encoding="utf-8") as f:
                            f.write(f"- {rec.get('title')} | {rec.get('journal')} | {rec.get('year')} | {detail}\n")
                # 内存中回写 selected.jsonl(整文件覆写,保持单源)
                with open(selected_path, "w", encoding="utf-8") as f:
                    for r in records:
                        f.write(json.dumps(r, ensure_ascii=False) + "\n")
                polite_sleep()
                if i % 20 == 0:
                    print(f"  [A5] 已完成 {i} 篇,长休 30s", flush=True)
                    time.sleep(30)
        finally:
            ctx.close()

    print(f"\n[A5] ✓ 完成: 成功 {stats['ok']}, CAJ跳过 {stats['caj_only']}, 失败 {stats['fail']}", flush=True)
    if stats["ok"] == 0 and len(todo) > 0:
        print("[A5] ⚠️ 0 成功(规则10): 检查登录态/权限/caj 占比", flush=True)
    return 0

if __name__ == "__main__":
    sys.exit(main())
```
