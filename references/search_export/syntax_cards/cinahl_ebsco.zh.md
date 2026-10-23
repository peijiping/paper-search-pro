```yaml
platform: "CINAHL"
host: "EBSCOhost (search.ebscohost.com)"
database: "CINAHL / CINAHL Plus with Full Text / CINAHL Complete (EBSCO)"
access: subscription
field_tags:
  # EBSCOhost 两字母码，大写，前缀式写法："TI term" 或 "TI (a OR b)"
  title: "TI"
  title_abstract: "XB"                      # CINAHL 专有的组合标签：Title OR Abstract
  abstract: "AB"
  subject_heading: "MH"                     # CINAHL Subject Heading（精确；major+minor）
  major_subject_heading: "MM"               # 仅主要概念主题词
  word_in_major_heading: "MJ"               # 主要主题词或副主题词中的词
  word_in_heading: "MW"                     # 主题词或副主题词中的词
  subject_field: "SU"                       # 宽主题字段（heading + KW + 其他）
  keyword: "KW"
  author: "AU"
  author_affiliation: "AF"
  journal_source: "JN"                      # 精确出版物名称；SO = 出版物名称中的词
  pub_type: "PT"
  all_text: "TX"                            # 全部文本（已授权处为全文）
  subheading: "two-letter subheading code"   # 挂在主题词后：MH \"Hypertension/DT\"（DT = 药物治疗）
boolean:
  and: "AND"
  or: "OR"
  not: "NOT"
  case: "uppercase-required"
proximity:
  unordered: "N{n}"                          # Near —— 任意词序；例 (decision* N3 (aid* OR support*))
  ordered: "W{n}"                            # Within —— 按输入词序；例 primary W3 care
  n_family: "gap"                            # n = 两词之间的最大词数（McGill："up to n words between"）
  field_limit: []                            # 跨字段可用
  constraints: ["truncation_allowed_inside_proximity", "uncheck_suggest_subject_headings_or_specify_TI_AB"]
  bare_operator_note: "不加限定的相邻词默认行为随宿主/界面/版本而变；生成器总会显式加上算符，因此输出不受影响"
truncation:
  multi_char: "*"                            # 零或多（therap*）
  single_char: "?"                           # 恰好一个字符，不是零个（ne?t -> neat/nest/next）；不能置于词尾
  zero_or_one: "#"                           # 零或一个（colo#r -> color/colour）
  min_chars_before: null
  phrase_truncation: "allowed"
phrase:
  quote: "\"\""                              # "perceived stress scale" = 精确有序短语
  exception: "EBSCO 会自动扩展变体；引号强制精确字符串"
controlled_vocab:
  name: "CINAHL Subject Headings (CINAHL Headings)"
  explode: "在主题词标签内用 + 后缀: (MH \"Pregnancy in Diabetes+\") -> 该主题词 + 其下所有更窄主题词"
  no_explode: "(MH \"Pregnancy in Diabetes\") -> 仅该主题词，不含更窄词条"
  major: "用 MM 代替 MH: (MM \"Pregnancy in Diabetes\") -> 仅主要概念；可与爆炸组合 (MM \"...+\")"
  subheading: "用 / 挂接: (MH \"Hypertension/DT\")；游离副主题词: (MW \"DI\" OR \"US\")"
  verification: llm_suggest_only             # 没有免费的 CINAHL-Headings 查询 API
line_search:
  supported: true
  syntax: "S1 AND S2"                         # EBSCO Search History（S1, S2, ...）
  history_cap: null
special_chars_escape: null                     # *, ?, # 是通配符；无文档化的反斜杠转义方案
deep_link:
  tier: "C"
  url_kind: "paste_only"
  url_template: "https://search.ebscohost.com/login.aspx?authtype=ip&profile=ehost&defaultdb=ccm"
  note: "EBSCOhost 的地址绑定机构客户 ID/档案；执行需要机构登录。没有公开的无状态深链。绝不构造绕过登录的 URL。"
source_url:
  - "https://libraryguides.mcgill.ca/c.php?g=477164&p=5387366"                      # McGill：CINAHL MH/MM/MJ/MW 字段码 + 爆炸 +
  - "https://libraryguides.mcgill.ca/epib629/search-tips"                           # McGill：CINAHL Nn/Wn 邻近 + suggest-subject-headings 说明
  - "https://about.ebsco.com/sites/default/files/acquiadam-assets/Top-Five-Searching-Strategies-Handout.pdf"  # EBSCO 官方：N/W、通配符
  - "https://connect.ebsco.com/"                                                     # EBSCO Connect 帮助中心（CINAHL/MeSH Headings、字段码）
  - "https://guides.library.manoa.hawaii.edu/cinahl/cinahlhdgs"                      # CINAHL 爆炸 / Major Concept / 副主题词（EBSCO 转录）
verified_date: "2026-07-16"
gotchas:
  - "爆炸是主题词上的 + 后缀，且在标签内部：(MH \"Heading+\") —— 不是独立算符。"
  - "主要概念是不同的标签（MM），不是 MH 的修饰符；MM 收窄范围，MH = major+minor。"
  - "要跑邻近/关键词检索，须取消勾选 'Suggest Subject Headings'，或用 TI/AB 限定 —— 否则 CINAHL 会把词拦截成主题词查找。"
  - "通配符陷阱：'?' = 恰好一个字符（不是零个，也不能用于词尾）；'#' = 零或一个。"
  - "CINAHL Headings 仅由 LLM 建议 —— 没有免费 API；每个 MH/MM 词条都要标记为人工核实（风险 A-6）。"
  - "订阅登录墙（C 档）：交付可直接粘贴的检索串，而不是可点击的无状态深链。"
hosts_note:
  summary: "CINAHL 由 EBSCO 独家提供 —— EBSCOhost 是唯一的生产宿主（EBSCO 拥有 CINAHL）。不存在 Ovid/ProQuest 版 CINAHL。因此本卡只有单一宿主；R4 的通用 EBSCO 卡此前代管 CINAHL，但 CINAHL Headings 词表（MH/MM/爆炸 +/副主题词）是 CINAHL 专有的，故在此作为独立卡记录。"
  note: "EBSCOhost 引擎与 EBSCO 上的 PsycINFO、MEDLINE、ERIC、EconLit 共用（同样的 N/W 算符、同样的 *,?,# 通配符、同样的 S1/S2 历史）。CINAHL 专有的是受控词表（CINAHL Subject Headings）和 XB（Title OR Abstract）组合标签。"
```

