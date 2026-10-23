# 停止决策

*本文件由主 Agent 在 `SKILL.md` 的 STEP 8 读取，即 `discovery_curve.py` 产出覆盖率估计之后。该决策结合 `curve.json`（仅供参考）、档级预算和用户意图——最终决定权在主 Agent。*

## 决策输入

| 输入 | 来源 | 含义 |
|-------|--------|---------|
| `coverage_estimate` | `curve.json` | 0.0-1.0；你已覆盖的相关文献量的估计值 |
| `ci_lower`, `ci_upper` | `curve.json` | 95% 区间；随着更多相关论文在多次检索中被反复找到，区间会收窄 |
| `method` | `curve.json` | `sample_coverage` = 由本次运行的检索估算而来；`prior` = 无可测对象（少于两次检索、没有找到相关论文，或相关论文太少、说明不了问题），因此这个数字是取自 Undermind 中位数先验并封顶 50% 的值，而不是证据 |
| `relevance_band` | `curve.json` | 该估计所用的 rcs 下限：7；当 rcs ≥ 7 的论文太少时为 6（例如只有两篇，各自被重复找到或只被找到一次，否则会被读成 100%） |
| `papers_evaluated` | KG 长度 | 已完成分级了多少篇论文 |
| `tier_budget` | `tier_decision.md` | Quick 60 / Standard 180 / Deep 400 / Audit 1000 |
| `user_intent` | 来自你对检索式的理解 | 已知主题（高先验）vs. 探索（低先验） |

## 决策矩阵（覆盖率 × 剩余预算）

| 覆盖率 \ 预算 | **还很多**（已用 < 50%） | **偏紧**（已用 50-80%） | **已耗尽**（已用 ≥ 80%） |
|---------------------|-----------------------------------|--------------------------|----------------------------|
| **高**（> 0.85） | 停止 —— 边际产出递减 | 停止 | 停止 |
| **中**（0.6 - 0.85） | 扩展 1 跳引文 | 停止或扩展 1 跳（看用户意图） | 停止 |
| **低**（< 0.6） | 扩展引文、放宽策略 | 再加 1 个策略然后停止 | 停止 |

## 决策树（自上而下套用，先匹配者胜）

```
1. User explicitly said "enough" / "stop" / "I have what I need"?
   YES → Stop. Write report. (Honor explicit user requests above all.)

2. papers_evaluated >= 80% of tier_budget?
   YES → Stop. Budget protection: never blow past 1.5× budget.

3. method == "prior"?
   → There is no evidence yet: decide by tier budget. For Standard+, the planned
     citation hop (STEP 9) adds the second search the estimate needs; re-check after it.

4. coverage_estimate > 0.85?
   YES → Stop. Quote the interval; further expansion mostly re-finds known papers.

5. coverage_estimate in [0.6, 0.85]?
   → Expand citations 1 hop from top-rcs papers (STEP 9), then re-check.

6. coverage_estimate < 0.6?
   → Multiple options. Pick based on tier:
      - Quick: stop (curve is unreliable with <50 papers anyway)
      - Standard: expand citations + add 1 strategy (recency)
      - Deep: expand 2 hops + add strategies (seminal + reviews)
      - Audit: expand 2 hops + journal whitelist + ask user about other databases

7. None of above match?
   → Default to "expand 1 hop" if tier allows; otherwise stop.
```

## 何时询问用户（覆盖率 0.7 - 0.85，预算约 50%）

这个区间确实是模棱两可。与其猜，不如把数字告诉用户：

> "覆盖率估计是 0.78（CI 0.71-0.84）。你在 Standard 档下已经检索到 95/180 篇（占预算 53%）。我可以：
>   (a) 就在这里停下并写报告——高引核心部分你已经有不错的覆盖
>   (b) 从 top-15 论文扩展 1 跳引文，并再增加约 40-80 篇，让图景更完整
>
> 你选哪个？默认是 (a)，因为你的覆盖率已经 > 0.75。"

等用户回复。没有回复时的默认动作是**停止**（选项 a）——对预算要保守。

## 边界情况

### 用户在检索途中明确叫停
- "Stop"（停）、"够了"、"I have enough"（我够了）、"this is plenty"（这些就够了）→ 立即停止，按当前 KG 状态写报告
- 即使覆盖率 < 0.5，也不要继续扩展

### 接近预算上限（papers_evaluated ≈ 预算的 80%）
- 主动停止，不要询问——说 *"已接近 Standard 档预算（170/180 篇）。在这里停止并写报告。"*
- 绝不超过档级预算的 1.5 倍（例如 Standard 档的 > 270 篇）——那是另一个档级，需要用户重新确认

### 相关论文很少（只有少数几篇评分 ≥ 7）
- 这个估计取决于那几篇被重复找到的频率；它的区间很宽
- 当它们少到说明不了问题时，`curve.json` 会如实写明：`relevance_band: 6`（改为在 rcs ≥ 6 上估计）或 `method: prior`。两种情况下核心都很薄——Standard 及以上档级应从这些论文出发追溯引文（STEP 9），而不是停下
- Quick 档经常就停在这里——检索完即停，并把这个数字连同它的区间一起引用

### curve.json 缺失
- 按低覆盖率处理（< 0.6）；落到决策分支 6
- 给用户记一条警告：*"覆盖率估计不可用，改按档级预算推进。"*

## 发现曲线仅供参考，不是权威

`coverage_estimate` 是本次运行中各次检索的样本覆盖率：`raw/` 里的不同检索（各策略、综述、中文来源、引文跳）有多大概率再次找到同样的 rcs ≥ 7 论文（`relevance_band` 为 6 时是 rcs ≥ 6）。`1 - coverage` 大致就是"再来一次同类检索能翻出一篇你还没有的相关论文"的概率。刻意不同的检索（经典 vs. 近期、另一种语言）会拉低它——这是真实情况，不是 bug。它**是一个信号，不是判决**。以下情况要推翻曲线的结论：

- **用户意图是探索性的**（"I don't know this field at all"（我完全不了解这个领域））→ 估计可能低估了你已掌握的核心量，因为用户要的是广度而不只是核心。再推 1 跳。
- **用户意图是已知主题**（"I just need the seminal papers on prospect theory"（我只需要前景理论的奠基论文））→ 曲线可能高估。如果 rcs ≥ 8 的论文已有充分覆盖，可以更早停下。
- **Audit 档** → 曲线仅供参考；无论覆盖率如何，你都必须跑完整的 Audit 流程（期刊白名单 + 2 跳）。在 Audit 档提前停止会让它失去意义。

## 停止时要告诉用户什么

在 STEP 14 面向用户的消息里，简短说明停止的理由：

> "在 142 篇处停止，覆盖率 0.87。高 rcs 核心（15 篇 ≥ 7）覆盖良好，曲线置信区间也很紧（CI 0.84-0.91）。再往后的引文扩展只会带来边缘文献。"

Audit 档停止时说：

> "已完成完整的 Audit 流程：评估 487 篇论文，其中 89 篇 rcs ≥ 6，已应用期刊白名单 Cochrane + medical_top，并对 top-23 论文做了 2 跳引文追溯。coverage 0.83（Audit 档这是正常的——尾部很长）。"

## 需要避免的常见错误

不要为了追逐更高的覆盖率，无限循环"扩展 → 重新分级 → 扩展 → 重新分级"。如果两轮扩展之后覆盖率提升不到 0.05，**停止**。你追逐的是边际产出递减——多出来的那些论文是噪声。
