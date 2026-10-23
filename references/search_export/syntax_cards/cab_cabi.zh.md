```yaml
platform: "CAB Abstracts"
host: "CABI Digital Library (cabidigitallibrary.org) — Atypon/Literatum engine"
database: "CAB Abstracts (product code sc:CA) — publisher-native platform"
access: subscription
field_tags:
  # CABI Digital Library 用小写字段标签，前缀式写法：tag:term
  title: "title"                            # 复合字段：检索 ET（英文题名）+ AT（附加题名）
  english_title: "et"
  abstract: "ab"
  author: "author"                          # 复合字段：AU + ED + CA
  author_affiliation: "aa"
  descriptor: "de"                          # CABI Thesaurus 主题描述符（受控）
  organism_descriptor: "od"                 # 受控的生物/分类学名称
  geographic: "gl"                          # 受控的地理位置
  broad_term: "up"                          # 受控的上位分类学/地理词
  identifier: "id"                          # 非受控/新兴术语 + 美式拼写变体
  indexing_term: "indexingterm"             # SUPER 字段：一次检索 DE + ID + OD + UP + GL
  cabicode: "cc"                            # CABICODE 主题分类码
  publication_title: "do"                   # 来源出版物（短语索引；用完整刊名）
  pub_type: "it"                            # Item Type（期刊论文、会议等）
  product: "sc"                             # 序号/产品代码；CAB Abstracts = sc:ca
  year: "yr"
  language_of_text: "la"                    # ISO 639-1 代码，不是语言名称
  doi: "oi"
  issn: "sn"
  isbn: "bn"
  subject_heading: "de"                     # （descriptor 的别名，保持跨卡一致）
  subheading: null                          # CABI 没有 MeSH 式的副主题词结构
  all: "n/a (default 'All fields' searches everything)"
boolean:
  and: "AND"                                # 也可用 &&
  or: "OR"                                  # 也可用 ||
  not: "NOT"                                # 也可用 !!
  case: "uppercase-required"                # 小写 and/or/not 会被当作检索词解析，而不是算符
proximity:
  unordered: "\"{term1} {term2}\"~{n}"      # Lucene phrase-slop：引号 + 波浪号 + n；例 "metabolic mechanism"~3
  ordered: null                             # 没有有序/相邻算符（slop 为无序）
  n_family: "gap"                           # n = 分隔两词的最大词数（CABI 官方措辞）
  field_limit: []                           # 任意文本字段
  constraints: ["phrase_exclusive", "no_truncation_inside_quotes", "proximity_ignored_if_inside_quotes"]
truncation:
  multi_char: "*"                           # 任意数量字符（duoden*；p*diatric -> pediatric/paediatric）
  single_char: null
  zero_or_one: "?"                          # ? 匹配 0 个或任意单个字符（l?st -> last/lest/list）
  min_chars_before: null
  phrase_truncation: "forbidden"            # * 和 ? 在引号内被忽略，且会导致检索错误
  leading_wildcard: "forbidden"             # 通配符不能出现在词首（报错）
phrase:
  quote: "\"\""                             # 关闭词干化 + 强制精确词/短语
  exception: "引号内的 AND/OR/NOT 变成字面词；通配符与邻近算符被忽略/报错"
controlled_vocab:
  name: "CABI Thesaurus (Descriptors, Organism Descriptors, Geographic Location, Broad Terms, Identifiers)"
  explode: "检索上位词 (up:) 可自动纳入 CAB 层级中其下更窄的词条"
  no_explode: "检索精确描述符 (de:) 以只取该词条"
  major: null
  subheading: null
  index_field_stemming: "OFF —— 索引字段要求拼写精确；变体请用截词 (de:colo?r) 或 CABI Thesaurus（英式拼写）"
  verification: llm_suggest_only            # 没有免费的 CABI-Thesaurus 查询 API
line_search:
  supported: true
  syntax: "Recent Searches -> Combine Searches (session set combination)"
  history_cap: 10                            # 每会话最近 10 次检索（会话空闲 30 分钟后过期）
special_chars_escape: null                    # 通配符 *,? ；引号切换词干化；无反斜杠转义方案
deep_link:
  tier: "C"
  url_kind: "paste_only"
  url_template: "https://www.cabidigitallibrary.org/action/doSearch?AllField={urlenc}"
  note: "CAB Abstracts 在所有宿主上均为订阅制；执行需要机构登录。此处仅记录 URL 结构：本轮未做 HTTP 验证，也不做登录绕过。交付可直接粘贴的专业检索串。"
source_url:
  - "https://help.cabi.org/cabi-digital-library-help/"                                       # CABI 官方帮助中心
  - "https://help.cabi.org/cabi-digital-library-help/search/instructions-for-recreating-saved-searches-from-cab-direct"  # 官方：邻近 ~n、通配符、布尔、索引字段词干化
  - "https://www.cabi.org/cabithesaurus/"                                                    # CABI Thesaurus（受控词浏览）
  - "https://ospguides.ovid.com/OSPguides/cabadb.htm"                                        # Ovid CAB Abstracts 指南（hosts_note）
verified_date: "2026-07-16"
gotchas:
  - "CABI Digital Library 已取代 CAB Direct —— 语法已变（Atypon/Literatum 引擎）；不要原样复用旧 CAB Direct 的 MyCABI 检索串。"
  - "邻近是 Lucene phrase-slop：\"a b\"~n。n = 分隔两词的最大词数。引号内的截词（* / ?）既被忽略又会报错 —— 加引号前先把截词展开成明确的变体。"
  - "布尔必须大写；小写 and/or/not 按字面词检索。"
  - "索引字段（de/od/gl/up/id）不做词干化 —— 拼写必须精确；请用 CABI Thesaurus 或 de:colo?r 式截词。CABI 用英式拼写。"
  - "此处的 '?' = 零或一个字符（不是恰好一个）；'*' = 任意数量。词首通配符报错。"
  - "CABI Thesaurus 词条仅由 LLM 建议 —— 没有免费 API；每个 de:/od:/gl: 词条都要标记为人工核实（风险 A-6）。"
hosts_note:
  summary: "CAB Abstracts（CABI，农业/兽医/环境/全球健康）托管于出版商原生的 CABI Digital Library（主卡），以及 Ovid、EBSCOhost 和 Web of Science。各宿主语法差异很大；CABI Thesaurus 这套受控词表则保持不变。主宿主 = CABI Digital Library（当前原生平台，取代 CAB Direct）。"
  ovid: "Ovid CAB Abstracts 使用 Ovid 点号语法（.ti./.ab./.mp.）、adjN 邻近（gap+1 家族：adj2 = 最多 1 个词隔开）、$ 或 * 截词、exp heading/ 做词表爆炸。题名字段分为 ET（Every Title）/ OT（Original Title）。来源：ospguides.ovid.com/OSPguides/cabadb.htm。"
  ebsco: "EBSCOhost 上的 CAB Abstracts 用 EBSCO 语法（大写两字母码、Nn/Wn 邻近 = gap 家族、*、?、# 通配符、S1/S2 历史）—— 机制与 CINAHL/EconLit 的 EBSCO 卡完全相同，只是词表换成 CABI Thesaurus。"
  wos: "Web of Science 上的 CAB Abstracts 用 WoS 语法（TS=()、NEAR/n = gap 家族、*、$、? 通配符）。WoS 平台上没有 CABI-Thesaurus 爆炸。"
  note: "CAB Direct（旧版）正在退役，由 CABI Digital Library 取代；CAB Direct 的保存检索式需要做转换。"
```

