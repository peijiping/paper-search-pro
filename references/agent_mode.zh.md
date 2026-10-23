# Agent / headless 模式 —— 面向 Agent 调用方的操作手册（SSOT）

**当 paper-search-pro 由另一个 Agent 驱动，或处于任何非交互 / headless 场景时，请读本文件** —— 也就是结果的消费方是一个程序（你自己的推理循环），而不是一个想要 HTML 报告的人。

本文件是 `agent_search` 结构化数据通道的唯一事实来源。`SKILL.md` 中面向人类的 14 步流程走的是*另一条*路径，不受这里任何内容影响（R-19）。

## 目录

1. **北极星** —— 下限 vs 人类质量的阶梯（你是一个 LLM，不是一根 JSON 管道）。
2. **何时使用 agent 模式** vs 14 步人类流程。
3. **这条命令**（`agent_search`）+ 它内建的确定性纪律。
4. **如何达到人类模式的质量** —— 自己打 RCS 分 · 迭代到饱和 · 扩展引文 · 选引擎（AI→SS）· 语言空间 + `meta.language`。
5. **JSON 信封结构** + 错误码 / 退出码。
6. **Flags** —— 完整参考（`--lang`、`--primary-source`、`--with-nssd/-yiigle`、`--zh-block`、`--with-strategies`、`--verify`、档位过滤）。
7. **相关性评分** —— 下限启发式，以及它*不是*什么（仅拉丁字母分词；SS 注意事项）。
8. **`--verify` / `--verify-refs`** —— 存在性与反幻觉检查。
9. **`--with-strategies`** —— 检索式导出：信封里的机械下限，质量由你补上。
10. **数据源选择**（引擎轴）· **多平台期刊分区**（中科院/JCR/SJR）。

---

## 北极星 —— 你是 LLM 调用方，默认按人类模式的质量来

你不是一个把 JSON 往外接的 shell 脚本。**你是一个 LLM。** agent 模式下的检索，默认标准与人类那条 14 步流程达到的质量**相同** —— 因为那条流程能做的事你都能做：从语义上判断相关性、识别学科领域、决定语言空间、迭代到饱和、通过引文扩展。你唯一缺的是 HTML 报告，而你的消费方（你自己）并不需要它。

所以心智模型是一架**质量阶梯**，*选哪一级由你决定*：

| 级别 | 你要做什么 | 什么时候 |
|---|---|---|
| **下限**（零投入） | 直接采用 `agent_search` 的确定性启发式相关性分数，然后收工。 | 只有当你确实要速度而不是质量时（快速摸底范围、一次性查点东西）。 |
| **人类质量**（默认） | 在其上叠加**你自己的**语义判断：按 RCS 评分表给每篇论文打分、迭代到饱和、通过引文扩展。 | 每一次真正的检索。这才是 "agent mode" *本该*产出的东西。 |

**`agent_search` 计算的确定性启发式相关性分数是下限，不是上限。** 它是一个基于查询、始终存在、零成本的信号，保证你*永远不会*收到完全没有排序的数据 —— 但它是覆盖率启发式，不是相关性判断。它分不出奠基性论文和堆砌关键词的论文，而且（见下面的 SS 注意事项）它会压低没有摘要的经典文献。把下限当成答案，就是选择了零投入那一级。**默认要爬到它上面去。**

> 下面这条命令内建了 Agent 手工实现时经常跳过的那些*确定性纪律*（多策略检索、
> 联邦去重、始终打分、饱和度 + 配额信号）。它**没有**内建*语义判断* ——
> 里面不跑任何 LLM。那部分判断是你的工作，也正是"下限级"结果和"人类质量"
> 结果之间的区别。
> **只有调用方能降低标准；
> 命令永远不会替你降低标准。**

---

## 何时使用 agent 模式（vs 14 步人类流程）

| 你是…… | 用 |
|---|---|
| 一个把论文喂给自己的判断逻辑 / 更上层编排的 Agent | **`agent_search`**（本文件） |
| 想要一份结构化 JSON，不要 HTML，不要 PRISMA | **`agent_search`** |
| 一个人（或替人操作），想要 Shadcn HTML 报告 + 导出 | `SKILL.md` 里的 14 步流程 |

**为什么要单独开一条路径**：对真实会话的取证分析显示，当 Agent 在某个编排流程内部调用本 Skill 时，它会加载 `SKILL.md`，然后只驱动 `openalex_helper` 做原始检索，静悄悄地跳过了确定性纪律（去重、始终打分、饱和度）*以及*语义纪律（相关性判断、饱和迭代）。更严格的提示词解决不了第一个问题（Opus 级别的模型即使有 CLAUDE.md + rules + hooks，也会跳过用户定义的执行序列）。可靠的解法是**把确定性纪律内建进一条 Agent 无法绕开的命令** —— 那就是 `agent_search`。*语义*纪律没法内建进一条确定性命令（它需要 LLM：就是你）。本文件就是你把它捡回来的地方。命令保证下限；质量由你补上。

---

## 这条命令

```bash
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.agent_search "<query>" [flags] > result.json
```

`$PSP_HOME` 是 Skill 的安装目录（按 `SKILL.md` STEP 0 里的方式精确解析；**绝不** `cd` 进 Skill 目录）。查询支持数据源的布尔算符（OpenAlex / SS 的 `+` = AND）。**stdout 是一份 JSON 信封，没有别的。**

### 它内部做了什么（你免费得到的确定性纪律）

一条命令按顺序跑完整套确定性内核，其中任何一步都无法跳过：

1. **数据源路由** —— 按配置选 OpenAlex（默认）或 Semantic Scholar。
2. **多策略检索** —— 3 种策略（被引 / 近期 / 相关性），让覆盖率不依赖单一排序。
3. **联邦去重** —— DOI 规范化、版本号剥离、同标题不同 DOI 的防护；把所有策略合并成一个去重集合。
4. **字段整理** —— 每篇论文都是一个完整的 `UnifiedPaperEntity`。
5. **启发式相关性分数** —— **每篇**论文都会算，始终算（见下文）。这是下限信号；不可能被静默跳过。
6. **饱和度信号** —— 提示性的边际产出 + 分数分布信号。
7. **配额探测** —— 一次 OpenAlex 限流快照。
8. **期刊分级**（唯一的期刊层，v2.2 合并）—— 每篇论文的多平台分区（中科院 / JCR / SJR），各占一个子槽位，另加一个 `openalex` 开放影响力子槽位（2 年均被引数 + h-index）。包含 NL 意图识别（`中科院一区` → 过滤，并从查询中剥离）、可选的档位过滤，以及当档位过滤命中不足时的自适应加深（见下文）。

