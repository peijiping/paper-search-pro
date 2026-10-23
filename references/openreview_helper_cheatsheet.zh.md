# openreview_helper CLI 速查表

**角色**：与 arXiv 并列的 AI 主题增强源。它的价值在于**录用状态**，而不是额外召回。OpenAlex 和 Semantic Scholar 把当年的 ICLR / ICML / COLM 论文都列成 arXiv 预印本。OpenReview 则是这些会议用的评审平台，所以它知道同一篇论文其实是 "ICLR 2026 Oral"。它返回的论文大多已经以同样的标题出现在 OpenAlex 或 arXiv 里。解析器按标题把它们合并，录用会议随即替换掉 "arXiv"。

**仅用于 AI / ML 主题。** OpenReview 承载 ML、AI、CV、NLP 和机器人领域的会议（ICLR、NeurIPS、ICML、COLM、CoRL、TMLR、经由 ARR 的 ACL、CVPR、AAAI、KDD）。心理学、医学或社科类检索式只会返回 OpenAlex / PubMed 已经有的论文的导入副本，而白名单会把它们全部丢掉。调用时机（路由）写在 `source_routing.md` 里。

**入口**：`python3 -m scripts.openreview_helper --search "<q>" [--n 30] [--year-min Y] [--year-max Y] [--max-pages 8] [--output-file F]`。
不需要 key，也不需要登录。stdout 是一个 JSON 列表（`UnifiedPaperEntity` 字典，与 `yiigle_helper` 结构相同）。署名行和一行诊断信息走 stderr。

## 语法 + 示例

```bash
# 某个 AI 主题上近期被录用的论文（典型：3 页，约 7 秒）
PYTHONPATH=$PSP_HOME python3 -m scripts.openreview_helper \
    --search "diffusion language model" --n 30 --year-min 2025 \
    --output-file "$SEARCH_DIR/raw/openreview.json"

# stderr 输出示例：
#   Data: OpenReview (openreview.net), metadata CC0 | kept only papers accepted by a main conference track or TMLR (...)
#   openreview: 3 page(s), 75 records seen, 22 accepted, 20 kept; stopped: n_reached
```

```python
from scripts import openreview_helper
papers = openreview_helper.search("diffusion language model", n=30, year_min=2025)
openreview_helper.last_search_stats   # {'pages_fetched', 'records_seen', 'accepted_seen', 'kept', 'stopped', 'query'}
```

`search()` **永不抛异常**。任何失败它都返回已经收集到的内容，可能是 `[]`。读 `last_search_stats["stopped"]` 以区分"OpenReview 把我们挡了"（`challenge`、`http_403`、`http_429`、`timeout`、`network_error`、`bad_json`、`bad_shape`）和"这个主题在这里没有录用论文"（`end_of_results`、`max_pages`、`n_reached`）。在 PRISMA-S 日志里写明属于哪一种。

## 保留什么：录用白名单

约 83% 的检索命中**不是**已录用论文。它们是 workshop 论文、被拒论文（venue 字符串字面上就是 "Submitted to ICLR 2026"）、已撤稿和评审中的论文（包括匿名的 ACL ARR 草稿），以及别处发表的论文导入的档案（`dblp.org/…`、`OpenReview.net/Public_Article`、`OpenReview.net/Archive`）。只有当 `content.venueid` **完全**等于下列值时，代码才保留该条记录：

| venueid | 示例 | 保留为 |
|---|---|---|
| `<org>[/<more>]/<YYYY>/Conference` | `ICLR.cc/2026/Conference`、`colmweb.org/COLM/2026/Conference`、`EMNLP/2023/Conference` | 主会场录用的论文 |
| `TMLR` | `TMLR`（"Accepted by TMLR"） | 录用的期刊论文 |
| `NeurIPS.cc/<YYYY>/Datasets_and_Benchmarks_Track`、`NeurIPS.cc/<YYYY>/Track/Datasets_and_Benchmarks` | — | NeurIPS Datasets & Benchmarks 赛道（同行评审，收录于论文集）；标注为 "NeurIPS 2025 Datasets & Benchmarks Poster" |

其余全部丢弃，包括 `…/Rejected_Submission`、`…/Withdrawn_Submission`、`…/Submission`、`…/Workshop/…`、`TMLR/Rejected`、`TMLR/Under_Review`、`TMLR/Decision_Pending` 以及所有导入记录。这是有意为之的白名单，因为每个会议对"未录用"状态的命名方式各不相同。判断状态从不使用 `venue` 字符串：它是自由文本，导入记录的读起来和已录用记录一模一样（"NeurIPS 2024"）。

