---
name: benchmark
description: Choose an LLM for a specific application by reading benchmarks and leaderboards critically and then running a custom eval on your own data. Covers the eight capability axes, what each major benchmark (MMLU, GPQA, MMLU-Pro, TruthfulQA, SimpleQA, HLE, GSM8K, AGIEval, SWE-bench) actually measures and where it's saturated or contaminated, how to read leaderboards without being misled by configuration gaming and human-preference bias, cost and latency modeling including prompt caching, and the full shortlist-then-bake-off procedure. Use whenever someone asks which model to use, compares models, mentions a benchmark or leaderboard by name, is deciding between providers or between hosted and open-weight models, needs to justify a model choice to a team, or asks whether a headline benchmark claim is trustworthy. Also use when someone is about to pick a model purely from leaderboard rank.
created_at: 2026-09-08T12:27:56Z
updated_at: 2026-09-08T13:10:00Z
---

# Benchmarks, Leaderboards, and Model Selection

## The one rule

**Leaderboards are a filtering tool, not a decision tool.** They narrow a field of hundreds to a shortlist of five. They cannot tell you which model wins on your task, because they never tested your task.

The demonstration that matters: a heavily-hyped model topping general leaderboards scored ~50–55% on a text-to-SQL task and threw outright syntax errors, while a less-discussed competitor hit ~90% on the same golden set. Public rank and task fitness are different quantities.

The corollary runs the other way too. A model that loses every public benchmark can be the right choice: 94% vs 91% accuracy is a 3-point gap, and if the loser costs 30× less and responds faster, it wins on value. Benchmarks can't weigh that trade for you.

## The eight capability axes

Every benchmark maps to at least one. Know the taxonomy before reading any results table.

1. **Knowledge and reasoning** — factual recall plus multi-step logical connection.
2. **Coding and software engineering** — generation, bug-fixing, multi-file refactors, function calling.
3. **Mathematics** — grade school through competition through research level.
4. **Long context** — actually *using* information deep in a large window, not merely accepting the tokens.
5. **Vision and multimodal.**
6. **Agentic and tool use** — browsing, structured tool calls, computer use.
7. **Safety and alignment** — harmful content, jailbreak resistance, truthfulness vs sycophancy.
8. **Instruction following** — doing exactly what was asked, in the requested format and length. Underrated, and directly drives user satisfaction.

## Saturation vs contamination

Two distinct ways a benchmark stops being useful. They need different fixes, and conflating them leads to the wrong conclusion.

**Contamination** — the benchmark's questions and answers leaked into pretraining or alignment data because the benchmark is public. The model scores well from memorization, not capability. This **inflates absolute scores** and worsens the longer a benchmark has been public. Fix: private held-out test sets, dynamic or rotating datasets.

**Saturation** — the benchmark is static and models genuinely improved until everyone clusters at 90%+ and it can no longer discriminate. A saturated benchmark isn't necessarily contaminated; models may really have gotten better. This **destroys ranking power** while leaving absolute scores meaningful. Fix: retire it, replace with something harder.

In short: saturation says *stop using this to rank models*; contamination says *don't trust this score as true capability*. Contamination accelerates saturation, which is why they co-occur and get confused.

The lifecycle repeats: a hard benchmark appears → models score poorly → generations improve → scores rise → frontier models cluster near the ceiling → declared saturated → replaced. MMLU → MMLU-Pro/GPQA/AGIEval → HLE. TruthfulQA → SimpleQA/MASK.

## Benchmark catalogue

