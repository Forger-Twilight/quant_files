# B1：筛选 Prompt 模板（fallback 版）

> **重要：本 prompt 仅用于「仅综述模式」的 fallback 场景。**
> 全流程模式下 B1 不执行——A4 题录初筛（基于标题+摘要）已完成筛选，直接进 B2。
> 仅当用户自带 PDF 文件夹、且没有任何题录信息（标题/摘要）时，才用本 prompt 对 PDF 前 3 页做粗筛。

## 模板（填入扩写后的判定标准）

```
你是一名学术文献综述助手。请判断以下文献是否与主题「{用户的预设主题}」相关。

判定标准：
- 相关(relevant)：{扩写后的正面判定标准}
- 不相关(irrelevant)：{扩写后的反面判定标准，逐一列举常见无关类型}

注意：即使文献中提到了相关字样，如果其核心内容并非围绕该主题展开，也应判定为不相关。

请严格按照以下 JSON 格式回答，不要输出其他内容：
{{"label": "relevant" 或 "irrelevant", "reason": "一句话说明理由"}}

---
文献标题：{title}
期刊：{journal}，作者：{authors}，发表年份：{year}
文献前3页内容：
{first_3_pages}
```

## 真实案例

当用户主题为"量价相关性预测未来股票回报"时，扩写后的 prompt：

```
你是一名学术文献综述助手。请判断以下文献是否与主题「量价相关性预测未来股票回报」相关。

判定标准：
- 相关(relevant)：文献核心内容涉及「成交量/成交额」与「价格/收益率」之间的统计关系（如相关系数、协方差、量价背离等），并将该关系用于股票收益预测或选股研究。
- 不相关(irrelevant)：纯综述/评述类文献、纯理论资产定价模型(无量价实证)、宏观流动性研究、债券/期货等非股票资产研究、纯机器学习预测框架(未显式建模量价关系)、教学或课程研究。

注意：即使文献中提到了"量价"字样，如果其核心内容并非分析量与价之间的统计关系来预测股票收益，也应判定为不相关。

请严格按照以下 JSON 格式回答，不要输出其他内容：
{{"label": "relevant" 或 "irrelevant", "reason": "一句话说明理由"}}

---
文献标题：{title}
期刊：{journal}，作者：{authors}，发表年份：{year}
文献前3页内容：
{first_3_pages}
```

## 关键代码片段

**PyMuPDF（fitz）提取 PDF 文本**（本机用 fitz，不用 pdfplumber）：

```python
import fitz

def extract_pdf_text(pdf_path: str, max_pages: int = None) -> tuple:
    """返回 (文本, 总页数)"""
    doc = fitz.open(pdf_path)
    pages = [page.get_text() for i, page in enumerate(doc)
             if max_pages is None or i < max_pages]
    page_count = doc.page_count
    doc.close()
    return '\n'.join(pages), page_count
```

**LLM 返回 JSON 容错解析**：使用 `robust_parse()` 函数，详见 [shared/llm_client.md](../shared/llm_client.md)。

## 输出

- `output/filter_result.csv` — 包含每篇文献的路径、标题、label、reason
- `output/relevant_texts.pkl` — 相关文献的全文文本（pickle 序列化的 list[dict]）

筛选完成后打印统计，方便人工抽检。
