# LLM System Design

Designing, serving, evaluating, and operating LLM systems in production: architecture, cost and latency engineering, evaluation, safety, and the full-system design questions that close out senior interviews.

**Questions:** 18

This file assumes the model internals from `06-transformers-llms-generative-ai.md`, retrieval from `09`–`10`, and agent architecture from `11`–`12`. Here we put them together at system scale.

---

## Easy

---

## Q1: How do you approach an LLM system design interview question?

### Answer

**The structure is the same as classical system design, with LLM-specific concerns layered in.** Use a consistent framework so you never freeze on a blank problem.

**1. Clarify requirements (3–5 minutes — do not skip this).**

| Dimension | Questions to ask |
|---|---|
| **Functional** | What exactly must it do? What is out of scope? |
| **Scale** | QPS, peak vs average, concurrent users, data volume |
| **Latency** | Time-to-first-token target? End-to-end? Is streaming acceptable? |
| **Quality** | What accuracy is required? What is the cost of being wrong? |
| **Cost** | Budget per request or per month? Is this a cost centre or revenue-generating? |
| **Data** | Where does the knowledge live? How often does it change? How sensitive is it? |
| **Users** | Internal or external? Adversarial? Regulated industry? |
| **Constraints** | Self-hosted required? Data residency? Existing stack? |

**2. Establish the quality bar and how you will measure it.** State up front how you would know the system works — the eval strategy. This is the single strongest differentiator in LLM system design interviews, because most candidates jump straight to architecture and never mention measurement.

**3. Choose the architecture with the least machinery that meets the bar.**

```text
Can a single well-prompted LLM call do it?              → do that
Does it need current or private knowledge?              → add retrieval (RAG)
Does it need to act on systems?                         → add tools
Is the sequence of steps knowable in advance?           → chain/workflow
Does the control flow depend on runtime discoveries?    → agent
Does behaviour/format need to change fundamentally?     → consider fine-tuning
```

**4. Design the data path.** For most systems the interesting engineering is in the data, not the model: ingestion, chunking, indexing, freshness, permissions.

**5. Do the arithmetic out loud.** Tokens per request × requests per month × price. Latency budget broken into components. This is where you demonstrate engineering rather than enthusiasm — see Q4.

**6. Address the cross-cutting concerns.** Evaluation, monitoring, safety and guardrails, cost controls, failure modes and degradation, caching, and rollout strategy.

**7. Name the trade-offs you made and what you would do differently at 10× scale.**

**Common mistakes to avoid:**
- Jumping to architecture before clarifying requirements.
- Never mentioning evaluation. If you say nothing about how you would measure quality, you look like you have not shipped one of these.
- Ignoring cost. LLM systems fail on unit economics more often than on quality.
- Over-engineering: proposing a multi-agent system for a classification problem.
- Ignoring latency. A technically excellent 40-second response is a product failure.
- Treating the model as the system. The model is one component; the data path, the guardrails, and the evaluation loop are the system.

**The framing that lands well:** LLM system design is mostly *ordinary distributed systems engineering* around a **non-deterministic, expensive, high-latency, occasionally-wrong component**. Every design decision follows from managing those four properties.

### Interview Follow-ups

- What if the interviewer gives you no numbers? (Propose reasonable ones and state them as assumptions. "Let's assume 100k requests/day, p95 under 3 seconds to first token" — then design against that and note how the design changes at 10×.)
- How much time on evaluation? (Enough to name the metrics, the dataset, and the loop — usually 2–3 minutes. It should never be zero.)

---

## Q2: What are the main architectural patterns for LLM applications?

### Answer

| Pattern | Structure | Use when | Cost/latency |
|---|---|---|---|
| **Single call** | Prompt → model → output | Classification, extraction, summarisation, generation from given input | 1×, lowest |
| **Prompt chain** | Fixed sequence of calls | Multi-stage transformation with known steps | N× |
| **RAG** | Retrieve → augment → generate | Needs private/current knowledge | 1× + retrieval |
| **Tool-using model** | Model calls functions | Needs live data or actions, single decision | 2–3× |
| **Agent** | Model in a loop with tools | Control flow unknown in advance | 3–50× |
| **Router** | Classify → dispatch to a handler | Heterogeneous request types | 1× + tiny classifier |
| **Multi-agent** | Coordinated specialists | Multi-domain tasks, genuine parallelism | 5–15× |
| **Fine-tuned model** | Specialised weights | Consistent behaviour/format/domain style at scale | Training + cheaper inference |
| **Hybrid** | Router over several of the above | Most real production systems | Varies by route |

**The escalation ladder — always start at the top:**

```text
1. Single call            ← try this first, always
2. Single call + RAG
3. Fixed chain
4. Chain + tools
5. Constrained agent
6. Multi-agent
```

Each step down adds capability and subtracts predictability, testability, and margin. The most common architectural error in production LLM systems is starting at level 5 or 6 for a problem that level 2 solves.

**The hybrid pattern is what mature systems converge on**, because request difficulty is not uniformly distributed:

```text
Request
  → classify (cheap, ~200ms, small model)
  ├── 60% simple      → cached or single call        ~300ms,  $0.001
  ├── 30% knowledge   → RAG chain                    ~2s,     $0.01
  ├── 8%  complex     → agent with tools             ~15s,    $0.25
  └── 2%  out of scope → refuse / escalate to human
```

The economics of this are decisive: if you send everything to the agent path, average cost per request is $0.25. With routing it is roughly $0.026 — nearly a 10× reduction — and the median user gets a sub-second answer. This is the highest-leverage architectural decision in most LLM systems.

**How to choose between RAG and fine-tuning** (detailed in `10-rag.md` Q3): RAG for *knowledge*, fine-tuning for *behaviour*. They compose — a fine-tuned model can be better at using retrieved context.

**A pattern worth knowing that is often missed: the offline/online split.** Anything that can be computed at ingestion time should be — chunk contextualisation, summaries, extracted metadata, precomputed embeddings, cached common answers. Moving work from serving time to ingestion time improves latency and cost simultaneously, and is usually cheaper because it is batchable.

### Interview Follow-ups

- How do you decide between a chain and an agent? (Can you enumerate the steps for every input? If yes, chain. See `11-ai-agents.md` Q25.)
- Where does caching fit in this taxonomy? (It is orthogonal — every pattern benefits. Exact-match response caching, prompt caching on the prefix, and retrieval result caching are three different layers, all worth having.)

---

## Q3: How do you choose a model for a given task?

### Answer

**Decision dimensions:**

| Dimension | Considerations |
|---|---|
| **Capability required** | Reasoning depth, instruction following, long context, multilingual, code, vision |
| **Cost** | Input/output price per million tokens × your volume |
| **Latency** | TTFT and tokens/sec; large models are slower per token |
| **Deployment** | API vs self-hosted; data residency; air-gapped requirements |
| **Context window** | Do your prompts genuinely need 200k, or is 32k plenty? |
| **Open vs closed weights** | Control, customisability, and cost vs capability and operational simplicity |
| **Fine-tunability** | Do you need to adapt behaviour? |
| **Rate limits and reliability** | Provider capacity, SLA, regional availability |

**A practical selection process:**

1. **Start with the most capable model available.** Establish whether the task is solvable at all, and get a quality ceiling to measure against. Do not optimise cost before you know the task works.
2. **Build the eval set** on that model's outputs and your quality bar.
3. **Step down and measure.** Try progressively cheaper/faster models against the same eval. You are looking for the cheapest model that clears the bar.
4. **Split by task.** Different steps in one system deserve different models — this is where most of the savings are.
5. **Re-evaluate quarterly.** The frontier moves; a model that was borderline six months ago may now be comfortable, and prices fall.

**The task-tiering pattern — the practical heart of the answer:**

| Task | Model tier | Why |
|---|---|---|
| Intent classification, routing | Small/fast | Simple decision, latency-critical |
| Query rewriting | Small/fast | Short, mechanical |
| Field extraction from a document | Small–mid | Structured, constrained output |
| Reranking | Purpose-built cross-encoder | Specialised, much cheaper than an LLM |
| RAG answer generation | Mid | The quality-visible step |
| Complex reasoning, planning, agent decisions | Large/frontier | Errors compound; do not economise here |
| Code generation and review | Large + code-strong | Correctness matters |
| Summarising a tool result | Small | Mechanical compression |

**Where NOT to economise, which is the insight interviews look for:** the model making **control-flow decisions** in an agent. A cheap model that picks the wrong tool adds steps, and each extra step resends the whole context — so total cost per completed task can *rise* even though per-call cost fell. Measure cost per **completed task**, not per call (`11-ai-agents.md` Q26).

**Self-hosted vs API:**

| | API | Self-hosted |
|---|---|---|
| Time to production | Hours | Weeks |
| Cost at low volume | Much cheaper | Idle GPUs are expensive |
| Cost at very high sustained volume | Can become expensive | Can be cheaper |
| Capability ceiling | Highest | Behind the frontier, closing |
| Data control | Contractual | Complete |
| Ops burden | None | Serving, scaling, upgrades, GPU supply |
| Customisation | Limited | Full fine-tuning, quantisation, custom decoding |

Self-hosting makes sense at sustained high volume, with strict data-residency requirements, or when you need deep customisation. Otherwise the ops cost usually dominates any inference savings — and the break-even volume is higher than teams expect because it must include engineering time.

