# B3：严格再筛 Prompt 模板

> **⚠️ v1.1(2026-11-09)起默认停用**:B3 严格再筛已从主流程移除(无排除机制,入选文献全部进入综述)。本文件保留存档;如需恢复,在 B 阶段模板中设 `ENABLE_B3=True` 并按本文适配 prompt。

## 为什么要做这一步

全流程模式下 A4 题录初筛只能看标题+摘要（仅综述模式下 B1 只能看 PDF 前 3 页），常常无法暴露文献的真实方法论，初筛通过的"沾边但不切题"文献会污染后续的主题汇总。

在原券商研报版流程中实测一个 173 篇的项目：初筛通过 108 篇，严格再筛后只剩 53 篇，**55 篇是误判**。学术场景的典型误判类型：
- 综述/评述类：梳理了主题相关文献，但自身无原创模型或实证
- 纯理论模型：只推导命题，无数据实证（主题要求实证研究时）
- 只在引言/文献综述中提到主题关键词，核心研究的是另一个问题
- 宏观/行业层面研究，而主题要求个体资产（股票/企业）层面
- 案例研究/单一案例剖析，无一般性方法或大样本检验

B2 已经把每篇文献压成结构化的「核心思路+模型与方法构建+关键创新点」三段，**让 LLM 在这上面做严格判定，几乎不会再误判**。这是必做步骤，不要省略。

## 模板（agent 需基于用户主题适配）

```
你是一名严格的学术文献综述助手。请用最严格的标准判断以下文献是否「直接」研究主题「{用户的预设主题}」。

# 严格判定标准（必须同时满足两条才算 relevant）：

A. 文献的**核心研究内容**是{扩写的核心问题/关系定义}
   例如（由 agent 按主题替换）：实证检验投资者情绪与股票收益的关系；构建 ESG 评级对企业绩效影响的回归模型并做机制检验；提出新的量价背离度量方法并检验其收益预测能力等

B. **该研究内容本身**就是主题所问的对象，而不是只在引言、文献综述或稳健性检验中附带提及主题关键词

# 明确判为 irrelevant 的情况（即使含主题关键词也不行）：

1. **综述/评述类**：文献综述、研究述评、理论展望等，自身无原创模型或实证方法。
2. **纯理论模型无实证**：只建数理模型、推导命题，无任何数据检验（当主题要求实证研究时）。
3. **宏观/行业层面而非个体资产层面**：研究对象是国家/地区/行业总量指标，而主题要求的是个股/企业/个体层面。
4. **教学方法/课程研究**：讨论课程设计、教学改革、案例教学，不是研究学术问题本身。
5. **案例研究/单一案例**：只剖析单个企业/事件，无一般性方法或大样本检验。
6. **学位论文与已入选期刊版同题**：同一作者的学位论文与其期刊论文内容高度重叠，期刊版已入选时学位论文判 irrelevant。
7. **{其他类型，由 agent 按主题补充}**

# 同时也要 relevant 的边界情况：
- {agent 列出几个典型的 relevant 例子，避免 LLM 矫枉过正}
- 使用工具变量/双重差分/断点回归等识别策略检验主题关系的 → relevant（识别策略是实证研究的标准范式）
- 只覆盖机制链条中的一环、但明确为主题问题服务的 → relevant（机制证据是主题综述的组成部分）

请严格按以下 JSON 格式回答（不要任何额外内容，reason 中不要包含大括号字符）：
{{"label": "relevant" 或 "irrelevant", "reason": "一句话给出严格判定理由"}}

---
文献文件名：{filename}
文献标题：{title}

【核心思路】
{core_idea}

【模型与方法构建】
{formula}

【关键创新点】
{innovation}
```

## 关键设计原则

1. **必须明确列出 N 类「沾边但应排除」的情况**——只说"严格判定"是没用的，LLM 会自由发挥。要把每一类常见的误判类型都点名。
2. **同时给 relevant 边界例子**——避免 LLM 矫枉过正，把真正相关的方法变体（如换用不同代理变量/不同识别策略的稳健性设计）也判成 irrelevant。
3. **输入用 individual 摘要的 3 段，不要用原始 PDF**——结构化信息密度比 PDF 高一个数量级，且已经过滤了封面/目录/参考文献的噪音。
4. **prompt 中明确叮嘱 reason 不要含 `{}` 字符**——降低 JSON 解析失败概率。

## JSON 解析

使用 `robust_parse()` 函数，详见 [shared/llm_client.md](../shared/llm_client.md)。A4/B1 和 B3 使用同一个函数。

## 处理流程

```python
import shutil
from pathlib import Path
# extract_section() 见 reference/shared/llm_client.md

INDIVIDUAL_DIR = OUTPUT_DIR / 'individual'
DROPPED_DIR = OUTPUT_DIR / 'individual_dropped'
DROPPED_DIR.mkdir(exist_ok=True)

results = []
for p in sorted(INDIVIDUAL_DIR.glob('*.md')):
    text = p.read_text(encoding='utf-8')
    title = re.match(r'#\s+(.+)', text).group(1).strip()
    core_idea = extract_section(text, '核心思路')
    formula = extract_section(text, '模型与方法构建')
    innovation = extract_section(text, '关键创新点')

    prompt = STRICT_PROMPT.format(
        filename=p.name, title=title,
        core_idea=core_idea[:1500],
        formula=formula[:1500],
        innovation=innovation[:600],
    )
    parsed = robust_parse(call_llm(prompt))
    results.append({'filename': p.name, 'title': title, **parsed})

    if parsed['label'] != 'relevant':
        # 移到 dropped/，不要删除
        shutil.move(str(p), str(DROPPED_DIR / p.name))

# 写 strict_filter_result.csv
```

## 输出

- `output/strict_filter_result.csv` — 每篇的严格判定 label + reason
- `output/individual_dropped/` — 被剔除的 individual.md（保留供回查，不要删除）
- `output/individual/` — 严格筛后保留的 individual.md（最终用于 B4）

## 特殊情况：解析失败的恢复

如果某些 LLM 响应解析失败（label=error），**不要简单当作 irrelevant 丢掉**——它们可能是真正相关的文献，只是 reason 里含特殊字符。处理方式：

1. 用 robust_parse 已经能救大部分（贪婪 + label 字段直抽）
2. 如果还是 error，单独写个恢复脚本对这少数文件重判，prompt 里加一句「reason 不要含大括号」
3. 实在判不出的，人工 review 一下原文
