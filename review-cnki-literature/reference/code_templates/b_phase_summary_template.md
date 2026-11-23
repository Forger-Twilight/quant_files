# B 阶段脚本模板：文献综述（B1 fallback→B2→B4;B3 严格再筛 v1.1 起默认停用）

复制到项目目录后，修改以下配置项再运行：

| 配置项 | 说明 |
|--------|------|
| `THEME` | 用户的研究主题 |
| `MODE` | `"full"`（完整模式：A4 已筛，跳过 B1）或 `"review_only"`（仅综述：用户自带 PDF，跑 B1 fallback） |
| `PAPERS_DIR` | A5 输出的 `papers/` 目录（完整模式）或用户指定的 PDF 目录（仅综述模式） |
| `OUTPUT_DIR` | B 阶段输出目录路径 |
| `POSITIVE_CRITERIA` | 扩写后的正面判定标准（参考 `../summary/extraction_guide.md`） |
| `NEGATIVE_CRITERIA` | 扩写后的反面判定标准（默认含"非 A 股市场"约束,见 extraction_guide） |
| `ENABLE_B3` | 默认 `False`(v1.1 起无排除机制,入选全部进入综述);如需恢复 B3 严格再筛查档,设 `True` 并适配 prompt(见 `../prompts/strict_filter_prompt.md`) |

