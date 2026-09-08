# The staged evaluation journey

Discovery and plan review come first; dataset preparation happens before each consuming stage. The three evaluation stages are **component → pipeline → application**. Online evaluation is currently unavailable. Regression/CI is optional follow-up, not a prerequisite for completing the offline setup.

## Plan format

Begin with the detected application, actual request flow, intended user outcome, and supporting code references. Use these table columns, replacing generic labels with concrete findings:

### Stage 1 — Component evaluation

| Component | Evaluations | Dataset / evidence | Proposed file |
|---|---|---|---|
| Independently useful boundary found in code | Minimum check and method | Required labels or controlled inputs | Actual proposed path |

End with the outcome: what failures this stage can localize.

### Stage 2 — Pipeline evaluation

| Connected flow | Evaluations | Dataset / evidence | Proposed file |
|---|---|---|---|
| Actual connected execution path | Combined behavior check and method | Runtime outputs + expected behavior | Actual proposed path |

End with the outcome: what integration failures this stage can reveal.

### Stage 3 — Application evaluation

| User journey / requirement | Evaluations | Dataset / evidence | Proposed file |
|---|---|---|---|
| Intended task | Correctness and completeness | Reviewed answers, required points, or expected final state | Quality/journey eval file |
| Applicable policy or access boundary | Safety | Expected allowed/forbidden behavior, benign controls | Safety eval file if distinct fixtures justify it |
| Same representative requests | Operations: latency, cost, reliability | Timing, usage, errors, timeouts | Instrument existing run; separate file only if useful |

End with the outcome: whether users can complete tasks correctly, completely, safely, and within measured operational constraints. State any dimension that is inapplicable and why. Style, tone, citation accuracy, and other checks are conditional on actual requirements.

Finish with dataset readiness/review needs, shared data, methods and why, proposed thresholds versus confirmed requirements, estimated run cost (or unknown and how to measure it), and the next action. State **online evaluation: currently unavailable**. Ask for agreement on the concrete plan, not a generic permission to start.

## Worked example: a RAG chatbot

Illustration only: suppose source inspection establishes question → retrieval → generation → chat response, with follow-up support. Replace these names/paths and add source references in a real plan. Do not invent conversation history, permissions, or tools if absent.

### Stage 1 — Component evaluation

| Component | Evaluations | Dataset / evidence | Proposed file |
|---|---|---|---|
| Retriever | Context relevance (DeepEval) | Questions + actual retrieved passages | `evals/components/test_retrieval.py` |
| Generator | Faithfulness, answer relevance (DeepEval) | Questions + reviewed controlled context | `evals/components/test_generation.py` |

> Outcome: distinguish poor retrieval from poor use of adequate context. Add recall only for a coverage requirement or observed missing evidence, with suitable labels.

### Stage 2 — Pipeline evaluation

| Connected flow | Evaluations | Dataset / evidence | Proposed file |
|---|---|---|---|
| Question → retrieval → generation | RAG triad: context relevance, faithfulness, answer relevance | Shared questions; capture the actual context passed to generation and its answer | `evals/pipeline/test_rag.py` |

> Outcome: evaluate the connected flow using the same metric definitions. A drop is evidence to inspect retrieval, context assembly, and configuration; it does not by itself prove overfitting.

### Stage 3 — Application evaluation

| User journey / requirement | Evaluations | Dataset / evidence | Proposed file |
|---|---|---|---|
| Ask a question and follow up | Correctness against reviewed facts; completeness of required points (separate G-Eval rubrics) | Shared answer cases plus short conversations where needed; expected abstention for unavailable facts | `evals/application/test_chatbot.py` |
| Malicious instructions in a user question or retrieved document | Scoped safety: follow the policy while still answering legitimate requests | Adversarial and benign controls; mixed-intent cases where meaningful; expected behavior | `evals/application/test_safety.py` |
| Complete the same requests | Latency, token/cost usage, errors and timeouts (instrumentation) | Reuse application runs; provider usage and confirmed budgets | Reuse `evals/application/test_chatbot.py` |

> Outcome: verify answers are correct and complete, assess relevant safety behavior, and measure operating cost and reliability. Add access-isolation scenarios if private or tiered documents actually exist. A small offline run does not establish production tail latency or rare-event safety.

Start with shared answer cases. Separate safety/conversation datasets only when their evidence differs; tags suffice otherwise. The RAG triad is not a substitute for correctness or completeness.

## Application methods across project types

Correctness compares to authoritative expected facts, labels, results, or state. Completeness checks required fields, requested sub-tasks, or reviewed key points; do not reward length or demand unsupported details. For an unanswerable request, appropriate clarification or abstention can satisfy the expected behavior.

Use code for exact outcomes and G-Eval for qualitative criteria with reviewed evidence. Keep distinct scores where they expose distinct failures. On a single-label classifier, one label comparison may cover both correctness and completeness; record this rather than adding an LLM judge. Pipeline and application stages may reuse that result if they add no new behavior.

Safety evaluates actual trust boundaries and unacceptable outcomes. An authorization assertion is stronger evidence of access enforcement than asking a judge whether a response sounds safe. Use sandbox tools and benign controls; do not silently skip applicable security checks to save effort.

Operations reuse existing runs: measure end-to-end duration, available usage/cost, errors and timeouts. TTFT applies only to streaming. Include failed requests in reliability counts; unavailable usage means cost is unknown, not zero. Separate application inference cost from evaluator cost. Load `eval:ops` for a requested SLO study or regression gate; do not create a load-testing system for a starter suite.

## Implement and resume

1. Prepare and validate the current stage's data using [golden-datasets.md](golden-datasets.md). Review candidate labels before treating them as ground truth.
2. Reuse the project's test runner and dependency manager. Check selected DeepEval APIs against the installed version; keep Python in an isolated eval environment if the app is in another language. No DeepEval dependency for deterministic-only checks.
3. Implement the smallest selected eval and verify it runs against the real component. Use controlled fixtures for isolation; never pass golden answers into the application as a shortcut. Mocks validate wiring but do not establish live model quality.
4. Check a known-good and known-bad example against each materially different evaluator. Calibrate qualitative judges against human judgments before relying on thresholds; do not tune only to make the application pass.
5. Run the current stage, save outputs and results, explain failures and coverage limits, then proceed to the next agreed stage. Keep failures visible; unresolved data/credential issues remain blocked, not passed. Do not silently change application behavior when the request is evaluation setup.
6. Record case IDs, versions of data/corpus/model/prompt/judge, settings, counts, scores/directions, reasons, errors, timing, and available costs. Reuse existing reporting. Check held-out cases after tuning; repeat ambiguous results as budget permits.
7. Keep commands and stage status in existing eval notes or `evals/README.md`, including what is completed, failed, pending, or awaiting review. "Continue" resumes from these artifacts. A new component changes only the affected plan rows and coverage.

Use dedicated eval files for meaningful boundaries, not a file per metric. A normal test-runner command selecting a stage is enough. Add shared helpers only when needed by multiple files. A saved baseline and per-metric comparison suffice until automated CI is requested; never overwrite the baseline simply because a new run finished.
