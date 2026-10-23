# 期刊分区与指标（SSOT —— 平台、抓取、ISSN 关联、命名）

paper-search-pro 关于论文**期刊**所报告的一切内容的唯一事实来源：多平台分区
（中科院 CAS / JCR / SJR）加上 OpenAlex 开放影响槽位——全部收进一条统一的
`journal_rank` 记录（v2.2 单层收拢）——再加上 ISSN 关联、归属说明和
命名规则。在展示或按任何期刊级数值过滤之前先读这份文件，无论走的是
人类路径（STEP 1、STEP 5 之后紧接着、STEP 10-12）
还是 agent 模式。

> **v2.2（A 线）有什么变化。** 本文件的上一版说 JCR
> 和中科院分区是"仅提供外部链接"，而 SJR 需要绕过 Cloudflare 的
> 浏览器下载。**那已经过时了。** 现在三个平台都在运行时从公开的
> GitHub 镜像用普通 `requests` 拉取（不需要 Cloudflare，不需要新
> 依赖），由 `scripts/journal_rank.py` 完成，按 ISSN 关联，并在每篇
> 论文上呈现为分区。对这三个平台而言，"仅外部链接"的姿态已经取消；
> 但我们仍然**绝不打包或提交任何分区数据**——它是被抓取到用户本地缓存中
> 并在那里使用的。

支配性规则是 **R-04（命名铁律）**：分区就是分区，影响因子就是影响因子，
两者不是可以互换的词。**只有 JCR 的 `IF(2024)` 是真正的影响因子。**
中科院"区"和 SJR quartile 是**分区 / quartile**。OpenAlex 2 年平均被引数
是一个**开放期刊影响力**数值（"期刊影响力"），
明确**不是** JIF（R-09）。

---

## 三个分区平台（A 线的多平台层）

| 平台 | 提供什么 | 原生分类体系 | 是真 IF 吗？ |
|---|---|---|---|
| **中科院 CAS**（中科院文献情报中心） | 大类分区（区 1-4）+ 大类内排名 + Top 标记 + 最多六个小类分区 | 区 1-4（1 = 最好） | 否——"区"是分区 |
| **JCR**（Clarivate） | Quartile Q1-Q4 + **IF(2024)** + 类别排名 | Q1-Q4 | **是**——`impact_factor` 才是真正的 JIF |
| **SJR**（SCImago） | SJR Best Quartile Q1-Q4 + SJR 指标数值 + 各类别分区 | Q1-Q4 | 否——quartile / 数值是分区 |

每篇被标注的论文都带**全部三个**平台的标签（每个槽位各自独立可选——
只在一个平台上找到的期刊同样生成一条有效记录）。当用户要求档位时，
过滤只作用于**一个**平台；另外两个保留为标签。

### 数据来源（GitHub raw 镜像——`requests`，无 Cloudflare，零新增依赖）

镜像 URL 写在配置里（`rank.sources`），用户可以替换。三个镜像在
2026-06-25 都经 curl 验证为 HTTP 200。

| 平台 | 默认镜像文件 | 格式 | ISSN 格式 |
|---|---|---|---|
| CAS | `FQBJCR2025-UTF8.csv`（hitfyd/ShowJCR） | 逗号分隔 CSV，UTF-8，23 列 | `2053-1583/2053-1583`（斜杠，带连字符） |
| JCR | `JCR2024-UTF8.csv`（hitfyd/ShowJCR） | 逗号分隔 CSV，UTF-8，7 列 | ISSN + eISSN 两列，带连字符（任一列都可能是 `N/A`） |
| SJR | `scimagojr 2024.csv`（zotero-sjr-ranker） | **分号**分隔 CSV，**欧式小数**，27 列 | `"15424863, 00079235"`（逗号，**不带连字符**） |

- **列一律按名称解析（模糊匹配），绝不按下标**——线上表头会偏离
  任何规范草案（CAS 使用全角 `OA Journal Index（OAJ）`；线上 SJR 2024
  文件新增了 `SDG` 列 → 27 列）。按名解析正是让关联在逐年快照之间
  保持稳健的原因。
