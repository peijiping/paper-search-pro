---
name: paper-search-pro
description: "跨最多 8 个来源检索学术文献（英文：OpenAlex / Semantic Scholar / CrossRef / PubMed / arXiv，AI 会议：OpenReview，中文原生检索：NSSD 国家哲社文献中心 + yiigle 中华医学期刊），检索深度可调——从快速扫描（5 分钟）到审计级准备（3 小时）。当用户想找论文、做文献检索、收集参考文献、界定研究范围、检索中文原生文献（中文文献/中文核心/CSSCI/C刊/国内研究/国内文献/中华××期刊/心理学报/经济研究），或按期刊分区筛选结果（中科院分区/一区/几区，Q1，JCR/SJR 分区，影响因子/impact factor，期刊分区，顶刊/top journal，'按分区筛'），或导出可粘贴到外部数据库的专业检索式（PubMed / WOS / Scopus / Embase / 知网 CNKI / 万方 / SinoMed——'写检索式'、'导出检索式'、'WOS 检索式'、'知网专业检索'、'search strategy'、'boolean search strategy'）时使用。触发词包含检索动作（'find papers'、'literature search'、'papers about X'）、综述类型（'scoping review'、'systematic review'、'SR prep'、'literature review'、'lit review'、'help me write a lit review'）、中文（'找文献'、'找论文'、'论文搜索'、'学术检索'、'文献检索'、'文献综述'、'综述前期'、'求文献'、'中文文献'、'中文核心'、'CSSCI'、'C刊'、'国内研究'、'找中文的'）。输出 Shadcn HTML 报告 + BibTeX/RIS/CSV + PRISMA-S 日志。不适用于：概念解释（'what is X' / 'X 是什么'，例如'影响因子怎么算'）、写作（'帮我写' / 'help me write a paragraph'）、单篇论文解读或带元数据的 PDF 下载（用 paper-downloader-portable），或当用户已有文献集合（用 literature-set-review）。"
license: Apache-2.0
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Task
metadata:
  author: Bo
  version: 2.5.0
  vendored-from: futurehouse/paper-qa (Apache 2.0)
---

# paper-search-pro

深度可调的多源文献检索。四个档级，五个数据源，由你（主 Agent）统一编排。Python 辅助脚本负责确定性工作；LLM 分级交由并行的 Inline SubAgent 完成——无需任何外部 API key。

## 何时使用本 Skill

- 用户想找学术文献 / 找文献 / 论文搜索
- 用户在准备文献综述、系统综述（SR）、范围综述（scoping review）或元分析
- 用户想为论文 / 提案 / 课程作业 / 新闻报道界定某个主题的研究范围
- 用户问"关于 X 有什么研究" / "找找关于 Y 的论文"
- 用户提交了暗示需要搜集文献的检索式（PICO、SPIDER、MeSH、RCT 等）

## 何时不要使用

- 用户想**阅读**某篇具体论文（用 PDF 阅读器 / 下载工具）
- 用户想**总结**一篇已知论文（用摘要工具）
- 用户想**下载**给定 DOI 的 PDF（用 `paper-downloader-portable`）
- 用户已经有文献集合，想写综述（用 `literature-set-review` / `factor-outcome-review`）
- 用户想要概念解释而不是文献（"什么是前景理论" → 直接回答即可）

---

## 🤖 被其他 Agent 调用 / headless 模式

**如果你是作为 Agent 调用本 Skill 来为自己服务的（而不是为需要 HTML 报告的人类用户）**，不要手工执行下面那套 14 步流程。有一条专门为你准备的结构化数据通道：

```bash
PYTHONPATH=$PSP_HOME python3 -m scripts.agent_search "<query>" > result.json
```

一条命令跑完整个确定性内核——多策略检索 → 去重 → 启发式相关性打分（每篇论文都会算）→ 饱和度信号 → 配额快照 → 逐篇期刊指标——并输出**一个 JSON 信封**（无 HTML、无 PRISMA、无 LLM 分级 SubAgent）。人类路径不受影响。
该命令给你的是一个确定性**下限**，而不是最终成果——agent 模式并不打算停在机器输出上；`references/agent_mode.md` 才是你在其上叠加自身语义判断、达到人类流程同等质量的地方（命令保证下限，质量由你补齐）。

📖 **阅读 `references/agent_mode.md`** 获取完整的信封结构、所有 flag
（`--verify`、`--min-relevance`、`--quartile`、`--min-impact` 等）、相关性公式、
错误码 / 退出码，以及来源选择。它是给 Agent 调用方的唯一事实来源（SSOT）——本节的其余内容只是指向它的指针。

从这里往下是**面向人类的 14 步流程**（HTML 报告 + 导出）。当消费方是一个人的时候用它。

---

## 🔥 执行纪律（动手之前先读）

四条不变式统辖每一步——真实会话中最主要的失败模式就是忽略它们：

- **A —— 绝不 `cd` 进 Skill 目录。** `cd $PSP_HOME` 会把 `./` 重绑到 Skill 资源目录，于是 `./paper-search-results/...` 会落在 Skill 内部而不是用户的工作区（而且重新安装会把它抹掉）。所有辅助脚本都从用户的 PWD 运行：`PYTHONPATH=$PSP_HOME python3 -m scripts.<name> … > "$SEARCH_DIR/..."`。`$PSP_HOME`（STEP 0）是安装目录；`$SEARCH_DIR`（STEP 0）是用户 PWD 下的绝对路径。
- **B —— 并行派发分类 SubAgent。** STEP 6 会在一条 assistant 消息中放入最多 5 个 `Task` 块；串行派发会把 Standard 档从约 10 分钟拉长到约 17 分钟。示例见 STEP 6——其他地方不再重复。
- **C —— 每一次跳过都要声明。** 如果你跳过某个 STEP（预算 / 数据为空 / 用户选择），要说清*跳过了什么*、*为什么*、*损失了什么*、*如何补回*（例如"改用 `--tier deep` 重跑"）。跳过没关系；让用户意外不行。
- **D —— 当某一步对当前案例而言不简单时，阅读它引用的参考文件。** 每个 STEP 都会指明一个 `references/<file>.md`，里面写着此处不复述的边界情况。你不会每次都读完所有参考文件，也不应该——但对*即将执行*的那一步跳过参考文件，正是边界知识（dict 与 list 的形状、enrich 而非 search、DOI 大小写）丢失的地方。读你眼前的那一个。

---

## 架构一览

```
你（主 Agent）按本 SKILL.md 驱动整个工作流。
Python 辅助脚本做确定性工作——内部没有 LLM，不需要外部 API key。

  L1 OpenAlex（主源）    → 深度 top-100 多策略
  L2 PubMed（医学）      → MeSH 增强器（主要用途；Audit 档可独立检索）
  L2 arXiv（CS/AI）       → 近一年近期层（遇到时效性词汇时再加 4 天哨兵）
  L2 OpenReview（AI）     → 录用会议标签（ICLR / NeurIPS / ICML …）
  L3 Semantic Scholar    → influentialCitationCount + 摘要兜底
  L3 CrossRef            → 资助方 / 许可协议 / 临床试验编号

  分级                   → Inline SubAgent（并行，文件 IPC，每条消息 5 个）
  输出                   → HTML（Shadcn）+ MD + BibTeX/RIS/CSV + PRISMA-S 日志
```

---

## 四个档级——先做选择

| 档级 | 实际耗时 | 文献数 | 何时选 |
|------|------------|--------|--------------|
| Quick | ~5-8 分钟 | 20-60 | "查一下" / "几篇" / "明天前要" / 快速摸底 |
| **Standard**（默认） | ~10-17 分钟 | 60-180 | 界定主题 / 写背景 / 常规文献检索 |
| Deep | ~30-45 分钟 | 180-400 | "thorough" / 写综述文章 / 综述写作 |
| Audit | ~2-3 小时 | 400-1000+ | "systematic review" / "PRISMA" / "Cochrane" / "meta-analysis" |

📖 **选择之前，请阅读 `references/tier_decision.md`。** 告诉用户你的选择以及理由。对于 Audit，先展示局限性警告并获得明确确认后再开始。

---

## 流程手册

每次文献检索都按顺序执行以下步骤。每一步都引用一个 `references/` 文件作为细节说明。仅当某一步对本案例显然无关紧要时才跳过该文件——并按规则 C 声明这次跳过。

### STEP 0 —— 环境准备（$PSP_HOME + 工作目录）

📖 执行本步之前，先阅读：`references/setup.md`。

**把 Skill 安装路径解析为 `$PSP_HOME`**（后续每一步都会用 `PYTHONPATH=$PSP_HOME`）。优先使用显式注入 / Agent 环境变量；否则扫描各 Agent 常见的安装位置。如果你的宿主已经暴露了本 SKILL.md 的绝对路径，直接 `export PSP_HOME="<该目录>"` 并跳过扫描。📖 完整原理、为什么这不能写成脚本、以及完整的路径清单：`references/runtime_bootstrap.md`。

