# review-cnki-literature — 交付说明

知网（CNKI）学术文献一站式检索与综述 skill，v1.1（2026-11-18 全流程实测通过）。

## 安装

把 `review-cnki-literature/` 整个文件夹复制到以下任一位置：

- 用户级：`C:\Users\<用户名>\.claude\skills\review-cnki-literature\`（所有会话可用）
- 项目级：`<项目>\.claude\skills\review-cnki-literature\`（仅该项目可用）

复制后在 Claude Code 里说"知网检索"、"CNKI 文献综述"、"总结这批论文"等触发词即可调用。

## 运行依赖

| 依赖 | 说明 |
|------|------|
| Python 环境 | 需装有 `openai`、`PyMuPDF(fitz)`、`playwright`(1.54)。本包按团队 `rshfp_2.0` 环境适配，文本提取一律用 fitz（无 pdfplumber） |
| 浏览器 | 系统安装的 Edge（`channel="msedge"`），首次运行 `python -m playwright install` 无需执行（用系统 Edge） |
| 知网权限 | 机构 IP 或机构账号。检索/勾选匿名可用，**导出题录与下载 PDF 必须登录**；首次运行脚本会暂停等用户在浏览器中手动登录（支持校外访问 CARSI），登录态存浏览器 profile 跨会话复用 |

> 说明：本包不含 `.playwright/` 浏览器 profile 目录——那只是 cookie 与页面缓存（且含原使用者的知网登录态，不可外发），与 Playwright 库无关。首次运行会自动创建新 profile，届时按要求登录一次即可。
| LLM | DashScope（阿里云）API key，运行时询问，不落盘 |

## 使用提示（来自 2026-11-18 实测）

- 若机器开着 Clash 等系统代理：LLM 脚本加 `NO_PROXY=dashscope.aliyuncs.com` 前缀；浏览器已在模板中加 `--no-proxy-server`（详见 `reference/shared/llm_client.md"系统代理陷阱"`）。
- 三种运行模式：完整模式（默认）、仅检索、仅综述（自带 PDF 文件夹）。
- 全流程有四处人工门控：API key 收集、关键词确认、登录/验证码、入选确认。

## 目录

```
review-cnki-literature/
├── SKILL.md                  # 入口：流程、规则、门控、keywords.txt 语法
└── reference/
    ├── crawler/              # A 阶段 SOP、知网定位器表（含实测记录）、探索指南
    ├── prompts/              # 各阶段 LLM prompt 模板
    ├── summary/              # 判定标准扩写指南、两步策略、黄金示例
    ├── shared/llm_client.md  # LLM 调用规范、JSON 容错解析、系统代理陷阱
    ├── code_templates/       # a1-a5 + b 阶段脚本模板（复制改配置即用）
    └── quality_checklist.md  # 验收清单
```

验证报告（2026-11-18 真实全流程）留在原仓库 `skill/review-cnki-literature/report/`，不随本交付分发。
