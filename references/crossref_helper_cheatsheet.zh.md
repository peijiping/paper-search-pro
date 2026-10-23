# crossref_helper CLI 速查表

由 `SKILL.md` STEP 10 引用（L3 丰富）—— 资助方、许可协议、参考文献、临床试验编号。

**角色**：只做丰富、不做检索的 L3 —— 绝不是独立检索源。CrossRef 掌握权威的资助方 DOI（OpenAlex 的 `grants` 字段通常为空）、许可协议的内容版本 + 延迟天数（OpenAlex 两者都没有）、对 NEJM / 较早论文更完整的 `reference[]`，以及临床试验编号（即使是 RCT 也大多为空 —— 主要来源看 PubMed）。

**入口**：`python3 -m scripts.crossref_helper --input-file <kg_classified.json> --mode <funder|license|refs|clinical|all> [--min-rcs N] [--output-file <path>]`。
**不需要 key** —— 只要一个 `crossref_email` 用于 polite pool 的 User-Agent（10 req/s × 3 并发，而匿名是 5/s × 1 并发）。

## 五种模式

| `--mode` | 新增字段 | 说明 |
|---|---|---|
| `all`（默认，推荐） | funders + license + refs_count + clinical_trial_number | 每篇论文只抓取一次 |
| `funder` | `funders=[{name, doi, award[]}]` | 不需要其他字段时单独用 |
| `license` | `license=[{URL, content_version, delay_in_days}]` | 禁运期 / OA 时机 |
| `refs` | `referenced_works_count`（仅当 CR > OA 时） | OA 计数 ≥ 10 时按阈值跳过 |
| `clinical` | `clinical_trial_number` | 大多为空 —— 优先用 PubMed |

`--min-rcs N` 把丰富范围限制在 `rcs` 为整数且 ≥ N 的论文上；不给则每篇论文都会被抓取。

## 语法 + 示例

```bash
# STEP 10: 一次性就地丰富 rcs >= 6 的论文（每篇抓取一次，约 0.6 秒）
PYTHONPATH=$PSP_HOME python3 -m scripts.crossref_helper \
    --input-file "$SEARCH_DIR/kg_classified.json" \
    --mode all --min-rcs 6 \
    --output-file "$SEARCH_DIR/kg_classified.json"

# 单个字段（每种模式各自发起一次抓取 —— 慎用）；license / refs / clinical 同理
PYTHONPATH=$PSP_HOME python3 -m scripts.crossref_helper \
    --input-file "$SEARCH_DIR/kg_classified.json" \
    --mode funder --min-rcs 6 \
    --output-file "$SEARCH_DIR/kg_classified.json"
```

**每条 KG 记录里会变的东西**：只有 `funders`、`license`、`referenced_works_count`、`clinical_trial_number`（每项仅在 CrossRef 有值时才写入），以及 `sources` 里追加 `"crossref"`。`authors`、`rcs`、`rcs_reasoning`、`journal_rank`、未知键和键顺序保持原样；CrossRef 补不出任何东西的论文完全不动。

JSON **列表**形式的论文字典仍然可以接受输入，会返回完整的实体字典，但 `authors` 被清空 —— 绝不要把列表形式的输出写回 KG。

## 输出字段格式

```jsonc
{
  "funders": [
    {"name": "BioNTech and Pfizer", "doi": "10.13039/100004319", "award": ["BNT162-01"]}
  ],
  "license": [
    {
      "URL": "https://www.nejm.org/about-nejm/...",
      "content_version": "vor",            // tdm | vor | am | unspecified
      "delay_in_days": 0
    }
  ],
  "referenced_works_count": 13,            // bumped from OA's 8 only when CR strictly more
  "clinical_trial_number": "NCT04368728",  // mostly null on NEJM RCTs
  "sources": [..., "crossref"]             // marked iff any field enriched
}
```

## 实证告警（22_crossref_research、24_v1_l3_enrichment_test、25_round2_synthesis）

