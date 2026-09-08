# LLM Eval Skills

Six Claude Code skills for evaluating LLM applications — from deciding what to measure, through picking a model, to running a gated regression suite in CI.

Split by **what they apply to**, not by the order they're taught. Only one of them is RAG-specific; the rest work for agents, chatbots, and classifiers too.

| Skill | Scope | Applies to |
|---|---|---|
| `eval:foundations` | Eval strategy — what to measure, programmatic vs human vs LLM-judge, reference-based vs reference-free, offline vs online, golden datasets, production monitoring and drift | any LLM app |
| `eval:benchmark` | Model selection — benchmark catalogue with saturation/contamination status, leaderboard biases, cost and latency modeling, custom bake-offs | any |
| `eval:rag` | **Retrieval only** — retriever and generator metrics, the RAG Triad, chunking/embedding/reranker levers, diagnostic tree | RAG |
| `eval:geval` | Custom judgment metrics — G-Eval mechanism, `evaluation_params` semantics, worked correctness / completeness / style | any |
| `eval:safety` | Toxicity, PII and content leakage, scope drift, prompt injection, red teaming, guardrail layers | any |
| `eval:ops` | Latency, cost, reliability, regression testing with noise thresholds, CI deploy gating | any |

## Install

```bash
claude plugin marketplace add harshrathod0585/llm-eval-skills
claude plugin install eval@eval
```

**New to this? Start with `/eval:foundations`** — it carries the end-to-end walkthrough and routes you into the others at the right stage.

Invoke with `/eval:rag`, `/eval:geval`, etc. — or just describe the problem and the right skill triggers.

## Why the split

The lectures this came from teach safety, ops, and G-Eval inside a RAG project, because that's the running example. But none of them are retrieval concepts: an agent needs prompt-injection testing more than a RAG bot does, and every LLM app has a cost and latency budget. Filing them under RAG would hide them from everyone not building RAG.

What's genuinely RAG-only is the part that requires a retriever — contextual precision/recall/relevancy, faithfulness, the Triad, and the chunking levers that move them.

## Where the depth is

- `eval:foundations` → `references/end-to-end.md` — the whole build, stage 0 to 8, with project layout
- `eval:foundations` → `references/golden-datasets.md` — dataset shape per metric, and the chunk-ID trap
- `eval:rag` → `references/metrics.md` — every metric: which test-case fields it needs, how it's computed, what a low score implies, and the ranked-precision worked example
- `eval:geval` — why naive "score this 1–10" judging swings between runs, and the two things G-Eval does about it
- `eval:ops` — 2×stddev noise thresholds, and why you must register each metric's direction before comparing anything

## Source

Method distilled from [CampusX's "Master LLM Evaluations" playlist](https://youtube.com/playlist?list=PLEneLIDJFpcA) (18 lectures, free). The skills are original writing — a restructuring of the method into agent-followable form — not a transcript. No lecture content is redistributed here.

Code examples were reconstructed against DeepEval's documented API rather than transcribed from lecture audio, so verify signatures against your installed version.

## License

MIT
