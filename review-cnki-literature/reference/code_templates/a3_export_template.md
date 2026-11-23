# A3 脚本模板：知网高级检索 + 批量导出题录

> **只在 `cnki_locators.md` 定位器表有数据后、即将生成脚本时才能读本文件；探索期间严禁读。**
> 复制到项目目录（如 `code/a3_export.py`）后，只改配置区再运行。结构性问题才允许重写（规则 12）。

| 配置项 | 说明 |
|--------|------|
| `THEME` | 用户的研究主题（仅用于目录命名与日志） |
| `KEYWORDS_FILE` | `output/keywords.txt`（A2 产出） |
| `OUTPUT_BASE` | 输出根目录，默认 `output` |
| `DATE_START` / `DATE_END` | 时间范围，默认近 5 年（如 `2021-01-01`/`2026-12-31`） |
| `DB_FILTER` | 数据库筛选，默认 `"学术期刊"`；设为 `None` 不筛选 |
| `PER_GROUP_LIMIT` | 每组导出上限，默认 500（超过则按年切片，见 phase3_pipeline.md §分批） |
| `EXPORT_FORMAT` | `"Refworks"`（首选，已实测）或 `"EndNote"` |
| `SMOKE_TEST` | 冒烟开关；冒烟时读 `SMOKE_KEYWORDS_FILE`、`SMOKE_LIMIT=10` |

参考：[phase3_pipeline.md](../crawler/phase3_pipeline.md)（SOP）、[cnki_session.md](../crawler/cnki_session.md)（代码片段）、[cnki_locators.md](../crawler/cnki_locators.md)（selector）。

