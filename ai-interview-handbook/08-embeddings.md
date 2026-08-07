# Embeddings

The full progression from one-hot encoding to modern embedding models, with the comparisons interviewers use to test whether you understand *why* each step happened.

**Questions:** 22

**Recommended reading order:** Q1 → Q9 traces the historical evolution in sequence. Q10 → Q16 are the comparison questions. Q17+ covers production concerns.

---

## Easy

---

## Q1: What is an embedding, and why do we need them?

### Answer

An **embedding** is a fixed-length vector of real numbers that represents a piece of data (a word, sentence, image, or user) in a continuous space where **geometric proximity encodes semantic similarity**.

**Why they exist.** Models compute with numbers, not text. But *how* you turn text into numbers determines what the model can learn:
- Assigning arbitrary integers (`cat=1, dog=2, car=3`) implies a false ordering and a false claim that `dog` is between `cat` and `car`.
- One-hot vectors avoid false ordering but make every word equidistant from every other — `cat` is exactly as similar to `dog` as to `democracy`.
- Embeddings place `cat` and `dog` near each other and both far from `democracy`, because that structure is *learned* from how the words are used.

**The core property.** Similarity in the space is meaningful:

```text
cos(embed("How do I reset my password?"),
    embed("I forgot my login credentials"))     ≈ 0.85    # no shared words
cos(embed("How do I reset my password?"),
    embed("What is the capital of France?"))    ≈ 0.10
```

Those two questions share almost no vocabulary, so keyword search scores them near zero. Embeddings capture that they mean the same thing. That single property is what makes semantic search, RAG, recommendation, clustering, and deduplication possible.

**Why "dense" matters.** A 768-dimensional dense vector packs meaning into every dimension. A 50,000-dimensional one-hot vector carries one bit of information. Dense vectors generalise (similar inputs get similar vectors) and are cheap to compare (one dot product).

### Interview Follow-ups

- Where do embeddings come from? (Learned as a by-product of some prediction task — predict the neighbouring word, predict whether two sentences are a matching pair, etc.)
- Are embeddings interpretable? (Individual dimensions generally are not; directions and neighbourhoods are. Sparse autoencoders are current research into recovering interpretable features.)

---

## Q2: What is one-hot encoding, and what are its limitations?

### Answer

**What it is.** Each word gets a vector as long as the vocabulary, with a 1 at its own index and 0 everywhere else.

```text
Vocabulary: [cat, dog, car, the]

cat  = [1, 0, 0, 0]
dog  = [0, 1, 0, 0]
car  = [0, 0, 1, 0]
```

**Why it exists.** It is the minimal faithful encoding of a categorical variable: it introduces no false ordering and no false magnitude relationships. For low-cardinality categorical features in tabular ML it remains a perfectly good choice.

**Limitations for text — and each one motivates a later technique:**

| Limitation | Consequence | Later fix |
|---|---|---|
| **No semantics** | `cos(cat, dog) = 0`, identical to `cos(cat, democracy) = 0`. All words are mutually orthogonal and equidistant. | Word2Vec / distributional semantics |
| **Extreme sparsity** | A 50k vocabulary means 50k dimensions with one non-zero | Dense vectors |
| **Dimensionality = vocabulary size** | Memory and compute scale with vocabulary | Fixed-size dense vectors (e.g. 300 or 768 dims) |
| **No generalisation** | Learning something about `cat` teaches nothing about `kitten` | Shared continuous space |
| **Out-of-vocabulary** | A new word has no vector at all | Subword tokenisation |
| **No word order** (when summed into a document vector) | "dog bites man" = "man bites dog" | N-grams, then attention |

**The fundamental problem:** one-hot encoding treats words as **atomic, unrelated symbols**. Every subsequent advance in this file is an attempt to encode relationships between them.

### Interview Follow-ups

- Is one-hot encoding ever the right choice? (Yes — low-cardinality categorical features in tabular models, and as the *input* layer of a neural network, where multiplying a one-hot vector by a weight matrix is exactly an embedding lookup.)
- Why is one-hot × weight matrix the same as a lookup table? (The 1 selects a single row — so an embedding layer is literally a compressed one-hot projection.)

---

## Q3: What is Bag of Words, and how does it differ from one-hot encoding?

### Answer

**One-hot represents a word. Bag of Words represents a document.**

BoW builds a vector as long as the vocabulary where each position holds the **count** of that word in the document. Equivalently, it is the sum of the one-hot vectors of all the document's tokens.

```text
Vocabulary: [the, cat, sat, on, mat, dog]

"the cat sat on the mat"  ->  [2, 1, 1, 1, 1, 0]
"the dog sat"             ->  [1, 0, 1, 0, 0, 1]
```

**Why it exists.** It converts variable-length text into a fixed-length numeric vector that any classifier can consume — the first practical text featurisation, and still a strong baseline for topic-level classification.

**What it adds over one-hot:** document-level representation, term frequency information, and non-zero similarity between documents that share words.

**What it still cannot do:**

| Limitation | Example |
|---|---|
| **No word order** | "dog bites man" and "man bites dog" get identical vectors |
| **Common words dominate** | `the` has the highest count and the least meaning → motivates TF-IDF |
| **No semantics between words** | "car" and "automobile" remain orthogonal dimensions |
| **Sparse and high-dimensional** | Vocabulary-sized vectors, almost all zeros |
| **Length bias** | Longer documents get larger vectors → motivates normalisation |
| **OOV** | Unseen words are dropped entirely |

**Mitigations within the paradigm:** n-grams recover a little local order; L2 normalisation fixes length bias; `min_df`/`max_df` prune the vocabulary; and **TF-IDF** fixes the common-word problem — which is the next step.

### Interview Follow-ups

- How do n-grams partially fix the word-order problem, and what does it cost? (Feature space explosion — see `04-nlp-text-preprocessing.md` Q7.)
- Is BoW dead? (No — it is the basis of sparse retrieval. BM25, the standard lexical retrieval algorithm, is a refined bag-of-words scorer and is still half of every good hybrid search system.)

---

## Q4: What is TF-IDF, and what problem does it solve that BoW does not?

### Answer

**The problem with raw counts.** In BoW, `the` has the highest weight in nearly every document and the least discriminative power. The words that identify what a document is *about* are the ones that are frequent *here* but rare *elsewhere*.

**The solution.** Weight each term by its frequency in the document, discounted by how many documents contain it:

```text
tf(t, d)    = count(t, d) / len(d)              # how much is this doc about t?
idf(t)      = log( N / (1 + df(t)) ) + 1        # how rare is t overall?
tfidf(t, d) = tf(t, d) * idf(t)
```

**Worked intuition.** In a corpus of 10,000 documents:
- `the` appears in all 10,000 → idf ≈ log(1) + 1 = 1 (minimal boost, and near 0 with other IDF variants)
- `photosynthesis` appears in 20 → idf ≈ log(500) + 1 ≈ 7.2

So a single mention of `photosynthesis` outweighs many mentions of `the`. Stop-word removal becomes largely redundant — IDF discovers stop words automatically from the corpus statistics.

