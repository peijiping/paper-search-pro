# pubmed_helper CLI 速查表

由 `SKILL.md` STEP 4（医学检索的 L2 增强器）引用 —— 主要用途：为 OA 论文做 MeSH 增强；Audit 档的次要用途：为 PRISMA 流程做独立的 MeSH 检索。

**角色**：L2 增强器。PubMed 的价值在于**通过 MeSH + publication_types + PMC URL 带来的精确度**，而不是独立召回（SA-V2 24_v2_l2_booster_test 已验证：30/30 篇"PubMed 独有"的论文在全球范围内本来就已在 OpenAlex 中）。默认把它当增强器用，只有在明确的 PRISMA 流程中才做独立检索。

**入口**：`python3 -m scripts.pubmed_helper <subcmd> [args]`。
**初始化来源**：`init(load_config())` —— 使用 `ncbi_email`（即使没有 key 也是 NCBI ToS 的要求）+ `ncbi_api_key`（把 3 → 10 req/s）。

## 四个子命令

| 子命令 | 用途 | 档级 |
|---|---|---|
| `enrich` | 批量增强已带 `pmid`（来自 OpenAlex）的论文 | Standard / Deep |
| `search-mesh` | 独立的 MeSH 精确检索（PRISMA 流程） | 仅 Audit |
| `search` | 通用关键词检索（内部自动扩展到 MeSH） | Audit 兜底 |
| `by-pmid` | 按 PMID 取单条记录 | 诊断 |

## 语法 + 示例

```bash
# enrich —— 给已有论文补上 MeSH 主题词 + publication_types + PMC URL
# 输入：任何含 UnifiedPaperEntity 字典的 JSON 列表；没有 .pmid 的论文会被静默跳过
python3 -m scripts.pubmed_helper enrich \
    --input-file ./paper-search-results/<id>/raw/openalex.json \
    --output-file ./paper-search-results/<id>/raw/openalex_enriched.json
# 单次批量 efetch —— 对整个符合条件的 pmid 列表只发一次网络请求（NCBI 每次接受约 200 个 id）。

# search-mesh —— Audit 档按 MeSH 描述词的独立检索
python3 -m scripts.pubmed_helper search-mesh "Diabetes Mellitus, Type 2" \
    --year-min 2020 \
    --limit 25 \
    > ./paper-search-results/<id>/raw/pubmed.json
# 多词 MeSH 主题词要完全按 NCBI MeSH 浏览器里的写法加引号。
# 结束年份：加 --year-max YYYY（含端点）。search-mesh 和 search 都接受它。

# 带出版类型过滤的 search-mesh（PRISMA 的 RCT 纳入）
python3 -m scripts.pubmed_helper search-mesh "Irritable Bowel Syndrome" \
    --year-min 2010 \
    --pub-type "Randomized Controlled Trial" \
    --pub-type "Meta-Analysis" \
    --limit 50

# search —— 通用关键词（明确 MeSH 未知时的 Audit 兜底）
python3 -m scripts.pubmed_helper search "metformin gut microbiome" --year-min 2020 --limit 30
# 对比 search-mesh 的召回：同一检索式走关键词能多覆盖约 70% 的论文；
# 精确度相应下降。除非你需要召回，否则用 MeSH。

# by-pmid —— 单条记录的诊断 / 交叉引用
python3 -m scripts.pubmed_helper by-pmid 33301246
```

## 输出 JSON 格式

`enrich` 输出：增强后的 UnifiedPaperEntity 字典，含以下新增 / 更新的字段：

| 字段 | 来源 |
|---|---|
| `mesh_terms: List[str]` | PubMed MedlineCitation.MeshHeadingList.DescriptorName（权威来源） |
| `publication_types: List[str]` | 例如 `["Journal Article", "Randomized Controlled Trial", "Multicenter Study"]` |
| `pmcid: str` | "PMC9876543"（仅在缺失时填充） |
| `pmc_url: str` | "https://www.ncbi.nlm.nih.gov/pmc/articles/PMC9876543/" |
| `abstract: str` | 仅在缺失时填充 —— OA 重建值优先；PubMed = 兜底 |
| `sources: List[str]` | 追加 `"pubmed"` |

`search-mesh` / `search` / `by-pmid` 输出：解析后的字典（不是完整的 UnifiedPaperEntity），字段为 `{pmid, doi, pmcid, pmc_url, title, abstract, year, mesh_terms, publication_types}`。主 Agent 在 STEP 5 联邦合并阶段再把它物化成 UnifiedPaperEntity。

