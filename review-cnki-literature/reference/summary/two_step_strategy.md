# B4 实现：两步策略 + 实战陷阱

## 为什么要拆成两步

B4 如果让 LLM 一次性从 individual 摘要生成完整的 thematic_summary.md，常见问题：
- LLM 容易丢失理论解释
- LLM 容易用过于学术的语言命名主题
- LLM 容易用 ``` 包裹输出
- 当文献数变多（>15 篇），LLM 在"涉及文献"段容易偷懒，把每篇压缩成一行甚至省略

## 推荐：两步策略（强烈建议在 `04_thematic_summary.py` 里这么写）

把"分组"和"摘录"拆开，让 LLM 只做它擅长的事，确定性的工作交给代码：

1. **第一步（LLM 做归类）**：把所有 individual 摘要喂给 LLM，让它只产出 JSON 格式的主题分组结果——每个主题包含：主题名、核心思路、理论解释、异质性与边界条件、归属的文献文件名列表，**外加一个 `flagship` 字段**：从 members 中选一篇最具代表性的文献（开山之作 / 方法定义最完整 / 该主题最经典实现）。**不要让 LLM 写"涉及文献"列表，也不要让 LLM 写公式**。
2. **第二步（代码做拼装）**：对每个主题：
   - 从所有 members 的 individual.md 用正则抽取`## 关键创新点`段原文 → 填充"涉及文献"小节（一行一篇）
   - 从 flagship 的 individual.md 用正则抽取`## 模型与方法构建`段原文 → 填充"代表性模型与方法公式"小节（每主题只 1 篇）
   - **核心思路全文不进汇总文档**，让读者去翻 individual

这样关键创新点和方法公式都是**从 individual 硬抄过来的**，零幻觉、零丢失。LLM 只负责归类、共性总结（核心思路 / 理论解释 / 异质性）、和挑 flagship。不需要 per-paper LLM 调用，速度快、token 省。

### 正则抽取

`extract_section()` 定义见 [shared/llm_client.md](../shared/llm_client.md)。B4 拼装时的用法：

```python
from pathlib import Path

def load_individual(md_path: Path) -> dict:
    text = md_path.read_text(encoding='utf-8')
    return {
        'filename':   md_path.name,
        'innovation': extract_section(text, '关键创新点'),
        'formula':    extract_section(text, '模型与方法构建'),  # 仅 flagship 用
    }
```

## 实战陷阱 1：LLM 会把同一篇文献归到多个主题

即使 prompt 里明确说"每篇文献必须且只能归入一个主题"，LLM 仍可能因为某篇文献的标题/副话题贴近另一个主题而把它二次归入。例如一篇"量价背离与股票收益"里用了双重差分做机制检验，就被同时塞进"量价背离因子"和"识别策略方法"两个主题。

**修复**：第一步拿到 LLM 的归类结果后，**代码强制做去重 + flagship 校验 + 兜底**。这三件事都必须由代码做，缺一不可：

```python
all_filenames = set(items_by_filename.keys())  # individual/ 实际存在的文件名

# ---- (a) 按首次出现去重 ----
# 保留首次出现 — LLM 通常先按主线方法论归类，
# 后面才把同篇文献因为副话题（如机制检验方法）二次归入特化主题。
# 首次出现 = 该篇文献的核心贡献所在主题，更准确。
# 不要用"保留最后出现"——会把主线文献错放进只匹配副话题的特化主题。
seen = set()
covered = set()
for th in themes:
    new_members = []
    for fn in th.get('members', []):
        # 同时过滤掉 LLM 幻觉出的、实际不存在的文件名
        if fn not in seen and fn in all_filenames:
            new_members.append(fn)
            seen.add(fn)
    th['members'] = new_members
    covered.update(new_members)

    # ---- (b) flagship 校验：LLM 经常幻觉文件名（拼错年份/期刊/标题） ----
    if th.get('flagship') not in all_filenames:
        # 回退到 members 第一篇
        th['flagship'] = th['members'][0] if th['members'] else None

themes = [th for th in themes if th['members']]  # 移除被清空的主题

# ---- (c) 兜底「其他」主题：未被任何主题覆盖的文献 ----
# LLM 偶尔会漏掉几篇——必须代码兜底，不要让它们消失
uncovered = sorted(all_filenames - covered)
if uncovered:
    themes.append({
        'name': '其他相关研究',
        'core_idea': '此类文献与主题相关，但研究思路不属于上述主题归并的方法论范畴，单独列出供检索。',
        'econ': '——（混杂类，无统一理论逻辑）',
        'interaction': '文献多未明确提及',
        'members': uncovered,
        'flagship': uncovered[0],
    })
```

