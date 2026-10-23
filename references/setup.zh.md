# 配置：5 个 key 的获取与配置模板

由 `SKILL.md` 的配置检查步骤引用（任何档级运行之前）。总耗时：约 15 分钟。所有 key 均免费。

> **关于 `$PSP_HOME`**：下面每条命令都使用 `PYTHONPATH=$PSP_HOME`，因此无论 Skill 安装在 `~/.claude/skills/paper-search-pro`（Claude Code）、`~/.codex/skills/paper-search-pro`（Codex）、`~/.agents/skills/paper-search-pro`（跨 Agent 约定），还是其他任何位置，同一段命令都能用。设置一次即可：`export PSP_HOME="<absolute directory containing SKILL.md>"`。如果你不想手动 export，SKILL.md 的 STEP 0 提供了一个三层自动解析器。

## 快速状态检查

在**用户当前的工作目录**下运行（不要 `cd` 进 Skill 目录——原因见 `output_files.md`）：

```bash
PYTHONPATH=$PSP_HOME python3 -c "from scripts.config import load_config; c = load_config(); \
  print('openalex:', bool(c.openalex_api_key)); \
  print('semantic_scholar:', bool(c.semantic_scholar_api_key)); \
  print('ncbi:', bool(c.ncbi_email)); \
  print('crossref:', bool(c.crossref_email))"
```

预期输出（功能完整时全部为 True）：

```
openalex: True
semantic_scholar: True
ncbi: True
crossref: True
```

最低可用组合：`openalex` + `ncbi`（这是支持医学场景的唯一"必需"组合）。SS / CrossRef / arXiv 可选但推荐。

## Key 1 —— OpenAlex（必需，L1 主源）

| | |
|---|---|
| 申请地址 | https://openalex.org/settings/api |
| 成本 | 免费，$1/天额度（约 10k 次列表调用 / 1k 次检索） |
| 发放时间 | 即时 |
| 填入 | `openalex_api_key` |
| 为什么必需 | L1 主源。自 2026-02-13 起 API key 变为强制；`mailto=` polite pool 已废弃并被忽略。 |

**步骤**：
1. 在 https://openalex.org 注册（任意邮箱）
2. 访问 https://openalex.org/settings/api → 生成 API key
3. 粘贴到配置中的 `openalex_api_key`

Standard 档每次检索约 200 次调用 → 远在免费的 $1/天额度之内。Deep/Audit 档的重度用户可以预付 $1-5；学术用户可以发邮件给 support@openalex.org 申请免费额度扩展。

**验证**：
```bash
curl -s 'https://api.openalex.org/works?search=cancer&api_key=YOUR_KEY&per_page=2' | head -c 300
# 应返回以 {"meta":{... 开头的 JSON
```

## Key 2 —— Semantic Scholar（推荐，L3）

| | |
|---|---|
| 申请地址 | https://www.semanticscholar.org/product/api |
| 成本 | 免费，不存在付费档 |
| 发放时间 | 约 1 周（人工邮件审核） |
| 填入 | `semantic_scholar_api_key` |
| 为什么推荐 | 增加 `influentialCitationCount`（独有信号）+ 摘要兜底 |

**步骤**：
1. 访问 https://www.semanticscholar.org/product/api
2. 点击 "Get API Key" → 填写 name、email、organization、use case（简要描述 paper-search-pro）
3. 等待审批邮件（通常 3-7 个工作日）
4. 把 key 复制到配置里

**验证**：
```bash
curl -s 'https://api.semanticscholar.org/graph/v1/paper/DOI:10.1162/neco.1997.9.8.1735?fields=influentialCitationCount' \
  -H 'x-api-key: YOUR_KEY' | head -c 200
# 应返回 {"paperId":"...","influentialCitationCount":10359}
```

## Key 3 —— PubMed / NCBI（医学场景必需，L2）

| | |
|---|---|
| 申请地址 | https://account.ncbi.nlm.nih.gov/settings/ |
| 成本 | 免费 |
| 发放时间 | 5 分钟 |
| 填入 | `ncbi_email`（服务条款强制要求）+ `ncbi_api_key`（把 3 → 10 req/s） |
| 为什么必需 | 医学检索；SR 工作流中的 PubMed MeSH 增强 |

**步骤**：
1. 访问 https://account.ncbi.nlm.nih.gov/settings/ → 登录（Google / ORCID / 新建 NCBI 账号）
2. 滚动到 "API Key Management" 部分
3. 点击 "Create an API Key"
4. 把 36 位十六进制字符串复制到配置里
5. 把邮箱另行填入 `ncbi_email`

