# Golden datasets: evidence matched to the check

Support multiple datasets by component, scenario, or risk category. Do not require a separate file per metric: one case can support several checks. Split files when evidence, review ownership, or execution environments differ; use category tags for filtering otherwise.

## Evidence requirements

The table lists stored evidence plus runtime outputs; exact framework field requirements must be verified against the installed version. Reference-free means no ideal answer, not no evidence.

| Evaluation | Stored case evidence | Captured during the run |
|---|---|---|
| Context relevance | Question | Retrieved passages |
| DeepEval contextual recall / precision | Question + reviewed ideal answer | Retrieved passages; required test-case fields |
| Deterministic recall@k / precision@k | Query + labeled relevant document IDs + corpus version and retrieval unit | Ranked IDs at k |
| Faithfulness | Question + controlled context for isolated generator tests | Answer + exact context supplied to generator; pipeline uses live context |
| Answer relevance | Question | Answer |
| Correctness | Input + reviewed answer, label, expected result, or state | Actual answer/result/state |
| Completeness | Input + required facts, fields, or sub-task outcomes | Actual answer/result/state |
| Summarization | Source + reviewed essential points | Summary |
| Tool behavior | Input + fixture state + acceptable tools/arguments/outcomes | Calls, results, final state |
| Conversation | Scenario/turns + expected behavior; valid alternatives | Full relevant transcript and task outcome |
| Safety | Policy, adversarial/benign/mixed category, expected behavior; permission fixture if applicable | Response, attempted actions, resulting state |
| Operations | Representative inputs; budget/configuration if known | Timing, usage, errors/timeouts; no golden answer required |

DeepEval's claim-based contextual recall is not document recall@k. Name the method precisely. Stable document IDs are valid retrieval labels when linked to a corpus version. Chunk IDs tied to one chunking setup must be regenerated/remapped when that setup changes. Store source IDs/spans or reviewed content when possible; content labels also need review after source changes.

## Minimal case contract

Use the project's existing JSONL/JSON/CSV format. Each case needs a stable ID, input, applicable evidence fields, and enough metadata to identify category, source/provenance, review status, and split. Dataset-level metadata can hold shared corpus version, policy, and schema; do not duplicate it on every row.

A RAG case may be as small as:

```json
{"id":"returns-01","input":"How long do I have to return an item?","golden_context":["Returns are accepted within 30 days of delivery."],"expected_output":"Within 30 days of delivery.","required_points":["30 days","from delivery"],"category":"returns","source":"returns-policy-v2","review_status":"candidate","split":"development"}
```

This is synthetic example data, not an approved policy. At runtime, store `actual_output` and the actual `retrieval_context` separately. Never substitute golden context into a pipeline result or send expected answers to the system being evaluated.

## Prepare and validate

1. Reuse reviewed domain examples and existing fixtures first. Authorized historical logs may supply realistic inputs; redact sensitive data before sending to an external judge and respect the project's data rules.
2. Draft missing examples from identified sources. Synthetic labels remain **candidate** until a qualified reviewer or an authoritative deterministic oracle validates them. Ask for review of the actual examples, not blanket approval that generated data is "golden".
3. Validate parseability, unique IDs, required evidence for selected metrics, allowed labels, source availability, and contradictory/duplicate cases. Check expected outputs mechanically where possible. Executable SQL is not necessarily correct SQL; use controlled read-only fixtures to validate intended results.
4. Include representative normal cases and the edge/failure cases the product needs. Keep adversarial stress-test results separate from typical-traffic estimates. Report per-category counts, not only a blended average.
5. Separate development/tuning from held-out validation before tuning. Keep paraphrases, turns from one conversation, and closely related source examples in the same split to avoid leakage. If data is too small, label the run exploratory rather than claiming an independent validation.
6. Route cases only to compatible metrics. Missing correctness labels block correctness scoring, not reference-free checks that have enough evidence. Report excluded and pending-review counts; do not fill missing labels with the application's own answer.

Start with a small reviewed sample sufficient to validate wiring and rubric behavior; size further runs by coverage, risk, uncertainty, and budget. No fixed row count establishes accuracy. A tiny set is a smoke test, not proof of rare-failure safety. When oversampling important cases, do not present the unweighted score as the production success rate.

## Maintain without adding infrastructure

Version datasets alongside eval code. Record corpus/policy changes and re-review affected labels. Keep held-out results separate from tuning feedback; cases exposed during tuning become development cases. Save reported failures as candidates for review, deduplicate them, and add the ones that extend coverage.

Review judge agreement on examples spanning good, bad, partial, and ambiguous outputs. Distinguish defective labels from application errors and rubric errors. Keep the judge, rubric, and settings recorded for comparisons; a passing score does not certify that the evaluator is accurate.
