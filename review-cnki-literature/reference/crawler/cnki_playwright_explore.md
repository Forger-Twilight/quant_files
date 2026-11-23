# CNKI 探索指南（Python Playwright 版）

Agent 用 **Python Playwright**（`rshfp_2.0` 环境已装 1.54，系统 Edge）探索 CNKI（KNS8）页面结构，填充 [cnki_locators.md](cnki_locators.md) 定位器缓存。

> 本机无 Node.js，不用 Node 版 `playwright-cli`。探索工具 = `python -m playwright codegen`（交互式选 selector）+ 探索辅助脚本（批量探测/快照/导出样本）。

---

## 前置条件

- `rshfp_2.0` 环境：playwright + 系统 Edge（`channel="msedge"`）
- **用户在场**：探索过程需要用户在弹出的浏览器中完成知网登录（机构账号/扫码/IP 认证），并可能需人工过验证码

## 何时触发

1. **locators 缓存为空**：`cnki_locators.md` 定位器表无数据
2. **locators 失效**：脚本输出 `LOCATOR_FAILURE` 退出码 2（核心定位器连续失败 3 次）

## 中间文件

快照、截图和**导出样本**存 `.playwright/exploration/`，每次探索前清空旧快照。

**导出样本（解析器 fixture）**：探索 A3 线时，必须实际导出 **RefWorks 和 EndNote 格式各一份**题录样本（10-20 条即可），存 `.playwright/exploration/export_refworks_sample.txt` / `endnote_sample.txt`。这两份文件是 a3 模板解析器的开发 fixture，也是导出格式最终选型的依据（哪种标签规整、摘要完整、编码正常选哪种），**不得删除**。

---

## 探索工具 1：codegen（交互式选 selector）

```bash
conda activate rshfp_2.0 && python -m playwright codegen --channel msedge --viewport-size=1920,1080 https://www.cnki.net/
```

- 打开真实 Edge + Inspector 窗口；在浏览器里操作，codegen 实时生成 Python 代码，**hover 页面元素即可看到 Playwright 推荐定位器**
- 适合：找检索框、按钮、菜单的 selector；验证点击路径；观察 iframe 结构（codegen 生成的代码会直接显示 frame 链）
- 不适合：批量统计匹配数、读取大段 DOM——用下面的辅助脚本

## 探索工具 2：探测辅助脚本（Agent 按需现写）

探索中的批量检查用一段临时 Python 脚本完成（Agent 按当时要查的问题写，用完留在 `.playwright/exploration/probe_*.py` 备查）。通用骨架：

```python
# probe_skeleton.py —— 探索辅助骨架(按需增删 probe)
import sys, time, json
sys.stdout.reconfigure(encoding='utf-8')
from playwright.sync_api import sync_playwright

URL = sys.argv[1] if len(sys.argv) > 1 else "https://www.cnki.net/"

with sync_playwright() as pw:
    browser = pw.chromium.launch(channel="msedge", headless=False)
    ctx = browser.new_context(viewport={"width":1920,"height":1080}, accept_downloads=True)
    page = ctx.new_page()
    page.goto(URL)
    input("在浏览器中完成操作(如登录)后,回到终端按 Enter 继续探测...")

    # probe 1: iframe 结构
    print("== frames ==")
    for f in page.frames:
        print(f"  name={f.name!r} url={f.url[:100]}")

    # probe 2: aria 快照(等价于 playwright-cli 的 snapshot,逐 frame 输出)
    for f in page.frames:
        try:
            snap = f.locator("body").aria_snapshot()
            open(f".playwright/exploration/snap_{f.name or 'main'}.yaml","w",encoding="utf-8").write(snap)
        except Exception as e:
            print(f"  snapshot fail frame={f.name!r}: {e}")

    # probe 3: selector 匹配数统计(strict mode 预检,改 SELECTOR 反复用)
    # print(page.eval_on_selector_all("SELECTOR", "els => els.length"))
    # 列表项级预检(遍历前 10 项):
    # print(page.evaluate("""() => {
    #   const items = document.querySelectorAll('<列表项_SELECTOR>');
    #   return Array.from(items).slice(0,10).map((item,i) => ({idx:i, count: item.querySelectorAll('<被检查selector>').length}));
    # }"""))

    page.screenshot(path=".playwright/exploration/final.png", full_page=False)
    input("探测结束,按 Enter 关闭浏览器...")
    browser.close()
```

要点：
- 登录等人工操作用 `input()` 暂停衔接（终端回车继续），不要用固定 sleep
- aria 快照逐 frame 输出——KNS8 是 iframe 结构，主页面快照可能看不到结果列表
- 截图与快照命名带步骤号（`snap_step3_result_list.yaml`），便于回溯

---

## 探索方法

探索分两条线，对应 [phase3_pipeline.md](phase3_pipeline.md) 的 A3/A5 操作 SOP。**探索即验证**：每个 selector 找到后必须通过一次端到端操作验证有效性（规则 1），不能仅凭"看起来匹配"就记录。

### A3 线：检索 → 勾选 → 导出

