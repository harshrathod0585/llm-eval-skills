# Building an eval suite end to end

The complete thread from an empty repo to a CI-gated regression suite. Each stage produces something runnable before the next begins — the same discipline as writing a function and testing it before writing the next one, rather than building the whole app and testing at the end.

Read this when you need the whole shape. The per-stage detail lives in the sibling skills, named at each step.

## Project layout

```
project/
├── data/                       raw source documents
├── src/
│   ├── retriever.py            load → chunk → embed → store → retrieve
│   ├── reranker.py             reorders retrieved chunks
│   ├── generator.py            (question, context) → answer
│   └── rag_pipeline.py         glue: retriever → generator
├── goldens/
│   ├── retriever_goldens.json      question + ideal_answer
│   ├── faithfulness_goldens.json   question + golden_context
│   ├── correctness_goldens.json    question + ideal_answer
│   ├── toxicity_goldens.json       adversarial / benign / mixed
│   ├── leakage_goldens.json        + expected_action
│   └── scope_goldens.json          + expected_action
├── evals/
│   ├── eval_retriever.py       stage 1a
│   ├── eval_generator.py       stage 1b
│   ├── eval_rag_pipeline.py    stage 2
│   ├── eval_application.py     stage 3 quality
│   ├── eval_safety.py          stage 3 safety
│   ├── eval_ops.py             stage 4 operational
│   └── harness.py              standardizes run()/output across eval files
├── metric_registry.py          direction + noise threshold per metric
├── run_suite.py                runs everything, emits one JSON
├── compare.py                  baseline vs candidate → improved/flat/regressed
├── promote.py                  → approve | review | block
└── baselines/
    └── baseline.json
```

Two structural notes that pay off later. Give every eval file a `run()` function rather than top-level procedural code under `if __name__ == "__main__"` — otherwise `run_suite.py` can't drive them. And keep the four quality evals as separate files while merging the three safety and three ops evals into one file each; the quality evals are edited constantly during tuning, the others aren't.

## Stage 0 — understand the system you're evaluating

Before any code. Skipping this produces an eval suite that measures whatever was easy to measure.

**If the app already exists, read it first.** Trace the entry point to the response and let that path define the components — don't assume the decomposition. `references/discovery.md` covers the inspection pass, how to classify the app type from its dependencies, and what to inventory before proposing anything. The layout above is RAG-shaped because that's the worked example; an agent or a classifier decomposes differently, and only stages 2–3 change.

Then answer:

1. What is the task, exactly?
2. What are the success criteria, stated as metrics or rubrics?
3. Which risk categories are actually in scope — quality always, safety and ops selectively? Write a short safety policy naming the failure modes that matter for *this* app. A no-tools internal bot and a public agent have genuinely different attack surfaces.
4. What are the latency and cost budgets, as numbers?

→ `eval:foundations` for the decision framework, `eval:benchmark` if the model isn't chosen yet.

## Stage 1 — golden datasets

Build these before the evals that consume them. Detail in `references/golden-datasets.md`.

The minimum to start: `question + ideal_answer` (15–50 rows) and `question + golden_context` (15–50 rows). Safety sets come later, at stage 5.

## Stage 2 — component evals

**2a. Build the retriever**, smoke-test it with one query, then evaluate it alone.

```python
# evals/eval_retriever.py
cases = [
    LLMTestCase(
        input=row["question"],
        expected_output=row["ideal_answer"],
        retrieval_context=[d.page_content for d in retriever.invoke(row["question"])],
        actual_output="",
    )
    for row in load_goldens("retriever_goldens.json")
]
evaluate(cases, [ContextualRecallMetric(...), ContextualPrecisionMetric(...)])
```

Record the baseline, then tune one variable at a time — chunk size, then reranker, then embedding model, then `k` — re-running after each. Delete and rebuild the vector store whenever chunking changes, or you're measuring the old index.

**2b. Build the generator**, then evaluate it **fed golden context, not the retriever**. This isolation is the whole point of stage 2b; wiring in the real retriever here makes every failure ambiguous.

```python
# evals/eval_generator.py
cases = [
    LLMTestCase(
        input=row["question"],
        retrieval_context=row["golden_context"],
        actual_output=generate(row["question"], row["golden_context"]),
    )
    for row in load_goldens("faithfulness_goldens.json")
]
evaluate(cases, [FaithfulnessMetric(...), AnswerRelevancyMetric(...)])
```

