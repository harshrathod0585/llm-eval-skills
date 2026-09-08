---
name: foundations
description: Work out what to evaluate in an LLM application and how — starting by reading the actual codebase to identify its components and application type (RAG, agent, classifier, summarizer, multi-turn), then choosing metrics to match. Covers programmatic vs human vs LLM-as-judge, reference-based vs reference-free, offline vs online, golden dataset construction, and production monitoring with sampling and drift detection. Carries the end-to-end walkthrough — project layout and the full build order from empty repo through golden datasets, component and pipeline evals, application quality, safety, operational evals, regression gating and monitoring. Start here for "how do I evaluate my app", "what should I measure", "how do I test this before shipping", "what does an eval pipeline actually look like", or "set up evals for this codebase". Also use when someone is relying on vibe testing, or when an eval suite looks healthy while users complain.
created_at: 2026-09-08T12:27:56Z
updated_at: 2026-09-08T12:27:56Z
---

# LLM Evaluation Foundations

## Before anything else: read the codebase

When someone asks for help evaluating *their* application, do not propose metrics from the description alone. Trace the actual code first — find the entry point, follow a request to the response, and let that path tell you what the components are. A plan built on an assumed architecture measures the wrong things, and the user has to correct you after you've written code.

`references/discovery.md` is that pass: what to grep for, how to classify the app type from its dependencies, how to inventory existing evals and golden data, which constraints to read from code versus ask about, and how to implement with DeepEval once the plan is agreed.

**Propose the plan and get agreement before writing eval code.** Users routinely correct the component decomposition, and that correction is cheap before implementation and expensive after.

This matters most because RAG is only one shape. Agents, classifiers, summarizers, and multi-turn chatbots each decompose differently, and only the component layer changes — everything from application quality downward is identical across all of them.

## Start here for the whole picture

If the question is "how do I actually build this end to end", read `references/end-to-end.md` — it carries the project layout and the full build order across all eight stages, naming which skill to load at each one. This file explains the *decisions* behind that sequence; that file is the sequence itself.

## What an eval actually is

An eval is not a metric. It's the entire testing setup: **what** you test, **against what criteria**, **on what dataset**, **when** you run it, and **with what tool**. A number falls out at the end, but the number isn't the eval.

Three properties are non-negotiable:

- **Systematic** — built on a curated dataset covering edge cases, not questions you thought of just now.
- **Repeatable** — the same dataset and method must re-run against a changed system to give a comparable result. If you can't re-run it, it isn't an eval.
- **Criteria-driven** — meaningful only relative to explicitly predefined success criteria.

The anti-pattern is **vibe testing**: trying an app with a handful of self-picked prompts and judging by feel. It's subjective and non-repeatable, so it can't compare versions or catch systemic failures. Fine for a toy; not for anything real. The Air Canada chatbot (invented a refund policy, the airline was held liable in court), the Chevrolet dealer bot (jailbroken into a binding $1 car offer), the fabricated legal citations that got a lawyer fined — all shipped on vibes.

Software testing habits transfer only partially. Classic software is deterministic and correctness is the whole benchmark. LLM apps are probabilistic — same input, different output — and need a multidimensional check: factuality, completeness, tone, groundedness, latency, cost. Which dimensions matter is application-specific.

## Model evals vs application evals

- **Model evals** judge the raw LLM's capabilities, via public benchmarks. Mostly the frontier labs' job; you need enough literacy to read them when choosing a model. See the `benchmark` skill.
- **Application evals** judge the full system built around the model — prompt, retrieval, tools, orchestration, guardrails, parsers, memory. **This is the actual work**, and where nearly all your effort should go.

A phone's chip benchmark doesn't tell you whether the phone is good. Camera, OS, battery, and sound each need their own evaluation regardless of how the chip scored.

