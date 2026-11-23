# A3/A4/A5: 检索导出 → 题录初筛 → 入选下载

每个篇关摘框只放一个词；多行用页面的 AND/OR 连接，检索后按相关性排序。关键词格式见 [phase2_keywords.md](phase2_keywords.md)。

读取 `keywords.txt` → A3 高级检索+批量导出题录 → A4 LLM 题录初筛 → A5 入选文献 PDF 下载。A3/A5 需要浏览器，A4 纯 LLM 前台运行。

浏览器操作使用 **Playwright Python SDK**（`from playwright.sync_api import sync_playwright`），辅助函数和技术规则见 [cnki_session.md](cnki_session.md)。定位器见 [cnki_locators.md](cnki_locators.md)。定位器探索见 [cnki_playwright_explore.md](cnki_playwright_explore.md)。

---

## 硬性门控

| # | 条件 | 检查方法 | 违反后果 |
|---|------|---------|---------|
| 1 | **定位器表有数据** | `cnki_locators.md` 定位器表 ≥1 行有效数据 | 为空时**严禁**生成或执行脚本，必须先跑探索 |
| 2 | **冒烟测试全 PASS**（仅重新生成的脚本） | 冒烟输出中阶段A+阶段B 全部 PASS | 任一项 FAIL → 修复脚本 → 重跑冒烟；复制适配模板免冒烟 |

---

## 执行规范

- `-u` 标志 + `PYTHONUNBUFFERED=1`
- 新脚本前杀掉残留的 Python 进程
- 依赖：`rshfp_2.0` 环境（playwright + fitz + openai）
- A4 前台直接运行；**A3/A5 后台运行**（登录/验证码协调需要 Agent 交互）
- ETA / timeout / 防超时输出见 [shared/llm_client.md](../shared/llm_client.md)

---

## 冒烟测试 → 正式运行

**冒烟测试仅适用于重新生成的脚本。** 从 `code_templates/` 复制适配的模板已经过验证，**跳过冒烟，直接正式运行**（`SMOKE_TEST = False`）。只有正式运行失败、诊断后需要重写脚本时，新脚本才需要走冒烟流程。

### 冒烟测试 = 两阶段，缺一不可

| 阶段 | 内容 | 说明 |
|------|------|------|
| **阶段A: UI检查** | 4 项检查 | 验证 检索/勾选/导出/下载 在页面上正确运作 |
| **阶段B: 管线验证** | 缩减版完整管线 | 用冒烟参数跑通 导出→解析→初筛→下载 全流程 |

**禁止行为**：只做阶段A就 return。

### 冒烟参数

| 参数 | 正式值 | 冒烟值 |
|------|--------|--------|
| 关键词 | 从 `keywords.txt` 解析 | 从 `smoke_keywords.txt` 解析（**走相同解析函数**，规则11） |
| 每组导出上限 | 500 | 10 |
| A4 批大小 / 批数上限 | 25 / 不限 | 5 / 2 |
| A5 下载上限 | 全部入选 | 1 |

### 阶段A：UI 检查（必须调正式函数，严禁独立 UI 片段）

阶段A 直接调正式函数 `run_a3(smoke_keywords, smoke_limit=10)` 与 `run_a5(smoke_selected, limit=1)`，验证：

1. **检索生效** → 提交后 `span.pagerTitleCell` 出现且计数 >0，结果与主题沾边
2. **勾选与导出生效** → 勾选数>0，导出文件落盘，`parse_refworks` 解析条数 == 导出页记录数
3. **时间范围生效** → 解析结果的最早年份 ≥ 起始年
4. **下载链路** → 对 1 篇已知标题走完 详情页→`a#pdfDown`→下载→三道校验

### 阶段B：缩减版完整管线

```
run_a3(smoke_keywords, limit=10)      → records_all.jsonl ≥1 条且字段完整(title/year/abstract 非空)
run_a4(records, batch=5, max_batch=2) → records_screened.jsonl 合法 JSON;0 入选时输出 LLM 原始响应诊断(规则10)
run_a5(selected, limit=1)             → papers/ 下 1 个 PDF,三道校验通过,selected.jsonl 回写 downloaded_filename
```

**通过条件**：上述每步产物满足检查；任一失败即 FAIL。

### 代码结构要求

`SMOKE_TEST` 分支**必须**依次调用阶段A和阶段B，**严禁**阶段A后直接 return。

---

## 前置：收集 API Key + 登录协调

**执行含 LLM 的脚本（A4）或爬虫脚本（A3/A5）前，按需收集 `ALIYUN_API_KEY`**：A3/A5 不需要 LLM（纯浏览器）可不收集；A4/B 阶段必须通过 `AskUserQuestion` 收集后 `sys.argv[1]` 传入。严禁硬编码。

**登录协调流程**（Agent 侧，A3/A5 各一次）：
1. 后台启动脚本（脚本自动打开浏览器并导航到知网，先探测机构权限：试检索一次成功则跳过登录等待）
2. 无权限时脚本写 `output/.wait_login`；Agent 用 `AskUserQuestion` 询问用户"是否已登录完成"
3. 用户确认后，Agent 删除标记文件
4. 脚本检测到文件消失，继续执行

