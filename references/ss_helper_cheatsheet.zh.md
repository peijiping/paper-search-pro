# ss_helper CLI 速查表

由 `SKILL.md` STEP 10 引用（对 rcs ≥ 6 的 top-N 论文做 L3 丰富）。

**角色**：只做丰富、不做检索的 L3 —— Semantic Scholar **不是**独立检索源。它补充 `influentialCitationCount`（SS 独有、OpenAlex 没有任何对应指标的信号）、`tldr`（仅用于 HTML 展示）、OA 重建为空时的摘要兜底，以及跨源被引数校验。

**入口**：`python3 -m scripts.ss_helper --input-file <kg_classified.json> --mode <enrich|validate> [--min-rcs N] [--output-file <path>]`。
**初始化来源**：`init(load_config())` —— 使用 config 中的 `semantic_scholar_api_key`。

## 两种模式

| `--mode` | 功能 | 输出 |
|---|---|---|
| `enrich`（默认） | 批量补充影响力被引数 + 摘要 + tldr | 整个 KG，其中被丰富的字段就地写入 |
| `validate` | 跨源 citation_count 差异 > 30% 的冲突 | 冲突列表 `[{paper_id, title, oa_count, ss_count, delta_pct}]` |

`--min-rcs N` 把任一模式限制在 `rcs` 为整数且 ≥ N 的论文上；不给则每篇论文都会被送出。

## 语法 + 示例

```bash
# STEP 10: 就地丰富 KG 中 rcs >= 6 的论文
PYTHONPATH=$PSP_HOME python3 -m scripts.ss_helper \
    --input-file "$SEARCH_DIR/kg_classified.json" \
    --mode enrich --min-rcs 6 \
    --output-file "$SEARCH_DIR/kg_classified.json"

# validate: 检测被引冲突（Audit 档的数据质量检查）；绝不写回 KG
PYTHONPATH=$PSP_HOME python3 -m scripts.ss_helper \
    --input-file "$SEARCH_DIR/kg_classified.json" \
    --mode validate --min-rcs 6 \
    > "$SEARCH_DIR/validation_conflicts.json"
# 列出 |OA - SS| / OA > 30% 的论文 —— 通常是 arXiv 兜底 DOI 造成的少计，
# 或 2000 年前论文在 SS 上的覆盖缺口
```

**enrich 在每条 KG 记录里改了什么**（只针对 SS 返回的论文；其余一切 —— `rcs`、`rcs_reasoning`、`authors`、`journal_rank`、未知键、键顺序 —— 保持原样）：

- `influential_citation_count`、`tldr` —— 来自 SS，直接赋值
- `abstract`、`ss_paper_id`、`venue`、`issn`、`issns` —— 仅在该记录原本没有时才填充
- `sources` —— 追加 `"semantic_scholar"`

JSON **列表**形式的论文字典仍然可以接受输入；它返回的每篇论文只有 11 个字段（`doi, arxiv_id, openalex_id, ss_paper_id, title, abstract, year, citation_count, influential_citation_count, tldr, sources`），因此不能把它写回 KG。

## stderr 那一行

每次 enrich 运行都以 stderr 上的一行结束：

```
[paper-search-pro] Semantic Scholar enrichment: 52/58 papers enriched (6 not found).
[paper-search-pro] Semantic Scholar enrichment: 0/58 papers enriched (HTTP 403, key rejected).
[paper-search-pro] Semantic Scholar enrichment: 0/58 papers enriched (HTTP 429, rate limited).
```

括号里的原因表示 Semantic Scholar 拒绝了请求或调用失败；辅助脚本已经做完了它该做的重试（见下文），未丰富的论文保持原样，退出码为 0。数量不足时把这行转述给用户 —— 见 `error_handling.md` E3。key 被拒时还会额外打印一行，提示用户续期 `semantic_scholar_api_key`。validate 只在失败时打印一行（`Semantic Scholar citation check failed (…); the conflict list is incomplete.`）。

## 内部函数（Python API）

| 函数 | 用途 |
|---|---|
| `enrich_with_metadata(papers)` | 批量丰富（每 500 个 DOI 发一次 `/paper/batch` 请求）。就地修改并返回；打印 stderr 那一行。 |
| `abstract_fallback(paper)` | 单篇论文的摘要重试；优先返回 SS 摘要，其次 `tldr.text`，都没有则返回 None。 |
| `cross_validate_citation(papers)` | 返回差异 > 30% 的冲突列表。 |

## 被丰富的输出 JSON 字段

- `influential_citation_count: int` —— SS 独有的信号（用来排出"最具影响力"的 top 论文，而不只是被引最多的）
- `ss_paper_id: str` —— SS 的十六进制 paperId（用于交叉引用排错）
- `abstract: str` —— 仅当 OpenAlex 留下的值为 None 时才填充
- `tldr: str` —— 2–3 句自动生成的摘要（**仅用于 HTML 展示**；**不要**喂给 AI 分类器）
- `venue`、`issn`、`issns` —— 来自 SS `publicationVenue`（含它所有的 ISSN），仅在缺失时填充；期刊分区关联会逐个尝试所有 ISSN
- `sources: [...]` —— 追加 `"semantic_scholar"`

