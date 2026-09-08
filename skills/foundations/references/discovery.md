# Discovery: deriving the eval plan from the codebase

Do this before proposing any metric. An eval plan built on assumptions about the architecture measures the wrong components, and the user has to correct you after you've written code.

The goal of this pass is to answer four questions: **what kind of app is this, what are its components, what already exists, and what constraints apply.** Everything downstream follows from those.

## Step 1 — find the entry point and trace the flow

Start from how a request enters and follow it to the response. The shape of that path *is* the component decomposition you'll evaluate.

```bash
# entry points
rg -l "FastAPI|Flask|streamlit|gradio|@app\.(route|post|get)|def main" --type py

# the LLM call itself — where generation happens
rg -n "ChatOpenAI|ChatAnthropic|AsyncOpenAI|OpenAI\(|anthropic\.|litellm|invoke\(|\.chat\.completions" --type py
```

Read the files those hit. You are looking for the sequence: what happens to the user's input before it reaches the model, and what happens to the model's output before it reaches the user. Each transformation is a component that can fail independently and therefore deserves its own eval.

## Step 2 — classify the application type

The presence of specific dependencies is strong evidence. Check both the code and the manifest.

```bash
rg -n "chroma|pinecone|qdrant|weaviate|faiss|pgvector|milvus" --type py      # vector store → RAG
rg -n "as_retriever|similarity_search|RecursiveCharacterTextSplitter|embed"  # retrieval → RAG
rg -n "tools=|@tool|bind_tools|function_call|tool_choice|ToolNode"           # tool use → agent
rg -n "langgraph|StateGraph|crewai|autogen|AgentExecutor"                    # orchestration → agent
rg -n "ConversationBufferMemory|chat_history|messages\[|session_id|thread_id" # multi-turn
cat requirements.txt pyproject.toml package.json 2>/dev/null
```

Map what you find to the component metrics:

| Evidence | App type | Component metrics | Skill |
|---|---|---|---|
| Vector store + retriever | RAG | contextual precision/recall/relevancy, faithfulness, answer relevancy | `eval:rag` |
| Tool bindings, agent loop | Agent | tool selection, parameter correctness, task completion, trajectory, error recovery | see below |
| Fixed label set, enum output | Classifier | accuracy, precision/recall/F1 — **programmatic, no LLM judge** | `eval:foundations` |
| Long input → short output | Summarizer | faithfulness against source doc, coverage | `eval:geval` |
| Message history, session state | Multi-turn | knowledge retention, role adherence, conversational relevancy | — |
| SQL/JSON output, schema | Structured | execution-based or schema comparison, never string match | `eval:benchmark` |

Most real apps are a combination. A support agent with a knowledge base is RAG *and* agent, and needs both metric sets.

## Step 3 — inventory what already exists

Don't rebuild what's there, and don't assume nothing is.

```bash
ls -d tests/ evals/ eval/ benchmarks/ 2>/dev/null
rg -l "deepeval|ragas|langsmith|langfuse|promptfoo|braintrust" .
fd -e json -e csv -e jsonl . | rg -i "golden|eval|test.*set|ground.?truth"
rg -n "LLMTestCase|GEval|evaluate\(|assert_test" --type py
```

Also check for signals already being captured that you can build on: existing logging, a tracing integration, thumbs-up/down storage, latency instrumentation. Production logs are the best golden-dataset source there is, and if they already exist, stage 1 gets much cheaper.

## Step 4 — extract the constraints

These set thresholds and decide which risk categories are in scope. Some are readable from the code; the rest you ask about.

**From the code:**
- Model and provider in use → cost per token, and whether swaps are easy
- `temperature` settings → whether eval runs will be reproducible
- `k`, chunk size, chunk overlap → the tuning levers available
- System prompts → what behavior is already being asked for, and what the eval should therefore verify
- Tool access → whether unsafe-action safety modes are in scope at all
- Auth/tiering logic → whether content-leakage evals matter

**Ask the user:**
- Who uses this — internal, public, tiered? Sets the safety scope.
- Query volume → cost budget math
- Latency expectation → the SLO
- What does a bad answer cost? A medical or legal answer and a casual suggestion carry different correctness bars.

## Step 5 — propose before building

Present the plan and get agreement before writing eval code. It's a short message and it prevents building the wrong suite:

- The app type and its components, as you traced them
- Which metrics per component, and why those
- What golden data is needed, in what shape, and how many rows
- What's in and out of scope for safety, given the actual attack surface
- Which stages to build now vs later

Users routinely correct the component decomposition at this point. That correction is cheap here and expensive after implementation.

## Step 6 — implement with DeepEval

DeepEval is the default choice: broader scope than RAGAS (agents, multi-turn, non-LLM apps), PyTest-styled so it's familiar, and actively converging on being the standard. RAGAS is equally capable for pure RAG — if a project already uses it, don't migrate for its own sake.

```bash
pip install deepeval          # or: uv add deepeval
export OPENAI_API_KEY=...     # judge model credentials
```

Structure the eval files as described in `end-to-end.md`: one per component level, each exposing a `run()` function, invoked as a module (`python3 -m evals.eval_retriever`) so imports from `src/` resolve. Add `__init__.py` to both directories — a missing one is the most common first error.

Build in this order, running each before writing the next:

1. One eval file for the most important component, with 15 golden rows. Confirm it runs end to end.
2. Expand golden data to 50+.
3. Add remaining components.
4. Add pipeline level.
5. Add application quality, then safety, then ops.
6. Wire the registry, suite, compare, and promote.

Getting one metric running against real data beats a complete suite that has never executed. The first run always surfaces something — a path issue, a field mismatch, an empty retrieval context — and finding that on one metric is much faster than on twelve.

## Agent component metrics

The source lectures are RAG-focused and don't build these, so treat this section as general practice rather than lecture-derived.

| Metric | Question | Approach |
|---|---|---|
| Tool selection correctness | right tool for the request? | compare called tool against expected; DeepEval `ToolCorrectnessMetric` |
| Parameter correctness | right arguments? | schema validation plus value comparison — largely programmatic |
| Task completion | did it finish the job? | DeepEval `TaskCompletionMetric`, or a custom G-Eval over the final state |
| Trajectory quality | sane path, or 14 steps for a 3-step job? | step count against a reference, plus G-Eval on the trace |
| Error recovery | survives a failed tool call? | inject failures deliberately and assert recovery |

Agents raise the stakes on `eval:safety` rather than lowering them: tool access adds hijacking and unsafe-action modes that a chat-only interface doesn't have, and operational evals need step caps and timeouts to bound runaway loops.