```bash
PSP_HOME="${PSP_HOME:-${CLAUDE_SKILL_DIR:-${CODEBUDDY_SKILL_DIR:-}}}"   # 显式注入 / Agent 注入
if [ -z "$PSP_HOME" ]; then                                            # 否则扫描已知安装位置
  for base in "$HOME/.claude" "$HOME/.codex" "$HOME/.agents" "$HOME/.config/opencode" \
              "$HOME/.codeium/windsurf" "$HOME/.config/goose" "$HOME/.cline" "$HOME/.roo" \
              "$HOME/.copilot" ./.claude ./.codex ./.agents ./.cursor ./.opencode ./.windsurf; do
    [ -f "$base/skills/paper-search-pro/SKILL.md" ] && PSP_HOME="$base/skills/paper-search-pro" && break
  done
fi
[ -z "$PSP_HOME" ] && { echo "ERROR: paper-search-pro install not found. Set PSP_HOME to the dir containing SKILL.md."; exit 1; }
export PSP_HOME; echo "Using Skill install: $PSP_HOME"
```

**校验配置项**（可从任意 cwd 执行，绝不 `cd` 进 Skill 目录）：

```bash
PYTHONPATH=$PSP_HOME python3 -c \
  "from scripts.config import load_config; c = load_config(); print('OK' if c.openalex_api_key and c.ncbi_email else 'MISSING — see references/setup.md')"
```

如果显示 "MISSING"，把用户引导到 `references/setup.md`（5 个 key，全部免费，总共约 15 分钟）并停止。

**设置工作目录变量**——后续每一步都使用 `$SEARCH_DIR`：

```bash
SEARCH_ID="<topic_slug>_<tier>_$(date +%Y%m%d_%H%M%S)"   # 例如 clt_education_quick_20260522_103045
SEARCH_DIR="$(pwd)/paper-search-results/$SEARCH_ID"
mkdir -p "$SEARCH_DIR/raw" "$SEARCH_DIR/batches" "$SEARCH_DIR/classifications"
echo "Outputs will land in: $SEARCH_DIR"
```

`$SEARCH_DIR` 现在是**用户 PWD 下的绝对路径**。在下面每条辅助脚本命令中都使用 `"$SEARCH_DIR/..."`（带引号、带变量），不要用 `./paper-search-results/...`。

### STEP 1 —— 规划检索式（所有档级均为必做）

📖 执行本步之前，先阅读：`references/query_planner.md`。

**检测报告 UI 语言**——`UI_LANG`（中文检索式为 `zh`，其余为 `en`）决定最终 HTML 报告用哪种 UI 语言渲染。论文标题 / 摘要 / 作者 / 期刊名**绝不翻译**——只翻译报告的界面外壳。STEP 12b 需传入 `--language $UI_LANG`。

```bash
UI_LANG=$(PYTHONPATH=$PSP_HOME python3 -m scripts.detect_language "$USER_QUERY")
```

检测器会把日语 / 韩语 / 欧洲语言检索式路由到**英语**（内置包只带 EN + ZH 两套词典；英语是国际学术默认语言）。📖 确切的 Unicode 规则，以及为什么要先查假名再查汉字，见 `references/runtime_bootstrap.md`。

**确定检索语言空间**（`search_language`，坐标轴 2——*检索哪一片文献海洋*，与上面那个只管*报告外壳*的 `UI_LANG` 不同）。📖 解析的唯一事实来源是 `references/source_routing.md` §"Language scope"；要在措辞构造检索式之前就定好这个空间，因为它会改变 STEP 3 检索式的构造方式。这是**增量且需主动触发**的——纯英文检索式会解析为 `en` 空间，不产生任何新提示或行为变化（R-19）；下面的一切只在中文检索式或明确信号出现时才生效。

- 读取持久化的默认值 `config.search_language`（auto | en | zh | both），并按优先级阶梯 **flags > 检索式内标记 > config > auto** 套用。CJK 字符的存在是上面 `detect_language` 给出的机械事实；标记（`CSSCI`、`中文文献`、`SSCI`、`知网` 等）与非信号（`中科院一区`、主题与中国相关）需要你按 §"Language scope" 表格做语义判断。
- **`auto` + 中文（CJK）检索式 + 无语言标记 + 无持久化值 → 在检索前只问一个问题**（这是人类路径该做的事；CLI / agent 路径则直接透传）。两句话，同时提出可以记为默认值，并且本次会话后续不要再问：

  > 你用中文提问——文献要英文、中文,还是都要?顺便可以说"以后都这样",我就记成默认、下次不再问。

  - "中文" / "都要" → 进入该空间（由 STEP 2 的学科路由接管；在那里报告一行）。
  - "英文" → v2.2 行为（把中文主题规划成英文检索式），报告一行。
  - "无所谓 / 都行" → **本次使用 `both`**（召回优先于精确），但不持久化；如果同一用户第二次回答"无所谓"，轻量提示一次可以把 `both` 设为默认，之后再不问。
  - 只有明确的"以后都…"才会持久化到 `config.search_language`（单次回答绝不自动持久化）。
- **如果本次同时触发了档位歧义（光秃秃的 "Q1"）**，把两个问题合并成**一条消息**——语言和平台一起问，绝不分成两轮（过度提问是一条红线）。
- 空间确定之后，按 `references/query_planner.md` §"Cross-language query handling" 措辞：**`zh` 保留中文检索词（不翻译）**，`en` 使用英文词（v2.2 行为），`both` 构建两套。

**把每条检索式写成概念块，而不是词表。** OpenAlex 要求每个裸词都出现，所以以词形式写出的同义词会排除掉那些只用了其中一个词的论文（用户一个 11 词检索式，五年间只靠标题和摘要命中了一篇）。同义词放进 OR 组，2–3 个概念块用 AND 连接，短语加引号——`("short video" OR TikTok OR Douyin) AND (attention OR concentration)`——并且至少让一个策略不包含人群块。📖 `references/query_planner.md` §"How OpenAlex reads a query"。

按领域套用 PICO / SPIDER / PEO：
- 医学 / 临床 → PICO（人群 Population / 干预 Intervention / 对照 Comparator / 结果 Outcome）
- 质性研究 → SPIDER
- 范围综述 → PEO（人群 Population / 暴露 Exposure / 结果 Outcome）
- 开放性主题 → 提取 2-4 个概念块，每块 2-5 个同义词

**期刊档位意图识别（增量——仅在检索式提到分区时才生效）。** 在提取概念块之前，检查用户的检索式是否带有期刊档位 / 分区措辞——"中科院一区"、"Q1"、"JCR Q1"、"SJR Q2"、"顶刊 / top journal"。如果有，该措辞是**过滤条件，不是检索词**，必须在检索前**从主题中剥离**。这正是催生整个特性的那个失败："中科院一区 情绪调节"过去会把"中科院一区"当作主题词送进搜索引擎，于是检索到的是*关于*中科院一区的论文，而不是*情绪调节*这个主题*并限定*中科院一区。确定性解析器会替你把两件事都做完（提取 + 剥离）：

```bash
PYTHONPATH=$PSP_HOME python3 -c "
from scripts.rank_intent import parse_rank_intent
i = parse_rank_intent('''<original user query>''')
import json; print(json.dumps({
  'platform': i.platform, 'tiers': i.tiers, 'quartiles': i.quartiles,
  'top': i.top, 'ambiguous': i.ambiguous, 'candidates': i.candidate_platforms,
  'cleaned_query': i.cleaned_query, 'stripped': i.matched}, ensure_ascii=False))
"
```

然后按解析结果行动：
- **`cleaned_query`** 是剥离档位后的检索输入——构建 STEP 3 检索式和检索计划时用它（不要用原始检索式）。它不自动等于 `SEARCH_TOPIC` 或报告标题：下面的语义规划步骤仍会移除对话性文本和操作性文本。当检索式里没有档位措辞时，`cleaned_query == query`，检索行为保持不变（R-19）。
- **`platform` + `tiers`/`quartiles`/`top`** 是你要在 STEP 5 之后立刻套用的过滤器——记住它们（并作为 `rank_filter` 记录进 `query_plan.json`）；此处不要过滤。
- **`ambiguous == True`**（有档位或分区但没有平台词，例如光秃秃的 "Q1" 或 "SSCI 一区"——识别器绝不猜平台）：**在继续之前就地向用户问一个简短问题**，给出 `candidates` 中的平台选项——*"按 JCR 还是 SJR 的 Q1 筛?"* / *"按中科院分区还是 JCR 分区?"*，再加一句*"顺便要不要设为以后的默认?"* CLI / headless 路径无法提问，所以这个就地问句是人类路径专属的职责。
- 如果检索式完全没提分区，跳过这一整段——STEP 1 的其余流程完全照旧。

