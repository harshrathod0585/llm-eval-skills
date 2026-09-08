---
name: ops
description: Measure and control an LLM application's latency, cost, and reliability, and catch silent quality regressions with a baseline-comparison suite wired into CI. Covers percentile latency and TTFT instrumentation, per-component breakdown, token cost modeling, error categorization, concrete cost/latency optimization tactics, metric registries with direction and noise thresholds, and baseline/compare/promote deploy gating. Use whenever someone asks about LLM app latency, cost per query, token spend, reliability, SLOs, performance regressions, eval gates in CI/CD, or how to know whether a change made things worse — including symptom phrasings like "it got slower after I changed the model", "this is costing too much", or "how do I stop shipping regressions". Applies to any LLM app, not just RAG.
created_at: 2026-09-08T12:27:56Z
updated_at: 2026-09-08T12:27:56Z
---

# Operational Evals and Regression Testing

## Operational evals

These answer "can it run reliably, quickly, and cheaply?" rather than "is the output good?" Critically, they need **no LLM judge and no golden dataset** — they're plain instrumentation, which makes them cheap enough to run on every change.

**Why run them offline at all**, given a laptop's numbers won't hold in production: the absolute value isn't the point, the *differential* is. Catching "this change took latency from 2.3s to 4.1s" before deploying is the entire value. This requires the experimental setup stay identical between runs, or the differential means nothing.

### Latency

Instrument start-to-finish around the pipeline, but report more than a mean.

- **Report the distribution** — P50, P95, P99, min, max. A mean hides the worst user experiences, which are the ones that matter.
- **Break down by component.** Retrieval vs generation. The generator typically dominates (~4× the retriever) because it's an external LLM call — knowing that tells you where to optimize instead of guessing.
- **Measure TTFT separately.** Streaming UX depends on time-to-first-token, not total generation time.
- **Discard warm-up runs** (first 1–2). Cold starts — model loading, DB connection init, network handshake — inflate them artificially.
- **Log token count and context size alongside.** Latency scales with both; without them the numbers aren't interpretable.
- **Repeat each question 5+ times and average.** External APIs are noisy; one call is not a point estimate.
- **State the concurrency level.** A single-user laptop number and a production number under load aren't comparable, and presenting them as if they were is misleading.
- **Track failures and timeouts alongside latency.** A P95 that "improved" while the timeout rate quadrupled is worse, not better — the timed-out requests silently left the latency sample.

Set explicit SLOs at both system and component level (e.g. end-to-end P95 ≤ 3000ms, TTFT P95 ≤ 1200ms). Without a threshold, "is this good enough" has no answer.

### Cost

Driven almost entirely by LLM tokens. Output tokens typically price ~4× input, which makes answer length the higher-leverage lever.

Cost is **stable across environments** — provider rates don't fluctuate — so offline measurement is reliable in a way latency isn't. One caveat: repeated identical questions in a test script trigger the provider's own prompt caching, making offline cost measurements optimistic relative to production where queries vary. Say so when reporting.

Compute per-query cost split into input and output, then extrapolate to daily and monthly at your expected volume. Segment by query type rather than reporting one blended average.

### Reliability

Success rate, error rate, timeout rate, retry rate.

**Categorize errors by cause** — LLM API failure, retriever failure, reranker failure, timeout, rate limit, parser error, internal exception — via try/except around each stage. One generic error rate doesn't tell you which stage to fix.

Reliability under concurrency is a separate measurement and needs hundreds to thousands of requests, not the 20–50 that suffice for a smoke check. Rare failures don't show up in small offline runs.

### The quality/ops tension

The central tradeoff: adding a reranker, raising `k`, and swapping to a larger model improves quality metrics *and* degrades every operational one simultaneously. Never optimize quality in isolation — measure both together or you'll ship an unusable improvement.

## Optimization tactics

