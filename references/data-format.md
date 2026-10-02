# 构建数据

UTF-8 JSON 是模型编写内容与 Word 排版之间的接口，不要求学习者手工填写。小型格式测试数据见 [reader.json](../tests/fixtures/reader.json)，仅用于脚本校验；实际交付效果见 [C15 Test 2 Passage 1 精简版 v4](../examples/C15_Test2_Passage1_考点精读_精简版_v4.docx)。

`source` 保存本次处理范围内的完整原文，包括未选入精读的句子。这个 JSON 是排版输入；完整题目、正确题作答、答案来源、学习者画像及待确认项保存在相邻工作记录中，后续跨会话复盘时一并读取。不要为了挤进正文而丢失审计上下文。

## 顶层字段

| 字段 | 内容 |
| --- | --- |
| `meta` | `title` 文稿标题，`subtitle` 文章英文标题，`label` 页脚简称，可选 `note` 一句文稿说明；`answer_label` 默认“你的答案”，原创演示可设为“虚构作答”；可选布尔值 `include_review_sections` 默认 `false`，控制三个独立复盘章节及 `checks` 答案是否输出 |
| `source` | 原文分段列表；每段含 `id` 和 `sentences`，每个原句含全局唯一 `id` 与 `text` |
| `sentences` | 按原文顺序选择的精读条目；通过 `id` 引用原句，不另写可漂移的原句副本 |
| `question_groups` | 可选，默认空列表；已有原题的内部考点与答案对应，保留整组说明及改写引用依据；仅恢复独立章节时输出对照表 |
| `reviews` | 可选，默认空列表；已核实的内部错题复盘，仅恢复独立章节时输出；未提供或无法确认作答时省略或用空列表 |
| `checks` | 可选，默认空列表；独立混合复核或原创错题迁移，仅恢复独立章节时输出题目及文末答案，不在题目旁提示结构名或答案 |
| `paraphrases` | 可选，默认空列表；区分可核对的原题改写与明确标注的原创迁移说法 |
| `vocabulary` | 词汇记录列表，可空 |
| `expressions` | 常用表达列表，可空 |

### 默认输出与恢复开关

`meta.include_review_sections` 必须是 JSON 布尔值，省略或为 `false` 时，默认正文为：**全文典型句精读（含句内考点联系和句后练习）→ 考点表达与同义改写 → 阅读词汇与常见改写 → 常用表达 → 练习参考答案**。三个独立章节“原题考点与快速核对”“错题复盘”“换材料再判断”及 `checks` 对应答案均不输出。所有句后 `practice` 及其参考答案照常保留。

只有用户明确要求恢复上述独立章节时，模型才将此字段设为 `true`，将三个章节按上述名称依次放在精读与考点表达之间，并恢复 `checks` 的文末答案。该字段由模型维护，不要求用户手填。不要将停印章节换个名称并入正文。

停印仅改变输出，不删数据、不跳过题目阅读或选句校准。`question_groups`、`reviews`、`checks` 可留作内部记录，提供时仍须通过校验；原题改写的 `question_numbers` 引用仍须指向存在的 `question_groups` 题号。

不要把未经核实的 OCR 当作确定原文。先回到原 PDF 校正跨行空格、连字符和标点，再写 `source`。如果给句子加 `/` 辅助标记，应在其他输出层处理，`source.text` 始终保存原文；默认构建器直接输出原文。

## 精读条目

```json
{
  "id": "A2",
  "depth": "full",
  "title": "哪些成分共同说明设计方案",
  "translation": "该句的完整译文。",
  "analysis": {
    "core": ["一个简短、可回到原文核对的结构示意"],
    "links": [
      {"anchor": "原句中的定位短语", "relation": "修饰对象或逻辑关系", "detail": "为什么这样判断，以及怎样理解。"}
    ],
    "focus": {"label": "需要补充时才写", "text": "原句与上下文的关键联系。"}
  },
  "practice": [
    {"text": "An original sentence using the same target structure.", "questions": ["一个结构判断问题", "一个含义判断问题"], "answer": "对应回答，并给完整译文。"}
  ]
}
```

`depth` 可省略，省略时与 `"full"` 完全相同，因此原有 JSON 无须迁移。