它**从不**渲染 HTML，**从不**写 PRISMA 日志，**从不**派发 LLM 分类 SubAgent。那些要么是人类流程该管的事（HTML/PRISMA），要么是你的事（LLM 判断 —— 见下一节）。

---

## 如何达到人类模式的质量（命令之上的那层流程）

这一节把下限结果变成人类质量的结果。默认就要做；只有当你有意选择零投入那一级时才跳过。

### 1. 用 RCS 评分表自己打相关性分（取代信任下限）

人类路径的核心质量关卡是 **RCS —— Relevance to Core Search**，由 LLM 为每篇论文打一个 0–10 的整数分。**你就是那个 LLM。** 完整读一遍 `references/rcs_rubric.md`，并把它应用到返回的论文上 —— 论文量少时直接内联处理，量大时拉起并行的分类 sub-agent（人类流程每个分类器约 10 篇；论文池大到一轮做完会粗糙时，就照这个来）。

评分表一段话概括（完整刻度 + 实例见评分表文件 —— 直接用，别自己另造分档）：

- **0–1 离题** · **2–3 边缘相关** · **4–5 相关但不核心** · **6–7 高度相关，可作引用候选** · **8–10 奠基性/开创性**。
- **与用户确切查询的相关性才是关卡，不是名气。** 一篇被引 50,000 次的论文*不一定*就是 10 分；一位名作者也不是 10 分 —— 具体这篇论文必须是对*这个*查询而言的奠基性工作。
- **纪律：相关但不核心默认给 4–5；6–7 留给直接命中；8–10 留给真正的奠基性工作。** 如果你给半批论文都打了 ≥8，你就是在虚高。两个分数难取舍时，**向下取整**。要不要标注不确定难取舍时，**就标注**。
- **不要**压低：不要因为新（还没被引 —— 按标题/摘要判断）、摘要非英文（按内容判断）或预印本而扣分。
- 特殊标记（`no_abstract_uncertain`、`off_topic_despite_keywords`、`recent_unindexed`、……）和每篇论文的确切 JSON 输出结构见评分表文件 —— 输出它们，好让你自己的下游推理能看见这个不确定性信号。

你给出的 RCS 排序 —— 而不是启发式下限 —— 才是你读、引用、往下传的东西。下限只是用来保证没有任何东西是未经排序到达的。

**注意前置过滤器 —— 它们跑在下限上，*在你的 RCS 看到任何论文之前*。** `--limit` 和 `--min-relevance` 在检索时按确定性下限分数裁剪结果集；被 `--min-relevance 0.4` 丢掉的论文，在你用自己的判断力把它救回来之前就已经没了。在 **zh 空间下限尤其弱** —— `heuristic_v1` 分词器只认拉丁字母，所以中文查询得到的 `terms` 几乎为空，下限会塌缩向"被引数 + 新近度"（§5、§"相关性评分"）。因此在 zh 查询上**不要收紧 `--min-relevance`**（保持 `0.0`），也要当心过紧的 `--limit`：先把论文放进来，打完分再用你自己的 RCS 做筛选，否则你会静悄悄地丢掉那些下限根本打不了分的好中文论文。

### 2. 迭代到饱和（不要停在第一次调用）

一次 `agent_search` 调用是一次探测，不是一次完成的检索。打分之后：

- 读 `meta.saturation` —— `looks_saturated`（后续策略新增的论文很少）和 `score_distribution` 是低成本的提示信号。**它们描述的是检索产出，不是相关性产出** —— 一次运行在论文数量上看起来"饱和"时，你可能还在不断找到高 RCS 的论文，反之亦然。
- 真正的停止条件是**你**的高 RCS 边际产出：只要新的调用仍然能捞出你打 ≥6 分的论文，就继续。加深（调大 `--per-strategy`、放宽 `--year-min`）并**换用改写过的词重新查询** —— 同义词、子主题、方法名、该领域的规范作者 —— 因为单个查询串永远覆盖不了一个主题。等改写不再带来新的高 RCS 论文，*并且* `meta.saturation.looks_saturated` 也认同时停下。那种收敛就是 agent 模式对应人类流程那条发现曲线的等价物。

### 3. 通过引文扩展（找回关键词检索漏掉的东西）

关键词检索会系统性地漏掉那些用不同词汇表达的奠基性工作。对你 RCS 最高的那些论文，**走一遍引文图谱**：拉出它们的参考文献（向后 —— 它们建立在哪片地基上）和被引文献（向前 —— 谁在它们之上继续建），然后用同一套评分表给新来者打分。这就是 agent 路径能在开创性层面追平人类召回率的方式。用 `--verify-refs`（见下文）确认任何你拿不准的引用是否真实存在，再依赖它。从少数几篇高 RCS 锚点出发走两跳，通常就能翻出那些扁平关键词检索埋掉或漏掉的规范论文。

### 4. 用自己的判断选引擎（AI→SS 的情形）

`primary_source`（OpenAlex / SS / auto）是**引擎 (engine) 轴** —— *谁来做主检索*，与语言和补充数据源正交。默认的 OpenAlex 对大多数领域都对，但**AI / CS / ML 主题上 Semantic Scholar 的覆盖明显更好**（它密集索引 arXiv + 会议论文集；实测它能捞出像 *Attention Is All You Need* / *Qwen2.5* 这样 OpenAlex 在引用数上低估的工作）。如果你判断查询属于 AI/CS，就优先用 SS 引擎 —— 要么持久设置（配置里 `primary_source: semantic_scholar`），要么针对单次查询用**临时 flag `--primary-source semantic_scholar`**（单次使用，永不持久化 —— 这是 `--lang` 在引擎上的对应物，仅本次运行覆盖配置）。这是你有能力做出的判断；没有任何东西会自动替你路由。