## 实证告警（24_v1_l3_enrichment_test、25_round2_synthesis）

- **硬性 1 RPS 限流** —— SS 没有付费档级；API key 只买到"认证形态"，买不到吞吐量。`/paper/batch` **就是**一次 HTTP 请求（2026-05-21 实测：2 个 DOI 用时 528ms），所以辅助脚本每次请求最多送出 500 个 DOI。
- **失败次数有上限** —— 批量请求遇到 429 时总共最多等待约 4 分钟（遵守 `Retry-After`，每次 ≤ 60 秒）；单篇请求最多重试 3 次（每次 ≤ 20 秒）；key 被拒时会去掉 key 重试一次，之后的流程都不带 key。若批量请求因自身原因失败（网络错误、HTTP 400/5xx），辅助脚本回退为逐篇请求，调用之间 `time.sleep(1.1)`；若 SS 直接拒绝客户端本身（即使不带 key 也 403，或重试后仍是 429），就到此为止，不再逐篇重复触发拒绝。
- **arXiv DOI 百分之百返回 404** —— `_doi_for_ss()` 会把它们过滤掉（任何含 `arxiv` 的 DOI）。根本不要尝试；SS 对 arXiv 的覆盖只走 paperId，不走 DOI。
- **`tldr` 仅用于 HTML 展示**，**不能**喂给 RCS 分类器 —— 依据用户指令（22_ss_research §4.3、25_round2 §4）。TLDR 是自动生成的，可能误导分类器；摘要才是权威来源。
- 出版商撤稿时 SS 有时会返回**空摘要** —— 按假值处理。必要时通过 STEP 4 回退到 PubMed（医学类）。
- **被引冲突 > 30% 的阈值是实证选定的** —— SA-V1 24_v1_l3_enrichment_test §3.1 显示真实数据中有 7/20 篇超过 30%；这个阈值能抓到 arXiv 兜底造成的伪影和老论文的覆盖缺口，同时不会把 SS 正常的少计（通常 20–50%）误报成冲突。

## 实例演示

`kg_classified.json` 之前的样子（其余字段省略）：

```json
{
  "doi:10.1162/neco.1997.9.8.1735": {"doi": "10.1162/neco.1997.9.8.1735", "title": "LSTM", "rcs": 9, "sources": ["openalex"]},
  "doi:10.48550/arxiv.1706.03762": {"doi": "10.48550/arxiv.1706.03762", "title": "Attention Is All You Need", "rcs": 8, "sources": ["openalex"]},
  "doi:10.1038/nature14539": {"doi": "10.1038/nature14539", "title": "Deep Learning Review", "rcs": 4, "sources": ["openalex"]}
}
```

跑上面那条 STEP 10 命令（`--min-rcs 6`）。之后：

```jsonc
{
  "doi:10.1162/neco.1997.9.8.1735": {
    "doi": "10.1162/neco.1997.9.8.1735", "title": "LSTM", "rcs": 9,
    "sources": ["openalex", "semantic_scholar"],
    "influential_citation_count": 10359,   // SS unique signal — 16% ratio = real-method usage
    "tldr": "A long short-term memory (LSTM) network can learn...",
    "ss_paper_id": "44d011f..."
  },
  // arXiv DOI: skipped, record unchanged
  "doi:10.48550/arxiv.1706.03762": {"doi": "10.48550/arxiv.1706.03762", "title": "Attention Is All You Need", "rcs": 8, "sources": ["openalex"]},
  // rcs 4 < 6: not sent, record unchanged
  "doi:10.1038/nature14539": {"doi": "10.1038/nature14539", "title": "Deep Learning Review", "rcs": 4, "sources": ["openalex"]}
}
```

stderr：`[paper-search-pro] Semantic Scholar enrichment: 1/2 papers enriched (1 without a usable DOI).`

影响力占比才是判别依据：BERT 19.5%、Adam 15.9%、LSTM 16%、GAN 14.5%、Attention 11.2%（真正被采用的方法）vs. LeCun DL Review 3.8%（被大量引用，但并没有被"使用"）。这是唯一值得为之付出延迟的 L3 信号。

## 中文检索式处理（跨语言）

ss_helper **只做丰富**（基于 DOI 查询）—— 没有检索式需要翻译。它作用于 OpenAlex + PubMed 已经收集好的 DOI/标题集合，所以只要上游给出了有效 DOI，中文论文也会被正确丰富；没有 DOI 的记录（例如 NSSD）保持不变，并被计入 "without a usable DOI"。

无论用户的检索式是什么语言，命令都相同（就是上面那条 STEP 10 命令）。

注意：Semantic Scholar 的 `influential_citation_count` 算法主要在英文引用模式上训练。对只发中文期刊的论文，这个信号不太可靠（即便是被引很高的区域性论文，也常常是 `null` 或数值偏低）。把 `influential_citation_count == null` 当作"无信号"，而不是"影响力低"。