Expect faithfulness high (~90%) before any work and answer relevancy lower. Tune the system prompt against the failure reasons, 3–4 rounds.

→ `eval:rag`

## Stage 3 — pipeline eval

Wire retriever and generator into `rag_pipeline.py`. The eval is structurally identical to 2b with **one change**: `retrieval_context` now comes from the live retriever inside the pipeline, not the golden set.

```python
answer, retrieved = pipeline.run(row["question"])
LLMTestCase(input=row["question"], actual_output=answer, retrieval_context=retrieved)
```

Add `ContextualRelevancyMetric` — the third leg of the Triad, and the one that catches intra-chunk noise the component evals can't see.

If faithfulness and answer relevancy hold up here versus stage 2b, your prompt tuning generalized. If they drop, it was overfit to golden-context phrasing.

→ `eval:rag`

## Stage 4 — application quality

Now the product-level questions: is the answer *right*, is it *complete*, does it sound like us? These need judgment rather than claim-counting, which means custom G-Eval metrics.

All three run in one `evaluate()` call over the same test cases. Track them separately — do not blend them into a composite.

→ `eval:geval`

## Stage 5 — safety

Build one golden set per failure mode, each containing adversarial, benign, **and** mixed-intent cases. Use built-in metrics where one fits (toxicity, PII), custom G-Eval metrics where the built-in is too coarse (scope adherence, system-prompt leakage).

Watch the polarity: toxicity is lower-is-better, PII leakage is higher-is-better.

→ `eval:safety`

## Stage 6 — operational

No LLM judge, no golden data — plain instrumentation. Latency with percentiles and TTFT, cost per query extrapolated to monthly, reliability with categorized errors. Compare against the budgets you set at stage 0.

→ `eval:ops`

## Stage 7 — tie it together

**Metric registry** — every metric gets a direction and a noise threshold:

```python
METRICS = {
    "contextual_recall":  {"direction": "higher", "noise": 0.04},
    "faithfulness":       {"direction": "higher", "noise": 0.03},
    "toxicity":           {"direction": "lower",  "noise": 0.05},
    "pii_leakage":        {"direction": "higher", "noise": 0.08},
    "latency_p95_ms":     {"direction": "lower",  "noise": 300},
    "cost_per_query":     {"direction": "lower",  "noise": 0.002},
}
```

Derive each noise value empirically: run the full suite 5–10 times with **nothing changed**, take the standard deviation per metric, set the threshold to 2×. Anything smaller than that after a real change is noise.

**The loop:**

```
run_suite.py                    → baseline.json      (unchanged system)
   ↓  make one change
run_suite.py                    → candidate.json
compare.py baseline candidate   → per-metric improved / flat / regressed
promote.py                      → approve | review | block
```

Moved into GitHub Actions, that *is* CI for an LLM app. `promote.py`'s verdict is the deploy gate.

→ `eval:ops`

## Stage 8 — online

Ship, then keep evaluating. Log every turn non-blocking with PII masked; dashboard the captured signals directly; sample the computed ones (stratified, oversampling thumbs-down and escalated conversations) and run reference-free metrics over them. Compare against a baseline, since correctness is unmeasurable without an answer key. Feed every real failure back into the golden sets.

→ `eval:foundations`

## Build order summary

| Stage | Produces | Skill |
|---|---|---|
| 0 | codebase understood, requirements + safety policy | `eval:foundations`, `eval:benchmark` |
| 1 | golden datasets | `eval:foundations` |
| 2 | retriever and generator evals | `eval:rag` |
| 3 | pipeline eval (Triad) | `eval:rag` |
| 4 | correctness, completeness, style | `eval:geval` |
| 5 | toxicity, leakage, scope | `eval:safety` |
| 6 | latency, cost, reliability | `eval:ops` |
| 7 | registry, suite, compare, promote, CI | `eval:ops` |
| 8 | logging, sampling, drift, feedback loop | `eval:foundations` |

Stages 0–3 are the minimum that beats vibe testing. Stages 4–6 are what you add as the app gets real users. Stage 7 is what stops you shipping regressions, and it's worth building as soon as more than one person touches the system.
