# Tabular Data Preprocessing

Practical feature engineering for structured data — the part of ML interviews where sloppy answers leak the test set.

**Questions:** 7

---

## Easy

---

## Q1: What are the main strategies for handling missing values?

### Answer

Pick a strategy based on *why* the data is missing, not just on what is convenient.

| Strategy | How | Use when | Risk |
|---|---|---|---|
| Drop rows | `df.dropna()` | Very few rows affected, missing at random | Loses data; biases the sample if missingness is informative |
| Drop column | `df.drop(columns=...)` | Column is mostly empty (e.g. >60–70%) | Discards a possibly strong signal |
| Mean / median imputation | `SimpleImputer` | Numeric, roughly unimodal | Shrinks variance, distorts correlations |
| Mode / "Unknown" category | `SimpleImputer(strategy="most_frequent")` | Categorical | Can create a fake dominant class |
| Forward/backward fill | `df.ffill()` | Time series where the last value carries over | Leaks future values if you `bfill` carelessly |
| Model-based (KNN, MICE/iterative) | `KNNImputer`, `IterativeImputer` | Missingness correlates with other features | Expensive; can overfit |
| Native handling | LightGBM / XGBoost / CatBoost | Tree models | Not available for linear models or NNs |
| Missingness indicator | `add_indicator=True` | Missing *itself* is predictive | Extra columns |

**Key nuance — the three missingness mechanisms:**
- **MCAR** (completely at random): dropping is safe, just wasteful.
- **MAR** (at random given observed features): model-based imputation works well.
- **MNAR** (not at random — e.g. high earners refuse to state income): imputation alone is misleading; add an indicator column so the model can learn the pattern.

**Critical rule:** fit the imputer on the **training set only**, then apply to validation/test. Computing the mean over the full dataset is target leakage.

### Example

```python
from sklearn.impute import SimpleImputer
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer

numeric = Pipeline([("impute", SimpleImputer(strategy="median", add_indicator=True))])
categorical = Pipeline([("impute", SimpleImputer(strategy="constant", fill_value="Unknown"))])

pre = ColumnTransformer([
    ("num", numeric, numeric_cols),
    ("cat", categorical, categorical_cols),
])

pre.fit(X_train)          # statistics learned from train only
X_val_t = pre.transform(X_val)
```

### Interview Follow-ups

- Why median over mean for skewed features? (Robust to outliers.)
- Why is imputing before the train/test split wrong? (The imputed value is computed from statistics — a mean, median, or model — and if it is fitted on the full dataset those statistics contain test-set information, which then leaks into training rows. Your validation score becomes optimistically biased and no longer predicts production, where the imputer will only ever have seen historical data. Fit the imputer on the training fold and `transform` the validation and test folds; putting it inside a `Pipeline` makes this structurally impossible to get wrong.)

---

## Q2: What are the main categorical encoding techniques, and when do you use each?

### Answer

| Encoding | Output | Cardinality it suits | Notes |
|---|---|---|---|
| One-hot | Sparse binary columns | Low (< ~15) | Safe default for linear models and NNs; no false ordering |
| Ordinal / label | Single integer column | Any, if genuinely ordered | Implies order — fine for trees, harmful for linear/distance models |
| Target (mean) encoding | Single float column | High (hundreds+) | Powerful but leaks badly without out-of-fold computation |
| Frequency / count | Single float column | High | Cheap; captures "how common is this value" |
| Hashing | Fixed-width columns | Very high / unbounded | No vocabulary needed; collisions accepted |
| Learned embedding | Dense vector | High | Neural nets, or CatBoost-style handling |
| Binary / base-N | log₂(k) columns | Medium-high | Compromise between one-hot and ordinal |

**Explanation:** the two things you are trading off are *dimensionality* and *information loss*. One-hot preserves all distinctions but explodes width. Ordinal keeps width at 1 but invents an ordering the model may believe (`red=1, blue=2, green=3` implies blue is between red and green). Target encoding compresses to one highly informative column, at the cost of the strongest leakage risk in tabular ML.

**Target encoding done safely:**