**验证**：
```bash
curl -s 'https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi?db=pubmed&term=metformin&api_key=YOUR_KEY&retmax=2' | head -c 400
# 应返回包含 <IdList> 的 XML <eSearchResult>
```

## Key 4 —— CrossRef（推荐，L3）

| | |
|---|---|
| 申请地址 | （无 —— 不需要 key） |
| 成本 | 免费 |
| 发放时间 | 即时 |
| 填入 | 仅 `crossref_email` |
| 为什么推荐 | 增加资助方 / 许可协议 / 临床试验编号；polite pool 10/s × 3 并发 |

**步骤**：把你的邮箱填入 `crossref_email` 即可——CrossRef 会从 User-Agent 头读取它，以进入 polite pool（无需 API key，无需注册）。匿名访问仍然可用，限速为 5/s × 1 并发；加上邮箱即可轻松把吞吐量翻倍。

**验证**：
```bash
curl -s -H 'User-Agent: paper-search-pro/2.0 (mailto:YOUR_EMAIL)' \
  'https://api.crossref.org/works/10.1056/NEJMoa2034577' | head -c 400
# 应返回 JSON {"status":"ok","message":{...}}
```

## Key 5 —— arXiv（无需 key）

| | |
|---|---|
| 申请地址 | （无） |
| 成本 | 免费 |
| 需要 | 无 |
| 为什么列出 | 仅作说明——无需注册，但限速严格（每 3 秒 1 次请求） |

arxiv 辅助脚本会自动强制执行这一限制。

## 可选 —— 期刊分区（中科院分区 / JCR / SJR；无需 API key）

只有在你想**按期刊档位过滤或标注结果**时才需要（中科院一区、JCR Q1、SJR quartile、顶刊、`--quartile`、`--rank-platform`）。这是一次性数据抓取——**与上面全部五个 key 无关**：它通过普通 HTTP 拉取公开的 GitHub raw 镜像，因此**不需要 OpenAlex / SS / 任何 API key**。

| | |
|---|---|
| 申请地址 | （无 —— 无需 key，无需注册） |
| 成本 | 免费 |
| 一次性抓取 | `PYTHONPATH=$PSP_HOME python3 -m scripts.journal_rank fetch` |
| 数据落在 | `~/.paper-search-pro/ranks/`（`cas_2025.csv` / `jcr_2024.csv` / `sjr_2024.csv`） |
| 重新抓取 | 一次初始化即可（文件存在且新鲜时绝不重新下载）；分区数据约每年更新一次 |

```bash
# 一次性：把三个平台（CAS / JCR / SJR）全部拉到 ~/.paper-search-pro/ranks/
PYTHONPATH=$PSP_HOME python3 -m scripts.journal_rank fetch
# 确认已缓存了什么：
PYTHONPATH=$PSP_HOME python3 -m scripts.journal_rank info
```

- 数据**绝不随本仓库分发**——它在运行时被抓取到你本地的缓存中，并且在任何显示分区的地方都会带上来源平台的归属说明（CAS 中科院文献情报中心 · 勿公开传播 / JCR © Clarivate / SJR © SCImago，非商业引用使用——**不是** CC BY-NC）。只有 JCR IF(2024) 是真正的影响因子；中科院"区"和 SJR quartile 是分区，而"期刊影响力"这个数值是 OpenAlex 2 年平均被引数（开放指标，只用于相对比较——R-04/R-09）。
- **`ranks/` 是期刊分区的唯一事实来源**。v2.2 之前遗留的 `~/.paper-search-pro/sjr/` 目录已不再被检索流程使用——你可以忽略（或删除）它；改用上面的 `journal_rank fetch` 来填充 `ranks/`。
- 不做这次抓取，检索仍然照常运行——只是论文不带分区标签，分区过滤（`--quartile` / `--rank-platform`）无法生效（本次运行会在 `meta.rank.note` 中报告这一点并继续，绝不报错）。完整的分区模型见 `references/journal_metrics.md`。

完整的期刊分区模型（平台、切换、R-04 命名）见 `references/journal_metrics.md`。

## 完整的 config.yaml 模板

收齐 key 后写入 `~/.paper-search-pro/config.yaml`（首次运行时辅助脚本会按默认值自动创建该文件；你只需要编辑它）：

