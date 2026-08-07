# Machine Learning Fundamentals

The ML core that AI engineering interviews still test: generalisation, evaluation, optimisation, and the classical algorithms you are expected to explain end to end.

**Questions:** 22

---

## Easy

---

## Q1: What is the difference between supervised, unsupervised, self-supervised, and reinforcement learning?

### Answer

| Paradigm | Supervision signal | Goal | Examples |
|---|---|---|---|
| Supervised | Human-provided labels | Map inputs → outputs | Classification, regression, fine-tuned rerankers |
| Unsupervised | None | Find structure | K-means, DBSCAN, PCA, topic models |
| Self-supervised | Labels derived from the data itself | Learn representations | Next-token prediction (GPT), masked LM (BERT), contrastive embeddings |
| Reinforcement | Reward from an environment | Learn a policy maximising cumulative reward | Game playing, robotics, RLHF/RLAIF for alignment |

**Why self-supervision matters most for LLMs:** labels are the bottleneck in supervised learning. Self-supervision manufactures labels from raw text — "predict the next token" is a supervised problem whose labels come free with the corpus. That is what made training on trillions of tokens possible, and it is the single idea underneath every modern foundation model.

**Where each appears in an LLM's life:**
1. Pretraining → self-supervised (next-token prediction)
2. Supervised fine-tuning (SFT) → supervised (instruction/response pairs)
3. Alignment (RLHF/DPO) → reinforcement learning or preference optimisation

### Interview Follow-ups

- Is next-token prediction supervised or unsupervised? (Mechanically supervised; the labels are self-generated, hence "self-supervised.")
- Where does semi-supervised learning fit? (Small labelled set + large unlabelled set: pseudo-labelling, consistency training.)

---

## Q2: Explain the bias-variance tradeoff.

### Answer

Expected prediction error decomposes into three parts:

```text
Expected Error = Bias² + Variance + Irreducible Noise
```

- **Bias** — error from wrong assumptions. The model is too simple to represent the true function. High bias → **underfitting**.
- **Variance** — error from sensitivity to the particular training sample. The model memorises noise. High variance → **overfitting**.
- **Noise** — irreducible; no model can beat it.

**The tradeoff:** increasing model capacity lowers bias but raises variance. The goal is not zero bias or zero variance but minimum *total* error.

| | High bias (underfit) | High variance (overfit) |
|---|---|---|
| Train error | High | Very low |
| Validation error | High, close to train | Much higher than train |
| Fix | Bigger model, better features, less regularisation, train longer | More data, regularisation, simpler model, early stopping, ensembling |

**Diagnostic:** compare train and validation error. Both high and similar → bias. Train low, validation high → variance. Also plot a learning curve: if validation error is still falling as you add data, more data helps (variance problem); if the curves have converged and both are high, more data will not help (bias problem).

**The modern caveat worth mentioning:** very large overparameterised networks exhibit "double descent" — test error rises, then falls again as capacity grows past the interpolation threshold. The classical U-shaped curve is not the whole story for deep learning, but it remains the right mental model for classical ML and for what regularisation is doing.

### Interview Follow-ups

- Which does bagging reduce, and which does boosting reduce? (Bagging → variance; boosting → bias.)
- Where do LLMs sit? (Massively overparameterised, yet they generalise — an active research area.)

---

## Q3: What are overfitting and underfitting, and how do you prevent overfitting?

### Answer

**Overfitting:** the model learns patterns specific to the training sample (including noise) that do not generalise. Train performance ≫ validation performance.

**Underfitting:** the model is too constrained to capture the real signal. Both train and validation performance are poor.

**Preventing overfitting, in rough order of impact:**

| Technique | Mechanism |
|---|---|
| More/better data | The most reliable fix — makes noise harder to memorise |
| Data augmentation | Synthetic variation acts as a prior on invariances |
| L1 / L2 regularisation | Penalise large weights, shrinking the effective hypothesis space |
| Early stopping | Halt when validation loss stops improving |
| Dropout | Randomly zero activations, preventing co-adaptation of units |
| Reduce capacity | Fewer parameters/layers/depth; prune features |
| Cross-validation | Better estimate so you do not fool yourself while tuning |
| Ensembling / bagging | Averages away individual models' variance |
| Tree constraints | `max_depth`, `min_samples_leaf`, `subsample`, `colsample` |
| Label smoothing | Prevents overconfident logits |
| Batch/layer norm | Regularising side effect plus optimisation benefit |

**In the LLM world the same idea appears as:** LoRA (drastically fewer trainable parameters), small learning rates and few epochs during fine-tuning (1–3 epochs is typical — more usually means catastrophic forgetting and memorisation), and holding out a real eval set that is not derived from the training data.

### Interview Follow-ups

- Why is early stopping described as an implicit regulariser? (Because stopping the optimiser early restricts how far the weights can travel from their small random initialisation, which bounds the effective size of the hypothesis space — the same thing an explicit penalty does, achieved by limiting optimisation rather than by adding a term to the loss. For linear models trained with gradient descent this equivalence is provable: the solution after t steps corresponds closely to an L2-penalised solution with a specific λ, where fewer steps means a stronger penalty. Intuitively, models fit the broad, high-signal structure first and memorise noise last, so cutting training at the validation minimum keeps the generalisable part and discards the memorisation. It is "implicit" because nothing in the loss function mentions regularisation — the constraint comes from the training procedure.)
- Can you overfit with a model that has fewer parameters than data points? (Yes — e.g. a deep tree on noisy labels.)

---

## Q4: Precision, recall, F1 — define them and explain when to optimise each.

### Answer

From the confusion matrix (TP, FP, FN, TN):

```text
Precision = TP / (TP + FP)     "Of what I flagged, how much was right?"
Recall    = TP / (TP + FN)     "Of what was actually there, how much did I catch?"
F1        = 2 * (P * R) / (P + R)     harmonic mean
Accuracy  = (TP + TN) / total
```

**Why the harmonic mean:** it punishes imbalance. A model with P=1.0 and R=0.0 has arithmetic mean 0.5 but F1 = 0 — correctly reflecting that it is useless.

