# CNKI Playwright 代码片段速查

可复制的代码模板。网站相关的 URL、selector 见 [cnki_locators.md](cnki_locators.md)。

---

## 浏览器生命周期（持久化 profile，登录态跨会话保持）

```python
from playwright.sync_api import sync_playwright

PROFILE_DIR = ".playwright/exploration/edge_profile"  # 项目内持久化 profile

def main():
    pw = sync_playwright().start()
    ctx = pw.chromium.launch_persistent_context(
        PROFILE_DIR, channel="msedge", headless=False,
        viewport={"width": 1920, "height": 1080}, accept_downloads=True,
        args=["--no-proxy-server"],  # 本机常开 Clash 系统代理;此 flag 强制直连(2026-11-18 实测必需)
    )
    page = ctx.pages[0] if ctx.pages else ctx.new_page()
    try:
        # ... 业务逻辑 ...
        pass
    finally:
        ctx.close()
        pw.stop()
```

用 `launch_persistent_context` 而非 `launch` + `new_context`：知网登录态存于 profile，跨脚本运行免重复登录。

> ⚠️ 系统代理：Chromium 默认跟随 Windows 系统代理（实测走代理出口 47.129.x），导致丢失机构 IP 识别且风控验证码增多。`proxy={"server": "direct://"}` 写法无效（ERR_PROXY_CONNECTION_FAILED），必须用 `args=["--no-proxy-server"]`。详见 [shared/llm_client.md](../shared/llm_client.md) 系统代理陷阱一节。

---

## 登录/验证码协调 — 标记文件轮询

```python
import os, time

def wait_marker_cleared(marker_path, timeout=600, label="登录"):
    """写入标记文件并轮询其消失(由 Agent 在用户确认后删除)。返回 False=超时"""
    open(marker_path, "w").write("waiting")
    print(f"等待用户{label}... 标记: {marker_path}", flush=True)
    t0 = time.time()
    while os.path.exists(marker_path):
        if time.time() - t0 > 600:
            return False
        time.sleep(2)
    print(f"用户{label}已确认,继续", flush=True)
    return True

# 登录: wait_marker_cleared("output/.wait_login", label="登录")
# 验证码: wait_marker_cleared("output/.wait_captcha", label="过验证码")
```

```python
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
```

---

## 频率纪律

```python
import random, time

def polite_sleep(lo=2, hi=5):
    time.sleep(random.uniform(lo, hi))

# 每 20 篇下载长休:
# if idx % 20 == 0 and idx > 0: time.sleep(30)
```

---

## safe_eval — JS 查询/滚动

```python
def safe_eval(js_code):
    global _browser_closed
    if _browser_closed:
        raise RuntimeError("Browser closed")
    try:
        return _page.evaluate(js_code)
    except Exception as e:
        err_msg = str(e).lower()
        if "closed" in err_msg or "target" in err_msg or "disposed" in err_msg:
            _browser_closed = True
        raise

# 参数化用 f-string 嵌入，不传第二参数
idx = 3
safe_eval(f"document.querySelectorAll('table.result-table-list tbody tr')[{idx}]?.scrollIntoView({{block:'center'}})")
```

只用于纯 JS 查询/滚动。点击、选择用 Playwright 原生 API。**唯一例外**：readonly 日期框用 JS 赋值（见下）。

---

## readonly 日期框赋值（实测有效）

```python
def set_date_range(page, start="2021-01-01", end="2026-12-31"):
    page.evaluate(f"""() => {{
      const d0 = document.getElementById('datebox0'), d1 = document.getElementById('datebox1');
      d0.value = '{start}'; d1.value = '{end}';
      for (const d of [d0, d1]) {{
        d.dispatchEvent(new Event('input', {{bubbles: true}}));
        d.dispatchEvent(new Event('change', {{bubbles: true}}));
      }}
    }}""")
```

---

## 点击-验证-重试（知网偶发失灵的筛选控件）

```python
def click_and_verify(page, click_fn, verify_fn, retries=3, wait=3, label=""):
    """知网筛选类控件偶发失灵: 点击后必须验证效果,无效则重试。返回 False=最终失败"""
    for attempt in range(retries):
        try:
            click_fn()
            time.sleep(wait)
            if verify_fn():
                return True
            print(f"  [{label}] 第{attempt+1}次点击未生效,重试", flush=True)
        except Exception as e:
            print(f"  [{label}] 第{attempt+1}次异常: {type(e).__name__}", flush=True)
    return False

# 例: 期刊筛选 —— 验证 pagerTitleCell 计数变化
# click_and_verify(
#     page,
#     lambda: page.get_by_role("link", name=re.compile(r"^学术期刊")).click(timeout=8000),
#     lambda: "学术期刊" 筛选后总数变小,  # 实际: 比较点击前后 pagerTitleCell 文本
#     label="期刊筛选")
```

---

## 下载 — expect_download + 三道校验

```python
import os

def download_pdf(page, btn_selector, dst_path):
    """详情页 PDF 下载 + 校验。返回 'ok' | 'caj_only' | 'fail'"""
    if page.locator(btn_selector).count() == 0:
        return "caj_only" if page.locator("a#cajDown").count() else "fail"
    try:
        with page.expect_download(timeout=60000) as dl_info:  # DOWNLOAD_TIMEOUT 毫秒
            page.locator(btn_selector).first.click()
        dl = dl_info.value
        dl.save_as(str(dst_path))
    except Exception:
        return "fail"
    # 三道校验: %PDF 头 / 页数 / 大小
    try:
        with open(dst_path, "rb") as f:
            if f.read(4) != b"%PDF":
                os.remove(dst_path); return "fail"
        if os.path.getsize(dst_path) < 10 * 1024:
            os.remove(dst_path); return "fail"
        import fitz
        with fitz.open(dst_path) as doc:
            if doc.page_count < 2:
                os.remove(dst_path); return "fail"
    except Exception:
        return "fail"
    return "ok"
```

---

## 等待结果表渲染

```python
from playwright.sync_api import TimeoutError as PlaywrightTimeout

try:
    page.wait_for_selector("table.result-table-list tbody tr", timeout=20000)
except PlaywrightTimeout:
    # 可能是 0 结果;读 pagerTitleCell 区分
    total = page.locator("span.pagerTitleCell").inner_text() if page.locator("span.pagerTitleCell").count() else ""
    print(f"结果表未出现,计数文本: {total!r}(0 结果属正常,按规则10输出诊断)", flush=True)
```

---

## 导出页抓取（降级路径：滚动加载 + innerText）

首选路径是导出页「导出」按钮直接下完整文件；按钮失败时的降级：

```python
def scrape_export_page(exp_page):
    """导出页滚动加载至记录数稳定,返回 innerText"""
    import re, time
    prev, stable = -1, 0
    while stable < 3:
        exp_page.evaluate("() => window.scrollTo(0, document.body.scrollHeight)")
        time.sleep(2)
        n = len(re.findall(r"^RT ", exp_page.evaluate("() => document.body.innerText"), re.M))
        if n == prev:
            stable += 1
        else:
            stable, prev = 0, n
    return exp_page.evaluate("() => document.body.innerText")
```