**SS 作为主源必须有 SS API key —— 而且它不会静默回退。** 如果 `primary_source` 解析为 `semantic_scholar`（通过配置或 `--primary-source`）而没有配置 `semantic_scholar_api_key`，这次运行会**硬报错 `E_CONFIG`（退出码 4）** —— 它*不会*悄悄退回 OpenAlex（静默回退会让一个配置错误藏在一个空 429 结果背后）。所以**没有 SS key 时就不要选 SS —— 让引擎留在 OpenAlex**（默认值）并记下覆盖面的缺口；不要指望设了 `primary_source: semantic_scholar` 就会优雅降级，因为并没有这种降级。（`auto` 是个部分例外：它先从 OpenAlex 起步，只在配额探测偏低时*粘滞*切到 SS —— 所以没有 key 的 `auto` 运行会留在 OpenAlex，这没问题，*只要那次探测没有触发切换*；如果它在没有 key 时切到 SS，一样会 `E_CONFIG`。净效果：只有配置了 SS key，`auto` 才真正有用。）注意下面关于 SS 的相关性注意事项（没有摘要的经典文献排名靠后 —— 按 `citation_count` 重新排序）。

### 5. 语言空间 —— 查询原样透传，读 `meta.language`

语言是**语言 (language) 轴** —— *在哪片海里打鱼*（`en` / `zh` / `both`），与引擎和补充数据源正交。agent 路径**从不询问、也从不猜**（没有人可以回答）。它只从显式信号解析出空间，并把结果报在 `meta.language` 里供你决策：

| 人类路径 | agent 路径 |
|---|---|
| 中文查询有歧义 → 问用户一个问题 | 中文查询有歧义 → **把查询原样透传**（中文查询打到 OpenAlex/SS 本来就会返回中文结果 —— 零改动行为），并置 `meta.language.ambiguous: true` |
| 补充数据源按学科信号自动路由 | 补充数据源**只能显式开启**：`--with-nssd`（社科）/ `--with-yiigle`（医学）—— 这里没有学科信号检测，跟 PubMed/arXiv 一样没有 |
| 用散文告诉用户 | 结构化的 `meta.language` —— 由你决定要不要重跑 |

- **`--lang en|zh|both`** —— 显式设定语言空间。不给则落到 `config.search_language`（非 `auto` 的值会被静默采用，`scope_source: config`），再落到 `auto`：英文查询 → `en`；中文查询 → `zh` 透传并置 `ambiguous: true`。
- **agent 模式绝不翻译 —— 这是与人类路径的关键区别。** 你的查询串会**原样**发给引擎。`--lang` 只做两件事：(a) 决定哪些补充数据源可达，(b) 设置 `meta.language` 注解。它**不**改写、不翻译、不换措辞。所以对中文查询用 `--lang en` **不会**产生一次英文检索 —— 它把中文字符串直接透传，只是给范围*贴上* `en` 标签；由于标签随后与实际查询语言不一致，信封会报一条 `meta.warnings`（`query_lang: "zh"` + `scope_used: "en"`）。**想要英文文献那片海，就自己给英文词** —— 你是 LLM；中→英的映射由你在自己的推理里做（就像人类路径上 `query_planner.md` 做的那样），然后再查询。同理，**`--lang both` 不做双语检索**：它打开 zh 范围，让 `--with-*` 数据源变得可达，并把*单条*查询串返回的结果合并起来 —— 它绝不会自动构建第二套翻译后的查询。要真正覆盖两片海，就发**两次** `agent_search` 调用（一条英文词、一条中文词）再合并。
- **`--with-nssd` / `--with-yiigle`** —— zh 空间的**补充源 (supplemental, a.k.a. boosters) 轴**：额外查询中文数据源，把它的结果**合并**进候选池（联邦去重，形状与其他数据源一样）。它们要求范围内含 zh；在 `en` 范围上用 `--with-*` 会把范围升级为 `both`，这样查询才真的能打到该数据源。
- **`meta.language.ambiguous: true` 是人类那句"问一句"的 agent 版本。** 它不是错误 —— 它的意思是"我把中文查询原样传了下去；如果你（拥有 CLI 没有的上下文）知道用户要英文文献，或想要中文社科 / 医学补充源，就带上 `--lang` / `--with-*` 重跑。"上下文在你手里；由你决定。

`meta.language` 结构：

```json
"meta": { "language": {
  "query_lang":           "zh" | "en",          // mechanical CJK detection
  "scope_used":           "en" | "zh" | "both",
  "scope_source":         "flags" | "config" | "query_passthrough",
  "ambiguous":            true,                   // auto + Chinese query + no explicit signal
  "chinese_sources_used": ["nssd"]               // [] unless --with-* was passed
} }
```

`meta.language` 是**增量式的，且只有在语言特性被触发时才出现**（显式的 `--lang` / `--with-*`、非 `auto` 的配置，或一个中文查询）。用默认配置跑一次纯英文检索时它不会出现 —— 信封与之前逐字节相同（R-19）。**确定性下限分数对中文查询没有变化** —— 并注意 `heuristic_v1` 只对拉丁词分词，所以中文查询下下限会塌缩向"被引数 + 新近度"，*尤其*弱。这正是第 1 步（自己打分）在 zh 空间上最重要的原因。

---

## JSON 信封结构（`schema_version: "1.0"`）

### 成功

