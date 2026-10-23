# 输出目录结构与文件约定

由 `SKILL.md` STEP 5 起引用——每个辅助脚本都写入这个结构。主 Agent 在运行前必须先创建 `<search_id>/raw/`、`<search_id>/batches/`、`<search_id>/classifications/`。

> ## ⚠️ 输出路径约定
>
> **本 Skill 中所有 `./paper-search-results/<search_id>/` 路径都相对于用户当前工作目录（PWD），而不是 Skill 安装目录。**
>
> 这意味着：
> - 如果用户从 `/Users/alice/projects/thesis/` 调用本 Skill，输出会落在 `/Users/alice/projects/thesis/paper-search-results/<id>/`
> - Skill 目录（`$PSP_HOME`——在 Claude Code 下解析为 `~/.claude/skills/paper-search-pro`，在 Codex 下为 `~/.codex/skills/paper-search-pro`，等等）是**只读**资源包，绝不能存放检索输出。
> - 运行辅助脚本前绝不要 `cd $PSP_HOME`——那会把 `./` 重新锚定到 Skill 目录，污染安装目录。
> - 改为在用户 cwd 下使用 `PYTHONPATH=$PSP_HOME python3 -m scripts.<helper> ...`。
>
> `$PSP_HOME` 在各 Agent 间如何解析（环境变量 → 回退链）见 `SKILL.md` STEP 0；保持 cwd 的 `PYTHONPATH=` 调用写法见 `references/setup.md`。

## search_id 命名规则

格式：`YYYYMMDD_HHMMSS_<slug>`，其中 `<slug>` 取检索式的前 3 个内容词，转小写并用连字符连接，长度 ≤30。

| 示例检索式 | search_id |
|---|---|
| `prospect theory decision making` | `20260521_143052_prospect-theory-decision` |
| `工作记忆训练老年` | `20260521_143052_working-memory-training`（中文音译） |
| `dietary interventions IBS` | `20260521_143052_dietary-interventions-ibs` |

主 Agent 在开始时生成一次，检索过程中绝不更改。

## 完整目录树

```
./paper-search-results/<search_id>/
├── report.html                  # 主交付物，浏览器直接打开
├── report.md                    # Markdown 版本，供文献管理器 / pandoc 使用
├── papers.csv                   # 表格导出——对 Excel / pandas 友好
├── bibtex.bib                   # BibTeX 导入（Zotero / Mendeley / LaTeX）
├── ris.ris                      # RIS——EndNote / Papers / 旧版管理器
├── papers.json                  # 完整结构化论文列表（UnifiedPaperEntity[]）
├── kg_classified.json           # 内部 KG：papers + RCS + flags + sources
├── kg.json                      # 分级前的 KG（联邦去重输出）
├── summary.md                   # 主 Agent 撰写的执行摘要
├── metadata.json                # original_user_query + search_topic + display_title + 运行元数据
├── execution_log.json           # PRISMA-S 16 项 + 停止原因 + 错误
├── curve.json                   # 发现曲线 + 饱和度估计 + CI
├── concept_model.json           # 仅 STEP 11.5——平台无关的概念模型（layer-1 LLM 输出）
├── search_strategies.md         # 仅 STEP 11.5——可直接粘贴的各平台检索式（WOS / Embase / CNKI / …）
├── search_strategies.json       # 仅 STEP 11.5——结构化检索式（汇入 report_data + PRISMA-S 条目 8）
├── raw/                         # 仅检索输出，一次检索一个文件（STEP 7 读取它）
│   ├── openalex.json
│   ├── pubmed.json              # 仅当启用 PubMed
│   ├── arxiv.json               # 仅当启用 arXiv
│   └── citations_01.json …      # 每个引文网络种子一个（STEP 9）
├── batches/                     # 分类器输入分片（每片 10 篇）
│   ├── batch_001.jsonl
│   ├── batch_002.jsonl
│   └── ...
├── classifications/             # SubAgent 输出（每批一个文件）
│   ├── batch_001_result.json
│   ├── batch_002_result.json
│   └── ...
└── report_data.json             # data_materialization 的输出，中间产物
```

## 文件产出者 + 用途 + 结构

