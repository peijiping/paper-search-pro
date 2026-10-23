# arxiv_helper CLI 速查表

由 `SKILL.md` STEP 4（L2 增强器）引用。两种用途：CS/AI 主题的**近期层**（最近一年最相关的预印本），以及**时效哨兵**（最近 4 天，任何索引都还没有）。

**角色**：L2 增强器。旧的前提 —— "超过约 4 天后，98% 的 arXiv 论文也已同时存在于 OpenAlex"（SA-V2，39/40）—— 在 2026-09-30 已不再成立：20 篇近期在题的 OpenReview 论文中有 7 篇不在 OpenAlex 里，而已存在的那部分也没被检索出来。所以 CS/AI 主题要跑一年的近期层，而不只跑哨兵。

**入口**：`python3 -m scripts.arxiv_helper <subcmd> [args]`。
**不需要 key**。限流：每 3 秒 1 次请求（arXiv ToU 强制执行）。辅助脚本 SDK 使用 `delay_seconds=4`（留 1 秒安全余量）+ `page_size=100` + 20 秒请求超时。失败或超时时，CLI 打印 `[]` 和一行 stderr 提示，而不是崩溃。

**普通词是全都必须满足的。** arXiv 会把裸词用 OR 连起来（"diffusion language model" 曾返回意第绪语和罗曼什语的语言模型论文），所以辅助脚本发送的是 `all:diffusion AND all:language AND all:model`。已经用了 `AND` / `OR` / `ANDNOT` 或字段前缀（`ti:`、`abs:`、`au:` …）的检索式按原样透传；带引号的短语仍然是短语。检索式控制在 2–4 个核心词：每多一个词，范围就再窄一分。

## 三个子命令

| 子命令 | 用途 | 档级 |
|---|---|---|
| `search --sort relevance --since-days 365` | 近期层：最近一年最相关的预印本 | 所有档级，CS/AI 主题 |
| `freshness` | T-0~T-N 天窗口（默认 4 = OA 滞后） | 检索式中的时效信号 |
| `search`（无窗口） | 按提交时间 / 相关性做通用 arXiv 检索 | Audit |
| `get` | 按 arxiv_id 取单篇论文 | 诊断 |

## 语法 + 示例

```bash
# 近期层 —— CS/AI 主题
python3 -m scripts.arxiv_helper search "diffusion language model" \
    --sort relevance --since-days 365 --limit 30 > ./paper-search-results/<id>/raw/arxiv.json
# 2026-10-01：30/30 条结果的标题里都含有 "diffusion"（在 AND 修复之前，前六条里混进了意第绪语和罗曼什语的语言模型论文）

# freshness —— OA 还看不到的预印本
python3 -m scripts.arxiv_helper freshness "LLM reasoning" --days 4 --limit 30 \
    > ./paper-search-results/<id>/raw/arxiv_fresh.json
# 返回 published >= now - 4 days 的预印本。arXiv 按日期降序返回，所以一旦跌破截止点，
# 就能干净地截断。

# 带全部分类的 freshness（物理科学类检索式要包含 physics/astro/cond-mat）
python3 -m scripts.arxiv_helper freshness "neutron star merger" --days 7 --limit 20 --all-cats

# search —— Audit 档通用检索
python3 -m scripts.arxiv_helper search "diffusion model robustness" --limit 30 --sort submitted
# 合法的 --sort：submitted（默认，最新优先）| relevance | lastUpdated

# get —— 单篇论文查询
python3 -m scripts.arxiv_helper get 1706.03762            # bare id
python3 -m scripts.arxiv_helper get 1706.03762v5          # version-suffixed
python3 -m scripts.arxiv_helper get https://arxiv.org/abs/2401.12345
python3 -m scripts.arxiv_helper get hep-th/9707234         # legacy form
```

## 分类 —— DEFAULT vs --all-cats

`DEFAULT_CATEGORIES`（除非加 `--all-cats`，否则用这一组）：

| 分类 | 领域 |
|---|---|
| `cs.*` | 计算机科学（CL/AI/LG/CV 等） |
| `math.*` | 数学 |
| `stat.*` | 统计学 |
| `q-bio.*` | 定量生物学 |
| `q-fin.*` | 定量金融 |
| `econ.*` | 经济学 |

`--all-cats` 额外加上：`physics.*`、`astro-ph.*`、`cond-mat.*`、`hep-th/ph/ex/lat`、`gr-qc`、`nlin.*`、`nucl-th/ex`、`quant-ph`、`eess.*`。

