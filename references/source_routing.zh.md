# 来源路由

*本文件由主 Agent 在 `SKILL.md` 的 STEP 1-2 期间读取。两个阶段有先后顺序：**STEP 1 决定语言空间**——下面的 §"Language scope" 是它的唯一事实来源（到哪片海洋里捕鱼，**包括中文检索式存在歧义时的那一次首次提问**）。**STEP 2 随后在这个已经确定的空间*内部*路由 L2/L3 补充来源**（本文件的其余部分——最终落到具体用哪些源）；STEP 2 不会重新决定空间。两个阶段的做事方式相同：**由你（LLM）做语义判断；表格是校准示例，而不是机械的关键词匹配；而确定性事实（CJK 检测、显式的 `--flags`）留在代码里。***

*来源路由规则基于 SA-V3 实测：20 条基准真值检索式上准确率 85%（24_v3_domain_signal_test.md），纯社科 / 人文上的误触发率 0%，医学或 CS 检索式上的漏触发率 0%，唯一的错误模式是跨学科检索式静默退化成单源。*

## 如何阅读本文件里的每一张信号表（语义判断，不是字符串匹配）

下面的关键词表（医学 / CS / 语言标记）是为了**校准你的判断**而存在的，不是用来 grep 的。**由你决定**一条检索式是医学、CS、社科，还是想找中文文献——用示例作为各领域样貌的锚点。一条不含任何所列词的检索式仍然可能明确属于医学；而一条恰好包含某个所列词的检索式（比如心理学语境里的 "attention"，而不是 "attention mechanism"）则可能不属于。判断的是语义，不是 token。

三件事保持**机械 / 确定性**（绝不靠 LLM 猜）：

- **CJK 检测**——检索式是否包含中文字符是一个代码事实（`detect_language.py`），也是下面 `auto`→提问分支的进入条件。
- **用户显式给出的 flag / 覆盖**——`--lang en|zh|both`、`--no-pubmed`、`--no-nssd`、`--source …` 以及显式持久化的 `config` 值一律照原样执行；它们优先于任何判断（见优先级阶梯）。*（在人类路径上，这些 `--flag` 写法是**自然语言覆盖的*记号***——一种紧凑写法，代替用户可以*说出口*的话，由你（LLM）去解读，不是任何脚本会解析的 token。真正被脚本解析的 flag 只存在于 `agent_search` 路径，而那里的中文源控制是**需主动开启**——只有 `--with-nssd` / `--with-yiigle`，从来不是 `--no-nssd` / `--source`。见 `agent_mode.md`。）*
- **下游的 `federate → dedup` 步骤**是确定性的；路由只决定*哪些*源喂给它。

在一个*语义*判断上拿不准时，优先保召回（把源加进来）——L2/L3 增强器只会增加候选，而且每个都可以用一个 flag 单独去掉。

## 语言空间（坐标轴 2 —— 在 STEP 2 的来源路由之前，于 STEP 1 决定）

*(v2.3，增量式。这是**三轴模型**的坐标轴 2：**引擎** `primary_source` = 谁来跑主检索；**语言** `search_language` = 到哪片海洋去检索；**增强器** = 在选定的那片海洋里按学科自动路由的网。**坐标轴 3 对同一个概念有三个可互换的名字——`boosters` = `补充源` = `supplemental sources`——在 SKILL.md / 本文件 / `agent_mode.md` 中可互换使用；它们绝不指代不同的东西。** 默认的 `search_language: auto` 对英文检索式完全复现 v2.2 行为——R-19。人类路径的交互脚本 / 话术在 `SKILL.md` STEP 1-2；本节是**解析逻辑**的唯一事实来源。)*

### 解析优先级（对应 rank 平台的 `flags > intent > default`）

自上而下套用；第一个命中的层级胜出，而层级 1 永远胜出：