| 文件 | 产出者 | 用途 | 结构（顶层） |
|---|---|---|---|
| `report.html` | `html_renderer_webartifacts.py` | 面向用户的主交付物 | Shadcn React 包（原始体积约 ~990 KB；250 篇论文水合后约 1.7 MB） |
| `report.md` | `md_report.py` | 便于文献管理器 / pandoc 使用的 markdown | YAML frontmatter + 章节 + 参考文献 |
| `papers.csv` | `generate_exports.py` | 便于 Excel / pandas 复核 | `doi,title,authors,year,venue,citation_count,influential,rcs,abstract,...` |
| `bibtex.bib` | `generate_exports.py` | LaTeX / Zotero / Mendeley | 以 DOI/arxiv 为键的 BibTeX `@article{` 条目 |
| `ris.ris` | `generate_exports.py` | EndNote / Papers | RIS 标签格式 |
| `papers.json` | `generate_exports.py` | 供程序化复用 | `UnifiedPaperEntity[]` |
| `kg_classified.json` | `rcs_parser.py` (STEP 6) | 分级后的 KG，含 rcs/reasoning/flag | `{<canonical_key>: UnifiedPaperEntity}` dict |
| `kg.json` | `federated_kg_resolver.py` (STEP 5) | 去重后的多源合并结果 | `{<canonical_key>: UnifiedPaperEntity}` dict |
| `summary.md` | 主 Agent (STEP 11) | 以主 Agent 语气写的执行摘要 | 约 300 字的纯 markdown |
| `execution_log.json` | `prisma_s_logger.py` (STEP 13) | PRISMA-S 符合性 / 审计轨迹 | `{prisma_s: {1-16 items}, discovery_curve_snapshots, agent_invocations, errors, stop_reason, search_id, user_query, tier, generated_at}` |
| `curve.json` | `discovery_curve.py` (STEP 7) | 覆盖率估计 | `{papers_evaluated, highly_relevant_count, coverage_estimate, ci_lower, ci_upper, method, relevance_band, occasions, n_total_estimate, …}` |
| `raw/openalex.json` | `openalex_helper double-sort` 或 `deep` (STEP 3) | OpenAlex 检索原始输出 | `UnifiedPaperEntity[]` |
| `raw/pubmed.json` | `pubmed_helper search-mesh` 或 `enrich` (STEP 4) | PubMed 原始 / 丰富化输出 | 解析后的 dict[] 或 UnifiedPaperEntity[] |
| `raw/arxiv.json` | `arxiv_helper freshness` (STEP 4) | arXiv 新鲜度检索原始输出 | `UnifiedPaperEntity[]` |
| `raw/citations_NN.json` | `openalex_helper citation-network` (STEP 9) | 引文扩展，每个种子一个文件 | `{references, cited_by}` |
| `batches/batch_NNN.jsonl` | 主 Agent (STEP 6) | 分类器输入分片 | 每行一篇论文：`{paper_id, title, abstract, year, venue}` |
| `classifications/batch_NNN_result.json` | Inline SubAgent (STEP 6) | RCS 分级输出 | `[{paper_id, rcs, reasoning, flag}]` |
| `metadata.json` | `data_materialization.py` (STEP 12) | 把审计身份与视觉身份分开 | `{query (legacy), original_user_query, search_topic, display_title, language, ...run metadata}` |
| `report_data.json` | `data_materialization.py` (STEP 12) | HTML/MD 渲染器的中间产物 | `{chart_data, paper_list, metadata, prisma_log}` |

## HTML 渲染器

- **只有一个渲染器，始终如此**：`webartifacts`（Shadcn React 包）。没有按体积触发的
  降级——早期的 "jinja2 lightweight fallback" 已被移除，因为它带来不一致的体验
  （同一个 Skill 会因为数据量不同产出两份视觉上不同的报告）。所有现代浏览器
  都能干净地处理 10+ MB 的自包含 HTML；原始包约 990 KB，典型 250 篇
  报告的水合后体积约 1.7 MB。
- 没有逐次渲染的配置开关——`html_renderer_webartifacts.py` 是唯一路径。
- HTML 与 Markdown 的 H1 只读取 `metadata.display_title`。旧的
  `metadata.query` 按原样视为用户请求，绝不提升为标题。缺标题时回退到
  `Literature Search Report` / `文献检索报告`。

## 清理约定

中间的 `batches/` 和 `classifications/` 保留用于调试。成功后如需清除：

```bash
rm -rf ./paper-search-results/<id>/batches/ ./paper-search-results/<id>/classifications/
# 保留 raw/（取证用）+ 所有顶层文件（交付物 + 审计）。
```

不要自动删除——用户有时想在不重新调 API 的前提下，用改进过的评分准则重新分级。

## 文件 IPC 完整性

主 Agent 采用单写入者模式（不并发写同一个文件）。并行 SubAgent 分级批次时，每个只写自己的 `batch_NNN_result.json`。合并步骤（`rcs_parser.py`）是串行的。

如果某个 SubAgent 在写入中途崩溃，文件可能是残缺的 JSON。`rcs_parser.py` 有 5 层兜底（正则解析 → 抢救能救回的条目 → 用 `flag=parse_failed_uncertain` 标记无法解析的条目）。见 `error_handling.md` §E7。
