# A1 脚本模板：学术文献主题概览

复制到项目目录后，修改以下配置项再运行：

| 配置项 | 说明 |
|--------|------|
| `THEME` | 用户的研究主题 |
| `OUTPUT_DIR` | 项目输出目录路径 |

排序 prompt 中的排序规则（`rank_prompt` 内）需要根据 THEME 适配。API key 经 `sys.argv[1]` 传入（见 `../shared/llm_client.md`）。

```python
# -*- coding: utf-8 -*-
"""
A1: 学术文献主题概览 — AI 联网搜索相关中文学术文献
多轮搜索 → 去重 → AI 排序 → top 10 写入 topics.md
"""
import sys, os, re, time
sys.stdout.reconfigure(encoding='utf-8')
sys.stderr.reconfigure(encoding='utf-8')

from openai import OpenAI

# ============================================================
# 配置（★ 每次运行前修改）
# ============================================================
THEME = "<用户的研究主题>"
OUTPUT_DIR = "output"

api_key = sys.argv[1] if len(sys.argv) > 1 else input("请输入 ALIYUN_API_KEY: ").strip()
client = OpenAI(
    api_key=api_key,
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
)

QUERY_TEMPLATES = [
    lambda t: f"{t} 研究综述",
    lambda t: f"{t} 实证研究",
    lambda t: f"{t} 影响机制",
    lambda t: f"中国 学术文献 {t}",
]

SEARCH_SYSTEM = """你是一位学术文献检索助手。请严格根据研究主题 theme 搜索相关中文学术文献(期刊论文、学位论文)。
- 必须是围绕该主题的原创研究(有明确研究问题与方法)
- 排除新闻报道、会议通知、征稿启事、书评等非研究性内容
- 优先中文学术期刊论文
按以下表格格式输出:
| 排名 | 文献标题 | 研究问题 | 方法数据 | 核心结论 |"""

TABLE_PATTERN = re.compile(r'\|\s*(\d+)\s*\|\s*([^|]+)\|\s*([^|]+)\|\s*([^|]+)\|\s*([^|]+)\|')

# ============================================================
# 工具函数
# ============================================================
def extract_rows(text):
    rows = []
    for m in TABLE_PATTERN.finditer(text):
        rank = int(m.group(1))
        title = m.group(2).strip()
        question = m.group(3).strip()
        method = m.group(4).strip()
        conclusion = m.group(5).strip()
        if len(title) > 5:
            rows.append({"rank": rank, "title": title, "question": question, "method": method, "conclusion": conclusion})
    return rows


def title_key(title):
    return title.lower().replace(" ", "").replace("　", "")


def dedup(accumulated, new_rows):
    existing_keys = {title_key(r["title"]) for r in accumulated}
    added = 0
    for row in new_rows:
        if title_key(row["title"]) not in existing_keys:
            accumulated.append(row)
            existing_keys.add(title_key(row["title"]))
            added += 1
    return added


def save_md(rows, path, total_info=""):
    md = f"搜索执行中... 当前累计去重 {total_info}\n\n---\n\n"
    md += "| 排名 | 文献标题 | 研究问题 | 方法数据 | 核心结论 |\n"
    md += "|------|----------|----------|----------|----------|\n"
    for i, r in enumerate(rows, 1):
        t = r["title"].replace("|", "\\|")
        q = r["question"].replace("|", "\\|")
        m_ = r["method"].replace("|", "\\|")
        c = r["conclusion"].replace("|", "\\|")
        md += f"| {i} | {t} | {q} | {m_} | {c} |\n"
    md += "\n---\n"
    with open(path, "w", encoding="utf-8") as fh:
        fh.write(md)


def search_round(round_num, query):
    print(f"\n{'='*60}")
    print(f"[{time.strftime('%H:%M:%S')}] 第 {round_num} 轮搜索: {query}", flush=True)
    print(f"{'='*60}", flush=True)

    msg = f"搜索主题：{query}"
    try:
        resp = client.chat.completions.create(
            model="qwen3.5-plus",
            messages=[
                {"role": "system", "content": SEARCH_SYSTEM},
                {"role": "user", "content": msg},
            ],
            extra_body={
                "enable_search": True,
                "search_options": {"forced_search": True, "search_strategy": "max"},
            },
        )
        text = resp.choices[0].message.content
        print(f"[{time.strftime('%H:%M:%S')}] 第 {round_num} 轮返回 {len(text)} 字符", flush=True)
        return text
    except Exception as e:
        print(f"[{time.strftime('%H:%M:%S')}] 第 {round_num} 轮失败: {e}", flush=True)
        return None


# ============================================================
# 主流程
# ============================================================
def main():
    os.makedirs(OUTPUT_DIR, exist_ok=True)
    print(f"[{time.strftime('%H:%M:%S')}] A1 启动 — 主题: {THEME}", flush=True)
    print(f"预计耗时: 约 20 分钟 (5-10 轮联网搜索)", flush=True)
    print(f"ETA: {time.strftime('%H:%M', time.localtime(time.time() + 1200))}", flush=True)

    accumulated = []
    round_num = 1

    # 断点续跑检查
    existing_rounds = []
    for f in os.listdir(OUTPUT_DIR):
        m = re.match(r'topics_(\d+)\.md', f)
        if m:
            existing_rounds.append(int(m.group(1)))
    if existing_rounds:
        round_num = max(existing_rounds) + 1
        all_path = os.path.join(OUTPUT_DIR, "topics_all.md")
        if os.path.exists(all_path):
            with open(all_path, "r", encoding="utf-8") as fh:
                text = fh.read()
            accumulated = extract_rows(text)
            print(f"[{time.strftime('%H:%M:%S')}] 断点续跑: 已有 {len(existing_rounds)} 轮, {len(accumulated)} 篇去重, 从第 {round_num} 轮继续", flush=True)

    while round_num <= 10:
        tmpl_idx = (round_num - 1) % 4
        query = QUERY_TEMPLATES[tmpl_idx](THEME)

        text = search_round(round_num, query)
        if text is None:
            round_num += 1
            continue

        new_rows = extract_rows(text)
        print(f"  提取到 {len(new_rows)} 行", flush=True)
        for r in new_rows:
            print(f"    - {r['title'][:60]}...", flush=True)

        added = dedup(accumulated, new_rows)
        print(f"  去重后新增 {added} 篇，累计 {len(accumulated)} 篇", flush=True)

        save_md(new_rows,
                os.path.join(OUTPUT_DIR, f"topics_{round_num}.md"),
                f"{len(accumulated)} 篇")
        save_md(accumulated,
                os.path.join(OUTPUT_DIR, "topics_all.md"),
                f"{len(accumulated)} 篇")

        print(f"  ✓ 中间结果已保存: topics_{round_num}.md, topics_all.md", flush=True)

        round_num += 1

        if len(accumulated) >= 20 and round_num > 5:
            print(f"\n[{time.strftime('%H:%M:%S')}] 累计 {len(accumulated)} 篇，已满足 ≥20 篇 + ≥5 轮，停止搜索", flush=True)
            break

    print(f"\n[{time.strftime('%H:%M:%S')}] 搜索完成: {round_num-1} 轮, 去重后 {len(accumulated)} 篇", flush=True)

    # AI 重排序: 取 top 10
    if len(accumulated) > 10:
        print(f"\n[{time.strftime('%H:%M:%S')}] AI 重排序 {len(accumulated)} → top 10...", flush=True)

        table_text = "| 序号 | 文献标题 | 研究问题 | 方法数据 | 核心结论 |\n"
        table_text += "|------|----------|----------|----------|----------|\n"
        for i, r in enumerate(accumulated, 1):
            t = r["title"].replace("|", "\\|")
            q = r["question"].replace("|", "\\|")
            m_ = r["method"].replace("|", "\\|")
            c = r["conclusion"].replace("|", "\\|")
            table_text += f"| {i} | {t} | {q} | {m_} | {c} |\n"

        # ★ 排序规则需要根据 THEME 适配
        rank_prompt = f"""以下是关于"{THEME}"的{len(accumulated)}篇学术文献。请严格按与主题"{THEME}"的相关度从高到低重新排序，选出最相关的10篇。
仅输出排序后的表格，保持每篇原始内容不变。

{table_text}

请输出重排序后的 top 10 表格："""

        try:
            resp = client.chat.completions.create(
                model="qwen3.5-plus",
                messages=[{"role": "user", "content": rank_prompt}],
            )
            rank_text = resp.choices[0].message.content
            print(f"[{time.strftime('%H:%M:%S')}] 排序返回 {len(rank_text)} 字符", flush=True)

            ranked = extract_rows(rank_text)
            if len(ranked) >= 10:
                accumulated = ranked[:10]
            else:
                print(f"  警告: 排序只返回了 {len(ranked)} 行，保留原始 top 10", flush=True)
                accumulated = accumulated[:10]
        except Exception as e:
            print(f"  排序失败: {e}，保留原始 top 10", flush=True)
            accumulated = accumulated[:10]

    # 输出最终 topics.md
    final_path = os.path.join(OUTPUT_DIR, "topics.md")
    with open(final_path, "w", encoding="utf-8") as fh:
        fh.write(f"# 学术文献主题概览: {THEME}\n\n")
        fh.write(f"搜索执行了多轮，累计去重后得到 **{len(accumulated)} 篇** 文献，按主题相关度排序：\n\n")
        fh.write("---\n\n")
        fh.write("| 排名 | 文献标题 | 研究问题 | 方法数据 | 核心结论 |\n")
        fh.write("|------|----------|----------|----------|----------|\n")
        for i, r in enumerate(accumulated, 1):
            t = r["title"].replace("|", "\\|")
            q = r["question"].replace("|", "\\|")
            m_ = r["method"].replace("|", "\\|")
            c = r["conclusion"].replace("|", "\\|")
            fh.write(f"| {i} | {t} | {q} | {m_} | {c} |\n")
        fh.write("\n---\n")

    print(f"\n[{time.strftime('%H:%M:%S')}] ✓ A1 完成! 输出: {final_path}", flush=True)

    print(f"\n{'='*60}")
    print("最终 Top 10 文献列表:")
    print(f"{'='*60}")
    for i, r in enumerate(accumulated, 1):
        print(f"  {i}. {r['title']}")


if __name__ == "__main__":
    main()
```