```
1. Explicit per-query instruction  — CLI flag (--lang en|zh|both) or NL wording
                                      ("这次只要英文" / "中英都查" / "找中文的")
                                      → transient, NEVER auto-persisted.
2. In-query language marker         — §"Language markers" hit → follow it, don't ask.
3. Persisted config value           — search_language ∈ {en, zh, both} → adopt silently.
4. auto fallback:
   4a. no CJK and no explicit "want Chinese" wording → en  (don't ask; v2.2 behavior)
   4b. query contains CJK (or an English query explicitly asks for Chinese lit) →
        · human path : ask ONE question (SKILL.md STEP 1, 22 §6.1), offer to persist
        · agent/headless path : pass the query through as-is (query language = space),
          add NO boosters, mark ambiguity in meta — never ask, never guess
```

每一个层级都可以被上一层的指令覆盖（层级 1 压倒一切，就像 `--no-pubmed`）。同一会话里一旦问过，就在该会话后续的中文检索式上复用这个答案（报告一行，不要重复问）。

### 语言标记（语义判断——示例，不是匹配清单）

**→ zh 空间**（命中任一个即可；可以出现在中文*或*英文检索式中）：

| 类别 | 标记示例 |
|---|---|
| 显式语言要求 | `中文文献` `中文论文` `中文期刊` `中文的研究` `国内研究` `国内文献` `Chinese-language papers` `literature in Chinese` |
| 中文期刊评价体系 | `CSSCI` `C刊` `北大核心` `中文核心` `CSCD`（这些体系只收录**中文**期刊——点名一个就是要中文文献） |
| 点名中文数据库 | `知网` `CNKI` `万方` `维普` `NSSD`（触发 zh 空间，**并且**触发 SKILL.md STEP 4 的合规提示，22 §6.5） |
| 点名中文期刊 | `中华××杂志` `心理学报` `经济研究` 以及其他具体的中文刊名 |

**→ en 空间**（出现在中文检索式内部；命中任一个即可）：

| 类别 | 标记示例 |
|---|---|
| 显式语言要求 | `英文文献` `英文论文` `外文文献` `只要英文` |
| 英文索引体系 | `SSCI` `SCI收录`（JCR/SJR 的措辞是 rank_intent 的职责，**不是**语言信号——见下文） |

**→ both**：`中英都要` `中英文` `both Chinese and English` 以及其他显式的连词表述。

标记是**路由 / 过滤条件，不是检索词**——一旦某个标记选定了空间，就要在检索前把它从主题里剥掉（与 rank_intent 同一纪律：`CSSCI 情绪调节` 检索的是*情绪调节*，CSSCI 作为 zh 信号 + 质量标签；绝不要把 "CSSCI" 这几个字母送进搜索引擎）。CSSCI/北大核心具有双重身份（语言信号 + 质量档级），处理方式与"中科院一区"的双重身份完全一致（平台 + 档级）。

### 非信号——绝不从这些推断语言（反模式）

| 措辞 | 为什么它不是语言信号 |
|---|---|
| `中科院一区/二区…` | rank 平台信号（CAS 主要覆盖国际 SCI 期刊）；由 `rank_intent` 处理，**零**语言含义 |
| `顶刊` `top journal` | rank 档级措辞，与语言无关 |
| 主题是关于中国的（`中国` `China` `乡村振兴` `中国大学生`…） | 主题地理 ≠ 文献语言偏好；关于中国的英文文献浩如烟海 |
| 检索式恰好是用中文写的 | 只是*进入 4b 提问分支*的触发条件，**不是**开启中文源的许可（一个中英双语的研究生常常用中文描述主题，却想要 SSCI） |
| `JCR/SJR` 措辞 | 保持各轴正交——档级是档级，语言是语言；把两者耦合会让"一项 SJR-Q1 的中国管理学研究"根本无法表达 |

### 硬边界（否定面——与肯定规则同样有约束力）