**What TF-IDF adds:** term importance weighting, corpus-aware discrimination, and (with L2 normalisation) length-invariant document vectors. It is still an excellent baseline for text classification and the conceptual parent of BM25.

**What it still cannot do — and this is the motivation for everything that follows:**

| Limitation | Example |
|---|---|
| **No semantics** | A query for "automobile" scores 0 against a document containing only "car" — the **vocabulary mismatch problem** |
| **No word order or syntax** | Still a bag |
| **No context** | `bank` has one representation regardless of meaning |
| **Sparse, vocabulary-sized** | Memory and no generalisation to new words |

That first limitation is the decisive one. It is why dense embeddings were needed.

### Interview Follow-ups

- How does BM25 improve on TF-IDF? (Term-frequency **saturation** via `k1` — the 20th occurrence adds almost nothing — and explicit document-length normalisation via `b`. See `09-vector-databases-retrieval.md` Q17.)
- Why is TF-IDF/BM25 still used alongside dense embeddings in 2026? (Exact matching on rare terms, product codes, error codes, names, and code identifiers — precisely where dense retrieval is weakest.)

---

## Intermediate

---

## Q5: What is Word2Vec, and how does it work?

### Answer

**1. Purpose.** Learn dense, low-dimensional word vectors in which semantic and syntactic relationships are encoded geometrically — replacing sparse, semantics-free one-hot representations.

**2. Core idea — the distributional hypothesis.** "You shall know a word by the company it keeps" (Firth). Words appearing in similar contexts have similar meanings. So: train a model to predict context from a word (or vice versa), and the *learned weights* become the embeddings. The prediction task is only a pretext — the by-product is the goal.

**3. Two architectures.**

**Skip-gram:** given the centre word, predict the surrounding context words.
```text
Input: "sat"  ->  predict: "the", "cat", "on", "the"
```

**CBOW (Continuous Bag of Words):** given the context, predict the centre word.
```text
Input: "the", "cat", "on", "the"  ->  predict: "sat"
```

| | Skip-gram | CBOW |
|---|---|---|
| Direction | centre → context | context → centre |
| Training speed | Slower | Faster |
| Rare words | **Better** (each occurrence generates several training pairs) | Worse (averaged away in the context) |
| Small corpora | Better | Needs more data |
| Typical default | Skip-gram with negative sampling (SGNS) | When speed matters |

**4. Step-by-step operation (skip-gram).**
1. Slide a window (size 5–10) over the corpus, generating (centre, context) pairs.
2. Look up the centre word's vector from the input embedding matrix.
3. Score it against candidate output vectors.
4. Compute the loss and update both matrices by gradient descent.
5. After training, keep the input embedding matrix — those rows are the word vectors.

**5. The key optimisation: negative sampling.** A softmax over a 1M-word vocabulary is prohibitively expensive per training step. Negative sampling replaces it with a binary classification: for each true (centre, context) pair, sample k random "negative" words (typically 5–20, drawn from a unigram distribution raised to the 3/4 power to slightly favour rare words) and train the model to distinguish the real pair from the fakes. This turns an O(V) operation into O(k). Hierarchical softmax is the alternative.

**6. Important parameters.** Vector dimension (100–300), window size (small windows → syntactic/functional similarity, large windows → topical similarity), min word count, negative samples k, epochs, and subsampling of very frequent words.

**7. The famous property — vector arithmetic.**
```text
vec("king") - vec("man") + vec("woman")  ≈  vec("queen")
vec("Paris") - vec("France") + vec("Italy")  ≈  vec("Rome")
```
Because relationships are encoded as consistent *directions* in the space, analogies become vector addition. (Worth noting in an interview: the effect is real but often over-sold — results depend on excluding the input words from the nearest-neighbour search, and it does not hold uniformly.)

**8. Advantages.** Dense and compact (300 dims vs 50k); captures semantic similarity; efficient to train on billions of words; embeddings are reusable across tasks — the first practical transfer learning for NLP.

**9. Limitations.**
- **One vector per word — no context.** `bank` gets a single vector averaging all its senses. This is the decisive limitation and the reason contextual embeddings were needed.
- **OOV:** unseen words have no vector (FastText fixes this with character n-grams).
- **Word-level only:** no principled sentence representation (averaging words loses order and is a weak baseline).
- **Ignores word order** within the window.
- **Inherits corpus bias**, and the analogy structure makes it measurable and visible.

**10. Typical use cases.** Historically, the input layer of NLP models. Today, mostly superseded — but still useful for lightweight similarity where transformers are too expensive, and as the intellectual foundation of everything after it.

### Example

```python
from gensim.models import Word2Vec

sentences = [["the", "cat", "sat", "on", "the", "mat"], ...]

model = Word2Vec(
    sentences,
    vector_size=300,
    window=5,
    min_count=5,
    sg=1,             # 1 = skip-gram, 0 = CBOW
    negative=10,      # negative sampling
    workers=8,
)

print(model.wv.most_similar("cat"))
print(model.wv["cat"].shape)      # (300,)
```

### Interview Follow-ups

- Why does the model keep only the input matrix? (Convention — the input embeddings are consistently better representations in practice; summing or concatenating both is sometimes done.)
- Why raise the sampling distribution to the 3/4 power? (Flattens it: rare words get sampled as negatives more than pure unigram frequency would allow, improving their representations.)
- What does FastText add? (Represents a word as the sum of its character n-gram vectors, so it handles OOV words and morphologically rich languages far better.)

---

## Q6: What is GloVe, and how does it differ from Word2Vec?

### Answer

**Purpose.** The same goal as Word2Vec — dense word vectors — but derived from **global corpus co-occurrence statistics** rather than from local sliding windows.

**Core idea.** Word2Vec learns from one window at a time, so it never sees the corpus-level statistics directly. GloVe builds the full word-word co-occurrence matrix X (where X_ij counts how often word j appears in the context of word i) and then factorises it, fitting vectors so that dot products reproduce the log of the co-occurrence counts.

**The insight about ratios.** GloVe's key observation is that meaning lives in co-occurrence *ratios*, not raw counts. Consider `ice` and `steam`:

| Probe word k | P(k \| ice) / P(k \| steam) | Interpretation |
|---|---|---|
| `solid` | large | related to ice, not steam |
| `gas` | small | related to steam, not ice |
| `water` | ≈ 1 | related to both |
| `fashion` | ≈ 1 | related to neither |

So the model is designed so that vector differences encode these ratios — which is what makes analogy arithmetic work by construction rather than by accident.

**Objective:**

```text
J = Σ_ij  f(X_ij) · ( wᵢᵀ w̃ⱼ + bᵢ + b̃ⱼ − log X_ij )²
```

`f(X_ij)` is a weighting function that caps the influence of very frequent pairs and down-weights rare, noisy ones — the practical trick that makes the least-squares fit work.

**Comparison:**

