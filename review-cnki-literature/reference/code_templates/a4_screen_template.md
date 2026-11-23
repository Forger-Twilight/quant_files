# A4 脚本模板：题录 LLM 初筛

复制到项目目录（如 `code/a4_screen.py`）后，只改配置区再运行。

| 配置项 | 说明 |
|--------|------|
| `THEME` | 用户的研究主题 |
| `WORK_DIR` | A3 产出目录 `output/downloads/{theme}_{timestamp}` |
| `YEAR_START` | 入选最早年份（A→B gate 用户确认后设置；`None` 不过滤） |
| `BATCH_SIZE` / `ABSTRACT_MAX_CHARS` | 每批 25 条 / 摘要截断 500 字 |
| `POSITIVE_CRITERIA` / `NEGATIVE_CRITERIA` | 由 agent 按 [extraction_guide.md](../summary/extraction_guide.md) 扩写主题后填入（反面默认含"非 A 股市场"约束） |
| `TARGET_COUNT` | 目标入选规模（v1.1 规则 N6，默认 10；`None` 不控制）。超出 → LLM 精选 top N；不足 → 打印发散匹配建议（补检索由 Agent 协调，脚本不自行重检索） |
| `SMOKE_TEST` | 冒烟：`BATCH_SIZE=5` 且只跑 2 批 |

LLM 约定（client/call_llm/禁 timeout）见 [../shared/llm_client.md](../shared/llm_client.md)。

