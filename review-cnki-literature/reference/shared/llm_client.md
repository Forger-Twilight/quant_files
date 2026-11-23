# LLM 调用规范（全阶段通用）

本文件是所有阶段 LLM 调用的唯一参考。脚本生成时从这里取客户端代码和约定。

> **⚠️ 绝对禁止使用流式输出**：所有 `client.chat.completions.create()` 调用**严禁**设置 `stream=True`，必须使用默认的同步非流式模式。违反此规则会导致输出缓冲混乱和进度追踪失效。

---

## 凭据

- `ALIYUN_API_KEY` — DashScope API 密钥，不存储在环境变量或文件中
- **Agent 收集方式**：执行脚本前，用 `AskUserQuestion` 向用户收集 API key，然后通过命令行参数 `sys.argv[1]` 传入脚本
- **手动终端执行**：脚本 fallback 到 `input()` 交互输入

## Python 环境

使用 `rshfp_2.0`（本机唯一可用环境；与全局 CLAUDE.md 默认一致）：

```bash
# Agent 执行（API key 通过参数传入）
conda activate rshfp_2.0 && python code/xxx.py <api_key>

# 用户手动执行（脚本会提示输入）
conda activate rshfp_2.0 && python code/xxx.py
```

> 本机环境说明：`rshfp_2.0` 已装 openai / PyMuPDF(fitz) / playwright(Python 1.54)；**未装 pdfplumber**，PDF 文本提取一律用 fitz（见 b 阶段模板）。该环境按团队规范只读，不为本 skill 安装新包。

## 系统代理陷阱（2026-11-18 实测，本机常开 Clash Verge）

- **LLM 脚本**：httpx（OpenAI SDK 底层）默认读 Windows 系统代理，API 流量会走代理出口（实测新加坡 47.129.x），有 IP 白名单的 key 会报 403 `IP access denied by API-Key restriction`。**所有 LLM 脚本启动命令统一加前缀**：`NO_PROXY=dashscope.aliyuncs.com no_proxy=dashscope.aliyuncs.com`（实测生效，无需改代码）。诊断出口 IP 要用 httpx 复测——Git Bash 的 curl 不读注册表代理，量到的是假直连。
- **Playwright 浏览器**：Chromium 默认跟随 Windows 系统代理（实测 `ifconfig.me` 显示代理出口）。`proxy={"server": "direct://"}` 写法**无效**（报 ERR_PROXY_CONNECTION_FAILED）；正确做法是在 `launch_persistent_context` 加 `args=["--no-proxy-server"]`（实测直连真实出口）。A3/A5 必须加——CNKI 走境外代理出口既丢机构 IP 识别，又是风控高危特征（验证码增多）。

## Client 创建

```python
from openai import OpenAI
import sys, time

api_key = sys.argv[1] if len(sys.argv) > 1 else input("请输入 ALIYUN_API_KEY: ").strip()

client = OpenAI(
    api_key=api_key,
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
)
```

### ⚠️ 重要：严禁在 API 调用中设置 timeout

- DashScope 联网搜索（`enable_search: True`）响应时间 30-120 秒属正常
- **严禁**在 `client.chat.completions.create()` 中设置 `timeout` 参数
- 原因：OpenAI SDK 默认 `max_retries=2`，设了 timeout 后超时会触发 SDK 内部静默重试，总等待时间 = timeout × (max_retries+1)，期间无任何输出，表现为脚本假死
- 超时控制由 Bash 命令的 timeout 参数作为最外层兜底

## 通用调用函数

```python
def call_llm(prompt: str, model: str = 'qwen-long', max_retries: int = 2) -> str:
    for attempt in range(max_retries + 1):
        try:
            print(f"  [{time.strftime('%H:%M:%S')}] 调用 DashScope ({model})...", flush=True)
            resp = client.chat.completions.create(
                model=model,
                messages=[{'role': 'user', 'content': prompt}],
            )
            print(f"  [{time.strftime('%H:%M:%S')}] API 返回成功", flush=True)
            return resp.choices[0].message.content
        except Exception as e:
            print(f'  LLM调用失败(attempt {attempt+1}): {e}', flush=True)
            if attempt < max_retries:
                time.sleep(2)
    return None
```

## 模型选择 + ETA + Timeout

| 阶段 | 模型 | 用途 | ETA | Bash timeout |
|------|------|------|-----|-------------|
| A1 | qwen3.5-plus | 联网搜索（`enable_search: True`） | 20 分钟 | 1500000 (25min) |
| A2 | qwen3.5-plus | 普通对话 | 5 分钟 | 360000 (6min) |
| A3 | 无 LLM | Playwright 检索+题录导出 | 视关键词组数（每组约 2-5 分钟） | 14400000 (4h) |
| A4 | qwen-long | 题录批量筛选（`response_format: json_object`） | 视题录量（每批 25 条约 1 分钟） | 14400000 (4h) |
| A5 | 无 LLM | Playwright 逐篇下载 | 视入选数（每篇约 15-30 秒） | 14400000 (4h) |
| B1-B4 | qwen-long | 筛选/摘要/归类 | 不设 ETA | 不设 timeout |