**验证码协调**：脚本检测到"安全验证/拖动拼图"**可见**元素时写 `output/.wait_captcha` 并轮询；Agent 请用户人工过验证后删标记。严禁自动绕过。

selector 从 [cnki_locators.md](cnki_locators.md) 查取。技术规则见 [cnki_session.md](cnki_session.md)。

---

## A3 操作 SOP：检索 + 勾选 + 导出 + 解析

### 关键词解析与行拼装（代码职责，规范 N5）

`parse_keywords(path)` 返回每组的行列表：`[{terms: [单个词], match: '模糊'|'精确', logic: null|'AND'|'OR'}]`。首行 logic 为 null，后续行表示与前一行的连接关系。

- `；` 分组，`||` 表示新增 OR 行，`&&` 表示新增 AND 行；同组不能混用两种连接词。
- `row_query_text` 仅返回一个词，不拼接括号或加号；旧多词输入必须报错。
- `setup_form_rows` 清空已有输入，复用已有行，不足时点击加行；每行字段显式设为篇关摘，连接词由 `set_row_logic` 选择并验证。

### 每组检索序列

```
1. 清除勾选(服务端跨会话保持,必须先清): 若"已选 N">0 点"清除"
2. 导航: goto 高级检索页(见 locators 页面 URL 表)
3. 清空所有已有行输入，复用已有行；不足时点 + 加行，并验证行数增加
4. 逐行配置: 字段→篇关摘；每框仅填一个关键词；设置匹配方式。第二行起显式设置 AND/OR，并验证回显
5. 时间范围: JS 设置 datebox0/datebox1(readonly,fill 无效;dispatch input+change)
6. 提交: click input.btn-search
7. 等待: wait_for_selector 结果行(见 locators 结果列表表)
8. 校验: 读 pagerTitleCell 总数;>0 才继续;为 0 输出诊断(检索回显文本)并跳过本组
9. 数据库筛选(配置项,默认学术期刊): 点击侧栏筛选 → 验证计数变化,无效重试 ≤3 次
10. 先设置并验证“相关性”排序，再设置每页条数 → 50(验证行数变化,无效重试)
11. 勾选循环(受 PER_GROUP_LIMIT 上限保护):
    while 未到末页 and 已选 < 上限:
      全选本页(#selectCheckAll1)
      抓取本页每行 (title, detail_href, authors, journal, date, dbtype) → row_links.jsonl 追加
      下一页(a.pagesnums 文本=下一页)
12. 导出: hover 导出与分析 → hover li.export:has(ul.secondUl) → 点 a[exporttype='Refworks']
    → 新标签页(导出页) → 点「导出」(role link, name 精确匹配) → expect_download 存 exports/{group}_{ts}.txt
    降级路径: 导出按钮失败 → 滚动加载至 RT 计数稳定 → innerText 抓取存文件
13. 解析: parse_refworks(文件) → 记录数 == 已选数校验(不一致即停,输出诊断) → 追加 records_all.jsonl(去重: 标题归一化+第一作者)
14. 清除勾选(为下一组)
15. 组间 polite_sleep()
```

### 导出 >500 条的分批（规范 N1）

单组结果数 >500 时：按年份切片（把 datebox 区间二等分，递归直至每片 ≤500），逐片检索导出。切片过程打印诊断（每片区间+条数）。

### parse_refworks 解析要点