**为什么默认排除物理学**：Q4 实测 (SA-V2 §3.4) —— 检索 "prospect theory" 而不加分类过滤时，返回的是名为 "PROSPECT" 的反应堆实验。分类过滤是社科类检索式唯一的防线。要用物理学必须显式开启。

## 输出 JSON 字段

每篇论文（`UnifiedPaperEntity`）：

```jsonc
{
  "doi": "10.48550/arxiv.2401.12345",      // canonical lowercase, no version
  "arxiv_id": "2401.12345",
  "title": "...",
  "abstract": "...",                        // arXiv always has full abstract
  "authors": [{"name": "Yann LeCun"}],
  "year": 2024,
  "venue": "arXiv",
  "type": "preprint",
  "arxiv_categories": ["cs.CL", "cs.AI"],
  "arxiv_comment": "Accepted at NeurIPS 2024",   // submitter's note; often empty
  "pdf_url": "https://arxiv.org/pdf/2401.12345v3",
  "sources": ["arxiv"],
  "discovery_path": "arxiv:freshness"
}
```

## 辅助脚本工具函数（Python API）

| 函数 | 用途 |
|---|---|
| `normalize_arxiv_doi(doi)` | `10.48550/arXiv.1706.03762v5` → `10.48550/arxiv.1706.03762` |
| `extract_arxiv_id(s)` | URL / 带版本号 → 裸 id（同时处理新格式与旧格式） |
| `search_freshness_window(q, days, limit, categories)` | 以编程方式做时效性检索 |

## 实证告警（SA-V2 24_v2_l2_booster_test、25_round2_synthesis）

- **arXiv 的 DOI 大小写混乱** —— arXiv 输出 `10.48550/arXiv.<id>`（大写 X），OpenAlex 输出 `10.48550/arxiv.<id>`（小写）。`normalize_arxiv_doi` 会转小写并去掉版本后缀。`federated_kg_resolver.normalize_doi` 做同样的事 —— 去重时两者都需要。
- **限流严格为每 3 秒 1 次请求** —— SDK 默认用 4 秒。跑得更快 = 429 + 临时封禁。不要并行调用。
- **arXiv 上的相关性排序很弱** —— SA-V2 §1.5：检索 "attention is all you need" 的前 5 条里并没有 1706.03762 本身。相关性用 OpenAlex；arXiv 只在 `submitted` 排序（最新优先）或按具体 id 查询时用。
- **OpenAlex 对 arXiv 的索引滞后约 4-5 天** —— T-0 / T-1 / T-2 在 OpenAlex 上都是 0 命中；T-3 开始零星出现；T-4+ 才稳定。这是 arXiv 唯一有独特价值的窗口。
- **跨列多分类**：一篇论文可以同时列出 `cs.LG`、`stat.ML` 和 `physics.bio-ph`。DEFAULT 分类过滤是 OR 关系，命中任意一个即可。社科类检索式要小心 `--all-cats` —— 会混进物理学的噪声。
- **arXiv 上摘要总是存在**（不像 OA 需要重建、也不像 SS 有撤下造成的空缺）—— 对于其他数据库尚未收录的最新论文，arXiv 是有用的摘要来源。
- **版本后缀到处都被剥掉** —— `arxiv_id` 始终是裸 id（没有 `vN`）。DOI 也做了归一化。如果你需要特定版本，直接查源站 URL；辅助脚本为了去重安全而去版本。

## 中文检索式处理（跨语言）

arXiv 是纯英文的预印本服务器 —— 标题、摘要和检索全是英文。对 `freshness` / `search` 传中文检索式，即使服务器上确实存在相关的中文作者论文（这些论文也是用英文提交的），命中数也约为 0。主 Agent 必须先把检索词翻译成英文再调用本辅助脚本；用户的中文检索式只作为报告元数据保留。

```bash
# 用户："扩散模型鲁棒性"  (diffusion model robustness)
python3 -m scripts.arxiv_helper search "diffusion model robustness" --limit 30 --sort submitted

# 用户："大语言模型推理 最新"  (LLM reasoning, recent)
python3 -m scripts.arxiv_helper freshness "LLM reasoning" --days 4 --limit 30 \
    > ./paper-search-results/<id>/raw/arxiv.json
```

对于在 arXiv 上没有英文对应物的纯社科 / 人文类中文检索式，直接跳过 arXiv 增强器（source_routing 已经处理了这一点 —— arXiv 只在 CS / 时效信号时启用）。