| `depth` | `analysis.core` | `analysis.links` | `practice` |
| --- | --- | --- | --- |
| `full` | 至少 1 条结构示意 | 至少 1 处关系解释 | 至少 1 道原创练习 |
| `brief` | 可以是空列表 | 至少 1 处关系解释 | 可以是空列表 |

`brief` 表示只需要点明一个局部关系的短说明，不表示学习者已经掌握该句。它仍保留完整原句、译文和关系解释；Word 不显示 `full` 或 `brief` 标签。`core` 为空时不出现空的“结构示意”，没有练习时也不生成空的练习或答案标题。需要时，`brief` 仍可包含结构示意和练习。

`focus` 可省略或为 `null`。各列表的详略由难点决定，不硬凑相同条数；常规完整解析配 1 道练习，确需区分两种结构时可配 2 道。脚本不设选句数量配额。

## 原题考点与快速核对

`question_groups` 按实际题组组织已有原题，可覆盖正确题及错题，不要求存在 `reviews`。每组包含以下字段：

| 字段 | 要求 |
| --- | --- |
| `title` | 非空题组标题 |
| `instruction` | 非空的整组题目说明，保留实际要求，包括适用的字数限制、选项或匹配规则；不要改写成新练习要求 |
| `items` | 至少一题，按原题顺序排列 |

每个 `items` 条目均需包含非空字符串 `number`、`prompt`、`answer`、`explanation`、`action`，以及非空字符串列表 `source_ids`。其中 `prompt` 保留题目内容，`explanation` 说明证据如何支持答案，`action` 说明核对后如何推进判断；`source_ids` 引用本次原文中存在的句子或段落 ID，不能重复。`number` 是题号字符串，在所有题组内必须唯一。同题号如也存在于 `reviews`，其 `correct_answer` 必须与这里的 `answer` 一致；比较时忽略大小写及首尾、连续空白差异，不做语义等价判断。

默认只保存题组数据；`meta.include_review_sections` 为 `true` 时，非空题组才渲染在全部精读句之后、错题复盘之前，新页章名为“原题考点与快速核对”。每组保留二级标题和说明，以“题目／原文依据与答案／核对后怎样推进”三列展示。这里的答案用于已有原题核对，句后练习答案仍统一放在文末；`checks` 答案仅在恢复独立章节时放入文末。

## 错题复盘

```json
{
  "number": "2",
  "prompt": "完整题干",
  "user_answer": "B",
  "correct_answer": "A",
  "evidence": [{"source_ids": ["A2"], "explanation": "这句具体怎样支持答案。"}],
  "paraphrases": ["题干表达 → 原文表达"],
  "why_selected_fails": "所选项与题目确有怎样的联系，又缺少哪项关键条件。",
  "hypothesis": "可选：尚未验证的卡点，明确使用可能、需确认等表达。",
  "check": "可选：一个能验证上述判断的短问题。"
}
```

`evidence.source_ids` 可以引用原文中存在的句子或段落 ID，不必都被选成长难句。简短证据句也可以支持错题。`reviews` 只承载已确定的错误记录；颜色不明、答案未填等情况记在工作笔记并向用户说明，不凭空填一个答案。

默认只保存复盘数据，不在 Word 中输出。`meta.include_review_sections` 为 `true` 时，若某道错题已列入 `question_groups`，仍需保存完整的复盘字段以供核验；Word 只省去该条复盘中重复的 `prompt`、`evidence` 和 `paraphrases`，保留原作答与正确答案、`why_selected_fails`，以及提供的 `hypothesis`、`check`。未列入题组的错题按原格式完整展示。

## 换材料再判断

顶层 `checks` 可省略或设为 `[]`，用于跨结构的混合复核，或围绕已发现问题设计原创迁移题。它与错题记录中的 `review.check` 不同：后者是待核对的问题，`checks` 则是有材料、作答问题和参考答案的独立练习。

```json
{
  "checks": [
    {
      "id": "M1",
      "title": "决定发生了什么变化",
      "text": "The school planned to remove the benches.\n\nAfter pupils explained how they used them, the school kept the benches.",
      "questions": ["学校最后采取了什么措施？", "什么信息改变了原来的决定？"],
      "answer": "学校保留了长椅；学生对长椅用途的说明改变了原来的拆除计划。",
      "source_ids": ["A2", "C"]
    }
  ]
}
```