```json
{
  "ok": true,
  "schema_version": "1.0",
  "data": [ { <UnifiedPaperEntity fields>,
              "relevance":     { "score": 0.0, "label": "high|medium|low",
                                 "method": "heuristic_v1", "is_llm_rcs": false,
                                 "components": { "title_coverage": 0.0,
                                                 "abstract_coverage": 0.0,
                                                 "citation_signal": 0.0,
                                                 "recency_signal": 0.0 } },
              "journal_rank":   { "cas": { "tier": 1, "rank": "5/100", "top": true,  /* THE single journal layer */
                                           "minor": [{"category","tier","rank"}], "source_year": 2025 },
                                  "jcr": { "quartile": "Q1", "impact_factor": 48.5,  /* the ONLY real IF */
                                           "rank": "2/135", "category": "…", "source_year": 2024 },
                                  "sjr": { "best_quartile": "Q1", "sjr": 14.5,
                                           "per_category": [{"category","quartile"}], "source_year": 2024 },
                                  "openalex": { "mean_citedness_2yr": 0.0, "h_index": 0 },  /* OPEN impact, NOT an IF (R-04/R-09) */
                                  "matched_issn": "0028-0836",
                                  "matched_platforms": ["cas","jcr","sjr"] }  /* or null; each slot independently null */,
              "verify":         { … } /* only with --verify */ },
            … ],
  "meta": {
    "query":      { "topic", "search_query", "year_min", "year_max", "per_strategy", "terms": [] },
    "counts":     { "retrieved_raw", "after_dedup", "after_relevance_filter",
                    "after_journal_filter", "after_rank_filter", "returned" },
    "relevance":  { "method": "heuristic_v1", "is_llm_rcs": false,
                    "min_relevance", "weights": {}, "note" },
    "enrichment": { "enriched", "impact_source", "impact_attached",
                    "filter_quartiles", "filter_min_impact", "category",
                    "issn_backfill_enabled", "issn_backfill_attempted",
                    "issn_backfilled", "note" },  /* single journal_rank layer audit */
    "rank":       { "platform", "platform_source", "keep_tiers", "keep_quartiles",
                    "top_only", "category", "applied_filter", "ambiguous",
                    "candidate_platforms", "kept", "filtered", "no_platform_data",
                    "data_loaded", "loaded_platforms", "switchable_platforms",
                    "switch_is_refilter_not_research", "stripped_phrases",
                    "deepen": { "active", "rounds", "saturated", "final_per_strategy" },
                    "attribution", "note", "keep_tiers_note", "naming" },  /* journal_rank partition-filter decision */
    "saturation": { "method": "heuristic_v1", "advisory": true, "looks_saturated",
                    "per_strategy_new_papers": [], "last_strategy_marginal_yield",
                    "score_distribution": { "high", "medium", "low" }, "note" },
    "ratelimit":  { <quota probe snapshot>, "switched_source": false },
    "language":   { "query_lang", "scope_used", "scope_source", "ambiguous",
                    "chinese_sources_used": [] },  /* ADDITIVE — only when the language feature is engaged; absent on the default English path (R-19) */
    "source_used": "openalex" | "semantic_scholar",
    "warnings":   [ … ],
    "verify_summary": { … }  /* only with --verify */
  }
}
```

`data` 是一个**扁平的论文列表**，按下限相关性分数降序排序，同分时按 citation_count 降序打破平局。这个顺序只是一个起点 —— 用你自己的 RCS 分数重新排序。分析信息全部在 `meta` 里。

### 失败

```json
{ "ok": false, "schema_version": "1.0",
  "error": { "code": "E_*", "message": "…", "retryable": bool },
  "meta": { … } }
```

| `error.code` | 退出码 | 可重试 | 含义 |
|---|---|---|---|
| `E_NO_RESULTS` | 2 | false | 检索跑了但没有返回结果 —— 放宽查询。 |
| `E_CONFIG` | 4 | false | 配置错误（例如查询为空，或 SS 作主源却没有 key）。 |
| `E_RATE_LIMITED` | 7 | true | 上游限流 / 配额耗尽 —— 退避后重试。 |
| `E_INTERNAL` | 6 | false | 未预期的失败（traceback 绝不会泄漏到 stdout）。 |

进程的**退出码与 `error.code` 对应**，这样非 LLM 的调用方可以直接对 `$?` 分支处理。

---

## Flags（完整参考，已对照 argparse 核验）