| | Word2Vec | GloVe |
|---|---|---|
| Training signal | Local windows, streamed | Global co-occurrence matrix |
| Objective | Predictive (classification) | Count-based least squares (matrix factorisation) |
| Statistics used | Implicit, local | Explicit, global |
| Memory during training | Low (streaming) | High (must build X) |
| Parallelism | Good | Good; matrix build is a preprocessing step |
| Handles corpus once | Multiple epochs over text | One pass to build X, then iterate on X |
| Quality | Comparable | Comparable — the choice rarely matters much in practice |

**The honest answer to "which is better":** they perform similarly, and the differences reported in papers are within the range of hyperparameter tuning. Both share the same fatal limitation.

**Shared limitation:** both are **static** — one vector per word, no context. That is what contextual embeddings fixed, and it is what makes both of them historical rather than practical choices today.

### Interview Follow-ups

- Is Word2Vec secretly also doing matrix factorisation? (Yes — Levy & Goldberg showed skip-gram with negative sampling implicitly factorises a shifted PMI matrix, which unified the two approaches theoretically.)
- Why did both get replaced? (No context, no sentence-level representation, no handling of polysemy.)

---

## Q7: What are contextual embeddings, and why were they a breakthrough?

### Answer

**What they are.** Token representations that are computed **as a function of the surrounding text**, so the same word receives different vectors in different sentences.

```text
Static (Word2Vec/GloVe):
  "river bank"       -> bank = [0.2, -0.5, 0.8, ...]
  "investment bank"  -> bank = [0.2, -0.5, 0.8, ...]     IDENTICAL

Contextual (BERT/GPT):
  "river bank"       -> bank = [0.7,  0.1, -0.3, ...]
  "investment bank"  -> bank = [-0.4, 0.9,  0.2, ...]    DIFFERENT
```

**Why this was the breakthrough.** Static embeddings collapse every sense of a word into one average vector. `bank` becomes a blur of riverbanks and financial institutions; `run` averages across a dozen senses. Since polysemy is pervasive in natural language, this ceiling was fundamental, not incidental.

Contextual embeddings resolve meaning at use time. The representation of a token is built by attending to the other tokens present, so `bank` next to `river` and `bank` next to `loan` land in different regions of the space. This immediately improved essentially every downstream NLP task.

**How they are produced.** Run the text through a pretrained transformer; take the hidden states at each position (usually from an upper-but-not-final layer). Attention has mixed in the surrounding context, so the output is context-dependent by construction. See `06-transformers-llms-generative-ai.md` Q22.

**The progression:**

| Model | Year | Mechanism |
|---|---|---|
| ELMo | 2018 | Deep bidirectional LSTM; concatenate layer representations |
| GPT | 2018 | Causal transformer — left context only |
| **BERT** | 2018 | Bidirectional transformer with masked LM — the model that made contextual embeddings standard |
| RoBERTa / DeBERTa | 2019–20 | Better training recipes and position handling |
| ModernBERT | 2024 | RoPE, FlashAttention, 8k context — a modernised encoder |

**The crucial caveat for retrieval.** BERT's *token* embeddings are contextual and excellent. But BERT out of the box produces **poor sentence embeddings** — mean-pooling its token vectors, or using the `[CLS]` token, gives similarity scores that are worse than averaged GloVe vectors. The reason: BERT was never trained with an objective that makes pooled representations comparable by cosine similarity. Fixing that required Sentence-BERT and contrastive training — the next question.

### Interview Follow-ups

- Which layer gives the best token representations? (Middle-to-upper layers usually; the final layer is specialised for the pretraining objective.)
- Are contextual embeddings usable for a vector index? (Not token-level — one vector per token is too many. You need a single vector per chunk, i.e. a sentence embedding. ColBERT is the exception that indexes token vectors deliberately.)

---

## Q8: How are modern sentence/document embedding models trained?

### Answer

**The problem to solve.** A vector index needs **one vector per text** such that cosine similarity reflects semantic relatedness. Pretrained encoders do not give this for free — their pooled outputs are not calibrated for cosine comparison.

**The solution: contrastive learning.** Train the model so that matching pairs are pulled together and non-matching pairs pushed apart in the embedding space.

**Sentence-BERT (SBERT)** introduced the practical architecture: a **bi-encoder** (a shared encoder producing one vector per text, pooled — usually mean pooling) trained on pair data.

**The dominant loss — InfoNCE / MultipleNegativesRankingLoss:**

```text
L = -log  exp(sim(q, p⁺)/τ)  /  Σ_j exp(sim(q, p_j)/τ)
```

For each query `q` and its true positive `p⁺`, treat the other examples **in the same batch** as negatives. This is a softmax over similarities, so the model must rank the true pair above all in-batch alternatives.

**Why this works so well, and why batch size matters enormously.** Every additional in-batch example is a free negative. A batch of 1024 gives 1023 negatives per query at no extra forward-pass cost. More negatives means a sharper contrastive signal, which is why embedding training uses very large batches (often with GradCache or gradient checkpointing to fit them).

**Hard negatives are the other half of the story.** Random negatives are easy — a query about password resets versus a document about shipping rates is trivially separated. **Hard negatives** are documents that look relevant but are not (retrieved by BM25 or an earlier model version, then verified as non-relevant). Training on hard negatives is what produces the fine-grained discrimination retrieval actually needs. Mining them well is one of the highest-leverage steps in building a strong embedding model.

**The typical modern training pipeline:**
1. Start from a pretrained encoder (BERT-family) or a decoder LLM adapted for embedding.
2. **Weakly-supervised pretraining** on hundreds of millions of noisy pairs scraped from the web (title–body, question–answer, citation pairs, Reddit post–comment).
3. **Supervised fine-tuning** on curated datasets (MS MARCO, NLI, retrieval benchmarks) with mined hard negatives.
4. Optionally **distil** from a cross-encoder teacher, which is more accurate but too slow to serve.
5. Optionally add **instruction prefixes** and **Matryoshka** training (see Q19, Q20).

**Other losses worth naming:** triplet loss (anchor/positive/negative with a margin — largely superseded by in-batch softmax), cosine similarity regression against labelled scores, and CoSENT.

**Modern examples:** the E5 family, BGE, GTE, Nomic Embed, Voyage, Cohere Embed, OpenAI `text-embedding-3-*`, and Qwen3-Embedding.

### Interview Follow-ups

- Why does temperature τ matter in InfoNCE? (It controls how sharply the model penalises hard negatives; too high and the signal is weak, too low and training destabilises.)
- Why is a bi-encoder needed rather than a cross-encoder for indexing? (See Q13 — a cross-encoder cannot precompute document vectors.)
- How would you fine-tune an embedding model on your own domain? (Mine (query, relevant chunk) pairs from logs or generate them synthetically with an LLM, mine hard negatives with the current model, train with MultipleNegativesRankingLoss, and evaluate with Recall@k and nDCG on a held-out set.)

---

## Q9: What is the complete evolution of text representations?

### Answer

Each step exists to fix a specific, nameable failure of the previous one.

