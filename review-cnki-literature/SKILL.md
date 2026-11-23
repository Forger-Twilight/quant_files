---
name: review-cnki-literature
description: 知网(CNKI)学术文献一站式检索与综述 skill。上游:高级检索(篇关摘、多行、每行独立精确/模糊、时间范围近5年)批量导出题录(RefWorks)→ LLM 题录初筛 → 只对入选文献下载 PDF;下游:批量结构化摘要、主题归纳,输出高质量文献综述。触发词:"知网检索"、"CNKI"、"检索知网"、"查学术文献"、"知网爬虫"、"中文学术文献"、"文献综述"、"知网下载论文"、"学术文献检索"、"总结这批论文"、"知网主题汇总"。
---

## 当前检索规则（2026-11-11）

- 每个“篇关摘”输入框只填一个关键词；可新增多行，通过页面的 AND/OR 下拉组合，禁止把 `(词1 + 词2)` 填进一个框。
- 近义词行用 OR 扩大召回；主题与市场等限定条件用 AND 缩小范围。不能把近义词默认 AND 连接。
- 首轮使用主题词；A股相关可用文献不足目标（上限10篇）时，可增加近义词 OR 行补检索。A股范围仍在 A4 核验；增减检索行不放开市场范围。
- 检索结果必须切换为“相关性”排序；数据库筛选后再次确认排序仍生效，再勾选、翻页和导出。不能确认时暂停并检查页面，不按默认排序继续。
- `keywords.txt` 中 `||` 表示新增 OR 行，`&&` 表示新增 AND 行；每行一个词和匹配标注。单组暂不混用两种连接词；需要混合条件时拆成独立检索组。


# review-cnki-literature — 知网学术文献一站式检索与综述

从研究主题出发，在知网**高级检索**(篇关摘字段)批量导出题录 → **LLM 题录初筛**(先看摘要相关性) → 只对入选文献下载 PDF → 批量结构化综述，输出文献综述文档。

## 步骤总表

Agent 按需加载参考文档，每步只读对应的 reference。用 TaskCreate 建 task 列表追踪进度。

| 步骤 | 说明 | 参考文档 |
|------|------|---------|
| A1 主题概览 | theme → AI 联网搜索 → `topics.md` (top 10) | [phase1_search.md](reference/crawler/phase1_search.md) |
| A2 关键词生成 | `topics.md` → 推荐关键词 + **每词联想发散约 5 个扩展词** → **★暂停等用户确认** → `keywords.txt` | [phase2_keywords.md](reference/crawler/phase2_keywords.md) |
| A3 检索+导题录 | `keywords.txt` → 高级检索(篇关摘) → 勾选 → 批量导出 RefWorks 题录 → 解析 → `records_all.jsonl`。locators 为空时先触发 [探索](reference/crawler/cnki_playwright_explore.md)；含冒烟测试 | [phase3_pipeline.md](reference/crawler/phase3_pipeline.md), [cnki_session.md](reference/crawler/cnki_session.md), [cnki_locators.md](reference/crawler/cnki_locators.md) |
| A→B 衔接 | **★必做**：统计 `records_all.jsonl` 年份分布，向用户反馈数量并询问是否限近 5 年(题录含年份,无需下载即可过滤) | — |
| A4 题录初筛 | 题录(标题+期刊+年份+摘要)分批 → LLM 判定 → `records_screened.jsonl` → **★向用户展示入选统计并确认** → `selected.jsonl`;入选数按目标规模控制(默认 ~10 篇:超出则 LLM 精选 top N,不足触发 N6 发散补检索) | [record_filter_prompt.md](reference/prompts/record_filter_prompt.md) |
| A5 入选下载 | `selected.jsonl` → 详情页(优先 A3 存的 href 直达) → `a#pdfDown` 下载 → 三道校验 → `papers/`;CAJ-only 记 `caj_skipped.md` | [phase3_pipeline.md](reference/crawler/phase3_pipeline.md) § A5 |
| Prompt 扩写 | 读 extraction_guide，将一句话主题扩写为正面/反面判定标准(A4/B1 共用;默认含市场范围约束,见 extraction_guide) | [extraction_guide.md](reference/summary/extraction_guide.md) |
| B1 粗筛(fallback) | **仅"仅综述模式且用户自带无题录 PDF"时执行**;PDF 前 3 页 → LLM 粗筛 | [filter_prompt.md](reference/prompts/filter_prompt.md) |
| B2 逐篇摘要 | 全文(fitz) → `individual/*.md` 学术六段 | [summary_prompt.md](reference/prompts/summary_prompt.md) |
| B4 主题汇总 | individual → `thematic_summary.md`(两步策略: LLM 出 JSON 分组,代码拼装正文) | [thematic_prompt.md](reference/prompts/thematic_prompt.md), [two_step_strategy.md](reference/summary/two_step_strategy.md) |
| 质量检查 | 对照 checklist 验收 | [quality_checklist.md](reference/quality_checklist.md) |