```python
# Out-of-fold target encoding: each row's encoding never sees its own target.
from sklearn.model_selection import KFold
import numpy as np

def oof_target_encode(df, col, target, n_splits=5, smoothing=20):
    global_mean = df[target].mean()
    encoded = np.full(len(df), np.nan)

    for tr_idx, val_idx in KFold(n_splits, shuffle=True, random_state=0).split(df):
        stats = df.iloc[tr_idx].groupby(col)[target].agg(["mean", "count"])
        # smooth rare categories toward the global mean
        smoothed = (stats["mean"] * stats["count"] + global_mean * smoothing) / (stats["count"] + smoothing)
        encoded[val_idx] = df.iloc[val_idx][col].map(smoothed).fillna(global_mean)

    return encoded
```

**Handling unseen categories at inference:** one-hot needs `handle_unknown="ignore"`; target/frequency encoders need a fallback to the global mean; hashing handles them for free.

### Interview Follow-ups

- Why do tree models tolerate ordinal encoding better than logistic regression? (Trees split on thresholds, so they can isolate any subset with enough splits; linear models multiply the integer by one coefficient.)
- What is the dummy variable trap, and does it matter for trees? (Perfect collinearity; matters for unregularised linear models, not for trees.)

---

## Intermediate

---

## Q3: Standardisation vs normalisation — what is the difference and when does scaling matter?

### Answer

**Standardisation (`StandardScaler`)** centres to mean 0 and scales to unit variance:

```text
z = (x - mean) / std
```

**Normalisation (`MinMaxScaler`)** rescales to a fixed range, usually [0, 1]:

```text
x' = (x - min) / (max - min)
```

| | Standardisation | Min-Max normalisation | Robust scaling |
|---|---|---|---|
| Formula | (x − μ)/σ | (x − min)/(max − min) | (x − median)/IQR |
| Output range | Unbounded, centred at 0 | Bounded [0, 1] | Unbounded, centred at 0 |
| Outlier sensitivity | Moderate | **Severe** (one outlier squashes everything else) | Low |
| Assumes | Roughly Gaussian-ish | Known bounds | Nothing |
| Use for | Linear/logistic regression, SVM, PCA, k-means, NNs | Image pixels, bounded features, some NNs | Heavy-tailed features |

**When scaling matters:**
- **Required:** distance-based methods (k-NN, k-means, SVM with RBF), PCA, anything gradient-descent-based (unscaled features cause elongated loss surfaces and slow/unstable convergence), and L1/L2 regularisation (penalties are otherwise unfair across features).
- **Not required:** decision trees, random forests, gradient-boosted trees — splits depend only on the *ordering* of values, which monotonic scaling does not change.

**Same leakage rule:** `fit` on train, `transform` on everything.

### Interview Follow-ups

- Why does PCA give misleading components on unscaled data? (It maximises variance, so the largest-unit feature dominates.)
- Which scaler for a feature with a long right tail? (Log or Yeo-Johnson transform first, then standardise; or `RobustScaler`.)

---

## Q4: What is data leakage, and what are the most common ways it happens?

### Answer

**Data leakage** is when information that would not be available at prediction time influences training. The symptom is beautiful validation scores and a model that collapses in production.

**Common sources:**

1. **Preprocessing before splitting.** Fitting a scaler, imputer, PCA, or target encoder on the whole dataset lets test statistics reach the model. **Fix:** split first, then fit transformers inside a `Pipeline` under cross-validation.
2. **Target leakage in features.** A column that is a proxy or downstream consequence of the label — e.g. `total_paid` when predicting default, or `discharge_date` when predicting readmission. **Fix:** ask for every feature, "would I know this at the moment of prediction?"
3. **Temporal leakage.** Random K-fold on time series trains on the future to predict the past. **Fix:** `TimeSeriesSplit` / a forward-chaining split with a gap.
4. **Group leakage.** The same patient, user, or document appears in both train and test, so the model memorises the entity. **Fix:** `GroupKFold` / `StratifiedGroupKFold` on the entity id.
5. **Duplicate rows** across the split — same as group leakage. Deduplicate first.
6. **Tuning on the test set.** Repeatedly selecting hyperparameters against the same holdout leaks it. **Fix:** train/validation/test, or nested CV.
7. **Resampling before splitting.** SMOTE applied before the split synthesises test-like rows into train.

