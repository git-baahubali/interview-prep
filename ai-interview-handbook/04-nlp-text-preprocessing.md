# NLP & Text Preprocessing

Classical text preparation plus the subword tokenisation that modern LLMs actually use. Knowing *which* steps still apply in the transformer era is the real test here.

**Questions:** 8

---

## Easy

---

## Q1: What is tokenisation, and what levels of tokenisation exist?

### Answer

**Tokenisation** splits raw text into the discrete units a model operates on, and maps each unit to an integer id.

| Level | Unit | Vocabulary size | Strengths | Weaknesses |
|---|---|---|---|---|
| Character | `c`, `a`, `t` | Tiny (~100s) | No OOV ever; tiny vocab | Very long sequences; little semantic meaning per token |
| Subword | `token`, `##isation` | 30k–200k | No OOV, handles morphology, moderate lengths | Words split unintuitively; not language-neutral |
| Word | `cat`, `running` | 100k–1M+ | Intuitive, meaningful units | Huge vocab, OOV on any unseen word, poor for agglutinative languages |
| Sentence / chunk | full sentences | n/a | Retrieval and embedding units | Too coarse for a language model |

**Why subword won:** it fixes the two failure modes of word tokenisation at once. A rare word like `hyperparameterisation` becomes known pieces rather than `<UNK>`, and the vocabulary stays small enough for a tractable softmax over the output layer. Meanwhile it keeps sequences far shorter than character-level, which matters because attention cost grows quadratically with sequence length.

### Example

```python
from transformers import AutoTokenizer

tok = AutoTokenizer.from_pretrained("bert-base-uncased")
print(tok.tokenize("Tokenization handles unknown words."))
# ['token', '##ization', 'handles', 'unknown', 'words', '.']
```

### Interview Follow-ups

- Why does the same English sentence cost more tokens in Hindi or Thai? (Vocabulary is fit predominantly on English-heavy corpora, so other scripts fragment into more pieces — this directly raises API cost and consumes context.)
- Why does an LLM struggle to count letters in a word? (It sees token ids, not characters.)

---

## Q2: What is the difference between stemming and lemmatisation?

### Answer

Both reduce word forms to a common base. Stemming chops affixes with heuristic rules; lemmatisation maps to a real dictionary form using vocabulary and part-of-speech information.

| | Stemming | Lemmatisation |
|---|---|---|
| Method | Rule-based suffix stripping | Dictionary + morphological analysis + POS |
| Output | May not be a real word (`studi`) | Always a valid lemma (`study`) |
| Speed | Very fast | Slower |
| Accuracy | Lower; over/under-stems | Higher |
| Needs POS | No | Yes, for good results |
| Example | `running → run`, `studies → studi`, `better → better` | `running → run`, `studies → study`, `better → good` |

### Example

```python
from nltk.stem import PorterStemmer, WordNetLemmatizer

print(PorterStemmer().stem("studies"))                       # studi
print(WordNetLemmatizer().lemmatize("studies"))              # study
print(WordNetLemmatizer().lemmatize("better", pos="a"))      # good
```

**Modern relevance:** you use these for sparse/lexical retrieval (BM25 benefits from normalised terms) and for classical feature engineering. You do **not** apply them before feeding text to a transformer — the model's pretrained tokenizer expects natural text, and stemming destroys signal it was trained to use.

### Interview Follow-ups

- Over-stemming vs under-stemming: which one hurts recall and which hurts precision? (Over-stemming collapses words that should stay distinct — `universe`, `university`, and `universal` all becoming `univers` — so a query for one matches documents about the others: **precision** falls. Under-stemming fails to collapse words that belong together — `datum` and `data`, or `ran` and `run` — so a query misses relevant documents: **recall** falls. Aggressive stemmers (Lovins, Porter on some inputs) skew towards over-stemming; conservative ones and lemmatisers skew towards under-stemming. Which error to prefer depends on the task, and in modern pipelines the question is often moot because dense retrieval handles morphological variation in the embedding space instead.)
- Would you lemmatise documents before embedding them for vector search? (No.)

---

## Q3: What are stop words, and should you always remove them?

### Answer

**Stop words** are extremely frequent, low-information words (`the`, `is`, `at`, `and`). Removing them shrinks the vocabulary and reduces noise in bag-of-words models.

