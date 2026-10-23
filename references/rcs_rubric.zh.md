# RCS 评分准则

*本文件由 `SKILL.md` STEP 6 中的分类器 Inline SubAgent 读取，`classifier_subagent_prompt.md` 引用了它。跨批次一致地应用本准则，让主 Agent 能够横向比较分数。*

RCS = **Relevance to Core Search**（与核心检索的相关性），取值 0-10 的整数。分数越高，表示与用户检索式越相关。

## 评分刻度

### 0-1：离题

论文没有触及检索式的主题。关键词可能只是表面重合，但实质内容无关。

- **0** = 毫无明显关联
- **1** = 提到了检索式中的某个关键词，但论文讲的是别的东西

*示例（检索式："prospect theory in decision making"）*
- 标题："Future prospects for renewable energy in developing economies" → **0**
- 标题："Theory of computation: a primer" → **1**（只是 "theory" 撞词）

### 2-3：边缘相关

论文碰到了这个主题，但只是作为背景、顺带提及，或者出现在很不一样的语境里。

- **2** = 主题只出现一次，不是核心
- **3** = 相关领域，但属于另一个子领域

*示例（检索式："working memory training in elderly"）*
- "Cognitive aging: a broad review"，其中把 WM 作为 12 个认知域之一提及 → **2**
- "Visual working memory training in college students"（同样的干预，人群错了）→ **3**

### 4-5：相关但非核心

论文实质性地讨论了检索式的部分内容，但没抓住核心问题或核心人群。

- **4** = 覆盖了 2-3 个检索维度中的 1 个
- **5** = 相关但太旧 / 设计薄弱 / 试点 / 非实证

*示例（检索式："GLP-1 receptor agonist for obesity"）*
- "Mechanism of action of GLP-1 in pancreatic β cells"（只讲机制，没有肥胖）→ **4**
- "Liraglutide and weight loss: a 2016 pilot study n=20"（相关，但只是试点）→ **5**

### 6-7：高度相关 —— 首选参考候选

论文直接回应检索式，人群、干预、方法学都到位。这类论文最可能被用户写进成果里引用。

- **6** = 相关、新、方法尚可、会被引用
- **7** = 直接命中检索式，引用充分（>100 次引用）或是近期的奠基性论文

*示例（检索式："GLP-1 receptor agonist for obesity"）*
- "Semaglutide 2.4mg once-weekly in adults with overweight or obesity: RCT, n=1961"（NEJM 2021）→ **7**
- "Tirzepatide for chronic weight management"（NEJM 2022 RCT）→ **7**

### 8-10：奠基性 / 开创性

论文是该领域的奠基性参考文献——定义构念、提出方法，或是该主题下被引最多的论文。

- **8** = 明确是奠基性工作，但不是那唯一一篇定义性论文
- **9** = 精确检索主题下被引最多 / 影响力最大的前三
- **10** = 那篇定义性论文——原始构念、诺奖级方法引入、人人都引的论文

*示例（检索式："prospect theory in decision making"）*
- Kahneman & Tversky 1979 "Prospect theory: an analysis of decision under risk" → **10**
- Tversky & Kahneman 1992 "Advances in prospect theory: cumulative representation of uncertainty" → **9**
- Thaler 1980 "Toward a positive theory of consumer choice" → **8**

*示例（检索式："transformer architecture"）*
- Vaswani et al. 2017 "Attention is all you need" → **10**
- Devlin et al. 2018 "BERT" → **9**

## 特殊 flag（适用时设置 `flag` 字段）

| flag 取值 | 何时设置 | 对 RCS 的影响 |
|------------|-------------|---------------|
| `no_abstract_uncertain` | 摘要为空 / null 且标题是唯一的判断信号 | RCS 降 1（上限 5，除非标题信号压倒性） |
| `off_topic_despite_keywords` | 标题 / 关键词与检索式匹配，但摘要显示论文讲的是别的东西 | 设 RCS = 1-2，并给出理由 |
| `parse_failed_uncertain` | 论文数据畸形（缺标题、摘要乱码） | 设 RCS = 0，并标记以供人工复核 |
| `abstract_unavailable` | 摘要字段存在，但值为 "N/A" / "[paywalled]" / 极短（< 20 字符） | 与 no_abstract_uncertain 相同 |
| `recent_unindexed` | 近期论文（最近一年左右——arXiv、OpenReview，或近期支路）尚无或少有引用（不要因新而扣分） | 只按标题 / 摘要评分；不要因 citation_count=0 扣分 |

## 评分纪律

### 避免 RCS 虚高
- 相关但非核心的论文默认给 4-5；6-7 只留给直接命中检索式的论文
- 8-10 只留给真正的奠基性工作——如果一批 10 篇里有 5 篇拿到 RCS≥8，那就是虚高
- 引用数 50,000+ 的论文**不自动等于 RCS=10**；与用户精确检索式的相关性才是闸门
- 名气大的作者不等于 RCS=10；必须是这篇具体的论文具有奠基性

### 避免 RCS 压低
- 不要因为论文新（还没有 citation_count）而扣分——按摘要 / 标题判断
- 不要因为论文是其他语言而扣分（OpenAlex 收录中文 / 日文 / 韩文 / 俄文摘要）——按内容判断
- 不要因为是预印本 / arXiv 论文而扣分——只要研究本身切题，发表载体不决定 RCS

### 不确定时
- 在相邻两个分数之间犹豫时，**向下取整**（5 还是 6 → 5）
- 在"要不要标记"之间犹豫时，**标记它**——主 Agent 需要看到不确定性信号
- 摘要缺失但标题强烈暗示相关时，设 RCS = 5 上限，并带上 `no_abstract_uncertain` flag

## JSON 输出结构（每篇论文）

```json
{
  "paper_id": "10.1126/science.aaa5760",
  "rcs": 7,
  "reasoning": "Directly addresses query on GLP-1 for obesity; phase 3 RCT in NEJM with n>1900 and clinical-trial number; well-suited as primary citation. Not foundational (semaglutide is post-liraglutide work).",
  "flag": null
}
```

- `paper_id`：必须与批次输入中的 `paper_id` 一致（用 `UnifiedPaperEntity` 的 `paper_id` 字段）
- `rcs`：0-10 的整数（不带小数）
- `reasoning`：1-2 句，最多约 50 词。具体说明为什么是这个分数，不要泛泛称赞
- `flag`：特殊 flag 表中的某个取值，或在不适用任何 flag 时为 `null`

## 示例输出

**输入批次条目（节选）**：
```json
{
  "paper_id": "10.1056/nejmoa2032183",
  "title": "Safety and Efficacy of the BNT162b2 mRNA Covid-19 Vaccine",
  "abstract": "BNT162b2 is a lipid nanoparticle-formulated...",
  "year": 2020,
  "citation_count": 14523,
  "venue": "New England Journal of Medicine"
}
```

**检索式："BNT162b2 COVID-19 vaccine efficacy"**：
```json
{
  "paper_id": "10.1056/nejmoa2032183",
  "rcs": 10,
  "reasoning": "The Pfizer-BioNTech BNT162b2 phase 3 trial paper — original efficacy report in NEJM, foundational to the query topic; cannot be ranked lower.",
  "flag": null
}
```

**检索式："mRNA vaccine technology for cancer"**：
```json
{
  "paper_id": "10.1056/nejmoa2032183",
  "rcs": 4,
  "reasoning": "BNT162b2 is mRNA but for COVID-19, not cancer. Tangentially relevant as a platform-validation reference but not addressing the cancer mRNA vaccine question.",
  "flag": null
}
```