## 输出字段

```jsonc
{
  "source_native_id": "openreview:g88nt4ieTG",       // forum id; also the paper_id (no DOI / arXiv id on venue papers)
  "title": "Diffusion Language Model Knows the Answer Before It Decodes",
  "abstract": "...",
  "authors": [{"name": "Pengxiang Li"}, ...],
  "year": 2026,                                     // conference: from venueid; TMLR: pdate, else cdate
  "venue": "ICLR 2026 Oral",                        // never the bare word "OpenReview"
  "type": "article",
  "keywords": ["diffusion language model", "discrete"],
  "tldr": "...",                                    // content.TLDR when the authors gave one
  "pdf_url": "https://openreview.net/pdf?id=g88nt4ieTG",   // only when the note has a PDF
  "oa_locations": ["https://openreview.net/forum?id=g88nt4ieTG"],  // reviews and discussion live here
  "sources": ["openreview"],
  "discovery_path": null                            // left unset on purpose
}
```

**会议标签** = `<Name> <YYYY>[ <Tier>]`。
- **Name**：当 venueid 的年份紧跟其后时，取 venue 字符串的第一个词（"NeurIPS 2025 poster" 得到 "NeurIPS"）。否则取 venueid 中紧邻年份之前的那一段，并去掉域名后缀（`ICLR.cc` 得到 "ICLR"，`colmweb.org/COLM/…` 得到 "COLM"）。
- **Year**：始终取 venueid 中的年份。
- **Tier**：只认完整单词，大小写归一。`oral` → Oral，`spotlight` 或 ICML 的 `spotlightposter` → Spotlight，`poster` → Poster，`findings` → Findings。Findings 优先于其他，所以 EMNLP 的 Findings 论文绝不会被读成主会场论文。
- **没有档级词**：ICML 的 "regular"、ICLR 2023 的 "notable top 5%"、"EMNLP 2023 Main" 和 "COLM 2026" 都不带档级（"ICML 2026"）。
- **复合词**："KSMI 2026 ShortOralPoster" 是一个词，所以它**不会**变成 "Oral"。

## 已知限制

- **只有检索端点能用。** `GET /notes?content.venueid=…`（列出整个会议）对匿名客户端一律返回 403 `ChallengeRequiredError`。辅助脚本只用 `GET https://api2.openreview.net/notes/search?…&limit=25&offset=k`。自 2025 年底以来 OpenReview 一直在收紧自动化访问，检索端点也可能撞上同一堵墙。届时辅助脚本返回 `[]`，并带上 `stopped: challenge`。把这个源当作可选源；任何环节都不得依赖它。
- **礼貌策略**：每页 25 条，页间隔至少 1 秒，超时 20 秒，最多抓 `--max-pages`（默认 8）页。OpenReview 没有公布限流。每次运行保持一到两次调用，绝不并行。
- **`count` 永远是 10000**（是个上限）。它无法估算命中总数，所以不要拿它去算覆盖率。
- **相关性基于关键词，而且噪声很大。** 前几条命中可能离题（检索 "diffusion language model" 出来的却是一篇罗曼什语的语言识别论文）。下游的 RCS 打分会处理这件事；不要把排名当相关性。
- 会议论文**没有 DOI，也没有 arXiv id**。与 OpenAlex / arXiv 的合并只按归一化标题进行，而 arXiv 版与 camera-ready 版标题有出入时就合不上。
- **白名单放行每个会议的主会场**，小的也不例外（例如 KSMI 2026，一个韩国的音乐信息学会议）。会议的分量来自分级层，而不是这个辅助脚本。
- **`…/Conference` 之外的其他"已录用论文"赛道会被丢弃**（例如 `ACM.org/AIWare/2026/Data_and_Benchmark_Track`、`DMLR/Special_Track`）。NeurIPS Datasets & Benchmarks 是唯一的例外，2026-10-01 起放行；它的被拒论文带有同样的 id 再加 `/Rejected_Submission`，精确匹配会把这部分挡在外面。
- **能看到哪些被拒论文取决于会议。** ICLR 会公开每一篇被拒和撤稿的论文，而 ICML、COLM 和 KDD 只公开已录用的。所以被拒的命中基本来自 ICLR，而白名单反正会把它们全部丢掉。
- **仅英文**：调用前先把中文检索式翻译成英文术语，与 arXiv 一样。