# CAB Abstracts（CABI Digital Library，主平台：出版商原生宿主）

**宿主选择。** CAB Abstracts（CABI 的旗舰应用生命科学数据库：农业、林业、兽医、人体健康/营养、环境）可在四个宿主上访问——出版商原生的 **CABI Digital Library**、Ovid、EBSCOhost 和 Web of Science。本卡以 **CABI Digital Library** 为主宿主，理由有三：(1) 它是当前的出版商原生平台，与卡集「原生优先」的模式一致（PubMed=NLM、ERIC=native、CNKI=native、Wanfang=native）；(2) 它有完整的官方文档（60 页的 CABI Digital Library Search Guide）；(3) 它承载原生的 CABI Thesaurus。它**取代了 CAB Direct**（Atypon/Literatum 引擎），因此旧的 CAB Direct 检索串需要转换。Ovid / EBSCO / WoS 的差异记录在 `hosts_note` 中（Ovid 是系统综述常用的替代方案）。CAB Abstracts 在所有宿主上均为订阅制 → 深链档位 **C（仅粘贴）**。

**招牌特性——CABI Thesaurus（五个受控索引族）。**官方定义：

> 原文："**Descriptor** – topic based terms … part of CABI's controlled vocabulary. **Organism Descriptor** – plant and animal names and scientific names … **Identifier** – new concepts or organisms that are not already in CABI's controlled vocabulary … **Geographic location** – Location where the research took place … **Broad term** – Terms which exist above a search term in the taxonomic or geographic hierarchy … Searching with Broader Terms will automatically find all relevant narrower terms immediately below that term." —— 中文：**Descriptor（描述符）**—— 基于主题的术语…… 属于 CABI 受控词表的一部分。**Organism Descriptor（生物描述符）**—— 植物与动物名称及学名…… **Identifier（标识符）**—— CABI 受控词表中尚未收录的新概念或生物…… **Geographic location（地理）**—— 研究开展所在的地点…… **Broad term（上位词）**—— 在分类学或地理层级中位于检索词之上的术语…… 用上位词检索时，会自动命中其正下方所有相关的更窄词条。
> —— CABI Digital Library Search Guide（help.cabi.org），核实于 2026-07-16

