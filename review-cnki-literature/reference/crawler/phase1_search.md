# A1: 学术文献主题概览

输入 theme → 多轮 AI 联网搜索 → 去重 → AI 排序 → top 10 写入 `topics.md`。不需要浏览器。

Client 创建和通用调用规范见 [shared/llm_client.md](../shared/llm_client.md)。

---

## API 调用

模型 `qwen3.5-plus`，启用联网搜索（在通用 `call_llm` 基础上加 `extra_body`）：

```python
resp = client.chat.completions.create(
    model="qwen3.5-plus",
    messages=[{"role": "system", "content": system}, {"role": "user", "content": prompt}],
    extra_body={
        "enable_search": True,
        "search_options": {"forced_search": True, "search_strategy": "max"},
    },
)
```

> ⚠️ 严禁设置 `timeout` 参数。联网搜索响应时间 30-120 秒属正常，由外层 Bash timeout 兜底。

## 搜索循环

至少 5 次，至多 10 次，累计去重 ≥20 篇时停止。每轮变换 query：

| 轮次 mod 4 | query 模板 |
|-----------|-----------|
| 1 | `{theme} 研究综述` |
| 2 | `{theme} 实证研究` |
| 3 | `{theme} 影响机制` |
| 0 | `中国 学术文献 {theme}` |

## 搜索 Prompt

```text
你是一位学术文献检索助手。请严格根据研究主题 theme 搜索相关中文学术文献(期刊论文、学位论文)。
- 必须是围绕该主题的原创研究(有明确研究问题与方法)
- 排除新闻报道、会议通知、征稿启事、书评等非研究性内容
- 优先中文学术期刊论文
按以下表格格式输出:
| 排名 | 文献标题 | 研究问题 | 方法数据 | 核心结论 |
```

## 表格提取

```python
import re
pattern = r'\|\s*(\d+)\s*\|\s*([^|]+)\|\s*([^|]+)\|\s*([^|]+)\|\s*([^|]+)\|'
# 每个 match → {rank, title, question, method, conclusion}
```

## 去重

按 `title.lower().replace(" ", "")` 去重。

## AI 重排序

收集 ≥20 篇后，用同一模型按原始 theme 相关度统一排序取 top 10。排序 Prompt 要求严格按相关度排，保持原内容不变，只输出表格。

## 输出 topics.md 格式

```markdown
搜索执行了多轮，累计去重后得到 **N 篇** 文献，最终筛选出 **10 篇** 最高质量的文献，按主题相关度排序：

---

| 排名 | 文献标题 | 研究问题 | 方法数据 | 核心结论 |
|------|----------|----------|----------|----------|
| 1 | {title} | {question} | {method} | {conclusion} |
| 2 | ... | ... | ... | ... |

---
```

注意：表格内容中的 `|` 字符需要转义为 `\|`。

## 中间结果落盘（强制）

**每轮搜索后必须立即保存中间结果**，不能等所有轮次结束后再一次性写入。这是为了：
1. 防止脚本中断导致已完成的搜索轮次全部丢失
2. 方便用户中途检查搜索质量
3. 支持断点续跑（后续轮次启动时先读取已有中间文件）

具体约定：
- 每轮 AI 联网搜索返回后，将当轮提取的表格直接写入 `output/topics_{i}.md`（`i` 从 1 开始编号）
- 格式与最终 `topics.md` 相同，只是包含当轮新增的条目
- 同时维护一份 `output/topics_all.md`，每轮追加新去重后的条目，作为累计快照
- 最终重排序完成后，从 `topics_all.md` 取 top 10 写入 `output/topics.md`
- 脚本启动时先检查是否存在 `topics_{i}.md` 文件，若有则从最大 `i+1` 轮继续（断点续跑）