1. **英文检索式且没有 zh 标记 → 本轮不存在中文源。** 不试探、不建议、也不"顺便也查中文"。这是 R-19 在交互层面的体现。
2. **持久化的 `search_language: en` + 中文检索式 → 仍然不开启中文源**，但要报告一行（SKILL.md STEP 4，22 §6.4）——绝不做静默翻译；单次检索的层级 1 指令仍然可以为这一轮覆盖它。
3. **Agent / headless 路径从不提问，也从不猜**——增强器只认显式 flag + config；有歧义就写进 `meta`，上交调用方（见 `agent_mode.md`）。
4. **单次选择绝不自动持久化**——只有明确的"以后都…"才会写入 config（rank 平台的纪律，原样复用）。
5. **工程闸门：** zh 空间开放之前，CJK 安全的去重修复（Phase 0）必须到位——没有它，中文结果集会静默塌缩成一行。（Phase 0 已在本分支落地，所以闸门是开着的。）

## 决策流程（按顺序套用——由你判断；这些是步骤，不是字符串匹配）

```
1. Detect medical signals → enable PubMed enricher
2. Detect CS/AI signals → enable arXiv's recent layer (last year, by relevance) + OpenReview;
   freshness signals → also the 4-day arXiv sentinel
3. Cross-domain whitelist match → force BOTH PubMed + arXiv (silent upgrade)
4. Pure non-L2 signals AND no medical/CS hits → OpenAlex only
5. Ambiguous → default upgrade to "BOTH PubMed + arXiv" (Recall > Precision)
6. User CLI override (`--no-pubmed` / `--no-arxiv` / `--source=...`) wins everything
```

决定之后，**对用户说一句话**：*"我检测到 [医学信号：'RCT' + 'metformin']——同时也检索 PubMed。如果只想用 OpenAlex，用 `--no-pubmed` 覆盖。"*

## 主源选择与配额回退

