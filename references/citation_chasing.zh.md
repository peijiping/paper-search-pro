# 引文追溯

*本文件由主 Agent 在 `SKILL.md` 的 STEP 9 读取，即饱和度曲线 / 档级预算表明应当扩展引文时。引文扩展是可选的——Quick 档总是跳过；Standard 有条件地做 1 跳；Deep 做 1-2 跳；Audit 做 2 跳。*

## 何时扩展（按档级 × 饱和度矩阵）

| 档级 | 默认跳数 | 触发条件 |
|------|--------------|--------------------|
| Quick | **0** | 永不扩展。保持结果紧凑。 |
| Standard | **1** | saturation < 0.6 且 papers_evaluated < 预算的 60% |
| Deep | **1-2** | saturation < 0.7，或用户明确要求 "thorough"（彻底）；若 rcs ≥ 8 的论文聚集在 3-5 篇奠基性著作周围，就做 2 跳 |
| Audit | **2 + 期刊白名单** | 总是做 2 跳；同时用 `openalex_helper journal-list` 补全每一跳（Cochrane / medical_top） |

## 反向（references）与正向（cited_by）

| 方向 | 强度 | 适合用来 |
|-----------|----------|---------|
| **反向**（本文引用的文献） | 找到奠基脉络——是什么造出了这篇论文 | 追溯理论谱系；找到最初的构念论文；"这篇论文依赖了什么" |
| **正向**（引用了这篇论文的文献） | 找到后续网络——谁用了这项工作 | 追踪一个方法的扩散；找到重复验证 / 扩展 / 反面证据 |

对大多数检索式，**一次调用同时做两个方向**——这正是 `openalex_helper citation-network` 的返回内容。成本差异很小（一次 API 调用而不是两次），而正向 + 反向一起做的召回增益很可观。

```bash
# 一次调用返回两个方向
python3 -m scripts.openalex_helper citation-network <openalex_id> \
    --refs-limit 30 --cited-by-limit 50 \
    > ./paper-search-results/<id>/raw/citations_<openalex_id>.json
```

## top-RCS 过滤（只从高相关论文出发扩展）

**规则**：只追溯 rcs **rcs ≥ 7** 论文的引文。较低 rcs 的论文按定义就与检索式只是擦边，它们的引文网络更加擦边。

对当前 KG 中每一篇 rcs ≥ 7 的论文：
1. 调用 `citation-network`
2. 把得到的 refs + cited_by 联邦合并进 KG
3. 只对**新**论文重新分级（已分级的跳过）

如果 top-RCS 的一簇非常小（rcs ≥ 7 的论文少于 3 篇），把阈值降到 rcs ≥ 6 再做引文追溯——否则扩展会太窄。

## 防爆炸护栏

不加约束时，引文扩展可能把一个 100 篇的 KG 膨胀到 5000+ 篇。通过以下方式约束：

1. **top-RCS 门控**（见上）——每一跳最多约 10 篇论文触发扩展
2. 辅助脚本调用中的 **refs / cited 上限**：`--refs-limit 30 --cited-by-limit 50`
3. **重新分级前去重**——`federated_kg_resolver` 已按 `paper_id` 去重；已在 KG 中的论文不会被重新分级
4. **除非用户明确要 2 跳，否则 1 跳后就停**——第二跳的扩展通常会把论文数放大 5-15 倍，而且大多是噪声

如果 1 跳之后新增论文 > 200 篇且 saturation 跳到 > 0.85，**停止**。边际产出递减是真实存在的。

## 联邦合并 + 重新分级循环

每次引文扩展之后：

```bash
# 第 1 步：把新的引文联邦合并进现有 KG
python3 -m scripts.federated_kg_resolver \
    --input-files ./.../kg_classified.json ./.../raw/citations_*.json \
    --output ./.../kg.json

# 第 2 步：只对新增论文重新分级（已分级的论文保留其 RCS）
#（rcs_parser 会跳过 rcs 字段已经非 null 的论文）
python3 -m scripts.rcs_parser \
    --input-dir ./.../classifications/round_2/ \
    --kg ./.../kg.json \
    --output ./.../kg_classified.json

# 第 3 步：重新计算饱和度
python3 -m scripts.discovery_curve \
    --kg ./.../kg_classified.json \
    --output ./.../curve.json
```

循环直到 saturation 稳定下来或预算耗尽（见 `stop_decision.md`）。

## 实测得出的警告

- **K&T 1979 在 OpenAlex 上是 `references=[]`**（SA-Z2 F19 已确认）。2000 年前的论文系统性地缺失参考文献列表，原因是上游 OpenAlex 的撤下。接受这个空列表——不要重试，也不要去试 CrossRef（对老论文它同样没用）。K&T 1979 的正向 `cited_by` 仍然可用（约 46,625 篇论文引用它）。对经典文献做反向追溯会一无所获——这是数据如此，不是 bug。
- **arXiv 预印本默认没有 OpenAlex 的 `references`**——它们在 `locations[]` 字段里，但没有被解析。正向 `cited_by` 可用。
- **Audit 档**：第 1 跳之后，把引文追溯与 `openalex_helper journal-list --preset Cochrane` 结合起来，补齐系统综述类期刊。这能捞回那些存在于 Cochrane Database、但在最初的 OpenAlex 检索中排名偏低的论文。

## 正向引文的特例：追踪方法的扩散

如果用户检索式以方法学为重点（例如 "BERT applications in clinical text"（BERT 在临床文本中的应用）），最有价值的信号是**该方法论文的正向引用**，而不是 refs。覆盖默认设置：

```bash
python3 -m scripts.openalex_helper citation-network <BERT_paper_oa_id> \
    --refs-limit 5 --cited-by-limit 100
```

`refs-limit 5` 用来削减噪声；`cited-by-limit 100` 给你扩散范围。然后在重新分级之前，按主题 / 年份过滤 cited_by 的结果。