| Stage | Representation | Key advance | Fatal limitation that motivated the next step |
|---|---|---|---|
| **1. One-hot** | Sparse binary, vocab-sized | No false ordering | All words orthogonal — no semantics at all |
| **2. Bag of Words** | Sparse counts, vocab-sized | Document-level vectors | Common words dominate; no word semantics |
| **3. TF-IDF** | Sparse weighted, vocab-sized | Corpus-aware term importance | **Vocabulary mismatch** — "car" ≠ "automobile" |
| **4. Word2Vec / GloVe** | Dense, ~300 dims, static | Learned semantics; similarity and analogies | **One vector per word** — no context, no polysemy |
| **5. FastText** | Dense, static, subword-composed | Handles OOV and morphology | Still static |
| **6. Contextual token embeddings** (ELMo, BERT) | Dense, per-token, context-dependent | Resolves polysemy | Token-level; pooled output not usable for similarity |
| **7. Sentence embeddings** (SBERT + contrastive) | Dense, one vector per text | Cosine-comparable text vectors → **semantic search** | Fixed dimension; single vector loses detail; no instruction awareness |
| **8. Modern embedding models** (E5, BGE, Voyage, OpenAI v3) | Dense, instruction-aware, Matryoshka, multilingual, long-context | Task conditioning, adjustable dimensions, strong multilingual support | Cost/latency; domain gaps; still one vector per chunk |
| **9. Multi-vector / late interaction** (ColBERT) | One vector per token, late scoring | Token-level precision with precomputation | Much larger index footprint |

**The two orthogonal axes of the whole story:**

1. **Sparse → Dense.** From vocabulary-sized indicator vectors to compact learned vectors. Buys generalisation and semantics; loses exact-term precision — which is why hybrid retrieval brings sparse back.
2. **Static → Contextual.** From one vector per word to one per usage. Buys disambiguation.

**The plot twist worth stating in an interview:** stage 3 did not die. BM25 (a refined TF-IDF) is still combined with stage 7/8 embeddings in production hybrid search, because sparse and dense fail in *complementary* ways. Sparse retrieval nails exact rare terms — error codes, part numbers, function names — where dense retrieval blurs them. The evolution is not a replacement chain; it is an accumulation of tools.

### Interview Follow-ups

- At which step did transfer learning arrive in NLP? (Step 4 for word vectors; step 6 for whole-model transfer, which is what made pretraining the standard paradigm.)
- What comes next? (Multi-vector retrieval at scale, learned sparse representations like SPLADE that get both properties in one model, and unified multimodal embedding spaces.)

---

## Q10: One-hot vs Word2Vec — compare.

### Answer

| | One-Hot | Word2Vec |
|---|---|---|
| Density | Sparse (one non-zero) | Dense (all dimensions used) |
| Dimensionality | Vocabulary size (10k–1M+) | Fixed, small (100–300) |
| Semantics | **None** — all words orthogonal | Learned semantic and syntactic structure |
| Similarity between related words | 0, always | High for related words |
| Analogies | Impossible | `king − man + woman ≈ queen` |
| Learned from data | No — assigned | Yes — trained on a corpus |
| Generalisation | None | Similar words get similar vectors |
| OOV handling | Cannot represent | Cannot represent (FastText can) |
| Memory for 50k words | 50k × 50k sparse | 50k × 300 dense = 15M floats |
| Interpretability of dimensions | Perfect (one word per dimension) | None individually |
| Context sensitivity | n/a | **No** — one vector per word |

**The decisive difference.** One-hot places every word at an equal distance from every other, which means a model must learn everything about every word independently. Word2Vec places words in a space structured by usage, so knowledge transfers: if the model learns that `cat` is an animal, `kitten` and `dog` being nearby vectors means that knowledge partially applies to them too.

**The dimensionality point is secondary but practical.** 300 dense dimensions carry vastly more information than 50,000 sparse ones, because every dimension is doing work.

**What they share:** both are static and context-free at the word level. Word2Vec fixed semantics; it did not fix polysemy.

### Interview Follow-ups

- Is a neural network's embedding layer one-hot or dense? (Both — it is a one-hot lookup into a dense matrix, which is exactly the projection from sparse to dense.)

---

## Q11: Word2Vec vs contextual embeddings — compare.

### Answer

| | Word2Vec (static) | Contextual (BERT/GPT) |
|---|---|---|
| Vectors per word | One, fixed | One per occurrence |
| Handles polysemy | **No** | **Yes** |
| Depends on sentence | No | Yes |
| Storage | A lookup table (vocab × dims) | Must run the model — nothing to store |
| Inference cost | A dictionary lookup | A full forward pass |
| Captures syntax / word order | No | Yes |
| Handles OOV | No | Yes (subword tokenisation) |
| Training data required | Millions of words | Billions of tokens |
| Model size | ~100 MB | 100 MB – several GB |
| Typical use in 2026 | Legacy, lightweight similarity | Everything |

**The canonical example.**

```text
Sentence A: "I sat on the river bank."
Sentence B: "I deposited money at the bank."

Word2Vec:   bank_A == bank_B                       (one blurred average of both senses)
BERT:       cos(bank_A, bank_B) ≈ 0.4              (clearly separated)
```

**Why context matters beyond polysemy.** Contextual models also capture:
- **Negation:** "not good" vs "good" — static embeddings average the words and lose the flip.
- **Syntactic role:** "the *dog* bit the man" vs "the man bit the *dog*."
- **Coreference:** what `it` refers to in this sentence.
- **Compositional meaning:** "hot dog" is not `hot` + `dog`.

**The cost trade-off.** Static embeddings are free at inference (a lookup) and can be precomputed for the entire vocabulary. Contextual embeddings require a model forward pass per text. That is a real cost — which is why RAG systems precompute chunk embeddings at ingestion and only embed the query at query time.

**When static embeddings are still defensible:** extremely high-throughput lexical similarity, resource-constrained edge deployment, or as fast features in a larger classical pipeline. Otherwise, contextual wins on every quality axis.

### Interview Follow-ups

- Do contextual models have static embeddings inside them? (Yes — the input embedding layer is static. Context is added by the attention layers on top.)
- Can you distil contextual quality into a static table? (Partially — "static embeddings from contextual models" is a real technique for extreme-speed use cases, trading a few points of quality for a 100×+ speedup.)

---

## Q12: Token embeddings vs sentence embeddings — compare.

### Answer

| | Token embeddings | Sentence embeddings |
|---|---|---|
| Granularity | One vector per token | One vector per text (sentence, paragraph, chunk) |
| Count for a 200-token chunk | 200 vectors | 1 vector |
| Produced by | Any transformer's hidden states | A model trained for text-level similarity, with pooling |
| Comparable by cosine? | Not meaningfully across texts | **Yes — that is the design goal** |
| Training objective | LM / MLM | Contrastive (InfoNCE, etc.) |
| Used for | Input to further layers; token classification (NER, POS); late-interaction retrieval | Vector search, clustering, classification, deduplication |
| Index size | Very large | Manageable |

**How sentence embeddings are produced from token embeddings — the pooling step:**

| Pooling | How | Notes |
|---|---|---|
| **Mean pooling** | Average token vectors, masked to exclude padding | The standard for encoder-based models; robust |
| CLS pooling | Take the `[CLS]` token's vector | Requires that the model was *trained* to make CLS meaningful |
| Max pooling | Element-wise max | Rarely best |
| Last-token pooling | Take the final token's hidden state | Used for decoder-based embedding models, since only the last position has seen everything |
| Weighted mean | Weight later tokens more | Sometimes used with causal models |