| Flag | 类型 / 默认值 | 作用 |
|---|---|---|
| `<query>` | 位置参数，必填 | 检索主题；布尔算符按数据源而定。 |
| `--year-min N` | int / none | 最早发表年份（含）。 |
| `--year-max N` | int / none | 最晚发表年份（含）。 |
| `--per-strategy N` | int / `50` | 每条检索策略在去重前的论文数。 |
| `--limit N` | int / 全部 | 打分 + 过滤后最多返回的论文数。 |
| `--min-relevance X` | float / `0.0` | 丢掉下限分数低于 `X`（0..1）的论文。分数**始终**会计算；这个 flag 只过滤**返回**哪些论文。 |
| `--primary-source {openalex,semantic_scholar,auto}` | choice / none | **引擎轴（临时）。** 本次运行执行主检索的**引擎** —— 覆盖 `config.primary_source`，**单次使用，永不持久化**（这是 `--lang` 在引擎上的对应物）。`semantic_scholar` **需要 SS API key，否则本次运行会以 `E_CONFIG` 失败（退出码 4）** —— 不会静默回退。`auto` = OpenAlex，配额低时粘滞切到 SS。不给 → `config.primary_source` → `openalex`。 |
| `--lang {en,zh,both}` | choice / none | **语言轴。** 检索所用的语言**空间**。不给 → `config.search_language`（非 `auto` 会被静默采用）→ `auto`。显式值优先（`meta.language.scope_source: flags`）。agent 路径从不询问/猜测。 |
| `--with-nssd` | flag / off | **补充源（zh）轴。** 额外查询 NSSD（国家哲社文献中心，中文社科）并把结果**合并**进候选池。要求范围内含 zh（`en` 范围会升级为 `both`）。只能显式开启 —— 没有学科自动路由。结果报在 `meta.language.chinese_sources_used`。 |
| `--with-yiigle` | flag / off | **补充源（zh）轴。** 额外查询 yiigle（中华医学期刊全文数据库，中文医学）并合并结果。zh 范围要求 + 报告方式同 `--with-nssd`。 |
| `--zh-block SYN1\|SYN2\|…` | 可重复 / none | **中文概念块（zh 轴）。** 每个 flag = 一个概念块，同义词用 `\|` 分隔；每个概念重复一次。作为真正的多块布尔表达式转发给 NSSD（字段代码 + 同义词 OR 展开 —— P2-5 召回修复）；**绝不推给 yiigle**（那边不支持布尔）。不给 → NSSD 仍然拿到扁平的查询：**单词元**中文查询与 v2.3 逐字节相同，但**多词**查询会按空白切成 AND 连接的概念块（P2-5 召回修复，222→519）—— **并非**逐字节相同，这是有意为之。同时也是 `--with-strategies` 下限那一层的概念块。 |
| `--with-strategies` | flag / off | **检索式导出（显式开启）。** 给信封加一个 `search_strategies` 字段，只承载**机械下限**：按卡片逐主机拼语法、邻近算符 n 值换算、A 级深链（PubMed / ERIC / ClinicalTrials.gov）、MeSH/ERIC 免费 API 核验、linter 判定。里面没有 LLM 语义（不重建概念块 / 不同义词扩展 / PRESS）。不加这个 flag 时信封与之前逐字节相同（R-19）—— 唯一被授权的例外是 `--zh-block` 那行里的多词中文 NSSD 切分（P2-5，与本 flag 无关）。见下面专门那节。 |
| `--strategy-platforms LIST` | csv / none | **`--with-strategies` 的配套项。** 覆盖下限那一层的主机集合。不给时的默认：`pubmed,wos` —— 加上 `cnki`（当 zh 被触发时：中文查询 / zh 或 both 范围 / 给了 `--zh-block`）。 |
| `--verify` | flag / off | 给每篇论文附上存在性 + 摘要 + 跨源一致性标记（见下文）。 |
| `--quartile Q1,Q2` | csv / none | 对 `journal_rank.sjr.best_quartile`（唯一那一层）的**显式开启**式过滤；只保留这些 SJR 分区。分区标签始终会附上。需要已缓存的期刊分级数据（`journal_rank fetch` → `ranks/`）。是 SJR 分区，**不是** JCR（R-04）。要控制区/category 用 `--rank-platform`。 |
| `--min-impact X` | float / none | 丢掉开放期刊影响力（`journal_rank.openalex.mean_citedness_2yr`，OpenAlex 2 年均被引数）低于 `X` 的论文。**不是** JCR 影响因子（R-09）；只用于相对比较。 |
| `--journal-category NAME` | str / none | **已废弃**（旧的仅 SJR 层）。改用 `--rank-platform sjr --rank-category` 在唯一那一层里锁定某个类目的分区。 |
| `--sjr-csv PATH` | str / none | **已废弃 / 忽略** —— 旧的 sjr_helper 仅 SJR 路径已退役。改用 `journal_rank fetch --platform sjr` 缓存 SJR 数据（→ `ranks/`）。 |
| `--no-journal-metric` | flag / off | 完全跳过期刊分级增强（`journal_rank` 保持 `null`：没有分区标签，也没有 OpenAlex 开放影响力）。 |
| `--rank-platform {cas,jcr,sjr}` | choice / none | **多平台层**（A 线分区特性）。本次运行用于**过滤**的平台：`cas`（中科院区）/ `jcr` / `sjr`。不给就从查询里的 NL 意图取平台（`中科院一区` → cas），否则取配置 `rank.default_platform`（它只**打标签**，从不过滤）。中科院区与 SJR 分区是**分区**；只有 JCR 是真正的影响因子（R-04）。 |
| `--keep-tiers 1,2` | csv / none | **多平台层。** 要**保留**的档位/分区，映射到所选平台：中科院用区号（`1,2`），JCR/SJR 用分区（`Q1,Q2`）。显式开启：不给出时，每篇论文都会被打上三个平台的标签，但不过滤任何东西。带档位过滤时，检索会**自适应加深**以达到目标数量。 |
| `--rank-category NAME` | str / none | **多平台层。** 把分区锁定到某个具体子类目，而不是期刊的最佳/大类：中科院读该类目的**小类档位**，SJR 读**逐类目分区**，JCR（源数据没有逐类目分区）把它当作**类目归属闸门**。不在锁定类目里的期刊会被过滤掉。类目匹配不区分大小写，按子串匹配。 |
| `--deepen-target N` | int / `--limit` 或 10 | **多平台层。** 档位过滤生效时，自适应加深要达到的目标留存数量。加深会以更大的深度重新检索，并对**同一批**已标注的论文池**重新过滤** —— 事后切换平台/档位是重新过滤，不是重新检索。 |
| `--no-issn-backfill` | flag / off（默认开启回填） | **仅 SS 主源路径。** 默认情况下，缺 ISSN 但有 DOI 的 SS 记录会走一次免费的 OpenAlex 单篇查询补回 ISSN，这样 `journal_rank` 关联不会悄悄丢失。传入它可以跳过 SS 结果集上的这些逐篇查询 —— 论文将保持无法关联（`meta.enrichment.issn_backfill_*` 审计会让缺口保持可见）。对 OpenAlex 路径无影响。`meta.enrichment.issn_backfill_enabled` 报告这个开关。 |
| `--verify-refs FILE.json` | path / none | **反幻觉模式**（而不是主题检索）：核验一批参考文献是否**真实存在**。位置参数查询省略。见下面专门那节。 |

所有过滤器遵循同一个约定：**信号即旋钮。** 底层数值（相关性分数、期刊分级 / 开放影响力）始终会计算，并附到每篇论文上；flag 只决定哪些论文被返回。`meta.counts` 暴露每一阶段的计数（`after_relevance_filter`、`--quartile`/`--min-impact` 的 `after_journal_filter`、`--rank-platform` 档位过滤的 `after_rank_filter`），所以过滤过程完全可审计。

---

## 相关性评分 —— 下限信号（以及它*不是*什么）

`agent_search` 为每篇论文计算一个**确定性、基于查询的启发式分数**。**这就是下限**（见北极星）：一个零成本的排序，保证没有任何东西是未排序就到达的。它明确**不是**相关性判断，也**不是**人类流程里基于 LLM 的 RCS（Relevance to Core Search）—— `method` 永远是 `"heuristic_v1"`，`is_llm_rcs` 永远是 `false`。没有 LLM 参与；不编造任何东西；公式是公开的：

```
score =  0.50 · title_coverage      # query terms found in the title (strongest precision signal)
       + 0.25 · abstract_coverage   # query terms found in the abstract (broader, noisier)
       + 0.15 · citation_signal     # log10(cites+1)/log10(2001), capped at 1 (quality prior)
       + 0.10 · recency_signal      # 25-year linear window; this year = 1, ≥25 yrs ago = 0
```

四个权重加起来是 1.0，所以最高分是 1.0。`components` 返回每一项，所以分数完全透明，不是黑盒。`label`：`high ≥ 0.6`，`medium ≥ 0.35`，否则 `low`。

关键性质：一篇离题论文（零覆盖）最多只能靠被引数 + 新近度拿到 0.25，所以**一篇被引极多但离题的论文永远不会排在真正切题的论文前面。** 但下限是一个*覆盖率*启发式，不是*相关性*启发式 —— 它分不出奠基性论文和关键词匹配的论文，而且对语义是盲的。**把它当作起始顺序，然后用你自己的 RCS 分数重新排序**（流程第 1 步）。你不会收到没有下限分数的数据；但你应该很少*停在*它上面。

