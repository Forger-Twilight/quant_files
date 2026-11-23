# A2 脚本模板：关键词生成 + 联想发散

每个篇关摘框只填一个词，多行通过 AND/OR 控件组合；详细契约见 `../crawler/phase2_keywords.md`。

复制到项目目录后，修改以下配置项再运行：

| 配置项 | 说明 |
|--------|------|
| `OUTPUT_DIR` | 项目输出目录路径（需已有 A1 输出的 `topics.md`） |

API key 经 `sys.argv[1]` 传入（见 `../shared/llm_client.md`）。

```python
# -*- coding: utf-8 -*-
"""
A2: 关键词生成 + 联想发散
读取 topics.md → AI 生成核心关键词并联想发散 → 用户确认
"""
import sys, os
sys.stdout.reconfigure(encoding='utf-8')
sys.stderr.reconfigure(encoding='utf-8')

from openai import OpenAI

# ★ 每次运行前修改
OUTPUT_DIR = "output"

KEYWORDS_TXT = os.path.join(OUTPUT_DIR, "keywords.txt")
KEYWORDS_REC = os.path.join(OUTPUT_DIR, "keywords_recommended.txt")

api_key = sys.argv[1] if len(sys.argv) > 1 else input("请输入 ALIYUN_API_KEY: ").strip()
client = OpenAI(
    api_key=api_key,
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
)

# 读取 topics.md
topics_path = os.path.join(OUTPUT_DIR, "topics.md")
with open(topics_path, "r", encoding="utf-8") as f:
    topics_content = f.read()
print(f"已读取 topics.md ({len(topics_content)} 字符)", flush=True)

# Prompt 与 phase2_keywords.md 的格式契约保持一致
prompt = f"""你是中文学术文献检索助手，请为知网高级检索生成“篇关摘”检索方案。

研究主题概览：
{topics_content}

要求：
1. 默认研究范围为A股，禁止推荐期货、港股、美股、基金或加密资产作为独立检索方向。
2. 推荐8-12个具有主题辨识度的核心词方向，每方向约5个强相关近义词。只作备用方案，不要求首轮全部执行；不要用通用方法词或宽泛概念凑数。
3. 每个篇关摘输入框只放一个完整关键词。近义词分行用 || 连接，由代码在页面设置 OR；主题与限定词分行用 &&，在页面设置 AND。
4. 每组只用一种连接词，禁止混用 && 与 ||。如需同时扩展近义词和限定市场，请拆成多组“主题词 && A股”，代码合并去重。
5. 默认匹配 [模糊]，可逐行指定 [精确]。禁止在输入词中加括号、加号、空格分词或其他布尔表达式。
6. 每组一行，组间用中文分号“；”分隔；不要编号、代码围栏或解释。第一组以用户主题词开头。

格式示例：
量价相关性 [模糊] || 价量相关性 [模糊] || 量价关系 [模糊]；量价相关性 [模糊] && A股 [模糊]
"""

print("调用 qwen3.5-plus 生成关键词...", flush=True)
resp = client.chat.completions.create(
    model="qwen3.5-plus",
    messages=[{"role": "user", "content": prompt}],
)
keywords_raw = resp.choices[0].message.content.replace("```", "").strip()
print(f"AI 返回:\n{keywords_raw}", flush=True)

# 保存推荐关键词
with open(KEYWORDS_REC, "w", encoding="utf-8") as f:
    f.write(keywords_raw)
print(f"\n推荐关键词已保存: {KEYWORDS_REC}", flush=True)

# 展示每个篇关摘行及连接关系
import re
groups = [g.strip() for g in keywords_raw.replace("；", ";").split(";") if g.strip()]
print(f"共 {len(groups)} 个候选检索组；首轮只执行用户确认的组", flush=True)
for i, group in enumerate(groups, 1):
    if "&&" in group and "||" in group:
        raise ValueError("同组不能混用 AND/OR，请拆成独立检索组")
    logic = "OR" if "||" in group else "AND"
    for j, clause in enumerate(re.split(r"&&|\|\|", group), 1):
        relation = "首行" if j == 1 else logic
        print(f"  组{i} 行{j}: {relation} 篇关摘 {clause.strip()}", flush=True)
print(f"请展示候选并确认本轮组，再写入 {KEYWORDS_TXT}；备用组留在推荐文件中", flush=True)

```
