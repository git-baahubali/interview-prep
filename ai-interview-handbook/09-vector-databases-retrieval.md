# Vector Databases & Retrieval

ANN indexing algorithms in depth — HNSW, IVF, IVF-PQ — plus similarity metrics, filtering, hybrid retrieval, and reranking. This is the file to know cold for a RAG-focused interview.

**Questions:** 26

---

## Easy

---

## Q1: What is a vector database, and how does it differ from a traditional database?

### Answer

A **vector database** stores high-dimensional vectors alongside metadata and answers **approximate nearest-neighbour** queries: "give me the k vectors most similar to this one."

| | Relational / document DB | Vector database |
|---|---|---|
| Primary query | Exact match, range, join | Nearest neighbour by distance |
| Index structure | B-tree, hash, inverted index | HNSW, IVF, PQ, LSH |
| Query result | Rows that satisfy a predicate | Top-k ranked by similarity |
| Result correctness | Exact | **Approximate** (tunable) |
| "WHERE" equivalent | The whole query | A *filter* combined with similarity search |
| Scaling concern | Rows, joins, transactions | Vector count × dimensions, memory footprint |

**Why a specialised system is needed.** Exact nearest-neighbour search over 10M 768-dimensional vectors requires 10M distance computations per query — about 7.7 billion floating-point operations. That is hundreds of milliseconds to seconds, per query. Traditional indexes cannot help: a B-tree indexes a *scalar ordering*, and there is no useful total ordering of points in 768-dimensional space. So vector databases implement fundamentally different index structures that trade exactness for speed.

**What a vector database gives you beyond a bare ANN library:**
- Persistence, durability, and crash recovery
- CRUD on vectors (updates and deletes are genuinely hard with graph indexes)
- **Metadata filtering** combined with vector search
- Horizontal sharding and replication
- Multi-tenancy and access control
- Hybrid (sparse + dense) search
- Monitoring, backup, and operational tooling

**The landscape:**

| Category | Examples |
|---|---|
| Purpose-built | Pinecone, Weaviate, Qdrant, Milvus, Vespa |
| Extensions to existing DBs | pgvector (Postgres), Redis, MongoDB Atlas Vector Search, Elasticsearch/OpenSearch |
| Embedded / in-process | Chroma, LanceDB, FAISS (a library, not a DB), DuckDB VSS |

**Practical guidance for interviews:** do not reach for a dedicated vector database reflexively. Under ~1M vectors, `pgvector` in a Postgres you already run is often the right answer — one system, transactional consistency with your metadata, no new operational surface. Purpose-built systems earn their keep at scale, with high write throughput, or when you need advanced filtering and hybrid search.

### Interview Follow-ups