(These two terms are a useful teaching split, not standard industry vocabulary — the industry says "LLM evals" for both and infers from context. Don't cite them as formal terminology without that caveat.)

## One application, many eval pipelines

Near-universally true: a single monolithic eval for a whole application is the wrong shape. Two independent reasons force this.

**Failure points exist at three levels:**

1. **Component** — retriever, reranker, query rewriter, embedding model, vector DB, output parser, tool selector, memory, guardrails. Each gets its own pipeline.
2. **Workflow** — the interaction between components, which fails even when every component passes.
3. **Application** — whole-system concerns like latency, cost per query, time-to-first-token, that only appear once assembled.

**Risk categories cut across all three:**

1. **Quality** — correct, relevant, complete, instruction-following.
2. **Safety** — no toxic or harmful content, no leakage, resistant to jailbreak and injection.
3. **Operations** — fast, cheap, reliable under load.

A correctness eval catches no safety problem and no latency problem, so each risk category typically needs its own pipeline even on the same component.

**The case that proves workflow evals are necessary.** A retriever passes its eval — the correct document is in the top-5. A generator passes its eval — it correctly prioritizes whichever documents it's told rank highest. Wired together they produce a wrong answer, because the correct document landed at position 5 while an irrelevant one at position 1 mentioned a different course's duration. Neither component was broken relative to its own spec. The fix was a reranker, which neither component eval would ever have suggested. Component correctness is necessary and not sufficient.

Pick quality dimensions by application type rather than applying a generic checklist: RAG apps add context relevance and groundedness; agents add tool selection correctness, parameter correctness, task completion, error recovery; multi-turn chatbots add context retention and clarification behavior.

## The eval workflow

1. **Define task and target** — exactly what system or component, doing what.
2. **Define success criteria** — the metric or rubric that operationalizes "working."
3. **Build a golden dataset** — real examples with verified expected outputs.
4. **Choose an evaluation method** — programmatic, human, or LLM-as-judge.
5. **Run the system** over the whole dataset, capturing outputs.
6. **Score** against the golden labels.
7. **Analyze** — diagnose *why* failures happened, not just that they did.
8. **Improve** — targeted fix (prompt, model, architecture).
9. **Iterate** — re-run on the *same* dataset and compare.
10. **Deploy.**
11. **Monitor** in production.
12. **Feed failures back** into the golden dataset, and loop forever.

Steps 11–12 are what keep the suite from rotting. An eval set frozen at creation time silently stops describing reality.

## Choosing an evaluation method

Exactly three options exist for who performs the judgment.

| Method | Use when | Cost |
|---|---|---|
| **Programmatic** | output is a discrete label or otherwise machine-checkable | cheapest |
| **Human** | judgment is too subjective for code and reliability matters more than scale | doesn't scale — salaries |
| **LLM-as-judge** | too ambiguous for code, too large for humans, but you can write a clear rubric | middle ground |

Reach for the cheapest that works. Classification into billing/technical/general is exact-match arithmetic — don't put an LLM on it. Comparing two paragraphs for semantic correctness genuinely can't be done by code, and that's when you move up the ladder.

**Human evaluation isn't one thing.** Five distinct modes: direct grading against a rubric; red teaming (adversarially probing, findings routed to the dev team); A/B testing on live traffic; golden dataset creation; and human-in-the-loop escalation for genuine gray areas.

Use **multiple graders on the same items to test your rubric**, not just to average scores. Widespread disagreement between graders means the rubric is ambiguous — fix the rubric, not the graders.

**Instructing a judge LLM:** give it role framing, the question, the max score, the exact rubric, the candidate answer, per-dimension grading instructions, and require written justification rather than a bare number. Include explicit anti-gaming language — "do not reward verbosity, keyword stuffing, or confident assertions lacking substance; reward structure, relevant examples, and balanced argumentation."

**Measure judge reliability** with mean absolute error against human scores on the same golden set. Drive MAE toward zero by using a stronger judge, refining the prompt, or sharpening the rubric. For a stability-focused approach to judge design, see the G-Eval material in the `rag` skill.

## Reference-based vs reference-free

The whole test: **does your golden dataset contain the correct answer for each item?** If yes, reference-based. If no, reference-free — you judge against a rubric or a standard instead.

This matters operationally because reference-free metrics are the only ones that can run in production, where no answer key exists.

## Golden datasets

- **Size:** 50–500 for a typical application task.
- **Source:** real production data wherever possible — 100 real user conversations labeled beats 100 invented questions, because it carries the actual input distribution and its edge cases.
- **Coverage:** normal, edge, difficult, and adversarial cases. An all-easy dataset is worse than none: it reports a healthy pass rate while production burns.
- **Categories:** track sub-scores per category (pricing, refunds, curriculum) so a regression in one is visible instead of averaged away.
- **Living artifact:** every production failure gets added.

Read `references/golden-datasets.md` for the per-metric column shapes, construction methods ranked by quality, sizing guidance, and why a chunk-ID-keyed set voids itself the first time you retune.

## Offline vs online

**Offline** runs before deployment against a fixed golden dataset. It answers *is this application correct?*

**Online** runs on live production traffic. There's no answer key, so it usually can't measure correctness at all. It answers *is this application behaving normally right now?*

These are complementary and always run together — online is not a replacement for offline.

**Why offline alone is structurally insufficient:**

1. **Out-of-distribution inputs** — real users write code-mixed language, ambiguous phrasing, angry messages, injection attempts you never anticipated.
2. **Failures visible only at scale** — latency under concurrent load; bias that only becomes statistically detectable across thousands of conversations and is invisible in a 50-item set.
3. **Drift** — the world changes (prices, policies, curriculum) while your golden set stays frozen. Offline scores stay green while users get wrong answers. This is the dangerous one, because nothing looks broken.

### Running online eval

**Logging comes first.** It must be non-blocking (never add user-facing latency), durable and queryable, support late-arriving signals linked by conversation ID (a user emailing support the next day), and apply PII masking before storage so nobody can mine sensitive data out of the log tool later.

Capture per turn: conversation ID, turn ID, user ID, session ID, timestamp, question, retrieved context, output, latency, token counts, cost, error codes, plus behavioral signals (thumbs up/down, escalation, repeated question).

**Two pipelines by signal type:**

- **Captured signals** (already present — latency, cost, tokens, thumbs up/down): Log → Dashboard → Alert. No evaluator needed.
- **Computed signals** (need an evaluator — faithfulness, toxicity, hallucination rate): Log → Sample → Evaluate → Aggregate over a window → Dashboard → Alert.

**Sample; never judge every conversation.** You already pay to serve each conversation; judging all of them more than doubles cost. Prefer **stratified sampling** over random: bucket conversations (got thumbs-down, ended abruptly, escalated, question repeated, money mentioned) and oversample the problematic buckets. Conversations that got a thumbs-up can be deprioritized. This raises hit rate per sample substantially.

**Proxies for correctness when no answer key exists:**
- Compare live score *distributions* against a historical baseline — a sudden shift signals something changed even when you can't say which items are wrong.
- Thumbs-down rate as a correctness proxy.
- Some metrics need no reference at all — faithfulness just compares answer to retrieved context, so it works identically online.

**A dashboard number means nothing without a baseline.** Faithfulness at 87 against an 85 baseline is fine; the same 87 against a 95 baseline is an incident. Aggregate over windows — never judge health from one conversation.

## Regression is the recurring trap

Changing anything means re-running everything. A prompt edit asking the bot to be "kind and polite" caused it to soften a precise price into a vague approximation — an unrelated, unnoticed factual regression from a tone change.

An improvement from 92% to 99% on your target metric is only good news if nothing else moved. Always check the rest.

**CI gating:** automate the offline suite as a release gate — above threshold deploys, below blocks and notifies. Pick the maturity level that matches your team: manual comparison, then experiment tracking with config logged alongside metrics, then full CI gating. The concept — compare against a baseline, decide pass/fail — applies at every level.

## Reference files

- `references/discovery.md` — inspect a codebase, classify the app type, derive the plan, implement with DeepEval
- `references/end-to-end.md` — project layout and the full eight-stage build order
- `references/golden-datasets.md` — dataset shapes per metric, construction, sizing, staleness

## Where to go next

| Need | Skill |
|---|---|
| Choosing which model to use | `eval:benchmark` |
| Retrieval quality — retriever, generator, RAG Triad | `eval:rag` |
| Custom judgment metrics — correctness, completeness, style | `eval:geval` |
| Toxicity, leakage, scope drift, injection | `eval:safety` |
| Latency, cost, reliability, regression gating | `eval:ops` |


## Provenance

Distilled from the CampusX "Master LLM Evaluations" lecture series (videos 01–06).