- `id`、`title`、`text` 和 `answer` 都是非空字符串，`questions` 至少含一个非空问题。标题与问题直接问内容或关系，不提前标注要识别的结构名称。
- `text` 是完整英文材料；自然段之间可用空行（`\n\n`）分隔。原文的 `source.text` 仍必须是一句连续文本，不能因新题支持分段而改变。
- `source_ids` 可省略或为空列表；如提供，必须引用本次 `source` 中存在的句子或段落。它仅用于内部关联，不在检查题旁展示；引用未选入精读的原句也合法。
- 检查题 ID 彼此唯一，并且不能与已有句后练习 ID 冲突。某句只有一道练习时，练习 ID 是句号本身，例如 `A2`；有多道时依次为 `A2.1`、`A2.2`。没有句后练习的 `brief` 不占用练习 ID。
- 默认不输出 `checks` 的题目或答案。仅当 `meta.include_review_sections` 为 `true` 时，题目按输入顺序显示在“错题复盘”之后、“考点表达与同义改写”和“阅读词汇与常见改写”之前，章名为“换材料再判断”；参考答案统一放到文末“练习参考答案”，排在句后练习答案之后，标题为 `检查 M1 决定发生了什么变化`，不会在题目旁提前显示。停印时不可留下孤立的检查题答案，句后练习答案不受影响。

## 考点表达与同义改写

顶层 `paraphrases` 与错题记录内的 `reviews[].paraphrases` 不同。它按条目记录表达及使用边界，每条包含以下字段：

| 字段 | 要求 |
| --- | --- |
| `origin` | `original` 表示已有原题中的改写；`transfer` 表示作者新增的迁移说法 |
| `source_ids` | 非空且不重复的有效原句或段落 ID 列表，不要求句子已选入精读 |
| `source_text` | 非空的原文表达 |
| `target_text` | 非空的原题改写或原创迁移说法，取决于 `origin` |
| `note` | 非空的关系说明与使用边界，避免把语境相关说法误教为无条件同义词 |
| `question_numbers` | 字符串列表；`original` 必须提供至少一个题号，并且每个题号存在于 `question_groups`；`transfer` 只能省略或为空列表 |

非空条目默认在全文精读之后、词汇之前另起新页，章名为“考点表达与同义改写”；恢复独立章节时则位于 `checks` 之后。按“原题改写”和“原创迁移说法”分组，仅生成存在条目的组，以“原文表达／原题改写或原创迁移说法／关系与使用边界”三列展示。说明旁标注原文引用；原题组另标原题号。原创迁移不得冒用原题号。校验器检查字段与引用，不判定语义同义性，也不证明 `target_text` 确实出现在原题；这些需人工对照原文与题目。

`question_groups` 和顶层 `paraphrases` 均可省略或设为 `[]`，不会生成空章节；但 `original` 改写引用的题组不可省略。原有 JSON 无须迁移，省略 `meta.include_review_sections` 即采用默认停印规则。

## 词汇与表达

```json
{
  "vocabulary": [{"term": "account for", "meaning": "占比", "context": "解释本篇中的具体搭配与含义"}],
  "expressions": [{"phrase": "take into account", "meaning": "把某事考虑进去", "example": "The plan takes local needs into account.", "translation": "该计划考虑了当地的需求。"}]
}
```

## 校验与运行

```bash
python scripts/build_reader.py /path/to/reader.json --check-only
python scripts/build_reader.py /path/to/reader.json --output output/精读.docx
```

校验检查字段类型（含 `meta.include_review_sections` 的布尔类型）、必要内容、重复标识、未知引用、`full` 缺失结构示意或练习、检查题与练习 ID 冲突，以及原题答案冲突和改写来源引用等结构问题。它不会判定译文、语法分析、选句适配性、语义同义性或标准答案是否正确，也不会把 `brief` 解释为“已掌握”；这些必须与原文及用户作答人工核对。不要把脚本通过写成教学质量或分数提升的证明。