词项处理：小写 → 去掉布尔标点（`+`、括号）→ 去掉停用词（包括 `study/review/effect/role` 这类学术噪声词）→ 保留长度 ≥ 3 → 去重。覆盖判定按**词集合成员关系**（不是子串），所以 `cat` 不会匹配 `category`。分词器只匹配**拉丁词** —— 对 CJK 查询，`terms` 为空，下限塌缩向被引数 + 新近度，所以在 zh 空间上你自己的 RCS 打分不是可选项，而是整个排序本身。

**SS 主源下的注意事项：默认顺序会压低没有摘要的经典文献。** 0.25 的摘要权重假定记录带摘要。在 OpenAlex 上几乎总是如此；在 **Semantic Scholar** 路径上则常常不是 —— SS 批量端点只给少数记录返回摘要（实测在 "prospect theory" 上约 40%），而缺口集中在**高被引的奠基性工作**上（旧经典的 `abstract=None` 就这么回来了）。对这些论文，摘要项得 0 分，总分塌缩到 `title(0.50)+recency(0.10)`，于是近期、引用少但*确实*带摘要的论文可能排在那些开创性论文之前 —— 实测：SS 主源下，Kahneman & Tversky 1979（36,671 次引用）掉到第 7 位（分数 0.65，`abstract_coverage:0.0`），被 2022/2026 年约 0.87 分的论文压住。这是已知的 `heuristic_v1` × SS 数据形态交互，**不是 bug**：每个信号仍然都在且透明（每篇论文都返回 `components` 和 `citation_count`）。所以当 `meta.source_used == "semantic_scholar"` 时，**不要**信默认相关性顺序去找规范文献 —— 在决定读什么之前**按 `citation_count` 重新排序**（或把它并进你自己的 RCS）。（叠加因素：这些老经典往往也缺可回填的 ISSN，于是它们同样错过 `journal_rank` 关联 —— 见 `references/journal_metrics.md`。）


---

## `--verify` —— 存在性与一致性标记（不额外发网络请求）

`--verify` 是 Agent 最需要、也最常做错的能力（编造引用）。每篇论文都会拿到一个 `verify` 块，**完全**由已经取回的记录推导 —— **零额外 API 调用，不可能编造**：

| 字段 | 含义 |
|---|---|
| `exists` | 论文有 DOI 或 OpenAlex/SS/PubMed/arXiv id 时为 True（它是从真实数据源查询里回来的 → 按构造就存在）。 |
| `has_doi` / `has_abstract` | 显式布尔值，让你知道想引用的元数据是否缺失。 |
| `multi_source` / `source_count` | 被 ≥2 个数据源命中 = 更强的存在性证据。 |
| `title_year_present` | 跨源一致性的代理指标：标出缺标题/年份的记录（Agent 最常引用的两个字段）。 |
| `flags[]` | `no_doi` / `no_abstract` / `no_title` / `no_year` / `single_source` 中的任意一个。 |

`meta.verify_summary` 汇总：total / with_doi / with_abstract / multi_source / missing_title_or_year。（这个 verify **不会**逐个 HTTP 解析 DOI —— "来自真实数据源查询"本身已经是很强的存在性证据；逐个 DOI 解析只会增加延迟 + 一处新的失败面，换不来任何约定上的收益。）

**在这个模式下 `multi_source` 的区分力很弱。** `agent_search` 从*单一*主源检索，所以一次运行里一篇论文几乎不会被 ≥2 个源命中 —— `multi_source` 基本是 `false`，`flags` 基本带 `single_source`（实测：5/5）。把它读作单源检索的产物，**而不是**可靠性红旗。（一个例外：带 `--with-nssd` / `--with-yiigle` 时，一篇带 DOI 的论文可能同时被主源和中文源命中，这时 `multi_source` 才有意义。）要做真正的**跨源存在性**核验（反幻觉用例），用下面的 `--verify-refs`，它会真的按每条引用去查 OpenAlex → CrossRef → SS。

---

## `--verify-refs FILE.json` —— 直接存在性检查（反幻觉入口）

上面的 `--verify` 只标注主题检索**刚刚返回**的论文。但 Agent 从本 Skill 得到的最有价值的东西恰恰是反面：**"我手上已经有一批打算引用的论文 —— 告诉我哪些真的存在，这样我就不会引用一条编造的文献。"** `--verify-refs` 就是这个直接入口。它**不**做主题检索；你把参考文献交给它，它逐条解析。（这也是在引文扩展阶段确认你捞出来的引用是否真实存在的工具 —— 流程第 3 步。）

```bash
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.agent_search --verify-refs refs.json > rulings.json
```

**输入**（`refs.json`）—— 一个参考文献 JSON 列表，每条带 `doi` 和/或 `title`。也接受顶层为 `{"references": [...]}` 或 `{"refs": [...]}` 的对象；裸字符串条目按标题处理：

```json
[ { "doi": "10.2307/1914185", "title": "Prospect Theory" },
  { "title": "Attention Is All You Need" },
  { "doi": "10.9999/this-was-hallucinated" } ]
```

**它对每条参考文献做什么** —— 运行时联网，对**免费的权威数据源**（OpenAlex → CrossRef → Semantic Scholar）做存在性交叉核对：

- **有 DOI** → 依次试 OpenAlex `get_work`、CrossRef（`_fetch_doi`，DOI 的权威登记处）、SS 单篇查询。第一个命中即确认存在；解析成功的 DOI 是最强的证据。
- **只有标题** → 做一次 OpenAlex 标题检索；只有当候选的实义词集合与请求标题的重叠度达到或超过 `title_match_threshold`（默认 0.85）时才接受。弱匹配会被报告但**不**接受（防误报 —— 我们宁可判"没找到"）。
- **DOI 失败但标题能解析** → 按标题判定为已核验，并附一条说明：给出的 DOI 很可能有误，应该用 `canonical.doi`。
- **哪儿都解析不到** → `exists: false`，并附说明。我们绝不编造"很可能真实"的结论 —— 抓出编造正是全部意义所在。

**输出** —— `{ ok, schema_version, data: [<ruling>...], meta: {...} }`，其中每条 ruling 是：

