```yaml
platform: "EconLit"
host: "EBSCOhost (search.ebscohost.com)"
database: "EconLit / EconLit with Full Text (American Economic Association)"
access: subscription
field_tags:
  # EBSCOhost 两字母码，大写，前缀式写法："TI term" 或 "TI (a OR b)"
  title: "TI"
  title_abstract: "TI(...) OR AB(...)"   # EBSCO 上的 EconLit 没有单一的 ti+ab 标签（对比 CINAHL 的 XB）
  abstract: "AB"
  subject_heading: "DE"                    # 精确的 EconLit Subject Descriptor（JEL）；SU = 更宽的主题字段
  subject_broad: "SU"
  keyword: "KW"
  author: "AU"
  author_affiliation: "AF"
  journal_source: "SO"                     # 来源/出版物名称；JN = 精确期刊名
  pub_type: "PT"
  language: "LA"
  all_text: "TX"                           # 全部文本 / 可得处的全文
  named_person: "PS"                       # Named Person（EconLit 的「人物即主题」限定）
  geographic: "GE"                          # Geographic Descriptor（区域主题）
  subheading: null                          # EconLit 的 descriptor 没有 MeSH 式副主题词结构
boolean:
  and: "AND"
  or: "OR"
  not: "NOT"
  case: "uppercase-required"
proximity:
  unordered: "N{n}"                         # Near —— 任意词序；例 tax N5 reform
  ordered: "W{n}"                           # Within —— 按输入词序；例 hiking W5 trails
  n_family: "gap"                           # n = 两词之间的最大词数（见 proximity_table.json）
  field_limit: []                           # 不限定字段；在 TI/AB/TX 等中可用
  constraints: ["truncation_allowed_inside_proximity"]   # 与 PubMed 不同，EBSCO 允许在 N/W 内使用 *
  bare_operator_note: "不加限定的相邻词默认行为随宿主/界面/版本而变（部分 EBSCO 指南说是 AND，部分说是 N5）；生成器总会显式加上算符，因此输出不受影响"
truncation:
  multi_char: "*"                           # 零或多；词尾或词中（organi*ation）
  single_char: "?"                          # 恰好一个字符，不是零个（wom?n -> woman/women）
  zero_or_one: "#"                          # 零或一个字符（colo#r -> color/colour）
  min_chars_before: null                    # EBSCO 没有文档化的 4 字符下限（对比 PubMed）
  phrase_truncation: "allowed"
phrase:
  quote: "\"\""                             # "perceived stress scale" = 精确有序短语
  exception: "EBSCO 会自动扩展复数/变体；引号强制精确字符串"
controlled_vocab:
  name: "EconLit Subject Descriptors = JEL Classification codes (AEA)"
  explode: "词表的 'Explode' 选项（browse Subjects/Indexes -> Subject Codes）"
  no_explode: "默认（单一 descriptor，不含更窄词条）"
  major: null                               # EconLit 的 descriptor 没有 major/minor 之分
  subheading: null
  code_format: "4 位数 = JEL 3 位码 + 末尾补 0（J16 -> J160）；通过 DE 或 Subject Codes 索引浏览"
  verification: llm_suggest_only            # 没有免费的 EconLit 词表/JEL 查询 API
line_search:
  supported: true
  syntax: "S1 AND S2"                        # EBSCO Search History（S1, S2, ...）
  history_cap: null
special_chars_escape: null                    # *, ?, # 是通配符；无文档化的反斜杠转义方案
deep_link:
  tier: "C"
  url_kind: "paste_only"
  url_template: "https://search.ebscohost.com/login.aspx?authtype=ip&profile=ehost&defaultdb=eoh"
  note: "EBSCOhost 的地址绑定机构客户 ID/档案；执行需要机构登录。没有公开的无状态深链。不要构造任何绕过登录的 URL。"
source_url:
  - "https://www.aeaweb.org/econlit/search-hints"                                   # AEA 官方：JEL descriptor、4 位码规则
  - "https://www.aeaweb.org/econlit/jelCodes.php?view=jel"                          # AEA 官方：完整 JEL 码列表
  - "https://about.ebsco.com/products/research-databases/econlit-full-text"         # EBSCO 官方产品页
  - "https://about.ebsco.com/sites/default/files/acquiadam-assets/Top-Five-Searching-Strategies-Handout.pdf"  # EBSCO 官方：N/W、通配符
  - "https://ospguides.ovid.com/OSPguides/econdb.htm"                               # Ovid EconLit 指南（hosts_note：CC 字段）
verified_date: "2026-07-16"
gotchas:
  - "JEL 主题码在 EconLit 上必须按 4 位数检索（J160，而不是 J16）；每个 JEL 码都追加一个末尾的 0。"
  - "EBSCO 字段码必须大写（小写 'de' 会被当作字面词，而不是 DE 字段）。"
  - "通配符陷阱：'?' = 恰好一个字符（绝不是零个）；'#' = 零或一个。二者不可互换。"
  - "订阅登录墙（C 档）：交付可直接粘贴的检索串；不要承诺可点击的无状态深链。"
  - "EBSCO 上的 EconLit 没有单一的题名+摘要标签 —— 展开为 TI(...) OR AB(...)。"
  - "JEL/EconLit descriptor 仅由 LLM 建议 —— 没有免费查询 API；每个受控词条都要标记为在目标宿主中人工核实。"
hosts_note:
  summary: "EconLit 由 AEA 授权给多个宿主；EBSCOhost 是最常见的学术宿主，也是本卡的主宿主。JEL descriptor 在各宿主完全相同（4 位码）；只有字段码语法和分类字段机制随宿主而异。"
  ovid: "Ovid EconLit 使用 Ovid 点号语法（.ti./.ab./.mp.；默认 .mp. = ab,ti,ct）、adjN 邻近（gap+1 家族，adj2 = 最多 1 个词隔开）、$ 或 * 截词。JEL 码有专用字段：CC（Classification Codes），可用通配符检索，例如 A2$.cc 或 A2*.cc 可取回全部 A2* 码；1991 年后用 3 位字母数字码（C13），1991 年前用 4 位数字码。来源：ospguides.ovid.com/OSPguides/econdb.htm。"
  proquest: "ProQuest EconLit 使用 ProQuest 字段码（SU/IF 等）和 NEAR/n（N/n）、PRE/n（P/n）邻近（gap 家族）。descriptor 字段不同；查阅 ProQuest 帮助。"
  note: "OCLC FirstSearch 上也有。截词符号因供应商而异（AEA 明确警告过这一点）—— 粘贴前务必重新核对目标宿主的通配符字符。"
```

