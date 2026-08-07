# LangGraph

Building agents as explicit graphs: state, nodes, edges, reducers, checkpointing, interrupts, subgraphs, and multi-agent workflows — with the conceptual distinctions interviews test.

**Questions:** 32

This file assumes the agent concepts from `11-ai-agents.md`. LangGraph is one concrete way of implementing them; the value of knowing it well is that it forces you to make state and control flow explicit.

---

## Easy

---

## Q1: What is LangGraph, and what problem does it solve?

### Answer

**LangGraph is a framework for building stateful, multi-step LLM applications as explicit graphs**, where you define nodes (units of work), edges (control flow), and a shared state object that flows through them.

**The problem it solves.** A simple agent loop is easy to write by hand:

```python
while True:
    response = llm(messages, tools=tools)
    if not response.tool_calls:
        return response
    messages += execute_tools(response.tool_calls)
```

That works until you need any of the following, at which point the hand-rolled version becomes unmanageable:

| Requirement | Why hand-rolling hurts |
|---|---|
| Pause for human approval and resume hours later | Needs durable state serialisation and resumption |
| Resume after a crash mid-task | Needs per-step checkpointing |
| Conditional branching between several sub-flows | Nested `if` statements sprawl quickly |
| Parallel execution of independent steps | Needs concurrency plus safe state merging |
| Streaming intermediate state to a UI | Needs an event bus threaded through everything |
| Cycles with different exit conditions per branch | Loop conditions become tangled |
| Multi-agent coordination with shared state | Needs explicit state ownership rules |
| Time travel / replay from a prior step | Needs a step-indexed state history |

LangGraph provides all of these as framework features built on one idea: **make the state and the control flow explicit and inspectable.**

**The mental model:** it is a **state machine for LLM applications**. Each node reads state and returns an update to it; edges decide which node runs next. Because state transitions are explicit and each one is checkpointed, everything else — persistence, resumption, human-in-the-loop, streaming, time travel — falls out naturally.

**When you do not need it.** A single LLM call, a two-step chain, or a simple tool loop with no persistence requirement does not benefit. LangGraph earns its complexity when you need durability, branching, or human interaction. Reaching for it on a trivial pipeline adds concepts without adding value.

### Interview Follow-ups

- How does LangGraph relate to LangChain? (Separate libraries. LangChain provides model wrappers, tool abstractions, and message types; LangGraph provides the orchestration layer. You can use LangGraph with LangChain components, or with a raw provider SDK.)
- Why a graph rather than a DAG? (Because agents need **cycles** — the ReAct loop is a cycle. A DAG cannot express "call the model, use a tool, call the model again." See Q13.)

---

## Q2: What is state in LangGraph?

### Answer

**State is a single shared object that flows through the graph.** Every node receives the current state and returns a partial update; LangGraph merges the update into the state and passes it to the next node.

```python
from typing import Annotated, TypedDict
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    messages: Annotated[list, add_messages]   # append, not overwrite
    user_id: str
    retrieved_docs: list[str]
    step_count: int
    needs_human_review: bool
```

**The three things to understand about state:**

**1. Nodes return partial updates, not the whole state.**

```python
def retrieve(state: AgentState) -> dict:
    docs = vector_store.search(state["messages"][-1].content)
    return {"retrieved_docs": docs}      # only this key is updated
```

Keys not mentioned are left unchanged. This keeps nodes decoupled — a node need not know about state fields it does not use.

**2. How updates are applied is controlled per key by a reducer.** The default is overwrite. `Annotated[list, add_messages]` means "append instead of replace" (see Q11).

**3. The state schema is the contract between nodes.** It is the single place that documents what information the graph carries. Designing it well is the main design activity when building a LangGraph app.

**What belongs in state:**
- The message history (almost always)
- Intermediate results nodes need to pass to each other
- Control flags that drive conditional edges
- Counters and budgets for loop termination
- The task's plan or todo list
- User/session identifiers

**What does not belong in state:**
- Large blobs — store them externally and keep a reference. State is serialised on every checkpoint, so a 50 MB field is checkpointed on every step.
- Secrets and credentials — state gets persisted and may be shown in traces.
- Things derivable cheaply from other state — redundant fields drift out of sync.
- Per-node scratch values nobody else needs.

**State design tip:** keep it flat and typed. Deeply nested state makes reducers awkward and updates error-prone. If you find yourself needing complex merge logic, that is a signal to flatten the schema or split into a subgraph with its own state (Q22).

### Interview Follow-ups

- Can different nodes see different state? (In one graph, all nodes share the same schema, though they may read and write different keys. Subgraphs can have their own schemas with explicit input/output mapping — that is the mechanism for scoping state.)
- What happens if two parallel nodes update the same key? (Without a reducer that can merge, it is an error or a lost update. This is exactly what reducers exist for — see Q11 and Q23.)

---

## Q3: What is a node, and what can you put inside one?

### Answer

**A node is a function (or any runnable) that takes the current state and returns a partial state update.** That is the entire contract:

```python
def my_node(state: AgentState) -> dict:
    # do anything
    return {"some_key": new_value}
```

**Because the contract is just "state in, state update out," a node can contain anything your code can do.** This is the point most people miss — a node is not "an LLM call" or "a tool." It is an arbitrary unit of work.

**What can go inside a node:**

| Category | Examples |
|---|---|
| **LLM calls** | Chat completion, with or without tools; structured output; a reasoning call |
| **Tool execution** | Calling APIs, running code, querying a database |
| **Retrieval** | Vector search, hybrid search, reranking, BM25 |
| **Database operations** | Reads, writes, transactions |
| **Pure computation** | Parsing, formatting, scoring, filtering, maths |
| **Validation** | Schema checks, business-rule checks, guardrails |
| **Transformation** | Reshaping state, summarising history, compressing context |
| **Routing preparation** | Computing a flag or classification that a conditional edge will read |
| **Human interaction** | Raising an interrupt to request approval |
| **Another graph** | A compiled subgraph invoked as a node |
| **A whole agent** | A full agent loop, including its own internal cycles |
| **External orchestration** | Enqueueing a job, sending a notification, writing a file |
| **Nothing at all** | A pass-through node that just marks a point in the flow |

**Concrete examples of each shape:**

```python
# 1. An LLM call
def call_model(state: AgentState) -> dict:
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}          # add_messages appends

# 2. Pure computation — no LLM involved
def compute_budget(state: AgentState) -> dict:
    used = sum(count_tokens(m) for m in state["messages"])
    return {"tokens_used": used, "over_budget": used > 60_000}

# 3. Retrieval
def retrieve(state: AgentState) -> dict:
    query = rewrite_query(state["messages"])
    docs = retriever.search(query, k=8, filters={"tenant_id": state["tenant_id"]})
    return {"retrieved_docs": docs}

# 4. Validation / guardrail
def validate_output(state: AgentState) -> dict:
    draft = state["messages"][-1].content
    problems = policy_checker(draft)
    return {"validation_errors": problems, "needs_revision": bool(problems)}

# 5. Routing preparation — computes a flag a conditional edge will read
def classify_intent(state: AgentState) -> dict:
    intent = small_model_classifier(state["messages"][-1].content)
    return {"intent": intent}

# 6. Context compression
def compress_history(state: AgentState) -> dict:
    if len(state["messages"]) < 20:
        return {}                            # no update — valid
    summary = summariser.invoke(state["messages"][:-6])
    return {"messages": [RemoveMessage(id=m.id) for m in state["messages"][:-6]]
                        + [SystemMessage(content=f"Earlier context: {summary}")]}
```

**Guidelines for node design:**
- **One responsibility per node.** If a node does retrieval *and* generation, you cannot retry, cache, or branch on them independently.
- **Nodes should be deterministic given state where possible** — it makes replay and testing meaningful.
- **Return `{}` when there is nothing to update.** That is valid and common.
- **Keep nodes testable as plain functions.** A node is just `f(state) -> dict`; unit-test it without the graph.
- **Do not hide control flow inside a node.** If a node decides what happens next, that decision belongs in a conditional edge (Q7) where it is visible in the graph.

### Interview Follow-ups

- Can a node be async? (Yes. Use async nodes for IO-bound work; the graph supports async invocation and will run parallel branches concurrently.)
- Can a node call the graph it belongs to? (Directly, that risks unbounded recursion. Use cycles via edges instead — that is what edges are for, and it keeps the loop visible and boundable.)

---

## Q4: What is an edge, and what are the types?

### Answer

**An edge defines which node runs next.** Edges are how control flow is expressed — and keeping control flow in edges rather than inside nodes is the core discipline of LangGraph.

**The types:**

| Edge type | API | Behaviour |
|---|---|---|
| **Normal (static)** | `add_edge("a", "b")` | Always go from `a` to `b` |
| **Conditional** | `add_conditional_edges("a", router_fn, path_map)` | A function inspects state and returns the next node name(s) |
| **Entry** | `add_edge(START, "a")` | Where execution begins |
| **Exit** | `add_edge("a", END)` | Where execution terminates |
| **Parallel fan-out** | Multiple `add_edge("a", ...)` from one node | All targets run concurrently |
| **Dynamic (Send)** | Router returns `[Send("n", payload), ...]` | Fan out to N dynamic instances (Q24) |

```python
from langgraph.graph import StateGraph, START, END

graph = StateGraph(AgentState)
graph.add_node("retrieve", retrieve)
graph.add_node("generate", generate)
graph.add_node("validate", validate_output)

graph.add_edge(START, "retrieve")          # entry
graph.add_edge("retrieve", "generate")     # static
graph.add_conditional_edges(               # conditional
    "validate",
    lambda s: "revise" if s["needs_revision"] else "done",
    {"revise": "generate", "done": END},   # note: "revise" creates a cycle
)
graph.add_edge("generate", "validate")

app = graph.compile()
```

**Static vs conditional — the distinction interviews ask about:**

| | Static edge | Conditional edge |
|---|---|---|
| Next node | **Fixed at build time** | **Decided at runtime from state** |
| Depends on state | No | Yes |
| Enables branching | No | Yes |
| Enables cycles | Only unconditional loops (infinite) | **Yes, with a controllable exit** |
| Visible in the graph diagram | As one arrow | As a fan of possible arrows |

**The practical rule:** if the next step is always the same, use a static edge — it is clearer and cheaper. Use a conditional edge exactly when the next step depends on what happened. Every loop in a useful agent has a conditional edge as its exit, because otherwise nothing can stop it.

**Where the routing decision should live.** The routing function should be a **cheap, deterministic read of state** — not a place to do work:

```python
# Good — reads a flag a node already computed
def route(state) -> str:
    return "tools" if state["messages"][-1].tool_calls else "end"

# Bad — does an LLM call inside a router
def route(state) -> str:
    return llm.invoke(f"Should we use tools? {state}")   # put this in a node
```

Do the expensive work in a node, write the decision into state, and let the edge read it. That keeps the decision checkpointed, traceable, and retryable.

### Interview Follow-ups

- Can a node have both a static and a conditional edge out of it? (No — that is contradictory. One or the other. A conditional edge can always include the static target as one of its options.)
- What happens if a routing function returns a name not in the path map? (An error at runtime. Always include a default branch, and prefer returning keys from a closed set — an enum or literal type.)

---

## Q5: What are START and END?

### Answer

**`START` and `END` are special sentinel nodes marking the graph's entry and exit points.**

```python
from langgraph.graph import START, END

graph.add_edge(START, "classify")     # execution begins at "classify"
graph.add_edge("respond", END)        # execution finishes after "respond"
```

**`START`** is a virtual node representing "before anything ran." The edge from `START` tells the graph which node consumes the initial input. You can also fan out from `START` to several nodes to begin with parallel work, or use a conditional edge from `START` to route the very first step:

```python
graph.add_conditional_edges(START, route_by_intent, {
    "faq": "simple_answer",
    "complex": "agent",
    "escalate": "human_handoff",
})
```

**`END`** is a virtual node representing termination. When execution reaches `END`, the graph returns the final state. Multiple nodes can point to `END` — that is normal and expected, since an agent can finish successfully, finish by escalating, or finish by hitting a budget cap.

**Why they exist as explicit sentinels rather than implicit conventions:**
1. **The graph is fully specified.** Entry and exit are part of the structure, not a separate configuration or a naming convention.
2. **Multiple entry and exit points are expressible** without special cases.
3. **Validation.** Compilation can check that every node is reachable from `START` and that some path reaches `END` — catching a whole class of structural bugs before the graph runs.
4. **Visualisation.** The rendered diagram has unambiguous start and finish markers.

**The most common bug involving them:** forgetting to route to `END` on some branch, so a path either dead-ends or loops. If a graph "hangs" or recurses to the step limit, check that every terminal branch of every conditional edge has `END` as an option.

**A related gotcha:** reaching `END` returns the final state, but if you are streaming, you must handle the case where the graph ends without producing the message you expected — for example when it terminated on a budget cap rather than a completed answer. Always inspect the final state rather than assuming the last message is an answer.

### Interview Follow-ups

- Can you have a graph with no `END`? (You can build one, but it will never terminate normally — it will hit the recursion limit. Practically, always have at least one path to `END`.)
- Does `END` mean the conversation is over? (No. It means this *graph invocation* finished. With a checkpointer, the next user turn resumes the same thread with accumulated state — `END` is a turn boundary, not a session boundary.)

---

## Q6: What is StateGraph, and what does compiling do?

### Answer

**`StateGraph` is the builder.** You instantiate it with a state schema, register nodes and edges, then `compile()` it into an executable application.

```python
graph = StateGraph(AgentState)                  # 1. declare the state schema
graph.add_node("agent", call_model)             # 2. register nodes
graph.add_node("tools", ToolNode(tools))
graph.add_edge(START, "agent")                  # 3. wire edges
graph.add_conditional_edges("agent", should_continue, {"tools": "tools", "end": END})
graph.add_edge("tools", "agent")

app = graph.compile(                            # 4. compile
    checkpointer=checkpointer,
    interrupt_before=["dangerous_tool"],
)
```