| Benchmark | Measures | Format | Status and caveats |
|---|---|---|---|
| **GSM8K** | grade-school math | ~8.5k problems, 8-shot, CoT, pass@1 | Saturated (90%+). Retired as a discriminator. |
| **MMLU** | knowledge breadth | 14k MCQ, 4 options, 57 subjects, 5-shot | Saturated (86–92% cluster). ~6.5% of labels are wrong or disputed, capping a "perfect" score near 92–93%. Very prompt-format sensitive. Heavy contamination — public since 2020. |
| **TruthfulQA** | truthfulness against common misconceptions | 817 adversarial questions, 38 categories; Generation / MC1 / MC2 scoring | Saturated ~2024. Famous early finding — larger models were *less* truthful — faded as RLHF improved. Contaminated via alignment data specifically. LLM-judged, so old scores aren't comparable to new. |
| **AGIEval** | knowledge via real human exams | ~8k questions, SAT/LSAT/Gaokao/civil service, bilingual EN+ZH | Saturated ~2025 (passed the 91% human baseline). Exam-style recall only. |
| **GPQA** | knowledge depth, science | PhD-level physics/chem/bio, expert-validated; Diamond subset = 198 questions | Near saturation (80%+ on Diamond). **Only 198 questions** — low statistical confidence. "Beats PhDs" claims rest on inconsistent baselines (69.7% vs 81.3% depending on source). |
| **MMLU-Pro** | knowledge breadth, hardened | 10 options not 4, reasoning-heavy, ~12k questions, 14 categories | Approaching saturation. No published human baseline. Gives reasoning models a ~20-point structural advantage — don't use it to compare a reasoning model against a fast non-reasoning one. |
| **SimpleQA** | factuality **and calibration** | 4,326 short free-text questions, LLM-graded Correct / Incorrect / Not-Attempted | Active. Reports Correct, Correct-given-Attempted, and an F-score blending both. Short-form only — says nothing about RAG hallucination. Adversarially selected against GPT-4, so possibly unfair to other families. Answers go stale. |
| **HLE** | knowledge + reasoning + math, breadth and depth | 2,500 expert-written questions, 100+ subjects, ~80% short-answer, ~10% multimodal, **private held-out set** | Active, hardest current knowledge benchmark. Scores accuracy *and* calibration. English-only, not agentic. |
| **SWE-bench** | coding on real GitHub issues | requires tool use | Referenced as largely saturated. |
| **MTEB** | embedding models | leaderboard | The one to consult when choosing a RAG embedding model. |
| **Berkeley Function-Calling** | tool/function calling | composite | Domain-specific agentic leaderboard. |

**Calibration vs accuracy** is worth separating: accuracy is whether the model got it right, calibration is whether it knows when it doesn't know. Most classic benchmarks measure only accuracy. SimpleQA and HLE measure both — which matters enormously for anything user-facing, since a confidently wrong answer is worse than an abstention.

## Reading a benchmark score critically

**Check the run configuration before comparing anything.** Zero-shot vs few-shot, CoT allowed or not, temperature, pass@1 vs pass@k vs majority@k, tool access. All of these materially move scores, and mismatched configs make "A beat B" meaningless. Pass@1 and pass@5 are not the same measurement.

**Watch for configuration gaming** — a lab reporting its own model with a code interpreter attached on a math benchmark while reporting the rival under default settings.

**Watch for aggregation gaming** — a 57-subject average hides a catastrophic weakness in the one subject you care about. Always open the per-category breakdown before deploying into a narrow domain.

**Scoring method changes the number.** Generation vs log-probability scoring produce a 2–3 point gap on MMLU for the same model.

**Small gaps are not real.** 84.3 vs 84.1 is a tie. Treat close scores as tied, especially on small-N benchmarks with no confidence interval.

**Check dataset size.** A 198-question benchmark doesn't support the precision a 14,000-question one does.

**LLM-judged scores drift across time.** As the judge model improves, TruthfulQA/SimpleQA/HLE scores become non-comparable year over year.

**Distrust lab self-reported numbers.** They're run in favorable conditions and represent a ceiling, not an expectation — the manufacturer's fuel-economy figure. Prefer third-party evaluators who test everyone under matched conditions and also report cost and latency, which labs tend to omit.

**Be skeptical of "beats human experts."** Beating an exam tests exam-style recall, not long-horizon reasoning, tool use, or real task completion.

## Leaderboard types

| Type | Example | Use for |
|---|---|---|
| Benchmark-specific | HLE's own board | narrow; tells you nothing about overall capability |
| **Multi-benchmark composite** | Artificial Analysis, LiveBench | **most useful** — capability plus cost, latency, context |
| Human preference | LM Arena | popular, but see bias below |
| Domain-specific | MTEB, Berkeley Function-Calling | matching your actual task |

**Human-preference leaderboards encode human bias.** People systematically prefer longer, more confident, better-formatted, more entertaining answers — not more correct ones. A more honest model can lose these votes.

**Goodhart's Law applies directly.** Once a leaderboard becomes influential, labs tune to win it — training toward flattering, well-formatted responses rather than genuine capability. Rank decouples from real-world quality.

**Check freshness** (missing new releases, listing discontinued models) and, for composites, **check what's included and how it's weighted**. Undisclosed weighting is a red flag.

## Cost and latency modeling

Do this before touching a leaderboard, because it hard-filters most of the field.

```
monthly_cost = (input_tokens/1e6 * input_rate + output_tokens/1e6 * output_rate)
               * queries_per_day * 30
```

Use the *actual* production prompt — system prompt plus schema plus query — in a token counter. Don't estimate.

**Blended pricing** on aggregator leaderboards uses a ratio, commonly 4:1 or 8:1 input:output: `(r_in * input_rate + r_out * output_rate) / (r_in + r_out)`. Check which ratio a leaderboard uses before trusting its single price number. If your app's own ratio happens to match, you can use it directly.