**When to prioritise which:**

| Situation | Optimise | Because |
|---|---|---|
| Cancer screening | Recall | A missed case is catastrophic; a false alarm costs a follow-up test |
| Spam filter | Precision | Deleting a real email is worse than letting spam through |
| Fraud detection | Recall (with a precision floor) | Missed fraud costs money; flagged transactions get reviewed |
| Search / RAG retrieval | Recall@k at the retrieval stage, precision at the rerank stage | Retrieval must not lose the answer; the reranker's job is to remove noise |
| Balanced cost | F1 | No strong asymmetry |

**Accuracy's failure mode:** with 99% negatives, predicting "always negative" gives 99% accuracy and zero utility. Always ask about class balance before quoting accuracy.

**Threshold dependence:** precision and recall are functions of the decision threshold. Reporting them without stating the threshold — or better, reporting the whole PR curve — is incomplete.

### Interview Follow-ups

- What is F-beta, and when would you use F2? (β>1 weights recall more; F2 for screening.)
- What is macro vs micro vs weighted averaging in multiclass? (Macro treats classes equally — better when small classes matter; micro is dominated by frequent classes.)

---

## Q5: ROC-AUC vs PR-AUC — what is the difference and which do you use?

### Answer

| | ROC curve | Precision-Recall curve |
|---|---|---|
| Axes | TPR (recall) vs FPR | Precision vs Recall |
| Uses true negatives | Yes (in FPR) | No |
| Baseline for a random model | 0.5 | Positive class prevalence |
| Sensitive to imbalance | **No** — can look misleadingly good | **Yes** — reflects it honestly |
| Interpretation | P(random positive ranked above random negative) | Precision achievable at each recall level |

**The decisive difference:** FPR = FP/(FP+TN). With 1,000,000 negatives and 100 positives, going from 100 to 1,000 false positives barely moves FPR (0.0001 → 0.001), so ROC-AUC stays high. But precision collapses from 0.5 to 0.09 — which is what the user actually experiences. PR-AUC exposes this; ROC-AUC hides it.

**Use ROC-AUC when** classes are roughly balanced, or you care equally about both classes, or you want a threshold-independent measure of ranking quality that is stable across different prevalences.

**Use PR-AUC (average precision) when** the positive class is rare and is the class you care about — fraud, disease, relevant-document retrieval, content moderation.

### Interview Follow-ups

- Why does PR-AUC change when the class ratio changes even for the same model? (Its baseline is prevalence, so it is not comparable across datasets with different balance.)
- What is a calibration curve, and why can a model with excellent AUC still be badly calibrated? (AUC only measures ranking, not probability quality.)

---

## Q6: What is cross-validation and what variants matter?

### Answer

**Cross-validation** rotates the validation set across the data so every point is used for both training and validation, giving a lower-variance performance estimate than a single split — critical when data is limited.

**K-Fold:** split into K folds; train on K−1, validate on 1; repeat K times; average. K=5 or 10 is standard.

**Variants and when they are mandatory:**

| Variant | Use when |
|---|---|
| Stratified K-Fold | Classification — preserves class ratios in each fold. Default for classification. |
| Group K-Fold | Repeated entities (patients, users, documents) must not span folds |
| StratifiedGroupKFold | Both of the above at once |
| TimeSeriesSplit | Temporal data — always train on the past, validate on the future |
| Leave-One-Out | Very small datasets; near-unbiased but high variance and expensive |
| Repeated K-Fold | Reduce split-luck variance further |
| Nested CV | You are tuning hyperparameters *and* need an unbiased performance estimate |

**Why nested CV exists:** if you use the same CV loop to select hyperparameters and to report performance, the reported number is optimistically biased — you selected the configuration that happened to suit those folds. Nested CV has an inner loop for selection and an outer loop for estimation.

**Non-negotiable:** all preprocessing must be fit inside each fold (use a `Pipeline`). See `03-tabular-data-preprocessing.md`.

### Interview Follow-ups

- Why can't you use standard K-Fold on time series? (Trains on the future — leakage; also breaks the i.i.d. assumption CV relies on.)
- What does a large standard deviation across folds tell you? (Instability — small data, or heterogeneous subgroups.)

---

## Intermediate

---

## Q7: Explain gradient descent in detail.

### Answer

**1. Purpose.** Find parameters θ that minimise a loss function L(θ) when there is no closed-form solution (or it is too expensive to compute).

**2. Core idea.** The gradient ∇L(θ) points in the direction of steepest *increase* of the loss. So step in the opposite direction, repeatedly. Intuition: you are on a foggy hillside and can only feel the slope under your feet; take a small step downhill and repeat.

**3. Step-by-step operation.**
1. Initialise θ (randomly, or with a scheme like He/Xavier for neural nets).
2. Compute predictions and the loss over a set of examples.
3. Compute the gradient ∇L(θ) via backpropagation.
4. Update: `θ ← θ − η · ∇L(θ)`.
5. Repeat until the loss plateaus, the gradient norm is tiny, or the budget is spent.

**4. Variants:**

| Variant | Gradient computed on | Pros | Cons |
|---|---|---|---|
| Batch (full) GD | Entire dataset | Stable, smooth descent | Very slow; whole dataset in memory |
| Stochastic GD | One example | Fast updates, noise escapes shallow minima | Very noisy, hard to vectorise |
| Mini-batch GD | 32–1024 examples | Best of both; GPU-efficient | Batch size is another hyperparameter |

Mini-batch is what everyone means by "SGD" in practice.

**5. Important parameters.**
- **Learning rate η** — the single most important hyperparameter. Too high → divergence or oscillation; too low → glacial convergence and getting stuck in poor regions.
- **Batch size** — larger gives less gradient noise and better hardware utilisation, but generalisation can suffer; commonly paired with a scaled learning rate.
- **Momentum β** — accumulates a velocity term, damping oscillation across ravines and accelerating along consistent directions.
- **LR schedule** — warmup then cosine/linear decay is the modern default for transformers. Warmup prevents early instability when gradients are large and Adam's moment estimates are still poor.
- **Gradient clipping** — caps the gradient norm, essential for RNNs and large transformers.