- 行首标签正则 `^([A-Z][A-Z0-9])\s+(.*)$`（\s 兼容不间断空格，innerText 路径实测需要）
- 记录以 `RT` 行开始；非标签行并入上一字段（续行）
- 标签映射：RT/A1(authors)/AD/T1(title)/JF(journal)/PB(publisher，学位论文）/YR(year)/IS/VO/OP/K1(keywords)/AB(abstract)/SN/CN/LA/LK/DO
- 后处理：authors/keywords 按 `;` 拆列表；year 取 4 位数字；journal 缺失时回退 publisher
- 与 row_links 按标题归一化合并，补 `detail_url` 字段（A5 直达用）

---

## A4 操作 SOP：题录 LLM 初筛

纯 LLM 阶段（前台运行，无浏览器）：

```
1. 读 records_all.jsonl → (可选)按用户确认的年限过滤
2. 分批(每批 25 条): 每条编号 + 标题 + 期刊 + 年份 + 摘要(截 500 字) + 关键词
3. qwen-long + response_format=json_object → 返回 JSON 数组 [{idx, label: relevant|borderline|irrelevant, reason}]
4. robust_parse_list 解析(见 shared/llm_client.md);失败批次重试 1 次,仍失败整批标记 label=error 不丢弃
5. 批次级落盘 records_screened.jsonl(断点续跑: 按 record id 跳过已筛)
6. 汇总统计 → 写 selected.jsonl(relevant+borderline,borderline 直接入选,无再筛)
7. 目标规模控制(规则 N6,默认 ~10 篇): 入选 > 目标 → LLM 按相关度精选 top N;入选 < 目标 → 发散匹配补检索后回到步骤 1(发散方案先向用户说明)
```

判定标准由 extraction_guide 扩写后填入 prompt 模板（见 [record_filter_prompt.md](../prompts/record_filter_prompt.md)）。宽松原则：宁滥勿缺，摘要声称即采信，无法验证不判负。

**★A4→A5 门控**：向用户展示入选统计（入选数/年份分布/期刊分布/预计下载耗时 = 入选数 × 20s），AskUserQuestion 确认后才启动 A5。

---

## A5 操作 SOP：入选文献下载

```
1. 读 selected.jsonl;已含 downloaded_filename 的跳过(断点续跑)
2. 逐篇:
   a. 有 detail_url → ctx 内当前页 goto 直达;无 → 高级检索 TI=('标题') 精确 → 取第一行标题链接 href → goto
      (详情页 URL 模式见 locators 页面 URL 表)
   b. 检测验证码 → .wait_captcha 流程
   c. 存在 a#pdfDown → expect_download 包裹点击 → 保存
      无 a#pdfDown 但有 a#cajDown → 记 caj_skipped.md(标题+作者+期刊+detail_url),跳过
      两者皆无 → 记 download_log 失败行,continue
   d. 三道校验: 文件头 %PDF / 页数 ≥2 / 大小 >10KB;失败重走完整流程重试 1 次
   e. 重命名 {year}-{journal}-{title}.pdf(Windows 非法字符清洗,长度截断 80)
   f. 回写 selected.jsonl 的 downloaded_filename;写 download_log.csv
   g. polite_sleep(2,5);每 20 篇长休 30s
3. 完成后打印统计: 成功/CAJ跳过/失败 各多少
```

### 禁止事项

| 错误写法 | 正确做法 |
|---------|---------|
| 直接 click `li.export`（歧义：命中"批量操作"） | `li.export:has(ul.secondUl)` 消歧 |
| 点"导出与分析"用 click（菜单 toggle 会合上） | hover 展开菜单链 |
| 日期框 fill()（readonly 报 not editable） | JS 赋值 + dispatch input/change |
| 假设勾选是新会话空白状态 | 开选前必须点"清除"（服务端跨会话保持） |
| 导出页 innerText 直接抓（只有前 20 条） | 优先「导出」按钮下完整文件；innerText 兜底需滚动加载至 RT 稳定 |
| A5 用标题搜索进详情页（慢且可能误配） | 优先 A3 落盘的 detail_url 直达；搜索为降级路径 |
| 学位论文行内下载（产出 CAJ） | 检索默认限定学术期刊；详情页无 pdfDown 即记 caj_skipped |
| 失败后改 selector 继续跑 | 定位器连续失败 3 次 → LOCATOR_FAILURE 退出码 2 → 触发探索 |

---

## 输出目录

`{output_dir}/downloads/{theme}_{timestamp}/`:

| 文件 | 内容 |
|------|------|
| `exports/{group}_{ts}.txt` | RefWorks 原始导出（题录即缓存） |
| `row_links.jsonl` | 结果行抓取（title/detail_url/journal/date/dbtype，按组） |
| `records_all.jsonl` | 解析合并去重后的题录（含 detail_url） |
| `records_screened.jsonl` | A4 判定全量（含 label/reason） |
| `selected.jsonl` | A4 入选（relevant+borderline），A5 输入 |
| `papers/` | PDF（`{year}-{journal}-{title}.pdf`） |
| `caj_skipped.md` | CAJ-only 清单 |
| `download_log.csv` | 每篇下载状态 |
| `analysis.md` | 检索汇总报告（检索式/各组条数/去重后总数/A4 分布/下载统计） |

---

## 鲁棒性设计

- 单篇/单批失败不中断（try/except + continue + 诊断输出）
- 期刊筛选、每页条数等偶发失灵步骤：**点击后验证效果，无效重试 ≤3 次**
- 下载超时 60s（`DOWNLOAD_TIMEOUT = 60000` 毫秒）
- 定位器连续失败 3 次 → `LOCATOR_FAILURE` 退出码 2 → 触发探索
- `main()` 用 `try/finally` 确保 `browser.close()` 执行

---

## 故障排查

异常信号时**暂停排查，严禁继续执行剩余组合**：

| 异常信号 | 排查 |
|---------|------|
| 检索后无结果表 | 检索回显文本是否符合预期；篇关摘是否设置成功；时间范围是否写入（读 datebox 值） |
| 导出文件 0 条 | 已选计数是否 >0；导出页是否打开（popups）；格式链接是否点对（exporttype） |
| 解析条数 ≠ 勾选数 | 导出文件是首 20 条（innerText 未滚动）还是全量；解析器是否匹配标签正则 |
| PDF 校验失败（非 %PDF） | 可能下到 CAJ（头 `CAJ\x00`）或错误页 HTML；检查该篇是否 caj_only |
| 频繁弹安全验证 | 操作间隔过短；放大 polite_sleep 区间 |
| 翻页后已选清零 | 不应发生（跨页保持已实测）；检查是否误点"清除"或重新检索重置了勾选 |
