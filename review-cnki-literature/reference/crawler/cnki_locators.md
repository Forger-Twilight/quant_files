# CNKI 定位器映射表

本文件是定位器缓存。定位器失效（脚本输出 `LOCATOR_FAILURE`）时，按 [cnki_playwright_explore.md](cnki_playwright_explore.md) 重新探索并更新本表。

> last_updated: 2026-11-06（首次探索，全部条目经端到端实测验证）
> 凭据: 用户手动在浏览器中完成登录（机构账号/扫码/IP 认证），脚本不接触任何凭据
> 探索环境: rshfp_2.0 + playwright 1.54 + 系统 Edge；机构网络（哈工大）已识别

---

## 页面 URL

| 页面 | URL | 备注 |
|------|-----|------|
| 首页 | `https://www.cnki.net/` | |
| 高级检索页 | `https://kns.cnki.net/kns8s/AdvSearch` | 直达可行；结果在同页 AJAX 渲染，URL 不变 |
| 文献详情页 | `https://kns.cnki.net/kcms2/article/abstract?v=<token>` | href 从结果行标题链接读取；可凭 href 直达，已验证 |
| 题录导出页 | `https://kns.cnki.net/dm8/manage/export.html?language=CHS&uniplatform=NZKPT` | 新标签页弹出 |

---

## 登录流程

脚本打开知网首页后**暂停等待用户登录**。使用标记文件 `output/.wait_login` 协调 Agent 与脚本：

1. 脚本启动浏览器，导航到知网首页，写入标记文件 `output/.wait_login`，开始轮询等待文件被删除
2. Agent 通过 `AskUserQuestion` 询问用户是否已登录完成
3. 用户确认后，Agent 删除 `output/.wait_login`
4. 脚本检测到文件消失，继续执行

A3 / A5 各自共用一个浏览器实例，各需登录一次。超时 600s。

**机构 IP 免登录探测**：写 `.wait_login` 前先探测——导航到高级检索页试检索一次，结果表出现即视为有权限，跳过登录等待。（实测：哈工大机构网络下无需任何操作即有权限，页眉显示机构名。）

> ⚠️ **2026-11-18 修订：btn-search 探测不够，必须加导出权限探测。** 检索和勾选对匿名用户开放，但**导出题录和 PDF 下载要求登录态**。实测（非机构 IP + 未登录）：高级检索页正常打开、检索/勾选全部成功，直到导出才发现——菜单点击被前端静默吞掉（无新 tab、无弹窗），直达导出页 `export.html` 被 302 到 `login.cnki.net`。**正确的探测：goto 导出页 URL，未跳转 login.cnki.net 才算有完整权限**；跳转则必须先走登录协调。登录方式：账号/短信/校外访问（CARSI 机构 SSO，如哈工大 ids.hit.edu.cn，跳转链长，验证时勿打断在途导航）。附带收益：登录态 + 国内直连 IP 下 A5 逐篇下载 10 篇 0 验证码（匿名机构 IP 时约每篇 1 次）。

**验证码协调（`.wait_captcha`）**：检测到"安全验证/拖动拼图"时，脚本写入 `output/.wait_captcha` 并轮询；Agent 用 AskUserQuestion 请用户在浏览器中手动完成验证，确认后 Agent 删除标记文件，脚本继续。**严禁尝试自动绕过任何验证码。**

---

## 定位器表

### 高级检索表单（`kns8s/AdvSearch`）