1. 打开知网首页，**请用户登录**（AskUserQuestion 确认）；记录登录态标志元素
2. 进入专业检索页（候选 `https://kns.cnki.net/kns8s/AdvSearch`），快照记录结构；用 probe 探测是否在 iframe 内
3. 填入测试检索式 `SU=('量价相关性')` 并提交（**先测 Enter 是否提交**，不提交则找检索按钮）→ 验证结果列表出现且与检索式相关
4. 记录：检索式输入框、提交按钮、结果列表项容器、列表项标题、结果总数元素
5. 勾选验证：点击"全选本页"→ 记录列表项 checkbox 与全选 checkbox → **翻页，验证已勾选是否保留**（决定分批导出写法）→ 翻回验证
6. 点击"导出与分析"→"导出文献"→ 记录菜单结构 → 选 **RefWorks** 格式导出一份样本 → 再选 **EndNote** 导出一份样本 → 两份存入 `.playwright/exploration/`
7. 记录导出全过程：格式选项、确认按钮、落盘形式（download 事件 / 新开页文本）、文件编码与分隔符（读样本文件确认）
8. 观察并记录：结果总数（判断 500 上限是否可实测，若测试检索式结果 <500 则换更宽检索式如 `SU=('股票')` 实测上限表现）
9. 若任何时刻出现滑块/验证码，记录其容器 selector（detect_captcha 用），请用户手动通过后继续

### A5 线：标题检索 → 详情页 → 下载

1. 用一篇已知文献标题（从 A3 线样本中取一条）做精确检索：`TI=('标题')` 或直接标题检索
2. 记录结果列表中标题链接的 selector（进详情页的入口）
3. 进入详情页，快照记录结构
4. 记录：PDF 下载按钮、CAJ 下载按钮（找一篇只有 CAJ 的学位论文对比验证 CAJ-only 形态）
5. 实际下载一篇 PDF，验证 download 事件与文件完整性（%PDF 头）
6. 记录详情页 URL 模式（是否含 dbcode/filename，可否凭 URL 直达）

### Strict mode 预检

列表项内的交互 selector 在不同项之间匹配数可能不一致（strict mode **静默失败**）。对每个列表项级别的交互 selector，用 probe 骨架里的"列表项级预检"eval 遍历前 10 项统计匹配数：

- 所有项 count=1 → 可用
- 任一项 count>1 → 加父级约束细化，重新计数确认
- 任一项 count=0 → 结构不同，记录备注

至少检查：列表项 checkbox、标题元素、标题链接。

### Selector 消歧策略

用 probe 统计 `querySelectorAll(selector).length` 匹配数：

| 匹配数 | 处理 |
|--------|------|
| = 1 | 可用 |
| > 1 | 消歧（优先级从高到低） |
| = 0 | 排查：此页面不存在该 selector（先确认是否在 iframe 内） |
| 动态 | 标注"动态"和出现时机 |

消歧优先级：
1. 限定父容器（含 frame 链）
2. 文本匹配
3. `:nth-child()` 索引
4. attribute 组合（`[data-xxx="yyy"]`）

---

## 生成脚本

探索完成后，**在生成 `code/` 下 A3/A5 脚本之前**，必须先通读 `reference/code_templates/` 下对应模板熟悉 Playwright Python SDK 语法与函数骨架。模板包含以下核心模式：

- **浏览器生命周期**：`sync_playwright().start()` → `chromium.launch(channel="msedge")` → `new_context(accept_downloads=True)` → `new_page()` → try/finally `browser.close()` + `pw.stop()`
- **全局状态管理**：`browser_closed` 标志 + `safe_eval()` 中检测 browser 是否已关闭
- **frame_locator 链**：iframe 内元素定位（按探索记录的 frame 链）
- **用户交互**：`page.fill()` / `page.click()` / `page.hover()` / `page.keyboard.press()`
- **下载**：`page.expect_download()` 包裹 click → `download.suggested_filename` → `download.save_as()` / `download.delete()`
- **异步等待**：`page.expect_response()` 等 API 返回 + `page.wait_for_selector()` 等 DOM 渲染
- **题录解析**：`parse_refworks()`（按 `.playwright/exploration/` 样本 fixture 校准）
- **去重**：标题归一化 + 第一作者双键
- **频率纪律**：`polite_sleep()` + 每 20 篇长休 + `detect_captcha()` + `.wait_captcha` 协调

然后根据文档 + 模板生成正式脚本：
1. `reference/code_templates/a3_export_template.md` / `a5_download_template.md` — 函数骨架与语法模式
2. `cnki_locators.md` — selector 与 frame 链
3. `cnki_session.md` — 代码片段（safe_eval / 下载 / 异步等待 / 频率纪律）
4. `phase3_pipeline.md` — 业务逻辑 + 操作 SOP

然后执行冒烟测试 → 正式运行（见 [phase3_pipeline.md 冒烟测试](phase3_pipeline.md)）。从模板**复制适配**（只改配置项）的脚本免冒烟，直接正式运行。

---

## 改版文件更新规则

| 文件 | 是否更新 | 说明 |
|------|---------|------|
| `cnki_locators.md` | **必须更新** | 定位器跟着网站结构走 |
| `cnki_session.md` | **通常不改** | 代码片段是通用模式，不随网站 UI 变化；仅全新交互范式出现时才补充 |
| `phase3_pipeline.md` | **不更新** | 纯业务逻辑，与 UI 无关 |
| `code/` 下 A3/A5 脚本 | **必须重新生成**（或重新复制适配模板） | 根据更新后的文档重新生成 |