*(v2.2，增量式。**默认行为不变：OpenAlex 是主源，STEP 3 完全按上面的写法执行。** 下面的配置项只在用户于 `~/.paper-search-pro/config.yaml` 中设置了 `primary_source` 时才生效。它管辖**人类 14 步路径**；headless 的 `agent_search` 路径会自动读取同一份配置——见 `references/agent_mode.md`。）*

### OpenAlex 无法服务某次调用时的自动回退（无需任何操作）

与 `primary_source` 无关，每一次 OpenAlex 无法服务的 `openalex_helper` 调用都会尽可能改由 Semantic Scholar 服务，这样额度耗尽不再中断一轮检索：

- **什么算"无法服务"：** 响应头显示当日预算已用尽的 429（`X-RateLimit-Remaining` 低于 `X-RateLimit-Credits-Required`，或者没有剩余 USD；这是官方 OpenAlex CLI 的测试方式）会立即切换。其他 429 属于限流：辅助脚本会先遵守 `Retry-After` 并重试 3 次。服务端错误和网络失败各重试一次。被 OpenAlex 拒绝的 key 也算。400/404 **不**切换：它们说明检索式或 ID 错了。
- **切到哪里：** Semantic Scholar（使用配置的 key；如果 SS 拒绝该 key，则改用无 key 调用，同时在 stderr 打一行提示用户续期）。`get` 和 `citation-network` 需要 DOI；`author` / `trends` 没有等价命令。`search --type` 过滤器始终保留：SS 无法应用的类型就意味着根本不去问 SS。CrossRef 被**刻意**排除在检索回退之外：它匹配的是元数据而非内容，经常缺少摘要，还会混入非文章类型的条目，那样会返回一个看起来正常但更弱的语料。它只支撑 `get`（DOI 元数据）和 `--verify-refs` 的存在性检查。
- **保留什么：** 切换之前已经取到的 OpenAlex 记录仍留在输出里，排在最前面。回退得到的记录带有 `sources: ["semantic_scholar"]`，这样 PRISMA-S 第 1 项列出的才是真正服务过本轮的数据库。
- **Semantic Scholar 也服务不了时：** 该调用的行为与回退存在之前的 OpenAlex 报错完全一样，所以过去能跑通的东西现在依然能跑通。它以退出码 `3` 退出，stdout 上是合法 JSON（为空，或只含切换之前 OpenAlex 返回的内容，绝不冒充完整结果），stderr 那行给出原因和重置时间；像处理任何 OpenAlex 错误一样报告给用户。`journal-list` 照旧返回空列表且退出码为 0。`quota_fallback: false` 会关掉这个切换。
- **你要做的：** 用一句话把 stderr 那行转述给用户，然后继续。
- **发生这种切换后，跳过下面的 OpenAlex 回填：** 那些查询会撞上同一堵墙。

`config.primary_source`（在 `~/.paper-search-pro/config.yaml` 中）决定 STEP 3 的**主检索**由哪个源服务：

| 值 | 行为 |
|---|---|
| `openalex` *(默认)* | STEP 3 完全按写法执行 OpenAlex。下面这些都不变。 |
| `semantic_scholar` | STEP 3 改为从 Semantic Scholar 检索。**需要 `semantic_scholar_api_key`**（没有 key 时 SS 在共享池上会立刻 429——R-06）。 |
| `auto` | STEP 3 通常使用 OpenAlex，但如果 OpenAlex 当日 USD 预算偏低，本轮会粘性地回退到 SS（需要 SS key + `quota_fallback: true`）。 |

### AI / CS 检索式 → 可考虑把 Semantic Scholar 作为主源（由你判断）

除了上面的配置项之外，**当你判断检索式属于 AI / CS 核心时，你可以在某一轮把主源提升为 Semantic Scholar。** 在这些主题上，SS 的语料比 OpenAlex 更完整地覆盖 AI/CS + arXiv 预印本文献——实测来看，"attention mechanism" / "Qwen2.5" 这类检索式，其经典预印本和引文结构在 SS 上更容易浮现，而 OpenAlex 会低估它们（OpenAlex 对 arXiv 有 3 天索引延迟 + 对预印本收录不足）。这是一次**语义判断**——*这条检索式是关于 LLM / transformer / CS 系统 / ML 方法的吗？*——由下面的 arXiv 信号表校准，而不是一条机械规则。

前提条件（确定性——必须全部成立，否则你留在 OpenAlex 上）：

- **配置了 SS API key。** 没有它，SS 在共享池上会立刻 429（R-06），所以无 key 的运行会静默留在 OpenAlex——按规则 C 声明这次跳过（"没有 SS key，所以这个 AI 检索式继续用 OpenAlex；加上 `semantic_scholar_api_key` 可以获得更完整的预印本覆盖"）。
- **用户没有为这一轮钉死 `primary_source`**（或 `--lang` 式的引擎选择）——显式的 config / flag 永远优先于你的判断。

真正切换时，按下面"### 本轮切换到 Semantic Scholar 时"的流程对 SS 跑 STEP 3（同样的 `ss_helper --search`，同样的 `raw/openalex.json` 落点，同样的 OpenAlex 单篇查询回填 OA-only 字段）——这个切换只换引擎，不损失能力。对用户说一行：*"核心 AI 检索式——用 Semantic Scholar 作为主源以获得更完整的预印本覆盖；OA-only 字段由 OpenAlex 单篇查询回填。"*

### 什么时候跑 STEP 3 之前的配额检查

只有当 `primary_source: auto` **或者**用户明确要求配额感知的回退时，才在 **STEP 3 之前**跑配额检查。（普通的 `primary_source: openalex` 或 `semantic_scholar` 跳过它。）

```bash
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.quota_guard --mode run
```

这个前置检查只是提前选好引擎；跑到一半预算耗尽的情况由上面的自动回退处理。`--mode run` 读取 OpenAlex 的 `X-RateLimit-Remaining-USD` 响应头并给出一个粘性的结论：剩余 USD 低于等于 `quota_fallback_threshold_usd`（默认 `0.05`）时为 `should_switch: true`。**探测失败（`ok: false`）意味着"留在 OpenAlex"——没有证据不等于已经耗尽；绝不在探测失败时切换。**

### 本轮切换到 Semantic Scholar 时

当 `primary_source: semantic_scholar`，或者 `auto` 解析为要切换时，改为对 SS 跑 STEP 3 而不是 OpenAlex：

```bash
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.ss_helper --search "<query>" \
    --n 50 [--year-min YYYY] \
    > "$SEARCH_DIR/raw/openalex.json"
```

`ss_helper --search` 输出与 `openalex_helper` 相同的 `UnifiedPaperEntity[]` 结构，所以 STEP 4-13 不变（写到与 OpenAlex 相同的 `raw/*.json`，然后照常联邦合并）。

### 切到 SS 之后回填 OpenAlex 独有字段（不要丢能力）

SS 记录缺少若干 OpenAlex 才有的字段（`institution` / `funder` / `topics` / `fwci` / 用于 SJR join 的期刊 ISSN / `openalex_2yr_mean_citedness`）。在以 SS 为主源跑完一轮后，用**免费的 OpenAlex 单篇查询**补回来——每篇有 DOI 的论文做一次廉价的 `get`：

```bash
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.openalex_helper get "<doi>"   # free single-work lookup, repeat per DOI
```

这样切换就是能力中立的：SS 提供主结果集，OpenAlex 单篇查询回填 OA-only 字段（institution / funder / topics / fwci / ISSN→SJR join / 开放获取影响力）。没有 DOI 的论文保持未回填——这个缺口是预期之内且可见的，不是被静默拼接上去的。（R-08：缺失 ISSN 的回填正是 `agent_search` 在自己那条路径上自动化掉的事情；在人类路径上你做同样的逐 DOI `get` 恢复。）

### 命名 / 合规提醒

OpenAlex 的开放影响力数值（`openalex_2yr_mean_citedness`）**不是** JCR 影响因子，SJR 分区也**不是** JCR / 中科院（R-04/R-09）。见 `references/journal_metrics.md`。

## PubMed 启用规则

**判断，不是字符串匹配**（文件开头的规则在这里完全适用）：下面的表格是**"一条明确属于医学的检索式长什么样"的校准示例**——*由你*决定检索式是否属于医学，并据此启用 PubMed 增强器。一条不含这些词的检索式仍然可能明确属于医学；一条只是包含某个所列词的检索式（纯教育语境下的 `intervention`）则可能不属于。真的拿不准时，优先保召回（启用它）——增强器只会增加候选。

### 强医学信号——最确定的判据（命中任一个通常就意味着"医学"）

| 类别 | 标记示例 |
|----------|-----------------------------|
| **二次证据** | `RCT`, `randomized controlled trial`, `systematic review`, `meta-analysis`, `PRISMA`, `Cochrane`, `umbrella review`, `network meta-analysis`, `GRADE` |
| **临床研究** | `clinical trial`, `cohort study`, `case-control`, `case series`, `phase I/II/III`, `dose-response`, `intention-to-treat` |
| **MeSH / 医学行为** | `MeSH`, `incidence`, `prevalence`, `mortality`, `morbidity`, `screening`, `differential diagnosis`, `prophylaxis`, `intervention` (medical context) |

### 中等医学信号——较弱的线索（几个同时出现就读作明确属于医学）

| 类别 | 标记示例 |
|----------|----------|
| **疾病名称** | 具体疾病名（diabetes, cancer, hypertension, IBS, Alzheimer's, COVID-19, sickle cell, asthma, depression 等）、ICD 编码、医学专科（cardiology, oncology, nephrology, pediatrics, geriatrics） |
| **药物 / 疗法** | 具体药名（metformin, GLP-1, statin, aspirin）、`receptor agonist/antagonist`、`inhibitor`、`monoclonal antibody`、`vaccine`、`gene therapy` |
| **生物医学研究** | `genomics`, `proteomics`, `metabolomics`, `epidemiology`, `pharmacology`, `pharmacokinetics`, `biomarker`, `clinical guidelines` |
| **中文医学** | `临床`, `医学`, `治疗`, `患者`, `疾病`, `药物`, `干预` |
| **患者 / 治疗** | `patient`, `treatment`, `therapy`, `intervention`（医学语境） |

## arXiv 启用规则

**判断，不是字符串匹配**（同一条规则）：下面的表格**校准"一条 CS/AI 或时效性驱动的检索式长什么样"**——*由你*决定检索式是否需要 arXiv，并据此启用它。CS/AI 主题会拿到近期层（`arxiv_helper search --sort relevance --since-days 365`）和 OpenReview；一个时效性词则再加上 4 天哨兵。语义重于 token：心理学检索式里的 `attention` 不是 CS 检索式里的 `attention mechanism`。拿不准时，优先保召回。

### 强 CS/AI 或时效性信号——最确定的判据（命中任一个通常就意味着"跑 arXiv"）

| 类别 | 标记示例 |
|----------|----------|
| **显式预印本** | `arxiv`, `preprint`, 最新预印本名（如 "arXiv 2401.12345"） |
| **时效性词** | `latest`, `recent`, `cutting-edge`, `state-of-the-art`, `SOTA`, `frontier`, `newest`, `最新` |
| **CS 顶会** | `NeurIPS`, `ICML`, `ICLR`, `CVPR`, `ECCV`, `ACL`, `EMNLP`, `AAAI`, `IJCAI`, `KDD`, `STOC`, `FOCS` |
| **年份新鲜度** | `2024`, `2025`, `2026` 与方法类词共同出现 |

### 中等 CS/AI 信号——较弱的线索

| 类别 | 标记示例 |
|----------|----------|
| **CS / AI 核心** | `LLM`, `large language model`, `transformer`, `attention mechanism`, `BERT`, `GPT`, `diffusion model`, `GAN`, `VAE`, `RLHF`, `RAG`, `agent`, `tool use`, `prompt engineering`, `chain-of-thought`, `MoE`, `speculative decoding`, `KV cache`, `quantization` |
| **AI 可解释性 / 安全** | `mechanistic interpretability`, `sparse autoencoder`, `feature circuits`, `alignment`, `red teaming`, `jailbreak`, `RLAIF` |
| **经典 ML** | `deep learning`, `reinforcement learning`, `unsupervised learning`, `self-supervised`, `contrastive learning`, `representation learning`, `meta-learning`, `few-shot`, `zero-shot`, `transfer learning` |
| **物理 / 数学 / 理论 CS** | `quantum computing`, `tensor network`, `category theory`, `lattice cryptography`, `homomorphic encryption` |

## 跨学科白名单（强制同时启用 PubMed + arXiv）

按 SA-V3 §5.1 规则 C，当一个医学信号与一个 CS 信号共同出现——**或者**检索式命中下面任一模式时——静默地同时启用两者。这救回了 3/5 条 C 类检索式（Q12 Q13 Q15），单源启发式在它们上面会失败。

| 白名单模式 | 命中示例 |
|-------------------|------------------|
| **医学影像 AI** | `radiology image` + AI, `medical imaging` + AI, `CT scan` + AI, `MRI` + DL, `ultrasound` + ML, `pathology image` + DL, `histopathology` + AI |
| **药物发现 ML** | `drug discovery` + (transformers/GNN/DL), `molecular design` + ML, `protein-ligand` + DL, `ADMET prediction` |
| **生物信息学 DL** | `protein structure prediction`, `genomics` + DL, `single-cell` + ML, `RNA folding` + DL |
| **临床 NLP** | `clinical notes` + NLP, `electronic health records` + LLM, `EHR` + transformer, `medical chatbot` |
| **神经 ML** | `fMRI` + (DL/ML), `EEG` + (DL/ML), `MEG` + DL, `brain decoding`, `brain-computer interface` |
| **精准医学 ML** | `personalized medicine` + (RL/ML), `precision oncology` + DL, `treatment recommendation` + RL |
| **流行病学 ML** | `epidemic forecasting` + ML, `disease prediction` + DL, `outbreak` + neural network |

## 纯非 L2（负向触发——只用 OpenAlex）

如果检索式**没有**医学信号**也没有** CS 信号，默认只用 OpenAlex。典型模式：

| 纯领域 | 信号词 |
|-------------|--------------|
| **心理学（非临床）** | attachment theory, prospect theory, self-construal, social identity, prejudice, intergroup, attitude change |
| **经济学 / 商业** | behavioral economics, market design, game theory, mechanism design, monetary policy, fiscal, supply chain |
| **社会学 / 政治学** | inequality, mobility, populism, democracy, authoritarianism, social movements |
| **历史 / 文学 / 哲学** | postcolonial, literary criticism, historiography, hermeneutics, phenomenology, ethics theory, Said, Spivak |
| **教育 / 语言学** | curriculum design, pedagogy, second language acquisition, sociolinguistics |
| **法律 / 公共政策** | constitutional law, legal theory, public administration（**不是** health policy——health policy → PubMed） |

## 字段优先级表（联邦合并）

当一篇论文存在于多个源中时，联邦 KG 解析器按这个优先级挑字段。这是规范版本，取代了 22_/23_/25_ 综合文档里早期的草稿。

| 字段 | 主源 | 回退 1 | 回退 2 | 备注 |
|-------|---------|------------|------------|-------|
| `title` | OpenAlex | PubMed | SS | OA 最一致 |
| `abstract` | OpenAlex（重建） | SS（直接取） | PubMed（仅医学） | SA-V1：SS 摘要回退能覆盖约 67% 的旧论文，这些论文在 OA 上是空的 |
| `year` | OpenAlex | CrossRef（`created.date-time`） | PubMed | OA 可靠 |
| `authors` | OpenAlex | PubMed | SS | OA 有 ORCID + 机构 |
| `doi` | OpenAlex（转小写） | CrossRef | PubMed | 归一化：转小写 + 去掉 URL 前缀 |
| `pmid` | PubMed | OpenAlex.ids.pmid | — | |
| `pmcid` | PubMed | OpenAlex.ids.pmcid | — | |
| `arxiv_id` | arXiv（去掉版本号） | OpenAlex.locations[].landing_page_url | — | 去掉 "vN" 后缀 |
| `citation_count` | OpenAlex | SS | — | 对**期刊论文** SS 比 OA 少算 20-50%；**在 CS / arXiv 预印本上则相反**（OpenAlex 对 arXiv 收录不足，所以那里 SS 的计数更高 / 更全）——预印本引用数不要把 OA 当作权威 |
| `influential_citation_count` | **仅 SS** | — | — | 独有信号，没有回退来源 |
| `references` | OpenAlex | CrossRef（补充前 N 条） | SS | 在 CrossRef 里**跳过 arXiv DOI** |
| `funders` / `grants` | CrossRef | OpenAlex.grants | — | OA 的 grants 常常是空的 |
| `license` | CrossRef | OpenAlex.primary_location.license | — | CR 有 `delay-in-days` + `content-version` |
| `mesh_terms` | **仅 PubMed** | — | — | 独有信号 |
| `publication_types` | PubMed（`["Clinical Trial", "Review"]`） | OpenAlex.type | — | |
| `clinical_trial_number` | CrossRef | PubMed | — | NEJM 临床试验的 CR 返回空 —— 回退到 PubMed |
| `pdf_url` / `open_access_url` | OpenAlex（`oa_url`） | arXiv（预印本） | PubMed（PMC 链接） | |

### 中文原生记录（NSSD / yiigle）——保留中文原文

对于来源中包含原生中文源（`sources` 含 `nssd` 或 `yiigle`）的论文，**中文原文的 `title` / `abstract` / `authors` / `venue` 优先——绝不能被英文的 OpenAlex 值覆盖。** zh 空间的存在意义就是用中文返回中文文献；用 OpenAlex 的英文版本盖掉中文标题既违背初衷，也会破坏报告的中文呈现。具体来说，一条中文原生记录的优先级是：

| 字段 | 主源（中文原生记录） | 备注 |
|-------|----------------------------|-------|
| `title` | NSSD / yiigle（中文原文） | 绝不用英文的 OpenAlex 标题替换 |
| `abstract` | NSSD / yiigle（中文原文） | 中文摘要正是 zh 源的意义所在 |
| `authors` | NSSD / yiigle（中文姓名） | 保留中文字符串，不要拉丁化 |
| `venue` | NSSD / yiigle（中文刊名） | 例如 `心理学报`，而不是英文外名 |

- **常见情况是自动处理的。** 多数 NSSD/yiigle 记录不带 DOI，所以它们会拿到属于自己的 `("native", source_native_id)` 规范键（federated_kg_resolver 0.2），永远不会和 OpenAlex 记录相撞——任何合并都碰不到它们的中文字段。
- **有风险的是 DOI 相撞那种情况。** 当一条中文原生记录*确实*带了一个在 OpenAlex 中也存在的 DOI 时，两条会按 `("doi", …)` 合并，而上面的通用规则让 OpenAlex 优先于 `title`/`authors`——对这条记录来说，那会用英文替换掉中文原文，在 zh 结果集里是错误的结果。**现在这一点已被强制保证（#4，已在本分支落地）：** `merge_paper_fields` 通过 `_has_chinese_source` 检查对中文来源做了特判，所以 DOI 相撞时中文原生的 `title`/`abstract`/`authors`/`venue` 胜过英文的 OpenAlex 值（既有中文 `existing` 挡住英文 `new` 的情况，以及中文 `new` 覆盖英文 `existing` 的情况都覆盖）。所有纯英文的合并仍然保持两个标记都是 `False`，所以那些路径与之前逐字节一致（R-19）。因此上表里"不得覆盖"的约定是实打实的保证，而不是一句愿望。

## 冲突解决（E5b 守卫）

如果两条记录有相同的 `title` + `year` 但 **DOI 不同**，**保持它们分开**。它们很可能是同题不同文的论文，或者是一对不该被合并的正式发表版 + 预印本。见 `federated_kg_resolver.py` 的 E5b 守卫测试用例。

## 错误陷阱（已实测验证）

- **arXiv 的 DOI 在 CrossRef 上 → 100% 404**（SA-V1 / SA-W2）。当 DOI 以 `10.48550/arxiv.` 或 `10.48550/arXiv.` 开头时，永远跳过 CrossRef 富化。`crossref_helper._is_arxiv_doi()` 负责这件事。
- **arXiv DOI 大小写漂移**：arXiv 输出 `10.48550/arXiv.<id>`（大写 X），OpenAlex 归一化为 `10.48550/arxiv.<id>`（小写 x）。解析器必须在比较前转成小写。`_strip_doi_prefix()` 已经处理了这一点。
- **K&T 1979 前景理论在 OpenAlex 上 `references=[]`**——这是 OpenAlex 上游对 2000 年前论文的撤下。接受这个空列表；不要重试，也不要试图修复。SA-Z2 F19 已确认。
- **NEJM 临床试验论文的 CrossRef `clinical-trial-number` 为空**（SA-V1）——回退到解析 PubMed 的 `clinical_trial_numbers`。
- **OpenAlex 对 arXiv 有 3 天索引延迟**：T-0 到 T-2 的论文在 OpenAlex 上命中率为 0%，T-3 为部分命中——4 天哨兵覆盖这个区间。再往后 OpenAlex 对 arXiv 仍不完整（2026-09-30：20 篇在题的近期 OpenReview 论文里有 7 篇不在 OpenAlex 中），这就是 CS/AI 主题也要跑一年近期层的原因。
- **OpenReview（仅限 CS/AI）**：已录用的 ICLR / NeurIPS / ICML / COLM / TMLR 论文及其录用标签；见 `openreview_helper_cheatsheet.md`。只有 search 端点可用（按会议列出在人工核查之后）；空结果意味着被封或没有录用，不意味着"没有文献"。
