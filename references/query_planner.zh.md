# 查询规划器

*本文件由主 Agent 在 `SKILL.md` 的 STEP 1 读取，在检索任何东西之前把用户请求转换成 1-3 条检索策略。按检索意图选择框架——用错框架会浪费一个策略位。*

## 框架选择

| 框架 | 何时使用 | 产出 |
|-----------|-------------|---------|
| **PICO** | 临床 / 定量比较（"X 相比 Z 是否改善结果 Y？"） | 4 个概念块：P、I、C、O |
| **SPIDER** | 质性或混合方法（"人们如何体验 X？"） | 5 个概念块：S、PI、D、E、R |
| **PEO** | 范围综述 / 观察性（"暴露与结果之间是什么关系？"） | 3 个概念块：P、E、O |
| **开放性** | 探索性 / 理论 / 非临床（"关于 X 有什么研究"） | 2-4 个概念块，带同义词 |

## PICO —— 人群 Population、干预 Intervention、对照 Comparator、结果 Outcome

适用于：药物试验、行为干预、手术对比、饮食研究、任何带假设因果效应的研究。

| 块 | 问题 |
|-------|----------|
| **人群 Population** | 研究的是谁？（例如 "≥18 岁的 2 型糖尿病成人"） |
| **干预 Intervention** | 给的是什么？（例如 "metformin 500-2000mg/day"） |
| **对照 Comparator** | 跟什么比？（安慰剂 / 磺脲类 / 标准治疗 / 无） |
| **结果 Outcome** | 怎么测量？（HbA1c 下降、体重减轻、不良事件） |

**示例 1 —— 糖尿病 RCT 检索**：
- P（人群）："adults" OR "adult"
- 疾病（Condition）："type 2 diabetes" OR "T2D" OR "T2DM" —— *疾病是与人群分开的一个概念；块之间用 AND，绝不 OR 进 P（见 `search_export/methodology.md` §2.1）*
- I："metformin"
- C：("placebo" OR "sulfonylurea" OR "standard care") —— 在 OpenAlex 上通常省略（限制太死）
- O："HbA1c" OR "glycemic control"
- 研究设计：在布尔库导出时挂一个**经过验证的** RCT 过滤器（Cochrane HSSS）—— 见 `search_export/filters_library.md`。*绝不要自己写 `("RCT" OR "randomized")` 过滤器——两词文本过滤器会漏掉标题 / 摘要里没有这些词的试验（methodology §4.2）。*
- 合并后：`("adults" OR "adult") AND ("type 2 diabetes" OR "T2D") AND metformin AND ("HbA1c" OR "glycemic control")` —— 导出到 PubMed/Ovid 时加上那个经过验证的 RCT 过滤器。

**示例 2 —— IBS 饮食干预**（来自 SKILL.md 示例 4 的 audit 档）：
- P（人群）："adults" OR "adult"
- 疾病（Condition）："IBS" OR "irritable bowel syndrome" —— *疾病自成一个块，与人群用 AND 连接*
- I："low-FODMAP" OR "fiber" OR "dietary"
- C：任意对照
- O："symptom severity" OR "quality of life" OR "abdominal pain"

**示例 3 —— 远程医疗 老年高血压**：
- P："elderly" OR "older adults" OR "geriatric"
- I："telehealth" OR "telemedicine" OR "remote monitoring"
- C："in-person" OR "usual care"
- O："blood pressure" OR "hypertension control"

## SPIDER —— 样本 Sample、关注现象 Phenomenon of Interest、设计 Design、评估 Evaluation、研究类型 Research type

适用于：质性研究、混合方法、用户体验研究、心理现象。

| 块 | 问题 |
|-------|----------|
| **样本 Sample** | 谁？（比 PICO 的 P 更宽松） |
| **关注现象 Phenomenon of Interest** | 研究的是什么？（体验、态度、知觉） |
| **设计 Design** | 怎么做的？（访谈、焦点小组、民族志、问卷） |
| **评估 Evaluation** | 有哪些结果？（主题、态度、满意度） |
| **研究类型 Research type** | 质性 / 定量 / 混合 |

**示例 —— 养老场景中的依恋 + HRI**（来自 SKILL.md 示例 3）：
- S："elderly" OR "older adults" OR "long-term care residents"
- PI（现象）："attachment" OR "bonding"
- PI（技术）："human-robot interaction" OR "companion robot" OR "social robot" —— *心理现象和技术是两个不同的概念 → 块之间用 AND，不要在同一个块里 OR（methodology §2.1）*
- D："interview" OR "ethnography" OR "case study" OR "RCT"
- E："loneliness" OR "wellbeing" OR "social bonding"
- R：质性或定量——不要过滤（混合方法）

## PEO —— 人群 Population、暴露 Exposure、结果 Outcome

适用于：流行病学、观察性研究、环境健康、生活方式因素。没有干预时比 PICO 更轻。

| 块 | 问题 |
|-------|----------|
| **人群 Population** | 被观察的队列 |
| **暴露 Exposure** | 被研究的因素（非人为分配） |
| **结果 Outcome** | 测量的终点 |