**Latency** (target the generator, it's the bottleneck):
- Faster/smaller model variant
- Model router — classify query complexity, send simple queries to a small model and hard ones to a large model
- Cap answer length; latency scales with output tokens
- Reduce context size — lower `k`, or contextual compression (compression itself costs time, so measure)
- Cache at every layer: embedding, retrieval, reranking, system prompt
- Co-locate vector DB, reranker, and LLM in the same region as users

**Cost** (fewer levers, since it's nearly all tokens):
- Switch to a cheaper model — by far the biggest single lever; everything else moves ±5%
- Cap answer length (output tokens cost ~4×)
- Trim the system prompt
- Reduce context size
- Prompt caching, which helps most when a large static block repeats verbatim. Note it helps RAG chatbots *less* than you'd hope, since retrieved context changes every query.

## Regression testing

Regression = the system returning to a worse state. RAG regresses silently because optimizing one metric routinely degrades others nobody was watching — you add a reranker and raise `k` to fix recall, and precision, contextual relevancy, and latency all quietly get worse.

**Trigger:** any deliberate change. Chunk size or overlap, `k`, adding or removing a reranker, model swap, prompt change, any config change.

### The metric registry

Every metric needs two properties registered before any delta means anything:

1. **Direction** — higher-is-better (faithfulness, recall, precision, PII leakage) or lower-is-better (latency, cost, toxicity). Getting this wrong inverts your entire report, and note that the two safety metrics point in *opposite* directions.
2. **Noise threshold** — the metric's inherent run-to-run variance with the system unchanged. Most quality and safety metrics use an LLM judge, so re-running with zero code changes still produces different numbers.

**Deriving the noise threshold:** run the full suite 5–10 times with no changes, collect the values per metric, compute the standard deviation, set threshold = **2 × stddev**. Any delta smaller than that after a real change is noise, not regression.

Smaller golden sets produce more variance — a 10–15 row set can swing a metric 20 points between identical runs purely from size. Bigger sets stabilize everything.

### The workflow

```
run_suite.py    → runs quality, then safety, then ops evals; logs all metrics to JSON
                  first run (unchanged system) → baseline.json
                  after the change            → candidate.json

compare.py      → baseline.json + candidate.json + metric_registry
                → per-metric: improved / flat (within noise) / regressed

promote.py      → reads compare output, applies tiered importance
                → approve | review | block
```

Structure the suite so each eval file exposes a `run()` function rather than top-level procedural code, with a harness standardizing the input/output contract across them so one outer script can drive them uniformly.

**Gating rules:** a safety metric dropping ~20% is an automatic block regardless of what else improved. Everything else is tiered by importance, with `review` routing to a human because trade-offs ("contextual relevancy dropped but five other metrics improved") are inherently application-specific.

**A worked example of why this matters:** reducing chunk size 1500→500 to fix a contextual relevancy problem. Most metrics came out flat or improved. Two regressed — contextual relevancy went *down* (45→38, the opposite of the intent) and PII leakage dropped ~20%. Correct outcome: block, on the safety regression alone.

### CI

Moved into GitHub Actions, this flow *is* CI for a RAG system. `promote.py`'s decision is the deploy gate: block means don't deploy, review means human sign-off, approve means promote and update the baseline.

Budget for it. A 14-metric suite over a small golden set runs a few minutes and costs real money per run, and that scales with golden set size. It's eval infrastructure cost, distinct from inference cost, and worth paying — but plan it rather than discovering it.

## Do not build a composite score

Resist collapsing all metrics into one weighted number. Different metrics exist because they explain different failure modes; merging them into a single dimension destroys the diagnostic value. Report the table.

## Scope

Applies to any LLM application. Operational evals need no LLM judge and no golden data, which makes them cheap enough to run on every change — and regression testing is what keeps every *other* eval honest.

| Need | Skill |
|---|---|
| The quality metrics this suite tracks, for RAG | `eval:rag` |
| The custom quality metrics it tracks | `eval:geval` |
| The safety metrics it tracks (mind the inverted polarities) | `eval:safety` |
| Production monitoring, sampling, and drift detection | `eval:foundations` |

## Provenance

Distilled from the CampusX "Master LLM Evaluations" lecture series (videos 17–18).