| 操作 | selector | 匹配数 | 说明 |
|------|----------|--------|------|
| 表单行容器 | `dl.inputs-list > dd` | 每行 1 个 | 所有行内操作限定在行容器内（多行篇关摘的关键） |
| 行字段下拉 | 行内 `div.sort.reopt` | 每行 1 个 | 点击展开选项 UL；选项含 主题/篇关摘/关键词/篇名/全文/作者/第一作者/通讯作者/作者单位/基金/摘要/小标题/参考文献/分类号/文献来源/DOI |
| 字段选项-篇关摘 | 弹层内 `text=篇关摘` | 1 | 实测点击有效 |
| 行匹配方式下拉 | 行内文本 `精确 ∨` | 每行 1 个 | 点击弹出 `ul.sort-list`（选项：精确/模糊） |
| 行检索词输入框 | 行内 `div.input-box input[type=text]` | 每行 1 个 | 每框只填一个关键词；多个词用多行及行首连接词组合 |
| 加行 / 减行 | 行末 `a` 文本 `+` / `-` | 动态 | 空行不参与检索（实测：空行被忽略） |
| 行连接词下拉 | 行内 `div.sort.logical`，当前项 `.sort-default`，选项 `li` 文本 AND/OR/NOT | 第2行起每行1个 | 2026-11-11 实测同一对关键词 OR 返回141条、AND返回12条，回显一致；NOT仅观察到选项，暂不启用 |
| 时间范围起 | `input#datebox0` | 1 | readonly！**JS 赋值 + dispatch input/change 事件**（实测过滤生效：min 日期 2021-04-01） |
| 时间范围止 | `input#datebox1` | 1 | 同上 |
| 检索提交按钮 | `input.btn-search` | 1 | 点击提交；不要指望 Enter |
| 结果总数 | `span.pagerTitleCell` | 1 | 文本"共找到 N 条结果" |
| 检索条件回显 | 页面文本 `检索范围： 总库 （篇关摘：(...)（模糊)）` | - | 校验查询是否符合预期用 |

### 结果列表（同页渲染）

| 操作 | selector | 匹配数 | 说明 |
|------|----------|--------|------|
| 结果行 | `table.result-table-list tbody tr` | 动态 | 列：td0=勾选+序号， td1=标题链接， td2=作者， td3=来源， td4=发表时间， td5=数据库类型， td6=被引， td7=下载数， td8=操作 |
| 行内 checkbox | 行内 `td:first-child input[type=checkbox]` | 每行 1 个 | |
| 行内标题链接 | 行内 `td:nth-child(2) a`（首个） | 每行 1 个 | href 即详情页 URL，**A3 导出时应同时落盘** |
| 全选本页 | `#selectCheckAll1` | 1 | |
| 已选计数 | 页面文本 `已选 N`（regex 提取） | - | **勾选状态服务端跨会话保持**：每次开选前必须先点"清除"（见下） |
| 清除勾选 | 文本 `清除` 链接（`filenameClear`） | 1 | 在全选/已选旁边 |
| 期刊筛选 | 侧栏 link，name 正则 `^学术期刊` | 1 | get_by_role("link", name=re.compile("^学术期刊"))；**偶发加载慢，点击后必须验证 pagerTitleCell 计数变化，不变化则重试** |
| 每页条数 | `div.page-show-count` 点击 → `li` 文本 10/20/50 | 3 | 当前项 `li.cur`；偶发失灵，需验证行数变化并重试 |
| 相关性排序 | 可见 `li`，文本精确为 `相关度` | 1 | 2026-11-11 实测点击后 class 为 `DESC cur`；数据库筛选后再次确认 `cur`，不要定位正文中的“相关性”文本 |
| 翻页 | `a.pagesnums` 文本"下一页" | 1 | 页码为 `div.pagesnums` |

### 导出链路

| 操作 | selector | 匹配数 | 说明 |
|------|----------|--------|------|
| 导出与分析（一级菜单） | `#batchOpsBox > li`，has_text="导出与分析" | 1 | **hover 打开**（click 会反复开关） |
| 导出文献（二级菜单） | `li.export:has(ul.secondUl)` | 1 | **hover 打开** `ul.secondUl`；⚠️ `li.export` 有歧义（`li.bulkdownload.export` 是"批量操作"），必须带 `:has(ul.secondUl)` |
| 格式-RefWorks | `a[exporttype='Refworks']` | 1 | 点击后新标签页打开导出页 |
| 格式-EndNote | `a[exporttype='EndNote']` | 1 | 备选格式 |
| 导出页记录容器 | 无 textarea/pre；`document.body.innerText` 直接含全部标签记录 | - | **滚动加载**：scrollTo 底部直至 RT 计数稳定再抓取 |
| 导出页「导出」按钮 | `get_by_role("link", name="导出", exact=True)` | 1 | **产出完整 .txt 下载**（实测 63 条全量在文件内，不受页面 20 条显示限制）——首选路径 |
| 导出页「批量下载」 | `a#bulkDownloadToggle` | 1 | 只有"下载到CNKI AI/研学"，无本地批量 PDF |

