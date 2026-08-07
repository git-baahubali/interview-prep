# AI Agents

Agent architecture from first principles: the ReAct loop, tool calling, memory, planning, multi-agent patterns, and the production concerns — guardrails, hooks, retries, and human-in-the-loop.

**Questions:** 31

Every concept here is tied to **where it sits in the agent execution lifecycle**, because that is what interviews probe. `12-langgraph.md` covers the framework-level implementation of the same ideas.

---

## Easy

---

## Q1: What is an AI agent, and how does it differ from an LLM call?

### Answer

An **agent** is an LLM in a loop with tools, where the **model decides what to do next** and the loop continues until the task is complete.

| | Single LLM call | Chain / workflow | Agent |
|---|---|---|---|
| Control flow | None | **Fixed by the developer** | **Decided by the model at runtime** |
| Steps | 1 | Predetermined N | Unbounded until done |
| Tools | No | Maybe, at fixed points | Yes, chosen dynamically |
| Can recover from failure | No | Only if coded | **Yes — it can observe and retry** |
| Predictability | High | High | **Low** |
| Cost per task | 1× | ~N× | 3–50× |

**The defining property is who controls the control flow.**

```text
LLM call:  input → model → output

Chain:     input → retrieve → model → parse → model → output      (you wrote the arrows)

Agent:     input → model decides → act → observe → model decides → ... → done
                        ↑______________________________________|
                        (the model wrote the arrows, at runtime)
```

**What makes something an agent, concretely:**
1. It can take **actions** that affect the world or gather information (tools).
2. It **observes** the results of those actions.
3. It **decides** the next action based on those observations.
4. It runs until a **termination condition** is met, not for a fixed number of steps.

**Why this matters.** An agent can handle tasks whose required steps are unknown in advance: "figure out why this build is failing" needs a different sequence of investigations every time. A chain cannot express that; an agent can.

**The cost of that flexibility** is predictability. A chain does the same thing every time and is easy to test, debug, and budget. An agent may take 3 steps or 30, cost $0.02 or $2.00, and fail in ways you did not anticipate. This is the central engineering trade-off, and the most important practical advice in agent development is: **use the least agentic architecture that solves the problem.** If a fixed chain works, ship the chain.

### Interview Follow-ups

- Is a single tool-calling LLM call an agent? (Not really — there is no loop, no observation, no iterative decision. It is a one-shot function selection. The loop is what makes it an agent.)
- What is the simplest useful agent? (An LLM with one tool and a loop — a search agent that queries, reads results, and searches again until it can answer.)

---

## Q2: What is the ReAct pattern?

### Answer

**ReAct = Reasoning + Acting.** The model alternates between generating reasoning traces and taking actions, using the result of each action to inform the next round of reasoning.

**The cycle:**

```text
Thought:      reason about the current state and what to do next
Action:       call a tool with arguments
Observation:  the tool's result is fed back into the context
  ↓ repeat until the model produces a final answer
Answer:       the final response
```

**Why interleaving matters — the core insight.** Before ReAct there were two separate ideas:
- **Chain-of-thought:** reason step by step, but with no access to external information. It reasons well and hallucinates facts.
- **Action-only agents:** call tools, but with no explicit reasoning. They act, but without a plan or the ability to interpret failures.

ReAct combines them, and each fixes the other's weakness:
- **Reasoning grounds acting** — the model plans which tool to use and why, rather than guessing.
- **Acting grounds reasoning** — real observations correct the reasoning chain, preventing it from drifting into plausible fiction.

**Worked example:**

```text
Question: "Is our API's p99 latency currently above the SLA?"

Thought:      I need the current p99 latency and the SLA threshold. Start with metrics.
Action:       query_metrics(metric="api.latency.p99", window="1h")
Observation:  {"p99_ms": 847}

Thought:      Now I need the SLA target, which should be in the docs.
Action:       search_docs(query="API latency SLA target p99")
Observation:  "The p99 latency SLA is 500ms for all public API endpoints."

Thought:      847ms > 500ms, so we are in breach. I can answer now.
Answer:       Yes — current p99 is 847ms against a 500ms SLA, a 69% breach.
```

**Where ReAct sits in the lifecycle.** ReAct is the **loop structure itself** — the scaffolding into which everything else in this file plugs. Tool calling is how the Action step executes; memory is what persists across iterations; guardrails wrap the Action step; reflection is an extra Thought step; planning front-loads the Thoughts.

**Modern reality.** The original ReAct paper used text parsing ("Thought:", "Action:") because models had no structured tool calling. Today, native tool calling handles the Action step and reasoning models produce the Thought step internally. So you rarely implement ReAct literally with string parsing — but **the conceptual loop is exactly what every agent framework implements**, and it is what interviewers mean when they ask about agent architecture.

### Interview Follow-ups