**Remove them when:**
- Building BoW/TF-IDF features for topic modelling or document classification.
- Using sparse lexical retrieval where they add index size but no discrimination. (Note that BM25/TF-IDF already down-weight them via IDF, so the gain is mostly efficiency.)

**Do NOT remove them when:**
- Feeding a transformer — they carry syntax and meaning the model uses.
- Sentiment analysis — `not good` becomes `good`. Negation words are decisive.
- Any task where phrase structure matters: NER, question answering, translation, `to be or not to be`.
- Semantic/dense retrieval — the embedding model was trained on natural sentences.

### Example

```python
# Danger: default stop word lists include negations.
from sklearn.feature_extraction.text import ENGLISH_STOP_WORDS
print("not" in ENGLISH_STOP_WORDS)     # True  -> destroys sentiment
```

### Interview Follow-ups

- Why is a domain-specific stop-word list often better than a generic one? (In a corpus of legal contracts, `agreement` is nearly a stop word.)
- How does IDF make explicit stop-word removal partially redundant? (IDF is `log(N / df)`, and a stop word appears in nearly every document, so `df ≈ N` and its IDF approaches zero — the term is automatically down-weighted to near-irrelevance without anyone maintaining a list. This is strictly better than a hand-written list in two ways: it is *corpus-specific* (in a corpus of Python documentation, "python" is effectively a stop word and IDF discovers that while no standard list would), and it is graded rather than binary, so a moderately common word is partially discounted instead of being kept or deleted outright. What IDF does not give you is the storage and speed benefit of not indexing those terms at all, which is why search engines still often drop them — a performance decision rather than a relevance one. BM25 keeps this property and adds saturation on top; see `09-vector-databases-retrieval.md` Q17.)

---

## Intermediate

---

## Q4: How do BPE, WordPiece, and SentencePiece/Unigram differ?

### Answer

All three learn a subword vocabulary from a corpus; they differ in the *merge criterion* and in how they treat whitespace.

**BPE (Byte-Pair Encoding)** — used by GPT-2/3/4, RoBERTa, Llama.
1. Start with a base vocabulary (bytes or characters).
2. Count all adjacent symbol pairs in the corpus.
3. Merge the **most frequent** pair into a new symbol.
4. Repeat until the vocabulary reaches the target size.
5. The ordered list of merges *is* the tokenizer; encoding replays them greedily.

**Byte-level BPE** starts from the 256 raw bytes, which guarantees any Unicode input is encodable — there is literally no OOV, no `<UNK>` token.

**WordPiece** — used by BERT, DistilBERT.
Same loop, but instead of raw frequency it merges the pair that most increases the training-corpus likelihood, scored roughly as:

```text
score(a, b) = freq(ab) / (freq(a) * freq(b))
```

This prefers merges where the pair is more frequent than its parts would predict, so it avoids gluing a rare token onto a very common one. Continuation pieces are marked `##`.

**Unigram (SentencePiece)** — used by T5, ALBERT, XLNet, mT5.
Works **top-down**: start with a large candidate vocabulary, then iteratively *remove* the tokens whose deletion least hurts corpus likelihood (EM-style). At encode time it finds the most probable segmentation over the whole word rather than greedily merging, and it can sample alternative segmentations (subword regularisation) for data augmentation.

**SentencePiece** is often confused with Unigram — it is actually the *library/framework* (it can train either BPE or Unigram). Its distinguishing feature is treating input as a raw stream and encoding whitespace explicitly as `▁`, so tokenisation is fully reversible and needs no language-specific pre-tokeniser.

| | BPE | WordPiece | Unigram |
|---|---|---|---|
| Direction | Bottom-up merging | Bottom-up merging | Top-down pruning |
| Criterion | Pair frequency | Likelihood gain | Likelihood loss when removed |
| Encoding | Greedy replay of merges | Longest-match greedy | Probabilistic best segmentation |
| Multiple segmentations | No | No | Yes (sampling) |
| Marker | `Ġ` prefix (byte-level) | `##` continuation | `▁` word start |
| Used by | GPT-*, Llama, RoBERTa | BERT | T5, ALBERT, mT5 |

### Interview Follow-ups