**Prompt caching** matters when a large static block (system prompt, schema) repeats verbatim. Cache writes cost a premium (~1.25× input rate) once per refresh window; subsequent hits within the window cost roughly a tenth of normal input. Pick the TTL from request density — high-frequency traffic is fine on a short window, sparse traffic wants the longer one. **Caching helps RAG chatbots much less**, since retrieved context changes every query.

**Latency budget** comes from the interaction pattern. Real-time human-facing: 2–3s is fine, 5s reads as broken. But weight latency by output length — an app emitting ~100 tokens finishes acceptably even on a slow model, while long-form output makes tokens/sec dominate.

**Context window** only matters if you have multi-turn or session state. Single-turn disjoint Q&A doesn't differentiate on it.

## The selection procedure

**Stage 1 — requirements, written down before any leaderboard.**

Task, cost ceiling converted to cost-per-query yourself, latency tolerance, context needs, deployment mode (privacy constraints decide hosted vs self-hosted; absent a constraint, prefer a hosted API for reliability), and a correctness bar justified by audience sensitivity. A medical application tolerates almost no error at any price; a casual consumer feature can trade accuracy for cost.

**Stage 2 — shortlist from leaderboards.**

1. Find a task-specific leaderboard. If none is current and trustworthy, use the closest **capability proxy** — a general coding leaderboard stands in for text-to-SQL, since SQL generation is a coding task.
2. Hard-filter by cost ceiling.
3. Rank survivors on a weighted normalized composite: min-max normalize each axis, then weight by what actually matters — e.g. `0.9 * capability + 0.1 * latency` when output is short. Weights are subjective; the discipline is *justifying* them from your UX, not the specific numbers.
4. Strike models unusable for structural reasons (region locks, licensing).
5. Hand-pick 5–10, deliberately spanning: the top scorer even if expensive (to test whether the premium is justified), one from each vendor family, and cheap alternatives scoring close to the top.

**Stage 3 — the bake-off on your own data.**

1. Build a golden dataset of 30–50 items minimum, matching real query distribution — mix difficulty (roughly 10 easy / 20 medium / 20 hard for 50) and mix query shapes. Don't make everything hard.
2. Have a domain expert author or verify every golden answer. Never trust unverified LLM-generated ground truth.
3. Validate mechanically where possible — for text-to-SQL, confirm every golden query actually executes. That's a cheap syntactic gate, not a correctness check.
4. Use a multi-provider gateway rather than N vendor SDKs.
5. **Smoke-test the harness with a throwaway model first.** Debug your plumbing against a model you don't care about, not against candidates whose scores you're trying to measure.
6. Run each candidate across the full golden set, tracking accuracy, cost, and latency together.
7. For statistical confidence, run the set multiple times per model and average — each call is independent and stochastic.

**Comparing structured outputs:** never string-match. Multiple syntactically different SQL queries produce identical correct results. Execute both the generated and golden query and compare *result tables*: check shape first, normalize values (`2.0` == `2`), sort into canonical form before comparing — **unless** the golden query has an `ORDER BY`, in which case order is semantically part of the answer and must not be sorted away. Tag each golden row with an `order_sensitive` flag so the evaluator knows which.

**Watch for qualitative failure modes benchmarks won't show** — outright syntax errors are a categorically worse failure than a wrong-but-valid answer, and generic benchmarks won't surface them.

The final call weighs accuracy, cost, and latency together. It's a judgment call, and reporting it as one — with the reasoning — is the deliverable.

## Running benchmarks yourself

An eval harness handles the scaffolding around a benchmark: batching, retries, rate limits, answer extraction, standardized scoring. The benchmark is the exam paper; the harness is the exam administration.

**LM Evaluation Harness** (EleutherAI) is the standard for running public benchmarks against any model with minimal code. **Inspect** and **HELM** are alternatives. DeepEval offers some benchmark support but requires substantially more custom code for this — it's built for application evals, so use it there instead.

Cap runs while iterating. A full GSM8K run costs real money; a 20-item capped run costs cents and validates your setup.

## Where to go next

| Need | Skill |
|---|---|
| What to measure in an application, and how | `eval:foundations` |
| Retrieval quality — retriever, generator, RAG Triad | `eval:rag` |
| Custom judgment metrics for your bake-off scoring | `eval:geval` |
| Cost and latency measurement once you've shipped | `eval:ops` |


## Provenance

Distilled from the CampusX "Master LLM Evaluations" lecture series (videos 07–11). Benchmark saturation status reflects the lectures' recording date — re-verify current standings before relying on them.