```python
# -*- coding: utf-8 -*-
"""
A3: 知网高级检索 + 勾选 + 批量导出 RefWorks 题录 + 解析合并
输入: keywords.txt (组间；分隔, && 或 || 分行, 每行一个关键词, 行尾 [精确]/[模糊])
产出: exports/*.txt, row_links.jsonl, records_all.jsonl
不需要 LLM / API key。
"""
import sys, os, re, json, time, random, hashlib, datetime
sys.stdout.reconfigure(encoding='utf-8')
sys.stderr.reconfigure(encoding='utf-8')

from playwright.sync_api import sync_playwright
from playwright.sync_api import TimeoutError as PlaywrightTimeout

# ============================================================
# 配置（★ 每次运行前修改）
# ============================================================
THEME = "<用户的研究主题>"
OUTPUT_BASE = "output"
KEYWORDS_FILE = os.path.join(OUTPUT_BASE, "keywords.txt")
SMOKE_KEYWORDS_FILE = os.path.join(OUTPUT_BASE, "smoke_keywords.txt")
SMOKE_TEST = False          # 重新生成的脚本先 True 跑冒烟; 复制适配的模板 False 直接正式跑
SMOKE_LIMIT = 10

DATE_START = "2021-01-01"   # 近5年
DATE_END = "2026-12-31"
DB_FILTER = "学术期刊"       # None = 不筛选
PER_GROUP_LIMIT = 500
EXPORT_FORMAT = "Refworks"

PROFILE_DIR = os.path.join(os.path.dirname(os.path.abspath(__file__)), "edge_profile")
WORK_DIR = None  # main() 里初始化为 OUTPUT_BASE/downloads/{theme}_{ts}/

# ============================================================
# keywords.txt 解析(规范 N5: 检索式只能由代码拼装)
# ============================================================
def parse_keywords(path):
    """返回组->篇关摘行；logic 表示本行与前一行的关系。"""
    with open(path, encoding="utf-8-sig") as fh:
        raw = fh.read().strip().replace(";", "；")
    groups = []
    for g in raw.split("；"):
        g = g.strip()
        if not g:
            continue
        if "&&" in g and "||" in g:
            raise ValueError("同组暂不混用 AND/OR：请拆为独立检索组，避免布尔优先级歧义")
        connector = "OR" if "||" in g else "AND"
        rows = []
        for clause in re.split(r"&&|\|\|", g):
            clause = clause.strip()
            m = re.search(r"\[(精确|模糊)\]\s*$", clause)
            match = m.group(1) if m else "模糊"
            clause = re.sub(r"\[(精确|模糊)\]\s*$", "", clause).strip()
            row_query_text([clause])  # 旧多词格式报错，不静默截取第一个词
            rows.append({"terms": [clause], "match": match,
                         "logic": connector if rows else None})
        if rows:
            groups.append(rows)
    return groups

def row_query_text(terms):
    if len(terms) != 1 or not terms[0] or re.search(r"[\s+&|()（）]", terms[0]):
        raise ValueError('每个篇关摘输入框只允许一个关键词；多词请用多行 AND/OR')
    return terms[0]


def ensure_relevance_sort(page):
    # 知网页面实际标签为“相关度”；选中后该 li 的 class 包含 cur。
    option = page.locator('li:visible').filter(has_text=re.compile(r'^相关度$'))
    if option.count() != 1:
        raise RuntimeError('SORT_UNVERIFIED: 请重新探索相关性排序控件')
    for attempt in range(3):
        if 'cur' in (option.get_attribute('class') or '').split():
            return
        option.click(timeout=5000)
        page.wait_for_timeout(2000)
    if 'cur' not in (option.get_attribute('class') or '').split():
        raise RuntimeError('SORT_UNVERIFIED: 点击后未确认相关度排序选中状态')


# ============================================================
# RefWorks 解析(已用真实导出文件验证; 思路: 行首两字母标签, 空行分隔, 续行并入上一字段)
# ============================================================
TAG_MAP = {
    "RT": "type_raw", "A1": "authors", "AD": "affiliations", "T1": "title",
    "JF": "journal", "PB": "publisher", "YR": "year", "IS": "issue",
    "VO": "volume", "OP": "pages", "K1": "keywords", "AB": "abstract",
    "SN": "issn", "CN": "cn", "LA": "language", "LK": "link", "DO": "doi",
}

def parse_refworks(text):
    records, cur, last_key = [], None, None
    for raw in text.splitlines():
        line = raw.rstrip("\r\n")
        if not line.strip():
            continue
        m = re.match(r"^([A-Z][A-Z0-9])\s+(.*)$", line)  # \s 兼容不间断空格(innerText 降级路径实测需要)
        if m:
            tag, val = m.group(1), m.group(2).strip()
            if tag == "RT":
                cur = {"type_raw": val}
                records.append(cur)
                last_key = None
                continue
            if cur is None:
                continue
            key = TAG_MAP.get(tag)
            if key:
                cur[key] = val
                last_key = key
            else:
                last_key = None
        elif cur is not None and last_key:
            cur[last_key] += " " + line.strip()
    return [normalize_record(r) for r in records]

def normalize_record(r):
    r = dict(r)
    r["authors"] = [a.strip() for a in re.split(r"[;,;，]", r.get("authors", "")) if a.strip()]
    r["keywords"] = [k.strip() for k in re.split(r"[;,;，]", r.get("keywords", "")) if k.strip()]
    m = re.search(r"(\d{4})", r.get("year", ""))
    r["year"] = m.group(1) if m else ""
    if not r.get("journal") and r.get("publisher"):
        r["journal"] = r["publisher"]
    return r

def title_key(t):
    return re.sub(r"[\s　《》「\"'：:，,。.、；;！!？?（）()\[\]【】]", "", (t or "").lower())

def record_id(r):
    base = title_key(r.get("title")) + "|" + (r["authors"][0] if r.get("authors") else "")
    return hashlib.sha1(base.encode("utf-8")).hexdigest()[:16]

# ============================================================
# 浏览器辅助(详见 cnki_session.md)
# ============================================================
_browser_closed = False

def polite_sleep(lo=2, hi=5):
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

def click_and_verify(click_fn, verify_fn, retries=3, wait=3, label=""):
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

def set_date_range(page, start, end):
    page.evaluate(f"""() => {{
      const d0 = document.getElementById('datebox0'), d1 = document.getElementById('datebox1');
      d0.value = '{start}'; d1.value = '{end}';
      for (const d of [d0, d1]) {{
        d.dispatchEvent(new Event('input', {{bubbles: true}}));
        d.dispatchEvent(new Event('change', {{bubbles: true}}));
      }}
    }}""")

# ============================================================
# A3 主流程
# ============================================================
def ensure_login(page):
    """机构 IP 探测: 试进高级检索页,能加载即放行;否则走 .wait_login"""
    page.goto("https://kns.cnki.net/kns8s/AdvSearch", wait_until="domcontentloaded", timeout=30000)
    time.sleep(4)
    if page.locator("input.btn-search").count():
        print("[A3] 高级检索页可用,视为具备权限(机构 IP 或已登录)", flush=True)
        return True
    return wait_marker_cleared(os.path.join(OUTPUT_BASE, ".wait_login"), label="登录")

def clear_selection(page):
    """勾选状态服务端跨会话保持,开选前必须清除。注意: 已选/清除 UI 只在结果列表渲染后可见,
    所以本函数必须在检索提交、结果表出现之后调用(在 AdvSearch 空表单页调用等于没清)。
    可见的清除链接是 .checkcount 内 href 含 filenameClear 的 a; a.clearall 是隐藏模板,勿用。"""
    try:
        m = re.search(r"已选\s*(\d+)", page.evaluate("() => document.body.innerText"))
        if m and int(m.group(1)) > 0:
            page.locator("a[href*='filenameClear']:visible").first.click(timeout=5000)
            time.sleep(2)
            print(f"[A3] 已清除遗留勾选 {m.group(1)} 条", flush=True)
    except Exception as e:
        print(f"[A3] 清除勾选异常(忽略继续): {e}", flush=True)

MATCH_VAL = {"精确": "=", "模糊": "%"}
FIELD_VAL = {"主题": "SU", "篇关摘": "TKA", "关键词": "KY", "篇名": "TI", "摘要": "AB"}  # 常用项;全表见 cnki_locators.md

def set_row(page, row_idx, terms, match, field="篇关摘"):
    """配置第 row_idx 行(行内属性选择器,全部实测):
    字段容器 div.sort.reopt(当前值 span[value]),选项 li[data-val];
    匹配方式容器 div.sort.special,选项 ul.sort-list li a[value='='精确/'%'模糊]。
    行容器必须用 dl.inputs-list:visible > dd(页面有多个隐藏 tab 表单)。"""
    row = page.locator("dl.inputs-list:visible > dd").nth(row_idx)
    fval = FIELD_VAL[field]
    # 字段
    ok = False
    for attempt in range(3):
        cur = row.locator("div.sort.reopt .sort-default span").first.get_attribute("value")
        if cur == fval:
            ok = True
            break
        try:
            row.locator("div.sort.reopt").first.click(timeout=5000)
            time.sleep(1.5)
            row.locator(f"li[data-val='{fval}'] a").first.click(timeout=5000)
            time.sleep(1)
        except Exception as e:
            print(f"  [set_row] 行{row_idx} 字段第{attempt+1}次异常: {type(e).__name__}", flush=True)
    if not ok:
        raise RuntimeError(f"行{row_idx} 字段设为{field}失败(重试3次)")
    # 填词(textbox 角色自动排除 hidden input #showField)
    row.get_by_role("textbox").first.fill(row_query_text(terms))
    # 匹配方式
    ok = False
    for attempt in range(3):
        cur_txt = row.locator("div.sort.special .sort-default").first.inner_text().strip()
        if match in cur_txt:
            ok = True
            break
        try:
            row.locator("div.sort.special").first.click(timeout=5000)
            time.sleep(1.5)
            row.locator(f"ul.sort-list li a[value='{MATCH_VAL[match]}']").first.click(timeout=5000)
            time.sleep(1)
        except Exception as e:
            print(f"  [set_row] 行{row_idx} 匹配方式第{attempt+1}次异常: {type(e).__name__}", flush=True)
    if not ok:
        raise RuntimeError(f"行{row_idx} 匹配方式设为{match}失败(重试3次)")

def set_row_logic(page, row_idx, logic):
    """行首原生下拉设置 AND/OR，并核对回显；不依赖默认 AND。"""
    if logic not in ("AND", "OR"):
        raise ValueError(f"不支持的行连接词: {logic!r}")
    control = page.locator("dl.inputs-list:visible > dd").nth(row_idx).locator("div.sort.logical")
    if control.count() != 1:
        raise RuntimeError(f"LOCATOR_FAILURE: 行{row_idx}连接词控件不唯一")
    for attempt in range(3):
        current = control.locator(".sort-default").inner_text().strip().split()[0]
        if current == logic:
            return
        control.click(timeout=5000)
        control.locator("li").filter(has_text=re.compile(rf"^\s*{logic}\s*$")).click(timeout=5000)
        page.wait_for_timeout(300)
    if control.locator(".sort-default").inner_text().strip().split()[0] != logic:
        raise RuntimeError(f"LOCATOR_FAILURE: 行{row_idx}连接词未切换为{logic}")


def setup_form_rows(page, rows):
    """每行一个词；复用已有行，不够再加行；清空其余行避免残留条件。"""
    if not rows:
        raise ValueError("检索组不能为空")
    form_rows = page.locator("dl.inputs-list:visible > dd")
    while form_rows.count() < len(rows):
        before = form_rows.count()
        page.locator("dl.inputs-list:visible").get_by_role("link", name="+", exact=True).first.click(timeout=3000)
        page.wait_for_timeout(500)
        if form_rows.count() <= before:
            raise RuntimeError("LOCATOR_FAILURE: 点击加行后行数未增加")
    for i in range(form_rows.count()):
        form_rows.nth(i).get_by_role("textbox").fill("")
    for i, config in enumerate(rows):
        set_row(page, i, config["terms"], config["match"])
        if i:
            set_row_logic(page, i, config["logic"])

def scrape_result_rows(page):
    """抓当前页结果行: 标题/详情href/作者/期刊/日期/库类型"""
    return page.evaluate("""() => Array.from(document.querySelectorAll('table.result-table-list tbody tr')).map(tr => {
      const tds = tr.querySelectorAll('td');
      if (tds.length < 6) return null;
      const a = tds[1].querySelector('a');
      return { title: a ? a.innerText.trim() : '', detail_url: a ? a.href : '',
               authors: tds[2].innerText.trim(), journal: tds[3].innerText.trim(),
               date: tds[4].innerText.trim(), dbtype: tds[5].innerText.trim() };
    }).filter(Boolean)""")

def export_selection(page, ctx, out_path):
    """菜单链导出: hover 导出与分析 → hover 导出文献 → 点格式 → 导出页点「导出」下载"""
    page.locator("#batchOpsBox > li", has_text="导出与分析").first.hover(timeout=5000)
    time.sleep(1.5)
    page.locator("li.export:has(ul.secondUl)").first.hover(timeout=5000)
    time.sleep(1.5)
    with ctx.expect_page(timeout=15000) as pinfo:
        page.locator(f"a[exporttype='{EXPORT_FORMAT}']").first.click(timeout=8000)
    exp = pinfo.value
    exp.wait_for_load_state("domcontentloaded", timeout=15000)
    time.sleep(2)
    # 首选: 「导出」按钮下完整文件
    try:
        with exp.expect_download(timeout=60000) as dlinfo:
            exp.get_by_role("link", name="导出", exact=True).first.click(timeout=8000)
        dlinfo.value.save_as(out_path)
        exp.close()
        return "button"
    except Exception as e:
        print(f"  [导出] 按钮下载失败({type(e).__name__}),降级 innerText 滚动抓取", flush=True)
        prev, stable = -1, 0
        while stable < 3:
            exp.evaluate("() => window.scrollTo(0, document.body.scrollHeight)")
            time.sleep(2)
            n = len(re.findall(r"^RT ", exp.evaluate("() => document.body.innerText"), re.M))
            if n == prev:
                stable += 1
            else:
                stable, prev = 0, n
        open(out_path, "w", encoding="utf-8").write(exp.evaluate("() => document.body.innerText"))
        exp.close()
        return "innertext"

def run_a3_group(page, ctx, rows, group_idx):
    """单组: 检索→筛选→勾选→导出→解析。返回该组解析出的 records"""
    tag = " + ".join(rows[0]["terms"][:2])
    print(f"\n[A3] ==== 组 {group_idx}: {tag} ({len(rows)} 行) ====", flush=True)

    page.goto("https://kns.cnki.net/kns8s/AdvSearch", wait_until="domcontentloaded", timeout=30000)
    time.sleep(4)
    setup_form_rows(page, rows)
    set_date_range(page, DATE_START, DATE_END)
    page.locator("input.btn-search").click(timeout=5000)

    try:
        page.wait_for_selector("table.result-table-list tbody tr", timeout=20000)
    except PlaywrightTimeout:
        total_txt = page.locator("span.pagerTitleCell").inner_text() if page.locator("span.pagerTitleCell").count() else ""
        print(f"  [A3] 组 {group_idx} 无结果表,计数: {total_txt!r},跳过(规则10: 检索回显如下)", flush=True)
        echo = page.evaluate("() => { const m = document.body.innerText.match(/检索范围：([^\\n]+)/); return m ? m[1] : 'N/A'; }")
        print(f"  [A3] 检索回显: {echo}", flush=True)
        return []

    total_txt = page.locator("span.pagerTitleCell").inner_text()
    m = re.search(r"(\d+)", total_txt)
    total = int(m.group(1)) if m else 0
    print(f"  [A3] 检索总数: {total}", flush=True)
    if total == 0:
        return []

    clear_selection(page)  # 结果渲染后"已选/清除"才可见,在此清除遗留勾选

    if DB_FILTER:
        # 侧栏筛选链接带计数("学术期刊 9");计数为 0 时点不动且会把结果清空,必须跳过(实测踩坑)
        flink = page.get_by_role("link", name=re.compile(rf"^{DB_FILTER}"))
        ftxt = flink.first.inner_text() if flink.count() else ""
        m_cnt = re.search(r"(\d+)", ftxt)
        if not m_cnt or int(m_cnt.group(1)) == 0:
            print(f"  [A3] {DB_FILTER} 计数为 0 或无链接(文本: {ftxt.strip()!r}),跳过筛选,保留全部 {total} 条", flush=True)
        else:
            def _f():
                page.get_by_role("link", name=re.compile(rf"^{DB_FILTER}")).click(timeout=8000)
            def _v():
                t = page.locator("span.pagerTitleCell").inner_text()
                return t != total_txt
            click_and_verify(_f, _v, label="数据库筛选")
            time.sleep(2)

    ensure_relevance_sort(page)  # 筛选后、勾选前确认相关性排序

    # 每页 50(失败可继续,只是翻页多)
    def _set50():
        page.locator("div.page-show-count").click(timeout=5000)
        time.sleep(1.5)
        page.locator("li").filter(has_text=re.compile(r"^50$")).first.click(timeout=3000)
    click_and_verify(_set50, lambda: page.locator("table.result-table-list tbody tr").count() > 20 or total <= 20, label="每页50")

    # 勾选循环 + 行抓取(翻页保护: 首行标题与上页相同 = 末页,停)
    row_file = os.path.join(WORK_DIR, "row_links.jsonl")
    n_page = 0
    prev_first_title = None
    while True:
        n_page += 1
        try:
            page.locator("#selectCheckAll1").click(timeout=8000)
        except PlaywrightTimeout:
            # 规则10: 无声失败即 bug,输出工具栏状态诊断
            diag = page.evaluate("() => { const t = document.querySelector('.top-operate-btns'); return t ? t.outerHTML.slice(0,300) : '(no .top-operate-btns)'; }")
            print(f"  [A3] 全选框不可用,工具栏状态: {diag!r}", flush=True)
            raise
        time.sleep(1.5)
        rows_now = scrape_result_rows(page)
        if rows_now and prev_first_title is not None and title_key(rows_now[0].get("title")) == prev_first_title:
            print(f"  [A3] 第{n_page}页首行与上页相同,判定已到末页,停止翻页", flush=True)
            break
        prev_first_title = title_key(rows_now[0]["title"]) if rows_now else prev_first_title
        with open(row_file, "a", encoding="utf-8") as f:
            for r in rows_now:
                r["query_group"] = group_idx
                f.write(json.dumps(r, ensure_ascii=False) + "\n")
        sel = page.evaluate("() => { const m = document.body.innerText.match(/已选\\s*(\\d+)/); return m ? parseInt(m[1]) : 0; }")
        print(f"  [A3] 第{n_page}页: 本页 {len(rows_now)} 行,累计已选 {sel}", flush=True)
        if sel >= PER_GROUP_LIMIT:
            print(f"  [A3] 达到每组上限 {PER_GROUP_LIMIT},停止勾选(如需全量请按年切片)", flush=True)
            break
        if n_page >= 60:
            print(f"  [A3] 页数保护(60页)触发,停止", flush=True)
            break
        nxt = page.locator("a.pagesnums", has_text="下一页")
        if nxt.count() == 0:
            break
        nxt.first.click(timeout=8000)
        time.sleep(3)
        polite_sleep(1, 2)

    # 导出
    exp_path = os.path.join(WORK_DIR, "exports", f"group{group_idx}_{int(time.time())}.txt")
    os.makedirs(os.path.dirname(exp_path), exist_ok=True)
    via = export_selection(page, ctx, exp_path)
    print(f"  [A3] 导出完成({via}): {exp_path}", flush=True)

    # 解析 + 条数校验
    recs = parse_refworks(open(exp_path, encoding="utf-8", errors="ignore").read())
    print(f"  [A3] 解析出 {len(recs)} 条题录", flush=True)
    if len(recs) == 0:
        print(f"  [A3] ⚠️ 解析为 0(规则10): 导出文件前200字符: {open(exp_path, encoding='utf-8', errors='ignore').read()[:200]!r}", flush=True)
    for r in recs:
        r["query_group"] = group_idx
        r["source_file"] = os.path.basename(exp_path)
    clear_selection(page)
    return recs

def main():
    global WORK_DIR, PER_GROUP_LIMIT
    ts = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
    theme_slug = re.sub(r"[^\w一-龥]", "_", THEME)  # f-string 内不能有反斜杠,先算好
    WORK_DIR = os.path.join(OUTPUT_BASE, "downloads", f"{theme_slug}_{ts}")
    os.makedirs(WORK_DIR, exist_ok=True)

    kw_file = SMOKE_KEYWORDS_FILE if SMOKE_TEST else KEYWORDS_FILE
    limit = SMOKE_LIMIT if SMOKE_TEST else PER_GROUP_LIMIT
    groups = parse_keywords(kw_file)
    print(f"[A3] 共 {len(groups)} 个检索组, 模式={'冒烟' if SMOKE_TEST else '正式'}, 每组上限={limit}", flush=True)

    PER_GROUP_LIMIT = limit

    all_records = {}  # record_id -> record
    with sync_playwright() as pw:
        ctx = pw.chromium.launch_persistent_context(
            PROFILE_DIR, channel="msedge", headless=False,
            viewport={"width": 1920, "height": 1080}, accept_downloads=True,
        )
        page = ctx.pages[0] if ctx.pages else ctx.new_page()
        try:
            if not ensure_login(page):
                print("[A3] 登录等待超时,退出", flush=True)
                return 1
            for gi, rows in enumerate(groups, 1):
                if detect_captcha(page):
                    if not wait_marker_cleared(os.path.join(OUTPUT_BASE, ".wait_captcha"), label="过验证码"):
                        return 2
                recs = run_a3_group(page, ctx, rows, gi)
                for r in recs:
                    rid = record_id(r)
                    r["id"] = rid
                    if rid not in all_records:
                        all_records[rid] = r
                polite_sleep()
        finally:
            ctx.close()

    # 合并 row_links 的 detail_url
    links = {}
    row_file = os.path.join(WORK_DIR, "row_links.jsonl")
    if os.path.exists(row_file):
        for line in open(row_file, encoding="utf-8"):
            d = json.loads(line)
            links[title_key(d.get("title"))] = d.get("detail_url", "")
    for r in all_records.values():
        r["detail_url"] = links.get(title_key(r.get("title")), "")

    out = os.path.join(WORK_DIR, "records_all.jsonl")
    with open(out, "w", encoding="utf-8") as f:
        for r in all_records.values():
            f.write(json.dumps(r, ensure_ascii=False) + "\n")
    print(f"\n[A3] ✓ 完成: 去重后 {len(all_records)} 条 -> {out}", flush=True)
    n_abs = sum(1 for r in all_records.values() if r.get("abstract"))
    print(f"[A3] 摘要非空率: {n_abs}/{len(all_records)}", flush=True)
    return 0

if __name__ == "__main__":
    sys.exit(main())
```
