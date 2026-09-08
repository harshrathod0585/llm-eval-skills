---
name: rag
description: Evaluate the retrieval half of a RAG application — contextual precision, contextual recall, contextual relevancy, faithfulness, answer relevancy, and the RAG Triad — plus the tuning levers (chunk size and overlap, embedding model, reranker, k) and the diagnostic logic that maps a low score to the component responsible. Use whenever someone is testing, tuning, or debugging a RAG or retrieval-augmented system, mentions DeepEval, RAGAS, the RAG Triad, faithfulness, groundedness, hallucination testing, or golden datasets for retrieval — and also when they describe the symptom instead ("my chatbot makes things up", "how do I know if my retriever is good", "my answers got worse after I changed chunking", "it retrieves the wrong documents"). Also use when deciding whether a bad answer came from the retriever or the generator.
created_at: 2026-09-08T12:27:56Z
updated_at: 2026-09-08T13:10:00Z
---

# Evaluating RAG Retrieval and Generation

This skill covers the part of an eval suite that only exists because you have a retriever. Safety, operational, and custom-quality metrics apply to any LLM app and live in sibling skills — see **Scope** at the bottom.

RAG fails in layers, and one end-to-end score can't tell you which layer broke. The discipline is **localizing failure**: measure components in isolation, then the wired pipeline. Each stage answers a question the other structurally cannot.

## Component and pipeline stages

These are the RAG-specific rows of the component and pipeline stages planned in `eval:foundations`. Build in this order; diagnose in reverse — the pipeline stage tells you *something is wrong*, the component stage tells you *where*.

| Stage | Under test | Metrics | Evidence needed |
|---|---|---|---|
| **Component** — retriever | retriever alone | contextual relevancy | questions only |
| | | contextual precision, contextual recall | question + reviewed ideal answer |
| **Component** — generator | generator alone, fed *controlled* context | faithfulness, answer relevancy | question + reviewed context |
| **Pipeline** | retriever + generator connected | RAG Triad | questions only |

Start with the reference-free checks — contextual relevancy on the retriever, faithfulness and answer relevancy on the generator. Add contextual precision and recall when a coverage requirement or an observed missing-evidence failure justifies the labeling cost, not by default.

**The isolation rule that makes the component stage work:** when evaluating the generator, feed it controlled context, *not* live retriever output. Otherwise a bad score is ambiguous — you can't tell whether the generator misused adequate context or the retriever handed it garbage.

**What changes at the pipeline stage:** nothing but the source of `retrieval_context`. Faithfulness and answer relevancy use identical definitions at both levels — controlled context in isolation, live retriever output when connected. A drop between them is evidence to inspect retrieval, context assembly, and configuration. It is not by itself proof that the generator prompt was overfit.

**The triad does not establish correctness.** All three metrics can pass on an answer that is grounded, relevant, and factually wrong, because the retrieved context was wrong. Correctness and completeness are application-stage concerns needing reviewed expected answers — see `eval:geval`.

## Which metric needs a reference

The most common source of confusion, so be precise:

- **Needs an ideal answer:** contextual precision, contextual recall
- **Needs golden context:** faithfulness (the context *is* the reference)
- **Needs nothing but questions:** contextual relevancy, answer relevancy

Faithfulness being reference-free trips people up constantly. It asks "is every claim traceable to the context I was given" — no ground-truth answer involved. And note what follows: **faithfulness is not correctness.** If the retrieved context is itself wrong, a perfectly faithful generator faithfully produces a wrong answer and scores well. Correctness needs a golden answer and a judgment-based metric — that's `eval:geval`.

Read `references/metrics.md` for the field-by-field breakdown of every metric, how each is computed, and what a low score implies.

## Building the golden dataset

**Never key a golden set to chunk IDs.** It's the intuitive design and it's a trap: change chunk size or overlap — your most common tuning lever — and every chunk ID shifts, voiding the whole dataset. You'd re-annotate against hundreds of chunks after every experiment.

Key it to *content* instead: `question + ideal_answer` for retriever metrics, `question + golden_context` for faithfulness. Information doesn't move when chunk boundaries do, so the set survives re-chunking and you build it once.