```python
# -*- coding: utf-8 -*-
"""
A4: 题录 LLM 初筛 — 宽松判定(宁滥勿缺;v1.1 起无 B3,超出目标规模由 LLM 精选 top N 控制)
输入: WORK_DIR/records_all.jsonl
产出: WORK_DIR/records_screened.jsonl (全量判定,断点续跑), WORK_DIR/selected.jsonl (入选)
"""
import sys, os, re, json, time
sys.stdout.reconfigure(encoding='utf-8')
sys.stderr.reconfigure(encoding='utf-8')

from openai import OpenAI

# ============================================================
# 配置（★ 每次运行前修改）
# ============================================================
THEME = "<用户的研究主题>"
WORK_DIR = "output/downloads/<theme_timestamp>"
YEAR_START = None           # 如 2021
BATCH_SIZE = 25
ABSTRACT_MAX_CHARS = 500
TARGET_COUNT = 10           # 目标入选规模(规则 N6); None = 不控制
SMOKE_TEST = False

# ★ 由 agent 读 extraction_guide.md 扩写主题后填入
POSITIVE_CRITERIA = """<正面判定标准: 什么算相关>"""
NEGATIVE_CRITERIA = """<反面判定标准: 什么算不相关(枚举沾边但排除的类型)>"""

api_key = sys.argv[1] if len(sys.argv) > 1 else input("请输入 ALIYUN_API_KEY: ").strip()
client = OpenAI(api_key=api_key, base_url="https://dashscope.aliyuncs.com/compatible-mode/v1")

def call_llm(prompt, model="qwen-long", max_retries=2):
    for attempt in range(max_retries + 1):
        try:
            print(f"  [{time.strftime('%H:%M:%S')}] 调用 DashScope ({model})...", flush=True)
            resp = client.chat.completions.create(
                model=model,
                messages=[{"role": "user", "content": prompt}],
                response_format={"type": "json_object"},
            )
            print(f"  [{time.strftime('%H:%M:%S')}] API 返回成功", flush=True)
            return resp.choices[0].message.content
        except Exception as e:
            print(f"  LLM调用失败(attempt {attempt+1}): {e}", flush=True)
            if attempt < max_retries:
                time.sleep(2)
    return None

def robust_parse_list(text):
    """数组版容错解析(见 shared/llm_client.md);兼容 {"results": [...]} 包裹与裸数组"""
    if not text:
        return []
    m = re.search(r"\{.*\}", text, re.DOTALL)
    if m:
        try:
            data = json.loads(m.group())
            if isinstance(data, dict) and isinstance(data.get("results"), list):
                return data["results"]
        except json.JSONDecodeError:
            pass
    m = re.search(r"\[.*\]", text, re.DOTALL)
    if m:
        try:
            data = json.loads(m.group())
            if isinstance(data, list):
                return data
        except json.JSONDecodeError:
            pass
    items = []
    for m2 in re.finditer(
            r'\{\s*"idx"\s*:\s*(\d+)\s*,\s*"label"\s*:\s*"(relevant|borderline|irrelevant)"\s*,\s*"reason"\s*:\s*"([^"]*)"\s*\}',
            text):
        items.append({"idx": int(m2.group(1)), "label": m2.group(2), "reason": m2.group(3)})
    return items

PROMPT_TMPL = """你是学术研究助手。给定研究主题和一批中文学术文献题录(标题/期刊/年份/关键词/摘要),判断每篇是否与主题相关。

## 研究主题
{theme}

## 判定标准
正面(相关): {positive}
反面(不相关): {negative}

## 判定原则
- 宽松判定,宁滥勿缺: 漏掉的代价远大于误判的代价(v1.1 起无 B3 再筛;入选超出目标规模默认 ~10 篇时,追加一轮 LLM 按相关度精选 top N)
- 摘要声称即采信: 题录信息有限,无法验证的说法不判负
- 无摘要条目: 用标题+关键词判断,reason 注明"无摘要"
- 核心标准部分满足、无法确定的,给 borderline
- 即使标题出现主题关键词,但从摘要看核心内容不围绕主题,判 irrelevant

## 文献列表
{items}

## 输出
只输出 JSON 对象,不要任何解释:
{{"results": [{{"idx": 1, "label": "relevant"|"borderline"|"irrelevant", "reason": "20字内依据"}}, ...]}}
reason 中不要包含大括号。"""

RANK_TMPL = """你是学术研究助手。研究主题: {theme}
正面标准: {positive}
反面标准: {negative}

以下 {n} 篇题录已通过初筛,但数量超过目标规模 {target} 篇。请按与主题的相关度从高到低排序,选出最相关的 {target} 篇。

## 题录列表
{items}

## 输出
只输出 JSON 对象,不要任何解释:
{{"ranking": [最相关的编号, 次相关的编号, ...]}}
恰好 {target} 个编号,编号须在 1~{n} 范围内且不重复。"""

def parse_ranking(text, n):
    """解析精选排序;失败返回 None 由调用方 fallback"""
    if not text:
        return None
    m = re.search(r'\{.*\}', text, re.DOTALL)
    if not m:
        return None
    try:
        data = json.loads(m.group())
        ranking = data.get("ranking")
        if isinstance(ranking, list):
            valid = [int(i) for i in ranking if isinstance(i, (int, float, str)) and str(i).isdigit() and 1 <= int(i) <= n]
            if len(set(valid)) == len(valid) and valid:
                return valid
    except (json.JSONDecodeError, ValueError):
        pass
    return None

def format_item(idx, r):
    abs_ = (r.get("abstract") or "").strip()[:ABSTRACT_MAX_CHARS] or "(无摘要)"
    kw = ",".join(r.get("keywords") or [])[:100] or "(无关键词)"
    return f"{idx}. {r.get('title','')} | {r.get('journal','')} | {r.get('year','')} | 关键词: {kw} | 摘要: {abs_}"

def main():
    records_path = os.path.join(WORK_DIR, "records_all.jsonl")
    screened_path = os.path.join(WORK_DIR, "records_screened.jsonl")
    selected_path = os.path.join(WORK_DIR, "selected.jsonl")

    records = [json.loads(l) for l in open(records_path, encoding="utf-8")]
    print(f"[A4] 载入 {len(records)} 条题录", flush=True)

    if YEAR_START:
        records = [r for r in records if (r.get("year") or "0") >= str(YEAR_START)]
        print(f"[A4] 年份过滤(>={YEAR_START})后: {len(records)} 条", flush=True)

    # 断点续跑: 跳过已筛 id
    done_ids = set()
    if os.path.exists(screened_path):
        for l in open(screened_path, encoding="utf-8"):
            done_ids.add(json.loads(l)["id"])
        print(f"[A4] 断点续跑: 已筛 {len(done_ids)} 条", flush=True)

    todo = [r for r in records if r["id"] not in done_ids]
    batch_size = 5 if SMOKE_TEST else BATCH_SIZE
    max_batch = 2 if SMOKE_TEST else None
    batches = [todo[i:i+batch_size] for i in range(0, len(todo), batch_size)]
    if max_batch:
        batches = batches[:max_batch]
    print(f"[A4] 待筛 {len(todo)} 条,分 {len(batches)} 批(每批 {batch_size})", flush=True)

    with open(screened_path, "a", encoding="utf-8") as fout:
        for bi, batch in enumerate(batches, 1):
            print(f"\n[A4] 批次 {bi}/{len(batches)} ({len(batch)} 条)", flush=True)
            items_text = "\n".join(format_item(i+1, r) for i, r in enumerate(batch))
            prompt = PROMPT_TMPL.format(theme=THEME, positive=POSITIVE_CRITERIA,
                                        negative=NEGATIVE_CRITERIA, items=items_text)
            raw = call_llm(prompt)
            results = robust_parse_list(raw) if raw else []
            got = {d["idx"]: d for d in results if isinstance(d, dict) and "idx" in d}
            if len(got) < len(batch):
                print(f"  ⚠️ 解析条数 {len(got)} < 批大小 {len(batch)},重试一次", flush=True)
                raw = call_llm(prompt)
                results = robust_parse_list(raw) if raw else []
                got = {d["idx"]: d for d in results if isinstance(d, dict) and "idx" in d}
            if len(got) < len(batch):
                # 规则10: 无声失败即 bug,输出诊断
                print(f"  ⚠️ 仍缺 {len(batch)-len(got)} 条;LLM 原始响应前500字: {(raw or '')[:500]!r}", flush=True)
            for i, r in enumerate(batch, 1):
                d = got.get(i)
                out = dict(r)
                if d:
                    out["label"] = d.get("label", "error")
                    out["reason"] = d.get("reason", "")
                else:
                    out["label"] = "error"
                    out["reason"] = "LLM 未返回该条"
                out["batch"] = bi
                fout.write(json.dumps(out, ensure_ascii=False) + "\n")
                fout.flush()
            time.sleep(2)

    # 汇总 selected.jsonl
    label_count = {}
    selected = []
    for l in open(screened_path, encoding="utf-8"):
        r = json.loads(l)
        label_count[r["label"]] = label_count.get(r["label"], 0) + 1
        if r["label"] in ("relevant", "borderline"):
            selected.append(r)

    print(f"\n[A4] ✓ 判定分布: {label_count}", flush=True)

    # ── N6: 目标规模控制 ──────────────────────────────────────
    if TARGET_COUNT and len(selected) > TARGET_COUNT:
        print(f"[A4] 入选 {len(selected)} > 目标 {TARGET_COUNT},LLM 精选 top {TARGET_COUNT}...", flush=True)
        items_text = "\n".join(
            f"{i}. {r.get('title','')} | {r.get('journal','')} | {r.get('year','')} | 摘要: {(r.get('abstract') or '')[:200]}"
            for i, r in enumerate(selected, 1))
        prompt = RANK_TMPL.format(theme=THEME, positive=POSITIVE_CRITERIA,
                                  negative=NEGATIVE_CRITERIA,
                                  n=len(selected), target=TARGET_COUNT, items=items_text)
        ranking = parse_ranking(call_llm(prompt), len(selected))
        if ranking and len(ranking) >= TARGET_COUNT:
            keep_idx = set(ranking[:TARGET_COUNT])
        else:
            # fallback(规则10诊断): relevant 优先,保持原顺序取前 N
            print(f"  ⚠️ 精选解析失败或数量不足(原始响应见上),fallback: relevant 优先取前 {TARGET_COUNT}", flush=True)
            ordered = [r for r in selected if r["label"] == "relevant"] + \
                      [r for r in selected if r["label"] == "borderline"]
            keep_idx = {selected.index(r) + 1 for r in ordered[:TARGET_COUNT]}
        overflow = [r for i, r in enumerate(selected, 1) if i not in keep_idx]
        selected = [r for i, r in enumerate(selected, 1) if i in keep_idx]
        overflow_path = os.path.join(WORK_DIR, "overflow.jsonl")
        with open(overflow_path, "w", encoding="utf-8") as f:
            for r in overflow:
                f.write(json.dumps(r, ensure_ascii=False) + "\n")
        print(f"[A4] ✓ 精选 {len(selected)} 条入选;{len(overflow)} 条溢出 -> {overflow_path}(留档不下载)", flush=True)

    elif TARGET_COUNT and len(selected) < TARGET_COUNT:
        # 发散匹配补检索需浏览器重跑 A3,脚本只负责触发与建议(规则 N6: 先向用户说明再执行)
        print(f"\n[A4] ⚠️ 入选 {len(selected)} < 目标 {TARGET_COUNT}(规则 N6): 触发发散匹配补检索", flush=True)
        print("[A4] 发散建议(按序尝试,Agent 向用户说明后执行):", flush=True)
        print("  1) 关键词组中的 [精确] 行改为 [模糊]；A股范围不放开", flush=True)
        print("  2) 为近义词新增 OR 篇关摘行（每框一词），或追加独立检索组", flush=True)
        print("  3) 放宽数据库范围(如纳入学位论文)或延长年限", flush=True)
        print("  补检索后重跑本脚本即可(断点续跑,已筛题录不重复判定)", flush=True)

    with open(selected_path, "w", encoding="utf-8") as f:
        for r in selected:
            f.write(json.dumps(r, ensure_ascii=False) + "\n")

    print(f"[A4] ✓ 入选 {len(selected)} 条 -> {selected_path}", flush=True)
    if label_count.get("error"):
        print(f"[A4] ⚠️ {label_count['error']} 条判定失败(label=error),请检查后可重跑(断点续跑只补 error 需先删除对应行)", flush=True)
    if len(selected) == 0:
        print("[A4] ⚠️ 0 入选(规则10): 请检查判定标准是否过严/LLM 响应", flush=True)

if __name__ == "__main__":
    main()
```