- CAS 大类分区读取 `3 [168/495]` → tier=3, rank="168/495"；`Top` 取值是
  `是/否`；最多六个小类分区（多数行只用一两个）。**默认分区 = 大类**；
  需要更精细的定位时可用小类。
- JCR 的 `IF(2024)` 是普通的美式浮点数（`232.4`）；`Category` 可能包含
  若干以 `;` 连接的条目。**这是整个系统里唯一真正的影响因子。**
- SJR 的 `SJR` 数值使用逗号作小数点（`145,004` = 145.004）；
  `Categories` 形如 `"Hematology (Q1); Oncology (Q1)"`。
- 2026 年起 CAS 停止更新；可以通过配置指向可选的 `XR2026-UTF8.csv`
  （民间 新锐 接棒）——但一旦使用，**必须标注为非官方**。

### 抓取（一次初始化；运行时；绝不提交）

数据在**运行时**被拉取到 `~/.paper-search-pro/ranks/`（`rank.cache_dir`），
**绝不进入仓库 / git 历史**（R-02 / R-03）。首次使用需要一次一次性抓取；
此后以缓存优先，只有在显式强制
或缓存过期（>365 天）时才重新拉取。

```bash
# 一次性（或一年之后）。三个平台全抓：
PYTHONPATH=$PSP_HOME python3 -m scripts.journal_rank fetch
# 单个平台：--platform cas | jcr | sjr ；强制重新拉取：--force
PYTHONPATH=$PSP_HOME python3 -m scripts.journal_rank fetch --platform cas

# 查看已缓存了什么 / 查询某一本期刊：
PYTHONPATH=$PSP_HOME python3 -m scripts.journal_rank info
PYTHONPATH=$PSP_HOME python3 -m scripts.journal_rank lookup 0028-0836
```

网络失败 / 404 会优雅降级（该平台缺失；其余平台照常加载；运行绝不崩溃）。
当没有任何缓存时，`journal_rank.load()` 返回 `None`，整个分区层静默空转——
报告逐字节保持不变（R-19），
与该功能被关闭时完全一样。

### 打标签 + 过滤（逻辑层）

在人类路径上使用该命令，它作用于 KG 文件并保留
其余所有字段：

```bash
PYTHONPATH=$PSP_HOME python3 -m scripts.rank_filter --kg "$SEARCH_DIR/kg.json" \
  --output "$SEARCH_DIR/kg_ranked.json" --platform cas --tiers 1,2   # label + filter
PYTHONPATH=$PSP_HOME python3 -m scripts.rank_filter --kg "$SEARCH_DIR/kg_classified.json" \
  --output "$SEARCH_DIR/kg_classified.json"                            # label only
```

底层的 Python API（agent 模式直接调用它）：

```python
from scripts import journal_rank, rank_filter
lk = journal_rank.load()                    # RankLookup | None (None → graceful degrade)
rank_filter.annotate_papers(papers, lk)     # stamps paper.journal_rank (三家全标), once
kept, dropped, nodata = rank_filter.filter_by_rank(
        papers, platform, tiers=[1], quartiles=["Q1"], top=False)
```

- **annotate 不区分平台**——它一次就为三个平台打上标签。每个结果集
  做一次即可。
- **filter 是对已打标签文献池的重新过滤**——切换平台 / 档位只需在
  同一批候选上再次调用 `filter_by_rank`（**不重新检索**；这就是
  "切换 = 重筛不重搜"契约，见下文流程）。
- **`no_platform_data` 会被报告，绝不静默丢弃**——不在所选平台上的
  期刊会被分到该桶并计数，使缺口始终可见。

---

## 默认 / 询问 / 过滤 / 报告 / 切换流程（spec §7）

这是人类路径的交互契约（agent 模式改为在 `meta.rank` 中
呈现同样的事实，而不提问）：

- **出厂默认标准 = JCR**（`rank.default_platform`，用户可改为
  `cas`/`sjr`）。默认平台只**打标签**；它本身绝不触发过滤。
- **没提分区 → 不过滤。** 展示三个平台的标签；
  让用户自行细化。