```python
"""
B 阶段：学术文献综述 (B1 fallback → B2 → B4; B3 默认停用, ENABLE_B3=True 恢复)
输入：完整模式 = A5 的 papers/ 目录; 仅综述模式 = 用户自带 PDF 目录
"""
import sys
import os
import re
import json
import time
import shutil
import logging
from pathlib import Path

sys.stdout.reconfigure(encoding='utf-8')
os.environ['PYTHONUNBUFFERED'] = '1'

from openai import OpenAI

# ── 配置（★ 每次运行前修改）──────────────────────────────────────
THEME = "<用户的研究主题>"
MODE = "full"               # "full" | "review_only"
PAPERS_DIR = Path("<A5 输出的 papers/ 目录 或 用户自带 PDF 目录>")
OUTPUT_DIR = Path("output/b_phase_summary")
OUTPUT_DIR.mkdir(parents=True, exist_ok=True)

ENABLE_B3 = False           # v1.1 起默认停用 B3 严格再筛(无排除机制); True 可恢复

INDIVIDUAL_DIR = OUTPUT_DIR / "individual"
INDIVIDUAL_DIR.mkdir(exist_ok=True)
DROPPED_DIR = OUTPUT_DIR / "individual_dropped"
DROPPED_DIR.mkdir(exist_ok=True)

api_key = sys.argv[1] if len(sys.argv) > 1 else input("请输入 ALIYUN_API_KEY: ").strip()
client = OpenAI(
    api_key=api_key,
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
)

# ── Prompt 扩写（★ 根据 extraction_guide.md 适配）──────────────────
POSITIVE_CRITERIA = "<扩写后的正面判定标准>"
NEGATIVE_CRITERIA = "<扩写后的反面判定标准>"

# ── 工具函数 ──────────────────────────────────────────────────

def call_llm(prompt, model='qwen-long', json_mode=True, max_retries=3):
    for attempt in range(max_retries):
        try:
            print(f"  [{time.strftime('%H:%M:%S')}] Calling {model}...", flush=True)
            kwargs = dict(
                model=model,
                messages=[{'role': 'user', 'content': prompt}],
            )
            if json_mode:
                kwargs['response_format'] = {"type": "json_object"}
            resp = client.chat.completions.create(**kwargs)
            print(f"  [{time.strftime('%H:%M:%S')}] OK", flush=True)
            return resp.choices[0].message.content
        except Exception as e:
            print(f"  LLM error (attempt {attempt+1}): {e}", flush=True)
            if attempt < max_retries - 1:
                time.sleep(3)
    return None


def robust_parse(text):
    if not text:
        return {'label': 'error', 'reason': '空响应'}
    m = re.search(r'\{.*\}', text, re.DOTALL)
    if m:
        try:
            return json.loads(m.group())
        except json.JSONDecodeError:
            pass
    label_m = re.search(r'"label"\s*:\s*"(relevant|irrelevant)"', text)
    if label_m:
        reason_m = re.search(r'"reason"\s*:\s*"([^"]*)"', text)
        return {'label': label_m.group(1),
                'reason': reason_m.group(1) if reason_m else '(reason解析失败)'}
    return {'label': 'error', 'reason': f'无法解析: {text[:200]}'}


def extract_section(md_text, section_name):
    pattern = rf'## {re.escape(section_name)}\s*\n(.*?)(?=\n## |\Z)'
    m = re.search(pattern, md_text, re.DOTALL)
    return m.group(1).strip() if m else ''


def sanitize_dollars(text):
    placeholders = []
    def stash(m):
        body = m.group(1)
        if '$' in body:
            body = body.replace('$', '')
        placeholders.append(f'$${body}$$')
        return f'\x00BLOCK{len(placeholders)-1}\x00'
    text = re.sub(r'\$\$(.*?)\$\$', stash, text, flags=re.DOTALL)
    text = text.replace('$', '')
    text = re.sub(r'\x00BLOCK(\d+)\x00', lambda m: placeholders[int(m.group(1))], text)
    text = re.sub(r'(\$\$)  +(?=\n|$)', r'\1', text)
    def wrap(m):
        return f'\n\n{m.group(0)}\n\n'
    text = re.sub(r'\$\$.*?\$\$', wrap, text, flags=re.DOTALL)
    text = re.sub(r'\n{3,}', '\n\n', text)
    return text.strip() + '\n'


def parse_filename(filename):
    """{year}-{journal}-{title}.pdf (A5 命名); 兼容旧格式与未知格式"""
    name = filename.replace('.pdf', '').replace('.md', '')
    m = re.match(r'(\d{4})-(.+?)-(.+)', name)
    if m:
        return m.group(1), m.group(2).strip(), m.group(3).strip()
    return '0000', '未知', name


# ── B1: 初筛(fallback: 仅 review_only 模式执行) ──────────────────

def b1_filter():
    """仅综述模式(用户自带无题录 PDF)的前置粗筛; 完整模式已由 A4 题录初筛替代"""
    print("=" * 60, flush=True)
    print("[B1] 粗筛(fallback) — 基于前3页", flush=True)
    print("=" * 60, flush=True)

    import fitz
    pdfs = sorted(PAPERS_DIR.glob("*.pdf"))
    print(f"[B1] {len(pdfs)} PDFs to screen", flush=True)

    results = []
    for i, pdf_path in enumerate(pdfs):
        print(f"\n[B1] ({i+1}/{len(pdfs)}) {pdf_path.name}", flush=True)
        year, journal, title = parse_filename(pdf_path.name)

        try:
            doc = fitz.open(str(pdf_path))
            pages_text = []
            for j in range(min(3, doc.page_count)):
                pages_text.append(doc[j].get_text())
            doc.close()
            first_3 = '\n'.join(pages_text)[:4000]
        except Exception as e:
            print(f"  [ERR] PDF read: {e}", flush=True)
            results.append({'filename': pdf_path.name, 'label': 'error', 'reason': str(e)})
            continue

        # Prompt 完整说明见 reference/prompts/filter_prompt.md
        prompt = f"""你是一名学术研究助手。请判断以下文献是否与主题「{THEME}」相关。

判定标准：
- 相关(relevant)：{POSITIVE_CRITERIA}
- 不相关(irrelevant)：{NEGATIVE_CRITERIA}

注意：即使文献中提到了相关字样，如果其核心内容并非围绕该主题展开，也应判定为不相关。

请严格按照以下 JSON 格式回答，不要输出其他内容：
{{"label": "relevant" 或 "irrelevant", "reason": "一句话说明理由"}}

---
文献标题：{title}
期刊：{journal}，发表年份：{year}
前3页内容：
{first_3}"""

        content = call_llm(prompt)
        parsed = robust_parse(content)
        results.append({'filename': pdf_path.name, 'title': title, **parsed})
        print(f"  -> {parsed['label']}: {parsed['reason'][:80]}", flush=True)

    csv_path = OUTPUT_DIR / "filter_result.csv"
    with open(csv_path, 'w', encoding='utf-8') as f:
        f.write("filename,title,label,reason\n")
        for r in results:
            f.write(f"{r['filename']},{r.get('title','')},{r['label']},{r['reason']}\n")

    relevant = [r for r in results if r['label'] == 'relevant']
    print(f"\n[B1] Relevant: {len(relevant)}/{len(results)}", flush=True)
    return relevant


# ── B2: 逐篇摘要 ───────────────────────────────────────────────

def b2_summarize(relevant_list):
    print("\n" + "=" * 60, flush=True)
    print(f"[B2] 逐篇摘要 — {len(relevant_list)} papers", flush=True)
    print("=" * 60, flush=True)

    import fitz

    for i, item in enumerate(relevant_list):
        pdf_path = PAPERS_DIR / item['filename']
        md_path = INDIVIDUAL_DIR / (pdf_path.stem + '.md')

        if md_path.exists():
            print(f"[B2] ({i+1}/{len(relevant_list)}) SKIP (cached): {pdf_path.name}", flush=True)
            continue

        print(f"\n[B2] ({i+1}/{len(relevant_list)}) {pdf_path.name}", flush=True)
        year, journal, title = parse_filename(pdf_path.name)

        try:
            text_parts = []
            doc = fitz.open(str(pdf_path))
            page_count = doc.page_count
            for pg in doc:
                text_parts.append(pg.get_text())
            doc.close()
            full_text = '\n'.join(text_parts)[:30000]
        except Exception as e:
            print(f"  [ERR] PDF: {e}", flush=True)
            continue

        # Prompt 完整内容见 reference/prompts/summary_prompt.md(学术六段版)
        prompt = f"""你是一名学术研究助手。请对以下学术文献进行结构化摘要。
本次文献综述的预设主题是「{THEME}」，请围绕该主题提取最相关的信息。

## 输出要求

请严格按照以下 Markdown 模板输出，不要添加额外章节，不要用代码块包裹输出：

# {{文献标题}}
- 期刊：{{期刊名}}
- 作者：{{作者}}
- 发表年份：{{年份}}
- 页数：{{页数}}
- 数据来源：{{数据获取的具体来源：数据库写明名称(如 Wind/CSMAR/RESSET/Choice/iFinD/交易所官网)；爬虫写明爬取对象和平台；公开数据写明发布机构；文献未交代写"未说明"}}

## 核心思路
{{用3-5句话概括该文献的主要研究思路/方法论}}

## 数据与样本
{{数据来源(与头部字段一致,可补细节)、样本区间、研究对象/市场;无则写"未说明"}}

## 模型与方法构建
{{完整、客观抄录模型/指标的数学定义/函数形式,保留原始公式符号}}

## 理论解释与机制分析
{{精炼总结结论背后的理论直觉或机制解释,1-3句话}}

## 异质性与边界条件
{{提取文献中提到的结论在某些条件下成立/不成立的内容(分样本、调节效应、适用边界)}}

## 关键创新点
{{该文献相对于已有研究的差异化贡献,1-2句}}

## 关键原则
1. 只关心思路，完全不关心实证结果的具体系数与显著性数字
2. 公式必须完整抄录
3. 理论解释要精炼
4. 异质性与边界条件要具体
5. 不要用代码块包裹输出
6. 所有数学公式必须使用 LaTeX 块公式语法 $$...$$ 包裹
7. 严禁任何形式的行内 $x$，变量注释用纯文字

---
文献标题：{title}
期刊：{journal}
发表年份：{year}
页数：{page_count}

文献全文：
{full_text}"""

        content = call_llm(prompt, json_mode=False)
        if not content:
            continue

        if content.startswith('```'):
            content = re.sub(r'^```\w*\n?', '', content)
            content = re.sub(r'\n?```$', '', content)
        content = re.split(r'\n## 关键原则', content)[0]
        content = sanitize_dollars(content)

        md_path.write_text(content, encoding='utf-8')

    print(f"\n[B2] Done. {len(list(INDIVIDUAL_DIR.glob('*.md')))} summaries generated.", flush=True)