通用规范：[LLM 调用/JSON解析/ETA/运行约定](reference/shared/llm_client.md)

> **v1.1 变更(2026-11-09)**：移除 B3 严格再筛——入选文献全部进入综述，不再有剔除环节（`strict_filter_prompt.md` 与 B 模板中的 B3 代码保留存档，默认停用 `ENABLE_B3=False`）。同时新增默认约束：市场范围限定 A 股、目标入选规模 ~10 篇（规则 N6）。

### 目录约定

- A3 产出：`output/downloads/{theme}_{timestamp}/` 下 `exports/`(原始导出)、`row_links.jsonl`(结果行标题+详情页 href)、`records_all.jsonl`
- A4 产出：同目录 `records_screened.jsonl`、`selected.jsonl`
- A5 产出：同目录 `papers/`(PDF,`{year}-{journal}-{title}.pdf` 命名)、`caj_skipped.md`、`download_log.csv`
- B1 输入 = A5 的 `papers/` 目录(或仅综述模式用户指定 PDF 文件夹)

---

### Agent 行为规范

以下规则适用于所有步骤，不限于 A3。违反即错误，不视为"可接受的近似"。

| # | 规则 | 说明 | 示例 |
|---|------|------|------|
| 1 | **探索即验证** | 收集到的信息（selector、页面行为等）必须通过一次端到端操作验证其有效性，不能仅凭"看起来匹配"就记录 | 找到检索按钮后，提交明确测试检索式 → 检查结果列表出现且与检索式相关 |
| 2 | **遍历完整列表** | 当流程中列出多个参数/关键词/输入时，逐一执行，不可"任选其一"或"先试第一个" | 冒烟测试列出两个关键词 → 两个都要跑 |
| 3 | **多选验证** | 当 selector 匹配到多个元素时，必须检查每个候选并排除，不能依赖 DOM 顺序或取第一个 | 知网 `li.export` 有两个（批量操作/导出文献）→ 必须用 `li.export:has(ul.secondUl)` 消歧 |
| 4 | **意图优先** | 遇到表格、列表、步骤时，先思考设计意图再行动 | 冒烟表列两个关键词 → 意图是验证不同输入下的一致性 → 都要跑 |
| 5 | **原生交互 + 稳定性重试** | 所有交互组件**必须**使用 Playwright 原生 `.click()`/`.hover()`，**严禁** JS `element.click()`（框架事件不触发）；菜单类用 hover 展开；**点击后必须验证效果(计数/行数/可见性变化),无效则重试 ≤3 次**(知网筛选偶发失灵,实测) | 期刊筛选点击后验证 pagerTitleCell 计数变化；日期框 readonly → JS 赋值+事件(唯一例外,已实测有效) |
| 6 | **先读参考再写代码** | 生成 A3/A5 脚本前，**必须**先读 `reference/crawler/` 下三份文档 + `reference/code_templates/` 对应模板。**但模板只在 locators 表有数据后、即将生成脚本时才能读**，探索期间严禁碰 | 定位器表有数据 → 直接用；为空 → 先触发探索填充 |
| 7 | **先验证假设再修** | 看到 bug 现象后，先验证根因假设再动手修 | 导出 0 条 → 先验证"勾选是否生效"、"导出页是否打开"、"格式是否选对"，排除后再修 |
| 8 | **safe_eval 单参数** | `safe_eval(js_code)` 只接受 1 个字符串参数，参数化用 f-string 嵌入 | `safe_eval(f"document.querySelectorAll(...)[{idx}].click()")` ✅ |
| 10 | **无声失败即 bug** | 批量操作（搜索、勾选、导出、解析、下载）返回空结果时，**必须**输出诊断数据说明原因 | 导出条数=0、勾选数=0、解析条数≠勾选数、题录筛选 0 入选、下载 0 成功——都必须输出中间量诊断 |
| 11 | **冒烟与正式共享路径** | 冒烟测试**严禁**用硬编码值或独立 UI 片段绕过正式运行的任何逻辑路径 | 正式从文件读关键词 → 冒烟也读文件(走相同解析函数) |
| 12 | **优先复制适配，跑不通再重写** | `reference/code_templates/` 已有各阶段模板：①先检查有无对应模板；②有 → 复制到项目目录只改配置项；③运行通过则继续；④不通过先诊断小修；⑤结构性问题才允许重写。**默认禁止跳过②直接从头写** | 模板已经过验证;不看模板直接写 600 行 ❌ |
| N1 | **导出分批纪律** | 导出前先点"清除"再勾选(**勾选状态服务端跨会话保持**)；单组结果 >500 条时按年份切片分批检索；每批导出后必须解析验证条数与勾选数一致，不一致即停 | 新会话发现"已选=63"(上次遗留) → 先清除再选 |
| N2 | **反爬频率纪律** | 页面操作间隔 `polite_sleep(2,5)` 随机；每 20 篇下载长休 30s；严禁并发开页；出现滑块/安全验证 → 写 `.wait_captcha` → AskUserQuestion 请用户人工过验证 → 删标记继续；**严禁自动绕过验证码** | 检测到"安全验证/拖动拼图"可见元素 → 暂停等人 |
| N3 | **下载格式纪律** | 只下 PDF；详情页无 `a#pdfDown` 只有 `a#cajDown` 时记入 `caj_skipped.md` 不硬下；下载后三道校验(文件头 `%PDF`、页数 ≥2、大小 >10KB),失败重走完整流程重试 1 次 | 学位论文多为 CAJ-only → 检索默认限定"学术期刊"规避 |
| N4 | **题录即缓存** | 导出原件、records_all/records_screened/selected.jsonl、row_links.jsonl 全部落盘；A3/A4/A5 均支持断点续跑(按组/批次/downloaded_filename 跳过) | A5 中断后重跑:已有 downloaded_filename 的跳过 |
| N5 | **检索条件单一事实源** | 代码从 `keywords.txt` 解析每行关键词、匹配方式和连接词，分别操作页面输入框与 AND/OR 控件；禁止硬编码词、禁止行内拼接表达式 | `量价相关性 [模糊] || 价量相关性 [模糊]` → 两行篇关摘，第二行选择 OR |
| N6 | **目标规模自适应** | 目标入选上限10篇。A4 后超出则按相关度精选；不足则精确改模糊、增加近义词 OR 行或补充独立检索组。先说明发散方案；A股范围不放开 | 主题词不足时增加近义词 OR 行 |