## MeSH 与关键词的精确度-召回（实证，22_pubmed_research）

| 检索式 | 关键词 | MeSH |
|---|---|---|
| `metformin` | 37,148 条记录 | 21,000 条记录（`"Metformin"[MeSH Terms]`） |
| `working memory` | 约 78,000 | 38,500（`"Memory, Short-Term"[MeSH]`） |
| `prospect theory` | 约 5,400 | 不适用（无对应 MeSH 词）—— 只能用关键词 |

MeSH 收窄到精确概念；关键词能覆盖同义词 + 相邻主题。Cochrane / PRISMA 流程要求主检索用 MeSH；关键词只用于敏感性分析。

## Publication Type 过滤取值

重复 `--pub-type "X"` 表示 OR。常见类型（区分大小写，必须与 NCBI 完全一致）：

| 过滤值 | 用途 |
|---|---|
| `Randomized Controlled Trial` | RCT 纳入标准 |
| `Meta-Analysis` | SR/MA 证据金字塔顶端 |
| `Systematic Review` | SR/MA 证据金字塔顶端 |
| `Clinical Trial` | 比 RCT 更宽（含 Phase I/II/III） |
| `Review` | 叙述性综述 + 范围综述 |
| `Multicenter Study` | 跨中心证据 |
| `Comparative Study` | 以对照为基础的设计 |
| `Practice Guideline` | 临床实践指南 |
| `Observational Study` | 队列 / 病例对照 |

## 实证告警（22_pubmed_research、24_v2_l2_booster_test、25_round2_synthesis）

- **相对 OpenAlex 没有独立召回** —— 测试集中 30/30 篇"PubMed 独有"的论文在全球范围内本来就已在 OpenAlex 中。以 enrich 为主模式；只有 PRISMA 流程要求显式 MeSH 检索时才用 search-mesh。
- **NCBI key 把 3 → 10 req/s**，但**不是必需** —— 匿名也能用，只是更慢。辅助脚本用 5 req/s（`time.sleep(0.2)`）作为有 key 时的安全速率。
- **`ncbi_email` 是必填项**，按 NCBI API ToS，即使没有 key 也要。没有它：429 / 被封。见 `setup.md`。
- **批量 efetch 无论数据量多大都只发一次网络请求**（单次逗号分隔列表最多约 200 个 PMID）。辅助脚本正是利用这一点 —— `enrich_with_mesh` 对整个符合条件的集合只发 1 次请求。不要自己循环。
- **MeSH 主题词总是被覆盖**为 PubMed 的权威数据（相对于 OA 兜底值而言）。PMCID / PMC URL / abstract 只在缺失时填充（已有 OA 值时 OA 优先）。
- **撤下（takedown）的论文没有摘要** —— 部分 2020 年后的论文摘要被撤下；辅助脚本返回空字符串，OA 的原值保持不变。

## 中文检索式处理（跨语言）

PubMed 的 MeSH 是一套英文受控词表 —— 中文关键词要走 PubMed 的翻译层，精确度一般；而 `search-mesh` 要求匹配**规范的英文 MeSH 描述词**。用户检索式是中文时，主 Agent 必须先把检索概念翻译成官方 MeSH 词（到 https://meshb.nlm.nih.gov/ 查询），再交给本辅助脚本。原始中文检索式作为报告元数据保留，不作为检索参数。

```bash
# 用户："青少年焦虑认知行为治疗"
# 把每个概念映射到 MeSH 描述词；不要把中文传给 PubMed。
# 青少年 → "Adolescent" (MeSH)
# 焦虑   → "Anxiety Disorders" (MeSH)
# 认知行为治疗 → "Cognitive Behavioral Therapy" (MeSH)
python3 -m scripts.pubmed_helper search-mesh "Cognitive Behavioral Therapy" \
    --limit 25 \
    > ./paper-search-results/<id>/raw/pubmed.json
# 需要时与 PubMed 的 MeSH AND 叠加组合使用 —— 但每个词都必须是精确的描述词。

# 反模式（行不通）：search-mesh "认知行为治疗" → 返回约 0 条命中。
```

在 `enrich` 模式（主要用法）下，辅助脚本查的是已经挂在 OA 论文上的 PMID —— 那里不需要翻译检索式。只有在 Audit 档跑 `search-mesh` / `search` 时，翻译才重要。