# EconLit — EBSCOhost（主宿主）

**覆盖范围 / 角色。** EconLit 是 American Economic Association 的经济学文献权威索引（期刊论文、图书、合著论文集文章、学位论文、工作论文，以及带全文的 JEL 书评）。EBSCOhost 是最常见的学术访问宿主；本卡即主卡。EconLit 在所有宿主上均为纯订阅 → 深链档位 **C（仅粘贴）**；交付可直接粘贴的专业检索串，而不是可点击 URL。

**招牌特性——JEL 分类码字段。** EconLit 的受控词表是 **Journal of Economic Literature (JEL) Classification System**（又称 AEA Classification System），即经济学界通用的标准主题方案。AEA 官方表述：

> 原文："The EconLit subject descriptors are the same as the Journal of Economic Literature (JEL) classifications also known as the American Economic Association Classification System. On EconLit, a zero has been added to the JEL classification to make a four-digit descriptor code. For example, the JEL classification 'A11 - Role of Economics; Role of Economists' appears on EconLit as Role of Economics; Role of Economists (A110). **When searching by subject code on EconLit, it is necessary to use all four digits, i.e. A110, or a truncation symbol.**" —— 中文：EconLit 的主题 descriptor 与 Journal of Economic Literature (JEL) 分类（即 American Economic Association Classification System）相同。在 EconLit 上，JEL 分类后会补一个 0，构成四位数 descriptor 码。例如 JEL 分类 'A11 - Role of Economics; Role of Economists' 在 EconLit 上显示为 Role of Economics; Role of Economists (A110)。**在 EconLit 上按主题码检索时，必须使用全部四位数字，即 A110，或者使用截词符。**
> —— AEA，*EconLit Search Hints*，https://www.aeaweb.org/econlit/search-hints（核实于 2026-07-16）