**What `compile()` does:**

1. **Validates the structure.** Are all edge targets real nodes? Is every node reachable from `START`? Are there conflicting edges out of a node? Does anything reach `END`? Catching these at compile time rather than at runtime is a real benefit of the build/compile split.
2. **Resolves the state schema and reducers.** Reads the `Annotated` metadata and builds the merge function for each key.
3. **Attaches runtime configuration** — checkpointer, store, interrupt points, retry policies.
4. **Returns an immutable, executable object** implementing the runnable interface: `invoke`, `stream`, `astream`, `batch`, plus state-management methods like `get_state` and `update_state`.

**Why the two-phase build/compile design matters:**
- **Structure is validated before execution**, so structural bugs surface at startup rather than mid-request.
- **The compiled graph is immutable and safe to share** across requests and threads. Compile once at application start, not per request.
- **The same structure can be compiled with different runtime config** — e.g. no checkpointer in tests, Postgres in production; interrupts enabled in a supervised environment, disabled in a batch job.
- **A compiled graph is itself a runnable**, so it can be used as a node in another graph (Q22).

**Runtime configuration passed at invocation, not compile time:**

```python
config = {
    "configurable": {"thread_id": "conv-123", "user_id": "u-42"},
    "recursion_limit": 50,
}
result = app.invoke({"messages": [HumanMessage("hello")]}, config=config)
```

`thread_id` selects the conversation for checkpointing; `recursion_limit` caps total steps (the safety net against infinite cycles — see Q13). Anything under `configurable` is available to nodes, which is the right place for request-scoped values like the authenticated user id — **not** in state, because state is model-influenceable and gets persisted.

### Interview Follow-ups

- Should you compile per request? (No. Compile at startup and reuse. Compilation does structural validation work that does not need repeating, and the compiled object is immutable.)
- What is the difference between state and config? (State is data the graph *computes and mutates*, checkpointed per step. Config is *request-scoped input* — thread id, user id, limits — that nodes read but do not update. Security-relevant values belong in config.)

---

## Q7: What is a conditional edge, and how does routing work?

### Answer

**A conditional edge calls a function that inspects state and returns the name (or names) of the next node(s).**

```python
def should_continue(state: AgentState) -> str:
    last = state["messages"][-1]
    if state["step_count"] > 10:
        return "budget_exceeded"
    if getattr(last, "tool_calls", None):
        return "tools"
    return "end"

graph.add_conditional_edges(
    "agent",
    should_continue,
    {
        "tools": "tools",
        "budget_exceeded": "escalate",
        "end": END,
    },
)
```

**The three parts:**
1. **Source node** — where the decision is made from.
2. **Routing function** — `state -> str | list[str]`. Pure, cheap, deterministic.
3. **Path map** — maps the returned keys to node names. Optional if the function returns node names directly, but using a map is better practice: it decouples the routing vocabulary from node names and makes the mapping visible in the diagram.

**Routing patterns:**

```python
# Classification-driven routing (a node computed `intent` earlier)
graph.add_conditional_edges("classify", lambda s: s["intent"], {
    "billing": "billing_agent",
    "technical": "tech_agent",
    "other": "general_agent",
})

# Quality gate with a retry cycle and a bounded number of attempts
def quality_gate(state) -> str:
    if state["score"] >= 0.8:
        return "accept"
    if state["attempts"] >= 3:
        return "give_up"
    return "retry"

graph.add_conditional_edges("evaluate", quality_gate, {
    "accept": "finalise", "retry": "generate", "give_up": "escalate",
})

# Parallel fan-out — return a list
def pick_sources(state) -> list[str]:
    return [s for s in ["docs", "tickets", "code"] if s in state["relevant_sources"]]

graph.add_conditional_edges("plan", pick_sources,
    {"docs": "search_docs", "tickets": "search_tickets", "code": "search_code"})
```

**Design rules:**

- **Routing functions must be cheap and side-effect free.** Do the LLM call or the computation in a node; write the result to state; have the edge read it. This keeps the decision checkpointed and replayable, and keeps routing fast.
- **Always have a default.** An unmapped return value is a runtime error. Use a `Literal` return type or an enum so the type checker helps you.
- **Every cycle needs a bounded exit.** If a conditional edge can route back to an earlier node, it must also be able to route to `END` or an escape node, and something in state must guarantee that eventually happens (an attempt counter, a budget, a deadline).
- **Prefer explicit flags over re-derivation.** `if state["needs_revision"]` is better than re-running the validation logic inside the router — the flag is visible in the checkpoint and in traces.

**Edge vs conditional edge summary:** a static edge is an arrow; a conditional edge is a switch. The switch is what makes the graph an agent rather than a pipeline, because it is the mechanism by which runtime state determines control flow.

### Interview Follow-ups

- Can a conditional edge return multiple targets? (Yes — return a list and all targets run in parallel. Their state updates merge via reducers, so the affected keys need mergeable reducers.)
- How do you visualise the possible paths? (`app.get_graph().draw_mermaid()` renders the structure, with conditional edges shown as dashed branches. Very useful for review and documentation.)

---

## Q8: What is a ToolNode, and how does tool calling work in LangGraph?

### Answer

**`ToolNode` is a prebuilt node that executes the tool calls found in the last message and returns the results as `ToolMessage`s.**

```python
from langgraph.prebuilt import ToolNode
from langchain_core.tools import tool

@tool
def get_weather(city: str) -> str:
    """Get the current weather for a city."""
    return f"22C and sunny in {city}"

tools = [get_weather, search_docs]
llm_with_tools = llm.bind_tools(tools)

graph.add_node("agent", lambda s: {"messages": [llm_with_tools.invoke(s["messages"])]})
graph.add_node("tools", ToolNode(tools))
```

**The canonical ReAct graph — the single most important structure in LangGraph:**

```python
def should_continue(state) -> str:
    return "tools" if state["messages"][-1].tool_calls else "end"

graph.add_edge(START, "agent")
graph.add_conditional_edges("agent", should_continue, {"tools": "tools", "end": END})
graph.add_edge("tools", "agent")            # <-- the cycle
```

```text
START → agent ──(tool_calls?)──→ tools
          ↑                        │
          └────────────────────────┘
          │
          └──(no tool_calls)──→ END
```

That is the entire ReAct loop from `11-ai-agents.md` Q2 expressed as a graph. The cycle `agent → tools → agent` is the loop; the conditional edge is the termination check.

**What `ToolNode` does for you:**
1. Reads `tool_calls` from the last AI message.
2. Looks each tool up by name.
3. Executes them — **in parallel** if there are several.
4. Wraps each result in a `ToolMessage` with the matching `tool_call_id`.
5. Returns `{"messages": [...]}`, which `add_messages` appends.
6. Catches exceptions and returns them as tool messages so the model can see and recover from the error (configurable via `handle_tool_errors`).

**When to write your own tool node instead.** `ToolNode` covers the common case. Write a custom node when you need:
- **Per-tool authorisation** based on the session (the before-tool hook from `11-ai-agents.md` Q18)
- **Argument injection** — adding `tenant_id` from config so the model cannot supply it
- **Result post-processing** — redaction, truncation, summarisation
- **Custom retry or timeout policies** per tool
- **State updates beyond messages** — e.g. recording retrieved documents into a dedicated state key

```python
def guarded_tools(state: AgentState, config) -> dict:
    tenant = config["configurable"]["tenant_id"]        # from config, not state
    out = []
    for call in state["messages"][-1].tool_calls:
        if call["name"] not in PERMITTED[config["configurable"]["role"]]:
            out.append(ToolMessage(
                content=f"Permission denied for {call['name']}.",
                tool_call_id=call["id"]))
            continue
        args = {**call["args"], "tenant_id": tenant}     # inject, never trust
        try:
            result = TOOLS[call["name"]].invoke(args)
        except Exception as e:
            result = f"Error: {e}. Check the arguments and try again."
        out.append(ToolMessage(content=cap_size(redact(str(result))),
                               tool_call_id=call["id"]))
    return {"messages": out}
```

**Interrupting before a tool.** `compile(interrupt_before=["tools"])` pauses the graph before the tool node runs, which is how you implement approval gates (Q17). This is only safe because the pause happens *before* the side effect.

### Interview Follow-ups

- What happens if the model calls a tool that does not exist? (`ToolNode` returns an error `ToolMessage`, and the model typically corrects itself on the next turn. This is the actionable-error principle from `11-ai-agents.md` Q13, built in.)
- Why must tool results be `ToolMessage`s with matching ids? (The provider APIs require every `tool_use` block to be answered by a `tool_result` with the same id. Mismatched or missing ids make the conversation malformed and the next API call fails.)

---

## Q9: What is the difference between a node and a tool?

### Answer

| | Node | Tool |
|---|---|---|
| **What it is** | A unit of work in the graph | A function the **model** can request |
| **Who invokes it** | **The graph**, via edges | **The model**, by emitting a tool call |
| **Signature** | `state -> dict` (state update) | Typed arguments -> result |
| **Visible to the model** | No — the model does not know nodes exist | **Yes** — via its schema in the prompt |
| **Control** | Deterministic, defined by you | Model's choice at runtime |
| **Contains** | Anything, including tool execution | A specific operation |

**The clearest way to state it: nodes are graph structure; tools are model capabilities.**

The graph decides which node runs based on edges you wrote. The model decides which tool to call based on the task. These are different layers, and `ToolNode` is the bridge — a *node* whose job is to execute the *tools* the model requested.

```text
Graph layer (you control):        START → agent → tools → agent → END
                                            │        ↑
Model layer (the model controls):           └─ "call get_weather('Tokyo')"
```

**Why the distinction matters practically:**

- **A tool cannot change control flow directly.** It returns a result, the graph resumes at whatever edge follows the tool node. If you want a tool's outcome to change the route, a node must write a flag to state and a conditional edge must read it.
- **A node can do things no tool can:** modify arbitrary state keys, raise an interrupt, invoke a subgraph, or route.
- **A tool can do something no node can:** be *selected by the model* from a set of options based on natural-language reasoning.

**The common confusion, and how to resolve it.** People ask "should this be a node or a tool?" The answer follows from *who should decide when it runs*:

```text
Should the MODEL decide when this runs, based on the task?
├── YES → tool
└── NO  → node
```

Retrieval is the classic example that can be either:
- **As a node:** it always runs before generation. Deterministic RAG pipeline. Predictable, cheap.
- **As a tool:** the model decides whether and what to retrieve. Agentic RAG. Flexible, more expensive.

Both are valid; they are different architectures (see `11-ai-agents.md` Q23), and choosing between them *is* the design decision.

### Interview Follow-ups

- Can a tool be implemented as a subgraph? (Yes — wrap a compiled graph in a tool function. That is one way to build a sub-agent the parent model can invoke by name.)
- Can a node call a tool without the model asking? (Absolutely — a node is just code. "Tool" in the LangChain sense means it also has a schema exposed to the model, but nothing stops a node from calling the same function directly.)

---

## Intermediate

---

## Q10: How are messages handled in LangGraph?

### Answer

**Messages are the conversation history, stored in a state key with the `add_messages` reducer.**

```python
from langgraph.graph.message import add_messages
from typing import Annotated

class State(TypedDict):
    messages: Annotated[list, add_messages]
```

**Message types:**

| Type | Role | Contains |
|---|---|---|
| `SystemMessage` | system | Instructions, persona, constraints |
| `HumanMessage` | user | User input |
| `AIMessage` | assistant | Model output; may carry `tool_calls` |
| `ToolMessage` | tool | A tool result, with `tool_call_id` |
| `RemoveMessage` | — | A sentinel instructing the reducer to delete a message |

**What `add_messages` does — and why it is more than `list.append`:**

1. **Appends** new messages to the existing list.
2. **Deduplicates and updates by id.** If an incoming message has the same id as an existing one, it *replaces* it. This is how streaming updates and corrections work.
3. **Assigns ids** to messages that lack them.
4. **Coerces** dicts and raw strings into proper message objects.
5. **Handles `RemoveMessage`** — removes the message with the matching id, which is the mechanism for trimming or rewriting history.

```python
# Append
return {"messages": [AIMessage(content="Hi")]}

# Replace an existing message (same id)
return {"messages": [AIMessage(content="Hi there", id=existing_id)]}

# Delete messages — used for compression
return {"messages": [RemoveMessage(id=m.id) for m in old_messages]}
```

**History management — the practical problem.** Messages grow without bound (see `11-ai-agents.md` Q6). Three approaches:

```python
# 1. Trim before the model call — simplest, does not mutate state
from langchain_core.messages import trim_messages

def call_model(state):
    trimmed = trim_messages(
        state["messages"], max_tokens=40_000, token_counter=llm,
        strategy="last", include_system=True, start_on="human",
    )
    return {"messages": [llm.invoke(trimmed)]}

# 2. Summarise and delete — reduces the checkpointed state itself
def compress(state):
    msgs = state["messages"]
    if len(msgs) < 20:
        return {}
    summary = summariser.invoke(msgs[:-6])
    return {"messages": [RemoveMessage(id=m.id) for m in msgs[:-6]]
                        + [SystemMessage(content=f"Summary of earlier conversation: {summary}")]}

# 3. Keep full history in state, send a curated subset to the model
#    (the "store everything, send a subset" pattern — usually the best default)
```

**Important subtleties:**
- **Never trim in a way that orphans a tool call.** If you drop an `AIMessage` containing `tool_calls` but keep its `ToolMessage`s (or vice versa), the provider API rejects the request. `trim_messages` with `start_on="human"` avoids this; hand-rolled trimming often does not. This is one of the most common LangGraph bugs.
- **Preserve the system message.** `include_system=True`, or re-prepend it.
- **Trimming for the model call vs deleting from state are different operations.** Option 1 keeps the full history checkpointed (good for audit, replay, and UI) while sending less to the model. Option 2 permanently reduces the checkpoint size. Choose deliberately.
- **When summarising, keep the record of what was tried and what happened.** Dropping that causes the agent to repeat failed work — the classic compression bug.