**The critical point people get wrong.** Mean-pooling a plain BERT's token embeddings gives *bad* sentence embeddings — measurably worse than averaged GloVe vectors on similarity benchmarks. Pooling is necessary but not sufficient. What makes a sentence embedding good is the **contrastive training objective** that shapes the space so cosine distance means something. Architecture alone does not produce it.

**Where token embeddings win: ColBERT / late interaction.** Instead of collapsing a chunk to one vector, keep all token vectors and score a query against a document with MaxSim (for each query token, take its best match among document tokens, then sum). This preserves fine-grained term-level matching and is markedly more accurate — at the cost of an index that is 10–100× larger. It is the middle ground between a bi-encoder's speed and a cross-encoder's accuracy.

### Interview Follow-ups

- Why does mean pooling need an attention mask? (Otherwise padding tokens are averaged in, corrupting short texts in a batch far more than long ones.)
- Why do decoder-based embedding models use last-token pooling? (With causal attention, only the last position has attended to the entire text; earlier positions have seen only a prefix.)

---

## Q13: Bi-encoder vs cross-encoder — compare.

### Answer

| | Bi-encoder | Cross-encoder |
|---|---|---|
| Input | Query and document encoded **separately** | Query and document encoded **together** as one sequence |
| Output | Two vectors; similarity computed after | A single relevance score |
| Query-document interaction | Only at the final similarity computation ("late") | Full token-level attention throughout ("early") |
| Precompute documents? | **Yes** — embed the whole corpus once, offline | **No** — every pair needs a forward pass at query time |
| Cost for 1M documents per query | 1 query encoding + 1M vector comparisons (ANN makes this ~µs) | 1M forward passes — completely infeasible |
| Accuracy | Good | **Better** — typically several points of nDCG |
| Latency | ~10 ms | ~10–50 ms per **batch of ~50 pairs** |
| Suitable for | First-stage retrieval over a large corpus | Reranking a small candidate set |
| Scalability | Excellent | Poor |

**Why the cross-encoder is more accurate.** In a bi-encoder, the document vector is computed with **no knowledge of the query**. It must be a general-purpose summary that works for every possible query — an information bottleneck. A cross-encoder attends across query and document tokens jointly, so it can directly check whether *this specific* query term is answered by *this specific* passage, catch negations, and weigh partial matches.

**Why the bi-encoder is indispensable anyway.** You cannot precompute anything with a cross-encoder. Scoring a million documents per query is impossible. The bi-encoder's whole value is that the expensive work happens once, offline, at ingestion.

**The resolution: use both, in stages.**

```text
Query
  → Bi-encoder + ANN index over 10M chunks   → top 100 candidates   (~10 ms)
  → Cross-encoder reranks 100 candidates     → top 5                (~50 ms)
  → LLM generates from the top 5
```

This two-stage architecture is the standard production design and the expected answer to "how do you get both speed and accuracy." The bi-encoder maximises **recall** cheaply; the cross-encoder maximises **precision** on a small set.

**The third option:** ColBERT-style late interaction sits between them — precomputable like a bi-encoder, token-level like a cross-encoder, with a much larger index.

### Interview Follow-ups

- Why does reranking help even when the retriever is good? (The retriever optimises recall at k=100; the LLM needs precision at k=5. Those are different objectives, and the reranker performs the second one.)
- Can a cross-encoder be distilled into a bi-encoder? (Yes — a standard and effective technique for training strong embedding models.)

---

## Q14: Static vs contextual embeddings — compare.

### Answer

This is the same axis as Q11 but stated as a general principle rather than a specific model comparison.

| | Static | Contextual |
|---|---|---|
| Definition | The representation of a unit is fixed, independent of surroundings | The representation is computed from the surroundings |
| Examples | Word2Vec, GloVe, FastText, an embedding lookup table | BERT hidden states, GPT hidden states, any transformer layer output |
| Storage model | A precomputed table you look up | A model you run |
| Handles ambiguity | No | Yes |
| Cost at inference | Negligible | A forward pass |
| Determinism | Identical every time | Depends on the whole input |

**Where the distinction gets subtle — and where interviews probe:**