**时间范围——在检索之前定好年份窗口。** 📖 `references/query_planner.md` §"Year filter heuristics"。把请求归入以下四种情形之一：
- **明确区间**（"2020–2024"、"近两年"、"2023 年以后"、"last 3 years"、"只要 2026 年的"）→ 设置 `year_min` / `year_max`；相对区间从当前年份往回算（"近两年" = 当前年份 − 1 … 当前年份）。不必提问。一句话：*"本次限定 2025–2026。"*
- **只说"新"没有数字**（"最新"、"最近"、"近年"、"前沿"、"recent"、"latest"、"cutting-edge"）→ 到底要多近确实有歧义，且会改变结果集。如果 `config.recent_years` 已设置，就用它并用一句话说明。否则**问一个问题**——与上面任何语言或档位问题同一条消息，绝不另起一轮：

  > 「最新」想要多近？近一年 / 近两年 / 近三年 / 近五年，或者直接给区间（如 2023–2026）。说「以后都这样」我就记成默认。

  只有明确的"以后都…"才会持久化（`recent_years: <N>` 写入 `~/.paper-search-pro/config.yaml`，机制与 `search_language` 相同）。
- **经典文献**（"经典"、"奠基"、"起源"、"seminal"、"classic"）→ 不设年份窗口；在 STEP 3 加入 `seminal`。
- **完全没提**（大多数请求）→ 不设年份窗口，并且**不要提问**：STEP 3 的近期支路仍会带来最近一年以及前两年中最相关的论文。一句话：*"没说时间范围：本次不限年份，并单独补一层近一年的新文献。只要近几年的，说「只要近两年」即可。"* 如果本来就要问语言或档位问题，可以在那条消息里一并给出时间选项。
- **Audit 档**：日期限制属于方案的一部分（PRISMA-S 会报告它）——把它并入你本来就要征得确认的 Audit 确认环节。

如果宿主提供结构化选项工具（Claude Code 的 AskUserQuestion；Codex 的 `request_user_input` 在可用列表中时），就用这些选项加上自由输入来问；否则用文字问。Agent / headless 路径从不提问——它直接采用给定的 `--year-min` / `--year-max`。

即使是 Quick 档也需要本步的轻量版——绝不能默默跳过。输出：1-3 条检索策略（概念块 + 年份范围 + 文献类型过滤）。写入 `"$SEARCH_DIR/query_plan.json"`，格式为 `{"strategies": [{"query": "..."}, ...], "year_min": …, "year_max": …, "rank_filter": {"platform": "cas", "tiers": [1, 2]}, "work_type_filter": …}`（其他键可以随意；不适用的省略即可）。报告的审计标签页和 PRISMA-S 日志（STEP 12a、13）会从中读取检索式与过滤条件。

**定义 `SEARCH_TOPIC`，但先不要定稿报告标题。** 阅读 `references/report_title.md` §"STEP 1"。`SEARCH_TOPIC` 是由概念块推导出的归一化语义检索范围，而不是把用户原话删掉几个正则匹配后的产物。剥离问候语、工具/输出指令、档位/语言/日期/数据库约束以及其他操作性指示；只有当人群、暴露/干预、结果、机制与场景本身是科学概念时才保留。另外把用户原始消息单独保存为 `USER_QUERY` 供 PRISMA-S 使用。最终的可视化标题在 STEP 11 撰写——那时保留的证据已经明确。

### STEP 2 —— 在 STEP 1 确定的语言空间内路由补充来源

📖 执行本步之前，先阅读：`references/source_routing.md`。

**这些路由调用由你根据检索式的学科判断来做——参考文件里的关键词表是校准示例，不是匹配清单**（机械事实——CJK 检测、显式 `--flags`——保持确定性）。在 STEP 1 固定的语言空间内，按学科路由各领域的增强源：

- **英文空间**（`en`，或 `both` 的英文半边）——与 v2.2 相同：
  - 医学信号（RCT、PRISMA、MeSH、clinical、疾病名）→ 启用 PubMed
  - CS / AI 主题（machine learning、LLM、transformer、NeurIPS、computer vision 等）→ 启用 arXiv 的近期层**以及** OpenReview（ICLR / NeurIPS / ICML / COLM / TMLR 评审平台——补充诸如 "ICLR 2026 Oral" 的录用标签，以及一些只有它才有的论文）
  - 时效性信号（preprint、"最新"、"this week"）→ 同时启用 4 天 arXiv 哨兵
  - 跨学科（例如 "AI in radiology"）→ 两者都启用
  - 纯社科 / 人文 → 只用 OpenAlex
  - *（判断题：AI/CS 核心检索式也可能把主引擎提升为 Semantic Scholar——见 `source_routing.md` §"AI / CS queries → consider Semantic Scholar as primary"。）*
- **中文空间**（`zh`，或 `both` 的中文半边）——同样按学科路由中文增强源：
  - 社科 / 人文信号 → 加入 **NSSD**（国家哲社文献中心；承载 OpenAlex 覆盖率近乎为零的 CSSCI 收录标识）
  - 医学信号 → 加入 **yiigle**（中华医学期刊全文数据库）；PubMed 仍覆盖 MEDLINE 收录的中华期刊，因此二者互补
  - 纯理工科且两者皆无 → 中文侧只跑 OpenAlex（科技类中文核心期刊大多注册了 DOI，OpenAlex 覆盖良好）

**报告一行**（坐标轴 3 的风格——是一句陈述，不是一个问题；22 §6.3）。纯英文运行时这就是 PubMed/arXiv 的提示（*"我检测到医学 + AI 信号——同时也检索了 PubMed、arXiv（近一年）和 OpenReview。可用 `--no-pubmed` 覆盖。"*）。中文空间例如：

> 本次按「中英都要」检索;中文侧检测到社科主题,已加 NSSD(国家哲社文献中心)。想去掉说 `--no-nssd`,只查一边说"只要英文/中文"。

**覆盖率的诚实与这条提示绑定：** 如果中文空间是社科主题但用户拒绝启用 NSSD，要补充说明 OpenAlex 在 CSSCI 旗舰期刊（经济研究 / 管理世界 等）上的命中近乎为零，因此这一层是缺失的。用户可以用明确指令覆盖以上任何一条（单次检索的显式覆盖优先级最高）。

**关于上面那些 `--flag` 简写（`--no-nssd`、`--no-pubmed`、`--source …`）：** 在这条人类路径上，它们是**自然语言覆盖的*记号***——一种紧凑写法，代替用户可以*说出口*的话（"去掉 NSSD" / "只查 OpenAlex"），由你（LLM）去解读。它们**不是可执行的 CLI flag**——这里没有任何脚本去解析它们。真正被脚本解析的 flag 只存在于 agent / headless 路径（`agent_search`），而那里的中文源控制是**需主动开启**：只有 `--with-nssd` / `--with-yiigle`（那里不存在 `--no-nssd` / `--source`）。见 `references/agent_mode.md`。

### STEP 3 —— 从 OpenAlex 深度检索

📖 执行本步之前，先阅读：`references/openalex_helper_cheatsheet.md`。

始终先跑 OpenAlex。（OpenAlex 是默认主源；`config.yaml` 中的 `primary_source` 可以改掉这一点——见 `references/source_routing.md` 的 *Primary source selection & quota fallback*。）

**当 OpenAlex 无法服务某次调用时**（日预算用尽、持续限流、宕机），`openalex_helper` 会把同一次调用改由 Semantic Scholar 提供，并打印相同的 JSON 结构——无需 flag，无需配置。它会在 stderr 打出一行以 `[paper-search-pro] OpenAlex unavailable` 开头的信息：用一句话告诉用户（规则 C）然后继续；不要停下来等。如果 Semantic Scholar 也服务不了，该调用的行为和 OpenAlex 出错时完全一样：退出码 `3`（stdout 仍是合法 JSON），stderr 那行会说明发生了什么以及 OpenAlex 何时重置——像处理任何其他 OpenAlex 错误那样报告给用户。`journal-list` 是例外：它照旧返回空结果且退出码为 0。

Standard 及以上档位，使用多策略深度抓取：

```bash
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.openalex_helper double-sort "<query>" \
    --n 50 \
    > "$SEARCH_DIR/raw/openalex.json"
```

三条支路，一个文件：**cited**（该主题被引最多的工作）、**recent**（最近 12 个月中最相关的工作，以及再往前两年）、**relevance**（整体最佳匹配）。近期支路才是找到今年论文的关键——相关性排序偏重引用，没有它新工作几乎不会出现。

Quick 档用相关性 + 更小的近期层：

```bash
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.openalex_helper search "<query>" \
    --limit 30 --recent 20 \
    > "$SEARCH_DIR/raw/openalex.json"
```

然后记录 OpenAlex 对主检索式持有多少成果——在报告年份图表下面加一行，这样偏薄的近期柱条不会被误读为整个领域偏薄（统计 STEP 1 时间窗内的标题+摘要匹配数，没有时间窗时取最近两年）：

```bash
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.openalex_helper count "<query>" [--year-min YYYY] [--year-max YYYY] \
    > "$SEARCH_DIR/pool_count.json"
```

如果 `count` 报告的成果少于 30 篇，或者辅助脚本警告检索式连续出现裸词，那么该检索式过窄（除非主题本身就很小众）：按 STEP 1 的方式重新分组并重跑检索，然后再进入分级。

也要把 STEP 1 的时间窗加到检索命令上：`--year-min YYYY` / `--year-max YYYY`（含端点）。没有给窗口就不加（辅助脚本自己会把上限截到当前年份）。近期支路始终留在窗口内：对于一个终点在过去的时间窗，它取该窗口的最后一年。