- When would you use FAISS instead of a vector database? (FAISS is a library — no persistence, no filtering, no CRUD, no server. Ideal for a static in-process index or research; not for a live application with updates.)
- Why is `SELECT ... ORDER BY embedding <-> query LIMIT 10` in Postgres without an index still useful? (It is exact brute force — perfect for small tables and as a ground-truth baseline for measuring your ANN index's recall.)

---

## Q2: What is approximate nearest neighbour (ANN) search, and why do we accept approximation?

### Answer

**Exact k-NN** compares the query against every vector — guaranteed correct, O(n·d) per query, and unusable at scale.

**ANN** uses an index that examines only a small fraction of the data, returning *most* of the true nearest neighbours *most* of the time. It trades a small amount of correctness for orders of magnitude in speed.

**Concrete comparison — 10M vectors, 768 dimensions:**

| Method | Latency | Recall@10 | Memory |
|---|---|---|---|
| Exact (flat) | ~500 ms | 1.00 | 30 GB |
| HNSW | ~2 ms | 0.98 | 45 GB |
| IVF (nprobe=32) | ~5 ms | 0.95 | 31 GB |
| IVF-PQ | ~3 ms | 0.85 | 2 GB |

(Illustrative — actual numbers depend on hardware, data distribution, and parameters.)

**Why approximation is acceptable — the key argument.** Retrieval feeds a downstream consumer that is itself imprecise:
1. The *embedding* is already an approximation of meaning. A vector 3 positions lower in the true ranking is often equally relevant.
2. In RAG, you retrieve k=50 and rerank to 5. Missing the true #7 result rarely changes the final answer, and a reranker fixes ordering errors anyway.
3. Relevance labels themselves are fuzzy — human annotators disagree on what the "correct" top-10 is.

So the marginal value of the last 2% of recall is very low, while the cost of exactness is 100–1000× latency. That is a trivially good trade.

**Where it is NOT acceptable:** deduplication with a hard threshold, legal/compliance retrieval that must be provably complete, small corpora where exact search is already fast, and evaluation baselines (you need exact results to *measure* ANN recall).

**The universal ANN trade-off triangle:** recall, latency, and memory. Every algorithm and every parameter moves you along these three axes; you cannot maximise all three.

### Interview Follow-ups

- How do you measure recall of an ANN index? (Compute exact ground truth for a sample of queries with brute force, then measure the overlap with your ANN results. Do this on your real data — recall on random vectors is meaningless.)
- What is "recall@k vs the exact top-k"? (The fraction of the true top-k that the ANN search returned. Note it is not the same as retrieval recall against human relevance labels — two different meanings of "recall" that interviews often conflate.)

---

## Q3: What is brute-force / flat search, and when should you use it?

### Answer

**What it is.** Compute the distance from the query to **every** vector in the collection, sort, and return the top k. No index structure at all — just a linear scan.

**Cost:** O(n · d) per query. For 1M vectors at 768 dimensions that is ~768M multiply-adds — roughly 10–50 ms with SIMD-optimised BLAS on a modern CPU, and under 5 ms on a GPU.

**Advantages:**
- **Perfect recall — 1.00, by definition.** No tuning, no recall/latency trade-off.
- No index build time; new vectors are immediately searchable.
- Minimal memory overhead: just the vectors themselves.
- Trivial deletes and updates.
- Filtering is free and exact — apply the predicate, then scan only what survives.
- Deterministic, debuggable, no parameters to get wrong.

**Limitations:** latency grows linearly with collection size, so it becomes unusable somewhere between 100k and a few million vectors depending on your latency budget and hardware.

**When flat search is genuinely the right answer — and interviewers like this answer:**

| Scenario | Why |
|---|---|
| Under ~100k vectors | ANN adds complexity for latency you did not need |
| Per-tenant collections that are individually small | 10,000 tenants × 500 vectors each — each search is trivially fast |
| **Highly selective filters** | If a filter narrows 10M vectors to 200, scan the 200 exactly. This beats ANN on both accuracy and speed. |
| Establishing ground truth | You need exact results to measure your ANN index's recall |
| Reranking stage | Rescoring 100 candidates with full-precision vectors after a quantised first pass |
| Correctness-critical retrieval | Compliance/legal, deduplication with a hard threshold |

**The rule of thumb:** start with flat search. Add an ANN index when you have measured that flat search is too slow, not before. Premature ANN indexing is a common source of unnecessary complexity and silent recall loss.

### Interview Follow-ups

- Why is flat search sometimes faster than HNSW under a restrictive filter? (HNSW's graph traversal breaks down when most nodes are filtered out — you either skip through a mostly-invalid graph or over-fetch heavily. Scanning 200 surviving vectors is faster and exact.)
- How does FAISS `IndexFlatIP` differ from `IndexFlatL2`? (Inner product versus L2 distance — identical ranking on normalised vectors, and IP is marginally cheaper since it skips the norm terms.)

---

## Q4: What are the main similarity metrics, and how do you choose?

### Answer

| Metric | Formula | Range | Higher is | Magnitude-sensitive |
|---|---|---|---|---|
| **Cosine similarity** | (a·b) / (‖a‖‖b‖) | [−1, 1] | more similar | No |
| **Dot product (inner product)** | Σ aᵢbᵢ | (−∞, ∞) | more similar | **Yes** |
| **Euclidean (L2) distance** | √Σ(aᵢ−bᵢ)² | [0, ∞) | less similar | Yes |
| Manhattan (L1) | Σ\|aᵢ−bᵢ\| | [0, ∞) | less similar | Yes |
| Hamming | count of differing bits | integer | less similar | n/a (binary) |

**The identity you must know.** For **L2-normalised** vectors (‖a‖ = ‖b‖ = 1):

```text
a · b            = cos(a, b)
‖a − b‖²         = 2 − 2·cos(a, b)
```

So on normalised vectors, **cosine, dot product, and Euclidean distance produce identical rankings.** This is why many vector databases normalise at ingestion and use dot product internally — it is the cheapest of the three to compute (no norms, no square root; just a fused multiply-add per dimension).

**How to choose — the decision rule:**

1. **Use the metric the embedding model was trained with.** This is not a preference, it is a requirement. Check the model card. Most text embedding models (sentence-transformers, OpenAI, E5, BGE) are trained with cosine similarity and produce normalised or near-normalised output.
2. If the model outputs normalised vectors, dot product is the efficient equivalent of cosine — use it.
3. Some models are deliberately trained for **dot product** with meaningful magnitudes, where a larger norm encodes importance or confidence. **Normalising those destroys information.**
4. Euclidean distance is the right choice for genuinely geometric data (coordinates, image feature spaces) where absolute position matters.
5. Hamming distance for binary-quantised vectors.

**The classic bug:** using L2 distance on unnormalised text embeddings. Document length inflates vector norms, so a long document sits "far" from a short query purely because of magnitude, regardless of topical relevance. Symptom: consistently poor retrieval that looks like a model problem but is a configuration problem.

**Dot product's known bias:** it rewards large norms, so long documents systematically score higher. Sometimes that is desirable (longer documents genuinely contain more), usually it is not. Normalising removes it.

### Interview Follow-ups

- Why do vector databases index "inner product" rather than cosine? (Cosine requires computing norms at query time; if vectors are pre-normalised at ingest, IP *is* cosine and is cheaper.)
- Does the triangle inequality matter for ANN indexes? (Yes for metric-tree structures like Annoy's or ball trees; HNSW and IVF do not strictly require it, which is why they work with cosine "distance" that is not a true metric.)

---

## Intermediate

---

## Q5: What is HNSW?

### Answer

**HNSW (Hierarchical Navigable Small World)** is a graph-based ANN index. It builds a multi-layer proximity graph over your vectors and answers queries by greedy traversal from the top layer down.

**The two ideas in the name:**

1. **Navigable Small World.** A graph where any node can be reached from any other in a small number of hops, using only *local* greedy decisions (always move to the neighbour closest to the target). Real-world social networks have this property — "six degrees of separation" — because they combine dense local clustering with a few long-range links.

2. **Hierarchical.** Multiple layers. The top layers are sparse with long-range links (for coarse, fast navigation); lower layers are dense with short links (for fine-grained accuracy). This is directly analogous to a **skip list**, and HNSW's author describes it that way: a skip list gives O(log n) search over a 1-D ordering; HNSW generalises the idea to a proximity graph in any metric space.

**The core intuition.** Think of navigating from London to a specific house in Tokyo. You do not compare your position to every building on Earth. You take a plane to Tokyo (top layer, huge jumps), then a train to the district (middle layer), then walk street by street (bottom layer). Each layer narrows the search region using progressively finer-grained moves.

**Why it dominates.** HNSW consistently sits at the top of ANN benchmarks for the recall-versus-latency trade-off on real-world data, and it supports incremental insertion (no separate training step). Its cost is memory — it stores the full vectors plus the graph edges.

**Where you find it:** essentially everywhere. FAISS, Qdrant, Weaviate, Milvus, Elasticsearch/OpenSearch, Redis, pgvector, Vespa, and Lucene all implement it. It is the de facto default index for vector search.

### Interview Follow-ups

- Where does the "small world" property come from in HNSW? (The layer assignment: a few nodes exist in high layers and have long-range edges; most nodes only exist at layer 0 with short-range edges.)
- Why is the skip-list analogy apt? (Same probabilistic level assignment, same coarse-to-fine descent, same O(log n) expected behaviour.)

---

## Q6: Is HNSW an indexing algorithm, a search algorithm, or both?

### Answer

**Both** — and this is a deliberately probing question, because the two halves are separable, have different parameters, and have different cost profiles.

HNSW specifies two distinct algorithms that share one data structure:

**1. The indexing (construction) algorithm.** How to build the multi-layer graph: how to assign each new node a maximum layer, how to find its candidate neighbours, and how to select which edges to keep. Governed by `M`, `M_max0`, `efConstruction`, and `mL`. This runs at ingestion time, once per vector.

**2. The search (query) algorithm.** How to traverse the existing graph to find the k nearest neighbours: greedy descent through upper layers, then a best-first beam search at layer 0. Governed by `efSearch` (and `k`). This runs per query.

**Why the distinction matters practically:**

| | Indexing | Search |
|---|---|---|
| When it runs | At insert time | At query time |
| Key parameters | `M`, `efConstruction` | `efSearch` |
| Tunable after the fact? | **No** — changing them requires a rebuild | **Yes** — per query, no rebuild |
| Cost paid in | Build time, memory | Query latency |
| Affects | Graph quality, memory footprint, the achievable recall ceiling | The recall you actually achieve, latency |

**The consequence you should state in an interview:** `efSearch` is your runtime dial — you can raise it to buy recall on the spot, even per-query, with zero rebuild. `M` and `efConstruction` are commitments made at build time. Get those wrong and you have capped your quality ceiling until you re-index a corpus that may take hours. Choose `M` and `efConstruction` generously; tune `efSearch` continuously.

**The framing that shows understanding:** the index *is* the graph — a data structure. HNSW is the pair of procedures that build it and traverse it. The graph is what gets persisted; both algorithms are needed to make it useful.

### Interview Follow-ups

- Can you change `efSearch` per query? (Yes — most implementations expose it at query time, which lets you serve a "high accuracy" tier and a "fast" tier from one index.)
- What happens if you set `efSearch` below `k`? (You cannot return k results reliably; implementations clamp `efSearch` to at least `k`.)

---

## Q7: What does an HNSW index actually contain?

### Answer

Three things:

**1. The vectors themselves.** Full-precision (usually float32) copies of every vector — HNSW is *not* a compression method. This dominates memory:

```text
vectors = n × dims × 4 bytes
```

**2. The graph: an adjacency list per node, per layer.** For each node, at each layer it exists in, a list of neighbour node ids.

```text
graph ≈ n × (M_max0 + M × avg_upper_layers) × 4 bytes   (4 bytes per node id)
      ≈ n × M × 2 × 4 bytes                              (common approximation)
```

Layer 0 contains **every** node and typically allows up to `2M` neighbours (`M_max0 = 2M`). Upper layers contain exponentially fewer nodes and allow up to `M` neighbours each. The geometric decay means the total edge count is roughly `n × 2M` — dominated by layer 0.

**3. Metadata.** The entry point (the single node in the highest layer where every search begins), each node's maximum layer, the parameters, and (in a database) a mapping from internal ids to external ids, deletion tombstones, and any payload/metadata used for filtering.

**Worked memory example — 1M vectors, 768 dims, M=16:**

```text
vectors:  1,000,000 × 768 × 4 bytes            = 3,072 MB
graph:    1,000,000 × 16 × 2 × 4 bytes         =    128 MB
overhead: ids, tombstones, layer info          ≈     20 MB
                                        total  ≈  3,220 MB
```

**The key insight:** for typical dimensions, the **vectors dominate** — the graph is only ~4% of the footprint here. That has two consequences:

1. Reducing `M` saves relatively little memory but costs recall — usually a bad trade.
2. If you are memory-constrained, **compress the vectors**, not the graph. That is exactly what HNSW-PQ, scalar quantisation, and binary quantisation do, and it is why they deliver order-of-magnitude savings where tuning `M` cannot.

Note the graph share grows for low-dimensional vectors: at 128 dims with M=32, the graph is a much larger fraction and `M` becomes a real memory lever.

### Interview Follow-ups

- Must the whole index be in RAM? (For good latency, effectively yes — graph traversal is a random-access pattern, and disk-backed HNSW suffers badly. This is what disk-oriented designs like DiskANN address with a different graph structure and layout.)
- What does a delete do to an HNSW graph? (Usually a tombstone — the node stays in the graph as a routing waypoint but is excluded from results. Accumulated tombstones degrade recall and waste memory, so periodic compaction/rebuild is required.)

---

## Q8: How is the HNSW graph constructed?

### Answer

Nodes are inserted **one at a time**; there is no global training phase.

**Step 1 — assign the new node a maximum layer.** Draw it from an exponentially decaying distribution:

```text
l = floor( -ln(uniform(0,1)) × mL )        where mL = 1 / ln(M)
```

This gives roughly: ~50% of nodes only at layer 0 with M=2... in practice with M=16, about 94% of nodes exist only at layer 0, ~6% reach layer 1, ~0.4% layer 2, and so on. The hierarchy is thin at the top by construction, which is what makes upper-layer navigation cheap.

**Step 2 — greedy descent through the layers above `l`.** Start at the global entry point in the top layer. At each layer, greedily move to the neighbour closest to the new node until no neighbour is closer. Carry that local minimum down as the entry point for the next layer. Only *one* candidate is tracked here — this phase is pure navigation, cheap and fast.

**Step 3 — beam search and connect, for each layer from `l` down to 0.**
1. Run a best-first search from the entry point, maintaining a candidate list of size **`efConstruction`**. This is the expensive part: a larger `efConstruction` explores more of the graph and finds better neighbour candidates.
2. From those candidates, **select up to `M` neighbours** (up to `M_max0 = 2M` at layer 0).
3. Add bidirectional edges between the new node and each selected neighbour.
4. **Prune** any neighbour that now exceeds its edge limit, re-running the same selection heuristic on its edge list.

**Step 4 — the neighbour selection heuristic (the subtle and important part).** Do not simply take the M closest candidates. HNSW uses a **diversity-preserving heuristic**: consider candidates in order of increasing distance, and keep a candidate only if it is closer to the new node than it is to any already-selected neighbour.

*Why this matters enormously.* Taking the M nearest neighbours creates a graph where all your edges point into the same dense cluster. The graph becomes locally over-connected and globally disconnected, so greedy search gets trapped in local minima and recall collapses. The heuristic deliberately keeps some *further* neighbours that lie in *different directions*, preserving long-range connectivity and making the graph genuinely navigable. This is the single most important implementation detail in HNSW — it is what turns a proximity graph into a *navigable* one.

**Step 5 — update the entry point** if the new node's layer exceeds the current maximum.

**Complexity:** O(log n) expected per insertion, so O(n log n) to build. Build time is dominated by `efConstruction` and is highly parallelisable across insertions (with locking on edge lists).

### Example

```python
# Simplified construction outline — illustrative, not production code.
def insert(graph, new_node, M, ef_construction):
    level = int(-math.log(random.random()) * (1 / math.log(M)))
    entry = graph.entry_point

    # Phase 1: cheap greedy descent through layers above `level`
    for layer in range(graph.max_layer, level, -1):
        entry = greedy_search_one(graph, new_node, entry, layer)

    # Phase 2: beam search + connect at each layer from `level` down to 0
    for layer in range(min(level, graph.max_layer), -1, -1):
        candidates = beam_search(graph, new_node, entry, ef_construction, layer)
        max_edges = 2 * M if layer == 0 else M

        neighbours = select_diverse(new_node, candidates, max_edges)   # the heuristic
        for nb in neighbours:
            graph.add_edge(new_node, nb, layer)
            if graph.degree(nb, layer) > max_edges:
                graph.set_edges(nb, select_diverse(nb, graph.edges(nb, layer), max_edges), layer)

        entry = candidates[0]

    if level > graph.max_layer:
        graph.entry_point, graph.max_layer = new_node, level


def select_diverse(target, candidates, max_edges):
    """Keep a candidate only if it is closer to target than to any kept neighbour."""
    kept = []
    for c in sorted(candidates, key=lambda x: dist(target, x)):
        if len(kept) >= max_edges:
            break
        if all(dist(target, c) < dist(c, k) for k in kept):
            kept.append(c)
    return kept
```

### Interview Follow-ups

- Why insert one node at a time rather than building globally? (It enables incremental updates with no retraining — a major operational advantage over IVF, which needs a training pass over a sample.)
- Does insertion order affect the final graph? (Yes, slightly — the graph is not unique. Quality is robust in practice, but it means two builds of the same data are not bit-identical.)

---

## Q9: How does a query traverse the HNSW graph?

### Answer

**Phase 1 — greedy descent (layers max → 1).** Coarse navigation.

```text
current = entry_point
for layer from max_layer down to 1:
    repeat:
        examine current's neighbours at this layer
        if some neighbour is closer to the query than current:
            current = that closest neighbour
        else:
            break        # local minimum at this layer
```

Only a single "current best" is tracked. Each layer's local minimum becomes the next layer's starting point. Because upper layers are sparse with long edges, this phase covers enormous distance in very few distance computations.

**Phase 2 — best-first beam search at layer 0.** Accurate refinement.

```text
candidates    = min-heap of nodes to explore, seeded with the layer-1 result
found         = max-heap of the best results so far, size efSearch
visited       = set

while candidates not empty:
    c = closest unexplored candidate
    if dist(c, query) > worst distance in `found` and found is full:
        break                                   # cannot improve — stop
    for each neighbour n of c at layer 0:
        if n not visited:
            mark visited
            if dist(n, query) < worst in `found` or found not full:
                push n onto candidates and into found
                if len(found) > efSearch: pop the worst
return top k from found
```

**Phase 2 is where accuracy comes from.** Maintaining `efSearch` candidates rather than just one lets the search explore multiple promising directions simultaneously and escape local minima — the exact failure mode that pure greedy search suffers.

**The stopping condition is the efficiency trick:** once the closest unexplored candidate is farther than the worst result already found, no unexplored node can improve the result set, so the search terminates. This is what keeps the number of distance computations small.

**Concrete numbers — 1M vectors, M=16, efSearch=100:** roughly 1,000–3,000 distance computations, versus 1,000,000 for brute force. About a 300–1000× reduction, at ~98% recall.

**Why `efSearch` ≥ `k` is required:** you cannot return 10 results from a candidate list of 5. Implementations clamp it.

**Relationship between the parameters:** `efSearch` controls the breadth of exploration and is the runtime recall dial. `M` determines how many neighbours each hop can consider — a higher `M` means better connectivity and fewer hops needed, so higher recall at a given `efSearch`.

### Interview Follow-ups

- Why is greedy search alone insufficient? (It gets stuck in local minima — a node with no closer neighbour that is nevertheless not the global nearest. The beam keeps alternatives alive.)
- What is the cost profile of a query? (Almost entirely distance computations and random memory access. It is memory-latency-bound, which is why the index wants to be in RAM and why cache locality optimisations matter.)

---

## Q10: Why does HNSW have multiple layers?

### Answer

**To make the search logarithmic instead of linear in the number of hops.**

**The problem with a single-layer proximity graph** (which is what NSW, HNSW's predecessor, was). If every node connects only to its ~M nearest neighbours, the graph has excellent local structure but terrible long-range structure. Starting from a random node and greedily walking toward the query means traversing the space in small steps — the number of hops grows roughly with `n^(1/d)` or worse, and each hop costs M distance computations. Getting from one side of the dataset to the other takes a very long walk.

You could fix this by adding random long-range edges (which is how classic small-world graphs work), but then greedy search wastes time evaluating long edges when it is already close to the target.

**The hierarchical solution: separate the scales.**

| Layer | Node count | Edge length | Role |
|---|---|---|---|
| Top (e.g. 4) | ~a handful | Very long | Cross the dataset in 1–2 hops |
| Middle (2–3) | Small fraction | Medium | Narrow to the right region |
| Layer 1 | ~6% of nodes | Shorter | Approach the neighbourhood |
| **Layer 0** | **100% of nodes** | Short | Fine-grained accurate search |

Because each layer holds an exponentially smaller sample, each layer's graph is a "zoomed-out" version of the space with proportionally longer edges. A greedy walk at layer 3 covers in one hop what would take hundreds of hops at layer 0.

**The result:** search cost becomes **O(log n)** in the number of hops. The hierarchy provides the long-range structure without polluting the fine-grained layer with useless long edges — you use long edges only while you are far away, and short edges only when you are close.

**The skip-list analogy, made precise.** A skip list over a sorted list adds express lanes: the top lane skips many elements, lower lanes skip fewer, the bottom lane has everything. Level assignment is probabilistic and exponentially decaying. Search descends from the top, moving as far as possible at each level. HNSW is exactly this, generalised from a 1-D ordering to a proximity graph in a metric space — same probabilistic level assignment, same coarse-to-fine descent, same O(log n).

**Cost of the hierarchy:** extra edges (small — upper layers hold few nodes) and extra implementation complexity. The empirical benefit is large and consistent, which is why HNSW replaced flat NSW.

### Interview Follow-ups

- What would happen with only layer 0? (You get NSW — still functional, but requires many more hops and a larger `efSearch` for the same recall, especially on large datasets.)
- How many layers does a typical index have? (`log_M(n)` roughly — for 1M vectors with M=16, about 5 layers. It emerges from the random level assignment rather than being configured.)

---

## Q11: What are `M`, `efConstruction`, and `efSearch`, and how do they affect recall, latency, memory, and build time?

### Answer

**`M` — maximum number of bidirectional edges per node per layer** (layer 0 allows `M_max0`, typically `2M`).

| Increasing `M` | Effect |
|---|---|
| Recall | **↑** — better connectivity, more escape routes from local minima |
| Query latency | ↑ slightly — more neighbours to evaluate per hop, but fewer hops needed; often close to neutral |
| Memory | **↑ linearly** — this is the only parameter that permanently changes index size |
| Build time | ↑ — more edges to select and prune |

Typical values: **8–64**. Use 12–16 for low dimensions or when memory-constrained; 32–48 for high-dimensional data or when you need high recall. There are diminishing returns past ~48.

**`efConstruction` — the size of the candidate list during *insertion*.**

| Increasing `efConstruction` | Effect |
|---|---|
| Recall | **↑** — better neighbour choices produce a higher-quality graph, raising the recall *ceiling* |
| Query latency | **No direct effect** — it is not used at query time |
| Memory | **No effect** — edge counts are capped by `M`, not by `efConstruction` |
| Build time | **↑ substantially** — this is the dominant build-cost parameter |

Typical values: **100–500**. This is the one to be generous with: you pay once at build time, and it raises the quality ceiling of an index you cannot cheaply rebuild. A common mistake is leaving it at a low default and then being unable to reach the recall you need no matter how high you push `efSearch`.

**`efSearch` — the size of the candidate list during *query*.**

| Increasing `efSearch` | Effect |
|---|---|
| Recall | **↑** — the primary runtime recall dial |
| Query latency | **↑ roughly linearly** — the primary latency cost |
| Memory | Negligible (a transient per-query heap) |
| Build time | No effect |

Typical values: **50–500**, and it must be ≥ `k`. Tunable **per query**, with no rebuild — so you can offer a fast tier and an accurate tier from one index, or raise it dynamically under a restrictive filter.

**Summary table:**

| Parameter | When used | Recall | Latency | Memory | Build time | Changeable later |
|---|---|---|---|---|---|---|
| `M` | Build | ↑↑ | ~neutral | **↑↑** | ↑ | **No** |
| `efConstruction` | Build | ↑↑ (ceiling) | — | — | **↑↑** | **No** |
| `efSearch` | Query | **↑↑** | **↑↑** | — | — | **Yes** |

**Practical tuning procedure:**
1. Build a ground-truth set: exact top-k for ~1,000 real queries via brute force.
2. Set `M = 32`, `efConstruction = 200` as a starting point (generous, since these are locked in).
3. Sweep `efSearch` over {16, 32, 64, 128, 256, 512} and plot recall against p95 latency.
4. Pick the smallest `efSearch` that meets your recall target.
5. Only if you cannot reach the target at acceptable latency, rebuild with a higher `M` and/or `efConstruction`.
6. Re-tune whenever the data distribution or its size changes materially.

### Interview Follow-ups

- Your recall is 0.80 and raising `efSearch` to 1000 does not help. What is wrong? (The graph quality is the ceiling — `efConstruction` and/or `M` are too low. You must rebuild. This is exactly why you should not skimp on build parameters.)
- Which parameter would you change to cut memory by 30%? (None of them effectively — `M` is only a small share of the footprint. Quantise the vectors instead.)

---

## Q12: Explain IVF (Inverted File Index).

### Answer

**1. Purpose.** Reduce the search space by partitioning vectors into clusters and searching only the clusters nearest the query.

**2. Core idea.** Cluster the vectors with k-means into `nlist` cells, each with a centroid. Every vector is assigned to its nearest centroid's list ("inverted list"). At query time, compare the query only against the `nlist` centroids, pick the closest `nprobe` cells, and exhaustively search only the vectors in those cells.

The name comes from the inverted index in text retrieval: instead of term → document list, it is centroid → vector list.

**3. Step-by-step operation.**

*Training (required, unlike HNSW):*
1. Sample vectors from the dataset (typically 30–256 per centroid).
2. Run k-means to find `nlist` centroids.

*Indexing:*
3. For each vector, compute the nearest centroid and append it to that centroid's list.

*Search:*
4. Compute distances from the query to all `nlist` centroids.
5. Select the `nprobe` closest cells.
6. Exhaustively compute distances to every vector in those cells.
7. Merge and return the top k.

**4. Important parameters.**
- **`nlist`** — number of clusters. Rule of thumb: `4·√n` to `16·√n` (e.g. ~4,000–16,000 for 1M vectors). More cells means smaller cells (faster per-cell scan) but more centroid comparisons and a higher chance of missing the right cell.
- **`nprobe`** — cells searched per query. **The runtime recall/latency dial**, exactly analogous to `efSearch`. 1 is fast and inaccurate; `nlist` degenerates to brute force.
- **Training sample size** — too small and the centroids poorly represent the data.

**5. Advantages.**
- **Very low memory overhead** — just `nlist` centroids plus the list assignments. No graph. Roughly 1–2% overhead versus HNSW's ~10–30%.
- Fast to build (one k-means pass plus one assignment pass) — much faster than HNSW for large datasets.
- `nprobe` is tunable at query time.
- Combines naturally with Product Quantization (→ IVF-PQ).
- Works well on GPUs (regular, batched memory access patterns).
- Filtering can be applied per-cell.

**6. Limitations.**
- **Requires training** on a representative sample — a real operational burden. New data that drifts away from the trained centroids degrades quality, so you must periodically retrain and re-index.
- **The boundary problem:** a query near the edge of a cell may have its true nearest neighbours in an adjacent cell that was not probed. This is the main source of recall loss, and it worsens in high dimensions where cells have many neighbours.
- Generally **lower recall than HNSW at the same latency** on real-world data.
- Sensitive to unbalanced clusters — if data is unevenly distributed, some cells become huge and slow to scan.
- Adding many vectors post-training skews the assignment quality.

**7. Typical use cases.** Very large datasets (100M+) where HNSW's memory is prohibitive, GPU-accelerated search, as the coarse quantiser in **IVF-PQ** (its most common role by far), and any setting where fast index builds matter more than the last few points of recall.

### Example

```python
import faiss

d, nlist = 768, 4096
quantiser = faiss.IndexFlatIP(d)
index = faiss.IndexIVFFlat(quantiser, d, nlist, faiss.METRIC_INNER_PRODUCT)

index.train(training_vectors)     # k-means — REQUIRED before adding
index.add(all_vectors)

index.nprobe = 32                 # runtime recall/latency dial
D, I = index.search(queries, k=10)
```

### Interview Follow-ups

- Why is `nprobe > 1` essential? (The boundary problem — with `nprobe=1` you miss any neighbour that fell into an adjacent cell, and in high dimensions that is common.)
- How does IVF handle new vectors? (Assign to the nearest existing centroid. It works, but centroids drift out of date as the distribution shifts, requiring periodic retraining — HNSW has no equivalent problem.)

---

## Q13: Explain Product Quantization (PQ) and IVF-PQ.

### Answer

**1. Purpose.** Compress vectors by 4–64× so that huge datasets fit in RAM, at the cost of approximating distances.

**2. Core idea.** Split each vector into `m` contiguous sub-vectors. Independently learn a small codebook (via k-means, typically 256 centroids = 1 byte) for each sub-space. Replace each sub-vector with the **id of its nearest sub-centroid**. A vector becomes `m` bytes instead of `d × 4` bytes.

Why "product": the effective codebook is the *Cartesian product* of the sub-codebooks. With m=8 sub-spaces and 256 centroids each, you can represent 256⁸ ≈ 1.8 × 10¹⁹ distinct vectors while storing only 8 × 256 sub-centroids. That combinatorial explosion is the whole trick.

**3. Step-by-step operation.**

*Training:*
1. Split the d-dimensional space into `m` sub-spaces of d/m dimensions each.
2. For each sub-space, run k-means with `2^nbits` (usually 256) centroids on the training data.
3. Store the `m` codebooks.

*Encoding:*
4. For each vector, split it into `m` sub-vectors.
5. For each sub-vector, find the nearest sub-centroid and store its 1-byte id.
6. The vector is now `m` bytes. **The original is discarded.**

*Search (Asymmetric Distance Computation — the clever part):*
7. Split the query into `m` sub-vectors. Keep the query at **full precision**.
8. Precompute a lookup table: for each sub-space, the distance from the query's sub-vector to all 256 sub-centroids. That is `m × 256` distances, computed once per query.
9. For each database vector, sum `m` table lookups using its byte codes. **No distance arithmetic per vector — just m memory lookups and adds.**

This is called *asymmetric* because the query is not quantised, only the database. Keeping the query exact meaningfully improves accuracy over quantising both.

**4. Compression example — 768 dims, m=96, 8 bits:**

```text
Original:   768 × 4 bytes = 3,072 bytes
PQ-encoded: 96 × 1 byte   =    96 bytes        -> 32× compression
```

For 100M vectors: 307 GB → 9.6 GB. That is the difference between a large cluster and a single machine.

**5. Important parameters.**
- **`m`** (sub-quantisers) — must divide `d`. Higher m means finer approximation, better recall, more bytes. Common: d/4 to d/16.
- **`nbits`** — bits per code, almost always 8 (256 centroids) for fast byte-aligned lookups.
- Training sample size — needs enough data per sub-space codebook.

**6. IVF-PQ — combining them.** IVF narrows *which* vectors to examine; PQ makes *each* comparison cheap and each vector small. Together:

```text
IVF-PQ search:
1. Compare query to nlist centroids           -> pick nprobe cells
2. Within those cells, score PQ codes via ADC -> cheap approximate distances
3. (Optional) Re-rank the top candidates with full-precision vectors
```

A refinement most implementations use: PQ encodes the **residual** (vector − its cell centroid) rather than the raw vector. Residuals have much smaller variance, so the same codebook budget achieves noticeably better accuracy.

**7. Advantages.** Massive memory reduction (the only practical way to hold billions of vectors in RAM); very fast distance computation via table lookups; composes with IVF and (as HNSW-PQ) with graph indexes.

**8. Limitations.**
- **Lossy.** Recall drops — typically to 0.7–0.9 depending on `m`. The original vectors are gone unless you store them separately.
- Requires training, and it is distribution-sensitive.
- Assumes sub-spaces are roughly independent; correlated dimensions quantise poorly. **OPQ** (Optimized PQ) fixes this by learning a rotation matrix first, which decorrelates the dimensions before splitting — a cheap and worthwhile addition.
- Reconstruction is approximate, so you cannot recover the original vector.

**9. Typical use cases.** Billion-scale search (image and web-scale retrieval), memory-constrained deployment, and any first-stage retrieval where a **rescoring pass** with exact vectors follows.

**The standard production pattern:** IVF-PQ (or binary quantisation) for a cheap, wide first pass to get ~500 candidates, then rescore those with full-precision vectors kept on disk or in a separate store. This recovers most of the lost recall while keeping the memory-resident index tiny — the best of both.

### Example

```python
import faiss

d, nlist, m, nbits = 768, 4096, 96, 8

quantiser = faiss.IndexFlatL2(d)
index = faiss.IndexIVFPQ(quantiser, d, nlist, m, nbits)

index.train(training_vectors)      # trains both k-means centroids and PQ codebooks
index.add(all_vectors)             # 96 bytes per vector instead of 3072

index.nprobe = 32
D, I = index.search(queries, k=100)   # over-fetch, then rescore with exact vectors
```

### Interview Follow-ups

- Why keep the query unquantised? (Asymmetric distance computation is strictly more accurate — quantising the query would add a second, avoidable error term.)
- What does OPQ add? (A learned rotation that decorrelates dimensions before splitting, so each sub-space carries comparable, independent variance. Usually a free recall improvement.)
- How does scalar quantisation compare? (Per-dimension int8 instead of sub-vector codebooks — only 4× compression but much less lossy, no codebook training, and simpler. Often the better first choice.)

---

## Q14: HNSW vs brute-force search — compare.

### Answer

| | Brute force (Flat) | HNSW |
|---|---|---|
| Recall | **1.00 guaranteed** | 0.90–0.99, tunable |
| Query complexity | O(n · d) | O(log n · d) approximately |
| Latency at 1M × 768 | ~50–200 ms | ~1–3 ms |
| Latency at 10M × 768 | ~500 ms–2 s | ~2–5 ms |
| Memory | Vectors only | Vectors + graph (~10–30% more) |
| Build time | Zero | O(n log n), minutes to hours |
| Insert cost | O(1) append | O(log n) graph insertion |
| Delete | Trivial | Tombstone + eventual rebuild |
| Filtering | Exact and free | **Problematic** (see Q15) |
| Parameters to tune | None | `M`, `efConstruction`, `efSearch` |
| Determinism | Fully deterministic | Depends on insertion order |

**When brute force wins:**
- Under ~100k vectors — the latency is already fine and you avoid all the complexity.
- **Highly selective filters.** If `WHERE tenant_id = X` leaves 200 vectors, scanning them exactly is faster *and* perfectly accurate. HNSW would struggle here.
- Small per-tenant collections (thousands of tenants, each with hundreds of vectors).
- You need guaranteed-complete results (compliance, dedup thresholds).
- Establishing ground truth to measure your ANN recall.
- Rescoring a small candidate set.

**When HNSW wins:** everything above roughly a million vectors with an interactive latency budget. The crossover is typically 100k–1M depending on dimensions, hardware, and your p95 target.

**The insight worth stating.** The comparison is not "approximate vs exact" as an abstract trade-off — it is a **scale-dependent** decision. Both should be in your toolkit, and a good system may use both: HNSW for the global corpus, brute force inside a heavily filtered subset. Several vector databases do exactly this automatically, choosing the strategy based on the filter's estimated selectivity.

**Do not skip the measurement.** Teams routinely add HNSW to a 50k-vector collection, inherit a recall problem and three tuning parameters, and gain nothing measurable. Start flat; index when you have profiled a real problem.

### Interview Follow-ups

- How would you decide the crossover point for your system? (Benchmark both on your actual data and dimensions against your p95 latency budget. It is a measurement, not a rule.)
- Can you get exact results from HNSW? (Only by setting `efSearch = n`, which is brute force with extra overhead. If you need exact, use flat.)

---

## Q15: HNSW vs IVF — compare.

### Answer

| | HNSW | IVF |
|---|---|---|
| Structure | Multi-layer proximity graph | k-means cells + inverted lists |
| Training required | **No** | **Yes** (k-means on a sample) |
| Recall at equal latency | **Higher** | Lower |
| Memory overhead | High (~10–30%: the graph) | **Very low** (~1–2%: centroids) |
| Build time | Slower, O(n log n) | **Faster**, one k-means + one assignment pass |
| Incremental inserts | **Native and clean** | Works, but centroids drift → periodic retraining |
| Deletes | Tombstones, needs compaction | Remove from a list — cleaner |
| Runtime recall dial | `efSearch` | `nprobe` |
| Scales to billions | Memory-limited | **Yes**, especially with PQ |
| GPU friendliness | Poor (irregular graph traversal) | **Good** (regular batched scans) |
| Filtering behaviour | Degrades badly under selective filters | Per-cell filtering is more tractable |
| Distribution shift | Robust | Sensitive — centroids go stale |
| Typical default in | Qdrant, Weaviate, Elasticsearch, pgvector, Redis | FAISS large-scale, GPU pipelines, as IVF-PQ's coarse quantiser |

**The essential trade-off: memory versus training.**

HNSW buys higher recall at a given latency by spending memory on a graph, and it needs no training — you can insert from an empty index and it just works. That operational simplicity is why it became the default for most vector databases.

IVF spends almost no extra memory and builds fast, but requires a representative training pass and gives up recall to the cell-boundary problem. Its enduring importance is as the **coarse quantiser in IVF-PQ**, where the goal is billion-scale on limited RAM.

**Why HNSW recall is higher.** IVF makes one hard, irreversible decision: which cells to probe. If the true nearest neighbour sits in an unprobed cell, it is lost, and no amount of within-cell exhaustiveness recovers it. HNSW's graph traversal is adaptive — it follows the actual local structure of the data and can route around a bad initial direction. There are no artificial hard boundaries.

**Choosing between them:**

```text
< 1M vectors, need simplicity          -> HNSW (or flat)
1M-100M, RAM available, recall matters -> HNSW
100M+, RAM constrained                 -> IVF-PQ
GPU-based pipeline                     -> IVF / IVF-PQ
Heavy filtered search                  -> IVF, or HNSW with careful filter strategy
Frequent inserts, no retraining window -> HNSW
Data distribution shifts often         -> HNSW
```

**A note on filtering, since it comes up constantly.** Under a selective filter, HNSW has a real problem: the graph's edges were built over *all* vectors, so traversal keeps landing on filtered-out nodes. You either accept degraded recall (skip invalid nodes and lose connectivity) or over-fetch massively. Common mitigations are filterable HNSW variants that maintain connectivity among matching nodes, or falling back to a filtered brute-force scan when selectivity is high. IVF sidesteps some of this because you can evaluate the filter per cell. This is one of the strongest practical arguments for IVF in a filtering-heavy workload.

### Interview Follow-ups

- Why is IVF better on GPUs? (Cell scans are dense, contiguous, batched matrix operations. Graph traversal is pointer-chasing with irregular access — bad for GPUs.)
- Could you combine them? (Yes — using HNSW as IVF's coarse quantiser to find the nearest centroids quickly is a real technique when `nlist` is very large.)

---

## Q16: HNSW vs IVF-PQ — compare.

### Answer

| | HNSW (uncompressed) | IVF-PQ |
|---|---|---|
| Vector storage | Full precision (d × 4 bytes) | Compressed to m bytes |
| Memory for 100M × 768 | ~330 GB | **~10 GB** (m=96) |
| Recall | 0.95–0.99 | 0.70–0.90 (recoverable with rescoring) |
| Latency | ~2–5 ms | ~1–3 ms |
| Distance computation | Exact on stored vectors | Approximate via lookup tables |
| Training | None | k-means + PQ codebooks |
| Original vectors recoverable | Yes | **No** (lossy) |
| Best at | Highest quality within a RAM budget | Maximum scale on limited RAM |

**These are not competing answers to the same question.** HNSW optimises *search structure*; PQ optimises *vector storage*. They address different bottlenecks and can be combined — HNSW-PQ and HNSW-SQ exist in FAISS, Qdrant, and Weaviate.

**The decision is set by your memory budget:**

```text
Do the full-precision vectors fit in RAM (with ~30% graph overhead)?
  YES -> HNSW. Highest recall, no training, simple operations.
  NO  -> Compress. Then choose:
         - Scalar quantisation (int8): 4x smaller, tiny recall loss, often enough
         - Binary quantisation: 32x smaller, needs rescoring, very fast
         - IVF-PQ: 16-64x smaller, needs training, proven at billion scale
```

**Worked example — 100M vectors at 768 dims:**

| Configuration | Memory | Recall@10 |
|---|---|---|
| Flat (exact) | 307 GB | 1.00 |
| HNSW, M=32 | ~360 GB | 0.98 |
| HNSW + int8 SQ | ~95 GB | 0.97 |
| IVF-PQ, m=96 | ~11 GB | 0.85 |
| IVF-PQ + rescore top 500 | ~11 GB + disk | **~0.95** |

**That last row is the important one.** The standard production pattern is: search a compressed in-memory index broadly (over-fetch 10–20× your k), then **rescore** those candidates against full-precision vectors held on disk or in an object store. You recover most of the lost recall while keeping the hot index tiny. Always propose this rather than accepting raw PQ recall — it is the answer that shows production experience.

**Binary quantisation deserves mention** as the current favourite for this pattern: 32× compression, Hamming distance computed with XOR and popcount (extremely fast), then rescore the top candidates with int8 or float32. On strong modern embedding models it retains ~95%+ of full recall after rescoring.

### Interview Follow-ups

- What is DiskANN, and how does it differ? (A graph index designed for SSD residency — keeps a compressed representation in RAM for routing and reads full vectors from disk for rescoring. Purpose-built for the "does not fit in RAM" case, and a strong alternative to IVF-PQ.)
- Why over-fetch before rescoring? (The approximate ranking is noisy, so the true top-10 is scattered through the approximate top-200 or top-500. Over-fetching gives the exact rescorer the material to sort correctly.)

---

## Q17: What is BM25, and how does sparse retrieval work?

### Answer

**BM25 (Best Matching 25)** is the standard lexical ranking function — a refined TF-IDF that has been the strongest keyword baseline for decades and is still half of every good hybrid retrieval system.

```text
score(q, d) = Σ_{t in q}  IDF(t) ·  ( tf(t,d) · (k₁ + 1) )
                                    ─────────────────────────────────────────
                                    tf(t,d) + k₁ · (1 − b + b · |d| / avgdl)
```

**The two improvements over TF-IDF, and why each matters:**

1. **Term-frequency saturation (`k₁`).** In TF-IDF, a term appearing 100 times scores 100× a single occurrence. That is wrong — the 50th mention of "python" tells you almost nothing new beyond the 5th. BM25's ratio form asymptotically approaches `k₁ + 1`, so score gains flatten out. `k₁` (typically 1.2–2.0) controls how quickly saturation kicks in; `k₁ = 0` ignores frequency entirely.

2. **Document-length normalisation (`b`).** Long documents contain more terms by chance, so they would win unfairly. The `|d| / avgdl` term penalises documents longer than average. `b` (typically 0.75) controls the strength: `b = 0` disables normalisation, `b = 1` applies it fully.

**IDF** is as in TF-IDF — rare terms carry more weight — usually with a smoothed variant to avoid negative values for terms in more than half the corpus.

**How sparse retrieval executes.** Terms are indexed in an **inverted index**: term → posting list of (doc_id, term_frequency). A query touches only the posting lists of its own terms, and optimisations like WAND/block-max skip documents that cannot enter the top k. This is why lexical search is so fast — it never touches documents that share no terms with the query.

**Advantages.** No training or model needed; excellent on exact and rare terms (identifiers, error codes, part numbers, names); interpretable ("matched on these terms with these weights"); extremely fast and mature; robust out of domain.

**Limitations.** No semantics — the **vocabulary mismatch problem**. A query for "cancel my plan" misses a document titled "terminate your subscription." No handling of synonyms, paraphrase, or cross-lingual matching.

**Learned sparse (SPLADE) as the middle ground.** A transformer produces a *sparse* vector over the vocabulary in which the model **expands** the document with related terms it did not literally contain and learns per-term weights. You keep the inverted index, the exact-match precision, and the interpretability, while gaining semantic expansion. It is the strongest counter to "dense embeddings replaced sparse retrieval."

**Tuning note:** `k₁` and `b` are worth tuning per corpus. Short-document collections (titles, product names) want low `b`; long heterogeneous documents want `b` near 0.75.

### Interview Follow-ups

- Why does BM25 still beat dense retrieval on some benchmarks? (Out-of-domain robustness — BEIR showed BM25 competitive with or better than dense retrievers on domains they were not trained on. Dense retrieval needs in-domain training to shine.)
- Where does BM25F fit? (A fielded variant that weights matches by field — title matches count more than body matches. Very useful in practice.)

---

## Q18: How does metadata filtering work with vector search, and why is it hard?

### Answer

**What it is.** Combining a similarity search with hard predicates: `WHERE tenant_id = 'acme' AND date > '2024-01-01' AND department IN ('finance','legal')`.

**Why you need it.** Semantic similarity has no concept of recency, permissions, or category membership. Filters handle:
- **Access control** — the single most important use, and a security requirement, not an optimisation
- Multi-tenancy isolation
- Recency and validity windows
- Category, language, source, or document-type scoping
- Deleted/archived exclusion

**Three strategies, and their failure modes:**

| Strategy | How | Problem |
|---|---|---|
| **Post-filter** | ANN search for top k, then discard non-matching | With a selective filter you may return **zero** results. Retrieving 100 and filtering to 2 is a silent recall disaster. |
| **Pre-filter** | Evaluate the predicate first, then search only the matching subset | Correct and exact, but an ANN graph built over *all* vectors cannot be traversed over a subset. Usually means brute force over the survivors. |
| **Filtered / in-search filtering** | Evaluate the predicate *during* traversal, skipping invalid nodes | The best of both, but graph connectivity degrades as more nodes are excluded — the search can get stranded. |

**Why it is genuinely hard for HNSW.** The graph's edges encode proximity over the *whole* dataset. Remove 99% of nodes and the remaining 1% may not be connected to each other at all — the paths between them ran through excluded nodes. Greedy traversal then either terminates early with poor results or has to explore a large fraction of the graph.

**How production systems handle it:**

1. **Selectivity-based strategy choice.** Estimate how many vectors pass the filter. High selectivity (few survivors) → brute-force scan the survivors, which is fast *and* exact. Low selectivity → filtered graph traversal. Qdrant, Weaviate, and Milvus all do a version of this, using a cardinality estimate to decide.
2. **Partition by the high-cardinality filter.** For multi-tenancy, give each tenant its own collection/namespace/shard. Then tenant filtering is free — you search the right index — and each index is small enough that flat search may suffice. **This is the right design for multi-tenant systems** and is a strong interview answer.
3. **Filterable graph construction.** Build extra edges that preserve connectivity within common filter categories.
4. **Increase `efSearch` under a filter** to compensate for lost connectivity.
5. **Over-fetch and post-filter** only when the filter is known to be weakly selective.

**Security requirement — state this explicitly in an interview.** Access-control filters must be:
- Derived from the **authenticated session**, never from user input or model output
- **Applied in the query**, not after the LLM sees the results
- **Mandatory** — enforce them in a wrapper the caller cannot bypass, so nobody can forget the filter

A prompt instruction like "only use documents this user can access" is not access control. The retrieved chunks are already in the model's context by then.

### Example

```python
# Enforce tenant + permission filters in code, derived from the session.
def retrieve(query: str, session: Session, k: int = 20):
    filters = {
        "must": [
            {"key": "tenant_id", "match": {"value": session.tenant_id}},
            {"key": "acl_group", "match": {"any": session.groups}},
            {"key": "status", "match": {"value": "active"}},
        ]
    }
    return client.search(
        collection_name="docs",
        query_vector=embed(query),
        query_filter=filters,      # mandatory, not optional
        limit=k,
        search_params={"hnsw_ef": 256},   # raise ef to offset filter-induced recall loss
    )
```

### Interview Follow-ups

- A tenant with 500 documents queries a 50M-vector shared index. What happens and what would you change? (Post-filtering likely returns nothing useful; filtered traversal is inefficient. Partition per tenant — then it is a 500-vector flat search, exact and instant.)
- How do you handle a filter on a high-cardinality field like `user_id` with millions of values? (Partitioning is impractical at that cardinality; use in-search filtering with a payload index on the field, and fall back to brute force when selectivity is high.)

---

## Q19: What is hybrid retrieval, and how do you fuse the results?

### Answer

**What it is.** Running both sparse (BM25) and dense (vector) retrieval and combining their result lists into one ranking.

**Why it works — complementary failure modes:**

```text
Query: "cancel my subscription"
  BM25   -> misses "terminate your plan"        (no shared terms)
  Dense  -> finds it

Query: "ERR_4021 timeout"
  BM25   -> exact hit on the error code
  Dense  -> returns semantically similar but wrong error codes

Query: "invoice INV-2024-88213"
  BM25   -> exact match
  Dense  -> returns other invoices
```

Neither method alone covers both. Hybrid reliably beats either on mixed real-world query distributions, and it is the production default.

**Fusion method 1 — Reciprocal Rank Fusion (RRF).** Combine on *ranks*, ignoring scores:

```text
RRF_score(d) = Σ_retrievers  1 / (k + rank_r(d))          k typically 60
```

**Why RRF is usually the right choice.** BM25 scores are unbounded and corpus-dependent; cosine scores live in [−1, 1]. They are not comparable, and normalising them well is genuinely hard (min-max normalisation is unstable because it depends on the particular result set). RRF sidesteps the problem entirely by using only ordinal information. It needs no tuning, is robust, and is what most vector databases implement natively. The constant `k` dampens the influence of the very top ranks so that a document ranked well by *both* retrievers can beat one ranked first by only one.

**Fusion method 2 — weighted score fusion.**

```text
score(d) = α · normalise(dense_score) + (1 − α) · normalise(sparse_score)
```

More expressive — you can tune α to your query mix — but requires careful, stable normalisation and per-corpus tuning. Use it when you have an eval set to tune α against and you need the extra control.

**Fusion method 3 — rerank everything.** Retrieve candidates from both, take the union, and let a cross-encoder score them all. The reranker becomes the fusion mechanism, and it is the most accurate option since it actually reads the query-document pair. It also makes the fusion weights largely irrelevant — a strong argument for putting your effort into the reranker rather than into tuning α.

**Implementation notes:**
- Run the two retrievals **in parallel** — they are independent, so hybrid should not double your latency.
- Over-fetch from each (e.g. 50 from each for a final 10) so fusion has material to work with.
- Deduplicate by document id before fusing.
- Many vector databases now support sparse and dense vectors in one collection with native hybrid queries — simpler than orchestrating two systems.

**Evaluate it.** Hybrid usually wins, but the gain size depends entirely on your query distribution. If your queries are all natural-language paraphrase questions, dense alone may be close. If they contain identifiers and codes, hybrid is essential. Measure on real queries.

### Example

```python
def reciprocal_rank_fusion(result_lists, k=60, top_n=20):
    """result_lists: list of ranked lists of doc ids, best first."""
    scores = {}
    for results in result_lists:
        for rank, doc_id in enumerate(results, start=1):
            scores[doc_id] = scores.get(doc_id, 0.0) + 1.0 / (k + rank)

    return sorted(scores, key=scores.get, reverse=True)[:top_n]

# Run both retrievers concurrently, then fuse.
dense = vector_search(query, k=50)
sparse = bm25_search(query, k=50)
candidates = reciprocal_rank_fusion([dense, sparse])
final = cross_encoder_rerank(query, candidates, top_n=5)
```

### Interview Follow-ups

- Why is `k=60` in RRF? (An empirical default from the original paper; it is not sensitive, and values between 20 and 100 behave similarly.)
- When would you skip the sparse side? (A small homogeneous corpus of natural-language prose with no identifiers — and even then it rarely hurts.)

---

## Q20: What is reranking, and why does it improve results?

### Answer

**What it is.** A second-stage model that rescores a small candidate set retrieved by the first stage, producing a more accurate final ordering.

**Why the first stage cannot do this job.** A bi-encoder embeds documents **without knowing the query** (see `08-embeddings.md` Q13). Each document vector must be a general-purpose summary serving every possible query — an information bottleneck. A **cross-encoder** reads the query and the document *together*, with full attention between their tokens, so it can check whether *this* query is actually answered by *this* passage, handle negation, and weigh partial matches.

**The two-stage architecture:**

```text
Query
  → Stage 1: hybrid retrieval over 10M chunks   → top 50-100    (~10-20 ms)   maximise RECALL
  → Stage 2: cross-encoder reranks those        → top 3-8       (~30-80 ms)   maximise PRECISION
  → Generation
```

**The objectives are genuinely different**, which is the key insight. Stage 1 asks "is the answer *somewhere* in these 100?" — it may be sloppy about order. Stage 2 asks "which 5 of these 100 are best?" — and the LLM only sees those 5.

**Why reranking gives large, reliable gains:**
1. Query-aware scoring (the fundamental advantage).
2. It fixes the "topically similar but not answering" failure — the most common RAG relevance error.
3. It lets you over-retrieve safely: fetch 100 for recall, then trim to 5 for precision.
4. It **improves generation quality by reducing context**. Fewer, better chunks means less distraction and no lost-in-the-middle degradation (see `06-transformers-llms-generative-ai.md` Q30). Reranking often improves answers *and* cuts token cost simultaneously — a rare win on both axes.
5. It normalises across retrievers, making hybrid fusion weights less critical.

**Reranker options:**

| Type | Examples | Latency (100 docs) | Quality |
|---|---|---|---|
| Cross-encoder (small) | `ms-marco-MiniLM-L6-v2` | ~20–50 ms | Good |
| Cross-encoder (large) | BGE-reranker-large, Jina reranker | ~100–300 ms | Better |
| Hosted API | Cohere Rerank, Voyage rerank | ~100–200 ms + network | Very good, no infra |
| **Late interaction** | ColBERT | ~10 ms | Good; needs a multi-vector index |
| LLM-as-reranker | Prompt a model to rank | ~1 s+ | Strong but slow and expensive |

**Practical guidance:**
- Rerank **50–100** candidates, not 1,000 — latency scales linearly and gains flatten.
- Pass **3–8** chunks to the generator. More is usually worse, not better.
- Batch the pairs in one forward pass.
- Cache rerank scores for repeated (query, doc) pairs.
- Use a small cross-encoder first; upgrade only if measurement justifies it.

**The trade-off:** added latency and cost, plus another model to serve. But reranking is consistently the **highest-ROI single addition** to a naive RAG pipeline — usually a larger gain than switching to a more expensive embedding model.

### Interview Follow-ups

- Why not rerank everything and skip the retriever? (Cross-encoders cannot precompute — scoring 10M documents per query is infeasible. The retriever exists to make the candidate set small.)
- How do you evaluate a reranker? (nDCG@5 and MRR before versus after, on a labelled set. Also measure end-to-end answer quality — the reranker's real job is improving the final answer, not its own metric.)

---

## Q21: How do you evaluate retrieval quality?

### Answer

**You need labelled data first:** (query, relevant_document_ids) pairs. Sources, in descending order of quality: human annotation of real queries; user click/thumbs-up signals; LLM-generated synthetic queries from your chunks (fast to bootstrap, but beware — the query is derived from the chunk, which biases toward lexical overlap and overstates performance).

**Core metrics:**

| Metric | Formula / meaning | What it tells you |
|---|---|---|
| **Recall@k** | (relevant docs in top k) / (all relevant docs) | **The retriever's real job** — did we find the answer at all? |
| Precision@k | (relevant in top k) / k | How much noise is in the results |
| **MRR** | mean of 1/rank_of_first_relevant | How high the first good result appears |
| **nDCG@k** | discounted gain, normalised by the ideal ordering | Best overall ranking metric; handles graded relevance |
| MAP | mean average precision across queries | Ranking quality with multiple relevant docs |
| Hit rate@k | fraction of queries with ≥1 relevant doc in top k | Simple, interpretable coverage measure |

**Which to prioritise, and why.** For a RAG pipeline:
- **Stage 1 (retrieval): Recall@50 or Recall@100.** If the answer is not in the candidate set, nothing downstream can recover it. This is the hard ceiling on your whole system.
- **Stage 2 (reranking): nDCG@5 and MRR.** Order matters now, because only the top few reach the LLM.
- **End to end: answer correctness and faithfulness** (see `10-rag.md`).

**Why nDCG is the best single ranking metric:**

```text
DCG@k  = Σᵏ  relevanceᵢ / log₂(i + 1)
nDCG@k = DCG@k / IDCG@k
```

It handles **graded** relevance (highly relevant vs marginally relevant, not just binary), applies a **positional discount** so a relevant document at rank 1 counts more than at rank 10, and normalises so scores are comparable across queries with different numbers of relevant documents.

**Two different meanings of "recall" — do not conflate them:**
1. **ANN recall** — the overlap between your ANN results and the *exact* nearest neighbours. Measures index quality. Ground truth comes from brute force.
2. **Retrieval recall** — the overlap between your results and *human-labelled relevant* documents. Measures system quality.

You can have ANN recall of 0.99 and retrieval recall of 0.4: the index is finding exactly the right vectors, and the embeddings or chunking are wrong. Diagnosing which one is broken is the whole point of measuring both.

**Practical evaluation discipline:**
- Build a golden set of 100–500 real queries and version it.
- Run it in CI on every change to chunking, embedding model, index parameters, or retrieval logic.
- Report confidence intervals — a 2-point nDCG move on 100 queries is noise.
- Track latency (p50/p95) and cost alongside quality.
- Keep a **failure log**: for each miss, record whether it was a coverage, retrieval, ranking, or generation failure. That distribution tells you where to invest.

### Example

```python
import numpy as np

def ndcg_at_k(retrieved_ids, relevance_map, k=10):
    """relevance_map: {doc_id: graded_relevance}, 0 for irrelevant."""
    gains = [relevance_map.get(d, 0) for d in retrieved_ids[:k]]
    dcg = sum(g / np.log2(i + 2) for i, g in enumerate(gains))

    ideal = sorted(relevance_map.values(), reverse=True)[:k]
    idcg = sum(g / np.log2(i + 2) for i, g in enumerate(ideal))

    return dcg / idcg if idcg > 0 else 0.0


def recall_at_k(retrieved_ids, relevant_ids, k=50):
    hits = len(set(retrieved_ids[:k]) & set(relevant_ids))
    return hits / len(relevant_ids) if relevant_ids else 0.0
```

### Interview Follow-ups

- Your Recall@100 is 0.95 but answers are wrong. Where is the problem? (Reranking or generation — the evidence is being retrieved but not surfaced or used. Check nDCG@5, chunk ordering, and faithfulness.)
- How do you build an eval set with no labelled data? (Generate synthetic queries per chunk with an LLM to bootstrap, then correct them with human review of a sample, and progressively replace them with real user queries as traffic arrives.)

---

## Advanced

---

## Q22: How do you shard and scale a vector search system?

### Answer

**When you need to scale:** the index exceeds one machine's RAM, or QPS exceeds what one machine's CPU can serve, or you need availability guarantees.

**Two independent axes:**

**1. Replication (for QPS and availability).** Copy the full index to N machines behind a load balancer. Each replica answers complete queries independently. Simple, linear read scaling, and it provides failover. Cost: N× memory, and writes must propagate to all replicas.

**2. Sharding (for index size).** Split the vectors across machines. Each shard holds a subset, and a query must:
1. Fan out to **all** shards (each returns its local top k)
2. Merge the k×shards results and take the global top k

Note the important property: unlike a relational database, you cannot route a vector query to one shard, because the nearest neighbours could be anywhere. **Every shard must be queried** — so latency is bounded by the *slowest* shard (tail latency amplification), and total work grows with shard count.

**Sharding strategies:**

| Strategy | How | Pros | Cons |
|---|---|---|---|
| **Random / hash** | Hash the id | Even load, simple | Every query hits every shard |
| **By tenant/namespace** | One shard per tenant or group | Query one shard only; free tenant filtering | Uneven sizes; huge tenants need sub-sharding |
| **By semantic cluster** | Cluster vectors, one shard per cluster | Can probe only nearby shards | Requires training; drift; unbalanced |
| **By time** | Recent data in a hot shard | Recency filters are cheap; old shards can go to disk | Poor for non-temporal queries |

**Tenant-based sharding is usually the best answer when it applies.** It converts a hard problem (filtered search on a huge shared index) into an easy one (small unfiltered search), and it gives you isolation for free.

**Practical scaling checklist, in order:**
1. **Reduce the footprint first.** Quantisation (int8 → 4×, binary → 32×) and smaller/Matryoshka dimensions are far cheaper than adding machines. Do this before distributing.
2. **Scale up before scaling out.** A 1 TB RAM machine holds a very large index; single-node is dramatically simpler operationally.
3. **Replicate for QPS**, shard for size.
4. **Partition by tenant** if your access pattern allows it.
5. **Two-tier: quantised memory index + full-precision rescoring** from disk/object storage.
6. **Cache** — both exact query caching and semantic caching cut real load substantially on skewed query distributions.

**Operational realities to mention:**
- Index builds are expensive; use blue-green re-indexing rather than in-place mutation.
- Deletes accumulate as tombstones and degrade both memory and recall — schedule compaction.
- Warm up a new replica before routing traffic to it (a cold index has terrible p99 while pages fault in).
- Monitor recall in production, not just latency. Recall degrades silently as data grows and distribution shifts, and nothing will alert you unless you measure it against a golden set.

### Interview Follow-ups

- Why does adding shards sometimes increase latency? (Tail amplification — with 10 shards, your p95 is roughly the p99.5 of a single shard. More shards means more chances one is slow.)
- How do you handle a single tenant that is 1000× larger than the rest? (Sub-shard that tenant across machines while keeping small tenants co-located — a hybrid partitioning scheme.)

---

## Q23: How do you handle updates and deletes in a vector index?

### Answer

This is one of the most under-appreciated operational problems in vector search, and a strong differentiator in interviews.

**Why it is hard.** ANN indexes are optimised for *read* performance on a *static* dataset. HNSW's graph edges encode global proximity structure; IVF's centroids encode the data distribution. Mutating the data undermines both.

**Deletes:**

| Approach | How | Problem |
|---|---|---|
| **Tombstone (soft delete)** | Mark the node deleted; keep it in the graph as a routing waypoint; filter it from results | Memory is not reclaimed; accumulated tombstones degrade recall (traversal wastes hops on dead nodes) and slow queries |
| Hard delete from a graph | Remove the node and repair its neighbours' edges | Expensive and can disconnect the graph; most implementations do not do this |
| **Periodic compaction / rebuild** | Rebuild the index without deleted nodes | The real fix; needs a maintenance window or blue-green swap |

The practical answer: **tombstone for immediate correctness, compact on a schedule.** Monitor the deleted-fraction as an operational metric and trigger compaction above a threshold (commonly 10–20%).

**Updates.** A vector update is a delete plus an insert — you cannot mutate a vector in place, because its position in the graph or its cell assignment was determined by its old value. Same tombstone-and-compact consequences.

**Inserts:**
- **HNSW handles inserts natively** — O(log n), no retraining. This is a major operational advantage.
- **IVF inserts degrade over time.** New vectors are assigned to the nearest *existing* centroid. If the distribution shifts, cells become unbalanced and unrepresentative, so recall decays until you retrain and re-index.

**Practical patterns:**

1. **Blue-green re-indexing.** Build a new index alongside the live one, validate it against your golden eval set, then atomically swap an alias. The only safe way to make a structural change (new embedding model, new chunking, new parameters).
2. **Hot/cold two-tier.** Recent writes go to a small, frequently-rebuilt index (or even a flat one); the bulk sits in a large stable index. Query both and merge. This is how you get near-real-time freshness without constant large rebuilds.
3. **Append-only with versioned documents.** Never update in place — write a new version with a `valid_from`/`valid_to` and filter by time. Gives you an audit trail and point-in-time queries.
4. **Batch your writes.** Per-vector inserts are far more expensive than batched ones, both in index maintenance and in embedding API calls.
5. **Store the source text.** You must be able to re-embed and rebuild; a vector store holding only vectors is a trap (see `08-embeddings.md` Q20).

**The delete requirement people forget: GDPR / right-to-erasure.** You must be able to prove a document's content is gone, not merely filtered from results. Tombstones do not satisfy that — the vector (and often the stored text) is still present. Design for verifiable hard deletion via compaction, and know your compaction SLA.

### Interview Follow-ups

- How do you keep an index fresh when documents change constantly? (Hot/cold tiering, with change-data-capture driving incremental re-embedding of only the modified chunks — hash the chunk text to skip unchanged content.)
- What breaks if you never compact? (Memory grows unboundedly, recall drifts down silently, query latency rises, and deletion compliance fails.)

---

## Q24: What is quantisation in the context of vector search, and what are the options?

### Answer

**Purpose.** Reduce the bytes per vector so more vectors fit in RAM and distance computations get faster (memory bandwidth is the bottleneck for both index traversal and scanning).

**The options, from least to most aggressive:**

| Method | Bytes for 768 dims | Compression | Recall impact | Training needed |
|---|---|---|---|---|
| float32 | 3,072 | 1× | none | no |
| float16 / bfloat16 | 1,536 | 2× | negligible | no |
| **Scalar quantisation (int8)** | 768 | **4×** | very small (~1%) | minimal (min/max or percentiles per dimension) |
| **Product Quantization** | 96 (m=96) | **32×** | moderate (5–15%) | yes (codebooks) |
| **Binary quantisation** | 96 | **32×** | large raw, small after rescoring | no |
| Binary + 2-bit residual | ~200 | ~15× | small | some |

**Scalar quantisation (int8).** Map each dimension linearly to an 8-bit integer using per-dimension min/max (or robust percentiles to resist outliers). 4× smaller, ~1% recall loss, no codebooks, trivially reversible for rescoring. **This is the default recommendation** — almost free quality-wise, and the easiest win available.

**Product Quantization.** Sub-vector codebooks (see Q13). Extreme compression, but lossy, requires training, and distribution-sensitive.

**Binary quantisation.** Keep only the sign of each dimension: `bit = 1 if value > 0 else 0`. Distance becomes **Hamming distance**, computed with XOR and popcount — extraordinarily fast, and hardware-accelerated.

Why does it work at all? In high dimensions, the *sign pattern* carries most of the directional information for a well-trained embedding model, and the model's dimensions are roughly zero-centred. Raw recall drops noticeably, but with a **rescoring pass** (retrieve 10–20× your k by Hamming distance, then rescore those candidates with int8 or float32) you recover ~95%+ of full recall at 32× less memory in the hot path.

**The standard production stack:**

```text
Hot in RAM:   binary or int8 vectors + HNSW graph    -> fast, wide first pass
On disk/SSD:  full-precision vectors                 -> rescore top ~500 candidates
Result:       ~full recall at a fraction of the RAM
```

**How to choose:**

```text
Fits in RAM at float32?           -> do nothing
Need 2-4x?                        -> int8 scalar quantisation (easy, safe)
Need 8-32x?                       -> binary quantisation + rescoring
Billion-scale, RAM-constrained?   -> IVF-PQ (+ OPQ) + rescoring
Cannot fit even compressed?       -> DiskANN-style SSD-resident index
```

**Always validate on your own data.** Quantisation impact depends heavily on the embedding model — models trained with quantisation-awareness or with well-conditioned dimension distributions degrade far less. Measure Recall@k against exact ground truth before and after, and measure it *end to end* on answer quality too.

### Interview Follow-ups

- Why is int8 quantisation nearly lossless while PQ is not? (int8 preserves every dimension independently at reduced precision; PQ replaces groups of dimensions with a single codebook entry, discarding within-cluster variation entirely.)
- Why does rescoring recover so much recall? (The compressed distances are noisy but *approximately* correct, so the true top-k is scattered within the compressed top-500. An exact rescorer over those 500 sorts them correctly.)

---

## Q25: How do you choose a vector database?

### Answer

**Start with the question that eliminates most options: do you need one?**

| Situation | Recommendation |
|---|---|
| < 100k vectors, already using Postgres | **`pgvector`** — or even a numpy array in memory. One system, transactional with your metadata. |
| < 1M vectors, Postgres shop | `pgvector` with HNSW. Genuinely sufficient for most applications. |
| Prototyping locally | Chroma, LanceDB, FAISS |
| Already running Elasticsearch/OpenSearch | Use its vector support — you get mature hybrid search and filtering for free |
| 10M+ vectors, high QPS, advanced filtering | Purpose-built: Qdrant, Weaviate, Milvus, Vespa, Pinecone |
| Want zero operational burden | Managed: Pinecone, Weaviate Cloud, Qdrant Cloud, Turbopuffer |
| Billion-scale | Milvus, Vespa, or a custom FAISS/DiskANN deployment |

**Evaluation criteria, in priority order:**

1. **Filtering capability and performance.** Almost every real application filters by tenant, permission, or date. Filtering support is where vector databases differ most, and it is chronically under-tested in benchmarks. Test it with *your* selectivity patterns.
2. **Hybrid search support.** Native sparse+dense in one query is much simpler than orchestrating two systems.
3. **Recall at your latency budget**, measured on your data — not on a benchmark's.
4. **Memory efficiency / quantisation options.** Directly drives cost.
5. **Write and update patterns.** How does it handle deletes, in-place updates, and index rebuilds without downtime?
6. **Multi-tenancy model.** Namespaces, per-tenant collections, or shared-index-with-filters — the design must match your isolation and cost requirements.
7. **Operational maturity.** Backups, replication, monitoring, resizing, upgrades. This is where teams get hurt after launch.
8. **Cost model.** Managed services often price on stored vectors and dimensions; self-hosting means RAM and engineering time.
9. **Ecosystem fit.** Client libraries, framework integrations, and whether your team can debug it.

**Red flags in a vendor comparison:** benchmarks without filtering; recall reported on random synthetic vectors (which are far harder for ANN than real embeddings, so numbers do not transfer); latency without a QPS figure or percentile; and no mention of memory footprint.

**The interview-strong answer.** Do not lead with a product name. Lead with: "What's the vector count, the QPS, the filtering pattern, the freshness requirement, and what does the team already operate?" Then note that most systems below a few million vectors are best served by `pgvector` or an existing search engine, because the dominant cost of a vector database is operational, not algorithmic. Reach for a specialised system when you have measured a specific need it addresses.

### Interview Follow-ups

- What are `pgvector`'s real limitations? (Historically weaker at very large scale and high write throughput; index builds can be slow and memory-hungry; HNSW build parameters are less tunable. But it has improved substantially and covers a large share of real use cases.)
- Why is filtering the differentiator? (It is the hardest part of vector search to do well — see Q18 — and it is what every production application needs, so implementation quality varies enormously between systems.)

---

## Q26: What is late interaction retrieval (ColBERT), and where does it fit?

### Answer

**The problem it solves.** Bi-encoders compress a whole passage into one vector, losing token-level detail. Cross-encoders keep all the detail but cannot precompute anything. ColBERT sits between them.

**Core idea.** Store **one vector per token** for each document (computed offline), and score a query against a document with a cheap, late aggregation over token similarities.

**MaxSim scoring:**

```text
score(q, d) = Σ_{i in query tokens}  max_{j in doc tokens}  ( q_i · d_j )
```

For each query token, find its best-matching document token, then sum those maxima. Interpretation: "every part of my query should be well-covered by *something* in this document."

**Why "late interaction."** The query and document are encoded **independently** (so document encoding is precomputable, like a bi-encoder), but they interact at the **token level** at scoring time (like a cross-encoder, just with a cheap dot-product-and-max instead of full cross-attention).

**Comparison:**

| | Bi-encoder | ColBERT (late interaction) | Cross-encoder |
|---|---|---|---|
| Vectors per document | 1 | One per token (~100–200) | None stored |
| Precomputable | Yes | **Yes** | **No** |
| Interaction granularity | Whole-text vectors | Token level | Full cross-attention |
| Index size (relative) | 1× | **10–100×** | n/a |
| Query latency | ~1 ms | ~10 ms | ~50 ms per 100 docs |
| Accuracy | Good | **Better** | **Best** |
| Scales to millions of docs | Yes | Yes, with effort | No |

**Advantages.** Substantially better than a bi-encoder, especially on out-of-domain data and on queries with several distinct terms that must all be matched; retains exact-term sensitivity that single-vector dense retrieval blurs; interpretable (you can see which document token matched each query token).

**Limitations.** The index is 10–100× larger — this is the dominant practical obstacle. Mitigations exist and matter: PLAID (aggressive candidate pruning), residual compression of token vectors to 1–2 bits, and pooling similar token vectors. Support in vector databases is improving but is still less mature than for single-vector search.

**Where it fits in a pipeline.** Two viable roles:
1. **As a reranker.** Retrieve with a cheap bi-encoder, then rerank with ColBERT — much faster than a cross-encoder with most of the quality. Very attractive when reranking latency is tight.
2. **As the retriever**, with PLAID-style pruning, when you can afford the index size and want the best first-stage recall.

**The multimodal descendant worth naming: ColPali.** It applies the same late-interaction scoring to **document page images** embedded by a vision-language model. That means retrieving PDF pages with tables, charts, and complex layout *without an OCR pipeline* — a genuinely important capability for enterprise document RAG, and a strong thing to raise in an interview about scanned or layout-heavy corpora.

### Interview Follow-ups

- Why is ColBERT more robust out of domain than a single-vector bi-encoder? (Token-level matching partially recovers the lexical precision of BM25, so it degrades less when the embedding model has not seen your vocabulary.)
- How do you make the index size manageable? (Compress token vectors to 1–2 bits with residual encoding, pool near-duplicate token vectors, and drop stopword-like tokens — combined, these bring the overhead down by an order of magnitude.)

---
