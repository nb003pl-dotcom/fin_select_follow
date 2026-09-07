# Selective Compliance in Financial Rule Reasoning: A Solver-Verified Chinese Benchmark and Empirical Analysis

## Dataset Overview

This directory contains the evaluation data used in Table 2 of the paper. The benchmark is derived from Chinese financial examination questions and augments each question with solver-verified synthetic rule interventions. It evaluates whether a language model follows an applicable, answer-changing conflict rule while resisting an inapplicable supplement rule.

The files use UTF-8 encoded JSONL format, with one independent JSON object per line.

## Files and Dataset Size

| File | Setting | JSONL records | Conflict instances | Supplement instances | Table 2 instances |
|---|---|---:|---:|---:|---:|
| `clean.jsonl` | Clean | 126 | 126 | 126 | 252 |
| `noisy.jsonl` | Noisy | 126 | 126 | 0 | 126 |
| `multi_hop.jsonl` | Multi-Hop | 67 | 67 | 67 | 134 |
| **All** | — | **319** | **319** | **193** | **512** |

The number of JSONL records differs from the number of evaluation instances reported in Table 2. For structural consistency, each JSONL record stores two candidate evaluation conditions for the same source question:

1. An applicable **conflict** rule used to compute Obedience Rate (OR).
2. An inapplicable **supplement / ID** rule used to compute Resistance Rate (RR).

Each Clean and Multi-Hop record produces one conflict instance and one supplement instance. The Noisy setting evaluates conflict interventions only. The `conditions.supp` field remains in `noisy.jsonl` to preserve a consistent schema across the three files, but it is excluded from the paper's RR, FR, and Table 2 instance counts. Therefore, the 319 JSONL records correspond to 512 evaluation instances:

```text
126 × 2 (Clean) + 126 × 1 (Noisy) + 67 × 2 (Multi-Hop) = 512
```

All Clean and Multi-Hop supplement conditions included in the evaluation satisfy `conditions.supp.strategy == "S2"`, corresponding to ID (Inapplicable Distractor).

## Top-Level Fields

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique sample identifier, such as `cflue_0166`. The same source question generally retains the same ID across settings. |
| `source` | string | Source dataset. All current records use `cflue`. |
| `subject` | string | Examination subject or financial domain. |
| `task` | string | Question type. Current values include single-choice questions (`单项选择题`) and true/false questions (`判断题`). |
| `question` | string | Original question text. |
| `choices` | object | Mapping from option letters to option text. Single-choice questions generally use A–D, while true/false questions generally use A–B. |
| `answer_orig` | string | Original correct answer without an applicable conflict rule. It is also the expected answer under the supplement/ID condition. |
| `backbone` | object | Structured logical representation of the source question, including facts, parameters, and derivation steps. |
| `intervention` | object | Parameter modifications used to construct the conflict rule, together with reasoning-depth annotations. |
| `conditions` | object | Conflict and supplement contextual-rule conditions associated with the same source question. |
| `closed_book_pass` | boolean | Whether the record passed the closed-book knowledge-stability filter. All submitted records are `true`. |
| `checks` | object | Quality-control metadata produced during construction and verification. |

## `backbone`: Logical Representation

| Field | Type | Description |
|---|---|---|
| `backbone.feasible` | boolean | Whether the source question can be represented by the structured logical backbone. |
| `backbone.facts` | object | Facts extracted from the question. Keys and value types vary by question. |
| `backbone.params` | array<object> | Original rule parameters. Each entry contains the fields described below. |
| `backbone.derivation` | array<object> | Derivation steps from facts and parameters to intermediate variables. This array may be empty for a small number of records. |
| `backbone.choice_logic` | object | Mapping from answer-option letters to logical decision expressions. |

### `backbone.params[]`

| Field | Type | Description |
|---|---|---|
| `name` | string | Parameter variable name. |
| `value` | string / number / boolean / array | Original parameter value. |
| `desc` | string | Natural-language description of the parameter. |
| `law` | string | Description of the regulation, rule, or knowledge source associated with the parameter. |
| `interventable` | boolean | Whether the parameter is eligible for intervention. |

### `backbone.derivation[]`

| Field | Type | Description |
|---|---|---|
| `var` | string | Intermediate variable produced by the derivation step. |
| `expr` | string | Logical expression defined over facts, parameters, or other intermediate variables. |
| `desc` | string | Natural-language description of the derivation step. |

