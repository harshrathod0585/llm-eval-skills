# RAG metric reference

Every metric here maps to a specific, nameable failure mode. If you can't say which failure a metric catches, you're chasing a number.

## Test case fields

DeepEval's `LLMTestCase` carries these; each metric reads only the ones it needs.

| Field | Meaning |
|---|---|
| `input` | the user's question |
| `actual_output` | what your system produced |
| `expected_output` | ideal answer you authored (ground truth) |
| `retrieval_context` | what the retriever **actually** returned at runtime |
| `context` | what **should** have been retrieved (ideal chunks, hand-labeled) |

`retrieval_context` = actual, `context` = ideal. Mixing these up silently changes what you're measuring.

---

## Retriever metrics

A retriever has exactly two failure modes: it misses relevant chunks (recall), or it drags in noise alongside them (precision). These trade off — raising `k` always helps recall and hurts precision. Beating both at once takes design work (chunking, embeddings, reranking), not a dial.

### Contextual Recall

**Question it answers:** of everything a fully correct answer needs, how much did the retriever actually surface?

**Fields:** `input`, `expected_output`, `retrieval_context`. Reference-based.

**Computation:** the judge decomposes the *ideal answer* into atomic claims. For each claim, it checks whether that content appears anywhere across the retrieved chunks. Score = claims found / total claims, averaged over questions.

**Low score means:** the retriever isn't surfacing information the answer depends on.

**Fixes:** larger chunks (each carries a more complete concept) → better embedding model → reranker → raise `k` (works, but costs precision).

### Contextual Precision

**Question it answers:** of everything retrieved, how much was useful — *and were the useful ones ranked first?*

**Fields:** `input`, `expected_output`, `retrieval_context` (order matters). Reference-based.

**Computation:** two steps. First, per-chunk relevance — for each retrieved chunk, the judge is asked whether it helps produce the expected answer. Second, rank-aware aggregation — walk the ranked list top-to-bottom computing precision@i at each cutoff, then average. This is average precision as used in information retrieval.

**Why rank-awareness matters:** two result sets with identical raw precision can differ enormously in quality.

```
Case A: [correct, correct, noise, noise, noise]
        precision@1..5 = 1.0, 1.0, 0.67, 0.50, 0.40  → avg 0.71

Case B: [noise, noise, noise, correct, correct]
        precision@1..5 = 0.0, 0.0, 0.00, 0.25, 0.40  → avg 0.13
```

Both are 2 correct out of 5. Case A is the far better retriever to hand a generator, which attends most strongly to what comes first. Naive precision can't see this difference; contextual precision can.

**Low score means:** too much noise, and/or good chunks buried.

**Fixes:** a reranker is by far the biggest lever — it directly reorders to push useful chunks up. Then smaller chunks, then overlap tuning.

### Contextual Relevancy

**Question it answers:** within the chunks retrieved, how much of the actual *content* is relevant?

**Fields:** `input`, `retrieval_context`. **Reference-free** — questions only, no ideal answer.

**Computation:** the judge decomposes *each retrieved chunk* into claims, then judges each claim's relevance to the question. Score = relevant claims / total claims across all chunks.

**Low score means:** chunks are internally noisy. This is invisible to precision, which only asks whether a chunk is useful *at all* — a chunk with one useful sentence out of six passes the precision bar while being 83% filler.

**The characteristic diagnostic case:** recall 99%, precision 89%, contextual relevancy 42%. Nothing is wrong with retrieval *targeting* — the right chunks are being found and ranked well. The problem is chunk *purity*. Fix by reducing chunk size, then re-measure the full set, because smaller chunks can cost you recall.

---

## Generator metrics

A generator has two failure modes: it invents things not in the context (unfaithful), or it stays grounded but doesn't answer the question (irrelevant).

### Faithfulness

**Question it answers:** is every claim in the answer supported by the context it was given?

**Fields:** `input`, `actual_output`, `retrieval_context`. **Reference-free** — no ideal answer involved.

**Computation:** decompose the *generated answer* into claims; check each against the context. Score = supported claims / total claims.

**Faithfulness is not correctness.** If the context is wrong, a faithful generator faithfully produces a wrong answer and scores perfectly. All four combinations are real states: faithful+correct (ideal), faithful+incorrect (your source material is wrong), unfaithful+correct (the model answered from training knowledge, ignoring your context), unfaithful+incorrect (hallucination). Only stage-3 correctness separates these.

**Expect this to score high without effort.** Modern models follow "answer only from this context" instructions well. A 90%+ baseline before any tuning is normal — don't read it as proof the system is good.

**Low score means:** hallucination. This is the high-liability failure. Air Canada was held legally responsible when its chatbot invented a refund policy.

**Fixes:** system prompt (explicit anti-hallucination and anti-overstatement rules) → stronger model. That's the entire lever set.

### Answer Relevancy

**Question it answers:** does the answer actually address what was asked?

**Fields:** `input`, `actual_output`. **Reference-free.**

**Computation:** decompose the generated answer into claims; for each, ask whether it helps answer the question. No reference comparison at all. Score = relevant claims / total.

**Low score means:** the generator is faithfully reciting adjacent context instead of answering. Common when context contains near-miss material.

**Fixes:** same two levers. The failure-driven loop that works: run the eval → read `include_reason` on the failures → feed the failure reasons plus your current prompt to an LLM for a rewrite → re-run → repeat 3–4 rounds. This routinely moves answer relevancy from ~73% to ~92%.

**Watch for overfitting.** Tuning a prompt against a small fixed golden set can tailor it to those specific rows. Validate by re-running at the pipeline level against live retriever context — if the gains hold there, they generalize.

---

## The RAG Triad

Three metrics, one per edge of the (question, context, answer) triangle:

| Edge | Metric |
|---|---|
| question ↔ context | contextual relevancy |
| context ↔ answer | faithfulness |
| question ↔ answer | answer relevancy |

All three are reference-free, which is precisely why the same code runs at stage 2 and again at stage 4 on production traffic.

**What it catches:** whether the three pairwise relationships are individually healthy.

**What it does not catch:** correctness against the real world, completeness, style, citation accuracy. And it doesn't replace recall/precision — the triad's contextual relevancy is a complementary signal, not a substitute. A full retriever assessment needs recall, precision, *and* relevancy.

---

## Metric summary

| Metric | Needs ideal answer | Needs golden context | Reference-free | Catches |
|---|---|---|---|---|
| Contextual recall | ✅ | — | ❌ | missing context |
| Contextual precision | ✅ | — | ❌ | noise / bad ranking |
| Contextual relevancy | — | — | ✅ | intra-chunk noise |
| Faithfulness | — | ✅ (as reference) | ✅ | hallucination |
| Answer relevancy | — | — | ✅ | off-topic answers |

---

## Judge configuration

- Use the **same model** for claim decomposition and claim judgment within one metric. The two sub-calls share no context, so splitting them across models adds complexity for no benefit.
- Spend on a strong judge model. Eval runs happen over a small golden set, not production traffic, so the cost is bounded and the reliability is worth it.
- `include_reason=True` always.
- `evaluate()` runs test cases in parallel, so a large golden set is less painful than it looks.