- Why does byte-level BPE eliminate `<UNK>` entirely? (Because the base vocabulary is the 256 possible byte values rather than a set of characters. Any input whatsoever — an unseen emoji, Cyrillic text in an English-trained model, a corrupted file, raw binary — is by definition a sequence of bytes, so it always decomposes into tokens the vocabulary contains. Character-level BPE cannot promise this: a character absent from the training corpus has no base token and must map to `<UNK>`. The cost is that rare scripts fragment into many tokens — a Devanagari character can take 3 bytes and therefore up to 3 tokens — which is why non-Latin languages consume more of your context window and cost more per word. GPT-2 onwards use byte-level BPE for exactly this guarantee.)
- What is subword regularisation and why does it help low-resource translation? (A Unigram model assigns a probability to each possible segmentation of a word, so instead of always taking the single best one you can *sample* from the distribution during training — the same word appears as `un_help_ful` in one epoch and `unhelp_ful` in another. This is data augmentation on the tokenisation itself: the model stops overfitting to one canonical segmentation, learns that several subword paths carry the same meaning, and becomes robust at inference to segmentations it would otherwise find surprising. It matters most in low-resource translation because there is little data to learn from, so cheap augmentation gives a real gain, and because morphologically rich languages have many plausible segmentations per word. BPE-dropout is the equivalent trick for BPE — randomly skip merges during training. At inference you use the single best segmentation as usual.)
- If you fine-tune on a new domain (e.g. chemistry), should you extend the vocabulary? (Sometimes — it shortens sequences, but new embedding rows start untrained and require enough data.)

---

## Q5: What is a typical text preprocessing pipeline, and how does it differ for classical ML vs transformers?

### Answer

**Classical pipeline (TF-IDF + logistic regression / BM25 index):**

1. Unicode normalisation (NFKC) and encoding fixes
2. Strip HTML/markup, URLs, emails (or replace with placeholder tokens)
3. Lowercase
4. Remove or normalise punctuation, numbers, emojis
5. Tokenise (whitespace/regex)
6. Remove stop words
7. Stem or lemmatise
8. Build n-grams
9. Vectorise (BoW / TF-IDF)

**Transformer pipeline:**

1. Light cleaning only — strip boilerplate, fix mojibake, normalise whitespace, deduplicate
2. Apply the model's own tokenizer
3. Truncate or chunk to the model's max length
4. Build attention masks; pad within the batch

**Why the difference:** classical models see each token as an independent feature, so reducing surface variation (`Running`, `running`, `runs` → `run`) helps generalisation. Transformers were pretrained on natural text — casing, punctuation, and stop words are signal they already know how to use, and their subword vocabulary already handles morphology. Aggressive cleaning creates a train/inference mismatch with pretraining and *lowers* accuracy.

### Example

```python
import re, unicodedata

def light_clean(text):
    """Safe for transformers and embedding models."""
    text = unicodedata.normalize("NFKC", text)
    text = re.sub(r"<[^>]+>", " ", text)          # strip HTML tags
    text = re.sub(r"\s+", " ", text)              # collapse whitespace
    return text.strip()
```

### Interview Follow-ups

- Which cleaning steps still matter for RAG document ingestion? (Boilerplate/nav removal, deduplication, encoding fixes, table/layout extraction — all of them affect retrieval quality directly.)
- Why is corpus deduplication important for pretraining? (Memorisation, wasted compute, and contaminated evaluation.)

---

## Q6: What is TF-IDF, and why is it better than raw counts?

### Answer

TF-IDF weights each term by how frequent it is in a document, discounted by how common it is across the corpus. It exists because raw counts rank common-but-uninformative words highest.

**Term frequency** — how much this document is about the term:

```text
tf(t, d) = count(t, d) / len(d)
```

**Inverse document frequency** — how rare and therefore how discriminative the term is:

```text
idf(t) = log( N / (1 + df(t)) ) + 1
```

where N is the number of documents and df(t) the number containing t.

```text
tfidf(t, d) = tf(t, d) * idf(t)
```

**Intuition:** `the` appears in every document, so its IDF is ~0 and its weight vanishes automatically. `photosynthesis` appears in few documents, so when it does appear it dominates the vector. This is why explicit stop-word removal becomes largely redundant.

### Example

```python
from sklearn.feature_extraction.text import TfidfVectorizer

vec = TfidfVectorizer(ngram_range=(1, 2), min_df=2, max_df=0.9, sublinear_tf=True)
X = vec.fit_transform(corpus)      # sparse, L2-normalised rows
```

`sublinear_tf=True` applies `1 + log(tf)`, since the tenth occurrence of a word says much less than the second — this is the same diminishing-returns idea that BM25 formalises with saturation.