# CINAHL — EBSCOhost（单一宿主；独立卡）

**为何独立成卡。** CINAHL（Cumulative Index to Nursing and Allied Health Literature）归 EBSCO 所有，**只**存在于 EBSCOhost——没有 Ovid 版或 ProQuest 版 CINAHL。R4 此前用通用 EBSCO（PsycINFO）卡代管 CINAHL，但 CINAHL 的受控词表——带 `MH`/`MM`、以 `+` 爆炸、带副主题词的 **CINAHL Subject Headings**——是 CINAHL 专有的，必须建模为独立卡（Wave-0 要求，风险 D-21 库×宿主建模）。

**招牌特性——CINAHL Subject Headings。**面向护理/相关健康学科的 MeSH 式结构受控词表。EBSCO 官方对爆炸的表述：

> 原文："In a database with a tree, such as MeSH or CINAHL Headings, exploding retrieves all documents containing any of the subject terms below the term you selected. … If a plus sign (+) appears next to a narrower or related term, there are narrower terms below it." —— 中文：在带层级的数据库（如 MeSH 或 CINAHL Headings）中，爆炸检索会取回包含所选主题词之下任一主题词的所有文献。……如果某个更窄或相关的主题词旁出现加号 (+)，说明其下还有更窄的主题词。
> —— EBSCO Help（Thesaurus），核实于 2026-07-16

具体语法（McGill，转录 EBSCO CINAHL）：

> 原文："**Explode** — Represented by the + sign at the end of the subject heading … e.g., `(MH "Pregnancy in Diabetes+")` will search for records indexed with 'Pregnancy in Diabetes' OR 'Diabetes Mellitus, Gestational' OR 'Fetal Macrosomia'. **Major concept** — Represented by MM (instead of MH) … e.g., `(MM "Pregnancy in Diabetes")`; can be combined with Explode, e.g., `(MM "Pregnancy in Diabetes+")`." —— 中文：**Explode（爆炸）**—— 由主题词末尾的 + 号表示……例如 `(MH "Pregnancy in Diabetes+")` 会检索以 'Pregnancy in Diabetes' OR 'Diabetes Mellitus, Gestational' OR 'Fetal Macrosomia' 标引的记录。**Major concept（主要概念）**—— 用 MM（而不是 MH）表示……例如 `(MM "Pregnancy in Diabetes")`；可与 Explode 组合，例如 `(MM "Pregnancy in Diabetes+")`。
> —— https://libraryguides.mcgill.ca/c.php?g=477164&p=5387366

主题词族字段码：
- `MH` = Subject Heading（精确；返回 **major + minor** 两级标引）。
- `MM` = 仅 Major Subject Heading（标引员指定的主要焦点 → 精度更高、命中更少）。
- `MJ` = 主要主题词或副主题词中的词。
- `MW` = 主题词或副主题词中的词。
- **副主题词**：用 `/` 挂在主题词后的两字母码，例如 `(MH "Hypertension/DT")`（DT = 药物治疗）；游离副主题词用 `(MW "DI" OR "US")`（诊断 / 超声检查）。
- `XB` = **Title OR Abstract**（CINAHL 便捷的组合标签；注意 EBSCO 上的 EconLit 没有这个标签）。

**关键执行注意（官方 CINAHL 行为）。** CINAHL 默认的不限定检索框会执行*主题词查找*（"Suggest Subject Headings" 开启）。要跑原始的关键词/邻近检索，必须取消勾选它，或限定字段：

> 原文："To use proximity operators in CINAHL, uncheck 'Suggest Subject Headings' if entering an unqualified search … or specify the TI OR AB fields in your search query." —— 中文：在 CINAHL 中使用邻近算符时，如果输入的是不限定检索，请取消勾选 'Suggest Subject Headings'……或者在检索式中指定 TI OR AB 字段。
> —— https://libraryguides.mcgill.ca/epib629/search-tips

**邻近 / 布尔 / 通配符。** 与其他 EBSCO 数据库同一套 EBSCOhost 引擎：`Nn`（near，任意词序）、`Wn`（within，按输入词序）、`n` = 两词**之间**的最大词数（gap 家族；McGill：*"up to n words between them"* —— 中文：最多相隔 n 个词）。布尔须大写。通配符 `*`（零或多）、`?`（恰好一个字符，不能置于词尾）、`#`（零或一个，例如 `colo#r`）。邻近算符内允许截词。Search History 用 `S1`、`S2`…… 组合为 `S1 AND S2`。

**受控词核验。** CINAHL Headings 属于 **`llm_suggest_only`**——不存在免费的 CINAHL-Headings 查询 API。每个 `MH`/`MM` 词条都必须在 CINAHL 中标记为人工核实；绝不伪造核实结果（风险 A-6）。