**示例 —— 空气污染 + 认知衰退**：
- P："older adults" OR "aging population" OR "60+"
- E："PM2.5" OR "air pollution" OR "particulate matter"
- O："cognitive decline" OR "dementia" OR "Alzheimer's" OR "MMSE"

## 开放性 —— 概念块 + 同义词扩展

当用户意图是探索性的（"关于前景理论有什么研究？""找找关于工作记忆训练的论文"），放弃结构化框架，直接提取概念块。

**步骤**：
1. 从检索式中识别 2-4 个核心概念
2. 每个概念列出 2-5 个同义词 / 缩写 / 相关词
3. 块内用 OR 组合，块之间用 AND

**示例 —— "elderly 中的工作记忆训练"**：
- 概念 1 —— 工作记忆（对象）：`("working memory" OR "WM" OR "executive function")`
- 概念 2 —— 训练（干预）：`(training OR intervention OR program OR exercise OR "cognitive training")` —— *"cognitive training" 是干预，不是认知对象；它属于训练块，不属于概念 1（methodology §2.1）*
- 概念 3 —— 老年人：`(elderly OR "older adults" OR aging OR geriatric OR "65+")`
- 合并后：`("working memory" OR "executive function") AND (training OR intervention OR "cognitive training") AND (elderly OR "older adults")`

## 多策略组合（什么时候用多条策略）

Standard 及以上档位跑 2-3 条策略。每条策略针对一个不同的角度：

| 策略 | 适用于 | OpenAlex helper CLI |
|----------|---------|---------------------|
| **按引用数** | 高引经典 + 奠基性工作 | `openalex_helper deep "<q>" --sort cited_by_count:desc` |
| **按时间新近** | 最近一年最相关的工作，以及再往前两年 | `double-sort` 的近期支路（或 `search --recent N`）—— 不是 `deep --sort publication_date:desc`，后者按日期返回最新的标题 / 摘要命中，相不相关都返回 |
| **按相关性** | 主题匹配最好 | `openalex_helper deep "<q>" --sort relevance_score:desc` |
| **按奠基年份截断** | 只看经典（2015 年之前） | `openalex_helper seminal "<topic>" --year-max 2015` |
| **按综述类型** | 该主题已有的综述 | `openalex_helper reviews "<topic>"` |
| **按主题 ID** | OpenAlex topic ID（已知时） | （无 CLI；只能编程调用 —— `Works().filter(topics__id=...)`） |
| **按期刊白名单** | 只看顶刊 | `openalex_helper journal-list "<q>" --preset Cochrane|UTD24|...` |

Audit 档还要再加期刊白名单（例如 `Cochrane`、`medical_top`）。

`double-sort` CLI 会自动完成策略 1/2/3，并在某篇论文出现在 ≥ 2 条策略中时提升它的档位——这是 Standard 及以上推荐的默认做法。

## 跨语言检索式处理（中英混合——感知 search_language）

**语言*空间*在你走到这里之前就已经定了。** 三轴模型的坐标轴 2 —— `search_language` ∈ `en | zh | both | auto` —— 更早在 **STEP 1** 解析完毕（检索语言空间这个子步骤，其唯一事实来源是 `source_routing.md` §"Language scope"；优先级：单次 flag > 检索式内标记 > config > `auto` 然后提问），*在你在这里措辞检索式之前*。（随后 STEP 2 在已固定的空间内路由补充来源——它也不决定空间。）到你规划检索式时，空间已经是 **en / zh / both** 三者之一。本节只讲空间已知之后*怎么措辞*——它不决定空间。

下面的中→英对照表是**英文侧的扩展词表**：只要英文空间参与就适用（空间为 `en`，或 `both` 的英文那一半）。空间为 `zh` 时，它绝不要求把中文翻译走。

| 中文词 | 英文对应 |
|--------------|-----------------|
| 工作记忆 | working memory |
| 工作记忆训练 | working memory training |
| 老年人 / 老年 | elderly / older adults / aging |
| 干预 | intervention |
| 综述 | review / literature review |
| 元分析 / 荟萃分析 | meta-analysis |
| 随机对照试验 | randomized controlled trial / RCT |
| 临床试验 | clinical trial |
| 认知训练 | cognitive training |
| 注意力 | attention |
| 抑郁 | depression |
| 焦虑 | anxiety |
| 心理治疗 | psychotherapy |
| 神经影像 | neuroimaging |

**按空间定措辞的规则：**

