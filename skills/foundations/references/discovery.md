# Discovery: derive the plan from the codebase

## Trace before classifying

Read repository instructions, manifests and relevant entry points. If a project knowledge graph exists, use its query/navigation facilities first, then verify relevant source files. Otherwise use `rg --files` and targeted `rg` searches in the project's languages; do not assume Python or install discovery tooling.

Follow a representative request through preprocessing, routing, model calls, retrieval, tools, memory, output validation, and the application response. Trace alternate routes that materially change success or risk, including failure handling and authorization. Record actual file/function references. Inspect the complete relevant paths, not every unrelated file in the repository.

For each candidate boundary, ask: can it fail independently, would an isolated check diagnose something useful, and is it already tested? Dependencies alone do not establish behavior: a vector database package may be unused, and an LLM application may combine several flows.

Inventory existing evals, tests, fixtures, reviewed examples, tracing, runtime interfaces, and framework versions. Reuse them. A non-Python application can keep native deterministic tests or expose its existing local API/CLI to a small Python DeepEval runner if qualitative judging is needed; do not rewrite the application.

## Choose the minimum useful method

These are choices, not a checklist. Select from observed behavior and product requirements.

| Observed component | Starting evaluation | Evidence / method | Add only when needed |
|---|---|---|---|
| Retriever | Context relevance | Question + actual retrieved passages; DeepEval contextual relevancy | Labeled recall for missing evidence; precision/ranking checks for noisy ranking |
| Grounded generator | Faithfulness and answer relevance | Controlled context + generated answer; DeepEval | Separate correctness and completeness at application level |
| Classifier/router | Expected label/action | Exact comparisons; accuracy with per-class counts | Precision/recall/F1 when class imbalance or error costs require them |
| Extractor | Expected field values | Native schema tests + normalized field comparisons | Field precision/recall for optional or repeated values |
| Summarizer | Source faithfulness and required-point coverage | Source + reviewed key points; rubric or direct checks | Style only for an explicit product requirement |
| Tool selector | Allowed tool and correct arguments | Expected acceptable calls + argument values; assertions | Order only when the task requires it; do not reject valid alternate paths |
| Task agent | Expected final state | Sandboxed task fixture; direct outcome assertions | Trace-based judging only for outcomes that cannot be checked directly |
| SQL/code generator | Correct execution result | Read-only/sandbox execution against controlled fixtures | SQL row ordering only when required; JSON schema alone does not prove semantic correctness |
| Conversation/memory | Required information used across turns | A few reviewed conversations | Separate memory eval only if isolation reveals a distinct failure |

Unknown application types: derive a check from the output contract and expected user outcome. Do not force a closest template or fabricate a framework metric.

## Establish product requirements

Extract intended behavior, model/configuration, tools, permissions, and relevant limits from code and docs. Prompts describe intent, not authoritative truth. Ask only what remains decision-relevant: what is correct, what information is required, what actions are forbidden, and what latency/cost limits apply? Bundle concise questions; users need not choose metric names.

Use the available evidence to draft the plan while identifying unknowns. Do not invent business rules or an SLO. Unconfirmed numeric thresholds remain proposed; missing budget numbers need not block a measured baseline. For side-effecting tools, plan mocks or sandbox state rather than real writes.

## Present the plan

Use the three tables in [end-to-end.md](end-to-end.md), populated with actual components, connected flows, user journeys, code references, necessary methods, evidence, and proposed files. Application evaluation must explicitly cover correctness, completeness, safety, and operations, even if a dimension is already covered or is inapplicable with a reason.

Explain what each metric catches in one phrase. Identify shared versus separate datasets, review status, held-out needs, applicable safety cases, and cost/configuration assumptions. State online evaluation is unavailable. Do not copy all rows from an example into a different application.

Get agreement before creating data or evals. Then implement the first stage; show results before progressing. The user may approve all offline stages in advance, so do not repeatedly ask for the same permission.

## DeepEval verification

Inspect the project's installed version and use matching official docs/source before writing code. Verify constructors, required fields, supported model configuration, score polarity, and error behavior. Pin/record the dependency and judge version used. Never infer semantics solely from a class name or copy a threshold from an example.

- [Metrics and method selection](https://deepeval.com/docs/metrics-introduction)
- [RAG triad](https://deepeval.com/guides/guides-rag-triad)
- [G-Eval and evaluation parameters](https://deepeval.com/docs/metrics-llm-evals)
- [Tool correctness](https://deepeval.com/docs/metrics-tool-correctness)

Load only the specialized references selected by the plan. If documentation or credentials are unavailable, validate what can run locally and state exactly what remains unverified.