**Limitations:** no word order, no synonyms (`car` and `automobile` are orthogonal dimensions), no context, and a vector length equal to the vocabulary size. These limitations are precisely what dense embeddings solve — see `08-embeddings.md`.

### Interview Follow-ups

- How does BM25 improve on TF-IDF? (Term-frequency saturation via `k1` and document-length normalisation via `b` — see `09-vector-databases-retrieval.md`.)
- Why is TF-IDF still used in production RAG systems in 2025? (Exact keyword, code identifier, and rare-entity matching, where dense retrieval fails.)

---

## Advanced

---

## Q7: What are N-grams, and what is the trade-off in choosing N?

### Answer

An **n-gram** is a contiguous sequence of n tokens. They give bag-of-words models a limited window onto word order.

For `"the model is not good"`:
- Unigrams: `the`, `model`, `is`, `not`, `good`
- Bigrams: `the model`, `model is`, `is not`, `not good`
- Trigrams: `the model is`, `model is not`, `is not good`

**Why they exist:** unigrams cannot distinguish `not good` from `good not`, so sentiment and negation are lost. `is not` as a single feature captures it.

**The trade-off:**

| Larger N | Effect |
|---|---|
| Captures more local order and phrases | ✔ Better precision on phrases (`machine learning` ≠ `machine` + `learning`) |
| Feature space grows near-exponentially | ✘ Sparsity, memory, overfitting |
| Each specific n-gram gets rarer | ✘ Most never appear in test data (poor generalisation) |

In practice `ngram_range=(1, 2)` or `(1, 3)` with `min_df` pruning is the sweet spot for text classification. Character n-grams (`analyzer="char_wb"`, 3–5) are surprisingly strong for noisy text, misspellings, and language identification.

**Character n-grams also matter for retrieval:** they power fuzzy matching and typo tolerance in lexical search engines.

### Interview Follow-ups

- Why did n-gram language models plateau? (Fixed window, no generalisation across similar contexts — the exact problems distributed representations and then attention solved.)
- What is the role of smoothing in n-gram LMs? (Assigning non-zero probability to unseen n-grams: Laplace, Kneser-Ney.)

---

## Q8: How do you handle text longer than the model's maximum length?

### Answer

**First, know your limit.** Encoder models like BERT cap at 512 tokens (positional embeddings); modern LLMs run 128k–1M+ but with real cost, latency, and attention-degradation consequences.

**Strategies:**

| Strategy | How | Best for |
|---|---|---|
| Truncation | Keep first N (or last N) tokens | Classification where the signal is front-loaded (news, abstracts) |
| Head + tail | First 128 + last 382 tokens | Documents with informative conclusions; a well-known strong baseline for long-doc BERT classification |
| Chunk + aggregate | Split, encode each, pool (mean/max) or vote | Long-document classification, embedding long texts |
| Sliding window with stride | Overlapping windows, merge predictions | Extractive QA, NER — avoids cutting an answer span in half |
| Hierarchical model | Encode chunks, then a second model over chunk vectors | Very long documents with structure |
| Retrieval instead of stuffing | Index chunks, retrieve only what is relevant | RAG — the standard answer for large corpora |
| Map-reduce / refine summarisation | Summarise chunks, then summarise summaries | Whole-document summarisation |
| Long-context architecture | Longformer, BigBird, or a long-context LLM | When global context genuinely matters |

**Key judgement call in interviews:** for a *corpus* the answer is retrieval, not a bigger context window. Long context and RAG are complements — long context helps a single document; retrieval is what scales to millions. Even with a 1M-token window, stuffing everything raises cost linearly, adds latency, and degrades accuracy in the middle of the context ("lost in the middle").

### Example

```python
# Sliding window for extractive QA: overlap prevents splitting the answer span.
enc = tok(
    question,
    document,
    max_length=384,
    truncation="only_second",
    stride=128,                       # overlap between windows
    return_overflowing_tokens=True,
    return_offsets_mapping=True,
    padding="max_length",
)
# Score every window, then take the best span across windows.
```

### Interview Follow-ups

- What is the "lost in the middle" effect, and how do you mitigate it? (Reorder so the strongest evidence sits at the start and end of the prompt.)
- Why does chunk overlap help retrieval? (See chunking in `10-rag.md`.)

---
