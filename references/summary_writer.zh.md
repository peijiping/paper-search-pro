# 摘要撰写指南

*本指南由主 Agent 在 `SKILL.md` 的 STEP 11 读取，用于为本次 paper-search-pro 运行撰写 `summary.md` 中的执行摘要。执行摘要由主 Agent 直接撰写（不外包给 SubAgent），因为你对这个领域的阅读判断就是交付物本身。*

## 长度与结构

- **目标**：约 300 字
- **硬上限**：500 字
- **格式**：纯 Markdown；1-3 个短段落或约 6 条要点；摘要内部不要有小标题（这个文件是一个章节，不是一份完整文档）

## 必需结构

```
[1 opening sentence]: "This search returned N papers on <query>, with M classified as highly relevant (RCS >= 7)."

[2-3 sentences on field consensus]: What the field broadly agrees on -- what the dominant theoretical / empirical position is.

[1-2 sentences on key methods or frameworks]: How researchers in this area typically study the question (RCT? cohort study? specific model? PICO-defined approach? mechanistic experiment?).

[2-3 sentences on the most influential 3-5 papers]: Name and 1-line characterization of each. Use the influential_citation_count field where available (from SS enrichment) to rank, with citation_count as fallback. Spell out why each paper is influential -- not just that it is.

[1-2 sentences on open questions / unresolved tensions]: Where the field disagrees, what's missing, what the user should be aware of.

[OPTIONAL: 1 sentence on caveats]: If the search has known limitations (PubMed not queried, OpenAlex 3-day arXiv lag, etc.), mention briefly.
```

## 必须包含的内容

| 要素 | 为什么 |
|---------|-----|
| **主要共识 / 趋同结论** | 领域内的主导答案——帮助用户定位 |
| **关键方法 / 理论框架** | 告诉用户该如何解读这些文献 |
| **top 3-5 篇高被引 / 高影响力论文** | 深入阅读的具体起点 |
| **至少一个开放问题或分歧** | 避免把领域说成铁板一块；同时标出新研究可能切入的地方 |

## 必须避免的内容

- **AI 味的措辞**：避免 "interesting"（有意思）、"important"（重要）、"comprehensive"（全面）、"significant"（重大）、"various"（各种）、"numerous"（众多）、"a multitude of"（大量的）
- **滥用破折号 / "不只是 X，而是 Y" 的排比**：读完草稿，把这些删掉
- **空洞的模糊表述**："may"（可能）、"could"（也许）、"potentially"（或许）——改用有证据支撑的具体论断
- **偷懒的套话**："Papers cover topics ranging from A to Z"（论文涵盖了从 A 到 Z 的各种主题）——改为一一点名具体主题
- **空泛的赞扬**："These papers represent significant contributions"（这些论文代表了重要贡献）——说清楚*是什么*让它们重要
- **不敢下结论**："There are many perspectives"（存在多种视角）——选出主要的两种或三种并点名
- **含糊的引用**："as several studies show"（若干研究显示）——点名具体研究

## 为 top 3-5 挑选论文时的排序

优先提及哪些论文的优先级顺序：

1. **rcs >= 9** 且 `influential_citation_count`（SS 字段）同样高的论文。这些是奠基性著作。
2. **rcs >= 8** 且 `citation_count` 高的论文。奠基性稍弱，但覆盖更广。
3. **rcs >= 7** 的近期论文（year >= 当前年份 - 3），且 `cited_by_percentile_year` 高。这些是当下的"热门"工作。

搭配：优先级 1-2 里取 2-3 篇 + 优先级 3 里取 1-2 篇 = 经典与当下兼顾的良好平衡。

## 示例（前景理论检索式）

**检索式**："find 5-6 high-impact papers on prospect theory in decision making, classics + a couple recent ones"（找 5-6 篇关于决策中前景理论的高影响力论文，经典 + 几篇近期）

**生成的示例输出**（约 298 字）：

> 本次检索返回了 24 篇关于决策中前景理论的论文，其中 11 篇被判定为高度相关（RCS >= 7）。该领域的核心共识由 Kahneman 和 Tversky（1979）确立：人的决策在两个方面系统性地违背期望效用理论——人们对损失比对等量收益更敏感（损失厌恶），并且以非线性方式加权概率（高估小概率）。这一论述取代了行为经济学与决策科学中占主导地位的理性行动者框架。
>
> 在方法上，奠基性工作使用带货币赌注的实验室选择任务；累积前景理论（Tversky & Kahneman, 1992）把该框架推广到连续结果域。近期的扩展使用神经影像（例如 De Martino 等，2006，Science）把损失厌恶定位到杏仁核—前额叶回路，并用田野研究（例如 Camerer 等，1997，关于纽约出租车司机）在实验室之外检验预测。
>
> 最有影响力的论文是：Kahneman & Tversky（1979, Econometrica）——原始框架，被引 46,000+ 次；Tversky & Kahneman（1992）——累积前景理论；Thaler（1980）——早期消费者选择应用；De Martino 等（2006）——神经相关证据；以及 Sokol-Hessner 等（2009）——损失厌恶的调节。
>
> 仍待解决的开放问题有：(1) 跨文化可推广性——现有证据大多来自 WEIRD 样本；(2) 损失厌恶是稳定特质还是情境依赖；(3) 前景理论参数与真实世界消费者行为之间的关系，近期的元分析报告了不一致的效果量。
>
> *注意：本次未检索 PubMed（未检测到医学信号）；覆盖范围集中在 OpenAlex。*

## 保存前的快速自检

- [ ] 少于 500 字？
- [ ] 包含 "highly relevant"（高度相关）或 RCS 计数？
- [ ] 按作者 + 年份点名了至少 3 篇具体论文？
- [ ] 点名了至少一种具体方法或框架？
- [ ] 指出了至少一个开放问题 / 分歧？
- [ ] 没有出现 "interesting" / "important" / "significant" / "comprehensive" / "various"？
- [ ] 破折号总数不超过 2 个？
- [ ] 没有 "不只是 X，而是 Y" 的句式？

任何一项检查没过，就修改。