```json
{ "ref":            { ...the input ref, echoed... },
  "exists":         true,
  "matched_source": "openalex" | "crossref" | "semantic_scholar" | null,
  "canonical":      { "title": "...", "year": 1979, "doi": "...", "venue": "..." } | null,
  "note":           "DOI resolved in OpenAlex." }
```

`meta.summary` = `{ total, verified, not_found, by_source }`；
`meta.sources_available` 报告哪些解析器初始化成功（某个源初始化失败会被跳过，不是致命错误 —— 有 DOI 的文献仍可通过其他源核验）。
退出码与检索时同一张表（输入文件缺失/损坏 → `E_CONFIG`，参考文献列表为空 → `E_NO_RESULTS`）。

> 在写任何参考文献列表之前先用它：把你起草好的参考文献喂进去，丢掉每一条
> `exists: false` 的裁定，并根据 `canonical` 修正标题/年份/DOI。

---

## `--with-strategies` —— 检索式导出（信封里是下限，质量在你手上）

v2.4 增加了一项可导出的交付物：面向外部平台（PubMed / WOS / Scopus / Embase / CNKI / 万方 / SinoMed …）可直接粘贴的专业检索式 —— 这是 agent 路径对应人类流程 STEP 11.5 的版本。分工与整份文件建立在的那个分工完全相同：**命令保证下限；质量由你补上。**

- **默认（不加 flag）：什么都不变。** 信封不带 `search_strategies` 字段，与之前逐字节相同（R-19；唯一被授权的例外与本 flag 正交 —— 自 v2.4 起，多词中文查询的 NSSD 调用会按空白切成 AND 连接的块，即 P2-5 召回修复 —— 见 `--zh-block`）。
- **`--with-strategies` = 机械下限。** 信封多出一个顶层 `search_strategies` 字段 —— 一份 `search_strategies.json` 形态的完整记录（`schema_version` / `topic` / `register` / `concept_model` / `strategies[]` / `supplementary` / `global_review_points`）—— 只装确定性代码自己能产出的东西：逐主机语法拼装、从表里换算邻近算符 n 值、免费 A 级平台的深链（PubMed / ERIC / ClinicalTrials.gov —— 绝不生成绕过登录/验证码墙的 URL）、通过免费 API 核验 MeSH/ERIC 条目、以及 linter 判定（被 linter 拒掉的字符串会被扣住，绝不输出）。策略对象带 `platform` / `host` / `strategy_string` / `deep_link` / `vocab_verification_status` / `controlled_vocab_terms` / `linter` / `review_points`。下限对自己的身份很诚实：它的 `concept_model` 只是把清理过的查询当作单个概念块 —— 或原样照搬你的 `--zh-block` 块 —— **不含任何编造的同义词**；`register` 标为 `quick`；PRESS 领域是待复核的占位符；并且有一句 `floor_notice` 字符串明说 LLM 质量层被有意省略。下限的主机默认 `pubmed,wos`（zh 被触发时加 `cnki`）；用 `--strategy-platforms` 覆盖。下限**不**包含的东西 —— 因为命令内部不跑 LLM —— 恰恰是那些让检索式变得专业的语义判断：概念块重建、同义词/入口词扩展、逐平台受控词候选、按学科裁剪平台集合、PRESS 自查。
- **达到人类质量（两条路，选一条）：**
  1. **把你自己的语义喂给生成器。** 按 `references/search_export/methodology.md` 自己完成第 1 层的工作（zh 轨道另加 `references/search_export/chinese_methodology.md`）：按分块纪律重建概念块、扩展同义词、逐平台提出受控词候选、裁剪平台集合；把结果写成 `concept_model.json`（包装形态：`{topic, framework, register, language_space, search_id, platforms, concept_model}`）；然后跑 `PYTHONPATH=$PSP_HOME python3 -m scripts.search_export.generate --concept-model <file> --out <dir>`，得到完整的双文件交付物（`search_strategies.md` + `.json`）。
  2. **或者把人类路径从头走一遍** —— SKILL.md 的 STEP 11.5 是一份流程，不是一个 UI 功能；里面除了触发条件，没有任何东西需要人在环。
- **两个层级上诚实契约都不变：** 三态词表标签（只有 MeSH/ERIC 可能永远是 🟩机械已验；Emtree/CINAHL/APA/CMeSH 保持 🟨 标注"待人工核对" —— 绝不伪称已核验）、质量声明封顶在"专业初稿 + 标注复核点"、订阅墙平台标注"执行需机构登录"、CNKI/万方/SinoMed 以"粘贴进检索框"的检索式交付，而不是 URL。

📖 方法论这一族文件在 `references/search_export/`：`methodology.md`（语义规则手册）、`chinese_methodology.md`（CNKI/万方/SinoMed 不是逐词照搬的西方布尔式）、`press_checklist.md`、`filters_library.md`、`syntax_cards/<host>.md`（逐主机语法事实，生成器机器读取）。

---

## 数据源选择 —— 引擎 (engine) 轴

引擎在 `~/.paper-search-pro/config.yaml` 里持久设置（`primary_source`），**或者**用 `--primary-source` flag 为某一次运行临时设置（单次使用，永不持久化 —— 它只在这次运行覆盖配置；不给则回落到配置）。这是**引擎轴** —— 与语言空间和补充数据源正交（见流程第 4/5 步）：切换引擎改变的是*谁来做主检索*，而不是你在哪片海里打鱼、也不是加了哪些补充数据源。

