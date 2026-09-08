# Golden datasets

The golden dataset is the artifact everything else depends on. A weak one makes every downstream metric decorative — it'll report a healthy pass rate while production burns.

## Shape depends on the metric

There is no single golden dataset. Different metrics need different columns, and building one universal file wastes effort on fields most metrics ignore.

| Metric | Columns needed |
|---|---|
| Contextual recall / precision | `question`, `ideal_answer` |
| Contextual relevancy | `question` |
| Faithfulness | `question`, `golden_context` |
| Answer relevancy | `question` |
| Correctness, completeness | `question`, `ideal_answer` |
| Style | `question` |
| Toxicity | `question`, `category` (adversarial/benign/mixed) |
| Leakage, scope adherence | `question`, `category`, `expected_action` |
| Classification tasks | `input`, `label` |
| Text-to-SQL | `question`, `golden_query`, `order_sensitive` |

Note how many need only questions. Reference-free metrics are cheaper to build for and are the only ones that can also run in production.

## The rule that saves you re-work

**Never key a golden set to chunk IDs, row indices, or anything tied to your current configuration.**

It's the intuitive first design — record which chunk contains the answer, then check whether the retriever returned it. It works exactly until you change chunk size or overlap, which is the most common tuning lever you have. Then every ID shifts and the whole dataset is void. Re-annotating means a human reading hundreds of chunks per question, once per experiment.

Key to *content* instead — an ideal answer, or the golden context as text. The information doesn't move when chunk boundaries do, so the set survives re-chunking and you build it once.

The exception: if documents are cleanly siloed one-topic-per-document and your chunking parameters are frozen for good, ID-based works. That's rarer than it sounds.

## Sizing

| Size | Use |
|---|---|
| 15 | enough to learn the loop; expect visible run-to-run noise |
| 30–50 | working minimum for a real decision |
| 50–500 | the normal range for an application task |

Smaller sets swing more. A 10–15 row set can move a metric 20 points between identical runs purely from size — which is why noise thresholds must be derived empirically rather than assumed.

Bigger isn't free: every row costs LLM-judge calls on every run, and that recurs on every regression run forever. Budget it as eval infrastructure cost.

## Coverage

**Match the real input distribution.** Not every question should be hard. A rough split for 50 rows: 10 easy, 20 medium, 20 hard. An all-hard set is a stress test, not an eval, and it'll make a fine system look broken.

**Cover the shapes that exist in your traffic** — different query types, different phrasings, multi-part questions, questions with no answer in the corpus.

**Tag rows by category** (pricing, refunds, policy, whatever your domains are) and track sub-scores per category. An aggregate hides a total failure in one category behind success in four others.

**Include the cases that will actually break it:** ambiguous phrasing, code-mixed language, questions whose answer isn't in the corpus, and — for safety sets — adversarial, benign, and mixed-intent prompts.

## Construction methods, best first

**1. Hand-authored by someone who knows the corpus.** Highest quality, doesn't scale. The author thinks of a plausible question, finds the source material, reads it, and composes the ideal answer. Hiring someone external is slower than it looks, because they lack the corpus knowledge that makes this fast.

**2. LLM-drafted, one row at a time, each reviewed.** What most people should actually do. Give the model your corpus, ask for one question/answer pair, review it against what you know is true, accept or delete, repeat. The one-at-a-time constraint is deliberate — bulk generation moves the quality burden to a review pass nobody does carefully, and an LLM will happily write confident golden answers about things your corpus never covered.

**3. Synthetic generation in bulk.** DeepEval ships a `Synthesizer`. In practice the output skews toward whatever is textually prominent in the corpus rather than what users actually ask — producing academic, over-formal questions no real user would type. Usable as a starting draft, but the human review overhead often cancels the time saved.

**4. Mined from production logs.** The best long-term source and unavailable at cold start. Once live, harvest real queries — especially ones that failed, got a thumbs-down, or triggered an escalation — and fold them in. This is the loop that stops the dataset going stale.

Most projects use 2 to start and 4 forever after.

## Validate mechanically where you can

Where a golden answer is machine-checkable, check it before trusting it. For text-to-SQL, execute every golden query against the real database and confirm it runs — that catches schema drift and typos in seconds.

Be clear about what this proves: syntactic validity, not correctness. A query that runs and returns the wrong answer passes this gate. Correctness still needs human review.

## Golden datasets are living

Two forces make a frozen dataset wrong over time:

**Drift.** The world changes — prices, policies, curriculum, product names — while the dataset holds the old answers. Offline scores stay green while real users get wrong answers. This is the dangerous failure, because nothing looks broken. Re-verify golden answers whenever the underlying source material changes.

**Coverage gaps.** Production surfaces question types you never imagined. Every one that fails should become a row.

Treat the dataset as versioned code, not a one-time deliverable. Reviewing it on a schedule is cheaper than discovering it went stale via a user complaint.

## Who writes the ideal answers

A domain expert, or an LLM draft that a domain expert verified. Never unverified LLM output — you'd be measuring your system against a hallucination and calling the result ground truth.

For subjective dimensions where no single correct answer exists, you don't need ideal answers at all. Write a rubric instead: a numeric scale with a description per band. That's the reference-free path, and it's the right shape for style, helpfulness, and tone.