**6. Advantages.** Scales to billions of parameters, needs only first-order information, memory-efficient relative to second-order methods, and works on any differentiable loss.

**7. Limitations.** Only finds local minima (fine in practice for deep nets — most local minima are near-equivalent, and saddle points are the real obstacle); sensitive to feature scaling; sensitive to learning rate; can plateau or diverge; requires differentiability.

**8. Typical use cases.** Training every neural network, logistic regression at scale, matrix factorisation, and the optimisation inside gradient boosting (which does gradient descent in *function* space).

### Example

```python
# Linear regression by gradient descent
import numpy as np

def gradient_descent(X, y, lr=0.01, epochs=1000):
    n, d = X.shape
    w = np.zeros(d)
    b = 0.0

    for _ in range(epochs):
        pred = X @ w + b
        error = pred - y                      # dL/dpred for MSE (up to a constant)

        grad_w = (2 / n) * X.T @ error
        grad_b = (2 / n) * error.sum()

        w -= lr * grad_w
        b -= lr * grad_b

    return w, b
```

### Interview Follow-ups

- What does Adam add over SGD+momentum? (Per-parameter adaptive learning rates from second-moment estimates; faster convergence, more memory — 2 extra states per parameter, which is why optimiser state dominates LLM training memory.)
- Why is warmup needed for transformers? (Early in training the gradients are large and badly scaled, and adaptive optimisers make this worse: Adam divides by a running estimate of the gradient's second moment, which at step 1–2 is computed from almost no data and is therefore an unreliable, high-variance denominator. A full-size learning rate applied through that noisy scaling produces huge, effectively random weight updates that can destroy the initialisation and diverge — often visible as a loss spike then NaN. Ramping the learning rate up over a few hundred to a few thousand steps lets the moment estimates stabilise before the steps get big. Transformers are especially sensitive because attention logits feed a softmax, so an early bad update saturates it and kills the gradient, and because post-norm architectures amplify residual-branch magnitude with depth. Related mitigations that reduce the need for warmup: pre-normalisation, gradient clipping, and careful residual scaling at initialisation.)
- Why are saddle points a bigger problem than local minima in high dimensions? (In d dimensions, a critical point being a true minimum requires all d curvatures positive — exponentially unlikely.)

---

## Q8: Explain K-Means clustering in detail.

### Answer

**1. Purpose.** Partition n points into K clusters, minimising within-cluster sum of squared distances to the cluster centroid (inertia).

```text
minimise  Σ_k Σ_{x in C_k} ||x - μ_k||²
```

**2. Core idea.** Alternate between two cheap steps: assign each point to its nearest centroid, then move each centroid to the mean of its assigned points. Each step provably decreases the objective, so it converges.

**3. Step-by-step operation.**
1. Choose K.
2. Initialise K centroids (k-means++ picks spread-out seeds by sampling proportional to squared distance from existing centroids).
3. **Assignment step:** assign each point to the nearest centroid (Euclidean).
4. **Update step:** recompute each centroid as the mean of its members.
5. Repeat 3–4 until assignments stop changing or centroid movement < tolerance.
6. Because the result depends on initialisation, repeat the whole thing `n_init` times and keep the lowest inertia.

This is Lloyd's algorithm — an instance of expectation-maximisation with hard assignments.

**4. Important parameters.**
- **K** — the number of clusters. Chosen by the elbow method (inertia vs K), silhouette score, gap statistic, or domain knowledge.
- **init** — `k-means++` (default, much better than random).
- **n_init** — number of restarts.
- **max_iter**, **tol** — convergence controls.

**5. Advantages.** Simple, fast (O(n · K · d · iterations)), scales to large data (MiniBatchKMeans handles millions of points), and always converges.

**6. Limitations.**
- You must specify K in advance.
- Assumes spherical, similarly sized, similarly dense clusters — because it minimises squared Euclidean distance. It fails on elongated or nested shapes.
- Sensitive to feature scaling and to outliers (means are not robust).
- Only finds a local optimum.
- Assigns every point, including noise.
- Euclidean distance degrades in very high dimensions.

**7. Typical use cases.** Customer segmentation, image colour quantisation, document clustering after dimensionality reduction, and — directly relevant to this handbook — **building the coarse partitions in an IVF vector index**, where centroids define the cells that queries are probed against.

### Example

```python
from sklearn.cluster import KMeans
from sklearn.preprocessing import StandardScaler

X_scaled = StandardScaler().fit_transform(X)      # scaling is required

km = KMeans(n_clusters=8, init="k-means++", n_init=10, random_state=0)
labels = km.fit_predict(X_scaled)
print(km.inertia_, km.cluster_centers_.shape)
```

### Interview Follow-ups

- Why does k-means++ help? (Bad random seeds can put two centroids in one true cluster and none in another; distance-proportional sampling makes that unlikely.)
- How does k-means relate to a Gaussian Mixture Model? (GMM with soft assignments and shared spherical covariance reduces to k-means in the limit.)
- Why is k-means used inside IVF and Product Quantization? (It is the cheapest way to learn representative centroids for space partitioning and codebooks.)

---

## Q9: Explain DBSCAN in detail, and compare it to K-Means.

### Answer

**1. Purpose.** Find clusters of arbitrary shape based on **density**, without specifying the number of clusters, while explicitly labelling outliers as noise.

**2. Core idea.** A cluster is a region where points are densely packed. If a point has at least `min_samples` neighbours within radius `eps`, it is a **core point** and its neighbourhood belongs to its cluster; clusters grow transitively through chains of core points. Points reachable from a core point but not themselves core are **border points**; unreachable points are **noise**.

**3. Step-by-step operation.**
1. Pick an unvisited point p.
2. Find all points within `eps` of p.
3. If fewer than `min_samples`, tentatively mark p as noise (it may later become a border point of another cluster) and move on.
4. Otherwise start a new cluster, add p and its neighbours.
5. For each new core point added, add *its* eps-neighbours too (this expansion is what allows arbitrary shapes).
6. Repeat until the cluster cannot grow; then move to the next unvisited point.

**4. Important parameters.**
- **eps** — the neighbourhood radius. The critical parameter. Choose it with a k-distance plot: sort each point's distance to its k-th nearest neighbour and look for the knee.
- **min_samples** — density threshold. Rule of thumb: ≥ d+1, commonly 2·d for d dimensions.
- **metric** — Euclidean by default; cosine is often better for embeddings.

**5. Advantages.** No need to specify K; finds non-convex/arbitrary shapes; identifies outliers natively; robust to outliers by construction.

**6. Limitations.**
- Struggles when clusters have **very different densities** — a single global `eps` cannot suit both. (HDBSCAN fixes this by varying the density threshold hierarchically.)
- Very sensitive to `eps`, which is hard to set in high dimensions.
- Suffers from the curse of dimensionality — distances concentrate, so density becomes meaningless. Reduce dimensions (UMAP/PCA) first when clustering embeddings.
- Border points can be assigned differently depending on processing order.
- O(n log n) with a spatial index, but O(n²) in high dimensions where indexes fail.

**7. Typical use cases.** Geospatial clustering, anomaly detection, and clustering sentence embeddings for topic discovery / near-duplicate document detection in a RAG corpus (typically UMAP → HDBSCAN, which is what BERTopic does).

### Example

**K-Means vs DBSCAN:**

| | K-Means | DBSCAN |
|---|---|---|
| Number of clusters | Must specify K | Discovered |
| Cluster shape | Spherical/convex only | Arbitrary |
| Outliers | Forced into a cluster | Labelled as noise |
| Parameters | K | eps, min_samples |
| Varying density | Handles it | Struggles |
| Scalability | Excellent (MiniBatch) | Moderate; degrades in high dim |
| Determinism | Depends on init | Mostly deterministic (border points aside) |
| Cluster "centre" | Explicit centroid | No centroid |

**Choosing between them:** if you need centroids (e.g. for an IVF index or for interpretable segment profiles) and roughly globular groups, use k-means. If shapes are irregular, the count is unknown, or you need outlier detection, use DBSCAN/HDBSCAN.

### Interview Follow-ups

- What does HDBSCAN change? (Builds a hierarchy over a mutual-reachability distance and extracts the most stable clusters — no `eps` needed.)
- Why does DBSCAN break in 768 dimensions, and what do you do about it? (DBSCAN's entire definition of a cluster rests on "how many points fall within `eps` of me," and in high dimensions that notion stops discriminating. Distances concentrate — the ratio between the nearest and farthest neighbour approaches 1 — so almost every point has a similar distance to almost every other point, and there is no `eps` that captures the dense regions without also capturing everything else. You typically get one giant cluster or all noise, with no useful setting in between. Volume makes it worse: an `eps`-ball's volume grows exponentially with dimension, so at fixed `eps` the neighbourhoods are essentially empty and every point looks like an outlier. The spatial indexes that give DBSCAN its O(n log n) behaviour also degenerate to brute force. The fix is to reduce dimensions before clustering — **UMAP to 5–15 dimensions, then HDBSCAN** is the standard recipe and what BERTopic does — because UMAP preserves local neighbourhood structure, which is precisely what a density-based method needs, and HDBSCAN removes the need to pick a single global `eps` at all. Using cosine distance instead of Euclidean helps for normalised embeddings but does not solve concentration on its own.)

---

## Q10: L1 vs L2 regularisation — what is the difference?

### Answer

Both add a penalty on weight magnitude to the loss, shrinking the effective capacity of the model.

```text
L1 (Lasso):        L + λ Σ |w_i|
L2 (Ridge):        L + λ Σ w_i²
Elastic Net:       L + λ₁ Σ |w_i| + λ₂ Σ w_i²
```

| | L1 (Lasso) | L2 (Ridge) |
|---|---|---|
| Penalty | Sum of absolute values | Sum of squares |
| Effect on weights | Drives many to **exactly zero** | Shrinks all toward zero, none exactly zero |
| Feature selection | Yes, built in | No |
| Solution with correlated features | Picks one arbitrarily, zeroes the rest | Spreads weight across them |
| Gradient of penalty | Constant ±λ (non-differentiable at 0) | Proportional to w (2λw) |
| Solution | No closed form | Closed form exists |
| Bayesian prior | Laplace | Gaussian |
| Use when | Many irrelevant features; want a sparse, interpretable model | Most features somewhat useful; multicollinearity |

**Why L1 produces exact zeros (the intuitive version):** L1's gradient is a constant λ regardless of how small the weight is, so it keeps pushing a weight all the way to zero and holds it there. L2's gradient is proportional to the weight, so as the weight shrinks the push weakens — it approaches zero asymptotically but never arrives. Geometrically, the L1 constraint region is a diamond whose corners lie on the axes, and the loss contour typically first touches a corner (where some coordinates are zero); the L2 region is a sphere, which has no corners.

**λ (or `1/C` in sklearn) controls strength:** λ→0 recovers the unregularised model (high variance); λ→∞ drives all weights to zero (high bias).

**Requires scaled features** — otherwise the penalty is applied unevenly across features with different units.

### Interview Follow-ups

- Is dropout more like L1 or L2? (Neither exactly; it approximates an ensemble average and has been related to adaptive L2.)
- Why does weight decay in AdamW differ from adding L2 to the loss? (With adaptive methods, L2 in the loss gets rescaled by the per-parameter denominator; AdamW decouples decay so it applies uniformly.)

---

## Q11: How do decision trees work, and what makes them split?

### Answer

A decision tree recursively partitions the feature space with axis-aligned threshold tests, choosing at each node the split that most reduces impurity.

**Splitting criteria (classification):**

```text
Gini impurity:  1 - Σ p_i²
Entropy:        - Σ p_i log₂ p_i
Information gain = impurity(parent) - weighted average impurity(children)
```

Both measure "how mixed" a node's labels are; both are 0 for a pure node and maximal for a uniform mix. Gini is slightly cheaper (no logarithm) and gives nearly identical trees — that is why it is sklearn's default.

**Regression:** minimise variance / MSE reduction (equivalently, sum of squared error within children).

**How a split is chosen:** for every feature, for every candidate threshold (midpoints between sorted unique values), compute the impurity reduction; take the best. This greedy, locally-optimal choice is why trees are fast but not globally optimal.

**Key hyperparameters (all are regularisation):** `max_depth`, `min_samples_split`, `min_samples_leaf`, `max_features`, `ccp_alpha` (cost-complexity pruning), `max_leaf_nodes`.

**Advantages.** Interpretable, no scaling needed, handles mixed feature types, captures non-linearities and interactions automatically, fast inference.

**Limitations.** High variance — a small data change can restructure the whole tree (which is exactly why bagging works so well on them); cannot extrapolate beyond the training range; biased toward high-cardinality features; axis-aligned splits struggle with diagonal boundaries; a single deep tree overfits badly.

### Interview Follow-ups

- Why does a fully grown tree have training error near zero? (It can isolate every point.)
- Why do trees not need feature scaling? (Splits depend only on value ordering.)
- What is cost-complexity pruning? (Grow fully, then remove subtrees whose complexity cost exceeds their accuracy benefit.)

---

## Q12: Bagging vs boosting — what is the difference?

### Answer

| | Bagging (e.g. Random Forest) | Boosting (e.g. XGBoost, LightGBM) |
|---|---|---|
| Training | Parallel, independent models | Sequential; each model fixes the previous ensemble's errors |
| Data per model | Bootstrap sample (with replacement) | Full data, reweighted or residual-targeted |
| Base learners | Deep, low-bias, high-variance trees | Shallow, high-bias "weak" trees (depth 3–8) |
| Primarily reduces | **Variance** | **Bias** |
| Overfitting risk | Low; more trees rarely hurts | Higher; more rounds *can* hurt — needs early stopping |
| Parallelisable | Yes, trivially | Not across rounds (but within-tree yes) |
| Typical accuracy on tabular data | Strong | Usually the best |
| Hyperparameter sensitivity | Low | High (learning rate × n_estimators interact) |

**Bagging's mechanism:** average many high-variance, roughly unbiased models. Because their errors are partly uncorrelated, averaging cancels variance without adding bias. Random Forest adds a second decorrelation source — random feature subsets at each split — which is what makes it better than plain bagged trees.

**Boosting's mechanism:** fit a weak model, compute where the ensemble is still wrong, then fit the next model to those residuals (gradient boosting fits the negative gradient of the loss). Each round reduces bias. The learning rate shrinks each contribution so the ensemble improves gradually and generalises better — hence the classic tradeoff: lower learning rate + more trees.

**Key boosting hyperparameters:** `learning_rate`, `n_estimators` (with early stopping on a validation set), `max_depth`/`num_leaves`, `subsample`, `colsample_bytree`, `min_child_weight`, `reg_lambda`.

**Practical guidance:** on tabular data, gradient-boosted trees (LightGBM/XGBoost/CatBoost) remain the strongest default and routinely beat neural networks. Random Forest is the better choice when you want a solid result with almost no tuning.

### Interview Follow-ups

- Why does Random Forest choose a random feature subset per split rather than per tree? (Stronger decorrelation.)
- What is the difference between AdaBoost and gradient boosting? (AdaBoost reweights misclassified examples; gradient boosting fits the loss gradient — AdaBoost is the special case with exponential loss.)
- Why is LightGBM faster than XGBoost? (Histogram binning, leaf-wise growth, and Exclusive Feature Bundling / GOSS.)

---

## Q13: What is the curse of dimensionality?

### Answer

As dimensionality grows, geometric intuition from 2D/3D breaks down in ways that damage learning and retrieval.

**Concrete consequences:**

1. **Data sparsity.** To maintain the same density, the sample size must grow exponentially with d. Ten points cover a line well; they cover a 10-dimensional cube not at all.
2. **Distance concentration.** The ratio between the nearest and farthest neighbour distances tends to 1 as d → ∞. "Nearest neighbour" becomes nearly meaningless, which undermines k-NN, k-means, and DBSCAN.
3. **Volume concentrates in the shell.** Almost all the volume of a high-dimensional ball lies near its surface, so points are all "on the edge."
4. **Everything is nearly orthogonal.** Random vectors in high dimensions have near-zero cosine similarity. (This is also *useful* — it is why high-dimensional embedding spaces can hold enormous numbers of distinguishable concepts.)
5. **Overfitting is easier.** More dimensions means more ways to separate points spuriously.

**Mitigations:** dimensionality reduction (PCA, UMAP), feature selection, regularisation, using domain structure (convolutions, attention), and **learned** embeddings rather than raw high-dimensional sparse features.

**The crucial nuance for RAG interviews:** embeddings are 768–3072 dimensional, so why does vector search work? Because embeddings are **not uniformly distributed** in that space — real data lies on a much lower-dimensional manifold, and the embedding model was trained specifically to make semantically similar items close and dissimilar ones far. The *intrinsic* dimensionality is far below the *ambient* dimensionality. ANN indexes like HNSW exploit exactly this structure; on genuinely uniform random high-dimensional data they degrade toward brute force.

### Interview Follow-ups

- Why does cosine similarity behave better than Euclidean distance in high dimensions for text? (It ignores magnitude, which is dominated by document length and frequency effects.)
- What does Matryoshka representation learning let you do? (Truncate an embedding to fewer dimensions with graceful quality loss — a direct lever on this problem.)

---

## Q14: Explain PCA.

### Answer

**1. Purpose.** Reduce dimensionality while retaining as much variance as possible, by projecting onto a new orthogonal basis ordered by explained variance.

**2. Core idea.** Find the direction along which the data varies most (PC1), then the orthogonal direction with the next-most variance (PC2), and so on. Keeping the first k components keeps most of the information in k dimensions.

**3. Step-by-step operation.**
1. **Centre** the data (subtract the mean per feature) — and standardise if features have different units.
2. Compute the covariance matrix (or use SVD directly on the centred matrix, which is what implementations do — more numerically stable).
3. Compute eigenvectors and eigenvalues. Eigenvectors are the principal components; eigenvalues are the variance explained along each.
4. Sort by eigenvalue descending, keep the top k.
5. Project: `X_reduced = X_centered @ W_k`.

**4. Important parameters.** `n_components` — an integer, or a float like `0.95` meaning "keep enough components to explain 95% of variance." Also `whiten` (scale components to unit variance) and `svd_solver`.

**5. Advantages.** Removes multicollinearity, speeds up downstream models, enables visualisation, denoises (small-variance directions are often noise), and has a closed-form deterministic solution.

**6. Limitations.** Linear only; components are linear combinations and therefore hard to interpret; requires scaling; variance is not the same as predictive usefulness (PCA is unsupervised and can discard a low-variance but highly discriminative direction); sensitive to outliers.

**7. Typical use cases.** Preprocessing before distance-based models, visualisation (usually with t-SNE/UMAP for non-linear structure), image compression, and **compressing embeddings** to cut vector-database memory — though for that purpose PQ or Matryoshka truncation is usually preferred.

### Example

```python
from sklearn.decomposition import PCA
from sklearn.preprocessing import StandardScaler

Xs = StandardScaler().fit_transform(X)
pca = PCA(n_components=0.95, random_state=0)
X_reduced = pca.fit_transform(Xs)

print(X_reduced.shape)
print(pca.explained_variance_ratio_[:5].cumsum())
```

### Interview Follow-ups

- PCA vs LDA? (PCA is unsupervised and maximises variance; LDA is supervised and maximises class separability.)
- PCA vs t-SNE/UMAP? (PCA is linear, deterministic, preserves global structure, and has a reusable transform; t-SNE/UMAP are non-linear, stochastic, preserve local neighbourhoods, and are for visualisation — do not feed t-SNE output to a model.)

---

## Q15: What are the main similarity and distance metrics, and when do you use each?

### Answer

| Metric | Formula | Range | Sensitive to magnitude | Typical use |
|---|---|---|---|---|
| Cosine similarity | (a·b)/(‖a‖‖b‖) | [−1, 1] | No | Text/semantic embeddings |
| Dot product | a·b | (−∞, ∞) | Yes | Embeddings trained with it; MIPS |
| Euclidean (L2) | √Σ(aᵢ−bᵢ)² | [0, ∞) | Yes | Geometric/tabular data, k-means |
| Manhattan (L1) | Σ|aᵢ−bᵢ| | [0, ∞) | Yes | Grid-like data, robust to outliers |
| Jaccard | \|A∩B\| / \|A∪B\| | [0, 1] | n/a | Sets, near-duplicate detection |
| Hamming | # differing positions | integer | n/a | Binary codes, hashed embeddings |

**Cosine measures angle, not length.** Two documents about the same topic, one 10× longer, have similar direction but very different magnitudes — so cosine is the natural choice for text.

**The crucial identity:** for **L2-normalised** vectors,

```text
euclidean(a, b)² = 2 - 2·cos(a, b)
```

so cosine, dot product, and Euclidean produce **identical rankings** on normalised vectors. This is why vector databases often just normalise at ingestion and use dot product internally — it is one fused multiply-add per dimension, cheaper than computing norms at query time.

**The rule that actually matters in interviews:** use the metric the embedding model was **trained with**. OpenAI and most sentence-transformers models are trained with cosine and output normalised (or near-normalised) vectors. Some models (certain retrieval models, and ColBERT-style scorers) are trained for dot product where magnitude encodes confidence or importance — normalising those destroys information. Check the model card; do not guess.

### Interview Follow-ups

- Why can dot product retrieval return a long document for a short query? (Larger norm inflates the score — a known bias, sometimes desirable, often not.)
- What is the difference between a metric and a similarity? (A metric satisfies non-negativity, identity, symmetry, and the triangle inequality; cosine similarity does not — but cosine *distance* = 1 − cosine similarity is commonly used, and it is not a true metric either. This matters for index structures that rely on the triangle inequality.)

---

## Q16: What is the difference between generative and discriminative models?

### Answer

| | Discriminative | Generative |
|---|---|---|
| Models | P(y \| x) — the decision boundary | P(x, y) or P(x) — the data distribution |
| Answers | "Which class is this?" | "What does this class look like?" / "What comes next?" |
| Can sample new data | No | Yes |
| Data efficiency for pure classification | Usually better | Usually worse |
| Examples | Logistic regression, SVM, random forest, BERT classifier, cross-encoder reranker | Naive Bayes, GMM, HMM, VAE, diffusion models, GPT-style LLMs |

**Explanation:** a discriminative model only needs to learn where classes differ. A generative model must model the entire data distribution — a harder problem, but it yields more: sampling, density estimation, and handling missing inputs.

**Where LLMs sit:** an autoregressive LLM is generative — it models P(next token | previous tokens), and by the chain rule that gives a distribution over whole sequences. Notably, generative models can *do* discriminative tasks by conditioning ("Classify this review as positive or negative"), which is why one pretrained generative model replaced dozens of task-specific discriminative ones.

**Where this distinction shows up practically in RAG:** a **bi-encoder** embedding model is discriminative-flavoured (it learns a similarity space), a **cross-encoder** reranker is discriminative (scores relevance of a pair), and the answer generator is generative. Choosing the right family per stage is a real design decision.

### Interview Follow-ups

- Why is Naive Bayes generative, and why does it work well with little data despite a wrong independence assumption? (Its strong bias reduces variance; and it only needs the argmax to be right, not the probabilities.)
- Can you use an LLM's token probabilities for classification? (Yes — constrained decoding or comparing logprobs of candidate labels; often better calibrated than free-text output.)

---

## Q17: What is the difference between parametric and non-parametric models?

### Answer

**Parametric:** a fixed number of parameters, decided before seeing the data. Training compresses the data into those parameters and then discards it.

**Non-parametric:** the number of effective parameters grows with the data; the model may retain the training data itself.

| | Parametric | Non-parametric |
|---|---|---|
| Parameter count | Fixed | Grows with n |
| Examples | Linear/logistic regression, neural networks (fixed architecture), Naive Bayes | k-NN, decision trees, kernel SVM, Gaussian processes, random forests |
| Assumptions about form | Strong | Weak |
| Bias / variance | Higher bias, lower variance | Lower bias, higher variance |
| Data needed | Less | More |
| Inference cost | Cheap and constant | Can grow with n (k-NN scans the data) |
| Interpretability | Often good | Varies |

**Note on terminology:** "non-parametric" does not mean "no parameters" — it means the complexity is not fixed in advance. A 400B-parameter LLM is technically parametric (fixed architecture), which is exactly why RAG is needed: you cannot add knowledge without retraining, so you attach a non-parametric memory (the vector index) alongside the parametric model. That framing — "RAG gives a parametric model a non-parametric knowledge store" — is a strong interview answer.

### Interview Follow-ups

- Why is k-NN called a lazy learner? (No training phase; all work happens at query time.)
- Is fine-tuning parametric knowledge editing? (Yes — and that is its key drawback versus retrieval for fast-changing facts.)

---

## Q18: How do you handle a train/serve performance gap, and what is model drift?

### Answer

**First, enumerate the possible causes in order of likelihood:**

1. **Data leakage** in training (see `03-tabular-data-preprocessing.md`) — the most common cause of a large gap.
2. **Training/serving skew** — features computed differently in the two paths (different library version, different default fill value, different timezone, unit mismatch). Fix by sharing one transformation artifact (a `Pipeline`) or a feature store.
3. **Distribution shift** — the live data differs from training data.
4. **Overfitting to the validation set** through excessive tuning.
5. **Feedback loops** — the model's own predictions change future data (a recommender narrows what users see).

**Types of drift:**

| Type | What changes | Example | Detection |
|---|---|---|---|
| Covariate shift | P(x), not P(y\|x) | New user demographic | KS test, PSI, KL divergence on feature distributions |
| Label/prior shift | P(y) | Fraud rate rises | Monitor prediction rate vs actual base rate |
| Concept drift | P(y\|x) itself | Post-pandemic spending patterns | Rolling performance metrics once labels arrive |
| Upstream data drift | Schema/pipeline change | A field starts arriving null | Data validation checks |

**Monitoring plan:** log inputs and predictions; track feature distributions against a training baseline (PSI > 0.2 is a common alert threshold); track prediction distribution; track true performance as labels land; alert on data-quality violations. Then retrain on a schedule or on a drift trigger, with a champion/challenger comparison before promotion.

**In LLM systems the analogue is:** prompt/model version changes, upstream document corpus changes altering retrieval quality, and shifting user query distributions. Same discipline — versioned artifacts, offline eval sets, online metrics, canary rollouts.

### Interview Follow-ups

- How do you detect concept drift with no labels? (Proxy signals: confidence distribution shifts, user corrections, thumbs-down rate, abandonment.)
- What is PSI and how is it computed? (Population Stability Index — binned distribution comparison, similar in spirit to a symmetric KL.)

---

## Advanced

---

## Q19: What is the difference between a loss function and a metric, and how do you choose a loss?

### Answer

A **loss** is what the optimiser minimises — it must be differentiable (for gradient methods) and defined per example. A **metric** is what humans evaluate — it can be non-differentiable, threshold-based, or defined only over a whole dataset.

Example: you minimise cross-entropy but you report F1 or PR-AUC. F1 is not differentiable (it depends on a threshold and on counts), so it cannot be a loss directly.

**Common losses:**

| Task | Loss | Why |
|---|---|---|
| Binary classification | Binary cross-entropy (log loss) | Proper scoring rule; gradient is well-behaved; penalises confident mistakes heavily |
| Multiclass | Categorical cross-entropy | Same, with softmax |
| Regression (Gaussian noise) | MSE | Penalises large errors quadratically; the MLE under Gaussian noise |
| Regression with outliers | MAE or Huber | Linear penalty in the tails, so outliers do not dominate |
| Regression, need quantiles | Quantile/pinhole loss | Asymmetric, gives prediction intervals |
| Ranking / retrieval | Contrastive, triplet, InfoNCE, MultipleNegativesRanking | Optimises relative order, not absolute values — the right family for embeddings |
| Imbalanced detection | Focal loss, weighted CE | Down-weights easy negatives |
| LLM pretraining | Next-token cross-entropy | Self-supervised, matches the generative objective |
| Preference alignment | DPO / PPO objectives | Optimises against pairwise human preference |

**Why cross-entropy over accuracy as a loss:** accuracy has zero gradient almost everywhere (it only changes when a prediction crosses the threshold). Cross-entropy gives a smooth signal proportional to how wrong the probability is.

**Choosing a loss is a modelling statement.** MSE says "errors are symmetric and Gaussian, big ones are much worse." MAE says "all errors scale linearly, and I do not trust my outliers." Weighted CE says "one class of mistake costs more." Align the loss with business cost as closely as differentiability allows, and keep the business metric as the reported metric.

### Interview Follow-ups

- Why is MSE a bad loss for classification? (Slow gradients when badly wrong under sigmoid; not a proper fit to a Bernoulli likelihood.)
- What does label smoothing do to cross-entropy? (Targets become 1−ε and ε/(K−1), reducing overconfidence and improving calibration.)
- What is InfoNCE and where does it appear? (Contrastive loss over one positive and many negatives — the basis of modern embedding model training; see `08-embeddings.md`.)

---

## Q20: What is model calibration and why does it matter?

### Answer

A model is **calibrated** if its predicted probabilities match observed frequencies: among samples predicted at 0.7, about 70% are actually positive.

**Why it matters:** any decision that combines a probability with a cost needs the probability to be real. Thresholding at an expected-cost-optimal point, ranking by expected value, routing low-confidence cases to humans, or combining multiple model scores all break if scores are miscalibrated.

**How to measure:**
- **Reliability diagram** — bin predictions, plot mean predicted vs observed frequency; the diagonal is perfect.
- **Expected Calibration Error (ECE)** — weighted average gap across bins.
- **Brier score** — mean squared error of probabilities; decomposes into calibration + refinement.

**Common miscalibration:**
- Modern deep networks are typically **overconfident** (pushing probabilities toward 0/1), a side effect of minimising cross-entropy on separable data with large capacity.
- Models trained on resampled/rebalanced data are systematically shifted.
- SVMs and boosted trees produce distorted scores (boosting pushes scores toward the extremes; SVM outputs are not probabilities at all).

**How to fix:**

| Method | How | Notes |
|---|---|---|
| Platt scaling | Fit logistic regression on validation scores | Parametric, needs little data |
| Isotonic regression | Fit a monotonic step function | Non-parametric, more flexible, needs more data |
| Temperature scaling | Divide logits by a learned scalar T | The standard for neural nets; single parameter, preserves accuracy exactly |

Always fit calibration on a **held-out** set, never on training data.

**LLM relevance:** temperature in decoding is literally this operation applied to the next-token logits. And LLM verbalised confidence ("I'm 90% sure") is notoriously poorly calibrated, which is why production systems prefer logprob-based confidence, self-consistency agreement rates, or retrieval-grounded citations over asking the model how sure it is.

### Interview Follow-ups

- Why does temperature scaling not change accuracy? (It is a monotonic transform of the logits, so the argmax is unchanged.)
- Which is better calibrated out of the box: logistic regression or a random forest? (Logistic regression, since it directly optimises log loss; RF probabilities from vote fractions are usually under-confident at the extremes.)

---

## Q21: Explain the ROC/threshold decision in cost terms — how do you pick an operating point?

### Answer

Choosing a threshold is a business decision, not a modelling one. Do it by minimising expected cost.

**Step 1 — get the cost matrix.** From stakeholders: cost of a false positive (C_FP) and a false negative (C_FN). For example, in a support-ticket auto-resolution system, a wrong auto-resolution costs an angry customer; an unnecessary escalation costs 10 minutes of agent time.

**Step 2 — the optimal threshold under a cost matrix.** For a calibrated model, act positive when

```text
p(y=1|x) * C_FN  >  (1 - p(y=1|x)) * C_FP

=>  threshold  t* = C_FP / (C_FP + C_FN)
```

So if a false negative is 9× as costly as a false positive, `t* = 1/10 = 0.1` — flag aggressively.

**Step 3 — validate empirically.** Sweep the threshold on the validation set, compute total expected cost at each point, pick the minimum. This handles miscalibration and non-linear costs that the closed form ignores.

**Step 4 — consider constraints instead of pure cost.** Often the real requirement is "maximise recall subject to precision ≥ 0.9" (a review-capacity limit) or "flag at most 500 cases/day." Both are threshold selections against a constraint rather than a cost.

**Step 5 — monitor it.** The optimal threshold drifts with prevalence. Re-tune on a schedule.

### Example

```python
import numpy as np
from sklearn.metrics import precision_recall_curve

precision, recall, thresholds = precision_recall_curve(y_val, p_val)

C_FP, C_FN = 1.0, 9.0
best_t, best_cost = 0.5, float("inf")

for t in thresholds:
    pred = (p_val >= t)
    fp = np.sum(pred & (y_val == 0))
    fn = np.sum(~pred & (y_val == 1))
    cost = fp * C_FP + fn * C_FN
    if cost < best_cost:
        best_t, best_cost = t, cost

print(f"threshold={best_t:.3f}  expected cost={best_cost:.1f}")
```

### Interview Follow-ups

- Why is 0.5 almost never the right threshold? (It is only optimal for a calibrated model with equal costs and balanced classes.)
- How does this extend to a three-way decision (auto-approve / auto-reject / human review)? (Two thresholds bracketing an abstention band — the standard design for a human-in-the-loop AI system.)

---

## Q22: How do you design an A/B test for a model change, and what can go wrong?

### Answer

**Design:**
1. **Define one primary metric** tied to business value (task success rate, conversion, resolution time). Pre-register it. Define guardrail metrics too (latency p95, cost per request, error rate).
2. **Unit of randomisation** — usually the user, not the request, so a single user gets a consistent experience. Randomise on a stable hash of the user id.
3. **Power analysis** — compute the sample size needed to detect your minimum detectable effect at the desired power (typically 80%) and significance (typically 5%). Do this *before* launching; underpowered tests are the most common waste.
4. **Run for a full number of business cycles** (usually ≥1–2 weeks) to absorb weekday/weekend effects.
5. **Sanity check with an A/A test** or a pre-experiment period to verify no bias in the assignment.
6. **Analyse once, at the pre-declared time**, on the pre-declared metric. Use sequential testing if you must peek.

**Common failure modes:**

| Problem | Consequence | Mitigation |
|---|---|---|
| Peeking / early stopping | Inflated false-positive rate | Fixed horizon, or sequential/Bayesian methods |
| Multiple metrics or segments | Multiple-comparisons false positives | Pre-register; correct (Bonferroni/FDR) |
| Sample ratio mismatch | Broken assignment or logging | SRM check as a hard gate |
| Novelty / primacy effects | Short-term lift that fades | Longer run; analyse by cohort tenure |
| Network / spillover effects | Treatment leaks to control | Cluster randomisation |
| Simpson's paradox | Aggregate direction differs from every segment | Pre-declared segment analysis |
| Underpowered test | "No difference" that is really "no idea" | Power analysis; report confidence intervals |

**For LLM features specifically:** offline eval first (a curated eval set with LLM-as-judge and/or human labels), then a shadow/canary deployment, then the A/B test. Add guardrails on cost per request and p95 latency, since a quality win that triples cost or doubles latency may not ship. And be careful with LLM-as-judge in an A/B context — judges have position and verbosity biases; randomise the order and validate the judge against human labels.

### Interview Follow-ups

- What is a switchback test and when is it needed? (Time-sliced randomisation, for marketplace/system-level effects where user-level randomisation leaks.)
- Your A/B shows +2% quality but +40% latency. What do you do? (Quantify the latency-to-engagement relationship, or find a cheaper path to the same quality — smaller model, caching, distillation.)
- How do you A/B test something with no clean automated metric, like summary quality? (Pairwise human preference on a sampled set, plus indirect behavioural signals like edit rate and regeneration rate.)

---
