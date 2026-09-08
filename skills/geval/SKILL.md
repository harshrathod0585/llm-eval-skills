---
name: geval
description: Build custom LLM-as-judge metrics that are stable enough to track across runs, using G-Eval. Covers why naive "score this 1-10" judging swings wildly between identical runs, what G-Eval's chain-of-thought evaluation steps and probability-weighted scoring actually fix, how evaluation_params determines what a metric really measures, criteria vs evaluation_steps vs rubric, and worked correctness / completeness / style metrics. Use whenever someone needs a metric that isn't built in — correctness, completeness, style, tone, helpfulness, coherence, domain-specific quality, or a custom safety check — or mentions G-Eval, LLM-as-judge, DeepEval custom metrics, or judge rubrics. Also use when someone's LLM judge gives inconsistent scores on unchanged inputs, when they're deciding which test-case fields a metric should compare, or when they want to combine several quality metrics into one score.
created_at: 2026-09-08T12:27:56Z
updated_at: 2026-09-08T12:27:56Z
---

# G-Eval: Custom Judgment Metrics

## Why the count-based metrics run out

Faithfulness, answer relevancy, contextual recall/precision/relevancy all share one mechanism: decompose into claims, check each claim, compute a ratio. That works when a claim can be judged true or false in isolation.

It breaks for properties that only exist at the level of the whole answer:

- **Style** — no individual sentence is "in the house style"; the register is a property of the whole.
- **Correctness** — an analogy extracted as a standalone claim gets falsely penalized as unrelated to the golden answer, because analogies only make sense read in place.
- **Completeness** — it's about what's *absent*, and you can't count the claims that aren't there.

These need judgment: something reading the whole answer and assigning a holistic score.

## Why naive LLM-as-judge doesn't work

The obvious approach — send {question, expected answer, actual answer} plus a one-line criterion to GPT-4 and ask for a score out of 10 — fails in practice. Not because the judge makes mistakes (assume a good judge; the problem remains) but because **variance across repeated runs on identical inputs is enormous**. Scores swing 60% → 70% → 75% with nothing changed. That's unusable for regression tracking, which is the whole point of having a metric.

Two root causes, both about the judge's reasoning being underconstrained:

1. **Loose criteria.** One vague sentence doesn't pin down what "correctness" means. Each fresh call is free to interpret it from a different angle, so the score drifts.
2. **Direct integer output.** Asking for a single digit makes the model emit one token. Under the hood it holds a distribution — say 8 at 51%, 7 at 40%, 6 at 9% — and greedily emits the argmax. A tiny shift in probability mass on a re-run flips 8 to 7 even though nothing about the input changed.

## What G-Eval actually does

Two innovations. It is still LLM-as-judge — nothing more exotic.

1. **Chain-of-thought expansion of criteria into numbered evaluation steps**, generated once before scoring. Every subsequent judge call follows the same explicit rubric instead of re-deriving its own reading of a loose instruction. This is the sense in which it's "deterministic": it constrains the judge's reasoning scope.

2. **Probability-weighted scoring.** Instead of taking the greedily-decoded token, it reads the top-N token log-probabilities at the scoring position, keeps the numeric tokens, normalizes them to sum to 1, and takes the weighted average. A distribution of 8:70%, 7:20%, 9:5% normalizes to ~0.73/0.21/0.05 and yields 7.84 rather than a bare 8. Between runs that moves to 7.4 or 7.9 — small continuous drift instead of integer jumps.

The original paper's own worked example: greedy decoding gives 3, weighted log-prob gives 2.59.

## Configuration

```python
from deepeval.metrics import GEval
from deepeval.test_case import LLMTestCaseParams

correctness = GEval(
    name="Correctness",
    evaluation_steps=[
        "Compare the actual output against the key facts in the expected output.",
        "Heavily penalize statements in the actual output that contradict the expected output.",
        "Reward statements matching the expected output in meaning, regardless of wording.",
        "Do not penalize the actual output for omitting information — only wrong statements count.",
        "Additional correct information must never lower the score.",
    ],
    evaluation_params=[
        LLMTestCaseParams.INPUT,
        LLMTestCaseParams.ACTUAL_OUTPUT,
        LLMTestCaseParams.EXPECTED_OUTPUT,
    ],
    model="gpt-4o-mini",
    threshold=0.7,
)
```

| Parameter | Notes |
|---|---|
| `name` | label only |
| `criteria` | one high-level sentence; G-Eval runs CoT to derive its own steps from it |
| `evaluation_steps` | hand-written numbered rubric; **skips** the CoT step entirely |
| `evaluation_params` | which test-case fields the judge may read — **this determines what's measured** |
| `model` | judge model; the paper reports best results with GPT-4-class |
| `threshold` | normalized score cutoff for pass/fail |
| `rubric` | optional score-band definitions; takes interpretation control away from the judge |
| `strict_mode` | `True` returns a hard boolean from raw output, bypassing weighted scoring — leave `False` |