- Do reasoning models make ReAct obsolete? (No — they improve the Thought step's quality, but you still need the loop, the tool execution, and the observation feedback. Reasoning models made *implicit* reasoning good enough that you no longer need to prompt for it explicitly.)
- What happens without the Thought step? (Tool selection degrades, multi-step tasks fail more, and failures cannot be diagnosed or recovered from — the model has no explicit representation of what it is trying to do.)

---

## Q3: Walk through the full agent execution lifecycle.

### Answer

This is the map that every other question in this file refers to.

```text
1. INITIALISATION
   - Build the system prompt: role, instructions, constraints, output format
   - Register tool schemas
   - Load memory: conversation history, long-term facts, retrieved context
   - Initialise state: step counter, token budget, deadline

2. INPUT GUARDRAILS
   - Validate/sanitise the user input
   - PII detection, injection screening, rate limits, policy checks
   - Reject or redact before the model ever sees it

3. LLM CALL (the "Thought")
   - Send: system prompt + memory + tool schemas + conversation
   - Model returns EITHER a final answer OR one or more tool calls

4. DECISION POINT
   - Final answer? → go to 8
   - Tool call(s)? → go to 5

5. BEFORE-TOOL HOOKS / PRE-EXECUTION
   - Validate arguments against the schema
   - Authorisation check: is this agent, for this user, allowed this call?
   - Guardrails: is this action destructive? Does it need approval?
   - Human-in-the-loop interrupt if required
   - Logging, tracing, cost accounting

6. TOOL EXECUTION
   - Run the tool (API call, DB query, retrieval, computation, sub-agent)
   - Timeout, retry with backoff, circuit breaking
   - Catch and convert errors into model-readable messages

7. AFTER-TOOL HOOKS / POST-EXECUTION
   - Validate/filter the result (secret redaction, size limits, truncation)
   - Output guardrails on tool results (they may be untrusted content)
   - Append the observation to the message history
   - Update memory/state
   - Check termination conditions: step cap, token budget, deadline, no-progress
   - → back to 3

8. FINAL OUTPUT
   - Output guardrails: policy, PII, groundedness, format validation
   - Stream to the user
   - Persist state and memory
   - Emit traces, metrics, and evaluation signals
```

**The three things this diagram makes obvious, and that interviews reward:**

1. **The loop is steps 3–7.** Everything interesting happens in the cycle, and the number of iterations is not known in advance.
2. **Hooks bracket tool execution (5 and 7).** This is where authorisation, human approval, redaction, and observability live — *not* in the prompt. Anything you need to *enforce* goes here.
3. **Termination is a separate concern from the model's judgement.** The model decides when it is *done*; the harness decides when it must *stop* regardless. You need both, or an agent can loop indefinitely.

**What you must instrument at every step:** step number, model input/output tokens, tool name and arguments, tool latency and result size, errors, and cumulative cost. Agent debugging without traces is impossible, because the failure is usually three steps before the symptom.

### Interview Follow-ups

- Where does context engineering fit? (Step 3's input construction — deciding what memory, tool results, and instructions to include, and what to compress or drop. See Q19.)
- What is the most common place production agents break? (Step 6 — tool execution: an API returns an unexpected error shape, the agent misreads it, and either loops retrying or reports success incorrectly.)

---

## Q4: What is tool calling / function calling, and how does it work mechanically?

### Answer

**Tool calling** (also *function calling* — the terms are used interchangeably) lets a model request the execution of a developer-defined function by emitting a structured call, which your code executes and returns the result of.

**The critical point that interviews test: the model never executes anything.** It emits a *request*. Your code decides whether and how to run it. The model is a very good argument-filler with a natural-language interface; all authority remains in your harness.

**Mechanically, step by step:**

```text
1. You send tool SCHEMAS with the request (name, description, JSON Schema for parameters)
2. The model decides a tool is needed and emits a structured tool_use block:
      {"name": "get_weather", "input": {"city": "Tokyo", "unit": "celsius"}}
   with a unique tool_use_id
3. The API returns with stop_reason = "tool_use" — generation pauses
4. YOUR CODE validates the arguments and executes the function
5. You append a tool_result message referencing the same tool_use_id
6. You call the model again with the full history including the result
7. The model either answers or requests another tool
```

**How the model does it under the hood.** The tool schemas are serialised into the prompt (or a special format the model was trained on). The model was post-trained on many examples of "given these schemas and this request, emit this structured call." Constrained decoding then ensures the emitted JSON is syntactically valid and conforms to the schema — so malformed JSON is largely a solved problem, while *semantically wrong* arguments are not.

**Where it sits in the lifecycle:** steps 3–7 of Q3. Tool calling is the mechanism of the ReAct **Action** and **Observation** steps.

**Parallel tool calls.** Modern models can emit several independent tool calls in one turn. Execute them concurrently and return all results together — a significant latency win when the calls do not depend on each other. Preserve the `tool_use_id` mapping so results match requests.

### Example

```python
tools = [{
    "name": "get_order_status",
    "description": (
        "Retrieve the current status and tracking information for a customer order. "
        "Use this when the user asks about the whereabouts, delivery date, or state "
        "of a specific order. Requires the exact order ID."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "order_id": {
                "type": "string",
                "description": "Order ID in the format ORD-XXXXXXX, e.g. ORD-8842190",
            },
        },
        "required": ["order_id"],
    },
}]

def run_agent(user_message, max_steps=10):
    messages = [{"role": "user", "content": user_message}]

    for _ in range(max_steps):
        response = client.messages.create(
            model="claude-sonnet-5", max_tokens=2048, tools=tools, messages=messages
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return response                      # final answer

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = execute_tool(block.name, block.input)   # YOUR code runs it
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,      # must match
                    "content": output,
                })
        messages.append({"role": "user", "content": results})

    raise StepLimitExceeded("Agent did not complete within the step budget")
```

### Interview Follow-ups

- What is the difference between "function calling" and "tool use"? (Essentially none — different vendor terminology for the same mechanism. "Tool" is now the more common term because tools include things that are not functions, like retrieval or a sub-agent.)
- What happens if you never return a tool result? (The conversation is malformed — most APIs reject a subsequent call, because every `tool_use` block must be answered by a matching `tool_result`.)

---

## Q5: What makes a good tool schema?

### Answer

**Tool schemas are prompts.** The model's only knowledge of your tool is its name, description, and parameter descriptions. Schema quality is the dominant factor in tool-selection accuracy — more than model choice, in most cases.

**The rules:**

**1. Write the description for a new colleague, not a compiler.** State what the tool does, **when to use it**, when *not* to use it, and any prerequisites. Include the "when" — it is the part that most affects selection accuracy.

**2. Describe every parameter, with format and examples.**

```python
# Weak
{"date": {"type": "string"}}

# Strong
{"date": {
    "type": "string",
    "description": "Date in ISO format YYYY-MM-DD, e.g. 2025-03-14. Must not be in the future.",
}}
```

**3. Use enums instead of free-form strings** wherever the value set is closed. This makes invalid values impossible rather than merely discouraged.

**4. Keep the parameter count small.** More than ~5 parameters and error rates climb sharply. Split the tool or provide sensible defaults.

**5. Make tool names distinct and unambiguous.** `search_docs` and `search_knowledge_base` will be confused. Either merge them or make the distinction explicit in both names and descriptions.

**6. Limit the number of tools.** Accuracy degrades noticeably past roughly 10–20 tools, and the schemas consume context. Beyond that, group tools by task and load only the relevant subset, or use a routing layer / sub-agents.

**7. Return errors the model can act on.** This is heavily underrated:

```python
# Useless
"Error: 400"

# Actionable
"Error: order_id 'ORD-884219' is invalid — expected 7 digits after ORD-, got 6. "
"Verify the ID with the customer and retry."
```

The model *can* self-correct given a good error message, which turns a hard failure into a recovered step.

**8. Design tool outputs for a context window.** Do not return 10,000 rows. Paginate, summarise, or return a handle the model can query further. Every token of tool output competes with everything else in the context.

**9. Make destructive operations explicit and separate.** `delete_customer_record` should not be reachable from a generic `update_record` tool. Separate tools make authorisation and approval gates possible.

### Example

```python
{
    "name": "search_support_tickets",
    "description": (
        "Search historical customer support tickets by text query and optional filters. "
        "Use this to find how similar issues were resolved previously, or to check a "
        "customer's support history. "
        "Do NOT use this for current order status (use get_order_status) or for "
        "product documentation (use search_docs). "
        "Returns at most 20 tickets ordered by relevance."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "Natural-language description of the issue, e.g. 'payment declined on renewal'",
            },
            "status": {
                "type": "string",
                "enum": ["open", "resolved", "escalated", "any"],
                "description": "Filter by ticket status. Defaults to 'any'.",
            },
            "customer_id": {
                "type": "string",
                "description": "Optional. Restrict to one customer's tickets. Format: CUST-XXXXXX.",
            },
            "since": {
                "type": "string",
                "description": "Optional ISO date YYYY-MM-DD. Only tickets created on or after this date.",
            },
        },
        "required": ["query"],
    },
}
```

### Interview Follow-ups

- Your agent keeps picking the wrong tool. What do you do? (Fix the descriptions first — add explicit "use this when / do not use this for" guidance. Then check for overlapping tools and consider merging. Model changes and prompt tweaks come after.)
- How do you handle 100 tools? (Do not expose 100. Route to a task-relevant subset, use a hierarchy of sub-agents each owning a domain, or retrieve relevant tool schemas semantically based on the query.)

---

## Intermediate

---

## Q6: How does an agent's context evolve across a conversation, and what breaks?

### Answer

**The context grows monotonically** as the loop runs:

```text
Step 1:  system + tools + user message                          ~1,500 tokens
Step 2:  + assistant tool_use + tool_result                     ~3,200 tokens
Step 3:  + assistant tool_use + tool_result (large)             ~14,000 tokens
Step 4:  + assistant tool_use + tool_result                     ~18,000 tokens
Step 8:  ...                                                    ~60,000 tokens
Step 15: ...                                                   ~150,000 tokens → problems
```

**What breaks as it grows:**

| Problem | Mechanism |
|---|---|
| **Cost grows quadratically over a task** | Every step resends the whole history. 15 steps averaging 50k tokens = 750k input tokens for one task. |
| **Latency grows** | Prefill scales with input length; each step gets slower. |
| **Attention dilutes** | The original instruction is buried under 100k tokens of tool output. The agent forgets its goal. |
| **Lost in the middle** | Middle-of-context information is attended to less reliably. |
| **Context window exhaustion** | Hard failure, usually mid-task. |
| **Error accumulation** | Failed attempts stay in context and the model may repeat them or be confused by them. |

**The dominant contributor is almost always tool results** — a database query returning 500 rows, a file read of 3,000 lines, a web page's full HTML. One careless tool can consume more context than the entire rest of the conversation.

**Mitigations, in order of impact:**

1. **Cap and shape tool output at the source.** Paginate, project only needed fields, truncate with an explicit note ("showing 20 of 4,312 results; refine your query"). This is the highest-leverage fix and it belongs in the tool, not the prompt.
2. **Prompt caching.** The stable prefix (system prompt + tools + early turns) is cached, so you pay full price once. This substantially reduces the cost of long agent loops — order the context stable-to-volatile to maximise the cacheable prefix.
3. **Compress old turns.** Summarise or drop tool results older than N steps, keeping the tool call and a one-line outcome. The agent rarely needs the raw output of step 2 at step 12.
4. **Externalise state.** Write intermediate results to a file or scratchpad and keep only a reference in context. The agent reads it back when needed — the filesystem becomes memory.
5. **Structured state instead of raw history.** Maintain an explicit "findings so far" object that the agent updates, and pass that rather than the full transcript.
6. **Sub-agents for context isolation.** Delegate a sub-task; the sub-agent burns its own context and returns only a summary (see Q13).

**The framing that shows understanding:** the context window is a **budget you actively manage**, not a container you fill. The discipline of deciding what belongs in context at each step is what "context engineering" means (Q19), and it is the difference between an agent that works for 5 steps and one that works for 50.

### Interview Follow-ups

- Why is compressing history risky? (You may drop the detail the agent needs later. Keep the tool *calls* and outcomes even when dropping raw results, so the agent knows what it already tried.)
- What is the cheapest big win? (Prompt caching plus capping tool output sizes. Neither changes agent logic and both routinely cut cost by more than half.)

---

## Q7: What is agent memory, and what types are there?

### Answer

**Memory is any information carried across steps or sessions that is not in the current message.** Different types serve different lifecycles.

| Type | Scope | Contents | Storage |
|---|---|---|---|
| **Working / short-term** | Current task | Message history, tool results, scratchpad | The context window |
| **Episodic** | Across sessions | What happened before — past conversations, past task outcomes | DB, retrieved when relevant |
| **Semantic** | Across sessions | Facts learned — user preferences, entity knowledge | Key-value store or vector store |
| **Procedural** | Across sessions | How to do things — learned workflows, successful patterns | Prompt/instructions, skill definitions |

**Mapping to the lifecycle (Q3):** working memory *is* the context assembled at step 3. Long-term memory is loaded at step 1 (initialisation) and written at step 7 or 8.

**Working memory** is the message history plus any scratchpad. It is bounded by the context window and is the subject of Q6. This is where most agents actually live.

**Episodic memory** answers "what happened before?" A support agent recalling that this customer had a billing problem last month. Implementation: store conversation summaries with metadata, retrieve semantically or by user id at session start.

**Semantic memory** answers "what do I know?" Facts extracted from interactions: "the user prefers Python", "their production region is eu-west-1", "they use pnpm not npm". Implementation: extract facts with an LLM after a session, deduplicate against existing facts, store, and inject relevant ones into the system prompt.

**Procedural memory** answers "how do I do this?" Learned or authored workflows. In practice this is usually the system prompt, tool definitions, and **skills** (Q9) — instructions rather than data.

**The hard problems, which are what interviews probe:**

1. **What to remember.** Storing everything reproduces the context problem in a database. Extract selectively: stable preferences and durable facts, not transient details.
2. **When to retrieve.** Injecting all memory into every prompt is wasteful and distracting. Retrieve by relevance to the current query.
3. **Conflict and staleness.** "The user works at Acme" is true until it is not. Store timestamps, prefer recent facts, and allow explicit updates and deletion.
4. **Contamination.** A wrong fact written to memory persists and corrupts every future session. Memory writes need validation, and users need visibility and a delete button.
5. **Privacy.** Persisted memory is personal data — subject to consent, retention limits, and erasure requests.

**Practical advice:** most agents need only working memory done well plus a small, explicit user-preference store. Elaborate memory architectures are frequently built before anyone has established that they improve outcomes. Start with a simple key-value preference store and a session summary; add sophistication when measurement demands it.

### Interview Follow-ups

- How is memory different from RAG? (RAG retrieves from a curated external corpus; memory retrieves from the agent's own accumulated experience. Mechanically similar — often the same vector store — but memory is written by the agent and is per-user, which makes correctness and privacy much harder.)
- How do you let users control memory? (Show them what is stored, allow editing and deletion, and require explicit confirmation for sensitive facts. Opaque memory is a trust and compliance problem.)

---

## Q8: What is the difference between a tool and a skill?

### Answer

| | Tool | Skill |
|---|---|---|
| What it is | A **function** the model can call | A **package of instructions** (and optionally resources) for doing a task |
| Provides | New *capability* — access to something outside the model | New *know-how* — procedure, conventions, expertise |
| Mechanism | Schema + executable code; the model emits a call | Text loaded into context; the model reads and follows it |
| Executes | Yes, deterministically in your runtime | No — it guides the model's own behaviour |
| Example | `query_database(sql)` | "How to write a compliant incident report at this company" |
| Failure mode | API error, timeout, bad arguments | Model ignores or misapplies the instructions |

**The distinction in one line:** a tool extends what the agent **can do**; a skill extends what the agent **knows how to do**.

**Why skills exist.** You cannot fit every procedure into a system prompt — it would be enormous, and most of it is irrelevant to any given task. Skills are **progressively disclosed**: the agent sees short skill descriptions, and loads a skill's full instructions into context only when the task calls for it. This is context engineering applied to expertise.

**A skill typically contains:**
- A name and a one-line description (so the agent can decide when it is relevant)
- Detailed step-by-step instructions
- Conventions, templates, and examples
- References to which tools to use and in what order
- Optionally: scripts, schemas, or reference files it can read

**Where each sits in the lifecycle:**
- **Tools** are registered at step 1 and invoked at step 6.
- **Skills** are discovered at step 1 (descriptions only) and their content is loaded into context at step 3 when relevant — changing *how* the agent reasons and *which* tools it chooses, without adding any new capability.

**A concrete comparison.** Suppose the task is "run our monthly revenue report."
- The **tools** are `query_warehouse`, `render_chart`, and `send_email`. Without them the agent cannot touch data.
- The **skill** is the document explaining which tables hold revenue, which exclusions apply, how the company defines "recognised revenue", the required chart format, and who receives the report. Without it, the agent has the capability but not the competence — it will produce a plausible, wrong report.

**When to use which:**
- Need to *access* or *change* something outside the model → **tool**
- Need the model to follow a specific *procedure* or use *domain conventions* → **skill**
- Both, usually: skills tell the agent how to use the tools correctly.

**The relationship to sub-agents.** A skill guides the *current* agent in the *current* context. A sub-agent runs the task in a *separate* context and returns a result (Q13). Use a skill when you want the work done here with guidance; use a sub-agent when you want the work done elsewhere to protect your context.

### Interview Follow-ups

- Can a skill include code? (Yes — a skill may bundle scripts the agent runs via a generic execution tool. That blurs the line, but the distinction still holds: the skill is the instructions, the execution tool is the capability.)
- Why not just put all skills in the system prompt? (Context cost and dilution. Ten skills at 2,000 tokens each is 20,000 tokens of mostly-irrelevant instruction on every call, degrading attention on the actual task.)

---

## Q9: What are planning and reflection in agents?

### Answer

**Both are ways of adding deliberate thinking to the loop — planning before acting, reflection after.**

**Planning: decide the sequence of steps before executing them.**

Approaches:

| Pattern | How | Trade-off |
|---|---|---|
| **Implicit (ReAct)** | Plan one step at a time as you go | Adaptive; can lack global coherence and may wander |
| **Plan-and-execute** | Generate a full plan upfront, then execute each step | Coherent, cheaper (planning once); brittle if reality differs |
| **Plan-and-replan** | Plan, execute, revise the plan when observations contradict it | Best of both; more LLM calls |
| **Decomposition** | Break into independent sub-tasks, possibly run in parallel | Great when sub-tasks are genuinely independent |
| **Tree/graph search** | Explore multiple plan branches and evaluate them | Powerful for search problems; very expensive |

**Where planning helps.** Long-horizon tasks (10+ steps), tasks where step order matters, tasks with parallelisable sub-work, and tasks where you want to show the user the plan before executing (a natural human-in-the-loop point). Writing the plan into a persistent artefact — a todo list the agent updates — is a simple and effective technique: it survives context compression and keeps the goal salient.

**Where planning hurts.** Short tasks, where planning is pure overhead. And highly uncertain tasks, where an upfront plan is fiction — you cannot plan a debugging session before seeing the first error.

**Reflection: evaluate your own output and revise it.**

```text
Generate → Critique → Revise → (repeat until acceptable or budget exhausted)
```

Variants:
- **Self-critique.** The same model reviews its own output against criteria. Cheap; limited, because the model's blind spots are shared between the generation and the critique.
- **Separate critic.** A different prompt, or a different model, evaluates. Meaningfully better — a fresh context has no commitment to the original answer.
- **Grounded reflection.** The critique uses *external* signals: test results, a linter, a compiler, a validation schema, a retrieval check. **This is where reflection genuinely works**, because the feedback is real rather than self-generated.

**The honest assessment, and the answer interviews reward.** Pure self-reflection produces modest and inconsistent gains — a model that could not get it right the first time often cannot diagnose why. Reflection grounded in **external verification** is transformative: run the tests, read the failures, fix, repeat. The difference is whether the feedback contains information the model did not already have.

So the design rule is: **prefer verifiable feedback over self-assessment.** If you can execute, lint, type-check, validate against a schema, or compare against a retrieved source, do that instead of asking the model whether it did well.

**Cost control for both:** cap planning to one plan plus N replans; cap reflection to 2–3 iterations (gains flatten fast and cost is linear); and require a *specific, actionable* critique rather than a vague quality score.

### Interview Follow-ups

- Why does self-critique often fail to catch errors? (The critique is generated by the same distribution that produced the error. If the model believed a wrong fact when generating, it likely still believes it when critiquing.)
- When is plan-and-execute clearly better than ReAct? (When the task is well-understood and long, e.g. "migrate these 40 files to the new API" — the plan is enumerable upfront, and planning once instead of 40 times saves substantial cost.)

---

## Q10: What is a router agent?

### Answer

**A router classifies the request and dispatches it to the right handler.** It makes one decision and does not execute the work itself.

```text
User request → Router → one of:
                        ├── billing specialist agent
                        ├── technical support agent
                        ├── simple FAQ chain (no agent needed)
                        └── human escalation
```

**Why route:**
1. **Specialisation.** Each handler gets a focused prompt and only its relevant tools — far more accurate than one agent with 40 tools and a 5,000-token prompt covering every domain.
2. **Cost.** Simple requests go to a cheap fast path; only complex ones pay for a large model and a full agent loop.
3. **Latency tiers.** An FAQ answer in 300 ms, a complex investigation in 20 seconds.
4. **Safety and permissions.** Different handlers can have genuinely different authority — the billing agent can issue refunds, the FAQ path cannot do anything.
5. **Maintainability.** Teams can own individual handlers independently.

**Implementation options (same trade-offs as RAG routing, `10-rag.md` Q9):**

| Method | Latency | Flexibility |
|---|---|---|
| Rules/regex on keywords | ~0 ms | Brittle |
| Embedding similarity to route descriptions | ~5 ms | Good, cheap |
| Small LLM classifier | ~200–500 ms | Best handling of nuance |
| Tool calling (routes as tools) | Part of the main call | Natural, composable |

**Router vs supervisor — a distinction interviews test.**

| | Router | Supervisor |
|---|---|---|
| Decisions | **One** — pick a handler | **Many** — orchestrates a multi-step workflow |
| Control returns to it | No | **Yes**, after each sub-agent |
| Can combine results | No | Yes |
| Cost | One classification | One call per delegation round |

A router is a **switch**; a supervisor is a **manager** (Q12). If control comes back for another decision, it is a supervisor.

**Design requirements:**
- **A default route.** Never fail because a query was unclassifiable; route ambiguous cases to a general handler or to fan-out.
- **Confidence thresholds.** Below the threshold, fan out or ask the user a clarifying question rather than guessing.
- **Handle mixed intent.** "My payment failed and the app is crashing" spans two routes. Either allow multi-route dispatch or route to whichever handler can escalate.
- **Route logging.** You cannot improve a router you do not measure; log the chosen route and the eventual outcome.

**Where it sits in the lifecycle:** the router runs *before* step 1 of the target agent's lifecycle — it selects which agent's lifecycle to start.

### Interview Follow-ups

- What is the failure mode of routing? (Confident misrouting — the billing agent competently answers a technical question wrongly. Mitigate with confidence thresholds, an escalation path from each handler back to the router, and monitoring of re-routed conversations.)
- When is a router unnecessary? (Under ~5 tools with one coherent domain, a single agent is simpler and often better. Routing adds a component and a failure mode.)

---

## Q11: What is a sub-agent, and when should you use one?

### Answer

**A sub-agent is an agent invoked as a tool by another agent.** It has its own system prompt, tools, and — crucially — **its own context window**. It performs a task and returns a result.

```text
Main agent context (stays small)
  └── calls research_subagent("find all callers of process_payment")
        └── Sub-agent context (burns 80k tokens reading 40 files)
              └── returns: "12 callers, in these 5 modules: ..."   (200 tokens)
```

**The primary reason to use one is context isolation.** A task like "read these 40 files and tell me which ones call this function" consumes enormous context but produces a small answer. Running it in the main agent poisons the main context with 40 file contents that are irrelevant once the answer is known. A sub-agent absorbs that cost and returns only the conclusion.

**Other reasons:**
- **Specialisation.** A focused prompt and a small tool set outperform a generalist.
- **Parallelism.** Spawn N sub-agents on independent sub-tasks and run them concurrently — the biggest wall-clock win available in agent design.
- **Permission scoping.** A sub-agent can have strictly fewer tools; e.g. a read-only research sub-agent that structurally cannot write.
- **Reusability.** The same sub-agent serves many parent workflows.

**When NOT to use a sub-agent:**
- The task needs the parent's full conversational context (the sub-agent does not have it, and passing it defeats the purpose).
- The task is short — the overhead of spawning, prompting, and summarising exceeds the benefit.
- The result is inherently large — if the sub-agent must return 50k tokens, you saved nothing.
- Tight coupling — if the parent needs to intervene mid-task, a sub-agent's opacity is a liability.

**The fundamental limitation to state in an interview: information loss at the boundary.** The sub-agent returns a summary. If it summarised away the detail the parent needed, the parent cannot recover it without re-running the work. This means:
- Sub-agent task descriptions must be **specific and complete** — the sub-agent cannot ask clarifying questions of the user.
- The **return contract matters**: specify exactly what to return and in what shape. A schema is better than a prose instruction.
- Parallel sub-agents **cannot coordinate** with each other. If sub-task B depends on A's findings, they must be sequential.

**Design guidance.** Give each sub-agent one clear objective, an explicit output format, and the minimum tools required. Pass all necessary context in the task description. Cap its steps and budget independently. Log its full trace — a sub-agent failure is invisible in the parent's transcript otherwise, which makes debugging painful.

### Interview Follow-ups

- Sub-agent vs tool: when does a task deserve an agent rather than a function? (If the task requires *judgement and iteration* — deciding what to look at next based on what it found — it needs an agent. If it is a deterministic operation, a tool is cheaper and more reliable.)
- How deep should nesting go? (One level in practice. Two levels multiply the information loss and make tracing very hard; most frameworks deliberately restrict it.)

---

## Q12: What is a supervisor agent, and what are the main multi-agent topologies?

### Answer

**A supervisor orchestrates other agents:** it decides who works next, passes them context, receives their results, and decides whether the overall task is complete.

**The main topologies:**

| Topology | Structure | Best for | Weakness |
|---|---|---|---|
| **Single agent** | One agent, many tools | Most tasks — the correct default | Degrades past ~15–20 tools |
| **Router** | Classify → dispatch once | Distinct request categories | No orchestration |
| **Supervisor** | Central coordinator delegating to specialists | Multi-domain tasks needing combination | Supervisor is a bottleneck; context passing is lossy |
| **Hierarchical** | Supervisors of supervisors | Very large task decomposition | Deep information loss, hard to debug |
| **Sequential pipeline** | A → B → C, fixed order | Known-stage workflows (research → write → edit) | Not adaptive; really a chain of agents |
| **Network / swarm** | Any agent hands off to any other | Fluid collaboration | Hardest to control, can loop, unpredictable cost |
| **Parallel + aggregate** | Fan out, then combine | Independent sub-tasks; multiple perspectives | Sub-agents cannot coordinate |

**The supervisor loop:**

```text
1. Supervisor receives the task
2. Decides which specialist to invoke, with what instruction
3. Specialist executes in its own context, returns a result
4. Supervisor evaluates: task complete? need another specialist? need a retry?
5. Loop back to 2, or synthesise the final answer
```

**Why multi-agent is attractive:** specialisation improves quality, context isolation keeps each agent's window manageable, parallelism cuts wall-clock time, and independent components are separately testable and ownable.

**Why it usually is not worth it — the point interviews look for:**

1. **Information loss at every boundary.** Each handoff is a summary. Detail that mattered gets dropped, and the receiving agent cannot ask for it back.
2. **Cost multiplication.** Each agent has its own system prompt and reasoning overhead. Multi-agent systems routinely cost 5–15× a single agent for the same task.
3. **Debugging difficulty.** A wrong final answer might originate in any of six agents' contexts. You need full distributed tracing.
4. **Coordination failures.** Agents duplicate work, contradict each other, or ping-pong a task without progressing.
5. **Latency.** Sequential handoffs serialise LLM calls, each of which is seconds.

**The strong recommendation: start with a single agent.** Move to multi-agent only when you hit a specific, identified wall:
- Too many tools for reliable selection → split by domain
- Context exhaustion on sub-tasks → sub-agents for isolation
- Genuinely parallelisable work with latency pressure → fan out
- Different permission boundaries required → separate agents

**And prefer the least connected topology that works.** A supervisor with specialists is far easier to reason about than a network where anyone can hand off to anyone. Constrain the graph.

### Interview Follow-ups

- Why do multi-agent systems fail more than expected? (Errors compound multiplicatively across handoffs. Five agents at 90% reliability each yield ~59% end-to-end. Reliability engineering matters more than adding agents.)
- What is the strongest genuine case for multi-agent? (Parallel research: N sub-agents investigating independent questions simultaneously, then one synthesis. The work is genuinely independent, so no coordination is needed, and the wall-clock saving is real.)

---

## Q13: How do agents handle errors and retries?

### Answer

**Errors are normal in agent execution, not exceptional.** APIs time out, arguments are wrong, rate limits hit, and results are empty. A production agent's quality is largely determined by how it handles these.

**Two categories, handled at different layers:**

**1. Infrastructure errors — handle in code, before the model sees them.**

| Error | Handling |
|---|---|
| Timeout | Retry with exponential backoff + jitter |
| Rate limit (429) | Backoff honouring `Retry-After`; queue |
| Transient 5xx | Retry with backoff, bounded attempts |
| Network failure | Retry; circuit-break after repeated failures |
| Model API overload | Retry; fall back to another model |

These are **deterministic and idempotent-safe to retry**. Do not involve the model — burning an LLM call to decide whether to retry a 503 is wasteful and unreliable. Handle it in the tool execution layer (step 6 of Q3).

**2. Semantic errors — surface to the model, which can actually fix them.**

| Error | Handling |
|---|---|
| Invalid arguments | Return a precise, actionable error; the model corrects and retries |
| Not found | Return "no results" plus a suggestion to broaden the query |
| Permission denied | Tell the model it lacks access so it stops trying and reports honestly |
| Ambiguous request | Return the ambiguity so the model can ask the user |
| Validation failure | Return which constraint failed and why |

**The key principle: write error messages for the model as you would for a developer.**

```python
# Bad — the model cannot act on this
return "Error: invalid input"

# Good — the model corrects itself next step
return ("Error: date '2025-13-01' is not a valid date (month 13 does not exist). "
        "Provide a date in YYYY-MM-DD format with month between 01 and 12.")
```

This turns a dead end into a recovered step, and it is one of the highest-value, lowest-effort improvements available.

**Retry policy essentials:**
- **Bound attempts** (2–3 for infrastructure, 1–2 for model self-correction).
- **Exponential backoff with jitter** to avoid thundering herds.
- **Only retry idempotent operations automatically.** Retrying `charge_card` may double-charge. Use idempotency keys, or require explicit confirmation.
- **Distinguish retryable from terminal.** A 400 will not succeed on retry; a 503 might.
- **Track cumulative failures** across the whole task, not just per call.

**Loop-level failure handling — the part that gets forgotten:**

| Condition | Response |
|---|---|
| Same tool with same arguments 3× | Break the loop — the agent is stuck. Report what it tried. |
| Step cap reached | Stop; return partial results with an explicit statement of what is incomplete |
| Token/cost budget exhausted | Stop; summarise progress |
| No progress (no new information for N steps) | Stop or escalate to a human |
| A required tool is persistently down | Degrade gracefully: tell the user what cannot be done right now |

**The most important design rule: fail loudly and informatively, never silently or falsely.** An agent that reports "I've updated your billing address" without having done so is far worse than one that says "I couldn't reach the billing service — please try again or contact support." Verify side-effecting operations succeeded before reporting success, and make partial completion visible.

### Interview Follow-ups

- How do you detect a stuck agent? (Hash (tool_name, arguments) per call and count repeats; also track whether the token count of *new* information is growing. Both are cheap and catch most loops.)
- Should the model see infrastructure errors at all? (Only after retries are exhausted, and then framed as a capability statement: "the metrics service is unavailable" so it can adapt its plan rather than retry blindly.)

---

## Q14: What are guardrails, and where do they belong in the lifecycle?

### Answer

**Guardrails are enforced constraints on agent behaviour.** The word "enforced" is the whole point — an instruction in a prompt is a preference, not a guardrail.

**The layers, mapped to the lifecycle (Q3):**

| Layer | Lifecycle step | What it does | Enforcement |
|---|---|---|---|
| **Input guardrails** | 2 | PII detection, injection screening, topic/policy filtering, rate limits | Hard — reject before the model sees it |
| **System prompt** | 1, 3 | Role, scope, refusal instructions, tone | **Soft — probabilistic** |
| **Tool availability** | 1 | Which tools exist at all | **Hard — cannot call what is not registered** |
| **Tool authorisation** | 5 (before-tool) | Is *this* call, by *this* user, permitted? | **Hard — the real boundary** |
| **Argument validation** | 5 | Schema, ranges, allowlists, blast-radius checks | **Hard** |
| **Human approval** | 5 | Interrupt for consequential actions | **Hard** |
| **Tool result filtering** | 7 (after-tool) | Redact secrets, cap size, screen for injected instructions | **Hard** |
| **Output guardrails** | 8 | PII, policy, groundedness, format validation | Hard — can block or rewrite |
| **Budget limits** | 5, 7 | Step cap, token cap, cost cap, deadline | **Hard** |

**The central insight: the tool authorisation layer is the real security boundary.**

A prompt saying "only issue refunds under $100" will be followed most of the time and can be talked around. A check in the refund tool that rejects amounts over $100 for this user's role cannot be. Every constraint that actually matters must be enforced in code at step 5, where you control execution — not in the prompt, where the model is merely persuaded.

State this clearly in interviews. It is the difference between an agent demo and an agent you can deploy.

**Guardrail design principles:**

1. **Deny by default.** Grant the minimum tools and the narrowest scopes. Most agents are given far more authority than their task requires.
2. **Bound the blast radius.** Not just "can it call this tool" but "on how many records, in which environment, up to what value."
3. **Separate read from write.** Read-heavy agents need no write tools. Splitting them makes the dangerous surface small and auditable.
4. **Human-in-the-loop for irreversible actions** (Q15). Anything you cannot undo needs approval.
5. **Treat all external content as untrusted** — retrieved documents, web pages, tool results, and other agents' output can all carry injected instructions (see `10-rag.md` Q19).
6. **Break the exfiltration path.** The lethal trifecta is untrusted content + private data access + an outbound channel. Remove any one of the three; the outbound channel is usually the easiest.
7. **Log everything** for audit and post-incident analysis.
8. **Fail closed.** If a guardrail service is unavailable, block rather than allow.

**On cost:** guardrails add latency (an input-screening model call, a validation step) and can produce false positives that frustrate users. Tier them — cheap deterministic checks always, expensive model-based checks only on risky paths.

### Interview Follow-ups

- Why is "add it to the system prompt" not a guardrail? (Instruction-following is probabilistic and adversarially bypassable. Prompts shape typical behaviour; code enforces limits. You need both, and you must know which is which.)
- What is the single most effective guardrail? (Not granting the tool. The second most effective is human approval on irreversible actions. Both are structural, not persuasive.)

---

## Q15: What is human-in-the-loop, and how do you implement it?

### Answer

**Human-in-the-loop (HITL)** inserts a human decision into the agent's execution — approving, editing, answering, or redirecting — before the agent continues.

**The four patterns:**

| Pattern | The human… | Use for |
|---|---|---|
| **Approve / reject** | Confirms an action before it runs | Irreversible or costly actions |
| **Edit** | Modifies the proposed action or arguments | Near-right actions needing correction |
| **Provide input** | Answers a question the agent cannot resolve | Missing information, ambiguity, preferences |
| **Review output** | Checks the final result before delivery | High-stakes external communication |

**Where it belongs in the lifecycle:** step 5 (before tool execution) for approvals and edits, step 3 or 6 for input requests, step 8 for output review. Approval must happen *before* the side effect, which means the interrupt has to be inside the pre-execution hook — not after the tool has already run.

**When to require a human — the decision criteria:**

```text
Require approval when the action is:
  - Irreversible          (delete, send, publish, pay, deploy to production)
  - Externally visible    (emails, tickets, social posts, customer messages)
  - Financially material  (above a threshold)
  - Legally consequential (contracts, regulatory filings, medical/legal advice)
  - Broad in blast radius (bulk operations, schema changes, permission changes)
  - Low confidence        (the agent itself signals uncertainty)
```

Conversely, do **not** gate reads, idempotent operations, reversible changes, or low-value routine actions — approval fatigue is a real failure mode. If humans approve 200 requests a day, they stop reading them, and the guardrail becomes theatre while still costing latency.

**The implementation requirement that makes HITL hard: durable state.**

An approval may take minutes, hours, or days. That means the agent cannot be a running process holding state in memory:

```text
1. Agent reaches an approval point
2. PERSIST the full execution state (messages, step count, pending action)
3. Emit an approval request (UI, Slack, email) with a resume token
4. Process exits — no resources held
5. Human responds, possibly much later, possibly on a different machine
6. LOAD the state by token, apply the decision, RESUME from that exact point
```

This is why **checkpointing/persistence is a prerequisite for HITL**, and why agent frameworks couple the two (see `12-langgraph.md` Q14 and Q17). Without durable state you can only support HITL within a single live request.

**Other implementation requirements:**
- **Show the human what they are approving** — the exact tool, the exact arguments, and the likely consequence. "Approve action?" is useless.
- **Timeouts with a default** — usually reject/escalate, never silently approve.
- **Audit trail** — who approved what, when, and with what modification.
- **Batch related approvals** to reduce fatigue.
- **Allow scoped standing authorisation** ("approve all refunds under $50 for this session") — but scope it in time and blast radius, and make it revocable.
- **Handle rejection gracefully** — the agent must incorporate "the human said no" as an observation and adapt, not retry the same action.

**The progressive-autonomy pattern:** launch with approval on everything, measure the approval/rejection rate per action type, and remove gates where the agent has proven reliable. This builds trust with data instead of assumptions, and it gives you a defensible story for why each remaining gate exists.

### Interview Follow-ups

- How do you avoid approval fatigue? (Gate by risk tier, not uniformly. Batch approvals. Auto-approve categories with a measured near-zero rejection rate. And show consequences clearly so review is fast.)
- What if the human never responds? (Timeout with a safe default — reject or escalate — plus a notification. Never leave an agent blocked indefinitely, and never default to approve.)

---

## Q16: How does streaming work in agents, and what should you stream?

### Answer

**Why it matters.** Agents are slow — 5 to 60 seconds is normal for multi-step tasks. Without streaming, the user stares at a spinner and assumes it is broken. Streaming does not make the agent faster; it makes the wait *legible*, and that is most of the perceived-quality battle.

**What you can stream, in increasing order of value:**

| Stream | Contains | Value |
|---|---|---|
| **Final answer tokens** | The response, token by token | Essential — fast TTFT |
| **Reasoning/thinking** | The model's intermediate reasoning | High — shows *why*, builds trust |
| **Tool calls** | "Searching the knowledge base for X…" | **Highest for agents** — shows progress |
| **Tool results** | Summaries of what came back | High — shows evidence being gathered |
| **Step/state updates** | "Step 3 of ~5", plan updates, todo progress | High for long tasks |
| **Custom progress events** | Domain-specific ("read 12 of 40 files") | High for bulk work |

**For agents specifically, streaming tool activity matters more than streaming tokens.** A user watching "Looking up order ORD-8842190… Checking shipment status… Reading the returns policy…" understands what is happening and will wait. A user watching a spinner for 20 seconds will not.

**Mechanics.** The model API streams server-sent events with incremental deltas. Your agent harness sits between the model and the client and must **re-emit** events, since the client is connected to your harness, not to the model API. Typical event types you forward:

```text
step_start          → which step, what the agent intends
text_delta          → assistant text tokens
thinking_delta      → reasoning tokens (if exposed)
tool_call_start     → tool name and arguments (redact sensitive args)
tool_call_end       → success/failure and a short result summary
state_update        → plan changes, todo items completed
interrupt           → human approval needed (pauses the stream)
done                → final result, usage, cost
```

**Implementation concerns:**
- **Buffering breaks streaming.** Proxies, load balancers, and frameworks will buffer SSE unless configured not to. This is the most common cause of "streaming doesn't work in production but works locally."
- **Errors mid-stream.** You have already sent a partial response. You need an error event type and client handling for it — you cannot return a 500 after streaming has begun.
- **Cancellation.** Propagate client disconnects so you stop paying for generation nobody will see.
- **Reconnection.** For long tasks, support resuming a stream (event ids and replay) or fall back to polling a task status endpoint.
- **Redaction in the stream.** Tool arguments may contain secrets or PII. Filter before emitting — the stream is user-visible output and needs the same output guardrails as the final answer.
- **Interrupts pause the stream** rather than ending it (see Q15).

**Where it sits in the lifecycle:** streaming is a cross-cutting concern spanning steps 3–8. Every step should be able to emit progress, which means your harness needs an event bus rather than a return value.

### Interview Follow-ups

- Should you stream raw reasoning to end users? (Sometimes. It builds trust and fills the wait, but it can be verbose, confusing, or expose internal considerations you would rather not show. A common compromise is a summarised or collapsed view.)
- How do you handle streaming with output guardrails that need the whole response? (Stream optimistically and be prepared to retract, or buffer sentence-by-sentence and validate per sentence. Full-response validation is fundamentally incompatible with token streaming — pick your trade-off deliberately.)

---

## Q17: What is middleware in an agent, and what does it enable?

### Answer

**Middleware wraps the agent loop, intercepting the flow at defined points** so cross-cutting concerns live in one place instead of being scattered through agent logic.

```text
Request
  → [logging middleware]
      → [auth middleware]
          → [rate-limit middleware]
              → [guardrail middleware]
                  → AGENT LOOP (steps 3-7)
              ← [output filter middleware]
          ← [cost tracking middleware]
      ← [tracing middleware]
  ← Response
```

**What middleware typically handles:**

| Concern | What it does |
|---|---|
| **Authentication/authorisation** | Resolve the caller; attach permission scopes used by tool authorisation |
| **Logging and tracing** | Structured logs and spans for every step, tool call, and model call |
| **Cost and token accounting** | Accumulate usage; enforce budgets; attribute cost per tenant |
| **Rate limiting** | Per user, per tenant, per tool |
| **Guardrails** | Input screening, output filtering (Q14) |
| **Context management** | Compress or trim history before each model call; inject memory |
| **Caching** | Prompt cache configuration; response caching for repeated calls |
| **Retry and fallback** | Model fallback, provider failover |
| **Error normalisation** | Convert provider-specific errors into a common shape |
| **Human-in-the-loop** | Detect actions requiring approval and interrupt |

**Why middleware rather than inline code.** These concerns apply at *every* step of *every* agent. Inlining them means duplicating them in each agent, forgetting one somewhere, and being unable to change policy centrally. Middleware makes them:
- **Composable** — stack them in a defined order
- **Reusable** — one implementation across all agents
- **Testable** — unit-test the concern independently of agent behaviour
- **Auditable** — a single place to verify that authorisation is applied

**Where middleware differs from hooks (Q18):** middleware typically wraps the whole request and can be *pass-through* (it sees and may transform both directions), while hooks fire at specific lifecycle events. In practice frameworks blur the line, and middleware is often *implemented* as a bundle of hooks. The useful distinction:
- Middleware = a **layer** the flow passes through, with before-and-after symmetry
- Hooks = **event callbacks** at named points

**Ordering matters, and it is a common bug.** Authentication must precede authorisation. Rate limiting should precede expensive work. Cost accounting must wrap the model call to see usage. Guardrail output filtering must be the outermost layer so nothing bypasses it. Get the order wrong and you have a guardrail that does not guard.

**Practical advice:** implement middleware for authentication, tracing, cost accounting, and guardrails on day one — they are painful to retrofit because they need to touch every path. Add the rest as needed.

### Interview Follow-ups

- Where does context compression belong? (Middleware just before the model call — it needs to see the assembled context and transform it, which is exactly the middleware shape. Putting it in agent logic means every agent reimplements it.)
- How do you prevent middleware from being bypassed? (Make the agent only constructible through a factory that installs the required middleware, and test that a directly-constructed agent fails. The same argument as the tenant-scoped retriever in `10-rag.md` Q22.)

---

## Q18: What are before-tool and after-tool hooks, and what do you put in them?

### Answer

**Hooks are callbacks that fire at named points in the lifecycle.** The two most important bracket tool execution — step 5 and step 7 of Q3 — because that is where the agent touches the real world.

**Before-tool hook (pre-execution).** Fires after the model has requested a tool call, before it runs. It can **allow, modify, or block** the call.

What belongs here:

| Purpose | Example |
|---|---|
| **Authorisation** | Does this user's role permit `issue_refund`? Reject if not. |
| **Argument validation** | Amount within limits; date well-formed; id matches the expected pattern |
| **Blast-radius checks** | `DELETE` with no `WHERE` clause → block. Bulk operation over N records → require approval. |
| **Human approval interrupt** | Pause and request confirmation for irreversible actions |
| **Argument injection** | Add `tenant_id` from the session so the model cannot set it |
| **Argument redaction/normalisation** | Trim, canonicalise, resolve aliases |
| **Rate limiting and quotas** | Per-tool, per-user caps |
| **Logging and tracing** | Record intent before execution, so a crash mid-call is still visible |
| **Cost pre-check** | Would this exceed the request budget? |
| **Caching** | Return a cached result without executing |

**Injecting arguments in the before-hook is an underrated pattern.** Never let the model supply `tenant_id`, `user_id`, or environment identifiers — inject them from the authenticated session. This makes cross-tenant access structurally impossible rather than prompt-dependent.

**After-tool hook (post-execution).** Fires once the tool returns, before the result enters the model's context. It can **transform or replace** the result.

What belongs here:

| Purpose | Example |
|---|---|
| **Secret redaction** | Strip API keys, tokens, and credentials from output |
| **PII filtering** | Redact personal data the agent should not see or echo |
| **Size capping / truncation** | 500 rows → top 20 plus "showing 20 of 500; refine the query" |
| **Result summarisation** | Compress a huge payload to protect the context window |
| **Injection screening** | Retrieved content may contain instructions — neutralise or flag them |
| **Error normalisation** | Convert a raw stack trace into a model-actionable message (Q13) |
| **Schema validation** | Confirm the tool returned the shape you expect |
| **Memory writes** | Persist facts learned from the result |
| **Metrics** | Latency, result size, success/failure per tool |
| **Termination checks** | Budget exhausted? Repeated identical call? Break the loop. |

**Why these two hooks matter more than any prompt instruction.** They are the only points where you have *deterministic control* over the agent's interaction with the world. Everything in the prompt is advisory; everything in these hooks is enforced. When an interviewer asks "how do you make an agent safe," the answer lives here.

### Example

```python
DESTRUCTIVE = {"delete_records", "issue_refund", "send_email", "deploy"}

def before_tool(ctx, tool_name, args):
    # 1. Authorisation — hard boundary
    if tool_name not in ctx.permitted_tools:
        return Block(f"You do not have permission to use {tool_name}.")

    # 2. Inject session-derived arguments — the model must not control these
    args = {**args, "tenant_id": ctx.session.tenant_id}

    # 3. Blast-radius check
    if tool_name == "delete_records" and not args.get("filter"):
        return Block("Refusing an unfiltered delete. Provide a filter.")

    # 4. Human approval for irreversible actions
    if tool_name in DESTRUCTIVE:
        return Interrupt(reason=f"Approve {tool_name}?", payload=args)

    # 5. Budget
    if ctx.cost_so_far > ctx.cost_limit:
        return Block("Request budget exhausted. Summarise progress and stop.")

    ctx.trace.tool_intent(tool_name, redact(args))
    return Allow(args)


def after_tool(ctx, tool_name, args, result):
    result = redact_secrets(result)

    if len(result) > MAX_TOOL_RESULT_TOKENS:
        result = truncate_with_notice(result, MAX_TOOL_RESULT_TOKENS)

    if contains_injection_patterns(result):
        result = wrap_as_untrusted_data(result)

    ctx.metrics.record(tool_name, latency=ctx.last_latency, size=len(result))

    key = (tool_name, canonical(args))
    ctx.call_counts[key] += 1
    if ctx.call_counts[key] >= 3:
        return Terminate("Repeated identical tool call — stopping to avoid a loop.")

    return result
```

### Interview Follow-ups

- Why inject `tenant_id` rather than validate it? (Validation requires the model to supply it correctly and you to check every path. Injection makes the wrong value unrepresentable — strictly safer.)
- What hooks exist besides these two? (Typically before/after model call, on agent start/end, on error, on state update, and on interrupt/resume. The tool hooks are the highest-value ones because tools are where side effects happen.)

---

## Advanced

---

## Q19: What is context engineering, and why is it the central skill in agent development?

### Answer

**Context engineering is deciding what goes into the model's context window at each step of the agent loop — and what does not.**

It supersedes "prompt engineering" for agents because in an agent the context is not a fixed prompt you author once. It is **assembled dynamically at every step** from many competing sources:

```text
System prompt        role, instructions, constraints
Tool schemas         one per available tool
Skills               loaded procedural instructions
Long-term memory     retrieved user facts and preferences
Retrieved documents  RAG context
Conversation history all prior turns
Tool results         every observation so far
Current state        plan, todo list, findings
```

All of these compete for a finite window, and every token you spend on one is a token unavailable to another.

**Why it is the central skill.** The most common cause of agent failure past step 5 is not a bad model or a bad prompt — it is a context that has become bloated, stale, or unfocused. The agent forgets its goal, re-reads what it already read, or drowns in a 40,000-token tool result. All of these are context-management failures.

**The four operations of context engineering:**

| Operation | What it means | Techniques |
|---|---|---|
| **Write** | Persist information outside the context | Scratchpad files, state objects, memory stores |
| **Select** | Choose what to bring in | Retrieval, memory relevance, loading only relevant tool schemas and skills |
| **Compress** | Reduce what is already there | Summarise old turns, truncate tool results, drop superseded steps |
| **Isolate** | Keep contexts separate | Sub-agents, per-task sessions, sandboxed environments |

**Concrete practices, in order of impact:**

1. **Cap tool output at the source.** The single biggest lever. A tool that can return 50,000 tokens will eventually ruin an agent run.
2. **Order context stable → volatile** so prompt caching covers the maximum prefix. This is a cost lever as much as a quality one.
3. **Externalise long-lived state.** A todo file or findings document the agent reads and updates survives compression and keeps the goal salient. This is why "write the plan to a file" works so well in practice.
4. **Compress with intent.** When summarising history, preserve *what was tried and what the outcome was* — the agent must not repeat failed attempts. Dropping that is the classic compression bug.
5. **Load skills and tool schemas on demand** rather than all upfront.
6. **Isolate expensive exploration in sub-agents** so their context cost does not accumulate in the main loop.
7. **Restate the objective** near the end of a long context — recency helps, and the original instruction is otherwise buried.
8. **Remove, do not accumulate.** Superseded plans, stale retrievals, and resolved errors should leave the context.

**The mental model to give in an interview:** treat the context window like RAM in a memory-constrained system. You have a fixed budget, you page things in and out deliberately, you keep hot data resident and cold data on disk, and you compact when fragmentation grows. The agent's *effective* intelligence at step 20 is determined almost entirely by how well you managed that budget for the previous 19 steps.

### Interview Follow-ups

- What is the difference between context engineering and prompt engineering? (Prompt engineering optimises a mostly-static instruction. Context engineering manages a dynamic, growing, multi-source information budget across a loop. The second subsumes the first for agents.)
- What is the most common context-engineering mistake? (Uncapped tool results, followed by summarising away the record of failed attempts — which causes the agent to repeat them.)

---

## Q20: How do you evaluate an agent?

### Answer

**Agents are much harder to evaluate than single LLM calls** because the output is a *trajectory*, not a string: many valid paths exist, cost and latency vary per run, and failures can occur at any step.

**The metric layers:**

| Layer | Metrics | What it tells you |
|---|---|---|
| **Outcome** | Task success rate, final answer correctness | Does it work? |
| **Trajectory** | Tool selection accuracy, unnecessary steps, redundant calls, recovery rate | *How* it works |
| **Efficiency** | Steps per task, tokens per task, cost per task, wall-clock latency | What it costs |
| **Reliability** | Loop rate, timeout rate, error rate, budget-exhaustion rate | Will it hold up in production |
| **Safety** | Guardrail trigger rate, unauthorised attempt rate, unapproved-action rate | Is it safe |
| **User** | Satisfaction, escalation rate, task abandonment, repeat-attempt rate | Does it help |

**Outcome evaluation — the primary signal.** Define task success **programmatically wherever possible**:

```python
# Best: verifiable end state
def check(task, final_state):
    return final_state.db.get_order(task.order_id).status == "refunded"

# Good: exact/structured match against expected output
# Acceptable: LLM judge against a rubric (calibrate it against human labels)
# Weak: human eyeballing (necessary, but does not scale or regress-test)
```

**Prefer verifiable end-state checks over judged text.** For an agent that acts on the world, "did the right thing happen?" is a far stronger signal than "does the answer read well?" This is the agent-specific version of preferring grounded reflection to self-critique (Q9).

**Trajectory evaluation matters because outcome alone hides problems.** Two runs both succeed: one takes 3 steps and $0.02, the other takes 22 steps, calls the wrong tool 6 times, and costs $0.90. Same outcome score, wildly different systems. Track:
- Did it call the tools a competent human would have?
- How many steps beyond the minimum?
- Did it repeat any call?
- When a tool failed, did it recover or give up?

**Building the eval set:**
- **20–100 tasks** spanning easy/typical/hard, plus known past failures.
- Include **unhappy paths**: tools that error, ambiguous requests, requests that should be refused, and requests requiring escalation.
- Include **adversarial cases**: prompt injection in tool results, attempts to exceed authorisation.
- Include tasks the agent **should not attempt** — measuring correct refusal is as important as measuring success.
- **Version the eval set** and run it in CI.

**Practical realities:**
- **Non-determinism.** Run each task 3–5 times and report the distribution, not a single number. A "90% success rate" from one run per task is noise.
- **Sandboxing.** Agents take real actions. Evaluation needs mock or sandboxed tools with a resettable state — this is real engineering work and is usually the biggest cost of an agent eval harness.
- **Cost.** A 50-task eval at 15 steps each is 750 LLM calls per run, times repeats. Budget for it, and keep a fast smoke-test subset for every commit with the full suite on merge.
- **Trace everything.** When a task fails you must be able to read the full trajectory. Aggregate metrics tell you *that* it failed; only the trace tells you *why*.

**The discipline that matters most:** every production failure becomes a permanent eval case. Agent quality improves through accumulated regression tests far more reliably than through prompt iteration.

### Interview Follow-ups

- How do you evaluate an agent whose task has no single right answer? (Rubric-based LLM judging on the *trajectory* plus outcome constraints — "did it consult the required sources", "did it avoid forbidden actions" — rather than comparing to a reference answer.)
- What is the most useful single metric? (Task success rate on a versioned eval set with repeats. Everything else is diagnostic; that one is the headline.)

---

## Q21: What are the main agent failure modes in production?

### Answer

| Failure | Symptom | Root cause | Mitigation |
|---|---|---|---|
| **Infinite / repeated loops** | Same tool, same args, forever | No progress detection; tool returns unhelpful results | Repeat detection, step cap, no-progress cap (Q13) |
| **Goal drift** | Ends up solving a different problem | Original instruction buried in a long context | Restate the objective; externalise the goal to a todo file (Q19) |
| **Wrong tool selection** | Uses `search_docs` for order status | Overlapping or vague tool descriptions | Fix schemas; reduce tool count; route (Q5) |
| **Hallucinated tool results** | Claims a tool returned data it did not | Model continues without executing; malformed loop | Validate every claimed action against the trace |
| **False success reporting** | "I've cancelled your order" — it did not | No verification of side effects | Verify end state before reporting success (Q13) |
| **Context exhaustion** | Hard failure mid-task | Uncapped tool output | Cap results; compress; sub-agents (Q6) |
| **Cost blowout** | One task costs $40 | No budget enforcement; loops | Hard token/cost caps per request |
| **Cascading errors** | One bad early step corrupts everything after | No intermediate verification | Verify critical intermediate results |
| **Prompt injection via tool results** | Agent follows instructions from a retrieved document | Untrusted content treated as instructions | Delimit and mark as data; break the exfiltration path (Q14) |
| **Over-permissioned action** | Deletes production data | Tool granted without scoping | Deny by default; blast-radius checks (Q18) |
| **Silent degradation** | Quality drops after a model or data change | No continuous evaluation | Eval set in CI; production monitoring |
| **Latency unpredictability** | p50 4 s, p99 90 s | Variable step counts | Step caps, deadlines, streaming, timeouts |
| **Approval fatigue** | Humans rubber-stamp everything | Uniform gating | Risk-tiered approval (Q15) |
| **Multi-agent coordination failure** | Duplicated or contradictory work | Lossy handoffs; no shared state | Fewer agents; explicit contracts (Q12) |

**The three that cause the most production incidents:**

1. **False success reporting.** An agent that says it did something it did not is uniquely damaging because it destroys user trust irrecoverably and the error is invisible until consequences surface. **Always verify the end state** of a side-effecting operation before reporting success — read back the record, check the status code, confirm the row changed.

2. **Cost blowout from loops.** Without a hard cap, one pathological request can cost hundreds of dollars. Enforce token and cost budgets in the loop (step 5/7), not just in monitoring — monitoring tells you after you have paid.

3. **Prompt injection through tool results.** The agent reads a document, web page, or ticket containing instructions and follows them. This is the highest-severity *security* failure and it is design-preventable: never combine untrusted content with private data access and an outbound channel.

**The general defensive posture:**

```text
Hard caps       steps, tokens, cost, wall-clock deadline
Verification    confirm side effects actually happened
Detection       repeated calls, no progress, budget approach
Least privilege minimum tools, narrowest scopes, read/write split
Observability   full traces; you cannot debug what you cannot see
Graceful exit   partial results with an explicit statement of what is incomplete
```

**The framing worth stating:** agents fail *differently* from traditional software — not by crashing but by confidently doing the wrong thing. So the engineering emphasis shifts from exception handling to **verification and bounded authority**. Assume the agent will occasionally be wrong, and design so that being wrong is detectable and cheap.

### Interview Follow-ups

- How do you detect false success reporting in production? (Reconcile the agent's claimed actions against the actual audit log of tool executions. Any claimed action with no corresponding trace entry is a bug — and this check can run automatically.)
- What is your first move when an agent starts failing after a model upgrade? (Run the versioned eval set to localise the regression, then diff trajectories between model versions on failing tasks. Tool-selection behaviour is the most common thing to shift.)

---

## Q22: How do you design an agent for reliability rather than capability?

### Answer

**The core trade-off:** every degree of freedom you give an agent adds capability and subtracts predictability. Production engineering is mostly about removing freedom the task does not need.

**The reliability hierarchy — always prefer the option higher on this list:**

```text
1. No LLM at all           deterministic code                    100% reliable
2. Single LLM call         classification, extraction, generation  very reliable
3. Fixed chain             predetermined steps with LLM stages     reliable
4. Constrained agent       few tools, low step cap, gated actions  workable
5. Open agent              many tools, unbounded loop              hardest
```

Most "agent" problems are actually level 2 or 3 problems. Ask: *does this task genuinely require runtime decisions about control flow?* If the sequence of steps is knowable in advance, write the chain.

**Techniques that buy reliability:**

**1. Constrain the action space.** Fewer tools, narrower scopes, explicit enums instead of free-form arguments. Every tool you remove eliminates a class of failure.

**2. Make the happy path deterministic.** Handle the common 80% with a fixed chain and escalate only the remainder to an agent. This is the highest-value architectural move available: it makes the majority of traffic predictable, cheap, and fast, while retaining agentic capability for the hard tail.

**3. Verify rather than trust.** Check side effects happened. Validate outputs against schemas. Use grounded feedback (tests, linters, validators) instead of self-assessment.

**4. Bound everything.** Steps, tokens, cost, wall-clock time, retries per tool, total tool calls. Every bound is a failure mode you have converted from unbounded to graceful.

**5. Make failures explicit and recoverable.** Partial results with a clear statement of what is incomplete beat both a confident wrong answer and an opaque error. Persist state so a failed run can be resumed rather than restarted.

**6. Structure the output.** Force structured output for anything downstream code consumes. A schema is enforcement; a formatting instruction is a hope.

**7. Gate consequential actions with humans** until you have measurement justifying autonomy (Q15).

**8. Idempotency everywhere.** Use idempotency keys on side-effecting tools so a retry cannot double-charge, double-send, or double-create. This makes automatic retry safe, which in turn makes the system far more robust.

**9. Instrument completely.** Traces, per-step metrics, cost attribution, and eval-set results in CI. You cannot maintain reliability you cannot measure.

**10. Design for the failure being visible to the user.** Citations, shown sources, previews of actions before execution, and undo where possible. If the agent is wrong, the user should be able to tell — that is a much more achievable property than never being wrong.

**The single most important piece of advice:** start with the *least* agentic architecture that could work, and add autonomy only where measurement shows it is necessary. Teams consistently build level 5 when level 3 would have worked, then spend months fighting reliability problems they created for themselves.

### Interview Follow-ups

- How do you decide whether a task needs an agent? (Ask whether the *sequence of steps* varies by input in ways you cannot enumerate. If it varies, you need an agent. If you can enumerate it, write the chain.)
- What does "progressive autonomy" look like concretely? (Ship with human approval on all writes; log approval and rejection rates per action type; remove gates where rejection is near-zero over a meaningful sample; keep gates where it is not. Autonomy earned by data.)

---

## Q23: How do agents interact with RAG?

### Answer

Three levels, in increasing order of agency:

**Level 1 — retrieval as a fixed step.** Not agentic: retrieve, then generate. Predictable and cheap. Covered in `10-rag.md`.

**Level 2 — retrieval as a tool.** The agent *decides* whether and what to retrieve.

```python
{
    "name": "search_knowledge_base",
    "description": (
        "Search internal documentation for information. Use this whenever the "
        "question concerns company policies, product behaviour, or procedures. "
        "Do not use it for the user's own account data (use get_account) or for "
        "general knowledge you already have."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "query": {"type": "string", "description": "Focused search query, not the raw user question"},
            "source": {"type": "string", "enum": ["docs", "policies", "tickets", "all"]},
        },
        "required": ["query"],
    },
}
```

This alone unlocks meaningful capability: the agent can skip retrieval for chit-chat, reformulate after poor results, retrieve multiple times for multi-hop questions, and combine retrieval with other tools.

**Level 3 — full agentic RAG.** The agent iterates: retrieve, assess sufficiency, reformulate, retrieve again, cross-check across sources, and decide when it has enough (see `10-rag.md` Q20).

**What agency adds over a fixed pipeline:**

| Capability | Why a pipeline cannot do it |
|---|---|
| Skip unnecessary retrieval | The pipeline always retrieves |
| Reformulate after failure | No feedback loop exists |
| **Multi-hop** | Hop 2's query depends on hop 1's *result* |
| Choose among sources | Requires a runtime decision |
| Combine retrieval with SQL, APIs, computation | Requires arbitrary tool composition |
| Verify a claim against a second source | Requires a second deliberate retrieval |

**The critical design points when retrieval is a tool:**

1. **The tool must return a bounded, summarised result.** Retrieval can produce enormous output. Cap it in the after-tool hook: top 5 chunks, truncated, with source ids. Uncapped retrieval is the fastest way to exhaust an agent's context (Q6).

2. **Keep citations attached through the loop.** The agent must be able to cite sources in the final answer, so chunk ids and titles must survive every intermediate step. Losing provenance mid-loop is a common bug.

3. **ACL filters are injected, never model-supplied.** The before-tool hook adds `tenant_id` and the user's permission groups from the session (Q18). The model must not be able to influence them.

4. **Retrieved content is untrusted input.** It may contain injected instructions. Delimit it, mark it as data, and screen it in the after-tool hook (Q14).

5. **Cache retrieval results within a task.** Agents frequently re-issue similar queries; caching by normalised query saves real cost and latency.

6. **Give the retrieval tool a good description of *when* to use it.** This is the dominant factor in whether the agent retrieves appropriately — more so than any retrieval-quality improvement.

**The architectural recommendation:** default to **level 2**. It captures most of the benefit — conditional retrieval, reformulation, multi-source — at close to pipeline cost, because most queries still resolve in one retrieval. Escalate to level 3 only for query classes you have identified as genuinely multi-hop, and route them there rather than paying agentic cost on every request.

### Interview Follow-ups

- Why does making retrieval a tool sometimes *reduce* quality? (The agent may skip retrieval when it should have retrieved, answering from parametric knowledge instead. Fix with a strong tool description, and instruct explicitly that company-specific questions always require retrieval.)
- How do you keep an agentic RAG loop from re-retrieving the same thing? (Cache by normalised query within the task, and include a short summary of "what you have already searched for" in the context so the model can see it.)

---

## Q24: How do you handle authentication and authorisation for agent tools?

### Answer

**The core principle: the agent acts on behalf of a user and must never have more authority than that user.**

**The wrong architecture, and why it is common.** The agent holds a service account with broad permissions and "decides" what it should access based on prompt instructions. This fails because the model's decisions are probabilistic and manipulable, and because a single prompt injection now has admin rights.

**The right architecture: authority flows from the authenticated session, and is enforced in code.**

```text
User authenticates → session carries identity + roles + scopes
                   ↓
Agent is constructed with a permission context derived from the session
                   ↓
Tool registry exposes ONLY tools this user may use          (step 1)
                   ↓
Before-tool hook authorises THIS call with THESE arguments  (step 5)
                   ↓
Tool executes with the USER's credentials, not a service account
```

**The layers, and what each catches:**

| Layer | Enforcement |
|---|---|
| **Tool registration** | The user cannot invoke a tool that was never registered for them. Cheapest and strongest. |
| **Before-tool authorisation** | Role/scope check for this specific call |
| **Argument injection** | `tenant_id`, `user_id`, and scope come from the session — never from the model (Q18) |
| **Downstream enforcement** | The API or database independently enforces permissions using the user's token |
| **Audit log** | Who did what, via which agent, when |

**Defence in depth matters here.** Do not rely solely on the before-tool hook — the downstream system should independently reject an unauthorised request. Two independent enforcement points mean a bug in one does not become a breach.

**Delegated credentials.** Prefer passing the user's own token (OAuth on-behalf-of, or a short-lived scoped token) to downstream services over using a service account. Then downstream authorisation happens naturally and correctly, and the audit trail in each system shows the real actor.

**Practical concerns:**

- **Token expiry mid-task.** Agent tasks can run longer than an access token's lifetime. Handle refresh in the tool layer, and for HITL flows that may pause for hours, re-authorise on resume rather than persisting a live token.
- **Scope minimisation.** Request the narrowest scopes the task requires, not the union of everything the agent might ever do.
- **Sub-agent inheritance.** A sub-agent must inherit a *subset* of the parent's permissions, never a superset. A read-only research sub-agent is a good pattern precisely because its authority is structurally smaller.
- **Tool-level vs record-level.** "Can call `get_order`" is not "can see order 8842190." Record-level authorisation must be enforced in the tool or downstream, using the session identity.
- **Non-human callers.** Scheduled or system-triggered agents have no user session. Give them a dedicated principal with minimal scopes and a distinct audit identity — never reuse a human's credentials.
- **Secrets never enter the context.** The agent should never see API keys or tokens. Tools hold credentials; the model holds only the intent. If a credential appears in a tool result, redact it in the after-tool hook.

**The interview-strong summary:** authorisation for agents is not a new problem — it is standard least-privilege access control, with the extra requirement that the *decision-maker is untrusted*. So every authority decision must be made in code from session-derived identity, and the model must be treated as a potentially-compromised client.

### Interview Follow-ups

- Why not let the model pass the user id? (Because a prompt injection can change it. Any security-relevant value the model controls is an attack surface; inject it instead.)
- How do you audit agent actions for compliance? (Log every tool call with the resolved principal, arguments (redacted), result status, and the agent/session id — and reconcile the agent's claimed actions against that log to catch false success reporting.)

---

## Q25: What is the difference between an agent and a workflow, and how do you choose?

### Answer

| | Workflow (chain) | Agent |
|---|---|---|
| Control flow | **Written by you** | **Decided by the model at runtime** |
| Steps | Fixed, known in advance | Variable, unknown until it runs |
| Cost per run | Predictable | Variable, 3–50× |
| Latency | Predictable | Variable |
| Testing | Straightforward — deterministic paths | Hard — trajectory space is large |
| Debugging | Read the code | Read the trace |
| Failure mode | A known step errors | Anything, anywhere |
| Handles novelty | No | **Yes** |
| Recovers from failure | Only if coded | **Yes — can observe and adapt** |

**The decision question is exactly one thing: is the sequence of steps knowable in advance?**

```text
Can you enumerate the steps for every input?
├── YES → Write a workflow. It will be cheaper, faster, more testable, and more reliable.
└── NO  → Does the variation come from a small set of cases?
          ├── YES → Workflow with branches, or a router over workflows.
          └── NO  → Agent.
```

**Examples that are workflows, not agents:**
- "Summarise this document" → one call
- "Classify the ticket, then route it" → classify → route
- "Retrieve, then answer with citations" → the standard RAG chain
- "Extract fields from this invoice and validate them" → extract → validate → maybe retry
- "Translate, then check the translation quality" → two calls

**Examples that genuinely need an agent:**
- "Find out why the nightly job failed" → the investigation path depends on what each log reveals
- "Fix this failing test" → read, hypothesise, edit, run, interpret, repeat
- "Research this topic and write a briefing" → what to read next depends on what you learned
- "Resolve this customer's issue" → could be billing, technical, or shipping, discovered mid-conversation

**Hybrid is usually the right answer, and this is the mature response.** Most real systems are a workflow *containing* agentic steps, or a router that sends most traffic to a deterministic path and the hard tail to an agent:

```text
Ticket arrives
  → classify (single LLM call)
  → if simple FAQ:      retrieve + answer         (workflow, 1s, $0.01)
  → if account query:   fetch + template          (workflow, 0.5s, $0.005)
  → if complex issue:   agent with 6 tools        (agent, 15s, $0.30)
  → if ambiguous:       ask a clarifying question (workflow)
```

This gives you predictable cost and latency for the majority of traffic and agentic capability where it is actually needed. It is also far easier to operate, because most requests never touch the unpredictable path.

**The recommendation to state plainly:** default to workflows. Introduce agency at the specific point where you have identified that control flow genuinely cannot be predetermined. Agents are a tool for handling unpredictability, not a general architecture — and the industry's most common mistake in 2024–2025 was building agents for problems that were fully specified in advance.

### Interview Follow-ups

- How do you migrate from an agent to a workflow? (Analyse production traces: if 85% of runs follow the same 4-step trajectory, encode that as a workflow and keep the agent as the fallback for the rest. Trace analysis is how you discover the workflow hiding inside your agent.)
- Is "agentic workflow" a meaningful term? (Yes, for the hybrid: a workflow whose individual steps may be agentic, or an agent constrained to a mostly-fixed structure. Most production systems land here.)

---

## Q26: How do you optimise agent latency and cost?

### Answer

**Understand the shape of the problem first: an agent's cost is roughly `steps × context_size`, and both grow during a run.** A 15-step task averaging 40k tokens of context is 600k input tokens — not 40k. That quadratic-ish growth is why agents are expensive and why context management is the primary lever.

**Cost levers, by impact:**

| Lever | Typical saving | Notes |
|---|---|---|
| **Prompt caching** | 50–90% on the cached prefix | Order context stable→volatile; the single biggest win for agents |
| **Reduce step count** | Linear | Better tool design, better instructions, parallel tool calls |
| **Cap tool output** | Large and compounding | Every token persists for the rest of the run |
| **Route by complexity** | 5–20× on simple traffic | Cheap model / fixed workflow for the easy majority |
| **Smaller model for sub-tasks** | 5–20× on those calls | Classification, extraction, summarisation rarely need a frontier model |
| **Compress history** | 30–60% on long runs | Preserve what was tried and its outcome |
| **Sub-agents for context isolation** | Large on exploration-heavy tasks | The sub-agent's context does not accumulate in the parent |
| **Cap output tokens** | Moderate | Output is priced higher than input |
| **Deduplicate tool calls** | Varies | Cache by (tool, normalised args) within a task |

**Latency levers, by impact:**

| Lever | Effect |
|---|---|
| **Streaming (especially tool activity)** | Transforms perceived latency; do this first (Q16) |
| **Parallel tool calls** | Often the largest real reduction — independent calls should never be sequential |
| **Parallel sub-agents** | Wall-clock reduction proportional to the fan-out |
| **Fewer steps** | Each step is a full model round trip, seconds each |
| **Smaller/faster model** | 2–5× per call |
| **Prompt caching** | Cuts prefill time as well as cost |
| **Speculative prefetch** | Start a likely retrieval before the model asks; discard if unused |
| **Shorter context** | Prefill scales with input length |

**The step count is the dominant term for both.** Reducing 12 steps to 6 halves cost and latency simultaneously. How to reduce steps:
- **Better tool granularity.** If the agent always calls three tools in sequence, provide one composite tool.
- **Parallel tool calls.** Encourage the model to batch independent lookups.
- **Better instructions.** Much wandering comes from an unclear objective or missing context that the agent has to discover.
- **Provide context upfront.** If you know the user's account id, put it in the prompt rather than making the agent look it up.
- **Plan first** for long tasks, so the agent does not rediscover the route each step.

**What not to sacrifice.** Do not remove guardrails, verification, or tracing to save latency. They cost tens of milliseconds and prevent the failures that cost far more. And do not downgrade the model on the *decision-making* calls — a cheap model that picks the wrong tool adds steps and ends up more expensive than the model you were avoiding.

**Measure before optimising.** Instrument per-step token counts, per-tool latency, and cost per task, broken down by task type. The distribution is almost always dominated by a small number of task types with runaway step counts — fix those rather than shaving milliseconds off the common path.

### Interview Follow-ups

- Why does prompt caching matter so much more for agents than for single calls? (An agent resends the entire history every step. With caching, the growing stable prefix is charged at a fraction of the price on every subsequent step — the saving compounds with step count.)
- What is the trap in using a cheaper model to save cost? (Poorer tool selection and reasoning increases step count, and each extra step resends the whole context. Total cost can rise even though per-call cost fell. Measure cost per *completed task*, not per call.)

---

## Q27: How do you handle multi-turn conversations with an agent?

### Answer

**The added difficulty over single-turn:** the agent must track what has already been established, resolve references to prior turns, decide what to carry forward, and handle the user changing their mind mid-task.

**What must persist across turns:**

| State | Why |
|---|---|
| Message history | Reference resolution, context |
| Tool results from earlier turns | Avoid re-fetching; the user may refer to them |
| Established facts and decisions | "the order we discussed", "use the second option" |
| Pending clarifications | The agent asked something; the answer is in this turn |
| Task progress | Multi-turn tasks that span several exchanges |
| User preferences learned in-session | Format, verbosity, tone |

**The key practices:**

**1. Query rewriting / decontextualisation** for any retrieval or search. "What about the second one?" is meaningless to a search index. Resolve references before retrieving (see `10-rag.md` Q8). This is mandatory, not optional.

**2. Distinguish conversation history from retrieved evidence** in the context. If they are interleaved, the model conflates what the *user said* with what a *document said* — a subtle and damaging failure. Use clear delimiters and separate sections.

**3. Handle topic changes.** A new turn may abandon the previous task. Detect it and reset working state rather than continuing to reason about the old goal. Carrying stale task state into a new topic is a common cause of confusing behaviour.

**4. Manage growth.** Multi-turn plus multi-step means context grows on two axes. Summarise older turns while preserving decisions, established facts, and what has been tried (Q19).

**5. Persist state durably.** Turns may be minutes or days apart, and requests may land on different processes. State must live in a store keyed by conversation id, not in process memory. This is the same requirement as HITL (Q15), and it is why checkpointing is foundational.

**6. Handle in-flight interruption.** The user may send a new message while the agent is mid-loop. Decide the policy explicitly: queue it, cancel and restart with the new information, or inject it as an observation. Silently ignoring it is the worst option, and it is the default if you do not decide.

**7. Re-authorise on resume.** Permissions and tokens change between turns. Do not trust a permission context captured at turn one for a turn arriving three days later (Q24).

**8. Re-check memory relevance per turn.** What was worth injecting in turn one may be irrelevant in turn five, and vice versa.

**The most common multi-turn bugs:**

| Bug | Cause |
|---|---|
| Retrieval fails on follow-ups | No query rewriting |
| Agent forgets an earlier decision | Summarisation dropped it |
| Agent repeats a tool call from an earlier turn | Tool results not carried forward |
| Agent continues the old task after a topic change | No topic-change detection |
| Context exhaustion at turn 12 | No compression strategy |
| Confuses user statements with document content | History and evidence interleaved |
| Stale permissions | Permission context cached across turns |

**Design summary:** treat a conversation as a **durable, evolving state object** — not an append-only message list. What you *store* (full history, tool results, decisions, progress) and what you *send to the model each turn* (a curated, compressed subset) are two different things, and conflating them is the root of most multi-turn problems.

### Interview Follow-ups

- How do you decide what to summarise versus keep verbatim? (Keep verbatim: the current task's state, recent turns, explicit user decisions and constraints, and the record of what has been tried. Summarise: resolved sub-tasks and old tool results. When in doubt, keep the record of attempts — repeating failed work is the worst outcome.)
- How do you handle a user correcting the agent mid-task? (Treat the correction as high-priority context, explicitly acknowledge it, discard the invalidated branch of work, and re-plan. Do not simply append it — the model may continue on the old plan.)

---

## Q28: What is the Model Context Protocol (MCP), and why does it matter?

### Answer

**MCP is an open protocol that standardises how applications provide tools, resources, and prompts to LLMs.** It defines a client-server interface so a tool implemented once can be used by any MCP-compatible host.

**The problem it solves — an N×M integration explosion.** Before a standard, every agent framework needed a bespoke integration for every tool, and every tool provider needed an integration for every framework. MCP turns N×M into N+M: tool providers implement a server; agent hosts implement a client.

**What an MCP server exposes:**

| Primitive | What it is | Analogy |
|---|---|---|
| **Tools** | Executable functions the model can call | Function calling |
| **Resources** | Readable data the host can supply as context | Files/URIs |
| **Prompts** | Reusable prompt templates the user can invoke | Slash commands |

**Architecture:**

```text
Agent host (Claude Code, an IDE, your application)
  └── MCP client
        ├── MCP server: GitHub      (tools: create_pr, list_issues; resources: repo files)
        ├── MCP server: Postgres    (tools: query; resources: schema)
        └── MCP server: your internal API
```

Transport is typically stdio for local servers or HTTP/SSE for remote ones. Servers can be written in any language.

**Why it matters practically:**
- **Reuse.** One integration serves every MCP-compatible client.
- **Ecosystem.** A large body of existing servers for common systems, so you do not write them.
- **Separation of concerns.** Tool implementation and credential handling live in the server, outside your agent code.
- **Dynamic discovery.** A host can enumerate a server's tools at connection time rather than hardcoding schemas.

**Where it sits in the lifecycle:** MCP is the *transport and discovery layer* for step 1 (tool registration) and step 6 (tool execution). It does not change the agent loop — the model still emits a tool call, your harness still executes it — it just standardises where the tool lives and how it is described.

**The security considerations, which are the important part in an interview:**

1. **An MCP server is code you are running.** A third-party server can do anything its process permits. Vet servers as you would any dependency, and prefer ones you or your organisation control for anything sensitive.
2. **Tool descriptions enter your prompt.** A malicious server can inject instructions via its tool descriptions — a supply-chain prompt-injection vector.
3. **Tool results are untrusted content.** Same as retrieved documents (Q14).
4. **Credentials live in the server**, which is good (they stay out of the model's context) but means the server is now a secret-holding component to secure.
5. **Aggregation increases blast radius.** Connecting many servers to one agent gives it broad authority across many systems — exactly the condition for the lethal trifecta. Scope deliberately.
6. **Your guardrails still apply.** Before/after-tool hooks (Q18) must wrap MCP tool calls just as they wrap local ones. Do not let the protocol boundary become a guardrail gap.

**Honest framing:** MCP is an integration and distribution standard, not an agent capability. It solves a real and painful plumbing problem, which is why adoption was rapid — but it does not change how agents reason, and it adds a supply-chain surface you must manage.

### Interview Follow-ups

- How does MCP differ from just calling an API? (It standardises *discovery and description* — the host learns what tools exist and how to describe them to the model at runtime. Calling an API still requires you to hand-write the schema and the invocation.)
- Would you connect an untrusted MCP server to an agent with production write access? (No. That combines untrusted content, private data access, and side-effecting authority. Isolate untrusted servers to read-only agents with no exfiltration path.)

---

## Q29: How do you debug an agent that is behaving incorrectly?

### Answer

**Agent bugs are usually several steps upstream of the symptom, so the method is to reconstruct the trajectory and find the first divergence.**

**Step 1 — get the full trace.** You need, for every step: the exact model input (fully rendered, not the template), the exact model output including reasoning if available, every tool call with arguments, every tool result verbatim, token counts, and timings. If you do not have this, stop and add it — debugging without traces is guessing.

**Step 2 — find the first step that went wrong.** Read forward, not backward. The final answer being wrong is usually a consequence; the interesting question is which step first deviated from what a competent human would have done.

**Step 3 — classify the divergence.** Each class has a different fix:

| Divergence | Diagnosis | Fix |
|---|---|---|
| Called the wrong tool | Tool descriptions overlap or are vague | Rewrite schemas; reduce tool count (Q5) |
| Right tool, wrong arguments | Parameter descriptions unclear; missing context | Add formats/examples/enums; provide context upfront |
| Tool returned an error and the agent gave up | Unhelpful error message | Make errors actionable (Q13) |
| Tool returned the right data, agent misread it | Result format confusing or too large | Reshape/summarise the tool output |
| Agent lost the goal | Context too long, objective buried | Restate the goal; externalise it; compress (Q19) |
| Agent repeated itself | No repeat detection; tool unhelpful | Add loop detection; fix the tool |
| Agent stopped early | Termination condition too eager or step cap hit | Adjust caps; clarify completion criteria |
| Agent claimed success falsely | No verification | Verify side effects (Q21) |
| Agent followed injected instructions | Untrusted content treated as instruction | Delimit; screen; break exfiltration (Q14) |

**Step 4 — reproduce in isolation.** Replay the exact context from the divergent step against the model directly. This tells you whether the problem is the *context you built* or the *model's judgement given that context*. That distinction determines whether you fix the harness or the prompt/model.

**Step 5 — check the boring things first.** In practice, most agent bugs are one of these:

1. **The rendered prompt is not what you think it is.** Print the fully-rendered final input. Missing variables, unescaped braces, and doubled system prompts are extremely common.
2. **A tool is silently returning something unexpected** — an empty list, a wrapped error, a truncated payload, a different schema than documented.
3. **The tool result is too large** and the important part was truncated away.
4. **A schema mismatch** between the declared tool schema and the actual function signature.
5. **Stale caching** — prompt caching or a response cache returning yesterday's answer.
6. **Non-determinism** — you are debugging a run that will not reproduce. Run it 5 times before concluding anything.

**Step 6 — turn it into a regression test.** Every fixed bug becomes a permanent eval case (Q20). Agents regress easily on model upgrades and prompt edits, and an accumulating eval suite is the only durable defence.

**Tooling that pays for itself:** a trace viewer showing the step-by-step trajectory with expandable inputs/outputs, the ability to replay a trace from any step, and diffing between two runs of the same task. These are the agent equivalent of a debugger, and building or adopting one early is worth more than any prompt-tuning effort.

### Interview Follow-ups

- Model output looks right but the outcome is wrong — where do you look? (The tool layer: argument marshalling, the actual side effect, and whether the result you fed back reflects what really happened. Also check for a silent exception swallowed into a success-looking result.)
- How do you debug a failure you cannot reproduce? (Increase trace fidelity and sampling in production, log the random seed/model version/prompt version, and run the task repeatedly to characterise the failure rate. Intermittent agent failures are usually a distribution over trajectories, not a deterministic bug.)

---

## Q30: Design a customer support agent. Walk through the architecture.

### Answer

**Clarify first:** volume and channels? What actions may it take autonomously (refunds, cancellations, address changes)? Escalation policy? Latency expectation? Regulatory constraints? Which systems must it integrate with? What is the cost per conversation target?

**Assume:** 50k conversations/month, chat + email, may issue refunds under $50 autonomously, must escalate anything legal or safety-related, sub-5-second first response, integrates with the order system, CRM, and knowledge base.

**Architecture:**

```text
Message arrives (chat/email)
  → AUTH: resolve customer identity → session with permission scopes
  → INPUT GUARDRAILS: PII detection, abuse detection, injection screening, rate limit
  → CLASSIFY (small model): intent + urgency + sentiment
  → ROUTE
      ├── FAQ/simple → RAG workflow (retrieve + answer + cite)     ~1s,  $0.01
      ├── Account query → fetch + template                          ~1s,  $0.01
      ├── Complex issue → AGENT (below)                            ~10s, $0.20
      ├── Legal/safety/abuse → immediate human escalation
      └── Ambiguous → clarifying question (workflow)

AGENT (constrained, 6 tools)
  Tools:
    search_knowledge_base(query, source)          read-only
    get_order(order_id)                           read-only, ACL-filtered
    get_customer_history(customer_id)             read-only, ACL-filtered
    issue_refund(order_id, amount, reason)        WRITE, gated
    update_shipping_address(order_id, address)    WRITE, gated
    escalate_to_human(reason, summary, priority)  always available

  Loop: max 8 steps, 30s deadline, 60k token budget
  Before-tool hook:
    - inject customer_id and tenant from session (never model-supplied)
    - issue_refund: amount ≤ $50 AND order belongs to this customer AND
      no refund already issued → else require human approval
    - update_shipping_address: block if the order has already shipped
  After-tool hook:
    - redact payment details and internal notes from results
    - cap knowledge-base results to 5 chunks with source ids
    - detect repeated identical calls → escalate
  Termination:
    - resolved, escalated, or budget exhausted → escalate with a summary

  → OUTPUT GUARDRAILS: PII, tone/policy check, citation validation, no promises
                       about dates or amounts not verified by a tool
  → STREAM response with tool-activity progress
  → PERSIST conversation state, memory, and full trace
  → LOG for evaluation; collect the customer's satisfaction signal
```

**Key design decisions and why:**

| Decision | Rationale |
|---|---|
| **Classify and route first** | Most support volume is simple. Paying agent cost on FAQs is the classic mistake — routing cuts average cost by an order of magnitude. |
| **Only 6 tools** | Tool selection accuracy degrades with count; support genuinely needs few. |
| **Read/write split with gates** | Reads are safe and unrestricted; writes are individually authorised and bounded. |
| **$50 refund cap in code, not prompt** | The prompt can be talked around; the hook cannot. This is the real boundary (Q14). |
| **`escalate_to_human` always available** | The agent must always have a safe exit. Never trap it into answering. |
| **Escalate on budget exhaustion** | Failing to a human is a good outcome; a wrong confident answer is not. |
| **Citation validation** | Support answers reference policy — wrong citations create liability. |
| **Output guardrail on promises** | "Your refund will arrive Tuesday" must be tool-verified, not generated. |
| **Full trace persistence** | Required for dispute resolution, quality review, and eval-set growth. |

**The hardest problems, stated honestly:**
1. **Knowing when to escalate.** Under-escalating frustrates customers and creates risk; over-escalating destroys the ROI. Tune the threshold on real data and monitor both directions.
2. **Tone and empathy** on an upset customer. This is a prompt and eval problem, and it needs human review in the loop — automated metrics will not catch it.
3. **Knowledge base staleness.** Support docs go out of date constantly; the agent will confidently cite an old policy (see `10-rag.md` Q23).
4. **Multi-turn state** across an email thread spanning days (Q27).
5. **Measuring success.** Resolution rate is gameable — an agent that says "resolved" is not. Use customer-confirmed resolution plus a repeat-contact rate.

**Rollout:** shadow mode first (the agent drafts, a human sends), then autonomous on the lowest-risk intent, expanding by measured category. Track resolution rate, escalation rate, customer satisfaction, cost per conversation, and guardrail trigger rates from day one.

### Interview Follow-ups

- Why not let the agent handle all refunds with a prompt limit? (Because the limit would be advisory. A code-enforced cap turns a policy into a guarantee, which is what makes autonomous refunds deployable at all.)
- How do you prevent the agent from making commitments it cannot keep? (Output guardrail: reject or rewrite any response containing a date, amount, or commitment that does not appear in a verified tool result. Combine with a prompt instruction, but enforce in code.)

---

## Q31: What is the difference between an agent, a chain, a tool, and a skill? Summarise the vocabulary.

### Answer

A consolidated reference for the terms that get conflated in interviews.

| Term | Definition | Key property |
|---|---|---|
| **LLM call** | One request/response | No loop, no tools |
| **Chain / workflow** | A fixed sequence of steps, possibly with LLM stages | **Control flow written by you** |
| **Agent** | An LLM in a loop with tools, deciding its own next action | **Control flow decided at runtime** |
| **Tool** | A function the model can request the execution of | Extends what the agent **can do** |
| **Skill** | A package of instructions for performing a task | Extends what the agent **knows how to do** |
| **Sub-agent** | An agent invoked as a tool by another agent | **Separate context window** |
| **Router** | Classifies a request and dispatches it once | **One decision**, no orchestration |
| **Supervisor** | Coordinates multiple agents across a task | **Many decisions**, control returns to it |
| **Hook** | A callback at a named lifecycle point | Enforcement point for guardrails |
| **Middleware** | A layer the request passes through, both directions | Cross-cutting concerns |
| **Guardrail** | An enforced constraint on behaviour | **Code, not prompt** |
| **Memory** | Information carried across steps or sessions | Working vs episodic vs semantic vs procedural |
| **Context engineering** | Managing what enters the context window each step | Write / select / compress / isolate |
| **ReAct** | Interleaved reasoning and acting | The canonical agent loop |
| **MCP** | A protocol standardising tool provision to hosts | Integration layer, not a capability |

**The distinctions most often confused, and the one-line discriminator for each:**

- **Chain vs agent:** who decided the sequence of steps — you, or the model at runtime?
- **Tool vs skill:** does it give a new *capability*, or new *know-how*?
- **Tool vs sub-agent:** does the task need *judgement and iteration*, or is it a deterministic operation?
- **Router vs supervisor:** does control come back for another decision?
- **Hook vs middleware:** a callback at a point, or a layer wrapping the flow?
- **Memory vs RAG:** written by the agent from experience, or curated externally?
- **Guardrail vs instruction:** enforced in code, or requested in the prompt?
- **Context engineering vs prompt engineering:** managing a dynamic multi-source budget across a loop, or optimising one static instruction?

**The single most useful framing to carry into an interview:** an agent is a **loop** (ReAct) with **capabilities** (tools), **know-how** (skills and prompts), **state** (memory and context), **bounds** (guardrails, hooks, budgets), and **escape hatches** (human-in-the-loop, escalation). Every question in this file is about one of those six components — and every production problem is a failure in one of them.

### Interview Follow-ups

- Where does LangGraph fit in this vocabulary? (It is a framework for building the loop explicitly as a graph, making state, edges, checkpointing, and interrupts first-class. See `12-langgraph.md`.)
- Which of these six components do teams most often get wrong? (Bounds. Capability is easy to add and fun to demo; guardrails, budgets, and verification are what make an agent shippable.)

---