A modern **sentence embedding** model is contextual *within* the text (attention runs over all its tokens) but produces a **fixed** vector for that text once computed. So an indexed chunk embedding is:
- *contextual* in how it was computed (the whole chunk's tokens influenced it), but
- *static* once stored (it does not change based on the query that later retrieves it).

That is precisely the bi-encoder bottleneck from Q13. The document vector cannot adapt to the query. Late interaction (ColBERT) and cross-encoder reranking are the two ways to reintroduce query-dependence — one at the token level with precomputation, one with full joint attention and no precomputation.

**Practical implication for RAG:** because chunk embeddings are query-independent, *how you chunk* determines what a single vector must summarise. A 2,000-token chunk covering four topics produces a vector that is a blurred average of all four and will match none of them strongly. This is why chunking quality has such a large effect on retrieval quality — see `10-rag.md`.

### Interview Follow-ups

- Is a positional encoding static or contextual? (Static in the sense of being a fixed function of position; it is what allows the *contextual* layers to be order-aware.)
- How does instruction-prefixing partially reintroduce task-dependence into a static index? (See Q19 — the prefix conditions the vector on the task, though not on the specific query.)

---

## Q15: Sparse vs dense representations — compare.

### Answer

| | Sparse | Dense |
|---|---|---|
| Dimensionality | Vocabulary size (30k–1M+) | Fixed (256–4096) |
| Non-zero entries | A handful (the terms present) | All of them |
| Dimension meaning | Interpretable — one term per dimension | Not individually interpretable |
| Matching | Exact lexical overlap | Semantic similarity |
| Handles synonyms | **No** | **Yes** |
| Handles rare/exact terms (`ERR_4021`, part numbers, names) | **Excellent** | **Weak** — blurred into nearby concepts |
| Out-of-domain robustness | Strong (no training needed) | Degrades — the model may never have seen your jargon |
| Training required | No (statistical) | Yes |
| Index structure | Inverted index | Vector index (HNSW/IVF) |
| Storage | Compact (only non-zeros) | dims × 4 bytes per vector, always |
| Explainability | "Matched on these terms" | Hard to explain a match |
| Examples | BoW, TF-IDF, **BM25**, SPLADE | Word2Vec, SBERT, OpenAI embeddings |

**Why both survive.** They fail in complementary ways:

*Dense retrieval fails on:* exact identifiers, rare proper nouns, numbers, code symbols, and out-of-domain vocabulary the embedding model never learned. A query for `ERR_4021` may return semantically adjacent but wrong error codes, because the model has no reason to distinguish two rare tokens it barely saw.

*Sparse retrieval fails on:* paraphrases and synonyms ("how do I cancel" vs "terminate my subscription"), and any query that shares no vocabulary with the answer.

**Hybrid retrieval** runs both and fuses the results (Reciprocal Rank Fusion or score normalisation). It reliably beats either alone and is the production default. See `09-vector-databases-retrieval.md` Q19.

**Learned sparse — the interesting middle ground.** SPLADE uses a transformer to produce a *sparse* vector over the vocabulary, where the model **expands** the document with related terms it did not literally contain and learns term weights. So you get semantic expansion *and* the exact-match precision, interpretability, and inverted-index efficiency of sparse retrieval. It is the strongest argument that this is not simply a dense-wins story.

### Interview Follow-ups

- Why is dense retrieval bad at exact matching? (Rare tokens have weak, under-trained representations and get compressed toward semantically similar neighbours; the embedding has no mechanism to guarantee exact-token fidelity.)
- Can you store a sparse vector in a vector database? (Yes — modern vector DBs support sparse vectors and native hybrid search over both in one query.)

---

## Q16: Semantic vs lexical search — compare.

### Answer

| | Lexical (keyword) search | Semantic (vector) search |
|---|---|---|
| Matches on | Shared terms | Meaning |
| Underlying method | Inverted index + BM25 | Embeddings + ANN |
| Query "cancel my plan" finds "terminate subscription" | **No** | **Yes** |
| Query `ERR_4021` finds that exact code | **Yes** | Unreliable |
| Handles typos | With fuzzy matching | Somewhat, naturally |
| Multilingual (query in one language, docs in another) | No | Yes, with a multilingual model |
| Needs a model | No | Yes |
| Index build cost | Low | Must embed everything |
| Query latency | Very low (ms) | Low (ms with ANN) + embedding call |
| Explainability | High — show matched terms | Low |
| Cold start on new jargon | Works immediately | May need fine-tuning |
| Tunable per-field weighting | Easy and mature | Harder |

**The problem each one solves.**

Lexical search solves *finding the literal thing*. It is precise, fast, cheap, decades-mature, and it never invents a match. Its failure is the **vocabulary mismatch problem**: the user's words and the document's words differ.

Semantic search solves exactly that mismatch. Its failure is the mirror image: it is *too* tolerant, blurring distinctions between similar-looking-but-different specifics.

**Concrete failure examples worth quoting in an interview:**

```text
Query: "How do I stop being charged monthly?"
  Lexical  → misses a doc titled "Cancelling your subscription"     (no shared terms)
  Semantic → finds it

Query: "python 3.12 asyncio TimeoutError"
  Lexical  → finds the exact issue
  Semantic → may return general asyncio docs and other exception types

Query: "invoice INV-2024-88213"
  Lexical  → exact hit
  Semantic → returns other invoices; the specific id is not well represented
```

**The answer to "which should I use": both.** Hybrid search with RRF fusion, then a cross-encoder rerank. That combination handles paraphrase queries and exact-identifier queries in one system, and it is the design you should propose by default.

**Metadata filtering is the third leg** and is often underweighted: `date > 2024`, `department = finance`, `access_level <= user_level`. Neither lexical nor semantic scoring handles hard constraints — filters do, and they are also where access control must be enforced.

### Interview Follow-ups

- Why is RRF preferred over score averaging for fusion? (BM25 scores are unbounded and corpus-dependent; cosine scores are in [−1, 1]. They are not comparable, so rank-based fusion avoids the normalisation problem entirely.)
- When would you use *only* semantic search? (A small, homogeneous corpus of natural-language prose with no identifiers, codes, or exact-match requirements — and even then hybrid rarely hurts.)

---

## Advanced

---

## Q17: How do you choose an embedding model?

### Answer

**Criteria, roughly in priority order:**

| Criterion | Why it matters |
|---|---|
| **Performance on *your* data** | MTEB rank is a starting filter, not an answer. Benchmarks are partly saturated and partly contaminated; your domain may behave completely differently. Build a small eval set and measure. |
| Domain fit | Legal, biomedical, and code corpora often need a specialised or fine-tuned model |
| Dimensionality | Directly drives index memory and cost. 3072 dims costs 4× the memory of 768. |
| Max sequence length | If it truncates at 512 tokens, your 1,000-token chunks are silently half-ignored |
| Multilingual support | Only if you need it — multilingual models sometimes cost a little monolingual quality |
| Latency & throughput | Matters for query-time embedding, and for the initial ingestion of millions of chunks |
| Cost model | API per-token cost vs self-hosted GPU cost; ingestion is a one-off, queries are forever |
| Matryoshka support | Lets you truncate dimensions later without re-embedding |
| Licence | Commercial use, self-hosting rights |
| **Stability** | Changing the model means re-embedding the entire corpus. This is the hidden switching cost. |

**The evaluation procedure — this is the real answer:**
1. Build a small labelled set: 50–200 (query, relevant chunk) pairs from real user queries.
2. Index your actual corpus with each candidate model.
3. Measure **Recall@k** (does the answer appear at all — the retriever's real job) and **nDCG@10** (ranking quality).
4. Record latency, index size, and cost per million chunks.
5. Pick the cheapest model that clears your recall bar, not the highest scorer.

**Common pitfalls:**
- Picking by MTEB leaderboard alone.
- Forgetting the model's required **instruction prefix** (see Q19) — a silent 3–8 point quality loss.
- Mismatching the query and document prefixes, or using the wrong similarity metric.
- Ignoring the max sequence length and truncating chunks without noticing.
- Not budgeting for re-embedding when you later want to switch.

**Practical defaults:** a strong open model (BGE, E5, GTE, Nomic) if you self-host and want control; a hosted API (OpenAI, Voyage, Cohere) if you want quality with no infrastructure. Either way, measure on your data.

### Interview Follow-ups

- What is the total cost of switching embedding models in production? (Re-embed the entire corpus, rebuild the index, re-validate retrieval quality, and run both indexes in parallel during migration — non-trivial, which is why the choice deserves care.)
- How would you fine-tune an embedding model for your domain? (See Q8 — mine pairs, mine hard negatives, MultipleNegativesRankingLoss, validate on held-out queries.)

---

## Q18: Does embedding dimensionality matter, and what are the trade-offs?

### Answer

**What higher dimensions buy.** More capacity to encode distinctions — finer-grained semantics, more concepts kept separable. Quality generally improves with dimension, but with **strongly diminishing returns**: going 384 → 768 typically helps noticeably; 1536 → 3072 often gains a point or two.

**What they cost — everything scales linearly:**

| Resource | Effect of doubling dimensions |
|---|---|
| Storage / RAM per vector | 2× (dims × 4 bytes for float32) |
| Distance computation time | 2× |
| Index build time | ~2× |
| Network transfer | 2× |

**Concrete example — 10M chunks:**

```text
384 dims  × 4 bytes × 10M =  15.4 GB
768 dims  × 4 bytes × 10M =  30.7 GB
1536 dims × 4 bytes × 10M =  61.4 GB
3072 dims × 4 bytes × 10M = 122.9 GB
```

Add HNSW graph overhead (roughly `M × 2 × 4–8 bytes` per vector) on top. At 3072 dims you are into multi-machine territory for a corpus that fits comfortably on one box at 384.

**How to reduce dimensions:**

| Method | Quality impact | Notes |
|---|---|---|
| **Matryoshka truncation** | Small, and graceful | Only if the model was trained for it (see Q19) |
| Naive truncation of a non-Matryoshka model | **Severe** | Information is not concentrated in the early dimensions — do not do this |
| PCA on the corpus | Moderate | Requires fitting and storing the projection; must apply it to queries too |
| **Product Quantization** | Small-to-moderate | The standard production answer — 4–32× compression, see `09-vector-databases-retrieval.md` Q13 |
| Scalar quantisation (int8) | Very small | 4× reduction, easy win, widely supported |
| Binary quantisation | Moderate, recoverable with rescoring | 32× reduction; combine with a rescoring pass over top candidates |
| Pick a smaller model | Varies | Often the best answer — a good 384-dim model can beat a truncated 3072-dim one |

**The engineering judgement:** dimensionality is a quality-vs-cost dial, and the sweet spot for most applications is 384–1024. Measure Recall@k at each candidate dimension on your own data and choose the smallest that clears your bar. Combining a moderate dimension with int8 or binary quantisation plus rescoring usually beats simply picking the biggest model.

### Interview Follow-ups

- Why does naive truncation destroy a normal embedding? (No dimension ordering exists — information is distributed arbitrarily across all of them, so cutting half removes half the signal at random.)
- Why does binary quantisation work at all? (In high dimensions the *sign pattern* carries most of the directional information; Hamming distance approximates cosine well enough for a first pass, then you rescore the top candidates with full precision.)

---

## Q19: What are instruction-aware embeddings and Matryoshka embeddings?

### Answer

Two of the most practically important recent developments in embedding models.

**Instruction-aware (asymmetric) embeddings.**

The insight: retrieval is **asymmetric**. A short question and the long passage that answers it are not "similar texts" — they play different roles. Embedding both with the same function is a mismatch.

Instruction-aware models take a task prefix that tells the model which role the text plays:

```python
# E5-family convention
query_vec = embed("query: How do I reset my password?")
doc_vec   = embed("passage: To reset your password, navigate to Settings...")

# BGE convention: prefix queries only
query_vec = embed("Represent this sentence for searching relevant passages: How do I reset my password?")
doc_vec   = embed("To reset your password, navigate to Settings...")

# Instructor/E5-Mistral style: a full natural-language task instruction
query_vec = embed("Instruct: Given a support question, retrieve the relevant help article\nQuery: ...")
```

**Why it works.** The prefix shifts the text into a region of the space appropriate to its role, and lets one model serve multiple tasks (retrieval, classification, clustering, similarity) with different prefixes. Reported gains are meaningful — typically several points on retrieval benchmarks.

**The trap, and it is a common production bug:** using the wrong prefix, or none, degrades quality substantially and *silently*. Retrieval still returns results; they are just worse. Always check the model card, and always use the **same** convention at ingestion and at query time. Asymmetry means the document prefix and query prefix are deliberately different — do not "fix" that by making them match.

**Matryoshka Representation Learning (MRL).**

The insight: train the model so that the **first k dimensions** of its output are themselves a usable embedding, for several values of k. The training loss is applied at multiple truncation points simultaneously, which forces the model to pack the most important information into the earliest dimensions — nested like Matryoshka dolls.

**What this enables:**

```python
full = embed(text)             # 3072 dims

small  = full[:256]            # still works well, needs renormalising
medium = full[:768]
large  = full[:1536]
```

| Dimensions | Typical relative quality |
|---|---|
| 3072 (full) | 100% |
| 1536 | ~99% |
| 768 | ~97% |
| 256 | ~92% |

(Illustrative — measure on your data.)

**Why it is so useful in production:**
1. **Adaptive retrieval / two-stage search.** Search a cheap 256-dim index to get 1,000 candidates, then rerank them with the full 3072-dim vectors. Near-full quality at a fraction of the memory and compute.
2. **Cost tuning without re-embedding.** Store the full vector once; serve at whatever dimension your budget allows, and change your mind later.
3. **Tiered deployment.** Small dimensions on edge/mobile, full dimensions in the datacentre.

**Requirement:** the model must have been *trained* with MRL. Truncating a non-MRL embedding destroys it (see Q18). OpenAI's `text-embedding-3-*` (via the `dimensions` parameter), Nomic Embed, and several BGE/Jina models support it.

### Interview Follow-ups

- Must you renormalise after truncating a Matryoshka embedding? (Yes — the truncated vector is no longer unit-length, and cosine/dot-product comparisons assume it is.)
- How would you combine MRL with quantisation? (Truncate *and* quantise the first-stage index for maximum compression, then rescore with full-precision full-dimension vectors — a very strong memory/quality point.)

---

## Q20: How do you handle embedding drift and model updates in production?

### Answer

**The core constraint: embeddings from different models are not comparable.** Vectors live in a space defined by the model that produced them. Mixing model-A and model-B vectors in one index produces meaningless distances — and, critically, it *fails silently*. Search still returns results; they are just wrong.

So changing the embedding model means **re-embedding the entire corpus**.

**What "drift" means here — three distinct things:**

1. **Model version drift.** The provider updates a model behind the same name, or you upgrade. Vectors become incompatible with the existing index.
2. **Corpus drift.** New documents introduce vocabulary and topics the model handles poorly (new product names, new jargon), so retrieval quality degrades for recent content.
3. **Query drift.** User query patterns change — new intents, new terminology — and the previously-tuned retrieval configuration no longer matches them.

**Mitigations:**

| Practice | Why |
|---|---|
| **Pin the model version explicitly** | Never use a floating alias for embeddings; a silent provider update corrupts your index |
| **Store the model id + version in every vector's metadata** | Makes incompatibility detectable rather than silent, and enables partial migrations |
| **Store the raw chunk text alongside the vector** | You cannot re-embed what you did not keep; do not treat the vector store as the source of truth for content |
| **Keep an ingestion pipeline you can re-run** | Re-embedding must be a routine operation, not a heroic effort |
| **Maintain a golden retrieval eval set** | The only way to detect quality regressions; run it on a schedule and on every change |
| **Monitor retrieval health** | Score distributions, zero-result rate, click-through on retrieved sources, thumbs-down rate |

**The migration procedure (blue-green re-indexing):**
1. Build a **new index** with the new model, in parallel — do not modify the live one.
2. Re-embed the corpus into it (batch job; this is the expensive step).
3. Run the golden eval set against both indexes and compare Recall@k and nDCG.
4. Shadow-serve: send live queries to both, log both result sets, compare offline.
5. Canary a small percentage of traffic to the new index.
6. Cut over via an alias/pointer change; keep the old index warm for fast rollback.
7. Decommission the old index once confident.

**Cost reality check.** Re-embedding 10M chunks is a real bill and a multi-hour-to-multi-day job. Budget for it, and factor it into model selection (Q17) — this switching cost is why "pick the best model on the leaderboard this month" is bad strategy.

**Design tip:** treat the embedding model as a versioned dependency of the index, and make `(model_id, model_version, chunking_config)` a first-class part of your index identity. If any of the three changes, it is a new index.

### Interview Follow-ups

- Can you avoid a full re-embed by learning a projection between the two spaces? (There is research on embedding-space alignment, but it is lossy and rarely worth the risk in production versus a clean re-index.)
- What if re-embedding takes days and the corpus is changing? (Dual-write new documents to both indexes during the migration, and backfill the old ones in batches.)

---

## Q21: What are multimodal embeddings?

### Answer

**What they are.** A single vector space shared across modalities, so an image and the text describing it land near each other. That means you can query images with text, query text with images, and retrieve across modalities in one index.

**How they are trained — CLIP as the canonical example.**
1. Take 400M+ (image, caption) pairs scraped from the web.
2. Encode images with a vision encoder (ViT) and captions with a text encoder.
3. Train contrastively: within a batch of N pairs, maximise similarity for the N correct pairings and minimise it for the N²−N incorrect ones (a symmetric InfoNCE loss over the similarity matrix).
4. The two encoders learn to project into a shared space, aligned by the pairing signal.

**Key properties:**
- **Zero-shot classification.** Embed the candidate label texts ("a photo of a cat", "a photo of a dog"), embed the image, take the nearest label. No task-specific training.
- **Cross-modal retrieval.** One index, mixed content, text queries over images.
- The alignment is only as good as the caption data — which is why CLIP is strong on common objects and weak on fine-grained detail, text within images, spatial relationships, and counting.

**Beyond CLIP:**

| Model / approach | Adds |
|---|---|
| SigLIP | Sigmoid loss instead of softmax — trains better at large scale |
| ALIGN, EVA-CLIP | Scale and data-quality improvements |
| ImageBind | Six modalities (image, text, audio, depth, thermal, IMU) bound through the image space |
| Voyage-multimodal, Cohere Embed v3/v4 (image) | Production multimodal retrieval APIs |
| **ColPali / ColQwen** | Embeds **document page images** directly with late interaction — no OCR pipeline. Very strong for PDFs with tables, charts, and complex layout. |

**Where this matters for RAG — this is the practically important part.** Real enterprise documents are full of tables, charts, diagrams, and scanned pages. A text-only pipeline requires OCR plus layout parsing and loses information at every step. Two viable approaches:

1. **Multimodal embedding of page images** (ColPali-style): render each page, embed the image, retrieve pages directly, and pass the page image to a vision-language model to answer. Skips OCR entirely and preserves layout.
2. **VLM-generated descriptions**: use a vision model to describe each figure/table in text, embed that text, and index it alongside the surrounding prose. Simpler to bolt onto an existing text pipeline, and the description is human-inspectable.

**Limitations to name:** the modality gap (text and image embeddings occupy somewhat separate cones in the shared space, so cross-modal scores are not directly comparable to within-modal ones); weaker fine-grained discrimination than specialised unimodal models; larger index footprints for late-interaction variants; and inherited web-scale bias.

### Interview Follow-ups

- What is the modality gap and why does it matter for hybrid retrieval? (Text-text and text-image similarity scores have different distributions, so fusing them needs per-modality normalisation rather than a single threshold.)
- How would you build RAG over 100k scanned PDFs with tables? (Evaluate ColPali-style page-image retrieval against an OCR + layout-parse + text pipeline on a real eval set; the answer depends on layout complexity, and a hybrid — text index for prose, image index for figure-heavy pages — is often best.)

---

## Q22: What are the common failure modes of embedding-based retrieval?

### Answer

Knowing these — and their fixes — is what separates someone who has *built* a RAG system from someone who has read about one.

| Failure mode | Symptom | Root cause | Fix |
|---|---|---|---|
| **Exact-identifier miss** | Query `ERR_4021` returns other error codes | Rare tokens are weakly represented and blurred | Hybrid search with BM25 |
| **Chunk too large** | Retrieved chunk is topically right but the answer is buried, or the vector matches nothing strongly | One vector must summarise multiple topics — a blurred average | Smaller chunks; semantic chunking |
| **Chunk too small** | Retrieved text lacks the context to be interpretable | Pronouns/references point outside the chunk | Overlap, parent-document retrieval, contextual chunk headers |
| **Truncation** | Long chunks silently half-ignored | Chunk exceeds the model's max sequence length | Check the limit; chunk below it |
| **Missing/mismatched instruction prefix** | Quality quietly 3–8 points worse | Model expects `query:`/`passage:` prefixes | Follow the model card exactly |
| **Wrong similarity metric** | Odd rankings | Model trained for cosine, index configured for L2 on unnormalised vectors | Match the metric to the model; normalise consistently |
| **Norm/length bias with dot product** | Long documents always win | Dot product rewards large norms | Normalise, or use cosine |
| **Domain mismatch** | Poor results on specialised jargon | The embedding model never saw your vocabulary | Fine-tune on domain pairs; hybrid search as a stopgap |
| **Asymmetry ignored** | Short queries match short chunks regardless of relevance | Query and document lengths/roles differ | Instruction-aware model; or index generated hypothetical questions per chunk |
| **Multilingual mismatch** | Cross-language queries fail | Monolingual model | A multilingual embedding model |
| **No metadata filtering** | Stale or unauthorised documents retrieved | Semantic similarity has no notion of recency or permissions | Mandatory pre-filters from the authenticated session |
| **Duplicate chunks** | Top-5 results are five copies of the same passage | Corpus duplication; overlapping chunks | Deduplicate at ingestion; MMR for diversity |
| **Over-retrieval** | Accuracy drops as you pass more chunks | Lost-in-the-middle, attention dilution | Rerank and pass 3–8, not 20 |
| **Semantic-similarity ≠ relevance** | A chunk about the same topic that does not answer the question | Cosine similarity measures topical closeness, not question-answering | Cross-encoder reranking |
| **Embedding-space mixing** | Nonsense results after a model change | Two models' vectors in one index | Version the index; full re-embed on model change |
| **Negation blindness** | "drugs safe in pregnancy" returns "contraindicated in pregnancy" | Embeddings represent negation weakly | Reranking; explicit filters; query decomposition |

**The two most impactful in practice** are usually chunking and the absence of hybrid search plus reranking. Fix those before reaching for a more expensive embedding model.

**The diagnostic discipline:** when RAG gives a bad answer, always determine *which stage* failed before changing anything.

```text
1. Is the answer in the corpus at all?           -> ingestion/coverage problem
2. Was the right chunk in the top 100?           -> retrieval (recall) problem
3. Was it in the top 5 after reranking?          -> ranking (precision) problem
4. Was it in the prompt but ignored?             -> generation/faithfulness problem
```

Each stage has different fixes. Skipping this triage is why teams spend weeks tuning prompts to fix a retrieval bug.

### Interview Follow-ups

- Your retrieval Recall@10 is 0.95 but answers are still wrong. Where do you look? (Reranking and generation — the evidence is being retrieved but not used. Check chunk ordering, prompt construction, and faithfulness.)
- How do you detect a chunking problem specifically? (Inspect the retrieved chunks by hand for a sample of failures — you will see immediately whether they are too broad, truncated mid-sentence, or missing their context.)

---