**`criteria` vs `evaluation_steps`:** passing `criteria` means the step-generation is itself a fresh LLM call each time, reintroducing a little variance. Hand-written `evaluation_steps` are byte-identical every call, so that source of variance goes to zero.

The workflow that works: start with `criteria` while you don't yet understand how your system fails — let the CoT draft something reasonable. Once you've run the eval a few times and can see the failure patterns, graduate to writing your own steps. Once mature, supply both `evaluation_steps` and `rubric`.

## evaluation_params determines the metric

This is the part people get wrong. G-Eval hardcodes no comparison. Only the fields you list are rendered into the judge prompt; anything omitted is invisible even if populated on the test case.

| Metric | Params | Effectively compares |
|---|---|---|
| Correctness | INPUT, ACTUAL_OUTPUT, EXPECTED_OUTPUT | answer vs ground truth |
| Completeness | INPUT, ACTUAL_OUTPUT, EXPECTED_OUTPUT | coverage of what was asked |
| Style | INPUT, ACTUAL_OUTPUT | register only, no reference |

Two failure modes to avoid: every listed field must be non-`None` or the run errors; and stacking `EXPECTED_OUTPUT` onto a style metric makes the judge silently start grading correctness too.

## The three application metrics

### Correctness

Distinct from faithfulness: faithfulness asks whether the answer is grounded in retrieved context, correctness asks whether it's actually right.

A characteristic first-pass failure: the metric scores 66% with 7 of 15 failing, and reading the reasons shows the judge penalizing answers for *incompleteness* — the golden answers were written exhaustively, the generated answers were correct but shorter. The criteria said not to penalize omission; the judge did anyway.

The fix is to make it explicit and add a rubric:

```python
rubric=[
    Rubric(score_range=(0, 4), expected_outcome="Clear factual errors."),
    Rubric(score_range=(5, 8), expected_outcome="Mostly correct, one or two small inaccuracies."),
    Rubric(score_range=(9, 10), expected_outcome="All claims factually correct."),
]
```

plus steps stating that brevity must not be deducted for. That moves 66% → 84%, 14/15 passing. Re-running gives 83% with the *same* single case failing — which is the demonstration of G-Eval's stability. Naive LLM-as-judge would have swung far more.

### Completeness

Does the answer cover every distinct part of a multi-part question?

Implementation is just another `GEval` object in the same `evaluate()` call, with its own steps and rubric.

When this scores badly, check the generator's system prompt before touching the rubric. A prompt saying "stick closely to the context, don't talk too much" causes the generator to truncate multi-part answers — that's a pipeline bug, not a metric calibration problem. Adding "identify every distinct part of the question and address all of them rather than stopping at the first" (while still forbidding invention beyond context) moves completeness 68% → 75% with 14/15 passing.

### Style

Reference-free — no `expected_output` at all. The rubric itself encodes the standard.

Two lessons from tuning it. First, if the generator was never told to write in the target style, expect a bad baseline (~54%) — that's not a metric problem. Second, watch for rubric over-correction: a rubric rewarding "concrete example or analogy" caused the judge to demand an analogy in *every* answer, penalizing ones that didn't need one. The correction is to make it conditional — "an analogy is a bonus when the concept is abstract, but a clear direct answer is fully acceptable without one."

**Stop tuning style before it hits the ceiling.** Pushing style up via generator prompt reliably pulls faithfulness down. ~74% with a threshold loosened to 0.6 is a reasonable landing place.

## Do not combine into a weighted sum

Track correctness, completeness, and style separately. Each exists because it illuminates a different failure mode; a composite score destroys exactly the diagnostic information you built them to get. If a stakeholder wants one number, hand them the table instead.

## Working practice

- Read the `reason` on every failure before deciding anything. Every worthwhile fix in practice comes from that text, not from guessing.
- Distinguish "the eval is miscalibrated" from "the pipeline is broken." Correctness over-penalizing brevity is the former; completeness catching a truncating generator is the latter. They get fixed in different files.
- When several failures share a root cause, generalize the fix rather than patching per-case.
- After any fix, check the previous fix didn't overshoot.
- Re-run 2–3 times before trusting a new number.
- Verify a prompt change actually took effect — a stale serving process will mask it and you'll misread the result.

## Scope

G-Eval is the mechanism for any metric that needs judgment rather than counting. It's used by application-quality metrics (correctness, completeness, style) and by custom safety metrics (scope adherence, system-prompt leakage) alike.

| Need | Skill |
|---|---|
| Count-based RAG metrics — recall, precision, relevancy, faithfulness | `eval:rag` |
| Applying custom metrics to toxicity, leakage, scope drift | `eval:safety` |
| Tracking these metrics across changes without false alarms | `eval:ops` |
| Whether to use an LLM judge at all vs code or humans | `eval:foundations` |

## Provenance

Distilled from the CampusX "Master LLM Evaluations" lecture series (video 15). Code reconstructed against DeepEval's documented API — verify signatures against your installed version.