**这三件事都是 LLM 输出的现实必然性，不是可选保护**（以下比例来自原券商研报版流程实测，机制在学术场景相同）：
- 去重：实测一个 100+ 篇的项目，LLM 把同一篇文献塞进 2-3 个主题的概率约 30%
- flagship 幻觉：LLM 写文件名时拼错日期/标题的概率约 5%
- 漏覆盖：LLM 漏掉几篇文献的概率约 50%

没有这三个保护，文档要么少几个主题、要么少几篇文献、要么有空主题。

## 实战陷阱 3：同团队重复发表需通过 auto_fold_series 折叠（实测 LLM 不会主动填 duplicate_groups）

**重要更新**：thematic_prompt.md 里写了让 LLM 用 `duplicate_groups` 字段折叠同团队重复发表，但**实测 LLM 几乎不会主动填这个字段**。在原券商研报版一个 49 篇核心相关文献的真实项目里，LLM 把 12 篇同系列报告全列成 12 个 members、`duplicate_groups` 是空的。学术场景的对应物是同一团队的重复发表（学位论文 vs 期刊版、中英文双发），机制相同。

prompt 加再多次"必须填 duplicate_groups"也没用 —— LLM 在 B4 第一次归类时认知负载已满（要同时分组+命名+选 flagship+写 4 段共性），没有余力做重复发表去重。

**正确做法**：在 B4 的代码里加 `auto_fold_series` 函数，按第一作者 + 标题归一化前缀自动识别同团队重复发表并折叠，**不依赖 LLM 是否填 duplicate_groups**：

```python
import re
from difflib import SequenceMatcher

def parse_filename(fn):
    """{year}-{journal}-{title}.md -> (year, journal, title)"""
    stem = Path(fn).stem
    year, journal, title = stem.split('-', 2)
    return year, journal, title

def get_first_author(md_text: str) -> str:
    """从 individual.md 头部的 '- 作者：张三, 李四' 行提取第一作者"""
    m = re.search(r'^-\s*作者：(.+)$', md_text, re.M)
    if not m:
        return ''
    return re.split(r'[,，;；、]', m.group(1))[0].strip()

def auto_fold_series(themes, author_by_filename):
    """按第一作者 + 标题归一化前缀自动识别同团队重复发表，折叠到 duplicate_groups

    学术场景适配说明：原券商研报版用 SERIES_KEYWORDS（绩效月报/周报/跟踪类词汇）
    先做候选过滤，学术文献不存在"同一方法每月发一篇"的模式，
    SERIES_KEYWORDS 置空、不再使用关键词门槛。
    折叠判据改为：同一第一作者 + 标题归一化前缀 80% 相同
    （对应学位论文 vs 期刊版、中英文双发、连载拆分发表等同团队重复发表）。
    """
    SERIES_KEYWORDS = []  # 学术场景无月报/周报跟踪模式，置空仅作标注，不参与逻辑

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
            first_author = author_by_filename.get(fn, '')
            by_first_author.setdefault(first_author, []).append((year, journal, title, fn))
        new_members = []
        new_dups = dict(th.get('duplicate_groups') or {})
        for author, items in by_first_author.items():
            if not author or len(items) == 1:
                new_members.extend(it[3] for it in items)
                continue
            # 同一第一作者内部按标题归一化前缀 80% 相似度聚类
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
                # leader 选择：期刊正式发表版优先（学位论文让位期刊版），同类按年份升序取最早
                cl.sort(key=lambda x: (is_thesis(x[1], x[2]), x[0]))
                leader_fn = cl[0][3]
                followers = [g[3] for g in cl[1:]]
                new_members.append(leader_fn)
                if followers:
                    new_dups.setdefault(leader_fn, []).extend(followers)
        # 保持原顺序
        ordered = [fn for fn in members if fn in new_members]
        for fn in new_members:
            if fn not in ordered:
                ordered.append(fn)
        th['members'] = ordered
        th['duplicate_groups'] = new_dups
        if th.get('flagship') not in th['members']:
            th['flagship'] = th['members'][0] if th['members'] else None
    return themes
```