# ── B3: 严格再筛 ──────────────────────────────────────────────

def b3_strict_filter():
    print("\n" + "=" * 60, flush=True)
    print("[B3] 严格再筛 — 基于 individual 摘要", flush=True)
    print("=" * 60, flush=True)

    md_files = sorted(INDIVIDUAL_DIR.glob("*.md"))
    print(f"[B3] {len(md_files)} summaries to re-screen", flush=True)

    results = []
    for i, md_path in enumerate(md_files):
        print(f"\n[B3] ({i+1}/{len(md_files)}) {md_path.name}", flush=True)
        text = md_path.read_text(encoding='utf-8')

        title_m = re.match(r'#\s+(.+)', text)
        title = title_m.group(1).strip() if title_m else md_path.stem
        core_idea = extract_section(text, '核心思路')
        method = extract_section(text, '模型与方法构建')
        innovation = extract_section(text, '关键创新点')

        # ★ 严格再筛 prompt 需根据 THEME 适配（见 reference/prompts/strict_filter_prompt.md）
        prompt = f"""你是一名严格的学术研究助手。请用最严格的标准判断以下文献是否「直接」研究主题「{THEME}」。

# 严格判定标准（必须同时满足两条才算 relevant）：
A. 文献的核心方法/研究设计与{THEME}直接相关
B. 该文献对该主题有原创性贡献(非综述、非泛泛而谈)

# 明确判为 irrelevant 的情况（按主题由 agent 补充调整）：
1. 综述/评述类（无原创方法）
2. 纯理论模型无实证（当主题要求实证时）
3. 宏观/行业层面而非个体资产层面
4. 教学方法/课程研究
5. 案例研究/单一案例
6. 学位论文与已入选期刊版同题

请严格按以下 JSON 格式回答（reason 中不要含大括号）：
{{"label": "relevant" 或 "irrelevant", "reason": "一句话"}}

---
文献文件名：{md_path.name}
文献标题：{title}

【核心思路】
{core_idea[:1500]}

【模型与方法构建】
{method[:1500]}

【关键创新点】
{innovation[:600]}"""

        content = call_llm(prompt)
        parsed = robust_parse(content)
        results.append({'filename': md_path.name, 'title': title, **parsed})
        print(f"  -> {parsed['label']}: {parsed['reason'][:80]}", flush=True)

        if parsed['label'] != 'relevant':
            shutil.move(str(md_path), str(DROPPED_DIR / md_path.name))

    csv_path = OUTPUT_DIR / "strict_filter_result.csv"
    with open(csv_path, 'w', encoding='utf-8') as f:
        f.write("filename,title,label,reason\n")
        for r in results:
            f.write(f"{r['filename']},{r['title']},{r['label']},{r['reason']}\n")

    kept = [r for r in results if r['label'] == 'relevant']
    dropped = [r for r in results if r['label'] != 'relevant']
    print(f"\n[B3] Kept: {len(kept)}, Dropped: {len(dropped)}", flush=True)
    return kept