格式全清单（exporttype）：GBTREFER（GB/T 7714-2025)、elearning（知网研学）、REFER(CAJ-CD)、MLA、APA、NEW（查新）、newdefine、**Refworks（首选）**、EndNote、NoteExpress、NodeFirst、BibTex、selfDefine

### 详情页与下载

| 操作 | selector | 匹配数 | 说明 |
|------|----------|--------|------|
| PDF 下载按钮 | `a#pdfDown`（容器 `li.btn-dlpdf`） | 1 | 期刊实测产出 `%PDF` 文件（文件名 `标题_作者.pdf`） |
| CAJ 下载按钮 | `a#cajDown`（容器 `li.btn-dlcaj`） | 1 | 仅有 CAJ 时记 caj_skipped.md |
| 行内「下载」链接（结果页） | 行内 `a` 文本"下载" | 每行 1 个 | 直接触发下载；⚠️ 学位论文实测产出 CAJ（文件头 `CAJ\x00`），期刊行内下载偶发不开下载——A5 统一走详情页 |
| 其他阅读按钮 | `li.btn-html`(HTML阅读）/`li.btn-yb-reader`（原版阅读）/`li.btn-cnki-ai` | - | 不用 |

---

## 筛选选项值

- 数据库范围（左侧栏）：学术期刊 / 学位论文 / 图书 / 会议 / 报纸 / 年鉴 / 专利 / 标准 / 法律法规（实测计数：66 条总量价相关性结果 = 期刊 9 + 学位 55 + 其他 0）
- 来源类别（左侧栏，可勾选）：北大核心 / CSSCI / AMI 等
- **默认建议**：数据库范围选"学术期刊"（学位论文是 CAJ 重灾区）；来源类别不限，由用户配置

---

## API endpoint

（未依赖：全链路走页面交互 + 下载事件，未使用 API 拦截。改版时如需 expect_response 再补充。）

---

## 特殊行为说明

| # | 行为 | 实测证据 | 处理方式 |
|---|------|----------|----------|
| 1 | 勾选状态**服务端跨会话保持** | 新会话打开时已选=63（前次遗留） | 每次开选前点"清除"；导出后也清除 |
| 2 | 勾选跨页保持 | 第1页全选→翻页→已选保持→第2页全选→66/66 | 分批导出循环可跨页累积勾选 |
| 3 | 导出页 20 条/页显示但滚动加载 | 滚动前 RT=20 → 滚动后 RT=63 | innerText 抓取前先滚动至底直至计数稳定；或直接用「导出」按钮下完整文件 |
| 4 | 「导出」按钮产出完整文件 | 63 条选中 → CNKI-YYYYMMDDxxx.txt 103KB 全 63 条 | 首选路径；innerText 为降级路径 |
| 5 | 日期框 readonly | fill 报 not editable | JS 赋值 `datebox0/1.value` + dispatch input/change（实测过滤生效） |
| 6 | 精确 vs 模糊差异大 | 同词组：精确 9 条 / 模糊 66 条 | 每行匹配方式独立可选（配置项），默认模糊 |
| 7 | 安全验证（腾讯拼图滑块，容器 `#tcaptcha_transform_dy`） | 容器常驻 DOM 但 opacity=0；导出样本尾部出现"拖动下方拼图完成验证" | 出现即 .wait_captcha 流程；**检测必须判定 opacity>0+有尺寸**（该容器常驻，纯文本/offsetParent 探测必误报——冒烟实测踩过） |
| 8 | 学位论文下载为 CAJ | 行内下载硕士论文得 `.caj`（头 `CAJ\x00`） | 检索默认限定"学术期刊"；CAJ-only 记 caj_skipped.md |
| 9 | 期刊筛选/每页条数偶发失灵 | 多次实测中间歇超时 | 点击后验证结果（计数/行数变化），不变则重试 ≤3 次 |
| 10 | 空检索行被忽略 | 加了空行检索结果不变，回显只含非空行 | 加行后务必填词，或先减行再加行 |