- **用户要求了档位 → 过滤一次，且在分级之前**（紧接 STEP 5：
  给即将被过滤掉的论文做分级才是最贵的部分）。档位来自 STEP 1 的
  意图（`parse_rank_intent`）或用户本轮的说明。单次请求的档位过滤是
  **临时**的——绝不自动持久化到 config。
- **歧义的裸 "Q1"（没有平台，也没有持久化默认值）→ 问一个简短问题**：
  *"按 JCR 还是 SJR?顺带设默认吗?"* 识别器绝不为裸分区
  猜测平台。
- **永远要报告本次运行做了什么**：*"本次按 {platform} 筛(留 N / 滤 M)"*，
  再轻量地提议切换标准 / 档位或设置持久化默认值。
  附上该平台的归属说明字符串。
- **切换标准 / 档位 = 重新过滤文献池，而不是重新检索**：在 `kg.json` 上
  重跑 `rank_filter`，只对新纳入的论文做分级。只有当存活论文太少时，
  才回到 STEP 3 加深检索。
- **报告要显示本次运行使用的平台**：当本次运行按某个平台过滤过时，把
  `--rank-platform <platform>` 传给 STEP 12b 渲染器；否则显示 config
  默认值（开箱即用为 JCR）。
- **只有在用户明确说"以后都用 X"时才持久化默认值** → 在
  `~/.paper-search-pro/config.yaml` 中设置 `rank.default_platform`。
  档位永不持久化，只有平台默认值会。

与之对应的 headless flag（`--rank-platform`、`--keep-tiers`、
`--rank-category`、`--deepen-target`、`meta.rank` 块、自适应加深），
见 `references/agent_mode.md`。

---

## 归属说明（按平台，逐字精确——纠正旧版对 SJR 的错误标注）

每当地显示一个分区，匹配的归属说明必须随它一起出现。规范字符串存放在
`journal_rank.ATTRIBUTION` 中，调用方因此无法走样；显示层会自动附上
它们（MD 报告中的 `journal_rank_attribution`，
agent 模式中的 `meta.rank.attribution`）。

| 平台 | 归属说明 / 许可实际情况 |
|---|---|
| **CAS** | 中科院文献情报中心期刊分区表。仅供个人 / 非商业使用；**勿公开传播**。 |
| **JCR** | Journal Citation Reports（Clarivate）。IF / quartile **© Clarivate**；仅供参考。 |
| **SJR** | SCImago Journal Rank，scimagojr.com——**在标注引用的前提下用于非商业用途（SCImago 自定义条款；不是 Creative Commons / "CC BY-NC" 许可）**。 |

> **"CC BY-NC" 这个标注是错的。** 本文件的上一版和旧的 `sjr_helper`
> 把 SJR 称作 "CC BY-NC"。SCImago 官网上没有任何 Creative Commons 文本；
> 实际的条款是一个自定义的"标注引用即可非商业使用"条件
> （法务审查 §2.4）。本文件和 `journal_rank.ATTRIBUTION` 都采用修正后的措辞。

更长的页脚说明见 `journal_rank.LEGAL_NOTICE`（强调运行时抓取 / 不再分发的
立场，以及"只有 JCR IF 才是真 IF"这条规则）。
用于"到源头核实"的外部链接，官方门户地址存放在
`journal_rank.PORTAL_URL`（CAS → fenqubiao.com，JCR → jcr.clarivate.com，
SJR → scimagojr.com）。

### 合规立场：我们不分发数据；只在运行时抓取

分区 CSV **绝不提交进仓库，也绝不再分发**。它们在运行时从公开镜像拉取到
用户自己的本地缓存中并在那里使用。这就是法务审查最终确定的
"用户侧抓取 + 运行时关联 + 外部链接"立场。残留的灰色地带风险由用户
知情选择承担；工具本身只分发**代码 + 来源 URL**，
绝不分发数据。

---

## ISSN 关联（连接键）

分区通过期刊的 **ISSN** 与论文关联：