# ── B4: 主题汇总 ──────────────────────────────────────────────
# 完整实现(2026-11-06 冒烟验证通过); 机制与实战陷阱见 reference/summary/two_step_strategy.md

# -*- coding: utf-8 -*-
# B4 完整实现(两步策略) — 用于替换 b_phase_summary_template.md 中的 b4 骨架
# 机制见 reference/summary/two_step_strategy.md

def get_first_author(md_text):
    """从 individual.md 头部 '- 作者：张三, 李四' 行提取第一作者"""
    m = re.search(r'^-\s*作者[：:](.+)$', md_text, re.M)
    if not m:
        return ''
    return re.split(r'[,，;；、]', m.group(1))[0].strip()


def get_data_source(md_text):
    """从 individual.md 头部 '- 数据来源：...' 行提取数据来源(B2 模板必填字段; 缺失返回"未提取")"""
    m = re.search(r'^-\s*数据来源[：:](.+)$', md_text, re.M)
    return m.group(1).strip() if m else '未提取'


def auto_fold_series(themes, author_by_filename):
    """同一第一作者 + 标题归一化前缀 80% 相似 → 折叠为 duplicate_groups
    (对应学位论文/期刊版、中英文双发等同团队重复发表)"""
    from difflib import SequenceMatcher

    def norm_prefix(title):
        t = re.sub(r'\d+\.?\d*%?', '', title)
        t = re.sub(r'第[一二三四五六七八九十百\d]+期', '', t)
        t = re.sub(r'[（(][^）)]*[）)]', '', t)
        t = re.sub(r'[：:—–-].*$', '', t)
        t = re.sub(r'\s+', '', t)
        return t[:20]

    def is_thesis(journal, title):
        return ('学位论文' in journal) or bool(re.search(r'[硕博]士', title))

    for th in themes:
        members = th['members']
        by_first_author = {}
        for fn in members:
            year, journal, title = parse_filename(fn)
            fa = author_by_filename.get(fn, '')
            by_first_author.setdefault(fa, []).append((year, journal, title, fn))
        new_members = []
        new_dups = dict(th.get('duplicate_groups') or {})
        for author, items in by_first_author.items():
            if not author or len(items) == 1:
                new_members.extend(it[3] for it in items)
                continue
            clusters = []
            for it in items:
                key = norm_prefix(it[2])
                placed = False
                for cl in clusters:
                    if SequenceMatcher(None, key, norm_prefix(cl[0][2])).ratio() >= 0.8:
                        cl.append(it)
                        placed = True
                        break
                if not placed:
                    clusters.append([it])
            for cl in clusters:
                cl.sort(key=lambda x: (is_thesis(x[1], x[2]), x[0]))  # 期刊版优先,同类取最早
                new_members.append(cl[0][3])
                followers = [g[3] for g in cl[1:]]
                if followers:
                    new_dups.setdefault(cl[0][3], []).extend(followers)
        ordered = [fn for fn in members if fn in new_members]
        for fn in new_members:
            if fn not in ordered:
                ordered.append(fn)
        th['members'] = ordered
        th['duplicate_groups'] = new_dups
        if th.get('flagship') not in th['members']:
            th['flagship'] = th['members'][0] if th['members'] else None
    return themes