**`raw/` 只存放检索输出——一次检索一个文件，保持辅助脚本写出的原样。** STEP 7 依据这些彼此独立的检索重复找到同一批论文的频率来估算覆盖率。你派生出来的子集、合并结果或筛查副本应放进 `"$SEARCH_DIR/_working/"`，而不是 `raw/`：放在 `raw/` 里看起来就像又做了一次检索并重新找到了全部内容。

完整的子命令 + flag 参考（`search` / `double-sort` / `seminal` / `reviews` / `journal-list` / `citation-network`，均已对照 argparse 核验）在 `references/openalex_helper_cheatsheet.md` 里——在用到上面两条命令之外的任何东西之前先读它。对于 Deep 及以上档位，还要调用针对具体主题的子命令（例如 `seminal`、`reviews`、`journal-list`），把输出追加到 `$SEARCH_DIR/raw/openalex_*.json`，并在 STEP 5 中把它们一起联邦合并。

### STEP 4 —— 运行 L2 增强源（若 STEP 2 已启用）

📖 执行本步之前，先阅读：`references/pubmed_helper_cheatsheet.md` 和 `references/arxiv_helper_cheatsheet.md`。

**PubMed —— 默认模式是 `enrich`，而不是 `search`**：
- **Standard / Deep 档**：用 MeSH 主题词增强 OA 找到的论文（原地改写 openalex.json 文件）：
  ```bash
  PYTHONPATH=$PSP_HOME \
    python3 -m scripts.pubmed_helper enrich \
      --input-file "$SEARCH_DIR/raw/openalex.json" \
      --output-file "$SEARCH_DIR/raw/openalex.json"
  ```
- **Audit 档且有显式 MeSH 检索式**：独立 MeSH 检索（产出一个新文件，稍后联邦合并）：
  ```bash
  PYTHONPATH=$PSP_HOME \
    python3 -m scripts.pubmed_helper search-mesh "Diabetes Mellitus, Type 2" \
      --year-min 2020 --limit 30 --pub-type "Randomized Controlled Trial" \
      > "$SEARCH_DIR/raw/pubmed.json"
  ```
- 通用的 `pubmed_helper search` 是没有已知 MeSH 词时的兜底——尽可能优先用 `enrich` 或 `search-mesh`。

**arXiv —— CS / AI 主题：近期层。** 最近一年最相关的预印本（OpenAlex 对 arXiv 的索引既晚又不全：一次核查中，OpenReview 上 20 篇在题近作里有 7 篇不在 OpenAlex 中）：

```bash
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.arxiv_helper search "<query>" \
    --sort relevance --since-days 365 --limit 30 \
    > "$SEARCH_DIR/raw/arxiv.json"
```

**时效性信号（"最新"、preprint、"this week"）—— 同时使用 4 天哨兵**，覆盖新到任何索引都还没收录的预印本：

```bash
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.arxiv_helper freshness "<query>" \
    --days 4 --limit 30 \
    > "$SEARCH_DIR/raw/arxiv_fresh.json"
```

检索式里的普通词是全部必须满足的（arXiv 本来会 OR 它们）。两条命令在 arXiv 失败或超时时都会打印 `[]` 和一行 stderr 提示——继续即可（规则 C：一句话说一声）。

子命令参考：
- `arxiv_helper freshness <query> --days N --limit M [--all-cats]`
- `arxiv_helper search <query> --limit M --sort submitted|relevance|lastUpdated [--since-days N] [--all-cats]`
- `arxiv_helper get <arxiv_id>`

**OpenReview —— 仅限 CS / AI 主题。** 📖 `references/openreview_helper_cheatsheet.md`。ICLR、NeurIPS、ICML、COLM 和 TMLR 的评审平台。辅助脚本只返回已录用论文，并标注类似 "ICLR 2026 Oral" 的标签——这是 OpenAlex 与 Semantic Scholar 没有的标签（它们把这些列为 arXiv 预印本）。STEP 5 会把它并入同一篇论文的 arXiv / OpenAlex 记录。

```bash
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.openreview_helper --search "<query>" --n 30 \
    [--year-min YYYY] [--year-max YYYY] \
    --output-file "$SEARCH_DIR/raw/openreview.json"
```

只有它的检索端点允许匿名使用；如果 OpenReview 拦截，辅助脚本会返回 `[]`——用一句话说明并继续。

**NSSD / yiigle —— 中文增强源（仅当 STEP 1-2 把本次运行放进 `zh` 空间，且只启用学科路由选中的那一个或几个）：**

两者都是中文空间里独立的主源（角色与 `ss_helper --search` 相同），输出与 openalex.json **相同的 `UnifiedPaperEntity` 结构**，因此 STEP 5 的联邦合并方式完全相同。检索式保持**中文**（不要翻译——见 query_planner §Cross-language），写入 `raw/nssd.json` / `raw/yiigle.json`：

```bash
# NSSD —— 中国人文社科（补上 OpenAlex 缺少的 CSSCI 层级）
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.nssd_helper --search "<中文检索式>" \
    --n 50 [--year-min YYYY] \
    > "$SEARCH_DIR/raw/nssd.json"

# yiigle —— 中文医学（中华医学期刊全文数据库；原生中文标题 + 摘要）
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.yiigle_helper --search "<中文检索式>" \
    --n 50 [--year-min YYYY] \
    > "$SEARCH_DIR/raw/yiigle.json"
```

两者在任何网络 / HTTP 失败时都会优雅降级为 `[]`（永不抛异常）——空文件联邦合并后什么都不产生。两者**只**取题录 + 摘要（合规：不下载全文、不缓存），并把来源归属打印到 stderr。`--year-min` 是客户端过滤；想去掉它就保留全部年份。

**config 中 `search_language: en` 但来了中文检索式（硬边界 2）：** 不要启用中文增强源，但要说一句话——绝不做静默翻译（22 §6.4）：

> 按你的默认(只查英文),我把中文主题规划成英文检索式了。想要中文文献这次说一声即可,想改默认说"以后…"。

**用户点名 知网 / CNKI / 万方 / 维普（标记命中 + 合规）：** 这些是 PSP 不抓取的封闭订阅库。说一句话，提供替代方案 + 导出，不要反复争论（22 §6.5）：

> PSP 不抓知网/万方（合规，不做封闭库抓取）；中文侧用 OpenAlex 中文底座 + NSSD(社科,含 CSSCI 标识)/yiigle(医学)覆盖。但可以**把你的知网专业检索式写好**——按 CNKI 专业检索语法（`SU %=` 相关匹配、块内 OR/块间 AND、全英文半角、检索值单引号）生成可直接**粘贴进「专业检索」框**的检索式，粘贴即查。中文医学题另可给 SinoMed（CMeSH 受控词双轨）。要导出吗？

"要导出吗?" 被接受 = STEP 11.5 的显式触发条件——任何档位都执行该导出。

### STEP 5 —— 联邦合并（去重 + 合并）

📖 执行本步之前，先阅读：`references/source_routing.md` §"Field priority table"。

把全部检索结果合并成一个去重后的知识图谱。**默认输出是以 canonical_key 为键的 dict**——后面 `rcs_parser` 期望的就是这个，所以**不要**传 `--as-list`：

```bash
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.federated_kg_resolver \
    --input-files "$SEARCH_DIR/raw/openalex.json" \
                  "$SEARCH_DIR/raw/pubmed.json" \
                  "$SEARCH_DIR/raw/arxiv.json" \
                  "$SEARCH_DIR/raw/openreview.json" \
    --output "$SEARCH_DIR/kg.json"
```

只传入你真正产出的输入文件——跳过 STEP 2 未启用的那些（4 天哨兵跑过则加上 `raw/arxiv_fresh.json`）。OpenReview 记录没有 DOI；解析器会按标题把它并入同一篇论文的 arXiv / OpenAlex 记录，并给它加上录用会议标签。**如果 STEP 4 运行了中文增强源，把 `"$SEARCH_DIR/raw/nssd.json"` / `"$SEARCH_DIR/raw/yiigle.json"` 加进同一个 `--input-files` 列表**——它们带着完全相同的实体结构，联邦合并方式与其他源一模一样（CJK 安全的去重由 Phase 0 的 canonical-key 修复处理，因此不同的中文标题不会被折叠到一起）。对于纯英文运行时这些文件不存在，所以该调用与 v2.2 逐字节相同（R-19）。它处理 DOI 规范化（arXiv X→x 大小写）、版本号剥离、E5b 防护（同标题同年份但 DOI 不同的会被分开保留），以及字段优先级合并。

`--as-list` 存在，但只给需要按 citation_count 排序的列表型消费方使用；不要在本流程中使用它。

**仅当 STEP 1 发现了期刊档位请求时——在分级之前过滤。** 档位是廉价的确定性过滤；给即将被丢弃的论文做分级才是昂贵的部分。📖 `references/journal_metrics.md`。保留 `kg.json` 作为完整文献池，对 `kg_ranked.json` 做分级：

