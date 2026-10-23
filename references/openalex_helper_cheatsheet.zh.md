# openalex_helper CLI 速查表

由 `SKILL.md` STEP 3（始终启用的 L1 检索）与 STEP 9（引文扩展）引用。

**入口**：`python3 -m scripts.openalex_helper <subcmd> [args]`（必须在 skill 根目录下运行）。
**初始化来源**：`init_pyalex(load_config())` —— 需要 `~/.paper-search-pro/config.yaml` 中的 `openalex_api_key`（自 2026-02-13 起为必填）。见 `setup.md`。
**输出**：每个子命令都向 stdout 写出一份 JSON `UnifiedPaperEntity[]`（或单个对象）。重定向到 `raw/openalex.json` 供 STEP 5 联邦合并。

## 11 个子命令

| 子命令 | 用途 | 默认预算 |
|---|---|---|
| `search` | 关键词 + 年份 + 类型过滤，按相关性取 top-N；`--recent N` 增加一层近期文献 | limit=25 |
| `get` | 按 OA-ID 或 DOI 取单篇论文 | n/a |
| `deep` | 按单一排序（被引 / 最新 / 相关性）取 top-N | n=100 |
| `double-sort` | 三条支路（被引 / 近期 / 相关性）合并 | 每条支路 n=50 |
| `seminal` | 高被引经典文献（year<=year_max，被引数降序） | limit=10 |
| `reviews` | type=review 过滤 | limit=10 |
| `journal-list` | 限定在 JOURNAL_PRESETS 白名单内 | limit=25 |
| `citation-network` | 单篇论文的反向引用 + 正向 cited_by | refs=50 cited=100 |
| `author` | 按 A-ID 或姓名取 h-index、i10、机构、主题 | n/a |
| `trends` | 通过 group_by("publication_year") 取年份直方图 | 2010-2026 |
| `presets` | 列出可用的期刊预设名称 + 规模 | n/a |

## 语法 + 示例

```bash
# search —— Quick 档默认；相关性 top-30 + 20 篇近期论文（再加前两年的 10 篇）
python3 -m scripts.openalex_helper search "prospect theory" --limit 30 --recent 20

# get —— 单篇论文查询；两种形式都接受
python3 -m scripts.openalex_helper get W3011865677
python3 -m scripts.openalex_helper get 10.2307/1914185           # DOI bare form
python3 -m scripts.openalex_helper get https://doi.org/10.1038/x

# deep —— 单一排序 top-100（Standard 档基线）
python3 -m scripts.openalex_helper deep "working memory training" --n 100 \
    --sort cited_by_count:desc --year-min 2015
# 合法的 --sort 取值：cited_by_count:desc | publication_date:desc | relevance_score:desc
# 除 relevance 外的任何排序都只匹配标题 + 摘要（按被引数或日期排序的全文本匹配返回的多为离题论文：
# 50 篇里命中主题的分别为 0–8 / 0–24，8 个主题）。

# double-sort —— Standard+ / Deep 多策略（推荐）
python3 -m scripts.openalex_helper double-sort "attachment human-robot interaction" \
    --n 50 > raw/openalex.json
# 支路：cited（标题+摘要匹配，按被引数）· recent（最近 365 天内的相关性：n 篇，再往前 730 天：n/2）
# · relevance（整体）。
# 排序：多条支路都命中的论文优先，然后各支路轮流交替——绝不能只按被引数排，
# 否则每篇新论文都会被压到末尾。
# 年份：--year-min / --year-max YYYY（含端点）；search / deep / double-sort / reviews 都接受
# 这两个参数。未给 --year-max → 上限截到当前年份（OpenAlex 里有 2027–2050 年的垃圾日期数据）。
# 近期支路始终留在窗口内：终点在过去的时间窗，取该窗口自己的最后一年。

# seminal —— 高被引经典文献（Deep 档的标志性动作）
python3 -m scripts.openalex_helper seminal "prospect theory" --year-max 2000 --limit 15
# 例如 K&T 1979 Econometrica 被引 46625，在 "prospect theory" 的结果中排第 1。

# reviews —— 对综述文章做 type 过滤（Deep 档；文献综述写作）
python3 -m scripts.openalex_helper reviews "working memory training elderly" --limit 15

# journal-list —— 白名单顶刊
python3 -m scripts.openalex_helper journal-list "ESG disclosure" --preset UTD24 --limit 30
python3 -m scripts.openalex_helper journal-list "BNT162b2 efficacy" --preset medical_top --limit 20

# citation-network —— 对高 RCS 种子做 STEP 9 扩展
python3 -m scripts.openalex_helper citation-network W2964249885 \
    --refs-limit 50 --cited-by-limit 100 >> raw/citations.json
# 返回 {"references": [UnifiedPaperEntity], "cited_by": [UnifiedPaperEntity]}

# 作者档案
python3 -m scripts.openalex_helper author "Daniel Kahneman"
python3 -m scripts.openalex_helper author A1969205032

# trends —— 该主题的年份直方图
python3 -m scripts.openalex_helper trends "transformer language model" --year-min 2017 --year-max 2026
```

## 输出 JSON 字段

每篇论文（`UnifiedPaperEntity`）：`doi`、`arxiv_id`、`openalex_id`、`pmid`、`pmcid`、`title`、`abstract`、`authors[]`、`year`、`venue`、`type`、`citation_count`、`referenced_works_count`、`fwci`、`cited_by_percentile_year`、`topics[]`、`keywords[]`、`sdgs[]`、`is_oa`、`doi_url`、`openalex_url`、`pdf_url`、`sources=["openalex"]`。完整 schema 见 `scripts/types.py`。