> 编号沿用源 skill `review-broker-research` 的规则序列；原规则 9（引文提取双任务）随引用追踪在 v1 移除。

### 执行顺序与硬性门控

**以下步骤必须严格按顺序执行，禁止跳过、禁止并行、禁止省略。Agent 必须显式验证每个门控条件。**

| 前置条件 | 当前步骤 | 禁止行为 |
|---------|---------|---------|
| **ALIYUN_API_KEY 已收集** | A1/A2/A4/B 各脚本执行 | **严禁**硬编码或自行推测 API key。执行脚本前通过 `AskUserQuestion` 收集，通过 `sys.argv[1]` 传入(见 `reference/shared/llm_client.md`) |
| **A3/A5 脚本后台启动 + AskUserQuestion 登录确认** | 爬虫开始执行 | 脚本启动后自动弹浏览器并写 `output/.wait_login` 等待(机构 IP 免登录探测通过则跳过)。Agent 必须用 `AskUserQuestion` 询问用户是否已登录，用户确认后删除标记文件。**严禁**跳过登录确认直接删除标记文件 |
| `cnki_locators.md` 定位器表**有数据**(非空表格) | A3/A5 脚本生成 | 定位器表为空时**严禁**生成或执行脚本，先触发探索 |
| 正在执行 selectors 探索 | — | 探索期间**严禁**读取 `reference/code_templates/` 下 a3/a5 模板 |
| **已阅读参考文档** + locators 有数据 | A3/A5 脚本生成 | 未读时**严禁**生成脚本 |
| 冒烟测试**两阶段全部通过**(阶段A UI检查 + 阶段B 缩减管线)。**仅重新生成的脚本需要;复制适配的模板直接正式运行** | 正式 A3/A5 | 重新生成的脚本未过冒烟严禁正式跑 |
| A4 完成,**入选统计已向用户展示并经确认**(入选数/年份分布/期刊分布/预计下载耗时) | A5 下载 | **严禁**未经用户确认直接下载(下载最贵且触发反爬,下载量必须用户背书) |
| A5 已成功完成 | B 阶段(综述) | A5 未完成时**严禁**进入 B 阶段(仅综述模式除外,由用户指定 PDF 目录) |