**In LLM/RAG systems this reappears as evaluation contamination:** your eval questions were built from the same documents in the index, or the benchmark is in the model's pretraining data. Same disease, different substrate.

### Example

```python
from sklearn.pipeline import Pipeline
from sklearn.model_selection import cross_val_score, StratifiedGroupKFold

pipe = Pipeline([("pre", pre), ("model", model)])   # everything refit per fold

cv = StratifiedGroupKFold(n_splits=5)
scores = cross_val_score(pipe, X, y, groups=patient_ids, cv=cv)
```

### Interview Follow-ups

- Your CV AUC is 0.99 and production AUC is 0.62. What do you check first? (Leakage — a gap that large is almost never a modelling problem. In order: inspect feature importances, because leakage usually shows up as one dominant feature; check whether any feature is computed *after* the label event (a status field, a resolution timestamp, an updated-at column); check whether the split respects time and entity boundaries, since random splitting of temporal or grouped data leaks the future and leaks the same customer into both sides; check whether any preprocessing was fitted before the split; and check for duplicate rows spanning train and test. Only after all of those would I look at drift between training data and production traffic.)
- Why is SMOTE-before-split worse than useless? (SMOTE synthesises minority rows by interpolating between real neighbours. If you oversample before splitting, a synthetic training row can be an interpolation of two rows that ended up in the validation set — so the model has effectively seen the validation data, and near-duplicates of the same point sit on both sides. Validation recall looks excellent and production recall collapses. Worse than useless is fair: you have not only failed to improve the model, you have destroyed your ability to detect that failure. Resample inside the training fold only, which `imblearn.pipeline.Pipeline` does correctly during cross-validation.)

---

## Q5: How do you handle class imbalance?

### Answer

Start by asking whether it is actually a problem: imbalance only hurts when the minority class is what you care about and your metric or loss ignores it.

**Fix the metric first.** Accuracy is meaningless at 99:1. Use precision/recall, F1, PR-AUC (better than ROC-AUC under heavy imbalance because it ignores the huge true-negative mass), or a cost-weighted metric.

**Then, in rough order of preference:**

| Technique | How | Notes |
|---|---|---|
| Class weights | `class_weight="balanced"`, `scale_pos_weight` | Cheapest, no data invented, usually first choice |
| Threshold tuning | Move the decision cutoff off 0.5 | Often the single biggest win; tune on validation |
| Random undersampling of majority | Drop majority rows | Fast; discards information |
| Random oversampling of minority | Duplicate minority rows | Risks overfitting exact copies |
| SMOTE / ADASYN | Synthesise minority points by interpolation | Only on training folds; poor for high-dimensional or categorical data |
| Focal loss | Down-weights easy examples | Deep learning, extreme imbalance |
| Ensemble (BalancedBagging, EasyEnsemble) | Many balanced subsamples | Strong, more expensive |
| Reframe as anomaly detection | One-class SVM, Isolation Forest | When minority is <0.1% and heterogeneous |

**Two things people get wrong:**
1. Resampling changes the predicted probability scale — a model trained on rebalanced data is no longer calibrated. If you need real probabilities, calibrate afterwards (`CalibratedClassifierCV`) or adjust the intercept.
2. Never resample the validation or test set. Evaluate on the true distribution.

### Interview Follow-ups