def b4_thematic_summary(kept_list):
    """两步策略: LLM 只产 JSON 分组; 代码做去重/校验/折叠/拼装(零幻觉)"""
    print("\n" + "=" * 60, flush=True)
    print(f"[B4] 主题汇总 — {len(kept_list)} papers", flush=True)
    print("=" * 60, flush=True)

    remaining_files = sorted(INDIVIDUAL_DIR.glob("*.md"))
    all_filenames = {f.name for f in remaining_files}
    items_by_filename = {f.name: f.read_text(encoding='utf-8') for f in remaining_files}

    # 喂给 LLM 的压缩摘要
    parts = []
    for name, text in items_by_filename.items():
        title_m = re.match(r'#\s+(.+)', text)
        title = title_m.group(1).strip() if title_m else name
        parts.append(
            f"### {name}\n**标题**: {title}\n"
            f"**核心思路**: {extract_section(text, '核心思路')[:600]}\n"
            f"**理论解释**: {extract_section(text, '理论解释与机制分析')[:400]}\n"
            f"**异质性**: {extract_section(text, '异质性与边界条件')[:300]}\n"
            f"**关键创新点**: {extract_section(text, '关键创新点')[:300]}\n")
    all_summaries_text = '\n\n'.join(parts)

    # ── 第一步: LLM 归类(带缓存; 调 prompt 删缓存重跑, 调拼装不删) ──
    cache_path = OUTPUT_DIR / "theme_groups.json"
    if cache_path.exists():
        print("[B4] Using cached theme_groups.json", flush=True)
        group_data = json.loads(cache_path.read_text(encoding='utf-8'))
    else:
        # ★ 完整 prompt 规范见 reference/prompts/thematic_prompt.md
        prompt = f"""你是一名学术研究助手。以下是多份与「{THEME}」主题相关的中文学术文献摘要(文件名 + 压缩摘要)。
请将这些文献按照**研究思路的相似性**归纳为若干主题类别。

## 归纳原则
1. 按研究思路/方法论的相似性归类,不按期刊或年份
2. 主题名要通俗直白(一句话说得清这类研究在干什么)
3. 每篇文献必须且只能归入一个主题
4. 每个主题给出 flagship(开山之作/方法最完整/最经典的那篇,必须是 members 之一)
5. members 里的文件名必须原样照抄上方给出的文件名,不得改写
6. 单主题过大时按方法论细分

## 输出(只输出 JSON,不要代码块,不要解释)
{{"themes": [{{
  "name": "主题名",
  "core_idea": "该主题共同的核心思路(3-5句)",
  "econ": "理论解释/机制逻辑(1-3句)",
  "interaction": "异质性与边界条件(什么时候成立/不成立)",
  "members": ["文件名1.md", "文件名2.md"],
  "duplicate_groups": {{"代表作文件名": ["同团队重复版本文件名"]}},
  "flagship": "代表作文件名"
}}]}}

---
所有文献摘要:
{all_summaries_text[:50000]}"""

        content = call_llm(prompt)
        if not content:
            print("[B4] LLM call failed", flush=True)
            return
        try:
            content_clean = content.strip()
            if content_clean.startswith('```'):
                content_clean = re.sub(r'^```\w*\n?', '', content_clean)
                content_clean = re.sub(r'\n?```$', '', content_clean)
            group_data = json.loads(content_clean)
        except json.JSONDecodeError:
            m = re.search(r'\{.*\}', content, re.DOTALL)
            if not m:
                print(f"[B4] JSON parse failed: {content[:300]}", flush=True)
                return
            group_data = json.loads(m.group())
        cache_path.write_text(json.dumps(group_data, ensure_ascii=False, indent=2), encoding='utf-8')

    themes = group_data.get('themes', [])

    # ── 第二步 (a) 按首次出现去重 + 过滤幻觉文件名 + flagship 校验 ──
    seen, covered = set(), set()
    for th in themes:
        new_members = []
        for fn in th.get('members', []):
            if fn not in seen and fn in all_filenames:
                new_members.append(fn)
                seen.add(fn)
        th['members'] = new_members
        covered.update(new_members)
        if th.get('flagship') not in all_filenames:
            th['flagship'] = th['members'][0] if th['members'] else None
    themes = [th for th in themes if th['members']]

    # (b) auto_fold_series: 同第一作者同题重复发表折叠
    author_by_filename = {n: get_first_author(t) for n, t in items_by_filename.items()}
    themes = auto_fold_series(themes, author_by_filename)

    # (c) duplicate_groups 校验(followers 计入覆盖,幻觉丢弃)
    for th in themes:
        clean_dups = {}
        for leader, followers in (th.get('duplicate_groups') or {}).items():
            if leader not in th['members']:
                # leader 已被去重分到其他主题时不得拉回(实测: LLM 把同一篇同时列进两个主题的 members/dups 会导致汇总重复)
                if leader in all_filenames and leader not in covered:
                    th['members'].append(leader)
                    covered.add(leader)
                else:
                    continue
            valid = [f for f in followers if f in all_filenames and f != leader]
            if valid:
                clean_dups[leader] = valid
                covered.update(valid)
        th['duplicate_groups'] = clean_dups

    # (d) 兜底「其他」主题
    uncovered = sorted(all_filenames - covered)
    if uncovered:
        themes.append({
            'name': '其他相关研究',
            'core_idea': '此类文献与主题相关，但研究思路不属于上述主题归并的方法论范畴，单独列出供检索。',
            'econ': '——（混杂类，无统一理论逻辑）',
            'interaction': '文献多未明确提及',
            'members': uncovered,
            'flagship': uncovered[0],
            'duplicate_groups': {},
        })

    # ── 拼装 thematic_summary.md ──
    lines = [f"# {THEME}文献主题汇总\n"]
    lines.append(f"共 {len(all_filenames)} 篇入选文献,归纳为 {len(themes)} 个主题。\n")

    # 主题速览
    lines.append("## 主题速览\n")
    lines.append("| 主题 | 规模 | 一句话核心思路 |")
    lines.append("|------|------|----------------|")
    for th in themes:
        n_main = len(th['members'])
        n_dup = sum(len(v) for v in (th.get('duplicate_groups') or {}).values())
        label = f"{n_main} 篇代表作 + {n_dup} 篇同团队重复版本" if n_dup else f"{n_main} 篇"
        one_line = (th.get('core_idea') or '').replace('\n', ' ').split('。')[0] + '。'
        lines.append(f"| {th['name']} | {label} | {one_line} |")
    lines.append("")

    # 数据来源汇总(取自 individual.md 头部"- 数据来源："字段,代码拼装零幻觉)
    lines.append("## 数据来源汇总\n")
    lines.append("| 文献 | 数据来源 |")
    lines.append("|------|----------|")
    for fn in sorted(all_filenames):
        text = items_by_filename.get(fn, '')
        title_m = re.match(r'#\s+(.+)', text)
        title = title_m.group(1).strip() if title_m else fn
        lines.append(f"| {title} | {get_data_source(text).replace('|', '、')} |")
    lines.append("")

    # 各主题详情
    for ti, th in enumerate(themes, 1):
        lines.append(f"\n## 主题{ti}: {th['name']}\n")
        lines.append(f"**核心思路**: {th.get('core_idea', '')}\n")
        lines.append(f"**理论解释与机制分析**: {th.get('econ', '')}\n")
        lines.append(f"**异质性与边界条件**: {th.get('interaction', '')}\n")

        lines.append("**涉及文献**:\n")
        for fn in th['members']:
            text = items_by_filename.get(fn, '')
            title_m = re.match(r'#\s+(.+)', text)
            title = title_m.group(1).strip() if title_m else fn
            innovation = extract_section(text, '关键创新点')
            lines.append(f"- 《{title}》: {innovation}")
            followers = (th.get('duplicate_groups') or {}).get(fn, [])
            if followers:
                years = sorted(m.group(1) for f in followers
                               for m in [re.match(r'^(\d{4})-', f)] if m)
                year_str = '、'.join(years) if years else f'{len(followers)} 篇'
                lines.append(f"  - *另有 {len(followers)} 篇同团队重复发表版本: {year_str}(内容本质相同)*")
        lines.append("")

        # 代表性公式: 从 flagship 的 模型与方法构建 段硬抄
        fs = th.get('flagship')
        if fs and fs in items_by_filename:
            formula = extract_section(items_by_filename[fs], '模型与方法构建')
            if formula:
                fs_title_m = re.match(r'#\s+(.+)', items_by_filename[fs])
                fs_title = fs_title_m.group(1).strip() if fs_title_m else fs
                lines.append(f"**代表性模型与方法**(出自《{fs_title}》):\n")
                lines.append(formula + "\n")

    # 无关文献清单(仅 ENABLE_B3=True 且有剔除记录时出现)
    dropped_dir = DROPPED_DIR
    dropped_csv = OUTPUT_DIR / "strict_filter_result.csv"
    if dropped_csv.exists():
        lines.append("\n## 无关文献清单(B3 严格再筛剔除)\n")
        lines.append("| 文件名 | 剔除原因 |")
        lines.append("|--------|----------|")
        for row in dropped_csv.read_text(encoding='utf-8').splitlines()[1:]:
            parts = row.split(',', 3)
            if len(parts) >= 4:
                lines.append(f"| {parts[1]} | {parts[3]} |")
        lines.append("")

    out_path = OUTPUT_DIR / "thematic_summary.md"
    out_path.write_text('\n'.join(lines), encoding='utf-8')
    print(f"\n[B4] ✓ 汇总完成: {out_path} ({len(themes)} 主题)", flush=True)