`author` 子命令：`{id, name, h_index, i10_index, two_year_mean_citedness, total_citations, works_count, top_affiliations[], top_topics[], orcid}`。

`citation-network`：`{references: [UnifiedPaperEntity], cited_by: [UnifiedPaperEntity]}`。

`trends`：`{year: count}` 字典，按年份排序。

## JOURNAL_PRESETS（6）

| 预设 | 规模 | 适用场景 |
|---|---|---|
| `UTD24` | 24 | 商学院学术标准（会计/金融/管理/营销/信息系统/运营/经济/组织行为） |
| `FT50` | 50 | Financial Times 更宽口径的商业研究方法 |
| `nature_science` | 4 | Nature / Science / Cell / PNAS —— 综合科学顶尖期刊 |
| `ml_top_venues` | 6 | NeurIPS / ICML / ICLR / JMLR / CVPR / ACL —— 作为会议来源的 ML 会议 |
| `medical_top` | 6 | NEJM / Lancet / JAMA / BMJ / Nature Medicine / Cell |
| `Cochrane` | 1 | Cochrane Database of Systematic Reviews —— SR/MA 金标准 |

名称在运行时被解析（RESOLVED）为 OpenAlex source ID（display_name 过滤不可靠；SA-Z2 §4 已验证）。

## OpenAlex 无法服务某次调用时

日预算耗尽、持续限流或宕机都不再以 traceback 结束这次调用。辅助脚本改由 Semantic Scholar 提供同一个子命令。JSON 结构不变，并会打印一行以 `[paper-search-pro] OpenAlex unavailable` 开头的 stderr 信息。带 `--json-envelope` 时，`meta.source` 标明服务本次调用的来源，`meta.fallback` 记录原因与重置倒计时。如果 Semantic Scholar 也服务不了，这次调用的失败方式与此前 OpenAlex 出错时完全一致：退出码 `3`，stdout 输出合法 JSON，stderr 给出原因与重置时间（`journal-list` 保持原样：返回空列表，退出码 0）。`get` 和 `citation-network` 需要有 DOI 才能回退；`author` 和 `trends` 没有回退。细节见 `source_routing.md` § *Automatic fallback*。

## 实证告警（SA-Y2 / SA-Z2 已验证）

- **per_page 硬上限为 20** —— per_page>20 时 pyalex 会返回约 25k token 的载荷，尽管 API 接受最高 200。辅助脚本内部强制 `_PER_PAGE=20` 并自行分页。
- **OpenAlex 的 W-ID 在版本之间会漂移** —— 手上的 W-ID 如果返回 404，改用 DOI 查询重试。`get` 会替你处理 `10.x/y` -> `https://doi.org/10.x/y`。
- **Attention 论文的 bug**（`10.48550/arxiv.1706.03762`）：OpenAlex 返回 `cited_by_count=6543`，这是数据合并的 bug，不是 pyalex 的缺陷。当作已知异常处理；SS 给出的数字更准确（约 80k）。
- **摘要必须重建**：用 `pyalex.invert_abstract()` 从 `abstract_inverted_index` 还原 —— 辅助脚本已经这么做了。早期论文（2000 年前）可能是 `null` 倒排索引 → 摘要为 `None`。这很关键时回退到 SS（STEP 10）。
- **期刊预设过滤需要解析 source ID**，而不是匹配 display_name。`journal-list` 先调用 `Sources().search(name)`，再按 source id 管道过滤 Works —— 多花约 1–2 秒，但这是唯一可靠的做法。
- **arxiv_id 位于 `locations[].landing_page_url`**，不是顶层字段。`_extract_arxiv_id` 会解析形如 `arxiv.org/abs/<id>v5` 的 URL 并去掉 `vN` 后缀。
- **OpenAlex 对 arXiv 的索引滞后约 4–5 天** —— 需要 T-0~T-4 的时效性时，用 `arxiv_helper freshness`（STEP 4）；别指望 OpenAlex 已经有新的预印本。

## 中文检索式处理（跨语言）

OpenAlex 强烈偏好英文关键词（它的索引是多语的，但英文覆盖最广，而且对语料中的绝大部分，`relevance_score` 都是拿英文摘要/标题算出来的）。当用户的检索式是中文时，主 Agent 应在把请求交给本辅助脚本之前，先把**检索词**翻译成英文。用户的原始中文检索式作为报告元数据保留（`--query` 值或 `user_query` 字段），这样最终的 HTML/MD 能显示用户真正问的是什么。

```bash
# 用户："青少年焦虑认知行为治疗"
# 只翻译检索词；原始检索式保留给报告元数据。
python3 -m scripts.openalex_helper double-sort "adolescent anxiety cognitive behavioral therapy" \
    --n 50 > raw/openalex.json
# 然后在 summary.md / report.md 中原样引用用户的原始检索式"青少年焦虑认知行为治疗"。

# 提示：按作者检索时同时带上中文音译名和英文拼写——
# OpenAlex 匹配 "Daniel Kahneman"，但不匹配"卡尼曼"。
```

反模式：把中文字符串直接传给 `--query`，在多数主题上召回率接近零（除中国机构所属期刊外，OA 的中文索引很稀疏）。必须翻译，不要跳过。