The `facts`, `params`, `derivation`, and `choice_logic` fields support data construction and auditing. During model evaluation, the primary inputs remain the question, answer choices, and contextual rule for the selected condition.

## `intervention`: Conflict-Rule Modification

| Field | Type | Description |
|---|---|---|
| `intervention.strategy` | string | Original construction-strategy label. Current values may include `C1`, `C2`, `C4`, and `C5`. |
| `intervention.hops` | integer | Recorded dependency distance from the rule modification to the final decision. |
| `intervention.param_updates` | object | Mapping from parameter names to updated expressions. Values are stored as strings and may be constants or conditional expressions resembling Python syntax. |

The intervention types displayed in Table 2 do not always correspond directly to the original `strategy` value:

| Original label | Table 2 type | Mapping rule |
|---|---|---|
| `C1` | CR | Constant Replacement |
| `C2` | CB | Conditional Branching |
| `C4` | EI | Exception Insertion |
| `C5` | CR or CB | Mapped to CB if any `param_updates` expression contains a conditional AST `IfExp`, generally written as `x if condition else y`; otherwise mapped to CR. |

`param_updates` contains annotations and must not be passed directly to `eval()` when processing untrusted data.

## `conditions`: Paired Evaluation Conditions

### `conditions.conflict`

The conflict rule applies to the current question and modifies a relevant rule parameter. Under this condition, the model should follow the contextual rule and produce the updated answer.

| Field | Type | Description |
|---|---|---|
| `text` | string | Conflict contextual-rule text provided to the model. |
| `answer_new` | string | Expected answer after applying the conflict rule, used to compute OR. |
| `variants` | object | Auxiliary textual variants of the conflict rule. |
| `buried` | boolean, optional | Appears only in `noisy.jsonl`. All current values are `true`, indicating that the target rule is embedded in a Noisy/Buried presentation. |

The `variants` object may contain:

| Field | Type | Description |
|---|---|---|
| `hypothetical` | string | Rule modification expressed as a hypothetical condition. |
| `contrast` | string | Contrastive statement of the original and updated rules. |
| `paraphrases` | array<string>, optional | Paraphrases of the conflict rule. Some Multi-Hop records omit this field or contain an empty array. |

### `conditions.supp`

The supplement rule is topically related to the source question, but its activation condition is not satisfied. The model should resist the distractor and retain the original answer. The paper evaluates this condition only in the Clean and Multi-Hop settings; the corresponding field in Noisy records is excluded from all reported metrics.

| Field | Type | Description |
|---|---|---|
| `strategy` | string | Supplement construction strategy. All three submitted files use `S2`; only the Clean and Multi-Hop conditions contribute to the ID counts in Table 2. |
| `text` | string | Inapplicable distractor rule provided to the model. |
| `target_distractor` | string | Option that the distractor attempts to induce. It is used only during construction and diagnosis and is not the correct answer under the supplement condition. |

## `checks`: Quality-Control Metadata

| Field | Type | Description |
|---|---|---|
| `roundtrip` | boolean | Whether the structured backbone reproduces the original question answer through round-trip solving. |
| `blind_verify` | string | Blind-verification mode recorded during construction. Current values are `single` or `dual`. |
| `supp_verify` | boolean | Whether the supplement rule passed the inapplicability verification. |

All submitted records have `roundtrip == true` and `supp_verify == true`.

## Evaluation Labels

Run both conditions for Clean and Multi-Hop records, but run only the conflict condition for Noisy records. The expected labels are obtained as follows:

```python
conflict_rule = item["conditions"]["conflict"]["text"]
conflict_expected = item["conditions"]["conflict"]["answer_new"]

supplement_rule = item["conditions"]["supp"]["text"]
supplement_expected = item["answer_orig"]
```

- **OR:** Accuracy against `answer_new` under the conflict condition.
- **RR:** Accuracy in retaining `answer_orig` under the supplement/ID condition for Clean and Multi-Hop. Noisy RR is undefined.
- **FR:** Harmonic mean of OR and RR. Because Noisy RR is undefined, Noisy FR is also undefined.

## Loading Example

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

All three files use strict one-object-per-line JSONL formatting and can be loaded directly by the `run` subcommand in `submit_code/methods.py`.
