# A4：题录初筛 Prompt 模板

A4 是分层架构的关键设计：**先看题录（标题+摘要）的相关性，再下载 PDF**。输入不是 PDF，而是 A3 导出的结构化题录。

- 完整可运行代码：`../code_templates/a4_screen_template.md`（prompt 内嵌其中，本文档解释设计与判定标准写法）
- 判定标准的扩写方法：`../summary/extraction_guide.md`（主题 → 正面/反面标准三步法）
- JSON 容错解析：`../shared/llm_client.md` 的 `robust_parse_list`

## 输入设计

每批 25 条题录，每条包含：

```text
{编号}. {标题} | {期刊} | {年份} | 关键词: {关键词} | 摘要: {摘要截500字}
```

- 摘要截断 500 字：题录摘要普遍 200-800 字，截断对判定影响小，控制 token
- 无摘要条目给标题+关键词，prompt 要求 reason 注明"无摘要"
- 无关键词条目同理降级

## 判定标准写法

`POSITIVE_CRITERIA` / `NEGATIVE_CRITERIA` 由 agent 读 extraction_guide.md 后把用户的一句话主题扩写填入：

- **正面**：一句话定义"什么算相关"（核心概念组合成的判定句）
- **反面**：枚举"沾边但不相关"的常见类型（学术场景典型：综述评述、纯理论无实证、宏观层面、教学研究、会议通知/征稿/书评——注意卷首语类噪音 A3 检索阶段不返回，但学位论文/报纸类可能漏入）
- **防御句**：即使标题出现主题关键词，摘要看核心内容不围绕主题也判不相关

## 判定原则（写死在 prompt 中）

| 原则 | 理由 |
|------|------|
| 宽松判定，宁滥勿缺 | 漏掉的代价 ≫ 误判的代价（误判只多花一篇下载+B2 摘要）；v1.1 起无 B3 再筛,入选全部进入综述,超出目标规模由"精选"控制(见下) |
| 摘要声称即采信 | 题录信息有限，无法验证的说法不判负 |
| 边界给 borderline | borderline 并入入选（selected.jsonl），直接进入综述,无再筛裁量 |
| 无摘要不判负 | 仅用标题+关键词判定并标注 |

## 目标规模控制(v1.1 起,规则 N6)

目标入选规模默认 **~10 篇**(用户可改):

- **超出目标**:判定完成后,将 relevant+borderline 题录交 LLM 按与主题相关度排序精选 top N 写入 `selected.jsonl`;溢出的写入 `overflow.jsonl` 留档,不下载、不丢弃(`records_screened.jsonl` 保留 LLM 原始判定不动);精选解析失败时 fallback 为 relevant 优先取前 N
- **不足目标**:触发发散匹配补检索（精确→模糊、为近义词新增 OR 篇关摘行、追加独立检索组），每框一个词；发散方案先向用户说明，A股范围不放开
- 精选/发散都发生在 ★A4→A5 门控展示之前,用户确认的是最终结果

## 输出契约

```json
{"results": [{"idx": 1, "label": "relevant|borderline|irrelevant", "reason": "20字内依据"}]}
```

- 要 `{"results": [...]}` 对象包裹（`json_object` 模式下比裸数组稳定）
- reason 不含大括号（防 `robust_parse_list` 正则失效）
- 解析失败：重试 1 次；仍缺条 → 该条 `label=error` 落盘并在日志输出 LLM 原始响应（规则 10），**不得静默丢弃**

## 产出与门控

- `records_screened.jsonl`：全量判定（含 label/reason/批次），断点续跑按 id 跳过
- `selected.jsonl`：relevant + borderline，A5 的输入
- **★A4→A5 门控**：向用户展示入选统计（入选数/年份分布/期刊分布/预计下载耗时 ≈ 入选数 × 20s），AskUserQuestion 确认后才启动 A5——下载是最贵且触发反爬的环节，下载量必须用户背书
