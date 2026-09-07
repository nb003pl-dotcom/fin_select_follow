# Selective Compliance in Financial Rule Reasoning: A Solver-Verified Chinese Benchmark and Empirical Analysis

## English Description

This directory contains the evaluation data used in Table 2 of the paper. The benchmark is derived from Chinese financial examination questions and augments each question with solver-verified synthetic rule interventions. It evaluates whether a language model follows an applicable, answer-changing conflict rule while resisting an inapplicable supplement rule.

The data are organized into three settings: Clean, Noisy, and Multi-Hop. The three JSONL files contain 319 records in total: 126 Clean records, 126 Noisy records, and 67 Multi-Hop records. Clean and Multi-Hop each evaluate both conflict and supplement interventions, whereas Noisy evaluates conflict interventions only. Consequently, the 319 JSONL records correspond to 512 evaluation instances reported in the paper. Each JSONL line is an independent UTF-8 encoded JSON object containing the source question, answer choices, original answer, executable logical backbone, synthesized interventions, expected answers, and verification metadata.

## Dataset README

本目录包含论文 Table 2 使用的三个评测数据文件。文件采用 UTF-8 编码的 JSONL 格式，每行是一个独立 JSON 对象。

## 文件与规模

| 文件 | 实验设置 | JSONL 记录数 | Conflict 实例 | Supplement 实例 | Table 2 实例数 |
|---|---|---:|---:|---:|---:|
| `clean.jsonl` | Clean | 126 | 126 | 126 | 252 |
| `noisy.jsonl` | Noisy | 126 | 126 | 0 | 126 |
| `multi_hop.jsonl` | Multi-Hop | 67 | 67 | 67 | 134 |
| **All** | — | **319** | **319** | **193** | **512** |

这里的“JSONL 记录数”和 Table 2 的“实例数”含义不同。每条 JSON 记录在统一的数据结构中保存同一道题的两个候选评测条件：

1. 一个适用于题目的 **conflict** 规则，用于计算 Obedience Rate（OR）；
2. 一个不适用于题目的 **supplement / ID** 规则，用于计算 Resistance Rate（RR）。

Clean 和 Multi-Hop 的每条记录均产生一个 conflict 实例和一个 supplement 实例。Noisy 只评测 conflict intervention；`noisy.jsonl` 中保留的 `conditions.supp` 仅用于维持三个文件的数据结构一致，不计入论文的 RR、FR 或 Table 2 实例数。因此，319 条 JSONL 记录对应论文中的 512 个评测实例：

```text
126 × 2 (Clean) + 126 × 1 (Noisy) + 67 × 2 (Multi-Hop) = 512
```

计入 supplement 评测的 Clean 和 Multi-Hop 记录均满足 `conditions.supp.strategy == "S2"`，对应 ID（Inapplicable Distractor）。

## 顶层字段

| 字段 | 类型 | 描述 |
|---|---|---|
| `id` | string | 样本唯一标识，例如 `cflue_0166`。同一基础题在不同实验设置中通常沿用相同 ID。 |
| `source` | string | 原始题目来源；当前数据均为 `cflue`。 |
| `subject` | string | 考试科目或专业领域。 |
| `task` | string | 题型，当前包括 `单项选择题` 和 `判断题`。 |
| `question` | string | 原始问题文本。 |
| `choices` | object | 选项字典，键为选项字母，值为选项文本。单项选择题通常有 A–D，判断题通常有 A–B。 |
| `answer_orig` | string | 不使用适用冲突规则时的原始正确答案，也是 supplement/ID 条件的期望答案。 |
| `backbone` | object | 将原题表示为事实、参数和推导步骤的结构化逻辑骨架。 |
| `intervention` | object | conflict 规则所实施的参数修改及推理跳数标注。 |
| `conditions` | object | 同一基础题对应的 conflict 和 supplement 两个外部规则条件。 |
| `closed_book_pass` | boolean | 样本是否通过闭卷知识稳定性筛选。当前提交数据中均为 `true`。 |
| `checks` | object | 数据构造和验证阶段留下的质量检查字段。 |

## `backbone`：逻辑骨架

| 字段 | 类型 | 描述 |
|---|---|---|
| `backbone.feasible` | boolean | 原题是否可被当前结构化骨架表达。 |
| `backbone.facts` | object | 从题目抽取的事实变量。键名和取值类型随题目而变化。 |
| `backbone.params` | array<object> | 原规则参数列表。每个元素包含下表所列字段。 |
| `backbone.derivation` | array<object> | 从事实和参数到中间变量的推导步骤；少数样本可以为空数组。 |
| `backbone.choice_logic` | object | 选项字母到逻辑判定表达式的映射。 |

### `backbone.params[]`

| 字段 | 类型 | 描述 |
|---|---|---|
| `name` | string | 参数变量名。 |
| `value` | string / number / boolean / array | 原规则中的参数值。 |
| `desc` | string | 参数的自然语言说明。 |
| `law` | string | 参数对应的法规、准则或知识来源说明。 |
| `interventable` | boolean | 该参数是否被标记为可干预。 |

### `backbone.derivation[]`