> **注意**：API 调用严禁设 `timeout` 参数，超时控制统一由上表中的 Bash timeout 兜底。

## ETA 同步与用户通知（强制）

- **每个阶段执行前**必须向用户播报 ETA，禁止未播报直接启动
- A3/A5 长爬取期间**每 30 分钟**主动推送进度摘要
- 阶段结束报告实际耗时与 ETA 偏差
- 超时/卡住时**必须主动告知用户**，不能沉默等待
- API 调用超时触发重试时向用户说明

## A 阶段运行约定

A 阶段（A1/A2/A4）脚本**前台直接运行**：

```bash
conda activate rshfp_2.0 && python code/xxx.py
```

- **禁止** `conda run -n rshfp_2.0 python ...`（输出缓冲、交互性差）

**A3/A5 例外**：A3/A5 脚本必须**后台启动**（`run_in_background`），因为脚本启动后需要等待用户登录（或等待用户人工过验证码），Agent 必须在脚本等待期间通过 `AskUserQuestion` 与用户交互确认，然后删除标记文件 `output/.wait_login`（或 `.wait_captcha`）让脚本继续。

- B 阶段不受此限制

## 防超时输出约定

所有脚本必须遵守：

```python
import sys
sys.stdout.reconfigure(encoding='utf-8')
sys.stderr.reconfigure(encoding='utf-8')
```

- 所有 `print` 加 `flush=True`
- 每次 API 调用前后输出带时间戳的进度提示
- 长循环中每轮开头有输出，间隔不超过 10 秒
- 严禁在 API 调用中设置 `timeout` 参数（由外层 Bash timeout 兜底）

---

## JSON 响应解析（A4/B 阶段共用）

### robust_parse — 容错 JSON 解析（对象版）

LLM JSON 响应常不标准（字段值含 `{}`、公式字符等）。贪婪匹配 + label fallback 双层防御：

```python
import re, json

def robust_parse(text: str) -> dict:
    """贪婪匹配 + label fallback 双层防御"""
    if not text:
        return {'label': 'error', 'reason': '空响应'}
    m = re.search(r'\{.*\}', text, re.DOTALL)  # 贪婪（不能用 \{.*?\}）
    if m:
        try:
            return json.loads(m.group())
        except json.JSONDecodeError:
            pass
    label_m = re.search(r'"label"\s*:\s*"(relevant|irrelevant)"', text)
    if label_m:
        reason_m = re.search(r'"reason"\s*:\s*"([^"]*)"', text)
        return {'label': label_m.group(1),
                'reason': reason_m.group(1) if reason_m else '(reason 解析失败)'}
    return {'label': 'error', 'reason': f'无法解析: {text[:200]}'}
```

### robust_parse_list — 容错 JSON 解析（数组版，A4 批量判定用）

A4 一次喂一批题录，要求返回 JSON 对象 `{"results": [...]}`（json_object 模式下比裸数组更稳）。解析兼容包裹与裸数组两种形态，fallback 为逐行抽取：

```python
def robust_parse_list(text: str) -> list:
    """A4 批量判定响应解析: 先试 {"results": [...]}, 再裸数组, 再逐条抽取"""
    if not text:
        return []
    m = re.search(r'\{.*\}', text, re.DOTALL)
    if m:
        try:
            data = json.loads(m.group())
            if isinstance(data, dict) and isinstance(data.get("results"), list):
                return data["results"]
        except json.JSONDecodeError:
            pass
    m = re.search(r'\[.*\]', text, re.DOTALL)
    if m:
        try:
            data = json.loads(m.group())
            if isinstance(data, list):
                return data
        except json.JSONDecodeError:
            pass
    # fallback: 逐条抽 {"idx": n, "label": "...", "reason": "..."}
    items = []
    for m2 in re.finditer(
            r'\{\s*"idx"\s*:\s*(\d+)\s*,\s*"label"\s*:\s*"(relevant|borderline|irrelevant)"\s*,\s*"reason"\s*:\s*"([^"]*)"\s*\}',
            text):
        items.append({'idx': int(m2.group(1)), 'label': m2.group(2), 'reason': m2.group(3)})
    return items
```

### extract_section — 从 individual 摘要抽段落

```python
import re

def extract_section(md_text: str, section_name: str) -> str:
    """从 individual 摘要 Markdown 中抽取 ## 段的正文（到下一个 ## 之前）"""
    pattern = rf'## {re.escape(section_name)}\s*\n(.*?)(?=\n## |\Z)'
    m = re.search(pattern, md_text, re.DOTALL)
    return m.group(1).strip() if m else ''
```
