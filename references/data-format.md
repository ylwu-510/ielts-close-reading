# 构建数据

UTF-8 JSON 是模型编写内容与 Word 排版之间的接口，不要求学习者手工填写。完整可运行示例见 [demo-reader.json](../examples/demo-reader.json)。

`source` 保存本次处理范围内的完整原文，包括未选入精读的句子。这个 JSON 是排版输入；完整题目、正确题作答、答案来源、学习者画像及待确认项保存在相邻工作记录中，后续跨会话复盘时一并读取。不要为了挤进正文而丢失审计上下文。

## 顶层字段

| 字段 | 内容 |
| --- | --- |
| `meta` | `title` 文稿标题，`subtitle` 文章英文标题，`label` 页脚简称，可选 `note` 一句文稿说明；`answer_label` 默认“你的答案”，原创演示可设为“虚构作答” |
| `source` | 原文分段列表；每段含 `id` 和 `sentences`，每个原句含全局唯一 `id` 与 `text` |
| `sentences` | 按原文顺序选择的精读条目；通过 `id` 引用原句，不另写可漂移的原句副本 |
| `reviews` | 已核实的错题复盘；未提供或无法确认作答时用空列表 |
| `checks` | 可选，默认空列表；换材料后的混合复核或原创错题迁移，不在题目旁提示结构名或答案 |
| `vocabulary` | 词汇记录列表，可空 |
| `expressions` | 常用表达列表，可空 |

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
- 题目按输入顺序显示在“错题复盘”之后、“阅读词汇与常见改写”之前，章名为“换材料再判断”。参考答案统一放到文末“练习参考答案”，排在句后练习答案之后，标题为 `检查 M1 决定发生了什么变化`，不会在题目旁提前显示。

## 词汇与表达

```json
{
  "vocabulary": [{"term": "account for", "meaning": "占比", "context": "解释本篇中的具体搭配与含义"}],
  "expressions": [{"phrase": "take into account", "meaning": "把某事考虑进去", "example": "The plan takes local needs into account.", "translation": "该计划考虑了当地的需求。"}]
}
```

## 校验与运行

```bash
python scripts/build_reader.py examples/demo-reader.json --check-only
python scripts/build_reader.py examples/demo-reader.json --output output/demo.docx
```

校验检查字段类型、必要内容、重复标识、未知引用、`full` 缺失结构示意或练习，以及检查题与练习 ID 冲突等结构问题。它不会判定译文、语法分析、选句适配性或标准答案是否正确，也不会把 `brief` 解释为“已掌握”；这些必须与原文及用户作答人工核对。不要把脚本通过写成教学质量或分数提升的证明。