```bash
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.rank_filter --kg "$SEARCH_DIR/kg.json" \
    --output "$SEARCH_DIR/kg_ranked.json" --platform cas --tiers 1,2
# JCR / SJR: --platform jcr --quartiles Q1,Q2 ; 只要顶刊: --top
```

它用全部三个平台给每篇论文打标签，保留你要求的那些，其余写入 `kg_ranked_set_aside.json`。用一行把打印出的计数告诉用户（*"本次按中科院 1–2 区筛：留 N / 区外 M / 不在该表 K"*），并附上它打印的归属说明。在 STEP 6 批量处理并合并 `kg_ranked.json` 而不是 `kg.json`。没有要求档位 → 整段跳过；不会有任何变化。

### STEP 6 —— 并行分批分级（LLM 就在这里——主 Agent + SubAgent）

📖 执行本步之前，先阅读：`references/classifier_subagent_prompt.md` 和 `references/rcs_rubric.md`。

把 KG 切成每批 10 篇。写入 `"$SEARCH_DIR/batches/batch_NNN.jsonl"`。如果文献池超出该档位可分级的量，要跨原始文件**均匀**裁剪，绝不按引用数裁剪：新论文几乎没有引用，按引用砍掉掉的正好是近期层。

**派发之前**，把 `$PSP_HOME/references/rcs_rubric.md` 展开成实际绝对路径（例如 `/Users/alice/.claude/skills/paper-search-pro/references/rcs_rubric.md`），并用它替换分类器提示词模板里的 `{rubric_path}`。每个 SubAgent 都在自己的 shell 中运行，那里 `$PSP_HOME` **没有**被导出——传字面量 `$PSP_HOME` token 会让 SubAgent 找不到评分准则，从而静默地拉低打分质量。完整的占位符对照表见 `references/classifier_subagent_prompt.md`。

🔥 **并行是强制的**（规则 B）：

你必须在**一条 assistant 消息中**用多个 `Task` tool_use 块派发最多 **5 个分类 SubAgent**。串行派发（一条消息一个 Task、逐个等待结果）是观察到的最大性能失败——它会把 Standard 档从约 10 分钟拉长到约 17 分钟。

✅ **正确——在同一条 assistant 消息里：**

```
Task tool_use #1  → subagent_type="general-purpose", prompt="<batch_001.jsonl 的分类器提示词>"
Task tool_use #2  → subagent_type="general-purpose", prompt="<batch_002.jsonl 的分类器提示词>"
Task tool_use #3  → subagent_type="general-purpose", prompt="<batch_003.jsonl 的分类器提示词>"
Task tool_use #4  → subagent_type="general-purpose", prompt="<batch_004.jsonl 的分类器提示词>"
Task tool_use #5  → subagent_type="general-purpose", prompt="<batch_005.jsonl 的分类器提示词>"
```

五个 tool_use 块位于同一条 `<assistant>` 消息中，因此并行运行。有些宿主会一次性返回五个结果；有些会在后台启动 SubAgent 并在各自完成时报告。无论哪种，都要等这一轮的每个 `batch_NNN_result.json` 都存在之后再合并。

❌ **错误——五条独立消息（串行派发就是这个样子）：**

```
消息 N:    Task tool_use #1 ─→ 等待结果
消息 N+1:  Task tool_use #2 ─→ 等待结果   ← 串行，让 Standard 档慢 70%
消息 N+2:  Task tool_use #3 ─→ 等待结果
...
```

如果批次超过 5 个，就每条消息发 5 个，分多条消息发——每条消息仍然是 5 个并行的 Task 块。

每个 SubAgent 读取自己的批次文件，应用 RCS 评分准则，写出 `"$SEARCH_DIR/classifications/batch_NNN_result.json"`。然后把分级结果合并进 KG：

```bash
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.rcs_parser \
    --input-dir "$SEARCH_DIR/classifications/" \
    --kg "$SEARCH_DIR/kg.json" \
    --output "$SEARCH_DIR/kg_classified.json"
```

### STEP 7 —— 计算饱和度曲线（所有档级均为必做）

📖 执行本步之前，先阅读：`references/stop_decision.md`。

本步**不是**可选项，即使 Quick 档也是：`curve.json` 正是 STEP 8 的决策依据，而 STEP 12 渲染的是同一个估计值。

```bash
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.discovery_curve \
    --kg "$SEARCH_DIR/kg_classified.json" \
    --output "$SEARCH_DIR/curve.json"
```

`curve.json` 给出 `coverage_estimate`（0-1）以及 `ci_lower` / `ci_upper`，还有 `method`：`sample_coverage` 表示它由本次运行中 `raw/` 里的多次独立检索重复找到同一批高相关论文的频率估算而来（`relevance_band` 7 = rcs ≥ 7；6 = rcs ≥ 6，用于 rcs ≥ 7 的论文太少、估计器说不准时）；`prior` 表示无可测对象——少于两次检索、没有找到相关论文，或任一带内论文过少——因此它是取自 Undermind 中位数先验并封顶 50% 的值，而不是测量值——如果你要引用它，务必说明这一点。`--raw-dir` 可覆盖检索结果的读取位置（默认是 `--kg` 旁边的 `raw/`）。

### STEP 8 —— 决定下一步动作（必做）

📖 执行本步之前，先阅读：`references/stop_decision.md`。

本步**不是**可选的。必须**显式**做出决定——依据 curve.json + 档位预算 + 意图——并把推理过程告诉用户。不要凭直觉跳过。

决策树（依据 `coverage_estimate`）：
- coverage < 0.6 且仍有预算 且档位属于 {standard, deep, audit} → 扩展引文（STEP 9）
- coverage > 0.85 或预算耗尽 → 停止，开始写报告（STEP 10+）
- 情况不明确 → 把数字告诉用户并询问

### STEP 9 —— 扩展引文（如适用）

📖 执行本步之前，先阅读：`references/citation_chasing.md`。

对 rcs 最高的论文（rcs >= 7），获取引文网络。种子有 DOI 时就传 DOI：DOI 在 Semantic Scholar 兜底方案上同样有效，而裸的 W-ID 在 OpenAlex 之外无法解析。

```bash
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.openalex_helper citation-network <doi_or_openalex_id> \
    --refs-limit 25 --cited-by-limit 25 \
    > "$SEARCH_DIR/raw/citations_01.json"      # 每个种子一个文件：citations_02.json, ...
```

然后回到 STEP 5：把 `raw/citations_*.json` 文件加入 `--input-files`（解析器会读取它们的 `references` / `cited_by` 列表），在 STEP 6 只对新增条目重新分级，并重跑 STEP 7。

### STEP 10 —— 丰富 top-N 论文（L3，可选但推荐）

📖 执行本步之前，先阅读：`references/ss_helper_cheatsheet.md` 和 `references/crossref_helper_cheatsheet.md`。

对 rcs >= 6 的论文，用 Semantic Scholar 丰富信息（influentialCitationCount + 摘要兜底 + tldr）。两个辅助脚本都读取已分级的 KG 并把新字段写回其中，其他字段保持原样；`--min-rcs 6` 把它们限制在最重要的论文上。要在 STEP 11 之前跑，这样摘要才能按 `influential_citation_count` 排序。

对 Quick 档，跳过 STEP 10 是可以接受的——但要**按规则 C 声明这次跳过**（"Skipped L3 enrichment → no influentialCitationCount; re-run at `--tier standard` to include this"）。

```bash
# Semantic Scholar —— 增加 influentialCitationCount + 摘要兜底 + tldr
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.ss_helper \
    --input-file "$SEARCH_DIR/kg_classified.json" \
    --mode enrich --min-rcs 6 \
    --output-file "$SEARCH_DIR/kg_classified.json"
```

它会打印一行 stderr 说明丰富了多少篇论文；如果 Semantic Scholar 拒绝了（key 被拒、限流），也会说明原因。数量偏少时把那句话转达给用户；无论哪种情况运行都会继续。

**CrossRef（资助方 / 许可协议 / 临床试验编号）只在这些字段重要时才跑**——临床或资助类问题，或者用户主动要求时：报告并不展示这些字段。

```bash
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.crossref_helper \
    --input-file "$SEARCH_DIR/kg_classified.json" \
    --mode all --min-rcs 6 \
    --output-file "$SEARCH_DIR/kg_classified.json"
```

**可选（增量）—— 期刊分区（中科院 / JCR / SJR）。**
多平台分区层会给每篇论文打上**全部三个**平台的标签。它是**需主动开启、默认关闭**——跳过它，报告逐字节不变（R-19）。📖 先阅读 `references/journal_metrics.md`（它是来源、ISSN 关联、归属说明和 R-04 命名的唯一事实来源）。

- **首次使用需要一次性抓取**（init-once；数据在运行时拉取到
  `~/.paper-search-pro/ranks/`，**绝不打包进仓库**）。如果你之前没有抓过，跑一次（并告诉用户这是一次性步骤）：
  ```bash
  PYTHONPATH=$PSP_HOME python3 -m scripts.journal_rank fetch          # 全部三个平台
  # 或只抓单个平台: ... journal_rank fetch --platform cas
  PYTHONPATH=$PSP_HOME python3 -m scripts.journal_rank info           # 已缓存了什么
  ```
