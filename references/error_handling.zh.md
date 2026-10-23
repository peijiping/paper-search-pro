# 错误处理手册

由 `SKILL.md` 的 STEP 3、4、5、6、10、12 引用——凡是外部调用或 LLM 输出可能失败的地方。逐条编号，便于在给用户的消息里直接引用（"E3 SS 限流 —— 退避 60 秒"）。

## 错误目录

### E1 —— 配置缺少必需键

| | |
|---|---|
| **症状** | `RuntimeError: Call pubmed_helper.init(config) first` 或 `openalex_email or openalex_api_key` 为空，或 SKILL.md 的设置检查打印 `MISSING — see references/setup.md` |
| **触发条件** | 首次运行；用户跳过了 `setup.md` |
| **处置** | 停止。不要尝试运行任何东西。告诉用户：`需要先做设置：见 references/setup.md（5 个 key，全部免费，总共约 15 分钟）。我已经停在这里，免得白费力气。` |

### E2 —— OpenAlex 限流 / 429

| | |
|---|---|
| **症状** | `pyalex` 抛出 `HTTPError 429`，或返回空列表且 stderr 出现 `rate exceeded` |
| **触发条件** | 每秒超过 10 次请求且没有 polite pool 凭据；或 polite pool 返回 429（罕见） |
| **处置** | 辅助脚本通过 SDK 重试 3 次。如果持续失败，停止当前策略。`double-sort` 会产生 3 个策略 × 5 页 = 15 次调用——遇到 429 要在策略之间中断。告诉用户：`OpenAlex 正在限流。等待 60 秒后重试——请确认配置里已设置 openalex_email 或 openalex_api_key。` |

### E3 —— Semantic Scholar 富化被拒 / 被限流

| | |
|---|---|
| **症状** | STEP 10 的 stderr 行会带一个原因：`[paper-search-pro] Semantic Scholar enrichment: 0/58 papers enriched (HTTP 429, rate limited).` 或 `(HTTP 403, key rejected)`，也可能出现在 `Semantic Scholar rejected the configured API key (HTTP 403); continuing without a key…` 之后 |
| **触发条件** | 429：持续超过 1 RPS，或共享的无 key 池被限速（不存在付费档；API key 只带来鉴权形态，不带来吞吐）。403：配置的 key 被吊销或输错，无 key 请求也被拒。 |
| **处置** | 辅助脚本已经做过重试：429 会在批量请求上最多等待约 4 分钟（遵守 `Retry-After`），key 被拒则等一次（此时没有 `Retry-After` 可遵守）；它选择停下，而不是对每篇论文重复被拒。未富化的论文在 `kg_classified.json` 中保持原样，退出码为 0——这些论文继续走下去，只是不带 `influential_citation_count` / `tldr`；本次会话内不要重跑。告诉用户：`Semantic Scholar 富化失败（<该行给出的原因>）：<k>/<n> 篇论文已富化；报告中其余论文使用 citation_count。` 如果是 key 被拒，再补一句：`请在 ~/.paper-search-pro/config.yaml 中更新 semantic_scholar_api_key；之后想再富化，重新运行：PYTHONPATH=$PSP_HOME python3 -m scripts.ss_helper --input-file "$SEARCH_DIR/kg_classified.json" --output-file "$SEARCH_DIR/kg_classified.json" --min-rcs 6`，然后重做 STEP 11–12。 |

### E4 —— NCBI 429 / PubMed 限流

| | |
|---|---|
| **症状** | Biopython `HTTPError 429`，或本应有记录时 efetch 返回空 |
| **触发条件** | `ncbi_api_key` 填错 / 过期；或超过 10/s（有 key）/ 3/s（匿名） |
| **处置** | 先验证 key 是否有效：`curl 'https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi?db=pubmed&term=cancer&api_key=YOUR_KEY'`。如果无效：到 https://account.ncbi.nlm.nih.gov/settings/ 重新签发。如果有效：辅助脚本的 `_RATE_LIMIT_SLEEP=0.2` 应该足以支撑；持续 429 说明有另一个进程在共用同一个 key——退避 60 秒。 |

### E5 —— arXiv 429

| | |
|---|---|
| **症状** | SDK 重试 3 次，然后抛出 `arxiv.UnexpectedEmptyPageError` 或 HTTP 429 |
| **触发条件** | 从多个进程调用 arxiv 辅助脚本；或上一次运行留下了被封禁的 IP（可能要冷却 24 小时） |
| **处置** | 辅助脚本使用 `delay_seconds=4`（比 ToU 的 3 秒最低要求多 1 秒余量）——绝不要把它改到更低。不要在用户层面并行化 arXiv 调用。如果被封禁：等待 1 小时后再继续。本次运行放弃 arXiv 新鲜度层；OpenAlex 已经覆盖 ≥T-5。 |

### E6 —— CrossRef 返回 404 且 Content-Type 为 text/plain、"Resource not found."

| | |
|---|---|
| **症状** | `requests.get` 返回 404，`Content-Type: text/plain`，body 是 "Resource not found." |
| **触发条件** | DOI 不在 CrossRef 里（arXiv 的 DOI、部分 2000 年前论文、被下架的论文都属此类） |
| **处置** | 已经处理过了——`_fetch_doi()` 在调用 `.json()` 之前会检查 `application/json` 内容类型。纯文本 404 → 静默返回 None。这篇论文只是拿不到 crossref 富化。无需用户做任何事。 |

### E7 —— SubAgent 分类器返回非法 JSON