- **`en` 空间 —— 以及 `auto`→en 分支（每条英文检索式的默认分支）：** 在检索式中使用**英文**词（OpenAlex 对英文元数据的索引最可靠）。理解用户意图时保留中文，但检索前先翻译。用一句话告诉用户：*"我会用英文检索，因为 'working memory training' 在 OpenAlex 上的覆盖比 '工作记忆训练' 更好。"* **这个分支与 v2.2 行为逐字节一致（R-19）**——纯英文检索式永远不会离开它，中文检索式只有在空间解析为 `en` 时才会走到这里。
- **`zh` 空间：** **不要**翻译。保留**中文词**作为检索式——OpenAlex 有多语言基础库，NSSD/yiigle 直接消费中文。你仍然可以做措辞归一化（去掉停用词、拆成概念块），但检索词保持中文。这里不套用中→英对照表。
- **`both` 空间：** 构建**两套**检索式——一套英文（按 `en` 规则），一套中文（按 `zh` 规则）——各自在自己的空间里跑；STEP 5 联邦合并两者的并集。PRISMA-S 日志器已经支持记录多套策略，所以两套都会被记录。

**选定空间的标记不是检索词。** 那些决定了空间的短语（`CSSCI`、`中文文献`、`SSCI`、`知网`……）是过滤 / 路由条件——要在检索前从主题里剥掉，和 `rank_intent` 剥掉"中科院一区"完全一样（见 `source_routing.md` §"Language scope"）。直接检索字面串 "CSSCI" 会返回*关于* CSSCI 的论文，而不是*你的主题*上的论文。

## OpenAlex 如何读一条检索式

这比 STEP 1 里任何其他选择都更能决定召回，而且它是静默失败的。

- **每个裸词都是必填。** `short video college students attention concentration` 的意思是 short AND video AND college AND students AND attention AND concentration。所以以词形式写出的同义词会*互相排除*那些只用了其中之一的论文。
- **写成概念块：同义词放 OR，块之间放 AND，短语加引号。** `("short video" OR "short-form video" OR TikTok OR Douyin) AND (attention OR concentration OR procrastination)`。检索用两到三个块；导出到布尔库时可以更多（`search_export/methodology.md`）。
- **至少让一条策略不含人群 / 场景块。** "college students" 会丢掉所有写成 "young adults" 或 "undergraduates" 的论文；人群可以事后由分类器判断。
- **把泛词挡在 OR 组外**（interest, effect, use, role, development, exploration）。其中一个放进块里，就会让成千上万篇只是 loosely related 的论文命中，把你要的那些埋掉。
- 检索会匹配标题、摘要，以及 OpenAlex 有全文时的全文。被引支路和近期支路只匹配标题和摘要，所以那里格式良好的检索式尤其重要。

| 实测于 2026-10-01，用户自己的检索 | 窗口期内标题+摘要命中 | 检索到的已知在题论文 |
|---|---|---|
| `short-form video use college university students attention attentional control sustained attention concentration`（11 个裸词，2021–26） | 1 | 2 of 6 |
| `("short video" OR "short-form video" OR TikTok OR Douyin) AND (attention OR attentional OR concentration OR procrastination OR burnout)` | 5,014 | 4 of 6 |
| `curiosity interest epistemic emotions development children`（2025–26） | 1 | 0 of 3 |
| `(curiosity OR "epistemic emotion" OR "information seeking") AND (child OR children OR infant OR infants OR toddler)` | 2,220 | 2 of 3 |

OpenAlex 辅助脚本在出现六个或更多不带 OR 的裸词时会打印警告，STEP 3 的 `count` 会在命中少于 30 篇时给出提示——两者都在说：先重构，再分级。arXiv 同样要求每个裸词（辅助脚本发送 `all:` 词项）；OpenReview 不解析布尔语法，所以它的辅助脚本只把这些词单独发过去。

## 年份过滤启发式

什么时候问、怎么问：SKILL.md STEP 1 "Time scope"。窗口如下：

- 明确区间（"2010-2024"、"近两年"、"2023 年以后"）：`year_min` / `year_max`；相对区间从当前年份往回算
- 只说 "Recent" / "最新" 而没有数字：提问（或用 `config.recent_years`）；绝不默默假定五年
- "Last decade"：`year_min = current_year - 10`
- "Classics"：不设年份过滤；加上 `seminal`
- 什么都没说：任何档级都不设年份过滤。近期支路（double-sort，或 Quick 档的 `search --recent`）仍会带来最近一年以及前两年，所以没有窗口也有新工作
- Audit 档：遵从方案里明确写出的 `IC: 2010-present` 等

为什么光有窗口不够：相关性排序倚重引用，所以在五年窗口内，相关性支路仍然偏向最早的年份。真正能找到今年论文的是近期支路。

## 反模式

- 不要在 OpenAlex 层面按 `language=english` 过滤——限制太死，会丢掉那些有英文摘要的中日韩研究。
- 不要给医学检索式加 `AND "human"`——相关论文里有一半标题 / 摘要中没有 "human"；改为依赖 OpenAlex 的主题聚类。
- 不要组合超过 4 个 AND 块——召回会暴跌。裸词也算块：见"OpenAlex 如何读一条检索式"。如果你有 5 个以上概念，拆成 2 条策略再合并。**这个上限是 OpenAlex 相关性引擎的启发式，不是布尔库的规则**——布尔库导出常规会 AND 5–8 个概念块（见 `search_export/methodology.md` §2.2）；导出专业检索式时不要继承这个 ≤4 的上限，它也不约束上面那个 5 块的 SPIDER 示例。