- Why PR-AUC over ROC-AUC when positives are 0.5% of the data? (ROC-AUC uses the false positive rate, whose denominator is the huge negative class — so 5,000 false positives out of 995,000 negatives moves FPR by only 0.005 and ROC-AUC barely notices. Precision uses the same 5,000 in a denominator of *predicted positives*, so it collapses. With 0.5% positives you can have a 0.95 ROC-AUC model whose top predictions are 90% false alarms, which is the number the fraud team actually cares about. PR-AUC also has a meaningful baseline (0.005, the positive rate) whereas ROC-AUC's baseline is always 0.5 regardless of imbalance. See `05-machine-learning-fundamentals.md` Q5.)
- How would you choose the threshold if a false negative costs 20× a false positive? (Minimise expected cost directly on the validation set.)

---

## Q6: How do you detect and treat outliers?

### Answer

**Detection:**

| Method | Idea | Suits |
|---|---|---|
| Z-score (|z| > 3) | Distance in standard deviations | Roughly normal, univariate |
| IQR rule (< Q1 − 1.5·IQR or > Q3 + 1.5·IQR) | Quartile-based, robust | Skewed, univariate |
| Percentile capping (1st/99th) | Hard bounds | Quick, pragmatic |
| Mahalanobis distance | Accounts for covariance | Multivariate, correlated features |
| Isolation Forest / LOF / DBSCAN noise points | Model-based | Multivariate, non-linear |
| Domain rules | Physically impossible values | Always do this first |

**Treatment:** the decision is *why* the point is extreme.
- **Data error** (age = 300, negative price): fix it or treat it as missing.
- **Genuine rare event** (a real fraudulent transaction): keep it — it may be the entire signal.
- **Genuine but destabilising**: cap/winsorise, log-transform, or switch to a robust model or loss (Huber, quantile, tree ensemble).

**Model sensitivity:** linear regression, k-means, PCA, and any squared-error loss are highly outlier-sensitive. Tree ensembles are largely immune in the feature space (splits ignore magnitude) but still sensitive to outliers in the **target** under MSE.

**Never** drop test-set outliers to make your metrics look better.

### Interview Follow-ups

- Why does the IQR rule survive skewness better than the z-score? (Quartiles are not inflated by the extreme values themselves.)
- Why can removing outliers hurt a fraud model? (Because in fraud the outliers *are* the signal. A £40,000 transaction at 03:00 from a new device is a statistical outlier and also exactly the positive class you are trying to detect — trimming at the 99th percentile deletes a large fraction of your labelled positives and teaches the model that fraud looks normal. The general principle: outlier treatment is only valid for values that are *errors* (a sensor reporting -999, an age of 300, a duplicated decimal point), not for values that are rare-but-real. Diagnose which you have before deciding, and when in doubt prefer a robust model or a winsorised *copy* of the feature to deleting rows.)

---

## Advanced

---

## Q7: Why use `sklearn` Pipelines and `ColumnTransformer` instead of transforming a DataFrame step by step?

### Answer

Because a pipeline makes leakage structurally hard and makes the training and serving paths identical.

**What it gives you:**
1. **Leakage safety under CV.** `cross_val_score(pipe, ...)` refits every transformer on each training fold. Manual preprocessing before CV silently leaks.
2. **One artifact to deploy.** `joblib.dump(pipe)` captures imputation statistics, category vocabularies, and scaler parameters together with the model, so training/serving skew cannot creep in.
3. **Hyperparameter search over preprocessing too.** You can tune `pre__num__impute__strategy` alongside `model__max_depth`.
4. **Column-type routing.** `ColumnTransformer` applies different transforms to numeric, categorical, and text columns in one object.

### Example

```python
from sklearn.compose import ColumnTransformer
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.impute import SimpleImputer
from sklearn.model_selection import GridSearchCV
from sklearn.linear_model import LogisticRegression

numeric_cols = ["age", "income", "tenure"]
categorical_cols = ["country", "plan"]

pre = ColumnTransformer([
    ("num", Pipeline([
        ("impute", SimpleImputer(strategy="median")),
        ("scale", StandardScaler()),
    ]), numeric_cols),
    ("cat", Pipeline([
        ("impute", SimpleImputer(strategy="constant", fill_value="Unknown")),
        ("ohe", OneHotEncoder(handle_unknown="ignore", min_frequency=10)),
    ]), categorical_cols),
], remainder="drop")

pipe = Pipeline([("pre", pre), ("model", LogisticRegression(max_iter=1000))])

search = GridSearchCV(
    pipe,
    {
        "pre__num__impute__strategy": ["median", "mean"],
        "model__C": [0.1, 1.0, 10.0],
    },
    scoring="average_precision",
    cv=5,
)
search.fit(X_train, y_train)
```

**Production note:** `handle_unknown="ignore"` and `min_frequency` are what keep the pipeline from crashing on a category it has never seen — a very common cause of 500s in a deployed tabular service.

### Interview Follow-ups

- How would you add a custom transformer? (Subclass `BaseEstimator, TransformerMixin`; implement `fit`/`transform`.)
- What breaks if the serving code reorders or renames columns? (Use `get_feature_names_out()` and validate the input schema — e.g. with Pydantic — at the API boundary.)

---
