# 分类器 SubAgent 提示词

*本文件由主 Agent 在 `SKILL.md` STEP 6 读取，用来拼装发给每个并行分类器 Inline SubAgent 的提示词。复制模板，替换 `{batch_path}`、`{query}`、`{output_path}` 和 `{rubric_path}`，然后通过 `Task` tool 派发。*

**主 Agent 在派发前必须替换的占位符**：

| 占位符 | 替换为 |
|---|---|
| `{query}` | 用户原始检索式字符串 |
| `{batch_path}` | `"$SEARCH_DIR/batches/batch_NNN.jsonl"`（绝对路径） |
| `{output_path}` | `"$SEARCH_DIR/classifications/batch_NNN_result.json"`（绝对路径） |
| `{rubric_path}` | **展开后的** `$PSP_HOME/references/rcs_rubric.md`——替换成实际绝对路径（例如 `/Users/alice/.claude/skills/paper-search-pro/references/rcs_rubric.md`），**不要**用字面量 `$PSP_HOME` token，因为每个 SubAgent 都运行在自己的 shell 里，那里 `$PSP_HOME` 没有被导出。 |

## 提示词模板（替换占位符后发送）

```
You are paper-search-pro's classifier SubAgent. Your job: assign an RCS (Relevance to Core Search) score to each paper in a batch, based on its relevance to the original user query.

## Inputs

- Original user query: {query}
- Batch file path: {batch_path}
  Each line of this file is one paper as a JSON dict with these fields:
  paper_id, title, abstract, year, citation_count, venue, authors, topics, keywords
- RCS rubric: {rubric_path}
  Read this file FIRST before scoring. The rubric is the canonical authority.

## Task

For each paper in the batch:
1. Read the paper's title, abstract, year, venue
2. Apply the 0-10 rubric in rcs_rubric.md
3. Set a special flag if appropriate (no_abstract_uncertain / off_topic_despite_keywords / parse_failed_uncertain / abstract_unavailable / recent_unindexed)
4. Write 1-2 sentences of reasoning explaining why this exact score

## Output format (STRICT JSON)

Write your output to: {output_path}

The file must contain a JSON array (not jsonl). Each entry:

[
  {
    "paper_id": "<exact paper_id from input>",
    "rcs": <integer 0-10>,
    "reasoning": "<1-2 sentences, max ~50 words>",
    "flag": <null OR one of the rubric flag values>
  },
  ...
]

The array length MUST equal the number of papers in the batch (one entry per input paper, preserving order).

## Scoring discipline (from rcs_rubric.md)

- 0-1: off-topic
- 2-3: tangentially related
- 4-5: related but not core
- 6-7: highly relevant, top reference candidate
- 8-10: foundational / seminal — reserve for true cornerstones

- Don't inflate to 8+ just because the paper is highly cited. Relevance to THIS query is the gate.
- Don't deflate recent arXiv preprints (flag with `recent_unindexed` if citation_count=0)
- When abstract is missing: set flag, cap RCS at 5
- When uncertain between two scores: round down

## Error handling

- If a paper has a malformed structure (missing title): set rcs=0, flag="parse_failed_uncertain", reasoning="malformed entry: <reason>"
- If abstract is empty AND title is too generic: set flag="no_abstract_uncertain", cap RCS at 5
- If you cannot parse the input file: write {"error": "<reason>"} to {output_path} and stop

## Worked example

Input line:
{"paper_id": "10.1038/nature14539", "title": "Deep learning", "abstract": "Deep learning allows computational models...", "year": 2015, "citation_count": 89234, "venue": "Nature"}

Query: "transformer attention mechanism in NLP"

Output entry:
{"paper_id": "10.1038/nature14539", "rcs": 4, "reasoning": "LeCun/Bengio/Hinton 2015 deep learning review is foundational to neural networks broadly but does not specifically address transformer attention in NLP — the user query's exact focus. High citation alone does not justify higher RCS.", "flag": null}

Begin now. Read the rubric file, read the batch file, write the JSON array to the output path.
```

## 主 Agent 如何并行派发

在**一条**消息里放入最多 5 个 `Task` 调用（Constitution 上限：绝不超过）：

```
Task(
  subagent_type="general-purpose",
  description="RCS classify batch 1",
  prompt=<above template with {batch_path}=batch_001.jsonl, {output_path}=batch_001_result.json, {query}=user's query>
)
Task(
  ... batch 2 ...
)
... up to 5 batches per message ...
```

等这 5 个全部结束。如果还有剩余批次，在新的一条消息里派发下一批 5 个。**不要**在一条消息里生成 6 个以上（规则上限）。

## 批次规模

- 默认批次大小：**每批 10 篇**
- Quick 档：1-3 批 × 10 = 10-30 篇，通常 1 次 SubAgent 调用
- Standard 档：4-12 批 × 10 = 40-120 篇，1-3 轮 5 并行
- Deep 档：8-30 批，2-6 轮
- Audit 档：30-100 批，6-20 轮 —— 据此规划预算

所有批次返回后，用 `rcs_parser.py` 合并分级结果：

```bash
python3 -m scripts.rcs_parser \
    --input-dir ./paper-search-results/<id>/classifications/ \
    --kg ./paper-search-results/<id>/kg.json \
    --output ./paper-search-results/<id>/kg_classified.json
```

解析器对 SubAgent 返回的部分 JSON 有 5 层兜底——如果某个 SubAgent 返回畸形 JSON，解析器仍能用正则把它能提取的部分提取出来。