- **归一化**——每个平台的 ISSN 都归一化到同一个 `XXXX-XXXX`
  大写键（8 位数字，`X` 大写）。JCR/CAS 的 ISSN 带连字符；SJR 的
  不带；它们在同一个规范键上相遇。一本期刊的纸本 + 电子 ISSN 都指向
  同一条记录，所以 eISSN 也能关联上。
- **OpenAlex 路径**——优先取 `source.issn_l`，否则取 `source.issn[]` 的
  第一项作为论文的 `issn`；该来源列出的所有 ISSN 都保留在 `issns` 中，
  当 `issn` 没匹配上时逐个尝试（期刊用于关联的 ISSN 往往不是表里列的
  那个：The Lancet，以及改过名的期刊）。
- **Semantic Scholar 路径**——ISSN 来自 `publicationVenue.issn`，只有约 2/3 的
  情况下存在（R-08）。Agent 模式会通过免费的 OpenAlex 单篇 DOI 查询
  **尽力回填**缺失项，避免关联被静默丢失（实测：一次 SS 运行中回填出 21 条
  里的 16 条）；人类路径在打标签前复用同一批回填后的 ISSN。回填
  **没有保证**：没有 DOI 的记录——或者其 OpenAlex DOI 记录本身不含 ISSN 的
  记录（某些老牌经典文献就是如此，例如 Kahneman & Tversky 1979）——
  仍然无法关联。残留缺口通过 `meta.enrichment.issn_backfill_*` 审计保持
  可见（agent 模式）。

---

## OpenAlex 开放期刊影响槽位（`journal_rank.openalex`）

除了三个分区平台，统一的 `journal_rank` 记录还带有一个 `openalex`
开放影响槽位，取自 OpenAlex `summary_stats`（CC0）。在 v2.2 中，它
**取代**了旧的只支持 SJR 单平台的 `journal_metric`：SJR quartile
现在位于 `journal_rank.sjr.best_quartile`，开放影响数值位于
`journal_rank.openalex`。agent 信封中**不再有单独的逐篇 `journal_metric` 键**，
人类界面渲染的是统一记录。

| 数值 | 现在放在哪里 | 备注 |
|---|---|---|
| **SJR quartile**（Q1–Q4） | `journal_rank.sjr.best_quartile` | 来自 SJR 镜像 CSV；归属说明为必填项。 |
| **OpenAlex 2 年平均被引数** | `journal_rank.openalex.mean_citedness_2yr` | CC0。一个开放口径的影响力数值，**不是** JIF（R-09）。 |
| **OpenAlex 期刊 h 指数** | `journal_rank.openalex.h_index` | 同一来源，开放。 |

R-09 是承重的：`mean_citedness_2yr` ≠ Clarivate 的 JIF（实测：此处 JPSP ≈
2.72，而 JCR JIF 约 7–8——语料不同，窗口也不同）。把它用于你结果集内部的
相对排序，绝不当作绝对 IF 阈值，也绝不把它标注为
"影响因子 / Impact Factor"。

agent 的 `--quartile` / `--min-impact` flag 是在这条统一记录上过滤的
（分别对应 `journal_rank.sjr.best_quartile` 和
`journal_rank.openalex.mean_citedness_2yr`）——它们不再读取已下线的
SJR-only 路径。（旧的 `sjr_helper` 模块和一段休眠的 MD 显示块仍留在
代码树里，只是为了兼容旧的缓存 `kg.json` 文件；线上信封和过滤器都不
依赖它们。）

**过滤契约（信号即旋钮）：** 分区 / 影响力始终会被计算并附到每篇论文上；
过滤是显式开启的，只决定哪些内容**被返回 / 保留**。
在没有过滤器、也没有缓存数据时，每篇论文的返回结果与此前完全一致——
期刊数据是一个纯粹的附加层（R-19）。
`meta.counts.after_journal_filter`（`--quartile` / `--min-impact` 之后的存活数）
和 `meta.counts.after_rank_filter`（`--rank-platform` 档位过滤之后的存活数）
让计数保持可审计。

agent 模式的 flag 与完整信封结构见 `references/agent_mode.md`。
