# 档级决策

*本文件由主 Agent 在 `SKILL.md` 的 STEP 0 读取，用于在任何检索开始之前从四个档级中选定一个。*

## 四个档级

| 档级 | 实际耗时（理想） | 实际耗时（现实） | 文献数 | OpenAlex 策略 | L2 增强源 | L3 丰富 | 引文跳数 |
|------|-----------:|-----------:|-------:|-------------------|-------------|-----------|--------------:|
| Quick | ~5 分钟 | **~7-9 分钟** | 20-60 | 相关性 top-30 + 近期 20（`search --recent 20`） | 通常跳过 | 跳过 | 0 |
| **Standard**（默认） | ~10 分钟 | **~14-18 分钟** | 60-180 | `double-sort` 每支路 top-50（被引 / 近期 / 相关性） | 按 source_routing | top-N rcs≥6 | 1 |
| Deep | ~30 分钟 | **~40-50 分钟** | 180-400 | `double-sort` top-100 + seminal + reviews | 按 source_routing | top-N rcs≥6 | 1-2 |
| Audit | ~2-3 小时 | **~3-4 小时** | 400-1000+ | 全量多策略 + 期刊白名单 | 允许独立 PubMed 检索 | 全量 top-N | 2 + 期刊白名单 |

> **时间预算的现实情况** —— "理想"那一列只是纯 API + 计算的下限。现实中会额外增加约 50-70% 的开销：诊断性往返（CLI argparse 漂移）、分类器 SubAgent 的调度延迟（并行派发并非免费）、大体积 bundle 的 HTML 渲染，以及主 Agent 在步骤之间的推理。**当你要告诉用户"这大概需要 ~X 分钟"时，请引用"现实"那一列** ——引用理想值会让用户实际等待时间超出 1.7×-1.9×（E3 Session-#1 / Session-#2 实测）。

## 触发信号（按档级）

**Quick** —— 检索式短、要 5-10 篇、有时间盒
- "查一下" / "几篇" / "扫一眼"
- "5-6 papers"（5-6 篇） / "a handful"（少数几篇） / "high-impact only"（只要高影响力）
- "before tomorrow"（明天之前） / "in 5 min"（5 分钟内） / "fast scope"（快速摸底）
- "classics"（经典）+ 数量少 / "just the seminal ones"（只要奠基性那几篇）
- "I'm not writing a review, just exploring"（我不是在写综述，只是随便看看）
- 明确的 "quick"（快） / "fast"（快速） / "short"（简短）

**Standard** —— 未指明深度的常规文献检索、划定某个主题的范围
- "find papers on X"（找关于 X 的论文），不带深度限定词
- "I'm writing background / 综述 / introduction"（我在写背景 / 综述 / 引言）
- "老板让我看一下" / "scope this topic"（划定这个主题的范围）
- "what's been published on Y"（关于 Y 已经发表了什么）
- "我对这块完全不懂"
- "lit review for proposal / class paper / thesis chapter"（为提案 / 课程作业 / 论文章节做的文献综述）

**Deep** —— 写综述文章、30 分钟以上预算、要真正的深度
- "I'm writing a proper literature review article"（我在写一篇正式的系统文献综述）
- "thorough"（彻底） / "real depth"（真正的深度） / "comprehensive"（全面）
- "submission"（投稿） / "for publication"（为了发表）
- "review for journal X"（为期刊 X 写的综述）
- "综述写作" / "深度文献综述"
- "I need to cite every important paper"（我需要引用每一篇重要论文）

**Audit** —— 系统综述 / PRISMA / 元分析的前期准备
- "systematic review"（系统综述） / "SR" / "Cochrane"
- "PRISMA" / "PRISMA-S" / "PRISMA flow diagram"（PRISMA 流程图）
- "meta-analysis"（元分析） / "网络元分析" / "umbrella review"（伞形综述）
- 已列出 "inclusion criteria"（纳入标准） / "exclusion criteria"（排除标准）
- 明确的 PICO，且人群 / 干预边界是硬性的
- "preparing a SR for [journal X]"（为期刊 X 做系统综述准备） / "Cochrane Library protocol"（Cochrane Library 方案）

## 决策树（按顺序套用，先匹配者胜）

```
1. Audit signals present?
   YES → Audit tier (show limitations warning, get user confirmation)
2. Deep signals present?
   YES → Deep tier
3. Quick signals present?
   YES → Quick tier
4. Otherwise:
   → Standard tier (default)
```

默认是 **Standard** ——绝不悄悄选 Quick 或 Deep。如果你选的不是 Standard，**在检索之前先用一句话向用户说明理由**。

## 实际示例映射（来自 SKILL.md §Examples）

| 示例 | 触发 | 选定档级 |
|---------|---------|-------------|
| "find 5-6 high-impact papers on prospect theory, classics + recent"（找 5-6 篇前景理论的高影响力论文，经典 + 近期） | "5-6"、"high-impact"（高影响力）、短、单一主题 | **Quick** |
| "用 paper-search-pro 帮我找一些工作记忆训练干预文献 老板让我看 我对这块完全不懂" | "找一些"、"老板让我看"、"完全不懂" → 默认 Standard | **Standard** |
| "I'm writing a proper literature review on attachment + HRI in elderly care, need real depth"（我在写一篇关于养老照护中依恋 + HRI 的正式文献综述，需要真正的深度） | "literature review article"（文献综述文章）、"real depth"（真正的深度）、"submission"（投稿） | **Deep** |
| "preparing a systematic review on dietary interventions for IBS in adults. IC: RCTs, ≥18, low-FODMAP/fiber, English, 2010-present"（为成人 IBS 的饮食干预准备系统综述。纳入标准：RCT、≥18 岁、低 FODMAP / 纤维类干预、英文、2010 年至今） | "systematic review"（系统综述）、明确的 PICO + 纳入标准清单 | **Audit** |

## Audit 档局限性警告（开始前原样粘贴）

```
本工具不是符合 PRISMA 规范的系统综述的替代品。

paper-search-pro 的 Audit 档是 SR 前期辅助——它能帮你：
- 覆盖 OpenAlex 深度检索 + PubMed MeSH + 期刊白名单
- 生成 16 项 PRISMA-S 日志以保持透明可追溯
- 产出可导入引文管理器的 BibTeX/RIS

但它不能替代：
- Cochrane Library / Embase / CINAHL（专业 SR 数据库）
- PROSPERO 方案注册
- 双人独立筛选（你仍然需要人工筛选者）
- 手工翻查纳入研究的参考文献列表（我们最多做 2 跳）
- 灰色文献 / 除 NCT 编号之外的临床试验注册库

如果你要投 Cochrane / 或要求符合 PRISMA 2020 的期刊，
请把我们的产出当作 PHASE 1 SCOPING（范围综述），而不是最终的系统综述。你仍然需要：
1. 在我们之外另做 Cochrane/Embase 检索
2. PROSPERO 注册
3. 双人独立评审筛选并解决分歧
4. 对每篇纳入研究使用偏倚风险评估工具（RoB 2 / ROBINS-I）

确认你已理解并希望继续使用 Audit 档吗？（yes/no）
```

等用户明确回答 "yes" 之后再开始检索。