对生成检索串的实际影响：主题码块比自由词更精确（AEA 自己的例子：`Country and Industry Studies of Trade (F140)` 比单词 `trade` 更窄）。在 EBSCOhost 上，浏览 **Subjects / Indexes → Subject Codes** 挑选 descriptor，或用 `DE` 检索精确 descriptor。专用的数字分类字段码存在于 **Ovid**（`CC`，例如 `A2$.cc`）；在 EBSCOhost 上这些码通过 descriptor/主题字段检索。截词符号因供应商而异——AEA 的说明明确提示了这一点，粘贴前须重新核对宿主。

**布尔 / 字段码（EBSCO 官方）。** `AND` / `OR` / `NOT` 必须大写。字段码是大写的两字母前缀（`TI teaching AND AB anxiety`）。EBSCO 自己的指引：*"each code must be entered in UPPER case … Otherwise your codes could be interpreted as simple text"* —— 中文：每个码都必须以大写输入……否则你的码可能被解释为普通文本（EBSCO Connect / 馆方转录）。用括号把同一个码复用到整个 OR 块中：`DE (J160 OR "economics of gender")`。

**邻近（EBSCO 官方 Top-5 Searching Strategies handout）。**

> 原文："Near Operator (N) finds results in which search terms are within a specified number of words of one another in any order. Example: tax N5 reform … Within Operator (W) finds results in which search terms are within a specified number of words of one another and in the order in which you entered them. Example: hiking W5 trails." —— 中文：Near 算符 (N) 找出检索词彼此间隔在指定词数以内、且顺序任意的结果。例：tax N5 reform……Within 算符 (W) 找出检索词彼此间隔在指定词数以内、且按你输入的顺序排列的结果。例：hiking W5 trails。
> —— https://about.ebsco.com/sites/default/files/acquiadam-assets/Top-Five-Searching-Strategies-Handout.pdf

这里的 `n` 是两词**之间**的最大词数（gap 家族——与 WOS/Scopus/ProQuest 同族；见 `proximity_table.json`）。EBSCO 的邻近算符内**允许**截词（与 PubMed 不同）。

**截词 / 通配符（EBSCO 官方 Top-5 handout）。**

> 原文："Use an asterisk (*) when you want to search for results containing various forms of a word … Use wildcards (? or #) … ? stands for one additional character, but not for zero characters. # stands for zero or one character. A search for wom?n will find … woman and women. A search of colo#r will find … color or colour." —— 中文：当你想检索包含一个词各种形态的结果时，用星号 (*)……使用通配符 (? 或 #)……? 代表一个额外字符，但不代表零个字符。# 代表零个或一个字符。检索 wom?n 会得到……woman 和 women。检索 colo#r 会得到……color 或 colour。

因此 `*` = 零或多，`?` = **恰好一个**字符（绝不是零个），`#` = 零或一个。三者不可互换——这是 EBSCO 最常见的通配符错误。

**受控词核验。** EconLit descriptor / JEL 码属于 **`llm_suggest_only`**——不存在免费的 EconLit 词表或 JEL 查询 API。每个拟用的 descriptor/码都必须带上「在目标宿主中人工核实」的标记；绝不伪造核实结果（风险 A-6）。