- **用户要求了档位** → 论文在 STEP 5 之后就已经打好标签并过滤过了；这里不用做什么。
- **没要求档位，但用户想看分区** → 只打标签（不过滤）：
  ```bash
  PYTHONPATH=$PSP_HOME python3 -m scripts.rank_filter \
    --kg "$SEARCH_DIR/kg_classified.json" --output "$SEARCH_DIR/kg_classified.json"
  ```
  本地没有缓存时它会打印一行提示且不做任何改变；告诉用户可以运行 `journal_rank fetch` 来启用分区。
- **R-04 命名**已在序列化 dict 中为你强制执行：只有 JCR 暴露 `impact_factor`（真正的 IF）；中科院"区"和 SJR quartile 是**分区/quartile**。

### STEP 11 —— 撰写执行摘要

📖 执行本步之前，先阅读：`references/summary_writer.md`。

📖 同时阅读：`references/report_title.md`，然后在摘要之外一并撰写 `DISPLAY_TITLE`。这是一个**有学术质感、受证据约束的报告标题**，基于保留下来的文献集定稿——绝不是 `USER_QUERY`、`SEARCH_TOPIC` 或过滤条件列表的复制。它承载报告的核心叙事与视觉识别。语言、日期、数据库、期刊档位、导出与工作流要求应留在 Methods / PRISMA-S 中，除非它们本身具有科学意义。如果证据不足以支撑某个具体标题，就用本地化的安全兜底（`文献检索报告` / `Literature Search Report`），而不是把原始请求放进 H1。在 STEP 12 之前先跑参考文件里的渲染前验收检查。

基于已分级论文，用你自己的话写一份约 300 字的执行摘要：
- 该领域的主要共识
- 关键方法 / 理论框架
- 值得注意的分歧或开放问题
- 最有影响力的 3-5 篇论文（有 `influential_citation_count` 时按它排序）
- *（可选）* 主要论文的期刊档位，**仅当**你在 STEP 10 附上了 SJR 指标——措辞用 "SJR分区 / 期刊影响力"，绝不用 "影响因子 / JCR"（R-04）。没有附任何指标时整条略过。

保存到 `"$SEARCH_DIR/summary.md"`。

**分区的 默认 / 询问 / 过滤 / 报告 / 切换 流程（增量——仅在涉及分区时启用）。** 当期刊分区被涉及时（STEP 5 之后过滤过，或 STEP 10 中打过标签），遵循以下流程；它完全是可选的，在默认的无分区路径上不改变任何东西（R-19）：

- **出厂默认标准 = JCR。** 持久化默认值存放在 config 的 `rank.default_platform`（开箱即用为 `jcr`；用户可以改成 `cas`/`sjr`）。默认平台只**给**每篇论文**打标签**——除非用户明确要求某个档位，它并**不**过滤。
- **没提分区 → 不过滤。** 只展示三个平台的标签（STEP 10 的标注已经做了），让用户自行阅读 / 微调。绝不发明用户没要求的档位过滤。
- **用户要求了档位（来自 STEP 1 的意图，或本轮用户提出）→ 在分级之前过滤一次**（紧接 STEP 5 之后，用 `rank_filter`）。使用 STEP 1 的解析结果：`platform` + `tiers`/`quartiles`/`top`。单次请求的档位过滤是**临时**的——绝不自动持久化到 config。持久化默认只会在用户明确说"以后都用 X"时才改变。
- **光秃秃的 "Q1" 有歧义，既没有平台也没有持久化默认 → 问一个简短问题**（你本应在 STEP 1 就问过；如果没问，现在问）：*"按 JCR 还是 SJR?顺带设默认吗?"* 小确认是受欢迎的，但不要过度提问。
- **永远要报告本次运行做了什么。** 过滤之后，用一行告诉用户：*"本次按 {platform} 筛(留 N / 滤 M"*，再加一句轻量提议：*"可换中科院/JCR/SJR 或换档位、可设为以后的默认。"* 要带上各平台的归属说明（`journal_rank.ATTRIBUTION[platform]`）。
- **切换标准或档位 = 重新过滤文献池，而不是重新检索。** 当用户接着说"换成中科院二区"或"看看 SJR Q1"时，**不要**重跑检索：改用新的平台 / 档位在完整文献池 `kg.json` 上重跑 `rank_filter`（只需几秒）。新纳入的论文还没有分数——只对它们分级（STEP 6），然后在所有 `classifications/` 上重跑 `rcs_parser`，带上 `--kg kg_ranked.json`。**只有当重新过滤后存活论文太少时**，才回到 STEP 3 加深检索。这条"切换=重筛不重搜"的规则，正是让分区探索变得廉价的关键。
- **持久化默认值**（仅在用户明确说"以后都用 X"时）：在 `~/.paper-search-pro/config.yaml` 中设置 `rank.default_platform`。档位本身永不持久化——只有平台默认值会。
- **摘要那条要点里的 R-04 命名同样适用：** 中科院"区"和 SJR quartile 是**分区 / quartile**；只有 JCR IF(2024) 才是**影响因子 / Impact Factor**。OpenAlex 的 2 年均被引数是"期刊影响力"（开放口径），绝不是 JIF。

### STEP 11.5 —— 导出检索式（需主动开启——默认路径上绝不执行）

**触发矩阵——在做任何其他事之前默默决定：**

| 运行 | 导出行为 |
|------|-----------------|
| Quick | **不执行。不要提，不要问**——直接进入 STEP 12。这次静默跳过是规则 C 的唯一例外：宣布它本身就会打扰未被动过的默认路径（R-19）。 |
| Standard | **检索过程中不执行**——不提问，直接进入 STEP 12。**到 STEP 14 时，只有当本次运行存在导出能填补的缺口才用一句话提议：** 检索空间包含中文（`zh` / `both`：知网 / 万方 掌握着 PSP 触达不到的大部分内容），或 `coverage_estimate` < 0.6（其他数据库可以补上）。没有这类缺口就什么都别说。如果 STEP 4 本次已经提过就不要重复。被接受 → 执行本步，然后重跑 12a–12c，让报告多出导出标签页。 |
| Deep | 提议一次，一句轻量的话：*"要不要顺带导出各平台可粘贴的专业检索式?"* 被接受 → 执行；被拒绝或未回应 → 跳过且不再提。 |
| Audit | **默认执行**——它是 Audit 免责声明所承诺交付物的实现（示例 4）。 |
| 任何档位的显式要求 | "给我 WOS 检索式" / "我要去知网查" / "导出检索式" / STEP 4 中的知网提议被接受 → 立即执行，无论档位。 |

📖 执行本步之前（当它被触发时），先阅读：`references/search_export/methodology.md`——它是下面每一处语义判断的规则手册。只要 zh/both 空间要导出 CNKI / 万方 / SinoMed，就再加上 `references/search_export/chinese_methodology.md`（中文检索式不是西方布尔式的逐词翻译）。

**第 1 层——语义。这是你的活（主 Agent）；脚本只负责拼装语法：**

1. **重建概念块**——从 `"$SEARCH_DIR/query_plan.json"` 出发，遵循 methodology.md 的分块纪律：块内 OR 只放同一个概念的同义词；块之间用 AND 连接；默认 2 个块（P+I），C/O 只在标题 / 摘要中处于核心且被一致报告时才加；不禁止。**不要照抄 query_plan 的块**：这次重建正是修正"混合概念块"的地方。
2. **按块扩展同义词与入口词**——拼写变体（tumour/tumor）、缩写 + 全称（MBSR + "mindfulness-based stress reduction"）、相关词。**在 zh / both 运行中，还要为每个块填好 `free_text_zh`**（简繁 / 别名 / 全称–缩略变体——中文没有截断，所以要显式枚举）：CJK 宿主库（CNKI / 万方 / SinoMed）**只读** `free_text_zh`，缺了它的块会让所有中文检索式被诚实地扣住不发——英文词绝不会被渲染进中文数据库。
3. **各平台的受控词候选**——逐库独立判断 MeSH / Emtree / CINAHL / APA / CMeSH；绝不在一个词表之间符号式地翻译到另一个。优先使用 STEP 4 已经回填进 KG 的 MeSH 词（来自真实命中结果的真实描述词）。
4. **按学科裁剪平台集合**——从 Audit 全集（PubMed、WOS、Scopus、Embase、Cochrane CENTRAL、PsycINFO、CINAHL、ERIC、IEEE Xplore、ACM DL、EconLit、CNKI、万方、SinoMed、ClinicalTrials.gov）中挑选——医学主题不需要 IEEE/ACM；社科主题不需要 Cochrane/SinoMed。**zh / both 运行总是包含 CNKI + 万方 + SinoMed。**
5. **register 随档位而定**（audit = 偏召回，受控词 + 自由词双轨）。需要研究设计过滤器（例如 RCT）？📖 `references/search_export/filters_library.md`——逐字使用现成的、经过验证的过滤器；绝不自己写，也绝不在 CENTRAL 上加。
6. **在生成任何语法之前做 PRESS 六域自检。** 📖 `references/search_export/press_checklist.md`。

