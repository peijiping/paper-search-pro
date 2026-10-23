# PRISMA-S 16 项清单

由 `SKILL.md` STEP 13 引用（`prisma_s_logger.py` 写 `execution_log.json`）。

PRISMA-S 是 PRISMA 2020 的检索报告扩展（Rethlefsen et al. 2021，*Systematic Reviews*）。每次 Audit 档运行都必须记录全部 16 项；较轻的档级记录能拿到的部分。下面的字段与 `scripts/prisma_s_logger.py::build_prisma_s_log()` 中的键一一对应。

**这不是要取代 PRISMA**——paper-search-pro 是系统综述（SR）准备阶段的辅助工具（Quick/Standard/Deep/Audit 档）。完整的 Cochrane 级系统综述仍然需要人工筛选、双人审阅、偏倚风险评分和 PROSPERO 注册。我们只记录能自动化的部分；其余由你补齐。

## 遵循范围

| 档级 | 自动填写的条目 | 用户填写（手动） |
|---|---|---|
| Quick / Standard | 1, 2, 8, 13, 14, 15, 16（16 项中的 7 项） | 3, 4, 5, 6, 7, 9, 10, 11, 12 —— 大多是"未执行" / "超出范围" |
| Deep | + 5（引文检索） | 同上 |
| Audit | + 9, 10, 11（过滤器 / 限定式 / 强制纳入） | 只剩 3, 4, 6（注册库 / 联网 / 作者） |

## 16 个条目

### 1. database_information
- **说明**：检索过的数据库列表 + 主源指定
- **自动填写**：`databases`（从全部 `paper.sources[]` 去重）、`primary`（用了 OpenAlex 时）、`note`
- **用户填写**：无 —— 完全自动

```jsonc
"1_database_information": {
  "databases": ["openalex", "semantic_scholar", "pubmed", "arxiv", "crossref"],
  "primary": "OpenAlex",
  "note": "OpenAlex polite pool + Semantic Scholar (supplement). Audit may add PubMed/arXiv."
}
```

### 2. multi_database_searching
- **说明**：是否使用了 ≥2 个数据库 + 理由
- **自动填写**：`performed`（≥2 个来源）、`rationale`（Bramer 2018 98.3% 召回率基准）
- **用户填写**：无

### 3. study_registries
- **说明**：试验 / 研究注册库（ClinicalTrials.gov / WHO ICTRP / Cochrane CENTRAL）
- **自动填写**：`queried=false`、`note`（默认超出范围）
- **用户填写**：如果你为 Audit 手动查了 Cochrane CENTRAL，把检索式 + 检索日期追加到这里

### 4. online_resources_browsing
- **说明**：独立浏览的会议论文集 / 预印本源
- **自动填写**：`performed=false`、`note`（超出范围）
- **用户填写**：如果你手动浏览了 NeurIPS 论文集或 arXiv 列表，在这里记一笔

### 5. citation_searching
- **说明**：前向 / 后向引文追溯
- **自动填写**：`performed`（当且仅当有论文的 `discovery_path` 以 `ref of` 或 `cites` 开头时为 true）、`method`（OpenAlex 双向）、`max_hops`、`seeds_count`
- **用户填写**：无 —— 完全自动

```jsonc
"5_citation_searching": {
  "performed": true,
  "method": "OpenAlex forward+backward citation chase up to configured max_hops.",
  "max_hops": 2,
  "seeds_count": 5
}
```

### 6. contacts
- **说明**：为获取灰色文献而联系的作者 / 主题专家
- **自动填写**：`performed=false`
- **用户填写**：如果你给作者发邮件索取未发表数据，记下联系了谁、何时联系

### 7. other_methods
- **说明**：手检、专家咨询、AI 辅助检索等
- **自动填写**：`performed=false`、`methods=[]`
- **用户填写**：如果你同时用了 Elicit / Undermind / ResearchRabbit，把它们列出来

### 8. full_search_strategies
- **说明**：完整布尔检索式，可跨数据库复现
- **自动填写**：来自 `state.query_plan` 的 `queries[]`、按来源区分的 `boolean_expressions[]` 变体
- **用户填写**：无 —— 主 Agent 在 STEP 1 的检索式规划里写好这些

```jsonc
"8_full_search_strategies": {
  "queries": [
    "prospect theory AND (loss aversion OR risk preference) AND decision making",
    "behavioral economics AND framing effect"
  ],
  "boolean_expressions": [
    {"text": "prospect theory ...", "openalex": "...", "semantic_scholar": "...", "type": "concept_block"}
  ],
  "note": "Strategy reproducible; same boolean expressions executed against each database."
}
```

### 9. limits_and_restrictions
- **说明**：施加的年份、语言、文献类型过滤器
- **自动填写**：来自检索计划的 `filters_applied[]`
- **用户填写**：如果你施加了 `language`、`publication_type`，就填上（目前它们是 null）