每次运行前先杀掉残留进程、清理上一轮输出。

---

## 用户输入

1. **研究主题描述**（一句话）
2. **运行模式**（可选）：
   - **完整模式**（默认）：A1→A2→A3→A4→A5→B2→B4（v1.1 起无 B3）
   - **仅检索**：只跑 A 阶段（"下载论文"/"检索知网"）
   - **仅综述**：只跑 B 阶段（需指定 PDF 文件夹路径；无题录时走 B1 fallback；"文献综述"/"总结这批论文"）
3. **检索约束**（可选，默认值如下）：字段=篇关摘、匹配方式=每行可配(默认模糊)、时间范围=近5年、数据库范围=学术期刊(可放开学位论文,注意 CAJ 多)、**市场范围=A股**(在判定标准层实现,见 extraction_guide;不改关键词层,避免砍召回)、**目标入选规模≈10篇**(规则 N6)
4. **输出目录路径**（可选，默认 `output/`）

## keywords.txt 语法(A2 产出,A3 消费)

```text
量价相关性 [模糊] || 价量相关性 [模糊] || 量价关系 [模糊]；量价相关性 [模糊] && A股 [模糊]
```

- `；` 分隔**组**（每组 = 一次独立检索）
- 每一段对应页面一行“篇关摘”，输入框仅含一个关键词；`[模糊]`/`[精确]` 分别设置该行匹配方式。
- `||`：下一行的连接词设为 OR（或），用于近义词扩展；`&&`：设为 AND（并且），用于叠加条件。
- 同一组不混用 AND/OR，避免页面布尔优先级歧义。例如需要市场限定时，可分别检索“量价相关性 AND A股”“价量相关性 AND A股”，再合并题录去重。

---

## 依赖

- **Python(rshfp_2.0)**：`openai`, `fitz`(PyMuPDF), `playwright`(1.54)。**无 pdfplumber，文本提取一律 fitz**
- **浏览器**：系统已安装的 Edge（`channel="msedge"`），playwright Python 驱动
- **知网权限**：机构 IP 或机构账号(用户手动登录,脚本不接触凭据)

## 已知边界(v1)

- **不做引用追踪**(滚雪球):知网批量导出召回充足,学术引文知网命中率低,反爬成本不值。v2 如需恢复,形态=从入选 PDF 解析参考文献→知网检索标题→只导题录回流 A4
- **CAJ 不解析**:CAJ-only 文献记录后跳过,由用户手动处理
- **专业检索/普通检索不走**:统一走高级检索(用户指定路径),检索式拼装只面向高级检索表单