把结果写入 `"$SEARCH_DIR/concept_model.json"`，采用生成器所需的包装结构：`{topic, framework, register, language_space, search_id, platforms, concept_model: {blocks (free_text + zh/both 下的 free_text_zh + controlled_vocab_candidates), operator_logic, omitted_blocks + reasons, register_notes}}`。**`omitted_blocks.reason` 和 `register_notes` 要用用户的语言书写**——它们会原样出现在面向用户的 `search_strategies.md` 中（说中文的用户拿到的是中文说明，而不是英文直译）。

**第 2、3 层——机械拼装 + 校验（脚本的活；绝不为不同宿主手工拼语法）：**

```bash
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.search_export.generate \
    --concept-model "$SEARCH_DIR/concept_model.json" \
    --out "$SEARCH_DIR"
```

可选 flag（已对照 argparse 核验）：`--platforms pubmed,wos,cnki`（csv）覆盖包装结构中的 `platforms`；`--topic` / `--framework` / `--register` / `--language-space` / `--search-id` 覆盖运行级字段；`--no-verify-vocab` 跳过 MeSH/ERIC 免费 API 检查（词条会被标为未核验）；`--live-verify-links` 实时 GET 每个 A 级链接以获得新鲜的 `verified_http`/日期。

它输出 `search_strategies.md`（按平台划分、可直接粘贴）+ `search_strategies.json`。底层它会读取各宿主语法卡（`references/search_export/syntax_cards/<host>.md`）和邻近算子转换表（n 值绝不手工计算——你也不手动算），只为免费的 A 级平台构建可点击的深链接（PubMed / ERIC / ClinicalTrials.gov；绝不生成任何绕过登录或验证码墙的链接），通过 API 核验 MeSH/ERIC 词条，把 Emtree/CINAHL/APA/CMeSH 标记为待人工核对，并对每条检索式做 lint——**被 linter 拒绝的字符串会被扣住，绝不输出**。下游衔接：STEP 12a 会自动发现 `report_data.json` 旁边的 `search_strategies.json` 并折叠进去（无需额外 flag）；STEP 13 则在 `prisma_s_logger` 命令上追加 `--search-strategies "$SEARCH_DIR/search_strategies.json"`，让 PRISMA-S 第 8/1/9/10 项从中得到充实。

**交付前请检查**（诚实性关口）：`.json` 中带有每个平台的三态校验标签（🟩机械已验 / 🟨语法已验·词表待核 / 🟦结构参考）和复核点——核实它们是否正确；面向用户的 `.md` 把这些渲染成各平台通俗的"使用前请核对"说明（不含内部术语）——核实每条说明都清晰可执行；订阅墙平台被诚实地标为"需机构登录，未在墙内实测"；CNKI / 万方 / SinoMed 以"粘贴进检索框"的检索式交付，而不是 URL。把质量主张保持在"专业初稿 + 标注复核点"——绝不说"可署名直用 / 馆员级"。

### STEP 12 —— 渲染报告

📖 执行本步之前，先阅读：`references/output_files.md`。

```bash
# 12a. 为渲染器准备数据（同时写出同级的 chart_data / paper_list / metadata / prisma_log）
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.data_materialization \
    --kg "$SEARCH_DIR/kg_classified.json" \
    --summary "$SEARCH_DIR/summary.md" \
    --query "<original query>" \
    --search-topic "$SEARCH_TOPIC" \
    --display-title "$DISPLAY_TITLE" \
    --language "$UI_LANG" \
    --tier "<quick|standard|deep|audit>" \
    --search-id "$SEARCH_ID" \
    --snapshots "$SEARCH_DIR/curve.json" \
    --output "$SEARCH_DIR/report_data.json"

# 12b. 渲染 HTML（Shadcn webartifacts —— 唯一渲染器；无体积上限）
#      --language $UI_LANG 选择 EN 还是 ZH 界面；内置包同时带两套
#      词典，$UI_LANG 只决定挂载哪一套。渲染器内部的解析
#      顺序是：显式 --language > metadata.language > en。
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.html_renderer_webartifacts \
    --data "$SEARCH_DIR/report_data.json" \
    --output "$SEARCH_DIR/report.html" \
    --query "<original query>" \
    --search-topic "$SEARCH_TOPIC" \
    --display-title "$DISPLAY_TITLE" \
    --language "$UI_LANG"
#     本次按某个平台过滤过时，加上 --rank-platform cas|jcr|sjr，
#     这样徽章和分区筛选器会显示该平台（默认：JCR）

# 12c. Markdown 报告（用物化目录提速）
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.md_report \
    --materialized-dir "$SEARCH_DIR" \
    --summary "$SEARCH_DIR/summary.md" \
    --query "<original query>" \
    --search-topic "$SEARCH_TOPIC" \
    --display-title "$DISPLAY_TITLE" \
    --language "$UI_LANG" \
    --tier "<quick|standard|deep|audit>" \
    --output "$SEARCH_DIR/report.md"

# 12d. 导出（BibTeX / RIS / CSV / papers.json —— 默认只含 rcs >= 5）
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.generate_exports \
    --kg "$SEARCH_DIR/kg_classified.json" \
    --output-dir "$SEARCH_DIR/" \
    --min-rcs 5
```

如果 12a–12c 打印 `No scholarly display title was given`，说明 `DISPLAY_TITLE` 为空：报告 H1 是通用兜底。补上标题（STEP 11，`references/report_title.md`）并重跑 12a–12c；不需要重新检索。

`data_materialization` 接受 `--wall-clock-seconds` 参数（如果你自己记录了耗时）；否则在可获得时会话时间戳时由辅助脚本自行计算。

### STEP 13 —— 写入 PRISMA-S 日志

📖 执行本步之前，先阅读：`references/prisma_s_checklist.md`。

```bash
PYTHONPATH=$PSP_HOME \
  python3 -m scripts.prisma_s_logger \
    --search-id "$SEARCH_ID" \
    --kg "$SEARCH_DIR/kg_classified.json" \
    --user-query "<original query>" \
    --tier "<quick|standard|deep|audit>" \
    --query-plan "$SEARCH_DIR/query_plan.json" \
    --snapshots "$SEARCH_DIR/curve.json" \
    --output "$SEARCH_DIR/execution_log.json"
```

它记录 PRISMA-S 的 16 个条目，用于透明性 / 可审计性。

### STEP 14 —— 打开报告并向用户汇报

**首先**，在用户的默认浏览器中自动打开 HTML 报告（不要等用户开口）。区分平台的 Bash：

```bash
# macOS —— 最常见的开发环境
open "$SEARCH_DIR/report.html"
# Linux 回退 —— xdg-open "$SEARCH_DIR/report.html"
# Windows 回退 —— start "" "$SEARCH_DIR/report.html"
```

macOS 上默认用 `open`。如果失败（很少见——只有纯 Linux 容器），依次回退到 `xdg-open` 然后 `start`。**绝不要跳过这一步**——用户刚等了 5-30 分钟才拿到报告，他们应当第一时间就看到它。

**然后**告诉用户：
- "已在你的默认浏览器中打开报告。"（1 行确认）
- 报告在磁盘上的位置（绝对路径：`$SEARCH_DIR/report.html`）——方便用户之后找到
- 主要发现（从执行摘要中提炼 3-5 句）
- 任何注意事项——包括你按规则 C 跳过的步骤（例如"未检测到医学信号，因此没有查询 PubMed"、"因为是 Quick 档所以跳过了 STEP 10 的 L3 丰富；改用 standard 档重跑可包含资助方 / 许可字段"）
- 仅当 STEP 11.5 的 Standard 行发现有缺口时：用一句话提议导出检索式，点明缺口与成本——例如 *"这次中文侧主要缺知网、万方的文献；需要的话我把这两个库的专业检索式写好（约 3–4 分钟），粘进检索框就能查。"* 或 *"估算覆盖率 45%，还有不少没找到；需要的话我把 WOS、Scopus、PubMed 的检索式写好（约 3–4 分钟），你拿去补检。"*

---

## 输出约定

📖 完整的目录结构见 `references/output_files.md`。下面所有路径都是**相对于用户的工作目录（PWD）**——绝不是 Skill 资源目录。

```
$(pwd)/paper-search-results/<search_id>/
├── report.html              # 主交付物（Shadcn 风格）
├── report.md                # Markdown 副本
├── papers.csv               # 表格导出
├── papers.bib               # 引文管理器导入（BibTeX）
├── papers.ris               # 备用引文格式
├── papers.json              # 完整结构化数据
├── kg_classified.json       # 带 RCS 分数的内部 KG
├── summary.md               # 你的执行摘要
├── metadata.json            # 含 original_user_query / search_topic / display_title
├── execution_log.json       # PRISMA-S 16 项日志
├── report_data.json         # 渲染器数据包
├── chart_data.json          # 同级：图表数据系列
├── paper_list.json          # 同级：逐篇论文列表
├── prisma_log.json          # 同级：PRISMA 日志 JSON 视图
├── curve.json               # 覆盖率快照（STEP 7）
├── query_plan.json          # STEP 1 输出
├── raw/                     # 仅检索输出，一次检索一个文件（openalex.json、pubmed.json、citations_01.json、…）
├── batches/                 # batch_NNN.jsonl 文件
└── classifications/         # batch_NNN_result.json 文件
```