| 字段 | 类型 | 描述 |
|---|---|---|
| `var` | string | 推导得到的中间变量名。 |
| `expr` | string | 使用事实、参数或其他中间变量表示的逻辑表达式。 |
| `desc` | string | 该推导步骤的自然语言说明。 |

`facts`、`params`、`derivation` 和 `choice_logic` 是用于数据构造与审计的形式化信息。运行论文方法时，模型的主要输入仍是题目、选项和相应条件下的规则文本。

## `intervention`：冲突规则修改

| 字段 | 类型 | 描述 |
|---|---|---|
| `intervention.strategy` | string | 原始构造策略标签，当前可能为 `C1`、`C2`、`C4` 或 `C5`。 |
| `intervention.hops` | integer | 构造阶段记录的规则修改到最终判断之间的推理跳数。 |
| `intervention.param_updates` | object | 参数名到新表达式的映射。值以字符串保存，可能是常量，也可能是类似 Python 条件表达式的形式化表达式。 |

Table 2 使用的展示类型并不总是直接等于原始 `strategy`：

| 原始构造标签 | Table 2 类型 | 映射规则 |
|---|---|---|
| `C1` | CR | Constant Replacement |
| `C2` | CB | Conditional Branching |
| `C4` | EI | Exception Insertion |
| `C5` | CR 或 CB | 若任一 `param_updates` 表达式包含条件表达式（AST `IfExp`，通常写作 `x if condition else y`），映射为 CB；否则映射为 CR。 |

`param_updates` 是数据标注，不应直接对不可信数据调用 `eval()`。

## `conditions`：配对评测条件

### `conditions.conflict`

Conflict 规则适用于当前题目，并改变相关规则参数。模型在该条件下应遵从外部规则并给出新答案。

| 字段 | 类型 | 描述 |
|---|---|---|
| `text` | string | 提供给模型的 conflict 外部规则文本。 |
| `answer_new` | string | 应用 conflict 规则后的期望答案，用于计算 OR。 |
| `variants` | object | conflict 规则的辅助文本变体。 |
| `buried` | boolean，可选 | 仅在 `noisy.jsonl` 中出现，当前均为 `true`，表示目标规则位于 Noisy/Buried 呈现设置中。 |

`variants` 的子字段：

| 字段 | 类型 | 描述 |
|---|---|---|
| `hypothetical` | string | 以假设形式表达规则修改。 |
| `contrast` | string | 对比原规则与新规则的表述。 |
| `paraphrases` | array<string>，可选 | conflict 规则的改写版本；部分 Multi-Hop 记录不含该字段，也可能为空数组。 |

### `conditions.supp`

Supplement 规则与当前题目主题相关，但其适用前提不满足当前题目；模型应抵抗该规则的干扰并保持原答案。论文仅在 Clean 和 Multi-Hop 设置中评测该条件；Noisy 记录中的该字段不参与论文指标计算。

| 字段 | 类型 | 描述 |
|---|---|---|
| `strategy` | string | Supplement 构造策略；三个提交文件中均为 `S2`。其中只有 Clean 和 Multi-Hop 的该条件计入 Table 2 的 ID。 |
| `text` | string | 提供给模型的 inapplicable distractor 规则文本。 |
| `target_distractor` | string | 该干扰规则试图诱导的选项，仅用于构造与诊断；它不是 supplement 条件的正确答案。 |

## `checks`：质量检查

| 字段 | 类型 | 描述 |
|---|---|---|
| `roundtrip` | boolean | 结构化骨架与题目/答案是否通过回译一致性检查。 |
| `blind_verify` | string | 构造阶段记录的盲审验证模式标签，当前值为 `single` 或 `dual`。 |
| `supp_verify` | boolean | Supplement 规则是否通过不适用性验证。 |

当前提交数据中 `roundtrip` 和 `supp_verify` 均为 `true`。

## 评测标签

对 Clean 和 Multi-Hop 记录分别运行两个条件；对 Noisy 记录只运行 conflict 条件。期望标签如下：

```python
conflict_rule = item["conditions"]["conflict"]["text"]
conflict_expected = item["conditions"]["conflict"]["answer_new"]

supplement_rule = item["conditions"]["supp"]["text"]
supplement_expected = item["answer_orig"]
```

- OR：模型在 conflict 条件下预测 `answer_new` 的准确率。
- RR：模型在 Clean 和 Multi-Hop 的 supplement/ID 条件下保持 `answer_orig` 的准确率；Noisy RR 不定义。
- FR：OR 与 RR 的调和平均数；由于 Noisy RR 不定义，Noisy FR 也不定义。

## 读取示例

```python
import json
from pathlib import Path

path = Path("submit_code/data/clean.jsonl")
with path.open(encoding="utf-8") as handle:
    records = [json.loads(line) for line in handle if line.strip()]

print(len(records))                         # 126
print(records[0]["question"])
print(records[0]["conditions"]["conflict"]["text"])
print(records[0]["conditions"]["supp"]["text"])
```

三个文件已经规范化为严格的一行一个 JSON 对象，可直接由 `submit_code/methods.py` 的 `run` 子命令读取。