| | |
|---|---|
| **症状** | `batch_NNN_result.json` 是畸形 JSON，或是一个对象而不是数组，或夹带多余的散文 |
| **触发条件** | LLM 多输出了评论、markdown 围栏，或者把 schema 重复了一遍而没有填内容 |
| **处置** | `rcs_parser.py` 有 5 层兜底：(1) 严格 `json.loads`；(2) 剥掉 ```` ``` markdown ```` 围栏后重试；(3) 用正则抽取 `[{...}, ...]` 子串；(4) 逐对象正则抽取，产出部分记录并标记 `flag=parse_failed_uncertain`、`rcs=0`；(5) 如果连逐对象解析也失败，整批跳过。除非该批中 ≥50% 的论文都命中第 5 层，主 Agent 不需要重跑这批——如果确实如此，修正 SubAgent 提示词模板并重新派发。 |

### E8 ——（2026-05-23 移除）HTML 体积兜底已取消

本 Skill 不再有"HTML 超过 N MB 时回退到更精简的 jinja2 渲染器"这条路径。理由：它带来不一致的体验（同一个 Skill 会因为数据量不同产出视觉上不同的报告），而现代浏览器能干净地处理 10+ MB 的自包含 HTML 文件。webartifacts 渲染器是唯一的路径，始终打包完整的 Shadcn bundle。如果某次异常运行产出了 20+ MB 的文件，记录下来待跟进，但照原样交付。

### E9 —— 零篇论文命中

| | |
|---|---|
| **症状** | STEP 5 之后 `kg.json` 是 `{}`——联邦合并没有产出任何条目 |
| **触发条件** | 检索式过窄 / 翻译错了；或所有数据源都 429 了 |
| **处置** | 不要生成空报告。明确告诉用户：`没有一篇论文在各个数据源中匹配检索式 "X"。可能原因：(1) 检索式过窄——试着去掉年份过滤，或加上同义词；(2) 领域判断有误——PubMed/arXiv 是不是被不必要地启用了？(3) 限流崩溃——5 分钟后重跑。我没有生成报告。` 然后基于 `references/query_planner.md` 的概念块分析，给出 2-3 个具体的检索式改写建议。 |

### E10 —— 用户中断（分级过程中触发 KeyboardInterrupt）

| | |
|---|---|
| **症状** | 并行 SubAgent 批次运行期间收到 SIGINT |
| **触发条件** | 用户按了 Ctrl-C |
| **处置** | 正在运行的批次会跑完（无法中止 SubAgent）。已经分类好的 `classifications/batch_NNN_result.json` 文件保留；可能缺最后几个批次的 `kg_classified.json` 只是不完整。告诉用户：`在第 N/M 批处停止。部分 KG 已保存到 kg_classified.json——可以重跑 rcs_parser.py 把已有的内容合并进来，或从第 N 批重跑以继续。` |

### E11 —— 2000 年前论文缺少参考文献（可接受）

| | |
|---|---|
| **症状** | `K&T 1979` 从 OpenAlex 返回 `referenced_works_count=0`；SS 也返回 `references=[]` |
| **触发条件** | 上游元数据缺口——OpenAlex / SS 对 2000 年前的高影响力论文没有完整的参考文献列表 |
| **处置** | 当作数据限制接受。不要把 CrossRef 当最后手段——它对老论文有同样的缺口。标记 `flag=null`（这不是解析错误，是上游数据缺失）。在 PRISMA-S 第 14 项的备注里提一句。 |

### E12 —— arXiv DOI 大小写归一化

| | |
|---|---|
| **症状** | OA 给出的 `10.48550/arXiv.1706.03762` 在联邦合并后变成 SS 给出的 `10.48550/arxiv.1706.03762` 的重复项 |
| **触发条件** | 没有做大小写不敏感的 DOI 匹配 |
| **处置** | 已经处理过了——`federated_kg_resolver.normalize_doi()` 在建键之前把一切转成小写。如果你看到重复项，检查所有辅助脚本是不是都走了联邦合并这一步，而不是直接用 Python 调用。 |

### E13 —— SS 找不到 DOI / 论文被下架

| | |
|---|---|
| **症状** | `sch.get_paper(sid)` 返回 None，或 `abstract` 是空字符串，或对一个已知有效的 DOI 返回 ` paperId` 为 None |
| **触发条件** | 出版商下架请求（部分 2020 年后的 Elsevier / Springer 论文）；或 SS 尚未索引该 DOI（很新的论文） |
| **处置** | 优雅跳过——论文只保留 OA 字段。没有 `influential_citation_count`。标记为 `sources=["openalex"]`。不向用户暴露错误。 |

## 主 Agent 的分诊流程

当某条辅助脚本命令产出意外输出时：

1. **先看 stderr**——大多数失败会在那里打出堆栈
2. **对照上面的错误编号定位**
3. **执行列出的处置**——大多数是自动 / 静默的
4. **告诉用户**——限于 E1（配置）、E2-E5（持续限流）、E8（渲染器切换）、E9（无结果）
5. **不要盲目重试**——如果处置说"本次放弃 X"，就照做，不要再试 X。卡在重试循环里比降级报告更糟。

## 怎么告诉用户

格式：`[错误码] 简短描述：我的动作。<如适用，给出一条建议的用户操作>`。示例：

- `[E2] OpenAlex 被限流。等待 60 秒后重试。（你不需要做任何事——自动处理。）`
- `[E3] Semantic Scholar 富化失败（HTTP 429，限流）：0/58 篇论文已富化；报告中只使用 citation_count。`
- `[E9] 没有论文匹配。我建议放宽年份范围，或去掉 "elderly"（年老的）过滤。要我用 <改写后的检索式> 重试吗？`