```yaml
# ============================================================================
# paper-search-pro config — 在这里填入你的 key
# ============================================================================

# --- OpenAlex（必需）---
openalex_api_key: "abcdef1234567890..."
openalex_email: ""            # 自 2026-02-13 起废弃；被忽略

# --- Semantic Scholar（推荐）---
semantic_scholar_api_key: "abcdef1234567890..."

# --- PubMed / NCBI（医学场景必需）---
ncbi_email: "you@example.com"
ncbi_api_key: "0123456789abcdef..."

# --- CrossRef（推荐）---
crossref_email: "you@example.com"

# --- 输出 ---
output_dir: "./paper-search-results"
default_tier: "standard"      # quick | standard | deep | audit
language: "en"                # en | zh — 影响消息，不影响数据

# --- HTML 渲染 ---
# （截至 2026-05-23 没有 `html_renderer` 选项。Skill 始终使用
#  webartifacts Shadcn React 包；基于体积的 jinja2 回退
#  已为 UX 一致性而移除。没有体积上限。）

# --- 缓存（自动启用，安全默认值）---
cache_enabled: true
cache_ttl_days: 7
cache_max_size_mb: 500

# --- 日志 ---
log_level: "INFO"
```

## 验证脚本

编辑完配置后，在**用户当前的工作目录**下运行完整检查（`PYTHONPATH=` 前缀让 `scripts.*` 无需 `cd` 即可导入）：

```bash
PYTHONPATH=$PSP_HOME python3 -c "
from scripts.config import load_config
c = load_config()
status = {
    'openalex_email':           bool(c.openalex_email),
    'openalex_api_key':         bool(c.openalex_api_key),
    'semantic_scholar_api_key': bool(c.semantic_scholar_api_key),
    'ncbi_email':               bool(c.ncbi_email),
    'ncbi_api_key':             bool(c.ncbi_api_key),
    'crossref_email':           bool(c.crossref_email),
}
for k, v in status.items():
    print(f'{k:30s} {\"OK\" if v else \"MISSING\"}')
all_ok = status['openalex_api_key'] and status['ncbi_email']
print('\\nMINIMUM:', 'READY' if all_ok else 'NOT READY — fill openalex_api_key + ncbi_email')
"
```

## 常见的配置失败

| 失败 | 原因 | 修复 |
|---|---|---|
| `config.yaml not found` | 从未进行过首次运行 | 任选一个辅助脚本运行一次——config.py 会按默认值自动创建 |
| `yaml.YAMLError: while parsing` | 配置里混入了制表符 / 引号 | 只用空格。含特殊字符的键要加引号 |
| `Permission denied: ~/.paper-search-pro/config.yaml` | 文件所有者不对 | `chown $USER ~/.paper-search-pro/config.yaml` |
| `openalex_email empty after editing` | 配置已加载，但字段名拼错了 | 核对键名——必须完全一致（例如 `openalex_email` 而不是 `openalex-email`） |
| `NCBI 403 immediately` | API key 尚未生效 | 创建后等 5 分钟；NCBI 的生效不是即时的 |
| `SS 429 from minute 1` | key 未关联付费档（也不存在付费档） | 属预期情况——1 RPS 是硬上限；辅助脚本靠 sleep 处理 |

## 隐私说明

`ncbi_email`、`openalex_email` 和 `crossref_email` 会随 HTTP 头发送——这些服务会按请求记录邮箱，用于限流归属。不要使用高价值的主邮箱；如果你的邮箱服务商支持加标签的变体（tagged variant），就用一个（例如 `you+pspro@gmail.com`）。

## 安全：配置文件权限

`~/.paper-search-pro/config.yaml` 存放真实的 API key，**权限必须是 0600**（仅所有者可读写）。`config.py` 会自动强制这一点：

- 首次运行时，文件按默认值创建，权限为 0600。
- 每次调用 `load_config()` 时，如果现有文件的权限被放宽了（例如被 `chmod 644` 或备份工具还原过），它会被重新 chmod 回 0600。发生时会向 stderr 打印一条警告，让你知道权限已被收紧。

你可以自己核查当前权限：

```bash
stat -f %p ~/.paper-search-pro/config.yaml    # macOS — expect 100600
stat -c %a ~/.paper-search-pro/config.yaml    # Linux — expect 600
```

如果文件权限显示为 `0644`（全局可读），同一台机器上的其他用户就能读到你的 API key。任选一个 paper-search-pro 辅助脚本重跑一次，自动 chmod 会恢复 0600；或者手动执行 `chmod 600 ~/.paper-search-pro/config.yaml`。

**Windows 说明**：`os.chmod` 在大多数 Windows 文件系统上是空操作。这种情况下自动 chmod 会静默跳过——请改用 NTFS ACL 或 Windows 的"属性 → 安全"选项卡来保护文件。