Sizing: 15 rows teaches you the loop but produces visible run-to-run noise; 50–500 is the working range. Mix difficulty deliberately — an all-hard set misrepresents production as badly as an all-easy one.

Construction, best first: hand-authored by someone who knows the corpus > LLM-drafted one row at a time with review of each > bulk synthetic generation > mined from production logs (unavailable at cold start, best long-term source). Generate one row at a time with an LLM; bulk generation shifts quality control to a review pass nobody does carefully.

General golden-dataset practice lives in `eval:foundations`.

## Running an eval

```python
from deepeval import evaluate
from deepeval.test_case import LLMTestCase
from deepeval.metrics import ContextualRecallMetric, ContextualPrecisionMetric

cases = [
    LLMTestCase(
        input=row["question"],
        expected_output=row["ideal_answer"],
        retrieval_context=retriever.invoke(row["question"]),
        actual_output="",           # not under test at this stage
    )
    for row in goldens
]

metrics = [
    ContextualRecallMetric(threshold=0.7, model="gpt-4o-mini", include_reason=True),
    ContextualPrecisionMetric(threshold=0.7, model="gpt-4o-mini", include_reason=True),
]
evaluate(test_cases=cases, metrics=metrics)
```

Always set `include_reason=True`. The aggregate score gives you a number; the per-failure reason text tells you what to fix, and every worthwhile fix comes from reading those rather than guessing.

Set `temperature=0` on the system under test. Some judge stochasticity remains regardless — expected, not a bug.

## Reading results honestly

**The judge is biased, and that's usable.** An LLM judge applies its bias consistently across runs and config variants — like a bad cricket pitch, bad for both teams equally. *Relative* comparisons between two configs stay valid even when the absolute number isn't ground truth. Don't over-invest in making the absolute score "true"; do keep judge and setup identical between runs you intend to compare.

**Small deltas on small golden sets are noise.** A few points between identical re-runs on a 15-row set is judge stochasticity. Establish the noise floor empirically before reacting — `eval:ops` covers how.

**Check the whole metric set, never one number.** Metrics trade off constantly. High recall and precision do not imply high contextual relevancy. Chasing one metric while blind to the others is how you ship a regression.

## Diagnostic tree

| Symptom | Cause | Fix, by leverage |
|---|---|---|
| Low contextual recall | retriever missing needed chunks | larger chunks → better embedding model → reranker → raise `k` (blunt, costs precision) |
| Low contextual precision | noise retrieved, or good chunks ranked low | **reranker** (biggest lever) → smaller chunks → tune overlap |
| Low contextual relevancy *despite* good recall + precision | chunk *purity*, not targeting — chunks qualify as "correct" on one useful sentence while carrying mostly filler | reduce chunk size, then re-measure everything (trades against recall) |
| Low faithfulness | generator hallucinating past its context | tighten system prompt (anti-hallucination, anti-overstatement) → stronger model |
| High faithfulness, low answer relevancy | grounded but off-topic | same two levers — the generator only ever has two |
| Pipeline scores ≪ component scores | generator overfit to golden context phrasing | re-tune against live retriever output |
| All retriever metrics good, answer still wrong | it's the generator | prompt or model |

The generator has exactly two levers (prompt, model). The retriever has four (chunking, embeddings, reranker, `k`). Budget effort accordingly.

Also worth ruling out before re-chunking: confirm the retriever config — chunk size, `k`, index — is genuinely identical between your component eval and your pipeline eval. A config mismatch produces the same signature as a real regression.

## Scope

This skill is retrieval-specific. Everything below applies to any LLM app and lives elsewhere:

| Need | Skill |
|---|---|
| Discovery, the staged plan, eval methods, dataset preparation | `eval:foundations` |
| Correctness, completeness, or any custom judgment metric | `eval:geval` |
| Applicable safety and authorization scenarios | `eval:safety` |
| Latency, cost, reliability; regression gating when requested | `eval:ops` |
| Choosing which model to use in the first place | `eval:benchmark` |

## Provenance

Method distilled from the CampusX "Master LLM Evaluations" lecture series (videos 12–14). Code reconstructed against DeepEval's documented API — verify signatures against your installed version.