# ── 主流程 ─────────────────────────────────────────────────────

def main():
    print(f"[START] B Phase — Theme: {THEME}, Mode: {MODE}", flush=True)
    print(f"[START] Papers dir: {PAPERS_DIR}", flush=True)
    print(f"[START] Output dir: {OUTPUT_DIR}", flush=True)

    if MODE == "review_only":
        relevant_list = b1_filter()
        if not relevant_list:
            print("[B1] No relevant papers found — aborting", flush=True)
            return
    else:
        # 完整模式: A4 已筛,直接全部进入 B2
        relevant_list = [{'filename': p.name} for p in sorted(PAPERS_DIR.glob("*.pdf"))]
        print(f"[B1] 完整模式跳过(A4 已筛): {len(relevant_list)} 篇进入 B2", flush=True)

    b2_summarize(relevant_list)

    if ENABLE_B3:
        kept_list = b3_strict_filter()
        if not kept_list:
            print("[B3] All papers dropped — aborting", flush=True)
            return
    else:
        # v1.1 默认: 无排除机制, individual 全部进入 B4
        kept_list = [{'filename': p.name} for p in sorted(INDIVIDUAL_DIR.glob("*.md"))]
        print(f"[B3] 已停用(ENABLE_B3=False): {len(kept_list)} 篇全部进入 B4", flush=True)

    b4_thematic_summary(kept_list)

    print("\n" + "=" * 60, flush=True)
    print("[DONE] B Phase complete", flush=True)
    print(f"[DONE] Output: {OUTPUT_DIR}", flush=True)
    print("=" * 60, flush=True)


if __name__ == "__main__":
    main()
```