字段标签：`de:`（描述符）、`od:`（生物描述符）、`gl:`（地理）、`up:`（上位词 → 等价于 "explode"）、`id:`（标识符），以及超级字段 `indexingterm:`，它把 DE+ID+OD+UP+GL 一并检索。短语必须加引号才能精确解析：`indexingterm:"disease prevention"`（不加引号会拆成 `indexingterm:disease` AND `prevention`）。CABI 建议以词表为锚点检索，以缩短检索串长度：`de:"complementary and alternative medicine"`。

**布尔（官方）。** `AND` / `OR` / `NOT` **必须大写**（符号 `&&` / `||` / `!!` 同样可用，前后加空格）：*"Boolean operators must be capitalised, otherwise they are parsed as search terms not as Boolean operators."* —— 中文：布尔算符必须大写，否则会被当作检索词而不是布尔算符来解析。相邻词之间自动 AND。三个及以上混合算符时请用括号。

**邻近（官方）—— Lucene phrase-slop。**

> 原文："To do a proximity search, use quotation marks around terms you wish to specify followed by a tilde ~ and a number indicating the **maximum number of words separating the terms**. For example, `"metabolic mechanism"~3` searches for the words metabolic and mechanism within 3 words of each other. Please note that Proximity operators are ignored within quotation marks." —— 中文：要做邻近检索，用引号把要限定的术语括起来，后跟波浪号 ~ 和一个数字，表示**分隔两词的最大词数**。例如 `"metabolic mechanism"~3` 检索 metabolic 与 mechanism 彼此间隔在 3 个词以内的记录。请注意：引号内的邻近算符会被忽略。
> —— CABI Digital Library Search Guide

因此 `n` = **分隔两词的最大词数** = gap 家族（与 PubMed 的 `[tiab:~N]`、WOS、Scopus 同族）。它是短语专属的（必须加引号），而且**引号内禁止截词**——先把截词展开成明确的变体。

**截词 / 通配符 / 词干化（官方）。** `*` = 任意数量字符（`duoden*`、`p*diatric` → pediatric/paediatric）；`?` = **零个或任意单个字符**（`l?st` → last/lest/list）。词首通配符会报错。双引号内的通配符被忽略，且**会导致检索错误**。文本字段（题名/摘要/全文以及 `indexingterm` 超级字段）默认自动开启词干化，但**各个索引字段**（`de`/`od`/`gl`/`up`/`id`）**关闭**，它们要求拼写精确——变体请用截词（`de:colo?r`）或 CABI Thesaurus（英式拼写）。双引号会关闭词干化。

**集合合并。** 没有经典的 `#1 AND #2` 行式语法；改用 **Recent Searches → Combine Searches**（每会话最近 10 次检索；空闲 30 分钟过期）。

**受控词核验。** CABI Thesaurus 词条属于 **`llm_suggest_only`**——没有免费的 CABI-Thesaurus 查询 API。每个 `de:`/`od:`/`gl:` 词条都要标记为需在 CABI Thesaurus 中人工核实；绝不伪造核实结果（风险 A-6）。