**注意**：折叠后某些主题里仍然有多个代表作，是因为同团队存在真正的方法演进（如更换识别策略、更换代理变量后标题前缀本身不同）—— 归一化后不会合并到一起，这是正确行为。学术场景的重复发表对（学位论文 vs 期刊版）数量通常远少于券商月报系列，折叠主要发生在 CNKI 同时收录了学位论文库和期刊库时。

无论 auto_fold_series 是否生效，代码仍要做三件事：
1. **校验 duplicate_groups 中的文件名**（防止 LLM 在少数情况下提供的幻觉值）
2. **把跟随者也算作"已覆盖"**，否则它们会被错误地塞进「其他」兜底主题
3. **拼装时为每个有跟随者的代表作输出小注**，告诉读者「另有 N 篇同团队重复发表版本：年份列表」

```python
# 校验阶段（紧跟 dedup 之后）
for th in themes:
    raw_dups = th.get('duplicate_groups', {}) or {}
    clean_dups = {}
    for leader, followers in raw_dups.items():
        if leader not in th['members']:
            # leader 必须在 members 里；若 LLM 把 leader 漏到 members 外了，把它提升为 member
            if leader in all_filenames:
                th['members'].append(leader)
                covered.add(leader)
            else:
                continue  # leader 是幻觉文件名，丢弃整组
        valid_followers = [f for f in followers if f in all_filenames and f != leader]
        if valid_followers:
            clean_dups[leader] = valid_followers
            covered.update(valid_followers)  # 关键：让兜底逻辑不再把它们当未覆盖
    th['duplicate_groups'] = clean_dups

# 拼装阶段（文件名格式 {year}-{journal}-{title}.md，年份在前 4 位）
for fn in th['members']:
    # ... 输出代表作的关键创新点 ...
    followers = th.get('duplicate_groups', {}).get(fn, [])
    if followers:
        years = sorted(re.match(r'^(\d{4})-', f).group(1)
                       for f in followers if re.match(r'^\d{4}-', f))
        year_str = '、'.join(years) if years else f'{len(followers)} 篇'
        lines.append(f'  - *另有 {len(followers)} 篇同团队重复发表版本：{year_str}（学位论文/期刊版/中英文双发等，内容本质相同）*')
```

主题速览也要反映：

```python
n_main = len(th['members'])
n_dup = sum(len(v) for v in (th.get('duplicate_groups') or {}).values())
if n_dup > 0:
    label = f'{n_main} 篇代表作 + {n_dup} 篇同团队重复版本'
else:
    label = f'{n_main} 篇'
```

## 实战陷阱 2：调试时不想反复烧 LLM token

B4 的归类调用是大头（要把所有 individual 摘要塞进去）。调一次拼装模板或改一行 bug 就重跑很浪费。

**修复**：把 LLM 输出落盘缓存，重跑时优先读缓存。

```python
groups_cache = OUTPUT_DIR / 'theme_groups.json'
if groups_cache.exists():
    group_data = json.loads(groups_cache.read_text(encoding='utf-8'))
else:
    group_data = call_llm_for_grouping(...)
    groups_cache.write_text(json.dumps(group_data, ensure_ascii=False, indent=2),
                            encoding='utf-8')
```

**调试流程**：
- 调主题分组 prompt → 删 `theme_groups.json` → 重跑
- 只调拼装模板 → 不删任何缓存，重跑只会拼文档，零 LLM 调用