### Interview Follow-ups

- Why does `add_messages` deduplicate by id rather than just appending? (So a node can *update* a message — essential for streaming partial output, correcting a message, and for `RemoveMessage` to target a specific entry.)
- Should you keep messages in state or in an external store? (State, for the working conversation — that is what checkpointing is for. Move to summaries plus an external archive when history grows beyond what you want to serialise on every step.)

---

## Q11: What are reducers, and why do they exist?

### Answer

**A reducer is a function that decides how a node's update to a state key is combined with the existing value.**

```python
def reducer(current_value, update_value) -> new_value
```

**Why they exist.** Without reducers, "returning a state update" would have to mean "overwrite." That is wrong for accumulating values, and it is *unsafe* for parallel execution:

```python
# Default (overwrite): each node's messages replace the previous ones — history lost
messages: list

# With add_messages: updates append — history accumulates
messages: Annotated[list, add_messages]
```

**Two distinct problems reducers solve:**

**1. Accumulation.** Message history, collected documents, and a growing findings list all need appending rather than replacing.

**2. Parallel merge — the harder problem.** When two nodes run concurrently and both write the same key, there is no defined "last writer." A reducer makes the merge *deterministic and total*:

```python
# Three retrieval nodes run in parallel, all writing to `docs`
class State(TypedDict):
    docs: Annotated[list, operator.add]     # results concatenate — safe
    count: int                              # NO reducer — parallel writes conflict
```

Without a reducer on a key that parallel branches both write, you get an error (LangGraph detects the conflict) — which is a feature, since a silent lost update would be far worse.

**Common reducers:**

```python
import operator
from typing import Annotated

class State(TypedDict):
    messages: Annotated[list, add_messages]      # append + dedupe by id
    docs: Annotated[list, operator.add]          # concatenate lists
    total_tokens: Annotated[int, operator.add]   # sum
    seen_ids: Annotated[set, operator.or_]       # set union
    current_step: str                            # default: overwrite
```

**Custom reducers** for domain-specific merges:

```python
def merge_scores(current: dict, update: dict) -> dict:
    """Keep the highest score seen per document."""
    merged = dict(current or {})
    for doc_id, score in (update or {}).items():
        merged[doc_id] = max(merged.get(doc_id, 0.0), score)
    return merged

def dedupe_by_source(current: list, update: list) -> list:
    """Concatenate but keep only the first chunk per source."""
    seen = {d["source"] for d in (current or [])}
    return (current or []) + [d for d in (update or []) if d["source"] not in seen]

class State(TypedDict):
    scores: Annotated[dict, merge_scores]
    docs: Annotated[list, dedupe_by_source]
```

**Requirements for a correct reducer used with parallel branches:**
- **Handle `None`/empty current value** — it runs on the first write too.
- **Be associative and commutative** if branches can complete in any order. `operator.add` on lists is associative but *not* commutative — the result order depends on completion order. That is usually fine for a document set, but not if downstream logic depends on ordering. If order matters, sort explicitly afterwards rather than relying on merge order.
- **Not mutate the inputs** — return a new value. Mutating breaks checkpointing and replay.
- **Be cheap.** It runs on every update to that key.

**The mental model:** reducers are to LangGraph state what a CRDT merge function is to distributed data — they make concurrent updates to shared state well-defined. That is what makes parallel execution (Q23) safe.

### Interview Follow-ups

- What happens if two parallel nodes write a key with no reducer? (LangGraph raises an `InvalidUpdateError`. Add a reducer, or have only one branch write that key.)
- Why is `add_messages` not just `operator.add`? (It also assigns ids, deduplicates and replaces by id, coerces types, and processes `RemoveMessage` — none of which plain concatenation does.)

---

## Q12: What is the difference between state and memory?

### Answer

| | State | Memory |
|---|---|---|
| **Scope** | One graph execution / one thread | **Across threads and sessions** |
| **Lifetime** | The conversation (thread) | Indefinite, until deleted |
| **Mechanism** | The state object, checkpointed per step | A `Store` (or external DB), queried explicitly |
| **Keyed by** | `thread_id` | `user_id` / namespace |
| **Contains** | Messages, intermediate results, control flags | Preferences, learned facts, past-session summaries |
| **Access** | Automatic — every node receives it | **Explicit** — a node must read/write it |
| **Analogy** | RAM for this task | A database of what we know about this user |

**The distinction in one sentence: state is *this conversation*; memory is *everything we know across conversations*.**

**How they relate to checkpointing.** A checkpointer gives you *thread-scoped persistence* — resume conversation `conv-123` where it left off, with its full state. That is often called "memory," and for a single ongoing conversation it is sufficient. But it is scoped to the thread: start a new conversation and the state is empty, even for the same user.

**Cross-thread memory needs a separate mechanism:**

```python
from langgraph.store.memory import InMemoryStore   # Postgres store in production

store = InMemoryStore()
app = graph.compile(checkpointer=checkpointer, store=store)

def load_memory(state: State, config, *, store) -> dict:
    user_id = config["configurable"]["user_id"]
    items = store.search(("memories", user_id), query=state["messages"][-1].content, limit=5)
    facts = "\n".join(f"- {i.value['text']}" for i in items)
    return {"messages": [SystemMessage(content=f"What you know about this user:\n{facts}")]}

def save_memory(state: State, config, *, store) -> dict:
    user_id = config["configurable"]["user_id"]
    for fact in extract_durable_facts(state["messages"]):     # LLM extraction
        store.put(("memories", user_id), key=fact["id"], value={"text": fact["text"]})
    return {}
```

Typical placement: `load_memory` near `START`, `save_memory` near `END` — mapping onto steps 1 and 8 of the agent lifecycle (`11-ai-agents.md` Q3).

**Mapping onto the memory taxonomy from `11-ai-agents.md` Q7:**

| Memory type | LangGraph mechanism |
|---|---|
| Working / short-term | **State** + checkpointer (thread-scoped) |
| Episodic (past sessions) | **Store**, namespaced by user, holding session summaries |
| Semantic (facts) | **Store**, with semantic search over extracted facts |
| Procedural (how-to) | System prompt, tool definitions, or store-held instructions |

**The design guidance:** get state and checkpointing right first — that covers the majority of real needs. Add a store only when you have a concrete requirement for cross-session knowledge, and keep what you write to it small, explicit, timestamped, and user-visible. Long-term memory is where correctness, staleness, and privacy problems live, and an unnecessary memory store is a liability rather than a feature.

### Interview Follow-ups

- Is a checkpointer "memory"? (It is thread-scoped memory. It gives conversation continuity but not cross-conversation knowledge. Interviews often use "memory" loosely — clarifying which one you mean is a good signal.)
- How do you prevent memory contamination? (Validate before writing, prefer explicit user confirmation for sensitive facts, timestamp everything, prefer recent over older on conflict, and give users visibility and deletion. A wrong persisted fact poisons every future session.)

---

## Q13: How do cycles work, and how do you keep them safe?

### Answer

**A cycle is an edge path that returns to an earlier node.** Cycles are the reason LangGraph is a *graph* framework rather than a DAG framework — the ReAct loop is inherently cyclic.

```python
graph.add_conditional_edges("agent", should_continue, {"tools": "tools", "end": END})
graph.add_edge("tools", "agent")            # agent → tools → agent
```

**Common cyclic patterns:**

| Pattern | Cycle | Exit condition |
|---|---|---|
| **ReAct loop** | `agent → tools → agent` | Model emits no tool calls |
| **Reflection loop** | `generate → critique → generate` | Quality threshold met or attempt cap |
| **Retry loop** | `act → check → act` | Success or retry cap |
| **Iterative retrieval** | `retrieve → assess → retrieve` | Sufficient information gathered |
| **Plan–execute–replan** | `plan → execute → evaluate → plan` | Plan complete |

**How cycles terminate — three independent layers, and you want all three:**

**1. The model's decision.** The primary exit: the model stops requesting tools, or a critic says the output is good enough. This is the *intended* exit path.

**2. Explicit state-based caps.** Do not rely on the model. Count and enforce:

```python
class State(TypedDict):
    messages: Annotated[list, add_messages]
    iterations: int
    tokens_used: int

def agent(state: State) -> dict:
    return {"messages": [llm_with_tools.invoke(state["messages"])],
            "iterations": state["iterations"] + 1}

def should_continue(state: State) -> str:
    if state["iterations"] >= 10:
        return "budget_exceeded"                 # graceful, informative exit
    if state["tokens_used"] > 100_000:
        return "budget_exceeded"
    return "tools" if state["messages"][-1].tool_calls else "end"

graph.add_conditional_edges("agent", should_continue, {
    "tools": "tools",
    "budget_exceeded": "summarise_partial",      # report what was achieved
    "end": END,
})
```

**3. `recursion_limit` — the framework backstop.**

```python
app.invoke(input, config={"recursion_limit": 50})
```

This caps total **super-steps** and raises `GraphRecursionError` when exceeded. It is a **safety net, not a control mechanism** — hitting it means an ungraceful crash with no partial result. Design your state-based cap to trigger well below it so the agent exits cleanly with a useful partial answer.

**The distinction worth stating in an interview:** `recursion_limit` prevents infinite loops; state-based caps produce *good behaviour at the limit*. Relying only on `recursion_limit` gives you an exception where you wanted a graceful "here is what I found before running out of budget."

**Also detect no-progress loops**, which caps alone will not catch until the budget is gone:

```python
def agent(state: State) -> dict:
    response = llm_with_tools.invoke(state["messages"])
    sig = tool_call_signature(response)               # (name, canonical args)
    repeats = state["signatures"].count(sig)
    return {"messages": [response],
            "signatures": state["signatures"] + [sig],
            "stuck": repeats >= 2}
```

Then route on `stuck` to an escape node. An agent calling the same tool with the same arguments three times will not succeed on the fourth attempt.

**Nested cycles.** A subgraph with its own cycle inside a parent cycle multiplies step counts. Give the subgraph its own budget and its own recursion limit, or the combined behaviour becomes very hard to bound.

### Interview Follow-ups