**Always build a provider abstraction.** Model choice will change — for cost, capability, availability, or an outage. A thin interface over "chat with tools and structured output" costs little and lets you switch or fall back without touching application logic.

### Interview Follow-ups

- How do you handle a provider outage? (Multi-provider fallback behind your abstraction, with the eval set run against both so you know the fallback's quality. Degrade explicitly rather than silently.)
- When is fine-tuning a small model better than prompting a large one? (High volume, narrow task, stable requirements, and a labelled dataset. You can often match a frontier model's quality on one specific task at a fraction of the cost and latency — but you now own a training and maintenance pipeline.)

---

## Q4: How do you estimate and control the cost of an LLM system?

### Answer

**Do the arithmetic explicitly — this is the part of LLM system design that interviews reward most and candidates skip most.**

**The model:**

```text
cost/request = (input_tokens × input_price) + (output_tokens × output_price)
monthly cost = cost/request × requests/month
```

**A worked example — a RAG chatbot:**

```text
Per request:
  system prompt + instructions        800 tokens
  8 retrieved chunks × 400 tokens   3,200 tokens
  conversation history              1,500 tokens
  user question                       100 tokens
  ─────────────────────────────────────────────
  input                             5,600 tokens
  output                              400 tokens

At mid-tier pricing (~$3/M input, $15/M output):
  input:  5,600 / 1e6 × $3  = $0.0168
  output:   400 / 1e6 × $15 = $0.0060
  per request                = $0.0228

Plus embedding the query (~$0.000002) and retrieval (~negligible per query).

100,000 requests/day → $2,280/day → ~$68,000/month
```

That number changes the conversation. Now optimise:

| Optimisation | New cost | Saving |
|---|---|---|
| Prompt caching on the 800-token system prompt + tools | $67,400/mo | ~1% (small prefix) |
| Reduce to 4 chunks × 400 (rerank first, better precision) | $53,000/mo | 22% |
| Trim history to 600 tokens via summarisation | $47,800/mo | 30% |
| Route 50% of traffic to a small model (~10× cheaper) | $26,300/mo | 61% |
| Exact-match cache for the top 15% of repeated questions | $22,400/mo | 67% |

Realistically stacking these gets you from $68k to roughly $22k — a 3× reduction with no quality loss, because fewer-but-better chunks and cached repeats do not degrade answers.

**The cost levers, ranked:**

| Lever | Typical impact | Notes |
|---|---|---|
| **Route by complexity** | 3–10× | The biggest single lever. Most traffic is easy. |
| **Response caching** | Proportional to repeat rate | 10–40% of queries repeat in many products |
| **Prompt caching** | 50–90% on the cached prefix | Huge for agents (context resent every step) |
| **Fewer, better retrieved chunks** | 20–40% of input | Reranking lets you send 4 instead of 12 |
| **Smaller model per subtask** | 5–20× on those calls | Classification, rewriting, extraction |
| **Cap output tokens** | Output is priced 3–5× input | Also improves latency |
| **Compress conversation history** | 20–40% on multi-turn | Summarise old turns |
| **Reduce agent step count** | Linear, and compounds | Each step resends everything |
| **Batch API for offline work** | ~50% | Anything not latency-sensitive |

**Cost controls you must build, not just measure:**

1. **Per-request hard caps** on tokens and steps — enforced in the loop, not in a dashboard. Monitoring tells you after you have paid.
2. **Per-user and per-tenant quotas.** Otherwise one user (or one attacker) can consume your entire budget — "denial of wallet."
3. **Cost attribution** per feature, tenant, and route. You cannot optimise an aggregate.
4. **Alerts on cost per request**, not just total cost. A rising per-request cost is the early signal of a regression (longer contexts, more agent steps).
5. **A kill switch** for expensive paths under budget pressure — degrade to the cheap path rather than exceeding the budget.

**The most common cost surprises in production:** unbounded agent loops; retrieved context growing as the corpus grows; conversation history with no compression; retries multiplying cost silently; and an evaluation suite that costs more than production traffic.

### Interview Follow-ups

- Where does the cost usually hide? (Input tokens. People focus on output because it is priced higher per token, but input is often 10–20× the volume. Retrieved context and conversation history are the biggest contributors.)
- How do you justify LLM spend? (Cost per resolved task versus the human-labour alternative, plus deflection rate. $0.25 per resolved support ticket against $6 of agent time is an easy case; the discipline is measuring *resolved*, not *handled*.)

---

## Intermediate

---

## Q5: How do you engineer latency in an LLM system?

### Answer

**First, decompose the budget.** You cannot optimise what you have not measured per component:

```text
Total: user hits send → sees first token

  Input guardrails / moderation            50–300 ms
  Query embedding                          10–30 ms
  Retrieval (ANN search)                   10–50 ms
  Reranking (cross-encoder, 50 docs)      100–300 ms
  Prompt assembly                          ~5 ms
  LLM prefill (TTFT) — scales with input   300–2,000 ms
  ─────────────────────────────────────────────────
  Time to first token                      ~0.5–2.7 s

  LLM decode: output_tokens / tokens_per_sec
              400 tokens at 60 tok/s        ~6.7 s
  Output guardrails (if not streaming)      +100–500 ms
```

**The two distinct metrics that matter, and conflating them is a red flag:**
- **TTFT (time to first token)** — dominated by prefill, which scales with *input* length. This is what the user perceives as responsiveness.
- **Total completion time** — dominated by decode, which scales with *output* length.

**Latency levers, ranked by impact:**

| Lever | Effect | Notes |
|---|---|---|
| **Stream the response** | Transforms perceived latency | Do this first. TTFT becomes the number that matters. |
| **Reduce output length** | Linear on total time | Concise prompts, output caps. Often the biggest real win. |
| **Reduce input length** | Reduces TTFT | Fewer chunks, compressed history |
| **Prompt caching** | Cuts prefill substantially | Cached prefix skips recomputation |
| **Smaller/faster model** | 2–5× | Route simple traffic here |
| **Parallelise independent work** | Overlaps retrieval, guardrails, embedding | Free latency; often forgotten |
| **Response cache** | Near-zero for hits | Exact match is safe; semantic caching is risky (Q9) |
| **Skip reranking when unnecessary** | 100–300 ms | Only rerank when the candidate set is large |
| **Speculative decoding** | 1.5–3× on decode | Mostly a self-hosted concern |
| **Colocate services** | 10–100 ms | Cross-region hops add up |

**Parallelisation is the most commonly missed easy win:**

```python
# Serial: 300 + 30 + 40 = 370 ms before the model call
moderation = moderate(query)
embedding  = embed(query)
history    = load_history(thread_id)

# Parallel: max(300, 30, 40) = 300 ms
moderation, embedding, history = await asyncio.gather(
    moderate(query), embed(query), load_history(thread_id))
```

The same applies to independent tool calls in an agent, and to fanning out multiple retrievals.

**The streaming caveat with output guardrails.** Full-response validation is fundamentally incompatible with token streaming. Options: stream optimistically and be prepared to retract; validate sentence-by-sentence as you stream; or buffer entirely and lose the TTFT benefit. Pick deliberately and state the trade-off — there is no free version of this.

**Tail latency.** p99 is what users complain about. Sources: model provider variance, retrieval fan-out to a slow shard, cold caches, retries, and — in agents — the variable step count. Mitigations: timeouts with fallbacks at every hop, hedged requests for critical low-latency paths, and hard step caps so an agent cannot take 40 steps.

**What users actually tolerate:**

| Interaction | Acceptable TTFT |
|---|---|
| Autocomplete / inline suggestion | < 300 ms |
| Chat response | < 1 s |
| Search with results | < 2 s |
| Complex analysis (with progress shown) | 5–30 s |
| Background job (with notification) | Minutes |

**The most important insight:** with streaming plus visible progress, users tolerate far more total latency than you would expect — but they tolerate very little *silence*. For agents, streaming tool activity ("Searching the knowledge base…") is worth more than shaving seconds off total time (`11-ai-agents.md` Q16).

### Interview Follow-ups

- Why does TTFT depend on input length? (Prefill processes the entire input in one forward pass before the first token is emitted; its cost scales with input tokens. A 100k-token prompt has a materially worse TTFT than a 5k one.)
- What is the latency cost of a reranker, and is it worth it? (100–300 ms for a cross-encoder over ~50 candidates. Almost always worth it, because it lets you send 4 chunks instead of 12 — improving quality, cost, *and* TTFT simultaneously.)

---

## Q6: How do you evaluate an LLM system end to end?

### Answer

**Evaluation is the difference between a demo and a product.** Without it you cannot ship changes safely, compare models, or know whether you are improving.

**The layers:**

| Layer | What it measures | Method |
|---|---|---|
| **Component** | Each stage independently | Retrieval: recall@k, nDCG. Classification: accuracy/F1. Extraction: field accuracy. |
| **End-to-end offline** | Whole-system quality on a fixed set | LLM-as-judge on rubrics, reference comparison, programmatic checks |
| **Online** | Real user behaviour | Thumbs, edit rate, escalation rate, task completion, retention |
| **Operational** | Health | Latency, error rate, cost, cache hit rate |
| **Safety** | Harm and abuse | Red-team suites, guardrail trigger rates, jailbreak resistance |

**Build the eval set properly — this is the actual work:**

1. **Start from real traffic**, not imagined queries. Sample production logs (or beta traffic) across the distribution.
2. **50–200 cases** to start. Quality over quantity; a well-chosen 80 beats a sloppy 1,000.
3. **Cover the distribution deliberately:** common cases, hard cases, edge cases, **out-of-scope cases that should be refused**, adversarial cases, and every past production failure.
4. **Version it in git** and run it in CI.
5. **Label with the outcome you care about**, not with a golden string when the task has many valid answers.
6. **Grow it from incidents.** Every production failure becomes a permanent case. This is the mechanism by which quality compounds.

**Choosing a metric — prefer programmatic over judged:**

```text
Is the output verifiable programmatically?
├── YES → assert it. (Schema valid? Code compiles? Tests pass? DB row correct?
│         Citation present in the source? Number matches the computation?)
└── NO  → LLM-as-judge on an explicit rubric, calibrated against human labels
```

Programmatic checks are cheap, deterministic, and trustworthy. Reach for them aggressively — a surprising amount of "subjective" output has verifiable properties (contains a citation, cites only retrieved sources, respects the requested format, does not assert a date absent from context).

**LLM-as-judge, done properly:**
- Give the judge a **specific rubric with criteria**, not "rate 1–10."
- Ask for **reasoning before the score** (the ordering matters — see `06-transformers-llms-generative-ai.md` Q29).
- **Calibrate against human labels** on a subset; report agreement. An uncalibrated judge is a random number generator with good manners.
- Prefer **pairwise comparison** over absolute scoring — models are better at "which is better" than "is this a 7."
- Watch for known biases: position, verbosity, and self-preference (a model favouring its own outputs).
- Use a **different model** as judge than as generator where practical.

**Component vs end-to-end — you need both.** End-to-end tells you *whether* the system works; component metrics tell you *where* it broke. The diagnostic tree for RAG (`10-rag.md` Q13) is exactly this: if the answer is wrong, was the right document retrieved? If yes, generation is at fault; if no, retrieval is.

**Online evaluation signals, ranked by usefulness:**
1. **Implicit behavioural signals** — did the user copy the answer, edit it, retry, rephrase, escalate, or abandon? These are abundant and honest.
2. **Task completion** — did the thing they wanted actually happen?
3. **Explicit feedback** — thumbs up/down. Sparse and biased, but a useful trend.
4. **A/B tests** on real outcomes for anything significant.

**Regression testing is the point.** The purpose of the eval set is not a leaderboard number; it is the ability to change a prompt, swap a model, or adjust chunking and know within minutes whether you broke something. LLM systems regress invisibly — a prompt edit that improves one case class silently degrades another. Without CI evaluation you find out from users.

### Interview Follow-ups

- How do you evaluate when there is no ground truth? (Rubric-based judging on properties you *can* specify — groundedness in retrieved context, format compliance, absence of unsupported claims, appropriate refusal — plus pairwise comparison against the current production version.)
- What is the minimum viable evaluation? (30–50 real cases with programmatic checks where possible, run before every deploy, plus production logging of implicit signals. That is achievable in a day and is enormously better than nothing.)

---

## Q7: How do you handle safety, abuse, and guardrails at the system level?

### Answer

**Layered defence, with each layer enforced at a place you control.** The same principle as `11-ai-agents.md` Q14, at system scale.

| Layer | Placement | Handles |
|---|---|---|
| **Authentication and rate limiting** | Edge | Abuse volume, denial of wallet, scraping |
| **Input classification** | Before the model | Prohibited content, injection attempts, off-topic, PII |
| **System prompt** | The model call | Scope, tone, refusal behaviour — **soft, probabilistic** |
| **Retrieval scoping** | The retrieval call | ACL filters from the session — **hard** |
| **Tool authorisation** | Before tool execution | What actions are possible — **hard, the real boundary** |
| **Output classification** | After generation | Harmful content, PII leakage, policy violations, ungrounded claims |
| **Structured output validation** | After generation | Format and schema compliance — **hard** |
| **Human review** | For flagged or high-stakes output | The residual |
| **Monitoring and audit** | Continuous | Detection, incident response, evidence |

**The core distinction to state: prompt-level constraints shape typical behaviour; code-level constraints define what is possible.** Anything that matters — permissions, spend, data access, irreversible actions — must be enforced outside the model.

**Prompt injection — the LLM-specific threat.** Untrusted content (a retrieved document, a web page, a user-submitted file, another agent's output, an MCP tool description) contains instructions the model follows.

The **lethal trifecta** (`06-transformers-llms-generative-ai.md` Q24): untrusted content + access to private data + an outbound communication channel. All three present means exfiltration is possible. Mitigation: **remove one leg**, and the outbound channel is usually the easiest — no arbitrary URL fetching, no arbitrary email/webhook sending, no rendering of model-supplied image URLs.

Additional defences (mitigations, not solutions):
- Delimit and label untrusted content explicitly as data, not instruction.
- Never let the model influence security-relevant values — inject them from the session.
- Apply output guardrails to *tool arguments*, not just to the final answer, since exfiltration happens through arguments.
- Screen retrieved content for injection patterns at ingestion and at retrieval time.
- Treat the model as a **potentially-compromised client** in your threat model.

**Abuse patterns and defences:**

| Abuse | Defence |
|---|---|
| Jailbreaking to produce prohibited output | Input + output classifiers; refusal training; red-team suite in CI |
| Prompt extraction | Accept that prompts leak; do not put secrets in them |
| Denial of wallet | Per-user quotas, hard token caps, anomaly alerts |
| Using your product as a free general-purpose LLM | Topic classification; scope enforcement in prompt *and* output check |
| Scraping your corpus via retrieval | Rate limits; result caps; monitoring for enumeration patterns |
| PII injection into memory or logs | PII detection at input; redaction before persistence |
| Training-data extraction | Output filtering; do not fine-tune on unfiltered sensitive data |

**Operational requirements:**
- **Fail closed.** If the moderation service is down, block rather than allow.
- **Tier the checks.** Cheap deterministic checks always; expensive model-based classification only on risky paths — otherwise latency and cost become unacceptable.
- **Log for incident response**, with retention that satisfies both investigation needs and privacy obligations.
- **Red-team before launch and continuously.** Maintain an adversarial suite in CI alongside the quality eval set.
- **Have an incident runbook**: how to disable a feature, purge a poisoned index entry, revoke a tool, and notify.
- **Measure false positives.** Over-blocking is a real product harm, not a safe default. Track the rate and the categories.

**The honest position for an interview:** you cannot make an LLM reliably refuse everything you want refused — instruction-following is probabilistic and adversarially bypassable. So the design goal is not a perfectly-behaved model; it is a system where **the model's worst-case output cannot cause serious harm**, because authority, data access, and outbound channels are all constrained in code.

### Interview Follow-ups

- Why is prompt injection not "solved"? (There is no reliable separation between instruction and data in a single token stream. Every mitigation is probabilistic. The durable fix is architectural — limiting what the model can do and reach — not better prompting.)
- How do you balance safety against usability? (Measure both. Track false-positive block rate alongside harmful-output rate, and tune per risk tier. A system that refuses 5% of legitimate requests has a real cost that must be weighed explicitly, not assumed away.)

---

## Q8: How do you monitor an LLM system in production?

### Answer

**LLM systems fail differently: they usually keep returning 200s while producing worse answers.** So monitoring must cover quality, not just availability.

**What to instrument:**

| Category | Metrics |
|---|---|
| **Traffic** | QPS, by route/feature/tenant; request type distribution |
| **Latency** | TTFT and total, p50/p95/p99, broken down per pipeline stage |
| **Errors** | Provider errors by class (429, 5xx, timeout), validation failures, tool failures |
| **Cost** | Tokens in/out per request, cost per request, cost per tenant/feature, cache hit rate |
| **Quality proxies** | Thumbs, retry rate, rephrase rate, edit rate, escalation rate, abandonment |
| **Retrieval** | Top-1 score distribution, zero-result rate, empty-after-filter rate, chunks used |
| **Generation** | Output length distribution, refusal rate, truncation rate, format-validation failures |
| **Agents** | Steps per task, tool call counts and failure rates, loop detections, budget exhaustions |
| **Safety** | Guardrail trigger rates by category, blocked requests, false-positive reports |
| **Drift** | Input length and topic distribution over time; embedding drift; retrieval score drift |

**The leading indicators — what actually catches problems early:**

1. **Retrieval score distribution.** A drop in mean top-1 similarity is the earliest signal of a retrieval or corpus problem, and it moves before user complaints.
2. **Refusal rate.** A spike means retrieval is failing or the guardrails are over-triggering. A drop can mean guardrails broke.
3. **Retry/rephrase rate.** Users rephrasing is the cheapest, most abundant quality signal you have.
4. **Cost per request.** Rising cost per request means contexts are growing or agents are taking more steps — a quality regression in disguise.
5. **Output length distribution.** Sudden shifts usually mean a prompt or model change had unintended effects.
6. **Steps per agent task.** The distribution's tail reveals loops before the cost alert fires.

**Tracing is non-negotiable.** For every request, capture: the fully-rendered prompt, retrieved chunk ids and scores, model and version, every tool call and result, token counts, latencies per stage, and the final output. Sample at 100% initially; reduce with sampling but keep 100% of errors, flagged outputs, and slow requests. You cannot debug an LLM system from aggregate metrics — you need the individual trace (`11-ai-agents.md` Q29).

**Continuous evaluation in production:**
- Run the eval set against production on a schedule, not only in CI — provider-side model updates can change behaviour without any deploy on your side.
- Sample real traffic and score it with an LLM judge to produce a continuous quality metric.
- Alert on quality-metric drift, not just on errors.
- Maintain a canary set of stable queries whose answers should not change; alert if they do.

**Alerting worth having:**

| Alert | Threshold shape |
|---|---|
| p95 TTFT above target | Sustained over 5 minutes |
| Error rate spike | Relative to a rolling baseline |
| Cost per request up >30% | Day over day |
| Retrieval mean top-1 score down | Relative to a 7-day baseline |
| Refusal rate outside a band | Both directions |
| Guardrail trigger rate spike | Possible attack |
| Zero-result retrieval rate up | Index or filter problem |
| Agent budget-exhaustion rate up | Loops or degraded tools |
| Provider error rate | Trigger a fallback |

**Privacy in observability.** Traces contain user content and possibly PII. Redact before storage, set retention limits, restrict access, and support deletion by user. "We log everything forever for debugging" is a compliance incident waiting to happen.

**The framing:** monitor the **pipeline stages** and **user behaviour**, not just the model. Most production degradations originate in the data path — a stale index, a changed document format, a filter that now matches nothing — and are invisible if you only watch model-level metrics.

### Interview Follow-ups

- How do you detect a silent quality regression? (Continuous judged sampling plus behavioural signals — retry rate, escalation rate — alongside a canary query set. Any one alone is noisy; together they catch it.)
- What if the provider silently updates the model? (This happens. Pin versions where the provider allows it, run the eval set on a schedule, and keep a canary set. Detecting it is the whole reason for scheduled production evaluation.)

---

## Q9: How does caching work in an LLM system, and where are the traps?

### Answer

**Four distinct caching layers, often confused with each other:**

| Layer | Key | Hit saves | Risk |
|---|---|---|---|
| **Exact-match response cache** | Normalised prompt hash | Everything — full request | Staleness; **must be scoped correctly** |
| **Semantic response cache** | Query embedding similarity | Everything | **Wrong answers on near-miss queries** |
| **Provider prompt cache** | Prompt prefix | Prefill cost and time | Low — provider-managed |
| **Retrieval cache** | Normalised query + filters | Embedding + ANN search | Staleness on index updates |

**1. Exact-match response caching.** Hash the fully-rendered prompt (including retrieved context and all parameters) and cache the response.

- **Effective when** there is genuine query repetition — FAQ-style traffic, documentation search, and support systems often see 10–40% repeats.
- **The critical trap: cache key scope.** The key must include **everything that affects the correct answer**: the user's tenant, their permission groups, the retrieval filters, the model version, the prompt version, and the language. A cache keyed only on question text will serve one tenant's answer to another — a data breach, not a bug. **Never cache across a permission boundary.**
- **Invalidation:** on prompt-version change, model change, and index update. Version the key with all three; then a change simply misses rather than serving stale content.

**2. Semantic caching — high value, high risk.** Embed the query, find a cached query above a similarity threshold, return its answer.

The failure mode is fundamental: **semantically similar is not semantically equivalent.**

```text
"How do I cancel my subscription?"       ← cached
"How do I cancel my subscription trial?" ← 0.94 similar, DIFFERENT answer
"Can I get a refund after cancelling?"   ← 0.89 similar, DIFFERENT answer

"What is our PTO policy for engineers?"  ← cached
"What is our PTO policy for interns?"     ← 0.96 similar, DIFFERENT answer
```

Negations, entity swaps, and qualifier additions all produce very high similarity with completely different correct answers. Guidance:
- Use a **high threshold** (0.95+) and accept a low hit rate.
- **Never** use semantic caching where a wrong answer is costly — medical, legal, financial, or anything with an entity or number in the query.
- Consider it only for genuinely paraphrase-heavy, low-stakes traffic.
- **Log every hit and audit them.** If you cannot measure the wrong-answer rate, you should not run it.
- A safer variant: semantic **retrieval** caching (cache the retrieved chunks, still generate fresh). You save retrieval cost and the generation step still sees the actual question.

**3. Provider prompt caching — the one to always use.** The provider caches the KV state for a prompt *prefix*, so repeated prefixes skip prefill.

- **Requires prefix stability.** Order context **stable → volatile**: system prompt, tool schemas, then long-lived context, then retrieved chunks, then conversation, then the current question. A timestamp or a user name at the *top* of the prompt destroys the entire cache.
- **Enormous for agents**, which resend the whole growing history every step — the saving compounds with step count (`11-ai-agents.md` Q26).
- Note the minimum cacheable length and TTL for your provider; short prompts may not qualify.

**4. Retrieval caching.** Cache the query embedding and the ANN result set keyed by (normalised query, filters, index version). Saves 20–80 ms and the embedding call. Invalidate on index version bump — including the index version in the key handles this automatically.

**Also cacheable, and often overlooked:** embeddings for repeated documents at ingestion time (dedupe by content hash — meaningful money on large re-indexes), reranker scores for repeated (query, doc) pairs, and precomputed answers for known-popular questions generated offline.

**Measuring caching properly:** track hit rate per layer, cost saved, latency saved, and — for semantic caching — a sampled correctness audit. A cache with a 40% hit rate and a 3% wrong-answer rate is not a win.

### Interview Follow-ups

- Why does prompt caching require careful prompt ordering? (It matches on a *prefix*. Anything variable placed early invalidates everything after it. Put stable content first — this single ordering decision can be the difference between a 90% and a 0% cache rate.)
- Would you deploy semantic caching? (Only with a high threshold, on low-stakes paraphrase-heavy traffic, with logging and a sampled correctness audit — and never across tenants or on entity-bearing queries. For most systems, exact-match plus prompt caching captures the benefit without the risk.)

---

## Q10: How do you design for multi-tenancy in an LLM system?

### Answer

**The requirement: complete data isolation between tenants, plus fair resource sharing, plus per-tenant configuration and cost attribution.** A cross-tenant leak in an AI product is an existential incident, so isolation must be structural rather than procedural.

**Isolation strategies:**

| Strategy | Isolation | Cost | Scale |
|---|---|---|---|
| **Separate infrastructure per tenant** | Strongest | Highest | Tens of tenants |
| **Separate index/collection per tenant** | Strong | Medium | Hundreds to low thousands |
| **Namespace/partition per tenant in a shared index** | Good | Low | **Thousands — the usual default** |
| **Shared index with metadata filters only** | Weakest | Lowest | Risky at any scale |

**Namespace-per-tenant is the right default** for a typical SaaS: strong logical isolation, one index to operate, and per-tenant deletion is straightforward. Reserve dedicated infrastructure for the few enterprise customers who contractually require it.

**Enforce isolation structurally, not by convention.** Every path to data must be scoped, with no unscoped path available (the pattern from `10-rag.md` Q22):

```python
class TenantScopedClient:
    """The only way to reach data. No unscoped method exists."""
    def __init__(self, tenant_id: str, user_groups: list[str]):
        if not tenant_id:
            raise ValueError("tenant_id is required")
        self._tenant_id, self._groups = tenant_id, user_groups

    def search(self, query: str, k: int = 20, extra: dict | None = None):
        must = [{"key": "tenant_id", "match": {"value": self._tenant_id}},
                {"key": "acl_groups", "match": {"any": self._groups}}]
        if extra:
            must.extend(extra.get("must", []))
        return self._client.search(namespace=self._tenant_id, filter={"must": must},
                                   vector=embed(query), limit=k)
```

Three properties matter here: the tenant comes from the **authenticated session** (never from a request body or model output), the filter is **mandatory and unremovable**, and namespace *plus* metadata filter gives **two independent enforcement layers** so a bug in one is not a breach.

**Beyond the vector store — every stateful component needs the same treatment:**

| Component | Isolation mechanism |
|---|---|
| Vector index | Namespace + mandatory metadata filter |
| Conversation state / checkpoints | `thread_id` namespaced by tenant; never guessable |
| Long-term memory store | Namespace keyed by (tenant, user) |
| Response cache | **Tenant in the cache key** — the classic leak vector |
| Logs and traces | Tenant-tagged, access-controlled |
| Prompt/config | Per-tenant overrides resolved from the session |
| Fine-tuned models | Per-tenant only if trained on that tenant's data — never share |

**The response cache is the most commonly missed leak.** A cache keyed on question text alone will happily serve Tenant A's answer, built from Tenant A's documents, to Tenant B. Tenant must be in the key (Q9).

**Fairness and resource management:**
- **Per-tenant rate limits and token quotas.** Without them, one tenant's runaway job degrades everyone.
- **Noisy-neighbour protection.** A tenant with 50M vectors slows shared-index queries for small tenants. Consider tiering by size.
- **Whale tenants** may need dedicated infrastructure; design the migration path before you need it.
- **Cold tenants** — most tenants are idle most of the time. Avoid per-tenant always-on resources; prefer shared infrastructure with logical isolation, and consider hot/cold tiering for storage.

**Per-tenant configuration** to support: their own documents and refresh cadence, custom system prompt or tone, feature flags, model tier by plan, guardrail strictness, and data-residency region. Keep this in a tenant config service, resolved per request into the request config — not in state (`12-langgraph.md` Q28).

**Compliance requirements:** per-tenant data deletion (including from indexes, caches, checkpoints, memory, and logs), data-residency routing, per-tenant audit logs, and an exportable record of what data was used. Build deletion early — retrofitting "delete everything for tenant X" across six stores is genuinely hard.

**Testing isolation** deserves dedicated tests: attempt cross-tenant access via every path (direct API, crafted filters, cache, adversarial prompt injection attempting to alter the tenant) and assert failure. Make these part of CI.

### Interview Follow-ups

- How do you handle a tenant needing data residency in a specific region? (Region-specific deployments with routing by tenant at the edge, and per-region indexes. This has to be an architectural decision — you cannot filter your way to residency compliance.)
- How do you provide shared plus private knowledge? (Two retrievals — the shared corpus and the tenant namespace — merged with RRF, with tenant results preferred on conflict. The tenant's own documents should win over generic content.)

---

## Q11: How do you version and deploy changes to an LLM system safely?

### Answer

**LLM systems have more mutable surfaces than traditional software, and most of them can silently change behaviour:**

| Surface | Changes affect |
|---|---|
| **Prompts** | Everything about behaviour |
| **Model / model version** | Everything, including tone, format, tool use |
| **Model parameters** | Determinism, verbosity, creativity |
| **Tool schemas and descriptions** | Tool selection accuracy |
| **Retrieval config** (k, filters, thresholds) | What evidence the model sees |
| **Chunking strategy** | Requires full re-indexing |
| **Embedding model** | Requires full re-indexing; **queries and index must match** |
| **The corpus** | Answers change with no code change |
| **Reranker** | Result ordering |
| **Guardrail thresholds** | Refusal and false-positive rates |

**Every one of these must be versioned, logged with each request, and evaluable.** The minimum viable discipline: log the version of the prompt, model, retrieval config, and index with every request, so any output can be reproduced and any regression localised.

**Prompt versioning.** Treat prompts as code: in the repository, code-reviewed, with a version identifier attached to every request. A prompt registry with variables and versions is worth building once you have more than a handful (`07-prompt-engineering.md` Q14). Do not edit prompts in a UI in production with no history — it is the fastest way to an unexplainable regression.

**The deployment sequence for any change:**

```text
1. Offline eval        run the versioned eval set; compare against production baseline
2. Review the diff     inspect changed outputs case by case, not just the aggregate score
3. Shadow              run the new version alongside, log both, compare (no user impact)
4. Canary              1–5% of traffic; monitor quality proxies and operational metrics
5. Ramp                10% → 50% → 100%, watching at each step
6. Keep a rollback     one flag, one deploy, instant revert
```

**The step people skip is (2).** An eval score can improve on average while a specific important case class breaks badly. Always look at the changed outputs, especially regressions.

**Embedding model changes deserve special treatment**, because query and index embeddings must come from the same model:

```text
Blue-green re-indexing:
1. Build a new index with the new embedding model, in parallel
2. Verify quality on the eval set against the new index
3. Switch reads atomically (a config flag)
4. Keep the old index until confident, then delete
```

Never mix embedding models within one index, and never point new-model queries at an old-model index — similarity becomes meaningless and quality collapses in a way that is confusing to debug.

**Corpus changes are deployments too.** A document update changes answers with no code change. Version the index, track what changed, and be able to answer "why did this answer change last Tuesday?" This means logging the retrieved chunk ids and index version per request.

**Model-provider updates you do not control.** A provider may update a model behind the same name. Pin versions where possible; run the eval set on a schedule (Q8); keep a canary query set whose answers should be stable.

**Handling in-flight state.** For stateful systems (agents, multi-turn), a conversation may span a deployment. Decide the policy: migrate the state schema on load, keep the previous version serving until threads drain, or refuse to resume incompatible threads with a clear message (`12-langgraph.md` Q27). Decide before your first breaking change.

**A/B testing LLM changes** is harder than for UI changes: outcomes are noisier, effects are heterogeneous across query types, and quality is partly subjective. Practical guidance: pick a behavioural primary metric (task completion, escalation rate, edit rate), run longer than feels necessary, and segment results by query type — an average change can hide one segment improving and another degrading.

### Interview Follow-ups

- How do you roll back a prompt change? (Prompt version is a config value, so rollback is a config change — seconds, not a deploy. This is a strong argument for keeping prompts in a versioned registry rather than inlined in code.)
- What if you cannot A/B test because volume is low? (Shadow mode plus manual review of the output diff on a sampled set, and a larger offline eval set. With low volume, human review of changed outputs is both feasible and the strongest signal available.)

---

## Q12: Design an LLM-powered search system over a company's internal knowledge.

### Answer

**Clarify:** corpus size and sources? Update frequency? Permission model? User count and QPS? Latency target? Must it answer, or just find documents? Budget? Data-residency constraints?

**Assume:** 500k documents across Confluence, Google Drive, Slack, and Jira; 5,000 employees; ~10k queries/day; strict per-document permissions; answers with citations required; p95 TTFT under 2 seconds.

**Architecture:**

```text
INGESTION (continuous)
  Connectors (Confluence, Drive, Slack, Jira)
    → change detection (webhooks + nightly reconciliation)
    → extract text + structure (layout-aware for PDFs; OCR for scans)
    → capture metadata: source, author, dates, ACL groups, doc type, URL
    → chunk structurally (headings), ~400 tokens, 15% overlap
    → contextualise chunks (prepend doc title + section path; contextual retrieval
      for high-value sources)
    → embed (dense) + build sparse/BM25 index
    → upsert to vector DB, namespaced, with ACL metadata
    → deletion reconciliation (documents removed at source must leave the index)

SERVING
  Query
    → authenticate; resolve the user's ACL group memberships
    → rewrite/decontextualise (multi-turn) + classify intent
    → parallel:
        dense retrieval (k=50, ACL pre-filtered)
        BM25 retrieval  (k=50, ACL pre-filtered)
    → fuse with RRF
    → cross-encoder rerank top 50 → top 6
    → assemble context (dedupe, bookend ordering, source tags with dates)
    → generate answer with mandatory citations
    → validate: every claim cites a retrieved chunk; quotes appear verbatim
    → stream response + render citation links
    → log everything for evaluation
```

**Key decisions and rationale:**

| Decision | Why |
|---|---|
| **Hybrid dense + BM25** | Internal corpora are full of product names, error codes, ticket ids, and acronyms — exactly where dense retrieval underperforms and lexical matching wins (`09-vector-databases-retrieval.md` Q17, Q19) |
| **ACL filtering as a pre-filter** | Post-filtering can return zero results after filtering and leaks existence information. Filters derive from the session, never the query (`09-vector-databases-retrieval.md` Q18) |
| **Cross-encoder reranking** | Lets us pass 6 chunks instead of 20 — better quality, lower cost, lower TTFT simultaneously |
| **Structural chunking with contextualisation** | Headings carry meaning; isolated chunks lose "which product/version is this about" (`10-rag.md` Q7) |
| **Mandatory verbatim citations** | Internal search demands verifiability; a verbatim-quote requirement is machine-checkable (`10-rag.md` Q12) |
| **Slack handled differently** | Messages are short and contextless; chunk by thread, not by message |
| **Freshness signalling** | Internal docs go stale constantly; surface dates and prefer `status=current` (`10-rag.md` Q23) |
| **Streaming** | 2s TTFT target is achievable; total time is not |

**The five hardest problems, honestly:**

1. **Permissions.** Confluence, Drive, Slack, and Jira have different, changing permission models. Getting ACLs into the index and keeping them current is the single hardest part, and an error here is a data breach. Design: sync ACL groups continuously, filter by group membership at query time, re-check at render time for anything sensitive, and treat permission sync failures as a page-worthy incident.

2. **Staleness and contradiction.** Five documents describe the deployment process; three are obsolete. Nobody marked them. Mitigations: recency and authority as separate ranking signals, surface dates prominently, prefer official spaces, and **surface the conflict** rather than silently picking one.

3. **Slack.** High volume, low signal density, no structure, heavy coreference. Options: exclude it, include only threads from designated channels, or summarise threads at ingestion. Do not naively chunk individual messages — it pollutes retrieval badly.

4. **Evaluation.** There is no ground truth for "what should this query return." Build it: collect 150 real queries from employees, have subject-matter experts label the correct source documents, and measure recall@k and answer faithfulness against that.

5. **Adoption.** Employees try it twice; if it fails, they go back to asking a colleague. So the launch bar is high — narrow the scope to a few well-curated sources where quality is genuinely good, prove it, then expand.

**Scale arithmetic:** 500k documents × ~8 chunks = 4M vectors. At 768 dimensions that is ~12 GB of raw vectors plus HNSW graph overhead — comfortably a single node, so HNSW with full-precision vectors and no quantisation is appropriate (`09-vector-databases-retrieval.md` Q7, Q24). 10k queries/day is ~0.5 QPS average — retrieval is not the bottleneck; cost is dominated by generation. Roughly 10k × $0.02 = $200/day, ~$6k/month.

**Rollout:** start with two high-quality sources and one department; measure recall against the expert-labelled set and gather user feedback; expand sources one at a time, re-measuring; add Slack last and behind a flag.

### Interview Follow-ups

- Why not just use the vendors' built-in search? (Cross-source synthesis and natural-language answering are the value. Confluence search cannot answer a question whose evidence spans a Confluence page and a Jira ticket.)
- What would you do differently at 50M documents? (Sharding with tenant/department-aware routing, quantisation to control memory, tiered hot/cold storage, and a much stronger query-routing layer so you do not fan out to every shard on every query.)

---

## Advanced

---

## Q13: Design a system to summarise 10,000 documents per day with quality guarantees.

### Answer

**Clarify:** document types and lengths? Summary length and audience? What "quality guarantee" means concretely? Deadline per document? Budget? Is human review available? Do summaries need to be consistent across documents?

**Assume:** mixed PDFs and HTML, 2–200 pages; a 300-word executive summary plus five bullet key points; guarantee means factual accuracy and no omission of material points; 24-hour turnaround; human review capacity for ~2% of output.

**Architecture:**

```text
INGESTION
  queue documents → dedupe by content hash → classify type and length
    → extract text (layout-aware; OCR for scans) → quality gate on extraction
    → route by length

SUMMARISATION
  Short (< 8k tokens)      → single call, structured output
  Medium (8k–100k tokens)  → single call with a long-context model
  Long (> 100k tokens)     → hierarchical:
        chunk by section → summarise each section (parallel)
        → summarise the section summaries → final summary

VERIFICATION (this is where the "guarantee" comes from)
  → schema/format validation                        (programmatic, always)
  → length and structure checks                     (programmatic, always)
  → entity and number consistency check             (programmatic: every number and
                                                     named entity in the summary must
                                                     appear in the source)
  → NLI/faithfulness check per claim                (model-based)
  → coverage check: were the source's key sections represented?
  → confidence score
    ├── high confidence  → publish
    ├── medium           → regenerate once with feedback, then re-verify
    └── low or repeated failure → human review queue

OUTPUT
  publish + store the summary, the source version, the model version,
  the verification results, and the trace
```

**The design centre is verification, not generation.** Any model can produce a plausible summary; the guarantee comes from checking it. The layered approach:

| Check | Type | Catches |
|---|---|---|
| Schema and length | Programmatic | Format failures |
| **Numbers and entities present in source** | Programmatic | **Hallucinated figures — the highest-severity failure in summarisation** |
| Date consistency | Programmatic | Fabricated or shifted dates |
| Per-claim NLI against source | Model | Unsupported assertions |
| Section coverage | Model | **Omission — the failure mode judges miss** |
| Sampled human review | Human | Everything else; calibrates the automated checks |

**Why the number/entity check matters so much.** Hallucinated figures in a summary are both the most likely error and the most damaging, and they are **cheaply detectable programmatically**: extract every number and named entity from the summary and assert its presence in the source. This is a far stronger guarantee per dollar than any LLM judge, and it is the kind of answer that distinguishes someone who has shipped this.

**Omission is the underrated failure.** Faithfulness checks confirm that everything said is supported; nothing in them detects that the most important point was left out. Coverage checking — verifying that each major source section is represented — is a separate, necessary check.

**Hierarchical summarisation trade-offs:**

| Approach | Quality | Cost | Notes |
|---|---|---|---|
| Single long-context call | Best coherence | Highest per doc | Risk of lost-in-the-middle on very long docs |
| Map-reduce (section → final) | Good | Lower, parallelisable | Loses cross-section connections |
| Refine (iterative accumulation) | Good coherence | Serial, slow | Early content can dominate |
| **Hybrid: section summaries with global context** | Best trade-off | Medium | Pass a document outline into each section call |

The hybrid is usually right for long documents: give each section-summariser the document title and outline so it knows how its section fits the whole, then compose. That mitigates map-reduce's main weakness at modest cost.

**Throughput and cost arithmetic:**

```text
10,000 documents/day, average ~15k tokens input, ~600 tokens output
Base:      10,000 × (15,000 × $3/M + 600 × $15/M) = 10,000 × $0.054 = $540/day
Verification adds roughly 40%                                        ≈ $756/day
                                                                     ≈ $23k/month

Optimisations:
  route 60% of short documents to a small model (~8× cheaper)   → ~$10k/month
  Batch API for the whole pipeline (no real-time need, ~50% off) → ~$5k/month
```

The batch-API point is important: with a 24-hour turnaround there is no reason to pay real-time prices. Recognising that a requirement permits batch processing is a significant cost decision.

**Operational design:** a durable queue with per-document idempotency (so retries do not duplicate work or spend), bounded concurrency against provider rate limits, a dead-letter queue for repeated failures, and a priority lane so urgent documents skip the batch. Human review is a queue with the source document, the summary, and the specific failed checks shown side by side.

**Quality measurement:** a labelled set of 200 documents with expert summaries; measure faithfulness, coverage of expert-identified key points, and human preference against expert summaries. Track the human-review rate as the primary operational quality metric — if it climbs, something upstream regressed. Sample and audit **published** summaries too, not only the flagged ones, or you will never learn your false-negative rate.

### Interview Follow-ups

- Can you guarantee accuracy? (Not absolutely — you can bound it. Programmatic checks catch the highest-severity errors deterministically, model checks catch more probabilistically, and sampled human review measures the residual. State the measured error rate, not a guarantee.)
- How do you keep summaries consistent across documents? (A fixed structured output schema, a shared style guide in the prompt, few-shot examples from approved summaries, and a low temperature. Consistency is a format problem, largely solved by schema plus examples.)

---

## Q14: Design a code assistant for a large private codebase.

### Answer

**Clarify:** repository size and languages? Is this completion, chat, or autonomous editing? IDE integration or CLI? Can code leave the network? Team size? Must it run tests? What is the acceptance bar?

**Assume:** 5M lines across Python, TypeScript, and Go; chat plus multi-file edits; must not send code to third-party APIs without approval (assume an approved enterprise provider); 200 engineers; it may run tests in a sandbox.

**Architecture:**

```text
INDEXING (incremental, on commit)
  parse with tree-sitter → symbol graph (definitions, references, imports, call graph)
    → chunk by semantic unit (function, class, method — never fixed-size)
    → contextualise each chunk: file path, module, class, signature, docstring
    → embed (code-specialised embedding model)
    → BM25 index over identifiers and comments
    → store: symbol table, call graph, file tree, per-file git recency
    → also index: README/docs, ADRs, PR descriptions, test files

RETRIEVAL (this is the differentiator)
  Query + open file + cursor context
    → parallel:
        symbol lookup      (exact: "where is process_payment defined?")
        call-graph traversal (callers, callees, transitive deps)
        dense semantic search over code chunks
        BM25 over identifiers
        recently-edited files (git signal)
        currently-open files (IDE signal)
    → fuse and rerank
    → assemble: definitions first, then usages, then related tests, then docs

GENERATION
  → model with tools: read_file, search_symbols, list_directory, run_tests, apply_patch
  → agent loop, bounded (12 steps, token budget)
  → propose changes as a diff, never a silent write
  → run tests in a sandbox → feed failures back → iterate (max 3)
  → present the diff + test results for human approval
```

**Why retrieval is the hard part and the differentiator.** Code has *precise structure* that pure semantic search throws away. The signals that matter:

| Signal | Answers |
|---|---|
| **Symbol table** | "Where is this defined?" — exact, not approximate |
| **Call graph** | "What breaks if I change this?" |
| **Import graph** | "What is in scope here?" |
| **Type information** | "What shape is this argument?" |
| **Git recency** | "What is the team actively working on?" |
| **Open files / cursor** | "What is the user thinking about?" |
| **Tests** | "What is the intended behaviour?" |
| **Dense embedding** | "Where is anything about retry logic?" |

A code assistant that only does embedding search over fixed-size chunks will feel obviously worse than one using the symbol graph, because most real questions are structural. **Chunk by semantic unit** — a function split across two chunks is useless.

**Grounded iteration is the quality mechanism.** Running the tests and feeding failures back is the highest-value loop in the system, because the feedback is *real* rather than self-assessed (`11-ai-agents.md` Q9). A code assistant that can execute is categorically better than one that cannot. Requirements: a sandboxed environment, fast test selection (run affected tests, not the full suite), and a bounded retry count.

**Safety and trust design:**

| Concern | Design |
|---|---|
| **Never silently modify code** | Always propose a diff for human approval |
| **Sandboxed execution** | No network, no credentials, ephemeral, resource-capped |
| **Secret hygiene** | Never index `.env`, credentials, or key material; scan chunks before indexing |
| **Licence contamination** | Flag suspiciously verbatim output from known open-source code |
| **Permissions** | Respect repository ACLs in retrieval; a user must not see code they cannot check out |
| **Data control** | Enterprise provider with no-training guarantees; or self-host if required |
| **Audit** | Log every proposed and accepted change |

**Latency requirements differ sharply by mode:**

| Mode | Target | Implication |
|---|---|---|
| Inline completion | < 300 ms | Small fast model, local context only, no retrieval round trip |
| Chat question | < 2 s TTFT | Full retrieval; streaming |
| Multi-file edit | 10–60 s | Agent loop; stream progress and show the plan |

These are three different systems sharing one index. Trying to serve inline completion through the full retrieval pipeline will fail the latency budget.

**Evaluation:**
- **Retrieval:** for 200 real questions, did the correct file/symbol appear in the top-k?
- **Edits:** a benchmark of real historical PRs — can it produce a change that passes the same tests?
- **Completion:** acceptance rate and, more honestly, **retention rate** (was the accepted completion still there an hour later?).
- **Online:** acceptance rate, edit-after-accept rate, test pass rate on first attempt, time-to-merge for assisted PRs.

**Scale arithmetic:** 5M lines ≈ ~150k semantic chunks. Trivial for a vector index; the symbol graph and incremental re-indexing on every commit are the real engineering. Incremental indexing must be fast (seconds after a commit), or the assistant is always answering about stale code — which destroys trust faster than any quality issue.

**The main risk to name:** trust. Engineers abandon a code assistant permanently after a few confidently wrong answers about their own codebase. So the priority order is **retrieval precision > generation quality > coverage**. A narrow assistant that is right about the code it knows beats a broad one that hallucinates function signatures.

### Interview Follow-ups

- Why not just use a large context window and skip retrieval? (5M lines is ~50M tokens. Even at 1M context you can fit 2% of the codebase, and cost and TTFT would be prohibitive per query. Retrieval is not optional at this scale.)
- How do you handle a monorepo with multiple languages? (Per-language parsers and symbol extraction, one unified index with language metadata, and language-aware retrieval so a Python question does not surface Go code. The call graph is per-language; cross-language calls need explicit handling, e.g. API contracts.)

---

## Q15: How do you design an LLM system for high reliability?

### Answer

**The premise: the model is an unreliable component.** Reliability comes from the system around it, not from the model getting better. Everything below follows from that.

**1. Degrade gracefully at every layer.** Every dependency needs a defined failure behaviour:

| Failure | Degradation |
|---|---|
| Primary model unavailable | Fall back to a secondary provider/model (with its quality pre-measured on the eval set) |
| Reranker unavailable | Serve dense-retrieval order, log the degradation |
| Vector DB unavailable | Fall back to BM25/keyword search |
| Retrieval returns nothing | Say so explicitly; do not generate unsupported content |
| Tool unavailable | Tell the model, which reports the limitation honestly |
| Guardrail service down | **Fail closed** — block rather than allow |
| Cache down | Serve uncached; higher cost and latency, same correctness |
| Everything down | A clear error message, never a fabricated answer |

The ordering principle: **degrade capability, never correctness.** An honest "I can't reach the knowledge base right now" is a good outcome; a confident answer generated without evidence is not.

**2. Bound everything.** Every unbounded quantity is an outage waiting to happen: tokens per request, steps per agent task, retries per call, wall-clock per request, cost per request, concurrent requests per tenant. Each bound converts a potential unbounded failure into a graceful, reportable one.

**3. Verify rather than trust.** Structured output validation, citation checking, entity/number presence checks, side-effect confirmation, and grounded feedback (tests, linters, validators) wherever the domain allows. Prefer programmatic verification to model self-assessment — see Q13.

**4. Idempotency on every side effect.** Idempotency keys on writes, sends, and charges, so retries are safe. This is what makes automatic retry — a major reliability tool — usable at all.

**5. Design so failures are visible.** Citations, shown sources, previews before action, and undo where possible. **"The user can tell when it is wrong" is a far more achievable property than "it is never wrong,"** and it is the single most important reliability principle in LLM product design.

**6. Make refusal safe and first-class.** If refusing is penalised (by the prompt, the eval, or the product), the model will guess instead. Enforce at three levels: instruct it explicitly, gate on retrieval scores so there is nothing to answer from, and **reward correct refusal in your eval set**.

**7. Full observability.** Traces with the rendered prompt, retrieved ids, tool calls, and versions. Without them, production failures are unfixable (Q8).

**8. Continuous evaluation as a regression gate.** Versioned eval set in CI plus scheduled production runs. LLM systems regress silently; only measurement catches it (Q6).

**9. Test the failure paths explicitly.** Inject provider errors, empty retrieval, timeouts, malformed tool responses, budget exhaustion, and guardrail rejections. The fallback path that has never been exercised does not work.

**10. Keep humans in the loop where the cost of being wrong is high** — and remove gates only where measurement justifies it (`11-ai-agents.md` Q15).

**The reliability-vs-capability trade-off, stated plainly:** every degree of freedom you grant — more tools, more steps, more autonomy, longer context, more agents — adds capability and subtracts predictability. Reliability engineering for LLM systems is largely the discipline of **removing freedom the task does not need** (`11-ai-agents.md` Q22). Use the least powerful architecture that meets the quality bar.

**The reliability numbers worth internalising.** If a single step is 95% reliable:

```text
1 step:   95%
5 steps:  77%
10 steps: 60%
20 steps: 36%
```

Errors compound multiplicatively. This is the arithmetic behind "fewer steps," "verify intermediate results," and "prefer a chain over an agent." It is also why multi-agent systems with five handoffs disappoint: five 90% components give ~59% end-to-end. **Reducing step count is a reliability intervention, not just a cost one.**

### Interview Follow-ups

- What is the highest-value reliability investment? (Evaluation in CI. Everything else is one-off; the eval set is the mechanism that keeps reliability from decaying as the system changes.)
- How do you handle the fact that the model is non-deterministic? (Do not fight it — bound it. Temperature 0 where possible, structured output for anything downstream code consumes, verification of outputs, and statistical rather than exact-match testing.)

---

## Q16: How do you decide between RAG, fine-tuning, long context, and an agent for a given requirement?

### Answer

**These are not competing alternatives — they solve different problems and compose.** The mistake is treating them as a single choice.

| Approach | Solves | Does not solve |
|---|---|---|
| **RAG** | Access to private/current knowledge, attribution, freshness | Behaviour, format, style, reasoning ability |
| **Fine-tuning** | Behaviour, format, tone, domain style, task specialisation | Knowledge freshness, attribution |
| **Long context** | Reasoning over a large *given* input | Selecting from a large corpus (cost, latency) |
| **Agent** | Unknown control flow, actions on systems, iteration | Knowledge or behaviour by itself |
| **Prompting** | Almost everything, first | Consistency at the margins; token cost |

**The decision framework:**

```text
What is actually missing?

Knowledge the model does not have?
  Is it a large corpus, changing, or requiring attribution?  → RAG
  Is it a bounded document the user supplies per request?    → long context
  Is it stable, small, and needed on every request?          → put it in the prompt

Behaviour the model does not exhibit?
  Try prompting and few-shot first — it usually suffices
  Still inconsistent at high volume with a stable task?      → fine-tune
  Need a specific output structure?                          → structured output, not fine-tuning

Actions the model cannot take?                               → tools
  Is the sequence of actions knowable in advance?            → chain
  Does it depend on runtime discoveries?                     → agent

Reasoning the model cannot do?
  → a more capable model, or a reasoning model
  → decomposition into simpler steps
  (fine-tuning rarely adds reasoning ability)
```

**The key misunderstandings to correct:**

**"We'll fine-tune so it knows our documentation."** Poor fit. Fine-tuning teaches *patterns of behaviour*, not retrievable facts; injected facts are unreliable, unattributable, and stale the moment the documentation changes. Use RAG for knowledge (`10-rag.md` Q3).

**"Long context replaces RAG."** It does not, on cost and latency. Stuffing 250k tokens per query is roughly 60× the cost of retrieving 4k tokens, and TTFT suffers badly. What long context *did* change is RAG's job: it moved from "find the exact sentence" to "find the right documents," which allows larger chunks and less aggressive precision tuning (`10-rag.md` Q4).

**"We need an agent."** Usually not. If the steps are enumerable, a chain is cheaper, faster, more testable, and more reliable (`11-ai-agents.md` Q25).

**They compose, and the strongest systems combine them:**

```text
A fine-tuned model (consistent domain output format)
  + RAG (current, attributable knowledge)
  + tools (live data and actions)
  + a bounded agent loop for the hard tail of queries
  behind a router that sends the easy majority to a cheap deterministic path
```

**The pragmatic sequence for a new project:**

1. **Prompt a capable model.** Establish feasibility and the quality ceiling. Build the eval set here.
2. **Add retrieval** if it needs knowledge it lacks.
3. **Add tools** if it needs live data or actions.
4. **Add a bounded loop** only if control flow genuinely cannot be predetermined.
5. **Route by complexity** once you see the traffic distribution — this is where the cost wins are.
6. **Fine-tune last**, when you have volume, a stable task, a labelled dataset, and a measured gap that prompting cannot close.

Fine-tuning is last not because it is bad but because it is the least reversible: it requires a dataset, a training pipeline, evaluation of the trained model, and ongoing maintenance as the base model moves. The other levers are configuration changes.

### Interview Follow-ups

- When is fine-tuning clearly the right call? (High volume, narrow stable task, and a real gap: consistent structured extraction in a specialised domain, matching a house style, or replacing a frontier model with a small one on one well-defined task at a fraction of the cost. All require a labelled dataset you can maintain.)
- Can you fine-tune and use RAG together? (Yes, and it is often the best combination — fine-tune the model to use retrieved context well and to produce your required output format, then supply current knowledge via retrieval. The techniques are orthogonal.)

---

## Q17: Design a real-time content moderation system using LLMs.

### Answer

**Clarify:** content types (text, image, video)? Volume and peak? Latency budget — pre-publication blocking or post-publication review? Policy complexity? Appeals process? Regulatory requirements? Is human review available and at what capacity?

**Assume:** text and images, 10M items/day peak 500/second, must decide before publication with a 200 ms budget, ~40 policy categories, appeals required, human review capacity ~20k items/day (0.2%).

**Architecture — a cascade, because 10M/day at 200 ms cannot all reach an LLM:**

```text
Content submitted
  ↓
TIER 0 — deterministic (< 1 ms, 100% of traffic)
  hash match against known-violating content, blocklists, regex for
  banned patterns, account reputation, rate anomalies
  → definite match: BLOCK
  ↓ (remainder)
TIER 1 — small classifier (~10 ms, ~99% of traffic)
  fine-tuned transformer classifier, multi-label over 40 categories
  → high-confidence violation:  BLOCK
  → high-confidence clean:      ALLOW           ← the vast majority exits here
  → uncertain:                  escalate
  ↓ (~2-5%)
TIER 2 — LLM with policy reasoning (~500 ms, ~2-5% of traffic)
  full policy text + content + context (thread, author history)
  structured output: category, severity, reasoning, confidence
  → confident:  ACT
  → uncertain:  escalate
  ↓ (~0.2%)
TIER 3 — human review (minutes to hours)
  queue prioritised by severity × reach
  decisions feed back into training data
```

**Why a cascade is the only viable design.** An LLM call at 500 ms and ~$0.002 across 10M items is $20,000/day and cannot meet a 200 ms budget. The cascade puts cheap deterministic checks in front, a fast classifier in the middle, and reserves LLM reasoning for the genuinely ambiguous minority. Cost drops to roughly:

```text
Tier 0:  10M × ~$0            = $0
Tier 1:  10M × $0.00002       = $200/day     (self-hosted small model)
Tier 2:  300k × $0.002        = $600/day
Tier 3:  20k × human cost     (the dominant cost, and the reason to minimise it)
                              ≈ $800/day in model spend
```

**The latency solution for the 200 ms budget.** Tier 2 at 500 ms breaks it. Options, and the trade-off must be explicit:
- **Optimistic publication:** publish immediately, run tiers 2–3 asynchronously, retract if violating. Good for low-severity categories; unacceptable for the worst content.
- **Severity-dependent blocking:** for categories where a false negative is catastrophic (CSAM, credible threats), hold the content pending the full check. For everything else, publish optimistically.
- **This is a product/policy decision, not just an engineering one** — say so.

**What the LLM tier adds that a classifier cannot:**

| Capability | Why it needs an LLM |
|---|---|
| **Policy reasoning** | 40 nuanced categories with exceptions; classifiers learn labels, not policy |
| **Context sensitivity** | Reclaimed slurs, quoted content, satire, counter-speech, educational discussion |
| **Novel violations** | No training data exists yet for an emerging harm pattern |
| **Explanations** | Required for user notices and appeals |
| **Rapid policy change** | Update the prompt today, not a retraining cycle |

That last point is strategically important: when a policy changes or a new abuse pattern emerges, the LLM tier can enforce it **the same day** via a prompt update, while the classifier catches up through the next training cycle. The LLM tier is both the nuance layer and the rapid-response layer.

**The flywheel — the most important architectural property.** Tier 2 and Tier 3 decisions become labelled training data for Tier 1. Over time the classifier absorbs the LLM's judgement, more traffic resolves cheaply at Tier 1, and escalation volume falls. Explicitly design for this: log every escalated decision with its reasoning, retrain the classifier regularly, and monitor the escalation rate as the key efficiency metric.

**Metrics — and the trade-off is the point:**

| Metric | Note |
|---|---|
| **False negative rate by category, weighted by severity** | The safety metric; the only one that matters for the worst categories |
| **False positive rate** | The user-harm metric; over-blocking is a real cost, not a safe default |
| Precision/recall per category | Aggregate numbers hide the categories that matter |
| Escalation rate per tier | Efficiency and cost |
| Appeal rate and overturn rate | **The most honest false-positive signal you have** |
| Latency p99 per tier | Product requirement |
| Time-to-enforcement for new patterns | Responsiveness |

**Set thresholds per category by asymmetry.** For CSAM or credible threats, accept a high false-positive rate to drive false negatives near zero. For spam or mild profanity, favour precision — over-blocking legitimate speech has real cost. A single global threshold is wrong, and saying so demonstrates you understand the domain.

**Additional design requirements:**
- **Adversarial robustness.** Users actively evade: unicode homoglyphs, spacing, image text, coded language. Normalise aggressively at Tier 0, run OCR on images, and monitor for evasion patterns.
- **Multilingual.** Quality varies sharply by language; measure per language and do not assume English performance transfers.
- **Appeals** need the reasoning from Tier 2/3 and a human path — and overturn rates are your best false-positive estimate.
- **Reviewer welfare.** Human reviewers see the worst content; blur/preview tooling, rotation, and support are real requirements, not niceties.
- **Auditability.** Regulatory regimes increasingly require transparency reporting on enforcement volumes and error rates. Log accordingly.

### Interview Follow-ups

- Why not fine-tune one large model on everything? (Latency and cost at 10M/day, and policy agility — a prompt change ships in an hour, a retraining cycle takes weeks. The cascade gets classifier economics with LLM nuance where it matters.)
- How do you handle a policy change? (Update the Tier 2 prompt immediately for same-day enforcement, backfill by re-running recent content through Tier 2, collect the new labels, and retrain Tier 1 on the next cycle. The LLM tier is the rapid-response mechanism.)

---

## Q18: What are the most common mistakes in LLM system design?

### Answer

A consolidated list, roughly in order of how often it causes real damage.

**1. No evaluation.** Shipping with vibes, then being unable to tell whether any change helps. The most consequential mistake, because it makes every other problem unfixable. Build 30–50 real cases before optimising anything.

**2. Over-engineering the architecture.** A multi-agent system for what a single prompted call plus retrieval would do. Start at the top of the escalation ladder (Q2) and move down only under measured pressure.

**3. Ignoring cost until the invoice arrives.** Do the arithmetic during design (Q4). A system that works but costs $200k/month for $50k of value gets cancelled.

**4. Treating the model as the system.** The data path — ingestion, chunking, freshness, permissions — is usually where quality actually comes from and where most production failures originate.

**5. Enforcing constraints in the prompt.** Anything that matters — permissions, spend limits, allowed actions — must be enforced in code. A prompt is a preference (Q7).

**6. No versioning of prompts, models, retrieval config, or index.** Then a regression appears and nobody can identify what changed or reproduce the old behaviour (Q11).

**7. Uncapped anything.** Unbounded agent steps, unbounded tool output, unbounded context growth, unbounded retries. Each is a latent outage or cost incident (Q15).

**8. Retrieving too many chunks.** Sending 20 chunks instead of reranking to 5 costs more, is slower, *and* is often worse quality due to dilution and lost-in-the-middle.

**9. Building generation before measuring retrieval.** In RAG, most quality problems are retrieval problems. Measure recall@k first — you cannot generate a good answer from bad evidence (`10-rag.md` Q13).

**10. Caching across a permission boundary.** A cache keyed on question text alone leaks one tenant's data to another. The tenant must be in the key (Q9, Q10).

**11. No degradation strategy.** When the reranker, vector DB, or provider fails, the system either errors out entirely or — worse — generates a confident answer with no evidence (Q15).

**12. Punishing refusal.** If the prompt, the product, and the eval set all reward answering, the model guesses. Make "I don't know" a first-class, rewarded output (`10-rag.md` Q14).

**13. Ignoring latency.** A technically excellent 40-second response is a product failure. Stream, and show progress (Q5).

**14. Semantic caching on high-stakes traffic.** "PTO policy for engineers" vs "for interns" is 0.96 similar with a different correct answer (Q9).

**15. No observability.** No traces means production failures are unfixable, because the cause is usually several steps before the symptom (Q8).

**16. Assuming long context solves retrieval.** It changes the shape of the problem; it does not remove the need to select what to send (Q16).

**17. Fine-tuning for knowledge.** Wrong tool: unreliable, unattributable, stale on the first document update (Q16).

**18. Building an agent for a knowable workflow.** Paying 10× the cost and losing all predictability for flexibility you do not need (`11-ai-agents.md` Q25).

**19. Not designing for deletion.** GDPR erasure across index, cache, checkpoints, memory, and logs is genuinely hard to retrofit (Q10).

**20. No human escalation path.** Every system encounters cases it should not handle. Without an exit, it produces a confident wrong answer instead.

**The meta-lesson.** Almost all of these reduce to two habits:

> **Measure before you optimise, and enforce in code what you cannot afford to merely request.**

The first prevents you from improving the wrong thing and from shipping regressions. The second is the boundary between a system that behaves well in a demo and one you can operate in production. Every item above is an instance of one or the other.

### Interview Follow-ups

- If you inherited a poorly-performing LLM system, what would you do first? (Add tracing and build an eval set from real failures. You cannot fix what you cannot measure, and the failure distribution will almost certainly surprise you — see `10-rag.md` Q21.)
- Which mistake is hardest to recover from? (No evaluation, combined with no versioning. You end up with a system nobody dares change, because any modification might break something and nobody would know.)

---