- **arXiv DOI 百分之百返回 404** —— `10.48550/arXiv.*` 和 `10.48550/arxiv.*` 两种写法都是；`_is_arxiv_doi()` 会预先过滤掉它们，不会发起网络请求。CrossRef 只收录已由出版商正式出版的文献。
- **HTTP 404 返回的是纯文本 "Resource not found." 而不是 JSON** —— `_fetch_doi()` 在调用 `.json()` 之前会检查 `Content-Type: application/json`。没有这道防护就会出现莫名的异常。
- **即使在 NEJM 的 RCT 上，`clinical-trial-number[]` 也大多为空** —— V1 测试里的 4 篇 NEJM 临床试验论文全部返回空值。该字段以 `pubmed_helper enrich` 为主来源；CrossRef 只作备份（搭资助方那次抓取的便车，不额外增加成本）。
- **对 NEJM / 较早论文，参考文献严格更完整** —— NEJM 2020 COVID：OA `referenced_works_count=8`，CrossRef `reference[]=13`。辅助脚本只在 CrossRef 严格更大时才覆盖；OA 计数已经 ≥ 10 的论文会跳过（默认阈值）。
- **polite pool 为 10 req/s × 3 并发** —— 辅助脚本实际用 10/s 串行（调用之间 `time.sleep(0.1)`）。20 篇 top-N 丰富约需 12 秒。不要在用户层尝试并行化；辅助脚本改由 `enrich_all` 批量处理。
- **许可协议的 `content-version` 就是禁运类型**：`tdm`（text-data-mining，文本与数据挖掘）/ `vor`（version of record，出版商正式版本）/ `am`（accepted manuscript，接受稿）/ `unspecified`（未指定）。`delay_in_days` 是从出版到 OA 开放的天数 —— 0 表示立即开放获取。

## 实例演示：NEJM COVID 2020 BNT162b2

KG 记录之前的样子（其余字段省略）：
```json
{"doi:10.1056/nejmoa2034577": {"doi": "10.1056/nejmoa2034577", "title": "Safety and Efficacy of the BNT162b2 mRNA Covid-19 Vaccine", "authors": [{"name": "F. P. Polack"}], "rcs": 9, "referenced_works_count": 8, "sources": ["openalex"]}}
```

跑上面那条 STEP 10 命令（`--mode all --min-rcs 6`）。之后：
```jsonc
{"doi:10.1056/nejmoa2034577": {
  "doi": "10.1056/nejmoa2034577",
  "title": "Safety and Efficacy of the BNT162b2 mRNA Covid-19 Vaccine",
  "authors": [{"name": "F. P. Polack"}],   // untouched
  "rcs": 9,                                 // untouched
  "referenced_works_count": 13,             // bumped from 8 — CR strictly more
  "sources": ["openalex", "crossref"],
  "funders": [
    {"name": "BioNTech and Pfizer", "doi": "10.13039/100004319", "award": []}
  ],
  "license": [
    {"URL": "https://www.nejm.org/about-nejm/permissions", "content_version": "vor", "delay_in_days": 0}
  ]
  // no clinical_trial_number: empty even on this RCT; use PubMed
}}
```

这是 CrossRef「物超所值」的典型案例 —— 资助方 DOI（10.13039/100004319）无法从 OpenAlex 的 `grants[]` 恢复（这篇论文在该字段为空），而 Audit 档报告里的资助来源偏倚分析必须有它。

## 中文检索式处理（跨语言）

crossref_helper **只做丰富**（基于 DOI 查询）—— 没有检索式需要翻译。它作用于上游流水线收集到的任何 DOI 集合，与原始检索式语言无关；命令就是上面那条 STEP 10 命令。

中国机构所属文献的注意事项：
- 资助方 DOI 的覆盖偏向北美与欧洲 —— 中国资助方（NSFC、MOST）也有，但更稀疏；即便是有充足经费的中文论文，`funders[]` 也经常为空。
- `license[].URL` 返回的是出版商页面；中国出版商（如高等教育出版社）的 `content_version` 元数据不如 NEJM/Springer 一致。
- `referenced_works_count` 与语言完全无关，始终准确 —— 这是 CrossRef 增强中最不受语言影响的一项。
