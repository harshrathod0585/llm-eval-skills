# LLM Eval Skills

Six Claude Code skills for evaluating LLM applications. You point them at a codebase and say *"set up evals"*; they read the code, propose a staged plan you approve, then build the smallest useful suite.

Works for RAG, agents, classifiers, extraction, summarization, chatbots, and applications that combine them — not just RAG.

## Install

```bash
claude plugin marketplace add harshrathod0585/llm-eval-skills
claude plugin install eval@eval
```

Then, in any project:

> Set up evals for this project.

`eval:foundations` is the entry point. It loads the others as the plan requires them.

## The journey

```
Discover  →  read the code, trace real request flows, inventory existing tests
Plan      →  three stage tables you approve or adjust, before anything is written
Stage 1   →  component evals   — each part checked in isolation
Stage 2   →  pipeline evals    — the connected flow
Stage 3   →  application evals — correctness, completeness, safety, operations
Online    →  currently unavailable, and stated as such rather than stubbed
```

One stage at a time, with results explained before the next begins. Say *"continue my eval setup"* later and it resumes from where it stopped.

The plan you approve looks like this — populated from your actual code, not a template:

| Component | Evaluations | Dataset / evidence | Proposed file |
|---|---|---|---|
| Retriever | Context relevance | Questions + actual retrieved passages | `evals/components/test_retrieval.py` |
| Generator | Faithfulness, answer relevance | Questions + reviewed controlled context | `evals/components/test_generation.py` |

## The skills

| Skill | Scope | Applies to |
|---|---|---|
| `eval:foundations` | **Entry point** — discovery, the staged plan, dataset preparation, running and reporting | any |
| `eval:rag` | Retriever and generator metrics, the RAG Triad, chunking and reranker levers | RAG |
| `eval:geval` | Custom judgment metrics — G-Eval mechanism, correctness, completeness | any |
| `eval:safety` | Toxicity, leakage, scope drift, injection, authorization | any |
| `eval:ops` | Latency, cost, reliability; regression gating when requested | any |
| `eval:benchmark` | Model selection — benchmarks, leaderboards, cost modeling, bake-offs | any |

Only `eval:rag` is retrieval-specific. Safety and operations are dimensions of the application stage, not RAG concepts — an agent needs injection testing and a cost budget as much as a RAG app does.

## Design principles

**Minimal by default.** One eval per distinct failure mode, not one per function. The starter suite for a RAG chatbot is the triad plus a few correctness cases — no registry, no CI, no bake-off unless the workflow asks for it.

**Evidence, not vibes.** Deterministic checks wherever an outcome is directly verifiable; LLM judging only where judgment is genuinely required. An authorization assertion beats asking a judge whether a response sounds safe.

**Honest about what a score establishes.** Synthetic labels stay *candidates* until reviewed. Held-out cases stay separate from tuning. A small passing run is not proof of production reliability, and the skills say so rather than declaring success.

**Distinctions that matter, kept distinct.** Faithfulness is not correctness — an answer can be perfectly grounded in a source that is wrong. Relevance is not completeness. The RAG Triad establishes neither.

## Where the depth is

- `eval:foundations` → `references/discovery.md` — trace, classify, choose the minimum useful method
- `eval:foundations` → `references/end-to-end.md` — plan format, worked example, implementation loop
- `eval:foundations` → `references/golden-datasets.md` — evidence per check, case contract, validation
- `eval:rag` → `references/metrics.md` — every metric: fields, computation, what a low score implies

## Source

The RAG metrics, G-Eval, safety, and operational material is distilled from [CampusX's "Master LLM Evaluations" playlist](https://youtube.com/playlist?list=PLEneLIDJFpcA) (18 lectures, free). The skills are original writing, not a transcript, and no lecture content is redistributed.

Codebase discovery, non-RAG component selection, the minimal staged workflow, and dataset validation are maintained guidance rather than lecture coverage — they're marked as such in the skills so nothing is misattributed.

Verify DeepEval APIs against your installed version; the skills link official docs and tell the agent to check before writing code.

## License

MIT