---

## 错误处理

📖 常见情况见 `references/error_handling.md`：

| 错误 | 应对 |
|-------|-----------|
| 配置缺少 key | 引导用户查看 `references/setup.md`，停止 |
| 限流（SS 429 / NCBI 429） | 辅助脚本会自动重试；持续失败则放弃该增强器 |
| SS 丰富打印 "0/N papers enriched (…)" | Semantic Scholar 拒绝了（key 被拒或限流），辅助脚本提前停止；转达该行并继续，不带它跑 |
| OpenAlex 不可用（预算用尽 / 被限流 / 宕机） | `openalex_helper` 会自行把调用切换到 Semantic Scholar；转达它的 stderr 行并继续。退出码 `3` = Semantic Scholar 也服务不了：像处理任何 OpenAlex 错误那样处理（告诉用户发生了什么以及 OpenAlex 何时重置） |
| OpenAlex 对 DOI 返回 404 | 改用标题检索兜底（辅助脚本已处理） |
| L2 增强源返回 0 篇 | 静默跳过，在 STEP 13 的 PRISMA-S 日志中注明 |
| SubAgent 分类器返回非法 JSON | `rcs_parser.py` 有 5 层兜底（正则解析） |
| HTML 输出体积 | 没有体积上限或降级——`html_renderer_webartifacts` 始终产出完整的 Shadcn 包。典型 250 篇报告约 1.7 MB；病态的 1000+ 篇 Audit 可能达到 5-10 MB。所有现代浏览器都能干净地处理 10+ MB 的 HTML。 |

---

## 参考文件（渐进披露——读你当前所在那一步对应的）

你不会每次都读完所有这些，也不应该。走到某一步且它对当前案例不简单时（规则 D），再读那一步的参考文件。**core** = 该步必读；**cond** = 仅在触发条件命中时读。

| 文件 | 加载 | 何时读 |
|------|------|---------|
| `tier_decision.md` | core | 选择档级时（STEP 0 之前） |
| `setup.md` | core | STEP 0 —— 配置 + 5 个 key 的获取 |
| `runtime_bootstrap.md` | cond | STEP 0/1 —— 仅当 `$PSP_HOME` 环境变量注入失败，或你需要完整的安装路径清单 / 语言路由原理 |
| `query_planner.md` | core | STEP 1 —— PICO / SPIDER / PEO 框架 |
| `source_routing.md` | core | STEP 1 语言空间（§"Language scope" SSOT）+ STEP 2 路由 + STEP 5 字段优先级合并 |
| `openalex_helper_cheatsheet.md` | core | STEP 3 + STEP 9 —— 子命令、参数、坑 |
| `pubmed_helper_cheatsheet.md` | cond | STEP 4 —— 仅当启用 PubMed |
| `arxiv_helper_cheatsheet.md` | cond | STEP 4 —— 仅当启用 arXiv |
| `classifier_subagent_prompt.md`, `rcs_rubric.md` | core | STEP 6 —— SubAgent 提示词 + RCS 0-10 评分准则 |
| `stop_decision.md` | core | STEP 7 + STEP 8 |
| `citation_chasing.md` | cond | STEP 9 —— 仅当扩展引文 |
| `ss_helper_cheatsheet.md`, `crossref_helper_cheatsheet.md` | cond | STEP 10 —— 仅当丰富 top-N |
| `summary_writer.md` | core | STEP 11 |
| `report_title.md` | core | STEP 1 归一化主题 + STEP 11 受证据约束的显示标题；防止用原始检索式当 H1 |
| `search_export/methodology.md` | cond | STEP 11.5 —— 仅当检索式导出被触发（Audit 默认 / 显式要求 / Deep 提议被接受）；第 1 层语义规则手册 |
| `search_export/chinese_methodology.md` | cond | STEP 11.5 —— 仅限 zh/both 轨道（CNKI / 万方 / SinoMed 检索式；不是西方布尔式的翻译） |
| `search_export/press_checklist.md` | cond | STEP 11.5 —— 对概念模型做 PRESS 六域自检 |
| `search_export/filters_library.md` | cond | STEP 11.5 —— 仅当需要研究设计过滤器（现成的经过验证的过滤器；绝不自己写） |
| `search_export/syntax_cards/<host>.md` | cond | STEP 11.5 —— 各宿主语法事实，由生成器/linter 机器读取；只有手工微调单个平台的检索式时才自己打开 |
| `journal_metrics.md`（SSOT） | cond | STEP 1 / STEP 10-11 —— 仅当用户想要期刊分区（中科院 / JCR / SJR）或 SJR 指标；ISSN 关联、归属说明、R-04 命名 |
| `output_files.md` | core | STEP 12 —— 输出目录结构（相对 PWD） |
| `prisma_s_checklist.md` | core | STEP 13 |
| `agent_mode.md`（SSOT） | cond | 仅当其他 Agent / headless 调用本 Skill —— `agent_search` 信封 + flags |
| `error_handling.md` | cond | 任何意外错误 |

---

## 示例

### 示例 1：快速扫描（5-8 分钟）

用户："find 5-6 high-impact papers on prospect theory in decision making, classics + a couple recent ones"

你：选 Quick 档（信号："5-6"、"high-impact"、检索式短）。以轻量方式执行 STEP 0-2。在 STEP 3 用 `openalex_helper seminal` 找经典文献 + `openalex_helper search` 找近期文献（年份 >= 2020）。STEP 4 不启用 L2 增强源（纯社科）。STEP 6 用一条消息中的 2 个并行 SubAgent 分级 20-30 篇。STEP 7 + 8 照常运行（曲线会渲染进报告）。按规则 C 声明跳过 STEP 9 + STEP 10。渲染报告。

### 示例 2：Standard 中文档（10-17 分钟）

用户："用 paper-search-pro 帮我找一些关于工作记忆训练干预的文献 老板让我看 我对这块完全不懂 要给老年人群体的最好 谢谢🙏"

你：选 Standard 档（默认；信号："找一些"、"老板让我看"）。在 STEP 2 检测到医学信号（"干预" + "老年"）→ 启用 PubMed 增强器。检索计划：PICO（P=老年人，I=工作记忆训练，O=认知结果）。STEP 3 用 OpenAlex `double-sort` 取 top-100。STEP 4 用 PubMed `enrich` 丰富 openalex.json。STEP 5 联邦合并（dict 输出）。STEP 6 用 4 批 × 5 个 SubAgent 分级 60-180 篇——**全部 5 个 Task 放在一条消息里**（规则 B）。STEP 7 曲线，STEP 8 若饱和度 < 0.6 则扩展。渲染报告。

### 示例 3：Deep × 综述写作（30-45 分钟）

用户："I'm writing a proper literature review article on attachment and human-robot interaction in elderly care contexts. Need real depth..."

你：选 Deep 档（"proper literature review article" + "real depth"）。STEP 2 中跨学科（心理学 + CS）→ 启用 arXiv 的近期层和 OpenReview。STEP 1 用 SPIDER 规划。STEP 3 用 OpenAlex `double-sort` top-200 + `reviews` 子命令。STEP 6 分级 200+ 篇，共 8 批——每条消息并行派发 5 个 Task，分两波。STEP 9 扩展引文 2 跳。STEP 10 用 SS + CrossRef 丰富 top-50。渲染报告并附 PRISMA-S 日志。

### 示例 4：Audit × SR 前期准备（2-3 小时）

用户："Need help — preparing a systematic review on dietary interventions for IBS in adults. Inclusion criteria: RCTs, adult populations (≥18), low-FODMAP or fiber-based interventions, English-language, published 2010-present."

你：选 Audit 档（"systematic review" + PICO + IC）。**先展示局限性警告并获得用户确认**——Audit 免责声明是一句承诺交付的承诺，由 STEP 11.5 兑现（Audit 默认执行）。交付时它读作：

> 这是 SR 前期助手，不是 PRISMA 替代。我已为你写好各平台**可粘贴的专业检索式初稿**——每个平台附「使用前请核对」清单（已对官方词表核实的 MeSH 主题词、与需要你在库内人工确认的 Emtree 等主题词，都分别写明了）；注册制 SR 仍需你在订阅库亲自执行，并补 **ClinicalTrials.gov（注册库检索是方法学硬要求，已给检索式+直达链接）与 ICTRP/灰色文献**。

（运行之前，用将来时把同样的承诺先说出来——"我会为你写好…"。）STEP 4 用 `pubmed_helper search-mesh "Irritable Bowel Syndrome" --pub-type "Randomized Controlled Trial"` 做独立 MeSH 检索。STEP 3 还要调用 `openalex_helper journal-list --preset Cochrane`。STEP 10 加入 CrossRef 丰富以获取资助方 + 临床试验编号。STEP 11.5 导出各平台检索式（Audit 默认执行）。STEP 12 渲染时带上 PRISMA 流程图。