- `openalex`（默认）—— v2.0/2.1 行为不变。
- `semantic_scholar` —— SS 作主源。**必须有 `semantic_scholar_api_key`，否则本次运行会以 `E_CONFIG` 硬失败（退出码 4）—— 它不会静默回退到 OpenAlex**（没有 key 时 SS 在共享池上会立刻 429，R-06；把配置错误摆到台面上是刻意的）。SS 主源的记录经常缺 ISSN；`agent_search` 会通过免费的 OpenAlex DOI 查询把它补回来，这样 `journal_rank` 关联不会悄悄丢失（`meta.enrichment.issn_backfilled` 报告补回了多少）。没有 DOI 的论文保持无法关联，缺口保持可见（`meta.enrichment.issn_backfill_*`）。AI/CS 查询优先用这个引擎（流程第 4 步）。
- `auto` —— 通常走 OpenAlex，但当 OpenAlex 的美元预算见底时，运行级配额探测会粘滞回落到 SS（`quota_fallback`、`quota_fallback_threshold_usd`）。`meta.ratelimit.switched_source` 会报告这件事。
- 无论怎么设置，如果 OpenAlex 在运行中途停止服务（预算耗尽、被限流、宕机），剩余检索由 Semantic Scholar 承接。`meta.ratelimit.switched_source` 变成 true，`meta.ratelimit.fallback` 给出 `{reason, reset_seconds, served_by}`。`source_used` 指明回退后的数据源，而 OpenAlex 专属的步骤（加深、开放影响力查询、ISSN 回填）在本次运行余下部分被跳过。如果 Semantic Scholar 也接不住（`served_by: []`），运行会带着已成功的那些策略继续下去（一直如此），并在 `warnings` 里说明损失了什么。自适应加深那一轮中途宕机会保留上一轮完整的结果（`meta.ratelimit.deepening_stopped`）。此时 `--min-impact` 不再生效（它的数值只来自 OpenAlex），并有一条警告说明。`--verify-refs` 会改为在 CrossRef 里查只有标题的文献，并把查不了的标为 `unchecked`，这不等于"没找到"的裁定。`quota_fallback: false` 会关掉这个切换。

`meta.source_used` 永远告诉你这次运行实际由哪个数据源服务。

---

## 多平台期刊分区（中科院 / JCR / SJR）—— A 线 `rank` 层

这就是**那个**期刊层（v2.2 单层合并）：它给每篇论文打上**全部三个**平台的标签（外加一个 `openalex` 开放影响力子槽位），并在请求了档位时按**其中之一**过滤。显式开启的 `--quartile` / `--min-impact` 过滤器也读这份记录（`journal_rank.sjr.best_quartile` / `journal_rank.openalex.mean_citedness_2yr`）；不再有单独的逐篇 `journal_metric`。数据在运行时由 `scripts/journal_rank.py` 从公开镜像取到用户本地缓存（`~/.paper-search-pro/ranks/`）—— **绝不随仓库打包**。首次使用需要跑一次 `journal_rank fetch`（见 `references/journal_metrics.md`）；在那之前这一层会优雅降级（`meta.rank.data_loaded: false`，各槽位保持 `null`，运行仍然成功 —— 它**绝不**就地拉取）。

**NL 意图识别（那个 bug 修复）。** 检索之前，`agent_search` 会解析查询里的分区表述并**把它从主题里剥离**，这样 `中科院一区 情绪调节` 找的是 情绪调节 *并限定* 中科院 1 区，而不是关于中科院一区的论文。`meta.query.search_query` 显示实际发给引擎的清理后主题（没有分区表述时 `== meta.query.topic` —— R-19 默认路径）。

**平台解析优先级**（`meta.rank.platform_source`）：
1. **`flags`** —— 显式的 `--rank-platform`。
2. **`intent`** —— NL 表述（`中科院一区` → cas）。
3. **`default`** —— 配置 `rank.default_platform`（出厂 `jcr`）—— 它只**打标签**，**不**过滤（"没要求分区 → 不过滤"）。
只有 `Q1` 而没有平台 → `meta.rank.ambiguous: true` + `candidate_platforms`；本次运行**不会**过滤（识别器从不猜平台）—— 调用方 Agent 应当**问**用户是哪个平台（CLI 是非交互的）。

**`meta.rank` 块**报告本次运行的分区决策：

| 字段 | 含义 |
|---|---|
| `platform` / `platform_source` | 用于过滤的平台 + 它是怎么选出来的（`flags`/`intent`/`default`/`none`）。 |
| `keep_tiers` / `keep_quartiles` / `top_only` / `category` | 请求的过滤条件。 |
| `applied_filter` | 只有档位过滤器真的跑过才为 True。 |
| `ambiguous` / `candidate_platforms` | 只有分区没有平台的情况：问用户。 |
| `kept` / `filtered` / `no_platform_data` | 三分；不在该平台表里的期刊会被计数，**绝不静默丢弃**。 |
| `data_loaded` / `loaded_platforms` | 分级数据是否已缓存，以及缓存了哪些平台。 |
| `switchable_platforms` / `switch_is_refilter_not_research` | 换标准/换档位是对**已标注论文池的重新过滤** —— 不重新检索。 |
| `stripped_phrases` | 从查询里剥掉的分区表述（透明化）。 |
| `deepen` | `{active, rounds, saturated, final_per_strategy}` —— 自适应加深的状态。 |
| `keep_tiers_note` | 当只给了裸数字 `--keep-tiers` 而没有 `--rank-platform` 时设置：这些数字被读成了默认平台的分区（出厂 JCR `Q1,Q2…`）；它提示调用方补上 `--rank-platform cas` 才能用于中科院区。否则为 `None`。 |
| `attribution` / `note` / `naming` | 平台归属、降级说明、R-04 提醒。 |

**自适应加深**（只在档位过滤器生效、数据已加载、查询无歧义、且 `source_used == openalex` 时）：第一次检索先标注 + 过滤；如果留存数量不够 `--deepen-target`，检索会以更大的深度重新检索，并对**同一批**已标注的论文池重新过滤，直到达到目标、达到 `max_deepen_rounds`（3），或者更深一轮没有新增去重结果（饱和）。档位过滤生效时排序优先按**被引数**（高区 ↔ 高被引，r≈0.96，因此高档位能更快浮出来、超额抓取更少）；没有过滤时默认相关性顺序不变。代价只在检索这一段 —— 没有额外的 LLM 分类（headless 永远不分类）。SS 路径没有深度旋钮，所以 SS 作主源不会加深（`meta.rank.deepen.active` 会体现这一点）。

**计数。** `meta.counts.after_rank_filter` 是多平台 `--rank-platform` 档位过滤后的留存数量；`after_journal_filter` 是 `--quartile` / `--min-impact` 这些显式开启的过滤器之后的留存数量（两者现在都取自 `journal_rank`）。两者都在，且互相独立。

关于数据源、ISSN 关联、归属、拉取方式，以及"绝不把它叫作影响因子"这条规则，见 `references/journal_metrics.md`。

---

关于期刊分区平台（CAS / JCR / SJR 的获取方式、OpenAlex 开放影响力槽位、ISSN 关联、命名规则），见 `references/journal_metrics.md`。