- Why does a DAG framework fail for agents? (A DAG cannot express "call the model again after using a tool," which is the defining structure of an agent. You would have to unroll the loop to a fixed depth, which defeats the point.)
- What is a super-step? (One iteration of the graph's execution: all nodes triggered in the current wave run, then state is merged and checkpointed, then the next wave is determined. Parallel nodes in the same wave count as one super-step.)

---

## Q14: What is checkpointing and persistence, and what does it enable?

### Answer

**A checkpointer saves the graph's full state after every super-step**, keyed by `thread_id`, so execution can be resumed, inspected, or replayed.

```python
from langgraph.checkpoint.postgres import PostgresSaver

checkpointer = PostgresSaver.from_conn_string(DB_URL)
app = graph.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "conv-123"}}
app.invoke({"messages": [HumanMessage("What's my order status?")]}, config)

# Later — possibly days later, in a different process
app.invoke({"messages": [HumanMessage("What about the other one?")]}, config)
#   ^ full prior state is loaded automatically for thread conv-123
```

**What checkpointing enables — and this is why it is foundational rather than optional:**

| Capability | How checkpointing provides it |
|---|---|
| **Multi-turn conversations** | Turn 2 loads turn 1's state by `thread_id` |
| **Human-in-the-loop** | The graph pauses, state is durable, resume hours later (Q17) |
| **Crash recovery** | Resume from the last completed super-step, not from scratch |
| **Time travel / replay** | Fetch a historical checkpoint and re-run from it (Q18) |
| **Debugging** | Inspect the exact state at every step of a past run |
| **Statelessness at the process level** | Any worker can serve any turn — required for horizontal scaling |
| **Audit** | A durable record of every state transition |

**Checkpointer implementations:**

| Checkpointer | Use |
|---|---|
| `MemorySaver` | Tests and local development only — lost on restart |
| `SqliteSaver` | Single-node apps, prototypes, local tools |
| `PostgresSaver` | **Production default** — durable, transactional, scalable |
| Redis / custom | Custom trade-offs; implement the `BaseCheckpointSaver` interface |

**Operational realities that interviews probe:**

1. **Serialisation.** Everything in state must be serialisable. Live DB connections, open file handles, and arbitrary objects will fail. Keep state to plain data.
2. **Checkpoint size.** State is written on **every** super-step. A large field — full documents, embeddings, images — multiplies storage and write latency by the number of steps. Store blobs externally and keep references in state. This is the single most common performance mistake.
3. **Retention.** Checkpoints accumulate. A 30-step conversation writes 30 checkpoints. You need a retention policy and a cleanup job, or the table grows unboundedly.
4. **Thread id design.** `thread_id` is the isolation boundary for conversation state. Include the tenant and user in it (or namespace by them) so a leaked or guessed id cannot expose another user's conversation. Never use a sequential integer.
5. **Concurrency on one thread.** Two simultaneous invocations on the same `thread_id` race. Serialise per thread with a lock or queue.
6. **Privacy.** Checkpoints contain full conversation content — personal data subject to retention limits and erasure requests. Plan for deletion by user, not just by thread.

**The framing to give:** checkpointing turns the graph from an in-memory function call into a **durable, resumable workflow**. Almost every advanced LangGraph feature — interrupts, HITL, time travel, crash recovery — is a direct consequence of having per-step durable state. If you understand that, the rest of the framework follows.

### Interview Follow-ups

- What is the difference between a checkpointer and a store? (Checkpointer: thread-scoped graph state, written automatically every step. Store: cross-thread key-value/semantic memory, read and written explicitly by nodes. See Q12.)
- Can you run without a checkpointer? (Yes, for stateless single-shot graphs. But you lose multi-turn state, interrupts, HITL, and replay — so most real applications need one.)

---

## Q15: How does streaming work in LangGraph, and what are the stream modes?

### Answer

LangGraph streams at several granularities, because for an agent the interesting output is not only the final tokens but the *progress* through the graph.

| Mode | Yields | Use for |
|---|---|---|
| `"values"` | The **full state** after each super-step | Simple UIs that re-render from complete state |
| `"updates"` | Only the **delta** each node returned | Progress display, logging, "step X finished" |
| `"messages"` | **LLM tokens** as they generate, with metadata | Token-by-token chat UI |
| `"custom"` | Data a node explicitly emits | Domain progress: "read 12 of 40 files" |
| `"debug"` | Detailed execution events | Tracing and diagnostics |

```python
# Token streaming for the chat UI
for chunk, metadata in app.stream(input, config, stream_mode="messages"):
    if metadata["langgraph_node"] == "generate":       # only stream the answer node
        print(chunk.content, end="", flush=True)

# Progress streaming
for update in app.stream(input, config, stream_mode="updates"):
    for node_name, node_update in update.items():
        print(f"[{node_name}] {summarise(node_update)}")

# Multiple modes at once
for mode, payload in app.stream(input, config, stream_mode=["updates", "messages"]):
    ...
```

**Custom progress events from inside a node:**

```python
from langgraph.config import get_stream_writer

def process_files(state: State) -> dict:
    writer = get_stream_writer()
    results = []
    for i, path in enumerate(state["files"]):
        writer({"progress": f"Reading {i + 1} of {len(state['files'])}: {path}"})
        results.append(analyse(path))
    return {"results": results}
```

**Choosing modes for a real application.** A production agent UI typically combines:
- `"updates"` → drive a step indicator and tool-activity display ("Searching the knowledge base…")
- `"messages"` filtered to the final answer node → token-by-token output
- `"custom"` → long-running node progress

**Why filtering by node matters.** Without it, you stream the tokens of *every* LLM call — including the classifier, the query rewriter, and the critic. Users should see the answer, not the internal monologue of every intermediate model call. Filter on `metadata["langgraph_node"]`.

**Practical concerns (the same as in `11-ai-agents.md` Q16, plus LangGraph specifics):**
- **Async for real streaming.** Use `astream` in an async web framework; sync `stream` in an ASGI app can block the event loop.
- **Proxy buffering** silently breaks SSE. This is the number one "works locally, fails in production" cause.
- **Errors mid-stream** need an explicit error event — you cannot return a 500 after streaming has started.
- **Interrupts pause the stream.** Your client must handle an interrupt event and render an approval UI rather than treating it as completion.
- **Subgraph streaming** requires `subgraphs=True`, otherwise inner node events are not surfaced — a common surprise when refactoring into subgraphs.
- **Redact before emitting.** Streamed tool arguments and state may contain secrets or PII; the stream is user-visible output and needs the same output guardrails as the final answer.

### Interview Follow-ups

- `values` vs `updates`? (`values` sends the whole state every step — simple but wasteful as state grows. `updates` sends only what changed — efficient, but the client must maintain the accumulated view. Prefer `updates` for anything non-trivial.)
- How do you stream from a node that is not an LLM call? (Custom mode with `get_stream_writer()`. This is the only way to give users progress on long deterministic work.)

---

## Q16: What are interrupts, and how do they work?

### Answer

**An interrupt pauses graph execution, persists the state, and returns control to the caller.** Resuming continues from exactly that point.

**Two mechanisms:**

**1. Static interrupts — configured at compile time.**

```python
app = graph.compile(
    checkpointer=checkpointer,
    interrupt_before=["execute_payment"],   # pause before this node runs
    interrupt_after=["draft_email"],        # pause after this node runs
)
```

**2. Dynamic interrupts — raised from inside a node, conditionally.**

```python
from langgraph.types import interrupt, Command

def approve_refund(state: State) -> dict:
    amount = state["refund_amount"]
    if amount <= 50:
        return {"approved": True}                    # no interrupt needed

    decision = interrupt({                            # pauses here
        "question": f"Approve a refund of ${amount}?",
        "order_id": state["order_id"],
        "reason": state["refund_reason"],
    })
    return {"approved": decision["approved"], "approver": decision["user"]}
```

**Dynamic interrupts are usually the right choice** because the decision to pause is itself state-dependent: pause only for high-value refunds, only for irreversible actions, only when confidence is low. A static interrupt pauses every time, which causes approval fatigue (`11-ai-agents.md` Q15).

**The resume flow:**

```python
config = {"configurable": {"thread_id": "conv-123"}}

result = app.invoke(initial_input, config)
if "__interrupt__" in result:
    payload = result["__interrupt__"][0].value     # show this to the human
    # ... process exits; hours may pass; a different worker may handle the resume ...
    app.invoke(Command(resume={"approved": True, "user": "alice"}), config)
```

**How resumption actually works — the subtlety worth knowing.** When you resume, LangGraph **re-executes the interrupted node from its beginning**, and the `interrupt()` call returns the resume value instead of pausing. It does not magically continue mid-function.

The consequence: **any side effect that occurs before `interrupt()` in the same node will run twice.**

```python
# WRONG — the charge happens, then we ask for approval, then on resume we charge again
def bad(state):
    charge_card(state["amount"])                  # runs on the first pass
    decision = interrupt("Approve?")              # pause
    return {"done": decision}                     # on resume, charge_card runs AGAIN

# RIGHT — approve first, act after
def good(state):
    decision = interrupt({"amount": state["amount"], "question": "Approve charge?"})
    if not decision["approved"]:
        return {"status": "rejected"}
    charge_card(state["amount"])                  # runs once, after approval
    return {"status": "charged"}
```

**The rule: put `interrupt()` at the top of the node, before any side effect.** If you cannot, split into two nodes — one that interrupts, one that acts.

**What you can do on resume:**

| Action | How |
|---|---|
| Approve | `Command(resume={"approved": True})` |
| Reject | `Command(resume={"approved": False})` — the node must handle this path |
| Provide input | `Command(resume={"value": "eu-west-1"})` |
| Edit state | `app.update_state(config, {"draft": corrected_text})` then resume |
| Redirect | `Command(goto="different_node", update={...})` |
| Abandon | Simply never resume; the checkpoint remains for audit |

**Requirements for interrupts to work at all:** a **checkpointer is mandatory** — the pause is durable only because state is persisted. And a stable `thread_id` is how you find the paused execution again. This is why Q14 and Q16/Q17 are inseparable.

### Interview Follow-ups

- Can you interrupt inside a parallel branch? (Yes, but it pauses the whole super-step, and on resume the other branches in that step may re-execute. Keep interrupts out of parallel sections unless the branches are idempotent.)
- What if the human never resumes? (The checkpoint sits there indefinitely. You need an external timeout job that either cancels the thread or resumes with a safe default — never leave it hanging silently.)

---

## Q17: How do you implement human-in-the-loop in LangGraph?

### Answer

The four HITL patterns from `11-ai-agents.md` Q15, mapped to LangGraph mechanisms:

| Pattern | Mechanism |
|---|---|
| **Approve / reject** | `interrupt()` before the action; resume with a boolean |
| **Edit** | `interrupt()` returning the proposal; `update_state` with the correction, then resume |
| **Provide input** | `interrupt()` asking a question; resume with the answer |
| **Review output** | `interrupt_after` the drafting node, before the send node |

**Approve/reject — the canonical implementation:**

```python
from langgraph.types import interrupt

DESTRUCTIVE = {"issue_refund", "send_email", "delete_record", "deploy"}

def maybe_approve_tools(state: State) -> dict:
    calls = state["messages"][-1].tool_calls
    risky = [c for c in calls if c["name"] in DESTRUCTIVE]
    if not risky:
        return {"approved_calls": [c["id"] for c in calls]}

    decision = interrupt({
        "type": "tool_approval",
        "calls": [{"id": c["id"], "name": c["name"], "args": redact(c["args"])}
                  for c in risky],
        "message": "These actions are irreversible. Approve?",
    })
    return {"approved_calls": decision["approved_ids"]}

graph.add_node("approve", maybe_approve_tools)
graph.add_edge("agent", "approve")
graph.add_conditional_edges("approve", lambda s: "tools" if s["approved_calls"] else "agent",
                            {"tools": "tools", "agent": "agent"})
```

Note the structure: **a separate approval node before the tool node.** This guarantees the interrupt precedes the side effect, avoiding the double-execution trap from Q16.

**Edit — letting a human correct the agent's proposal:**

```python
result = app.invoke(user_input, config)
if "__interrupt__" in result:
    proposal = result["__interrupt__"][0].value
    corrected = show_editor_to_human(proposal)                  # your UI
    app.update_state(config, {"draft_email": corrected})        # write the correction
    app.invoke(Command(resume={"approved": True}), config)      # continue
```

**Provide input — asking the user rather than guessing:**

```python
def get_missing_region(state: State) -> dict:
    if state.get("region"):
        return {}
    region = interrupt({"question": "Which AWS region?",
                        "options": ["us-east-1", "eu-west-1", "ap-south-1"]})
    return {"region": region}
```

**Review output before an external send:**

```python
app = graph.compile(checkpointer=checkpointer, interrupt_before=["send_email"])
```

**Production requirements — the parts a demo skips:**

1. **A durable checkpointer.** `MemorySaver` loses everything on restart. Use Postgres.
2. **Interrupt payloads must be self-describing.** The human sees only the payload, possibly in a Slack message hours later. Include the action, the arguments, the reasoning, and the consequence.
3. **Timeouts with a safe default.** A scheduled job that finds threads paused beyond an SLA and resumes them with `{"approved": False}` plus a notification. Never default to approve.
4. **The rejection path must exist.** The node must handle `approved: False` and the graph must route somewhere sensible — usually back to the agent with the rejection as an observation so it can adapt rather than retry.
5. **Authorisation on the resume.** Verify the *resuming* human is entitled to approve this action, and record who did. The resume endpoint is a privileged operation — treat it like one.
6. **Idempotency on the acting tool.** Use an idempotency key so a duplicated resume cannot double-charge.
7. **Audit trail.** Who approved, when, with what modification. Checkpoints give you the state history; add explicit approver identity to state.
8. **Risk-tiered gating.** Interrupt only where it matters (dynamic interrupts), or approvals become rubber stamps.

**Deployment shape.** Because state is durable, the natural architecture is: an API endpoint that invokes/resumes, a queue for the actual graph execution, and an approvals UI or bot that calls the resume endpoint. Nothing holds a process open waiting for a human — which is the entire point of durable interrupts.

### Interview Follow-ups

- Why is a separate approval node better than `interrupt_before=["tools"]`? (Two reasons: it interrupts *only* for risky calls rather than every tool call, and it puts the interrupt in a node with no side effects, sidestepping the re-execution hazard.)
- How do you handle approval for one of several parallel tool calls? (Approve per `tool_call_id`, return the approved ids in state, and have the tool node execute only those — returning explicit "not approved" tool messages for the rest so the model knows.)

---

## Q18: What is time travel, and what is it useful for?

### Answer

**Time travel means fetching a historical checkpoint and resuming execution from it**, optionally with modified state. It follows directly from checkpointing every super-step.

```python
config = {"configurable": {"thread_id": "conv-123"}}

# Inspect the full history
for snapshot in app.get_state_history(config):
    print(snapshot.config["configurable"]["checkpoint_id"],
          snapshot.next,                     # which node(s) would run next
          len(snapshot.values["messages"]))

# Resume from a specific earlier checkpoint
past = {"configurable": {"thread_id": "conv-123", "checkpoint_id": "1ef2a..."}}
app.invoke(None, past)                       # re-runs forward from that point

# Fork: modify state at that point, then continue down a different path
app.update_state(past, {"retrieved_docs": better_docs})
app.invoke(None, past)                       # creates a new branch of history
```

**What it is useful for:**

| Use | How |
|---|---|
| **Debugging** | Inspect exact state at the step where behaviour diverged (`11-ai-agents.md` Q29) |
| **Replay after a fix** | Fix a prompt or tool, re-run from just before the failure instead of the whole task |
| **Counterfactuals** | "What if retrieval had returned these documents?" — edit state and continue |
| **Human correction mid-history** | Fix a wrong intermediate result and re-run downstream |
| **A/B comparison** | Fork the same checkpoint with two different configurations and compare outcomes |
| **Undo** | Roll back to before an unwanted action and take a different branch |
| **Eval harness** | Replay production traces against a new prompt or model version |

**Why this is genuinely valuable for agents specifically.** Agent bugs typically manifest several steps after their cause, and the run is non-deterministic so you cannot simply re-run and expect the same failure. Time travel lets you pin the exact state at the divergent step and iterate against it deterministically. That converts "run it 20 times and hope it reproduces" into a normal debugging loop — the single biggest productivity difference between debugging agents with and without state history.

**Key concepts:**
- **`checkpoint_id`** identifies a specific point in a thread's history.
- **`snapshot.next`** tells you which node would execute next from that checkpoint — essential for understanding where you are resuming.
- **Forking** — resuming from a historical checkpoint after `update_state` creates a *branch* rather than overwriting. The original history is preserved, which is what makes comparison possible.
- **Resume with `None` as input**, meaning "continue from the checkpoint" rather than "start with new input."

**Caveats:**
- **Side effects already happened.** Time travel rewinds *state*, not the world. If the agent already sent an email at step 4, rewinding to step 3 and re-running sends a second email. Guard external effects with idempotency keys, or use a sandbox for replay.
- **Storage cost.** Retaining full history for every thread is expensive; sample or apply retention limits, keeping full history for flagged or failed runs.
- **Non-determinism remains.** Replaying an LLM call gives a different response unless temperature is 0 and the model version is pinned. Time travel makes the *state* deterministic, not the model.

### Interview Follow-ups

- How is time travel different from resuming an interrupt? (Resuming an interrupt continues from the *latest* checkpoint, which is the normal flow. Time travel deliberately targets an *earlier* checkpoint, usually to inspect or fork.)
- How does this help evaluation? (Replay real production trajectories from step N with a changed prompt or model and compare downstream behaviour — a much stronger signal than synthetic test cases, because the state is real.)

---

## Q19: What is the difference between an edge and a conditional edge?

### Answer

| | Edge (static) | Conditional edge |
|---|---|---|
| **Next node** | Fixed at build time | Computed at runtime |
| **API** | `add_edge("a", "b")` | `add_conditional_edges("a", fn, path_map)` |
| **Reads state** | No | **Yes** |
| **Number of possible targets** | Exactly one | Two or more (or a dynamic list) |
| **Enables branching** | No | **Yes** |
| **Enables a terminating loop** | No | **Yes** |
| **In the diagram** | A solid arrow | Dashed branches from a decision point |

**Static edge: "after A, always do B."**

```python
graph.add_edge("retrieve", "generate")     # generation always follows retrieval
```

**Conditional edge: "after A, decide based on what happened."**

```python
graph.add_conditional_edges("validate", lambda s: "retry" if s["errors"] else "done",
                            {"retry": "generate", "done": END})
```

**The conceptual point: a conditional edge is where runtime information enters control flow.** A graph made only of static edges is a fixed pipeline — a chain. The moment you add a conditional edge whose outcome depends on model output, the system becomes capable of the branching and looping that makes it an agent. **The conditional edge is what turns a chain into an agent** (see `11-ai-agents.md` Q25).

**When to use which:**

```text
Is the next step always the same?
├── YES → static edge (simpler, self-documenting, cheaper)
└── NO  → conditional edge
```

Do not use a conditional edge that always returns the same value — it adds a function call and obscures the diagram for no benefit.

**Common conditional-edge mistakes:**

| Mistake | Consequence | Fix |
|---|---|---|
| Doing expensive work in the routing function | Slow, not checkpointed, not traceable | Compute in a node, write a flag, route on it |
| Returning an unmapped key | Runtime error | Use `Literal`/enum returns and always include a default |
| No path to `END` from some branch | Hangs or hits the recursion limit | Ensure every branch can terminate |
| Side effects in the routing function | Runs unpredictably, breaks replay | Keep routers pure |
| Cycle with no bounded exit | Infinite loop until `GraphRecursionError` | Attempt counter or budget in state (Q13) |

**A subtlety worth mentioning:** a conditional edge can return a **list** of targets, which fans out to parallel execution (Q23). So conditional edges express not just "which one" but also "how many" — including zero, which effectively skips a branch.

### Interview Follow-ups

- Can you replace all static edges with conditional ones? (Mechanically yes; practically no. Static edges document intent and are validated structurally. Using conditionals everywhere makes the graph unreadable and hides where real decisions occur.)
- Where should the decision logic live — the node or the edge? (Compute in the node, decide in the edge. The node writes a flag to state, which is checkpointed and traceable; the edge does a cheap read of that flag.)

---

## Q20: What is the difference between a node and an agent, and between a graph and an agent?

### Answer

These two conflations are among the most common in LangGraph interviews.

**Node vs agent:**

| | Node | Agent |
|---|---|---|
| **Definition** | A unit of work: `state -> update` | An LLM in a loop with tools, choosing its own actions |
| **Granularity** | One step | A whole loop, many steps |
| **Contains a loop** | No — a node runs once per visit | **Yes** — that is what defines it |
| **Relationship** | An agent is built **from** nodes and edges | An agent can be **wrapped as** a node |

Concretely: `call_model` is a node. `agent → tools → agent` with its conditional exit is an *agent*. And a compiled agent graph can be added as a single node inside a larger graph — which is exactly how multi-agent systems are built (Q22, Q26).

```python
# A node
graph.add_node("call_model", call_model)

# An agent, built from nodes + edges + a cycle
graph.add_edge(START, "call_model")
graph.add_conditional_edges("call_model", should_continue, {"tools": "tools", "end": END})
graph.add_edge("tools", "call_model")
research_agent = graph.compile()

# The agent used as a node in a bigger graph
supervisor.add_node("researcher", research_agent)
```

**Graph vs agent:**

| | Graph | Agent |
|---|---|---|
| **What it is** | A **structure**: nodes + edges + state | A **behaviour**: autonomous iterative decision-making |
| **Requires a cycle** | No | **Yes** |
| **Requires an LLM** | No | Yes |
| **Control flow** | Whatever you wired | Model-determined within the wiring |

**The relationship: a graph is the *mechanism*; an agent is a *pattern you can build with it*.** LangGraph is a graph framework, not an agent framework — which is why it can equally express:

```text
A chain:      START → retrieve → generate → END              (no cycle, not an agent)
A router:     START → classify → {a | b | c} → END           (branching, not an agent)
An agent:     START → model ⇄ tools → END                    (cycle, agent)
A workflow
with agents:  START → classify → {chain | agent} → verify → END   (hybrid)
```

That last shape — a mostly-deterministic workflow with agentic steps where needed — is what most production systems actually look like, and being able to express both in one framework is LangGraph's main argument for itself.

**Why the distinction matters practically.** If someone says "build an agent," and the task has a knowable sequence of steps, the right answer in LangGraph is a graph *without* a cycle — a chain. You get the state management, checkpointing, streaming, and HITL benefits without the unpredictability of a loop. Recognising that you can use the framework without building an agent is a mark of judgement (see `11-ai-agents.md` Q22 and Q25).

### Interview Follow-ups

- Is `create_react_agent` a graph? (Yes — it is a convenience constructor that builds and compiles the standard `agent ⇄ tools` graph. Useful to start with; you outgrow it as soon as you need custom state, guardrails, or approval gates, at which point you write the graph explicitly.)
- Can a graph contain zero LLM calls? (Yes, and that is sometimes useful — a purely deterministic graph still gets checkpointing, durability, streaming, and time travel. It just is not an agent.)

---

## Q21: How do you handle errors and retries in LangGraph?

### Answer

**Three layers, matching the general agent picture in `11-ai-agents.md` Q13:** node-level retries for transient failures, graph-level routing for semantic failures, and budget-based termination for pathological runs.

**1. Node-level retry policies — for transient infrastructure failures.**

```python
from langgraph.pregel import RetryPolicy

graph.add_node(
    "call_api",
    call_external_api,
    retry=RetryPolicy(
        max_attempts=3,
        initial_interval=0.5,
        backoff_factor=2.0,
        jitter=True,
        retry_on=(TimeoutError, ConnectionError, RateLimitError),
    ),
)
```

Retries happen **inside the node's execution**, transparent to the graph. Requirements:
- **Only retry idempotent operations.** Retrying a node that charges a card double-charges. Use idempotency keys, or move the non-idempotent call to its own node with no retry.
- **Be specific in `retry_on`.** Retrying a `ValidationError` wastes time — it will fail identically.
- **Bound attempts and use jitter** to avoid synchronised retry storms.

**2. Graph-level error handling — for failures the model can act on.**

Catch the error in the node, write it to state, and route on it:

```python
def call_api(state: State) -> dict:
    try:
        return {"api_result": external_api(state["query"]), "api_error": None}
    except PermanentAPIError as e:
        return {"api_error": str(e), "api_result": None}

def route_after_api(state: State) -> str:
    if state["api_error"]:
        return "fallback" if state["attempts"] < 2 else "escalate"
    return "continue"

graph.add_conditional_edges("call_api", route_after_api, {
    "fallback": "alternative_source",
    "escalate": "human_handoff",
    "continue": "generate",
})
```

**This pattern — errors as state, recovery as routing — is the idiomatic LangGraph approach.** It makes the error path visible in the graph diagram, checkpointed, and traceable, rather than buried in exception handling.

**3. Model-visible errors — let the model recover.**

For tool failures, return the error as a `ToolMessage` so the model can correct itself:

```python
try:
    result = tool.invoke(args)
except Exception as e:
    result = (f"Error: {e}. Verify the arguments against the tool schema and retry, "
              f"or use a different approach.")
messages.append(ToolMessage(content=str(result), tool_call_id=call["id"]))
```

`ToolNode` does this by default. Actionable error text turns a dead end into a recovered step.

**4. Budget-based termination — for pathological runs.**

```python
def check_budget(state: State) -> str:
    if state["iterations"] >= 12: return "exhausted"
    if state["tokens_used"] > 100_000: return "exhausted"
    if state["consecutive_errors"] >= 3: return "exhausted"
    if state["stuck"]: return "exhausted"
    return "continue"
```

Route `"exhausted"` to a node that **summarises partial progress and escalates** — never a bare exception. `recursion_limit` remains the backstop, but it produces a crash rather than a graceful outcome (Q13).

**5. Crash recovery is free with a checkpointer.** If the process dies mid-run, re-invoking with the same `thread_id` resumes from the last completed super-step. Note the granularity: the *super-step* is the unit of recovery, so a node that crashed halfway re-runs from its start — which is another reason nodes should be idempotent or narrowly scoped around side effects.

**Design summary:**

| Failure | Layer |
|---|---|
| Timeout, 429, 5xx | `RetryPolicy` on the node |
| Permanent API error | Catch → state → conditional edge |
| Bad tool arguments | `ToolMessage` error → model corrects |
| Repeated no-progress | Budget check → graceful exit |
| Process crash | Checkpointer resume |
| Unbounded cycle | `recursion_limit` (last resort) |

### Interview Follow-ups

- Why not wrap everything in try/except inside nodes? (You can, but then error handling is invisible in the graph. Errors-as-state plus routing makes recovery paths explicit, reviewable, and testable — and they show up in the rendered diagram.)
- What is the risk of node retries with a checkpointer? (Retries occur within one super-step, so no checkpoint is written between attempts. A node with side effects that partially completes will redo them on retry — make such nodes idempotent or split them.)

---

## Q22: What are subgraphs, and when should you use them?

### Answer

**A subgraph is a compiled graph used as a node inside another graph.** Because a compiled graph implements the runnable interface, this composes naturally.

```python
# Build and compile the inner graph
research = StateGraph(ResearchState)
research.add_node("search", search_node)
research.add_node("summarise", summarise_node)
research.add_edge(START, "search")
research.add_edge("search", "summarise")
research.add_edge("summarise", END)
research_graph = research.compile()

# Use it as a node in the parent
parent = StateGraph(ParentState)
parent.add_node("research", research_graph)        # a whole graph as one node
parent.add_node("write", write_node)
parent.add_edge(START, "research")
parent.add_edge("research", "write")
parent.add_edge("write", END)
```

**State sharing — the crux of subgraph design.** Two cases:

**1. Shared schema keys — state flows automatically.** If parent and subgraph share state keys, those keys pass in and updates pass out with no glue code. Simple, but couples the two schemas.

**2. Different schemas — wrap with explicit translation.** Preferable for genuine encapsulation:

```python
def research_node(state: ParentState) -> dict:
    result = research_graph.invoke({
        "query": state["research_question"],       # parent → subgraph
        "max_sources": 5,
    })
    return {"research_summary": result["summary"], # subgraph → parent
            "sources": result["sources"]}
```

This makes the interface explicit and lets the subgraph evolve independently — the same argument as a well-defined function signature.

**When to use subgraphs:**

| Reason | Detail |
|---|---|
| **Reuse** | The same sub-workflow used by several parents |
| **Team ownership** | Different teams own different subgraphs behind a stable interface |
| **Encapsulation** | Hide internal complexity; the parent sees one node |
| **Multi-agent** | Each agent is a subgraph with its own state and tools (Q26) |
| **Independent testing** | Test the subgraph in isolation |
| **Independent configuration** | Different checkpointing, retry policy, or recursion limit per subgraph |
| **Separate state schemas** | The subgraph needs fields the parent should not carry |

**When *not* to use them:**
- Just to group nodes visually — that adds indirection without benefit.
- When the sub-flow is three simple nodes — inline them.
- When the state translation is more code than the subgraph saves.

**Gotchas that matter in practice:**

1. **Streaming needs `subgraphs=True`.** Otherwise inner node events are invisible, and your progress UI goes silent inside the subgraph. This surprises people refactoring a working graph.
2. **Checkpointing is nested.** Subgraph checkpoints live under the parent thread's namespace. Interrupting *inside* a subgraph works, and resuming targets the parent thread — but be deliberate about it, because the resume semantics are easy to get wrong.
3. **Recursion limits multiply.** A subgraph with a cycle inside a parent cycle can blow through step budgets fast. Budget each level separately.
4. **Debugging depth.** A failure three levels down is harder to trace. Keep nesting shallow — one or two levels in practice.
5. **Information loss at the boundary.** If the subgraph returns only a summary, the parent cannot recover the detail — the same lossy-handoff problem as sub-agents (`11-ai-agents.md` Q11). Make the return contract explicit and sufficient.

### Interview Follow-ups

- Subgraph vs a plain function call inside a node? (Use a subgraph when you need its own checkpointing, streaming, interrupts, cycles, or independent state. A plain function is simpler for straight-line logic and should be preferred when it suffices.)
- How deep should nesting go? (One or two levels. Deeper multiplies both information loss and debugging difficulty for little architectural gain.)

---

## Q23: How does parallel execution work?

### Answer

**Nodes with no dependency between them run concurrently within the same super-step.** You get parallelism by fanning out edges from one node to several.

```python
# Fan out
graph.add_edge("plan", "search_docs")
graph.add_edge("plan", "search_tickets")
graph.add_edge("plan", "search_code")

# Fan in — all three must complete before "synthesise" runs
graph.add_edge("search_docs", "synthesise")
graph.add_edge("search_tickets", "synthesise")
graph.add_edge("search_code", "synthesise")
```

```text
              ┌─→ search_docs ────┐
plan ─────────┼─→ search_tickets ─┼─→ synthesise
              └─→ search_code ────┘
```

**Execution semantics:**
1. All nodes triggered in a super-step run concurrently (truly concurrent for async nodes; threaded for sync ones).
2. When all complete, their state updates are **merged via reducers**.
3. The merged state is checkpointed.
4. The next super-step is determined.

**Reducers are mandatory for keys written by multiple parallel branches** (Q11):

```python
class State(TypedDict):
    docs: Annotated[list, operator.add]        # merges cleanly
    sources_searched: Annotated[set, operator.or_]
    status: str                                # only ONE branch may write this
```

Writing `status` from two parallel branches raises `InvalidUpdateError` — LangGraph detects the conflict rather than silently losing an update.

**Conditional fan-out — the number of branches decided at runtime:**

```python
def pick_sources(state: State) -> list[str]:
    return [s for s in state["candidate_sources"] if state["relevance"][s] > 0.5]

graph.add_conditional_edges("plan", pick_sources, {
    "docs": "search_docs", "tickets": "search_tickets", "code": "search_code",
})
```

**Dynamic fan-out with `Send` — N instances of the same node, one per item.** This is the map-reduce pattern and the most powerful parallel construct:

```python
from langgraph.types import Send

def fan_out_documents(state: State) -> list[Send]:
    return [Send("analyse_document", {"doc": d}) for d in state["documents"]]

graph.add_conditional_edges("load", fan_out_documents, ["analyse_document"])
graph.add_edge("analyse_document", "aggregate")

def analyse_document(payload: dict) -> dict:
    return {"analyses": [llm.invoke(f"Analyse: {payload['doc']}")]}   # list reducer
```

`Send` differs from ordinary fan-out in that each instance receives its **own payload** rather than the shared state, so you can process 40 documents in parallel with one node definition. Use it for: analysing N documents, evaluating N candidates, querying N sources, generating N variations.

**What you gain and what it costs:**

| Benefit | Cost / risk |
|---|---|
| Wall-clock reduction proportional to fan-out | Concurrent API calls → rate limits |
| Independent work genuinely overlaps | Cost is incurred for every branch even if one suffices |
| Multiple perspectives combined | Branches **cannot see each other's results** |
| — | Reducer design becomes mandatory |
| — | Partial failure semantics need deciding |
| — | Interrupts inside parallel branches are hazardous (Q16) |

**Important constraints:**
- **Parallel branches cannot coordinate.** If branch B needs branch A's result, they must be sequential. This is the same limitation as parallel sub-agents.
- **Failure handling.** By default a failing branch fails the super-step. If you want partial results, catch errors inside each node and return them as state (Q21).
- **Rate limits are the practical ceiling.** Fanning out to 100 LLM calls hits provider limits; bound the concurrency.
- **Non-deterministic merge order.** With `operator.add`, output order depends on completion order. Sort explicitly if order matters.

### Interview Follow-ups

- `Send` vs multiple static edges? (Static edges fan out to *different, named* nodes. `Send` fans out to *N instances of one* node, each with its own payload, with N determined at runtime. `Send` is for map-reduce over a collection.)
- How do you limit concurrency? (`max_concurrency` in the config, plus a semaphore or rate limiter in the node. Necessary for large `Send` fan-outs against a rate-limited API.)

---

## Advanced

---

## Q24: How do you build a multi-agent system in LangGraph?

### Answer

**Each agent is a subgraph (or a node); coordination is expressed as the parent graph's edges.** The topology choice matters more than the implementation.

**Supervisor pattern — the most common and most controllable:**

```python
class SupervisorState(TypedDict):
    messages: Annotated[list, add_messages]
    next_agent: str
    findings: Annotated[list, operator.add]
    delegations: int

def supervisor(state: SupervisorState) -> dict:
    decision = supervisor_llm.with_structured_output(Decision).invoke([
        SystemMessage(content=(
            "You coordinate specialists. Choose the next one, or FINISH.\n"
            "researcher: gathers information\n"
            "analyst: analyses data and computes\n"
            "writer: produces the final document\n"
        )),
        *state["messages"],
    ])
    return {"next_agent": decision.next, "delegations": state["delegations"] + 1,
            "messages": [AIMessage(content=f"Delegating to {decision.next}: {decision.task}")]}

def route(state: SupervisorState) -> str:
    if state["delegations"] > 10:
        return "FINISH"                     # hard cap — supervisors can loop
    return state["next_agent"]

graph = StateGraph(SupervisorState)
graph.add_node("supervisor", supervisor)
graph.add_node("researcher", research_agent)     # compiled subgraphs
graph.add_node("analyst", analyst_agent)
graph.add_node("writer", writer_agent)

graph.add_edge(START, "supervisor")
graph.add_conditional_edges("supervisor", route, {
    "researcher": "researcher", "analyst": "analyst",
    "writer": "writer", "FINISH": END,
})
for agent in ["researcher", "analyst", "writer"]:
    graph.add_edge(agent, "supervisor")           # control always returns
```

```text
            ┌──────────→ researcher ──┐
START → supervisor ─────→ analyst ─────┼─→ (back to supervisor)
            │      └────→ writer ──────┘
            └──(FINISH)──→ END
```

**Other topologies:**

| Topology | Implementation |
|---|---|
| **Sequential pipeline** | Static edges: `research → analyse → write → END` |
| **Router** | One conditional edge from `START`, each agent goes straight to `END` |
| **Parallel + aggregate** | Fan out to N agents, fan in to a synthesis node with list reducers |
| **Hierarchical** | A supervisor whose "agents" are themselves supervisor subgraphs |
| **Network / swarm** | Each agent has a conditional edge to every other — use `Command(goto=...)` for handoffs |

**Handoff with `Command` — an agent transferring control directly:**

```python
from langgraph.types import Command
from typing import Literal

def researcher(state: State) -> Command[Literal["analyst", "supervisor"]]:
    findings = do_research(state)
    if needs_analysis(findings):
        return Command(goto="analyst",
                       update={"findings": findings, "task": "analyse these"})
    return Command(goto="supervisor", update={"findings": findings})
```

`Command` combines a state update with a routing decision in one return value — useful for swarm handoffs, though it moves control flow *into* the node, which reduces the visibility that edges provide. Prefer edges when you can; use `Command` for genuine dynamic handoff.

**Design guidance — the part that shows judgement:**

1. **Start with a single agent.** Multi-agent multiplies cost 5–15× and makes debugging much harder (`11-ai-agents.md` Q12). Move to it only when you hit a specific wall: too many tools, context exhaustion, genuine parallelism, or different permission boundaries.
2. **Prefer supervisor over network.** A constrained topology is far easier to reason about, bound, and debug than "anyone can hand off to anyone."
3. **Cap delegations explicitly.** Supervisors loop — ping-ponging between two agents without progressing is the characteristic failure. A hard delegation counter is mandatory.
4. **Design the handoff contract.** What exactly does each agent receive and return? Vague handoffs are the primary source of multi-agent failure, because the receiving agent cannot ask for clarification.
5. **Scope permissions per agent.** A research agent gets read-only tools; only the writer can publish. This is a real benefit of the architecture — use it.
6. **Trace across agents.** Without cross-agent tracing, a wrong final answer is untraceable. Use `subgraphs=True` when streaming and tag events with the agent name.
7. **Decide the state-sharing model deliberately.** Full shared state is simple but couples agents and grows large; scoped subgraph state with explicit translation is cleaner but requires mapping code. Choose based on how independent the agents genuinely are.

### Interview Follow-ups

- Shared state or isolated state per agent? (Shared for tightly cooperating agents that need each other's intermediate work; isolated with explicit handoff contracts when agents are genuinely independent — it keeps contexts small and interfaces clear.)
- What is the characteristic multi-agent failure in LangGraph? (Supervisor ping-pong: two agents handing work back and forth without progressing until the delegation cap or recursion limit hits. Fix with a delegation counter, progress checks, and clearer task contracts.)

---

## Q25: How do you implement RAG in LangGraph, including corrective and self-correcting variants?

### Answer

**Basic RAG — a chain, not an agent (no cycle):**

```python
class RAGState(TypedDict):
    messages: Annotated[list, add_messages]
    question: str
    documents: list
    answer: str

graph.add_edge(START, "rewrite_query")
graph.add_edge("rewrite_query", "retrieve")
graph.add_edge("retrieve", "rerank")
graph.add_edge("rerank", "generate")
graph.add_edge("generate", END)
```

Even with no cycle, LangGraph buys you checkpointing (multi-turn state), streaming, and observability over a hand-written pipeline.

**Corrective RAG (CRAG) — grade the documents and act on the grade:**

```python
def grade_documents(state: RAGState) -> dict:
    grader = llm.with_structured_output(Grade)
    kept = [d for d in state["documents"]
            if grader.invoke(f"Q: {state['question']}\nDoc: {d.text}\nRelevant?").relevant]
    return {"documents": kept, "enough_context": len(kept) >= 2}

def decide(state: RAGState) -> str:
    if state["enough_context"]:
        return "generate"
    if state["rewrites"] < 2:
        return "rewrite"          # try a different query formulation
    return "web_search"           # fall back to an external source

graph.add_conditional_edges("grade", decide, {
    "generate": "generate", "rewrite": "rewrite_query", "web_search": "web_search",
})
graph.add_edge("rewrite_query", "retrieve")     # the corrective cycle
```

**Self-RAG — additionally verify the generated answer:**

```python
def check_grounding(state: RAGState) -> str:
    verdict = grounding_checker.invoke({"answer": state["answer"],
                                        "documents": state["documents"]})
    if not verdict.grounded:
        return "regenerate" if state["attempts"] < 2 else "refuse"
    if not verdict.answers_question:
        return "retrieve_more" if state["attempts"] < 2 else "refuse"
    return "done"

graph.add_conditional_edges("verify", check_grounding, {
    "regenerate": "generate",
    "retrieve_more": "rewrite_query",
    "refuse": "say_cannot_answer",
    "done": END,
})
```

**Agentic RAG — retrieval as a tool, model decides:**

```python
retriever_tool = create_retriever_tool(
    retriever, name="search_knowledge_base",
    description=("Search internal documentation. Use for questions about company "
                 "policies, products, or procedures. Not for the user's account data."))

graph.add_node("agent", lambda s: {"messages": [llm.bind_tools([retriever_tool]).invoke(s["messages"])]})
graph.add_node("tools", ToolNode([retriever_tool]))
graph.add_conditional_edges("agent", should_continue, {"tools": "tools", "end": END})
graph.add_edge("tools", "agent")
```

**Choosing among them:**

| Variant | Cycle | Extra LLM calls | Use when |
|---|---|---|---|
| Basic | No | 0 | Retrieval quality is already good — **the default** |
| Corrective | Retrieval loop | 1 per document + retries | Retrieval sometimes misses |
| Self-RAG | Retrieval + generation loops | 2–4 | Groundedness is critical |
| Agentic | Model-driven | Variable | Multi-hop, multi-source, conditional retrieval |

**The engineering advice:** every added cycle multiplies cost and latency. Measure first (`10-rag.md` Q13) — if 90% of queries retrieve well, adding grading to all of them pays 10× the cost to fix 10% of cases. Better: classify upfront and route only hard queries into the corrective path. This is the classify-then-escalate pattern from `10-rag.md` Q20, and LangGraph's conditional edges express it directly.

**LangGraph-specific points worth making:**
- Keep `documents` in state so the generation node, the verification node, and the citation formatter all see the same evidence.
- Store chunk ids alongside text so citations survive the whole graph.
- Put tenant/ACL filters in **config**, not state, and read them in the retrieval node — the model must never influence them (`10-rag.md` Q22).
- Bound every cycle with a rewrite/attempt counter, and route exhaustion to an explicit refusal node rather than a low-quality answer.

### Interview Follow-ups

- Why is `documents` a separate state key rather than just messages? (So non-generation nodes — graders, verifiers, citation formatters — can access structured evidence with ids and scores, which would be lost if it were flattened into message text.)
- Where do you put the refusal path? (An explicit terminal node reached when grading or grounding checks fail after N attempts. Making refusal a first-class node makes it testable and measurable, rather than hoping the prompt produces it.)

---

## Q26: How do you test a LangGraph application?

### Answer

**Test at four levels, from cheapest to most expensive.**

**1. Node unit tests — the highest-value tests.** A node is just `state -> dict`, so test it directly with no graph:

```python
def test_grade_documents_filters_irrelevant():
    state = {"question": "What is our refund policy?",
             "documents": [Doc("Refunds within 30 days"), Doc("Office opening hours")]}
    with patch("app.grader", FakeGrader(relevant_for=["Refunds within 30 days"])):
        result = grade_documents(state)
    assert len(result["documents"]) == 1
    assert result["enough_context"] is False        # only 1 doc, threshold is 2
```

Nodes that contain no LLM call — validation, budgeting, compression, formatting — should be tested exhaustively. They are ordinary functions and are where a surprising share of bugs live.

**2. Routing function tests — cheap and catch real bugs.** Routers are pure functions of state:

```python
@pytest.mark.parametrize("state,expected", [
    ({"iterations": 3, "messages": [AIMessage("done")]},                  "end"),
    ({"iterations": 3, "messages": [AIMessage("", tool_calls=[{...}])]},  "tools"),
    ({"iterations": 15, "messages": [AIMessage("", tool_calls=[{...}])]}, "budget_exceeded"),
])
def test_should_continue(state, expected):
    assert should_continue(state) == expected
```

Test the boundary cases and the exhaustion paths specifically — those are the branches that only fire in production incidents.

**3. Graph structure tests — cheap, no LLM required.**

```python
def test_every_conditional_branch_can_terminate():
    g = app.get_graph()
    assert "__end__" in {e.target for e in g.edges}
    # assert all nodes reachable from START; assert no node has zero outgoing edges

def test_graph_renders():
    app.get_graph().draw_mermaid()          # catches structural errors at build time
```

Compilation already validates a lot; add assertions for your own invariants — e.g. that every risky tool is behind the approval node.

**4. Integration tests with fake models — the workhorse.** Substitute a scripted model so trajectories are deterministic:

```python
class ScriptedModel:
    def __init__(self, responses): self.responses, self.i = responses, 0
    def invoke(self, messages, **kw):
        r = self.responses[self.i]; self.i += 1; return r
    def bind_tools(self, tools): return self

def test_agent_uses_tool_then_answers():
    model = ScriptedModel([
        AIMessage("", tool_calls=[{"name": "get_order", "args": {"order_id": "ORD-1"}, "id": "t1"}]),
        AIMessage("Your order shipped on Tuesday."),
    ])
    app = build_graph(model=model, tools=[fake_get_order]).compile(checkpointer=MemorySaver())
    result = app.invoke({"messages": [HumanMessage("Where is ORD-1?")]},
                        {"configurable": {"thread_id": "t"}})
    assert "shipped" in result["messages"][-1].content
    assert fake_get_order.called_with == {"order_id": "ORD-1", "tenant_id": "acme"}
```

This tests **your graph logic** — routing, state merging, tool wiring, argument injection — without model non-determinism or cost. Most graph bugs are found here.

**Specific things to test that teams routinely miss:**

| Test | Why |
|---|---|
| **Interrupt and resume** | Invoke → assert interrupted → resume → assert continued. The resume path is where HITL bugs hide. |
| **Rejection path** | Resume with `approved: False` and assert the graph handles it sensibly. |
| **No double side effects on resume** | Assert the acting function was called exactly once across interrupt + resume (Q16). |
| **Budget exhaustion** | Force the cap and assert a graceful partial result, not an exception. |
| **Parallel merge** | Two branches writing the same key merge as intended; conflicting writes raise. |
| **Reducers** | Test each custom reducer directly, including `None` current value and order-independence. |
| **Message trimming** | Assert no orphaned tool calls after trimming — the classic API-rejection bug (Q10). |
| **Tenant isolation** | Assert retrieval filters always include the tenant from config, even with adversarial state. |
| **Checkpoint resume** | Invoke, discard the app object, rebuild, resume the same `thread_id`, assert continuity. |

**5. End-to-end evaluation with real models.** Necessary but slow, costly, and non-deterministic. Keep it as a versioned eval set (`11-ai-agents.md` Q20) run on merge rather than on every commit, with a small smoke subset in the fast path. Run each case several times and report distributions.

**Test infrastructure worth building early:** a `build_graph(model=..., tools=...)` factory with injectable dependencies, fake tools that record their calls, and a helper that runs a graph and returns the full trajectory for assertion. Without dependency injection, LangGraph apps become effectively untestable — this is the single most important structural decision for testability.

### Interview Follow-ups

- How do you make a graph testable? (Dependency injection at construction: pass the model, tools, retriever, and store into a factory. Never instantiate them at module scope inside nodes.)
- How do you test non-deterministic behaviour? (Separate concerns: test graph mechanics with fake models deterministically, and test model *quality* statistically with an eval set over repeated runs. Conflating the two gives flaky tests that people disable.)

---

## Q27: How do you deploy and scale a LangGraph application?

### Answer

**The key architectural property: with a checkpointer, the graph is stateless at the process level.** All conversation state lives in the checkpoint store, so any worker can serve any request. That makes horizontal scaling straightforward — and it is the main reason to use a durable checkpointer even when you do not need HITL.

**A typical production topology:**

```text
Client
  ↓ HTTP / WebSocket / SSE
API layer (FastAPI)                     stateless, N replicas
  ├── POST /chat        → enqueue or invoke, stream results
  ├── POST /resume      → resume an interrupted thread (authorised!)
  └── GET  /threads/:id → fetch state for the UI
  ↓
Execution layer                          stateless workers, N replicas
  └── compiled graph (built once at startup)
  ↓
Postgres  ← checkpoints + store (long-term memory)
Redis     ← per-thread locks, rate limiting, pub/sub for streaming
Vector DB ← retrieval
Tracing   ← LangSmith / OTel
```

**Deployment decisions:**

| Concern | Guidance |
|---|---|
| **Compile once** | Build and compile at application startup, not per request. Compilation validates structure and is wasted work per request. |
| **Sync vs async** | Async nodes and `astream` throughout. Agent work is IO-bound; sync nodes in an ASGI app block the event loop. |
| **Short vs long tasks** | Sub-30s: handle in-request with streaming. Longer or interruptible: enqueue to a worker and stream progress via pub/sub. |
| **Checkpointer** | Postgres with connection pooling. Size the pool for checkpoint writes on **every super-step** — this is a real write load. |
| **Per-thread concurrency** | Lock per `thread_id` (Redis). Two concurrent invocations on one thread corrupt state. |
| **Streaming transport** | SSE is simplest; WebSocket if you need bidirectional (e.g. approvals in-stream). Disable proxy buffering. |
| **Timeouts** | Per-node timeouts, a whole-graph deadline, and a `recursion_limit` well above your state-based caps. |
| **Graceful shutdown** | Drain in-flight requests. Checkpointing means an interrupted run resumes rather than being lost — a significant operational advantage. |

**Scaling considerations:**

1. **The checkpoint store is the bottleneck before the models are.** Every super-step is a write. Reduce it by keeping state small (references, not blobs), trimming message history, and applying a retention policy. Monitor checkpoint size and write latency as first-class metrics.
2. **Model rate limits are the real ceiling.** Implement per-tenant rate limiting and queueing, and handle 429s with backoff (Q21). Parallel fan-out multiplies this pressure.
3. **Cost per conversation must be attributed.** Track tokens per node per thread and tag by tenant. Without this you cannot find the runaway task types.
4. **Retention and privacy.** Checkpoints contain full conversation content. Implement deletion by user, retention windows, and encryption at rest.
5. **Warm-up.** First requests pay import and connection-pool costs. Pre-warm on startup.

**Observability — non-negotiable:**

| Signal | Why |
|---|---|
| Per-node latency and error rate | Find the slow or flaky step |
| Tokens and cost per node per thread | Attribute spend; find runaway loops |
| Super-steps per conversation | Distribution reveals loops |
| Interrupt rate and time-to-approval | HITL health |
| Checkpoint size and write latency | The scaling canary |
| Budget-exhaustion and recursion-limit rate | Reliability signal |
| Full traces (LangSmith/OTel) | Non-negotiable for debugging (`11-ai-agents.md` Q29) |

**Versioning — the underrated problem.** Graph structure changes while threads are mid-flight. A thread checkpointed under v1 may resume under v2 with a different state schema or node set. Mitigations: version the state schema and migrate on load, keep the old graph deployed until in-flight threads drain, or refuse to resume threads created under an incompatible version with a clear message. Deciding this *before* your first breaking change saves an incident.

**On LangGraph Platform:** the managed option provides the API layer, persistence, queueing, streaming, and a threads UI. Worth it if you want to skip the plumbing; self-hosting is straightforward given the stateless property, and gives you full control over the data path — often the deciding factor in regulated environments.

### Interview Follow-ups

- How do you handle a task that takes 10 minutes? (Enqueue to a worker, return a thread id immediately, stream progress via pub/sub or let the client poll `get_state`. Never hold an HTTP request open for 10 minutes.)
- What breaks first as you scale? (Checkpoint write volume and size, then model rate limits. Both are addressed by keeping state small and bounding step counts — the same discipline as context engineering.)

---

## Q28: How do you implement guardrails and authorisation in LangGraph?

### Answer

The layers from `11-ai-agents.md` Q14 map onto specific LangGraph mechanisms:

| Layer | LangGraph mechanism |
|---|---|
| Input guardrails | A validation node right after `START`, with a conditional edge to a rejection node |
| Tool availability | Bind only the permitted tools, chosen from config at graph construction or in the model node |
| **Tool authorisation** | A **custom tool node** that checks each call before executing — the real boundary |
| Argument injection | Inject from `config["configurable"]`, never from state |
| Human approval | `interrupt()` in an approval node before the tool node (Q17) |
| Result filtering | Redact/truncate inside the tool node before creating `ToolMessage`s |
| Output guardrails | A validation node before `END`, with a revision cycle or a refusal path |
| Budget limits | Counters in state + a conditional edge (Q13) |

**Input guardrail as a node:**

```python
def screen_input(state: State) -> dict:
    text = state["messages"][-1].content
    issues = []
    if contains_pii(text):            issues.append("pii")
    if injection_score(text) > 0.8:   issues.append("injection")
    if off_topic(text):               issues.append("off_topic")
    return {"input_issues": issues}

graph.add_edge(START, "screen_input")
graph.add_conditional_edges("screen_input",
    lambda s: "reject" if s["input_issues"] else "proceed",
    {"reject": "refuse", "proceed": "agent"})
```

**Tool authorisation — the critical layer.** Never rely on the prompt:

```python
PERMITTED = {"support_agent": {"search_kb", "get_order", "issue_refund"},
             "readonly":      {"search_kb", "get_order"}}

def guarded_tool_node(state: State, config) -> dict:
    cfg = config["configurable"]
    role, tenant = cfg["role"], cfg["tenant_id"]     # from the authenticated session
    out = []

    for call in state["messages"][-1].tool_calls:
        if call["name"] not in PERMITTED[role]:
            out.append(ToolMessage(content=f"Permission denied: {call['name']}.",
                                   tool_call_id=call["id"]))
            continue

        args = {**call["args"], "tenant_id": tenant}   # inject — never model-supplied

        if call["name"] == "issue_refund" and args["amount"] > 50:
            if call["id"] not in state.get("approved_calls", []):
                out.append(ToolMessage(
                    content="Refunds over $50 require approval. Request it first.",
                    tool_call_id=call["id"]))
                continue

        try:
            result = TOOLS[call["name"]].invoke(args)
        except Exception as e:
            result = f"Error: {e}. Check the arguments and retry."

        out.append(ToolMessage(content=cap_size(redact_secrets(str(result))),
                               tool_call_id=call["id"]))
    return {"messages": out}
```

**Three points this code makes, and each is worth stating explicitly in an interview:**

1. **Authorisation reads from `config`, not `state`.** State is influenced by model output and is checkpointed; config is request-scoped and set by your authenticated API layer. Putting `role` or `tenant_id` in state would let a prompt injection potentially alter it.
2. **Arguments are injected, not validated.** The model cannot supply `tenant_id` at all, so cross-tenant access is unrepresentable rather than merely forbidden.
3. **Denials return `ToolMessage`s, not exceptions.** The model sees "permission denied", explains it to the user, and moves on — graceful degradation instead of a crash.

**Output guardrail with a bounded revision cycle:**

```python
def check_output(state: State) -> dict:
    draft = state["messages"][-1].content
    return {"output_issues": policy_check(draft) + groundedness_check(draft, state["documents"])}

graph.add_conditional_edges("check_output",
    lambda s: ("revise" if s["output_issues"] and s["revisions"] < 2
               else "refuse" if s["output_issues"] else "done"),
    {"revise": "generate", "refuse": "safe_refusal", "done": END})
```

**Why doing this in the graph is genuinely better than ad-hoc code.** Every guardrail becomes a **visible node with visible edges**: reviewable in the rendered diagram, unit-testable as a function, checkpointed so a rejection is auditable, and traceable in production. Compliance review of a diagram showing "all writes pass through an approval node" is a far easier conversation than reading through exception handlers.

**And the honest caveat:** none of this stops the model from being *persuaded* to try something — it stops the attempt from *succeeding*. That distinction is the whole point of enforcing in code rather than in prompts.

### Interview Follow-ups

- Why not put role in state? (State is persisted, visible to the model, and updatable by nodes — so it is influenceable. Config is set once by your authenticated API layer per request and is the only trustworthy source for authority.)
- How do you prove guardrails cannot be bypassed? (Structural tests: assert that the only path from the model node to any write tool passes through the authorisation node, and that the graph has no edge bypassing it. Combined with the rendered diagram, this is auditable.)

---

## Q29: When should you not use LangGraph?

### Answer

**LangGraph adds concepts — state schemas, reducers, nodes, edges, checkpointers, super-steps. That cost is worth paying only for specific benefits.**

**Do not use it when:**

| Situation | Use instead |
|---|---|
| **A single LLM call** | The provider SDK directly |
| **A two- or three-step linear chain with no persistence need** | Plain functions |
| **A simple tool loop, no HITL, no durability** | A 15-line `while` loop |
| **Latency-critical, sub-100ms budget** | Direct calls; skip the framework overhead |
| **Pure batch processing** | A data pipeline tool — you do not need conversational state |
| **The team will not learn it** | The simplest thing they will maintain correctly |

**The honest test — do you need at least one of these?**

```text
Durable, resumable state across turns or crashes?
Human-in-the-loop with pauses measured in minutes or hours?
Cycles with non-trivial exit conditions?
Runtime branching between several substantial sub-flows?
Parallel execution with state merging?
Streaming of intermediate progress, not just final tokens?
Time travel / replay for debugging or evaluation?
Multi-agent coordination with shared state?

None of the above → you probably do not need LangGraph.
```

**The costs, stated plainly:**
- **Conceptual overhead.** Reducers, super-steps, and checkpoint namespaces are real things to learn, and mistakes in them produce confusing bugs.
- **Debugging indirection.** A stack trace through the graph executor is less direct than through your own loop.
- **State design as a required activity.** You must think about the schema upfront; getting it wrong is painful to change once threads are checkpointed.
- **Framework coupling.** Versioning, breaking changes, and mid-flight thread migrations become your problem (Q27).
- **Over-engineering risk.** It is easy to build a 12-node graph for something three functions would do, because the framework makes nodes cheap to add.

**The counter-argument worth acknowledging.** Requirements grow. A project that starts as "a simple tool loop" frequently acquires a need for approvals, multi-turn state, or resumption within a few months, and retrofitting durable state onto a hand-rolled loop is genuinely painful. If you can see those requirements coming, starting with LangGraph is a defensible bet.

**The balanced position to give in an interview:** LangGraph is an excellent fit for **stateful, long-running, human-interactive, or cyclic** LLM applications, and overkill for **stateless, linear, single-shot** ones. The mature answer is not "always use it" or "never use it" — it is naming the specific capability you need it for. If you cannot name one, do not use it.

### Interview Follow-ups

- What is the migration path from a hand-rolled loop? (Usually straightforward: your loop body becomes the model node, your tool dispatch becomes the tool node, your exit condition becomes the conditional edge. Add the checkpointer and you have durability. It is often a one-day change — which is also an argument for not adopting it prematurely.)
- Is LangGraph the only option? (No — there are other orchestration frameworks and provider-native agent loops. The concepts here (explicit state, graph control flow, durable checkpoints, interrupts) transfer; the API does not. Interviews mostly test the concepts.)

---

## Q30: Design a document-processing pipeline in LangGraph. Walk through the graph.

### Answer

**Requirement:** ingest uploaded documents, extract structured data, validate it, route low-confidence extractions to human review, and write approved results to a database. Handle 5,000 documents/day, mixed PDFs and images, with an audit trail.

**Graph:**

```text
START
  ↓
classify_document          → type, page count, has_tables, is_scanned
  ↓ (conditional)
  ├── scanned → ocr ──────┐
  ├── digital → parse ────┤
  └── unsupported → reject_and_notify → END
                          ↓
                    chunk_and_layout      (structure-aware, per 03/10 guidance)
                          ↓
                  ┌── Send fan-out per section ──┐
                  ↓                              ↓
            extract_section  ...          extract_section     (N parallel instances)
                  └────────── merge (reducer) ───┘
                          ↓
                    validate_extraction    → schema, business rules, cross-field checks
                          ↓ (conditional)
  ┌── invalid & attempts<2 → re_extract_with_feedback ──┐ (cycle)
  ├── low_confidence      → human_review (interrupt)    │
  ├── valid               → persist                     │
  └── invalid & exhausted → quarantine → END            │
                          ↓                             │
                       persist  ←───────────────────────┘
                          ↓
                    emit_audit_record
                          ↓
                         END
```

**State:**

```python
class DocState(TypedDict):
    document_id: str
    raw_pages: list[str]                              # reference, not content, if large
    doc_type: str
    is_scanned: bool
    sections: list[dict]
    extractions: Annotated[list[dict], operator.add]  # parallel merge
    merged: dict
    validation_errors: list[str]
    confidence: float
    attempts: int
    review_decision: dict | None
    persisted_id: str | None
```

**Key design decisions and why:**

| Decision | Rationale |
|---|---|
| **`Send` fan-out per section** | Sections extract independently; parallelism turns a 40-page document from serial to ~one-page latency (Q23) |
| **`operator.add` on `extractions`** | The reducer makes parallel merging well-defined and order-tolerant |
| **Store page content externally, keep references in state** | Checkpoints are written every super-step; blobs would multiply storage and write latency (Q14) |
| **Validation as its own node, not inside extraction** | Lets us retry extraction with feedback, and lets validation be unit-tested without an LLM (Q26) |
| **Bounded re-extract cycle (`attempts < 2`)** | Retries fix genuine mistakes; unbounded retries burn cost on impossible documents (Q13) |
| **`interrupt()` for human review, not a static interrupt** | Only low-confidence documents pause — otherwise reviewers face 5,000 approvals/day (Q16) |
| **Interrupt node has no side effects before the interrupt** | Avoids the re-execution hazard on resume (Q16) |
| **Quarantine as a terminal node** | Failed documents are explicit and countable, not silently dropped |
| **Postgres checkpointer** | A document may sit in human review for days; state must be durable (Q14) |
| **`emit_audit_record` as a separate node** | Compliance requires an immutable record of what was extracted, by which version, approved by whom |
| **RetryPolicy on `ocr` and `parse`** | These call external services with transient failures; idempotent, so safe to retry (Q21) |

**The human review node:**

```python
def human_review(state: DocState) -> dict:
    decision = interrupt({
        "type": "extraction_review",
        "document_id": state["document_id"],
        "extracted": state["merged"],
        "confidence": state["confidence"],
        "flagged_fields": [k for k, v in state["merged"].items() if v.get("confidence", 1) < 0.7],
        "document_url": presigned_url(state["document_id"]),   # reviewer must see the source
    })
    if not decision["approved"]:
        return {"validation_errors": ["rejected by reviewer"], "review_decision": decision}
    return {"merged": decision.get("corrected", state["merged"]),
            "review_decision": decision}
```

Note that the payload includes a link to the source document — a reviewer cannot approve an extraction without seeing the original. This is the "show the human what they are approving" rule from `11-ai-agents.md` Q15.

**Throughput.** 5,000 documents/day is ~3.5/minute average, but arrival is bursty. Run graph invocations as queued worker jobs keyed by `document_id` as the `thread_id`; scale workers horizontally (statelessness makes this trivial); rate-limit model calls per worker pool; and bound `Send` fan-out concurrency so a 200-page document does not exhaust the model quota.

**What I would monitor:** extraction accuracy against a labelled sample, human-review rate (the key ROI metric — if it climbs, extraction is degrading), time-to-review, quarantine rate by document type, cost per document, and checkpoint size distribution.

**Why LangGraph is genuinely the right tool here** (per Q29): durable multi-day human review, a bounded retry cycle, dynamic parallel fan-out with state merging, and a required audit trail. Four of the criteria, not one — this is the shape of problem the framework exists for.

### Interview Follow-ups

- Why not a plain queue-and-worker pipeline? (You could, but you would hand-build durable state for the human-review pause, the retry cycle, the parallel merge, and the audit history. That is most of what the framework provides.)
- How would you handle a document type you have never seen? (Classify to `unsupported`, quarantine with a clear reason, and alert. Do not attempt extraction and produce plausible garbage — a wrong extraction that passes validation is far worse than a rejected document.)

---

## Q31: How do you migrate a hand-rolled agent to LangGraph, and what do you gain?

### Answer

**The mapping is mostly mechanical.**

```python
# BEFORE — hand-rolled
def run_agent(user_input, max_steps=10):
    messages = [SystemMessage(SYSTEM), HumanMessage(user_input)]
    for _ in range(max_steps):
        response = llm.bind_tools(tools).invoke(messages)
        messages.append(response)
        if not response.tool_calls:
            return response.content
        for call in response.tool_calls:
            result = TOOLS[call["name"]].invoke(call["args"])
            messages.append(ToolMessage(content=str(result), tool_call_id=call["id"]))
    return "Step limit reached."
```

```python
# AFTER — LangGraph
class State(TypedDict):
    messages: Annotated[list, add_messages]
    steps: int

def agent(state: State) -> dict:
    return {"messages": [llm.bind_tools(tools).invoke(state["messages"])],
            "steps": state["steps"] + 1}

def should_continue(state: State) -> str:
    if state["steps"] >= 10:
        return "limit"
    return "tools" if state["messages"][-1].tool_calls else "end"

g = StateGraph(State)
g.add_node("agent", agent)
g.add_node("tools", ToolNode(tools))
g.add_edge(START, "agent")
g.add_conditional_edges("agent", should_continue,
                        {"tools": "tools", "limit": "summarise_partial", "end": END})
g.add_edge("tools", "agent")
app = g.compile(checkpointer=PostgresSaver.from_conn_string(DB_URL))
```

**The mapping:**

| Hand-rolled | LangGraph |
|---|---|
| The `while`/`for` loop | The `agent → tools → agent` cycle |
| `if not response.tool_calls: return` | Conditional edge to `END` |
| `messages.append(...)` | `{"messages": [...]}` with `add_messages` |
| Tool dispatch loop | `ToolNode` |
| `max_steps` | Step counter in state + conditional edge (plus `recursion_limit`) |
| Local variables | State keys |
| Nothing | **Checkpointer** |

**What you gain, concretely:**

| Gain | Would have required (hand-rolled) |
|---|---|
| **Multi-turn continuity** | Your own serialisation, storage, and load logic |
| **Crash recovery** | Per-step persistence |
| **Human-in-the-loop** | Durable pause/resume with state serialisation — substantial work |
| **Time travel / replay** | A step-indexed state history |
| **Structured streaming** | An event bus threaded through every function |
| **Parallel execution with merge** | Concurrency plus your own conflict resolution |
| **Visualisable structure** | Nothing comparable — the loop is not introspectable |
| **Testable routing** | Extracting the exit condition into a pure function (you can do this anyway) |
| **Per-node retry policies** | Retry logic at each call site |

**What you lose:**
- Directness — a stack trace goes through the graph executor.
- ~40 lines of very readable code become a graph definition plus a state schema.
- A learning curve for anyone new to the codebase.

**The pragmatic migration sequence:**
1. **Port the loop as-is** to the canonical `agent ⇄ tools` graph. Verify behaviour is unchanged against your existing tests.
2. **Add the checkpointer.** Multi-turn continuity and crash recovery come for free.
3. **Extract the exit condition into a real router** with explicit branches for budget exhaustion, and add a graceful partial-result node.
4. **Add the approval node** where you need HITL, using dynamic interrupts.
5. **Split the model node** if you need guardrails, context compression, or memory loading as separate steps.
6. **Only then** consider parallelism, subgraphs, or multi-agent — and only against a measured need.

**The judgement to express:** do this migration when you have a *named requirement* the loop cannot serve — durability, approvals, replay. Migrating "because it is the proper framework" is how a 40-line function becomes a 12-node graph nobody can debug. And if your loop already does everything you need, leaving it alone is a legitimate engineering decision (Q29).

### Interview Follow-ups

- Can you migrate incrementally? (Yes — wrap the existing loop as a single node inside a LangGraph graph, gaining checkpointing and streaming at the boundary, then split it into nodes over time. A good de-risking path for a large codebase.)
- What is the first thing to get right? (The state schema. Everything else is edges and functions you can refactor cheaply; changing the schema after threads are checkpointed in production requires a migration.)

---

## Q32: Summarise the LangGraph vocabulary and the key distinctions.

### Answer

**The vocabulary:**

| Term | Definition |
|---|---|
| **StateGraph** | The builder: declare state schema, add nodes and edges, compile |
| **State** | The shared object flowing through the graph; nodes return partial updates |
| **Node** | A function `state -> dict` — can contain anything |
| **Edge** | Static control flow: after A, always B |
| **Conditional edge** | Runtime control flow: a function of state chooses the next node(s) |
| **START / END** | Sentinel entry and exit nodes |
| **Reducer** | How an update to a state key merges with the current value |
| **`add_messages`** | The message reducer: append, dedupe by id, handle `RemoveMessage` |
| **ToolNode** | Prebuilt node executing the model's requested tool calls |
| **Super-step** | One execution wave: all triggered nodes run, state merges, checkpoint written |
| **Checkpointer** | Persists state every super-step, keyed by `thread_id` |
| **Store** | Cross-thread key-value/semantic memory, read and written explicitly |
| **`thread_id`** | Identifies a conversation for checkpointing |
| **Interrupt** | A durable pause returning control to the caller |
| **`Command`** | A combined state update + routing decision returned from a node |
| **`Send`** | Dynamic fan-out to N instances of a node, each with its own payload |
| **Subgraph** | A compiled graph used as a node |
| **`recursion_limit`** | Backstop cap on super-steps |
| **Time travel** | Resuming from a historical checkpoint, optionally forked |

**The distinctions that get tested, each with its one-line discriminator:**

- **Node vs tool:** who invokes it — the *graph* (node) or the *model* (tool)?
- **Node vs agent:** a single unit of work, or a whole loop with model-driven decisions?
- **Graph vs agent:** a structure (nodes + edges), or a behaviour (autonomous iteration)? A graph without a cycle is not an agent.
- **Edge vs conditional edge:** fixed at build time, or computed from state at runtime? The conditional edge is what turns a chain into an agent.
- **State vs memory:** this thread's working data (checkpointed automatically), or cross-thread knowledge (a store, accessed explicitly)?
- **State vs config:** data the graph computes and mutates, or request-scoped input like `thread_id`, `user_id`, and role? **Security-relevant values belong in config.**
- **Checkpointer vs store:** automatic per-step thread state, or explicit cross-thread memory?
- **Interrupt vs recursion limit:** a deliberate durable pause, or a crash backstop?
- **`Send` vs fan-out edges:** N instances of *one* node with per-item payloads, or one instance each of *several different* nodes?
- **Reducer vs overwrite:** how concurrent and accumulating updates merge, versus last-write-wins.

**The unifying idea, and the best single thing to say about LangGraph in an interview:**

> LangGraph makes **state** and **control flow** explicit. Everything else follows from that. Because state is explicit and checkpointed every step, you get durability, resumption, human-in-the-loop, time travel, and streaming. Because control flow is explicit in edges, you get branching, cycles, parallelism, and a graph you can visualise, validate, and audit.

The framework's cost is the concepts; its value is that the two hardest parts of a stateful LLM application — what information exists, and what happens next — stop being implicit in a `while` loop and become things you can inspect, test, and reason about.

### Interview Follow-ups

- What is the single most important design decision in a LangGraph app? (The state schema. It is the contract between every node, it determines what reducers you need, it drives checkpoint size, and it is the expensive thing to change after deployment.)
- If you could only enforce one practice on a team using LangGraph? (Keep control flow in edges and work in nodes. The moment nodes start deciding what runs next, you lose the visualisability, testability, and auditability that were the reason to adopt a graph framework at all.)

---
