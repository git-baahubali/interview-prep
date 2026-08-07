# AI/ML/LLM Engineer Interview Handbook

A modular, revision-oriented question bank for AI/ML and LLM Engineer interviews. Every question has a complete written answer — explanations, comparison tables, code, worked arithmetic, algorithm walkthroughs, and interview follow-ups — written for someone with roughly 2–5 years of software or AI engineering experience.

Each topic lives in its own file with its own independent question numbering, so questions can be added, removed, reordered, or rewritten without touching any other file.

**Total questions: 256**, plus a **[100-problem easy Python warm-up bank](14-easy-python-problems.md)** for screening rounds.

---

## Topics

| # | Topic | Questions | Focus |
|---|---|---|---|
| 01 | [Python Fundamentals](01-python-fundamentals.md) | 8 | Mutability, scoping, generators, the GIL, idiomatic patterns |
| 02 | [DSA & Problem Solving](02-dsa-problem-solving.md) | 8 | Core patterns with complexity analysis on every solution |
| 03 | [Tabular Data Preprocessing](03-tabular-data-preprocessing.md) | 7 | Missing data, encoding, scaling, and leakage |
| 04 | [NLP & Text Preprocessing](04-nlp-text-preprocessing.md) | 8 | Tokenisation, subword algorithms, normalisation, long text |
| 05 | [Machine Learning Fundamentals](05-machine-learning-fundamentals.md) | 22 | Bias-variance, metrics, gradient descent, clustering, PCA, drift |
| 06 | [Transformers, LLMs & Generative AI](06-transformers-llms-generative-ai.md) | 32 | Attention, positional encoding, KV cache, training, fine-tuning, serving |
| 07 | [Prompt Engineering](07-prompt-engineering.md) | 16 | Structure, few-shot, CoT, structured output, caching, guardrails |
| 08 | [Embeddings](08-embeddings.md) | 22 | The full evolution from one-hot to modern models, and every key comparison |
| 09 | [Vector Databases & Retrieval](09-vector-databases-retrieval.md) | 26 | ANN, HNSW in depth, IVF, PQ, similarity metrics, hybrid search, reranking |
| 10 | [RAG](10-rag.md) | 26 | Chunking, query transformation, context assembly, citations, evaluation |
| 11 | [AI Agents](11-ai-agents.md) | 31 | ReAct, tool calling, memory, planning, multi-agent, guardrails, HITL |
| 12 | [LangGraph](12-langgraph.md) | 32 | State, nodes, edges, reducers, checkpointing, interrupts, subgraphs |
| 13 | [LLM System Design](13-llm-system-design.md) | 18 | Architecture, cost and latency engineering, evaluation, safety, design problems |

### Supplementary

| # | File | Items | Focus |
|---|---|---|---|
| 14 | [Easy Python Problems](14-easy-python-problems.md) | 100 | Warm-up drill bank — Armstrong, palindrome, Fibonacci, prime, FizzBuzz, and the rest, with solutions, complexity, and the edge case each one is really testing |
| — | [`easy-python-problems.csv`](easy-python-problems.csv) | 100 | The same 100 problems as a filterable spreadsheet for tracking revision progress |

The 100 easy problems are counted separately because they are a drill bank rather than interview questions — no `Q1:`/`Answer` structure, just problem, solution, complexity, trap. Use them for screening rounds and to warm up before a coding interview; use `02-dsa-problem-solving.md` for the pattern-based questions that follow.

---

## Recommended learning order

**If you have time to work through everything**, read in file order — the sequence is deliberately cumulative. Later files assume earlier ones and cross-reference them by filename and question number.

**If you are preparing for a specific role, use one of these paths.**

### LLM / AI Engineer (the most common target)

```text
06 Transformers & LLMs  →  07 Prompt Engineering  →  08 Embeddings
   →  09 Vector Databases  →  10 RAG  →  11 AI Agents
   →  12 LangGraph  →  13 LLM System Design
```

Fill in `05 Machine Learning Fundamentals` if your background is not ML, and `01`/`02` if the loop includes a coding screen. If there is an online screening round, drill the Very High frequency rows of `14 Easy Python Problems` first — those rounds test speed on the classics, not depth.

### RAG / Search-focused role

```text
08 Embeddings  →  09 Vector Databases  →  10 RAG  →  04 NLP  →  13 LLM System Design
```

`09` Q5–Q11 is the HNSW block; expect these to be pushed hard in retrieval interviews.

### Agent / Applied AI role

```text
07 Prompt Engineering  →  11 AI Agents  →  12 LangGraph  →  10 RAG  →  13 LLM System Design
```

### Traditional ML with an LLM component

```text
03 Tabular Data  →  05 ML Fundamentals  →  01 Python  →  02 DSA
   →  06 Transformers  →  10 RAG
```

### Final-week revision

Read the vocabulary-summary questions first — they are dense maps of each area's terminology:

- `11-ai-agents.md` Q31 — agent vocabulary
- `12-langgraph.md` Q32 — LangGraph vocabulary
- `13-llm-system-design.md` Q18 — the most common system design mistakes
- `08-embeddings.md` Q9 — the embedding evolution table
- `09-vector-databases-retrieval.md` Q7 and Q11 — what an HNSW index contains and what its parameters do

Then work backwards through the `## Advanced` sections of the topics your interview will emphasise.

---

## How the handbook is organised

**One file per topic.** No file depends on another file's numbering. Cross-references are written as `` `10-rag.md` Q13 `` so they survive edits elsewhere.

**Questions are grouped by difficulty** using `## Easy`, `## Intermediate`, and `## Advanced` headings where the distinction is meaningful. Some topics are uniformly foundational and have no Advanced section.

**Every question follows the same shape:**

```markdown
## Q1: <question>

### Answer
<the direct answer first, then the explanation>

### Example
<code, table, or worked calculation — omitted when it adds nothing>

### Interview Follow-ups
<the questions an interviewer asks next>

---
```

**Conventions used throughout:**

- Important concepts are explained as *what it is → why it exists → how it works → when it is used*.
- Comparison questions lead with a table, then explain the differences underneath.
- Algorithm questions (HNSW, IVF, Product Quantization, K-Means, DBSCAN, PCA, gradient descent) follow a fixed structure: purpose, core idea, step-by-step operation, key parameters, advantages, limitations, use cases.
- DSA solutions always state **Time Complexity** and **Space Complexity** with the reasoning.
- Intuition comes before mathematics.

---

## Adding your own questions

The handbook is designed to be extended. To add a question:

1. Open the relevant topic file.
2. Add the question at the end of the appropriate difficulty section, numbered one higher than the last `## Qn` in that file.
3. Follow the question format above.
4. Update `**Questions:**` in that file's header and its row in the table above, plus the total at the top of this README.

Nothing else needs to change. Renumbering is only ever required within a single file, and only if you insert or delete a question in the middle — appending never requires it.

To reorder or remove questions, edit that one file. Cross-references from other files point at question numbers, so if you renumber, search the directory for `` `<filename>.md` Q `` to update any references.
