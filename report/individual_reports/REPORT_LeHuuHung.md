# Individual Report: Lab 3 - Chatbot vs ReAct Agent

- **Student Name**: Lê Hữu Hưng 
- **Student ID**: 2A202600098
- **Date**: 06/04/2026

---

## I. Technical Contribution (15 Points)

- **Modules Implemented**: `src/agent/agent.py` (lines 31–43), `src/tools/tools.py` (lines 8–12)

### 1. Realtime Datetime Injection (`src/agent/agent.py:31-43`)

The `get_system_prompt()` method calls `datetime.now()` at runtime and injects the current date into the `ReAct.v2.txt` template via the `{current_date}` placeholder:

```python
def get_system_prompt(self) -> str:
    current_date = datetime.now().strftime("%A, %d/%m/%Y")
    return Path("src/prompts/ReAct.v2.txt").read_text().format(
        current_date=current_date,
        tool_descriptions=...,
    )
```

This is called on **every LLM generation step**, so the date is always fresh. The benefit: the agent can resolve relative references like "this weekend" in its first `Thought` block without needing an extra `get_system_time[]` tool call.

### 2. Web Search Tool (`src/tools/tools.py:8-12`)

Originally built on the Brave Search API, which consistently returned empty results for Vietnamese queries. Switched to **DuckDuckGo (`ddgs`)** — no API key required, reliable for Vietnamese content:

```python
ddgs = DDGS()

def web_search(query: str) -> str:
    res = ddgs.text(query, max_results=5)
    return "\n".join(t['body'] for t in res)
```

The output is plain text (no JSON/markdown) so the LLM can directly read and quote it in the next `Thought` block. The original Brave code (lines 13–41) remains as unreachable dead code to document the migration.

---

## II. Debugging Case Study (10 Points)

- **Problem Description**: `PARSE_ERROR` when the agent called `get_system_time` — it output `Action: get_system_time` instead of `Action: get_system_time[]`, causing the step to fail entirely.

- **Log Source** (`logs/2026-04-06.log`):
```json
{ "event": "PARSE_ERROR", "data": { "step": 2, "reason": "No Action or Final Answer found", "output": "Action: get_system_time" } }
```

- **Diagnosis**: The parser at `agent.py:103` uses regex `r"Action:\s*(\w+)\[([^\]]*)\]"`, which **requires** square brackets. For no-argument tools, the LLM sometimes omits the empty `[]` since it looks syntactically odd — a prompt design flaw, not a model bug.

- **Solution**: Added a nudge prompt (`agent.py:152-155`) that appends a correction reminder when a parse error occurs. The LLM self-corrects on the next step. Long-term fix: add an explicit example in the system prompt — `Action: get_system_time[]` — to eliminate the ambiguity.

---

## III. Personal Insights: Chatbot vs ReAct (10 Points)

1. **Reasoning**: The `Thought` block forces explicit decomposition before acting. For *"Thời tiết Đà Nẵng cuối tuần này?"*, the Chatbot guessed from stale training data. The Agent decomposed it: *(1) what dates are "this weekend"? → (2) search weather for those dates.* This step-by-step reasoning is impossible in a single Chatbot call, and when things go wrong, the Thought block makes the failure visible and debuggable.

2. **Reliability**: The agent performed *worse* in two situations:
   - **Simple factual queries**: ~10× token overhead from the 366-token system prompt, with zero benefit for knowledge the LLM already has.
   - **Search failure cascades**: When `web_search` returned empty results repeatedly, the agent eventually fabricated answers — worse than the Chatbot, which would at least hedge.

3. **Observation**: Every tool result is appended back to the running prompt (`agent.py:138`), creating a feedback loop. Two observed behaviors: **(a)** when a search failed, the agent's next Thought broadened the query and retried autonomously; **(b)** for budget planning, it chained `web_search` (gather prices) → `calculator[]` (compute total) — sequential multi-tool composition impossible in a single Chatbot turn.

---

## IV. Future Improvements (5 Points)

- **Scalability**: Tool calls are strictly sequential. Use `asyncio` to dispatch independent lookups in parallel and merge Observations before the next LLM step. Add a TTL cache to avoid redundant searches within a session.

- **Safety**: `web_search` results are appended verbatim into the prompt — vulnerable to prompt injection from malicious web content. Add a Supervisor LLM to sanitize Observations before they enter the agent's context.

- **Performance**: With 50+ tools, injecting all tool descriptions wastes context tokens. Use a vector database (ChromaDB/FAISS) to retrieve only the top-k relevant tools per query, keeping the prompt compact.

---