### 10. search_filters
- **说明**：经过验证的检索过滤器 / 限定式（例如 Cochrane RCT 过滤器）
- **自动填写**：`validated_filters_used=[]`
- **用户填写**：如果你用了 Cochrane Highly Sensitive Search Strategy，注明出处（例如 "Cochrane HSSS 2008 sensitivity-maximizing RCT filter"）

### 11. prior_work
- **说明**：凭先验知识强制纳入的论文（种子列表）
- **自动填写**：来自配置的 `force_includes[]`
- **用户填写**：如果你手动注入了事先已知的 DOI，把它们连同理由列出来

### 12. updates
- **说明**：为更新既有综述而做的增量检索
- **自动填写**：`incremental_search=false`
- **用户填写**：如果本次运行是在更新先前某篇系统综述，列出此前检索的结束日期

### 13. dates_of_searches
- **说明**：开始 / 结束时间戳 + 挂钟耗时
- **自动填写**：`search_started_at`、`search_ended_at`、`wall_clock_seconds`
- **用户填写**：无 —— 完全自动

```jsonc
"13_dates_of_searches": {
  "search_started_at": "2026-05-21T14:30:00Z",
  "search_ended_at":   "2026-05-21T14:42:15Z",
  "wall_clock_seconds": 735.4
}
```

### 14. total_records
- **说明**：检出的记录数、去重后、筛选后、纳入数
- **自动填写**：`records_screened`、`papers_in_kg`、`highly_relevant_count`（rcs ≥ 7）、`coverage_estimate`（来自发现曲线）
- **用户填写**：Audit 档的人工筛选 N —— paper-search-pro 不做标题 / 摘要的双人审阅

### 15. deduplication
- **说明**：去重方法
- **自动填写**：`performed=true`、`method`（DOI → arXiv ID → PMID → title-Jaro-Winkler）、`note`（单写入者 FederatedKG）
- **用户填写**：无

```jsonc
"15_deduplication": {
  "performed": true,
  "method": "FederatedKG 3-level dedup: DOI (Level 1) -> arXiv ID (Level 2) -> Jaro-Winkler title similarity >= 0.92 (Level 3). Main agent single-writer pattern prevents race conditions.",
  "note": "Provenance preserved per paper in sources[]."
}
```

### 16. record_management
- **说明**：使用的工具 / 格式 / 产出的文件
- **自动填写**：`tool="paper-search-pro/2.0"`、`format`、`outputs_produced[]`
- **用户填写**：无

```jsonc
"16_record_management": {
  "tool": "paper-search-pro/2.0",
  "format": "JSONL append-only checkpoint + materialized JSON outputs",
  "outputs_produced": ["report.html", "report.md", "papers.csv", "bibtex.bib", "ris.ris", "papers.json", "kg_classified.json", "summary.md", "execution_log.json"]
}
```

## 完整的 execution_log.json 结构

```jsonc
{
  "prisma_s": { "1_database_information": {...}, "2_multi_database_searching": {...}, ... "16_record_management": {...} },
  "discovery_curve_snapshots": [
    {"n_evaluated": 50, "n_highly_relevant": 12, "round": 1},
    {"n_evaluated": 100, "n_highly_relevant": 22, "round": 2}
  ],
  "agent_invocations": [],   // reserved for future use; reconstructed from CheckpointJSONL
  "errors": [
    {"code": "E3", "stage": "ss_enrich", "msg": "rate limit 429", "stop_reason": null}
  ],
  "stop_reason": "complete" | "budget_max_papers (200)" | "budget_max_wallclock" | ...,
  "search_id": "20260521_143052_prospect-theory-decision",
  "user_query": "prospect theory in decision making",
  "tier": "standard",
  "generated_at": "2026-05-21T14:42:15.234567"
}
```

## 用户向期刊投稿时

对于 Cochrane / JBI / Campbell 的系统综述，自动填写的字段覆盖了约 70% 的 PRISMA-S。剩下需要手动完成的 30%：条目 3（注册库）、6（联系作者）、10（经过验证的限定式）、14（筛选后的人工计数）。这些是研究团队的责任，不是检索工具的责任。

对于较轻的系统综述（快速综述 / 范围综述 / 叙述性综述），paper-search-pro 的自动日志 + 一页方法学叙述通常就够了。

## 读取日志

```bash
# 只打印 PRISMA-S 部分
jq '.prisma_s' ./paper-search-results/<id>/execution_log.json

# 获取停止原因 + 耗时
jq '{stop_reason, search_id, tier, prisma_s: .prisma_s["13_dates_of_searches"]}' \
   ./paper-search-results/<id>/execution_log.json

# 查看覆盖率估计
jq '.prisma_s["14_total_records"].coverage_estimate' ./paper-search-results/<id>/execution_log.json
```
