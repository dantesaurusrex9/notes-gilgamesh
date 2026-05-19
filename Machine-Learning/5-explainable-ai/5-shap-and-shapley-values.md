---
title: "5 - SHAP and Shapley Values"
created: 2026-05-19
updated: 2026-05-19
tags: []
aliases: []
---

# 5 - SHAP and Shapley Values

[toc]

> **TL;DR:** SHAP (SHapley Additive exPlanations) unifies additive feature attribution methods — LIME, DeepLIFT, vanilla gradient, and others — under a single game-theoretic framework, and proves that Shapley values are the *unique* attribution satisfying three properties: local accuracy (Completeness), missingness, and consistency. Lundberg & Lee (2017) then derive computationally tractable approximations — Kernel SHAP (model-agnostic, unbiased), Deep SHAP (deep nets via DeepLIFT), and Tree SHAP ($O(TLD^2)$ exact for tree ensembles) — that make the theoretically ideal attribution practically deployable.

## Vocabulary

**SHAP value** ($\phi_i$): The Shapley value assigned to feature $i$ in the SHAP attribution framework — the marginal contribution of feature $i$ averaged over all possible subsets of other features. The unique solution satisfying local accuracy, missingness, and consistency.

```math
\phi_i = \sum_{S \subseteq F \setminus \{i\}} \frac{|S|!\,(|F|-|S|-1)!}{|F|!} \left[f_{S \cup \{i\}}(x_{S \cup \{i\}}) - f_S(x_S)\right]
```

---

**Additive feature attribution model** ($g$): SHAP's unified model class. An explanation function $g(z') = \phi_0 + \sum_{i=1}^{M} \phi_i z'_i$ where $z' \in \{0,1\}^M$ is the simplified binary input (coalition) and $\phi_i \in \mathbb{R}$ are the attribution weights.

---

**Local accuracy (Completeness)**: The first SHAP property — the explanation model matches the original model at $x$:

```math
f(x) = g(x') = \phi_0 + \sum_{i=1}^{M} \phi_i
```

Ensures attributions are calibrated to the actual output value, not a relative scale.

---

**Missingness**: The second SHAP property — features absent from the input ($x'_i = 0$) receive zero attribution ($\phi_i = 0$). Prevents credit assigned to features that aren't present.

---

**Consistency**: The third SHAP property — if a feature's marginal contribution to a model's output increases when that model is replaced by a different model, its SHAP value must not decrease. Ensures attributions are monotone with respect to model changes; SHAP proves this holds for Shapley values and that *no other* additive attribution satisfies all three simultaneously.

---

**Kernel SHAP**: A model-agnostic SHAP approximation that uses a specially designed LIME-like sampling procedure with a specific kernel weight that makes the LIME solution exactly recover Shapley values in expectation.

---

**SHAP kernel weight**: The specific proximity kernel that makes Kernel SHAP unbiased:

```math
\pi_{x'}(z') = \frac{(M-1)}{\binom{M}{|z'|}|z'|(M-|z'|)}
```

where $|z'|$ is the number of non-zero entries in the coalition $z'$, and $M$ is the total number of features.

---

**Deep SHAP**: Combines SHAP values with DeepLIFT's backpropagation rules. Uses a set of background reference samples to estimate SHAP values; each reference produces a DeepLIFT attribution and these are averaged.

---

**Tree SHAP**: An exact, polynomial-time algorithm for computing SHAP values for tree ensembles (gradient boosting, random forests). Exploits the tree structure to enumerate all feature subsets efficiently in $O(TLD^2)$ where $T$ = number of trees, $L$ = max leaves, $D$ = max depth.

---

**Expected value baseline** ($\phi_0$): In SHAP, $\phi_0 = \mathbb{E}[f(x)]$ — the expected model output over the background dataset. The Completeness property guarantees that feature attributions sum to $f(x) - \phi_0$.

---

**Coalition** ($S \subseteq F$): A subset of features. Shapley values measure the marginal contribution of feature $i$ by averaging over all possible coalitions $S$ that don't include $i$.

---

**Background dataset**: The reference distribution in SHAP from which the expected value $\phi_0$ is computed and missing features are marginalized. Typically a random sample of 50–200 training examples.

---

**Interaction effect**: The synergistic contribution of two features $(i, j)$ jointly, beyond their individual SHAP values. Captured by SHAP interaction values (Lundberg et al., 2018) — a tensor of pairwise Shapley interactions.

---

## Intuition

Imagine a three-player game where players A, B, and C collectively earn a reward, but contribute unequally. How should the reward be divided fairly? Shapley (1953) solved this for cooperative game theory: assign each player the average of their marginal contribution across all possible orderings in which players join the coalition. This satisfies Efficiency (total payout = total reward), Symmetry (equally contributing players get equal shares), Dummy (no-contribution players get zero), and Additivity (fair over sums of games).

In ML attribution, the "players" are input features, the "game" is the model's output, and the "reward" is $f(x) - \mathbb{E}[f(x)]$ — the excess prediction above average. The Shapley value for feature $i$ is its average marginal contribution to the model's output, averaged over all possible subsets of the other features.

Lundberg & Lee's key contribution is proving that Shapley values are the *unique* additive attribution that satisfies the three properties they formalize as SHAP properties. This is not just a game-theoretic nicety — it means any other additive attribution (LIME's LASSO coefficients, DeepLIFT scores, vanilla gradient) is either violating one of these properties or, in a specific limit, approximating Shapley values.

## How it works

### Exact Shapley computation

The naive exact Shapley computation enumerates all $2^M$ subsets of features. For each subset $S$ not containing feature $i$, the method computes the marginal contribution of adding $i$ to $S$: $f_{S \cup \{i\}}(x_{S \cup \{i\}}) - f_S(x_S)$. The marginal contribution is averaged with the combinatorial weight that gives each ordering equal probability.

This is intractable for $M > 20$ features. The SHAP paper introduces three tractable approximations for different model types.

### Kernel SHAP: model-agnostic approximation

Kernel SHAP observes that if the LIME objective is solved with the specific kernel weight $\pi_{x'}(z')$, the solution converges to Shapley values as the number of samples grows. The trick is the kernel weight, which assigns high weight to coalitions of very few or very many features (where Shapley values have most influence) and downweights medium-sized coalitions.

```python
import shap
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from sklearn.datasets import load_breast_cancer

# Load tabular dataset and train an RF classifier
data = load_breast_cancer()
X, y = data.data, data.target
feature_names = data.feature_names

clf = RandomForestClassifier(n_estimators=100, random_state=42)
clf.fit(X, y)

# Kernel SHAP: model-agnostic, uses the RF's predict_proba as a black box
# Background dataset: first 50 training samples for marginal expectation
background = X[:50]
explainer = shap.KernelExplainer(clf.predict_proba, background)

# Explain a single test instance
x_test = X[100:101]                  # shape [1, 30]
shap_values = explainer.shap_values(
    x_test, nsamples=500             # 500 coalition samples per feature
)
# shap_values is a list of [1, 30] arrays, one per class
# shap_values[1] = attributions for class 1 (malignant)
print("Sum of SHAP values (class 1):", shap_values[1].sum())
print("Expected output (class 1):", explainer.expected_value[1])
print("Model output (class 1):", clf.predict_proba(x_test)[0, 1])
# Completeness: shap_values[1].sum() + expected_value[1] ≈ model output
```

> [!IMPORTANT]
> Kernel SHAP's `nsamples` parameter controls convergence, not model queries. Each "sample" generates one coalition vector, which requires one model query per coalition. For $M=30$ features and `nsamples=500`, that is 500 model forward passes per explanation. For `nsamples=2000` (recommended for production), it is 2000 passes. Plan compute budgets accordingly.

### Tree SHAP: exact polynomial-time for tree ensembles

Tree SHAP avoids sampling entirely by exploiting the decision-tree structure. Each tree defines a partition of feature space, and the SHAP value for feature $i$ at input $x$ can be computed by traversing the tree and tracking, at each split node, the fraction of training data that each child receives. This allows exact marginalization over all coalitions using dynamic programming on the tree structure.

```python
import shap
import numpy as np
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.datasets import load_breast_cancer

data = load_breast_cancer()
X, y = data.data, data.target

gbc = GradientBoostingClassifier(n_estimators=100, max_depth=4, random_state=42)
gbc.fit(X, y)

# Tree SHAP: exact, O(T * L * D^2) per sample
explainer = shap.TreeExplainer(gbc)
shap_values = explainer.shap_values(X[:10])   # shape [10, 30]

# Verify Completeness on first sample
expected = explainer.expected_value
print(f"Expected value: {expected:.4f}")
print(f"SHAP sum:       {shap_values[0].sum():.4f}")
print(f"Model output:   {gbc.predict_proba(X[:1])[0, 1]:.4f}")
# expected + shap_values[0].sum() should ≈ model output

# Global summary: beeswarm plot (feature importance × direction)
shap.summary_plot(shap_values, X[:100], feature_names=data.feature_names)
```

> [!TIP]
> Tree SHAP's `TreeExplainer` in the `shap` library automatically selects the exact algorithm for scikit-learn GradientBoostingClassifier, XGBoost, LightGBM, and CatBoost. For random forests, it uses an approximation by default; pass `check_additivity=True` to verify Completeness on each sample and detect any numerical drift.

### Deep SHAP: attribution for neural networks

Deep SHAP extends DeepLIFT to compute SHAP values by running DeepLIFT with multiple reference samples from the background distribution and averaging the resulting attributions. The DeepLIFT multipliers (which approximate integrated gradients on piecewise-linear networks) serve as per-reference attributions; averaging over the background distribution marginalizes out the reference dependence.

```python
import torch
import shap
from torchvision import models, transforms
from PIL import Image
import numpy as np

model = models.resnet50(weights=models.ResNet50_Weights.IMAGENET1K_V2)
model.eval()

preprocess = transforms.Compose([
    transforms.Resize(224), transforms.CenterCrop(224), transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
])

# Background: 20 random training images (for marginalization)
background = torch.stack([
    preprocess(Image.fromarray(np.random.randint(0, 255, (224, 224, 3), dtype=np.uint8)))
    for _ in range(20)
])  # [20, 3, 224, 224]

test_img = preprocess(Image.open("cat.jpg").convert("RGB")).unsqueeze(0)  # [1, 3, 224, 224]

# Deep SHAP — wraps DeepLIFT internally
explainer = shap.DeepExplainer(model, background)
shap_values = explainer.shap_values(test_img)   # list of [1, 3, 224, 224] per class
# Attribution for class 281 (tabby cat)
attr = shap_values[281]   # [1, 3, 224, 224]
```

![SHAP beeswarm plot — feature attributions sorted by global importance with per-sample direction and magnitude shown.](./assets/5-shap-and-shapley-values/shap-003.png)

## Math

The Shapley value formula expanded for the additive attribution model:

```math
\phi_i(f, x) = \sum_{S \subseteq F \setminus \{i\}} \frac{|S|!\,(M - |S| - 1)!}{M!} \left[ f\!\left(x_{S \cup \{i\}}\right) - f\!\left(x_S\right) \right]
```

where $M = |F|$ is the total number of features, and $f(x_S)$ denotes the model output when only the features in $S$ are "present" (others marginalized out using $\mathbb{E}[f(x) \mid x_S]$).

The Completeness (local accuracy) property is a direct consequence of the Efficiency axiom:

```math
\sum_{i=1}^{M} \phi_i = f(x) - \mathbb{E}[f(x)]
```

The Kernel SHAP weighted least squares is:

```math
\phi = \arg\min_{w \in \mathbb{R}^M} \sum_{z' \in \{0,1\}^M} \pi_{x'}(z') \left[ f(h_x(z')) - \phi_0 - \sum_{i=1}^{M} w_i z'_i \right]^2
```

subject to $\sum_{i} w_i = f(x) - \phi_0$ (Completeness constraint). The Kernel SHAP weight is derived by matching the LIME solution to the Shapley expectation:

```math
\pi_{x'}(z') = \frac{(M-1)}{\dbinom{M}{|z'|} \cdot |z'| \cdot (M - |z'|)}
```

For $|z'| = 0$ or $|z'| = M$ the weight is infinite — these boundary coalitions (empty set and full set) are always included and handled separately.

## Real-world example

The canonical use case for SHAP in production is gradient boosting model audit for credit scoring. SHAP values let compliance officers explain individual loan decisions in the format required by GDPR Article 22 ("right to explanation").

```python
import shap
import pandas as pd
import numpy as np
from xgboost import XGBClassifier
from sklearn.model_selection import train_test_split

# Synthetic credit scoring dataset
np.random.seed(42)
n = 5000
df = pd.DataFrame({
    "age":              np.random.randint(20, 70, n),
    "income":           np.random.exponential(50000, n),
    "debt_ratio":       np.random.beta(2, 5, n),
    "credit_history":   np.random.randint(0, 10, n),
    "num_accounts":     np.random.poisson(3, n),
    "late_payments":    np.random.poisson(0.5, n),
    "loan_amount":      np.random.exponential(10000, n),
})
# Synthetic label: higher debt_ratio + late_payments → default
logit = (2 * df.debt_ratio + 0.3 * df.late_payments
         - 0.01 * df.credit_history - 0.5)
prob = 1 / (1 + np.exp(-logit))
df["default"] = (np.random.uniform(size=n) < prob).astype(int)

X = df.drop("default", axis=1)
y = df["default"]
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# Train XGBoost
clf = XGBClassifier(n_estimators=200, max_depth=4, learning_rate=0.05,
                    use_label_encoder=False, eval_metric="logloss", random_state=42)
clf.fit(X_train, y_train)

# Tree SHAP: exact SHAP values in milliseconds per sample
explainer = shap.TreeExplainer(clf)
shap_values = explainer.shap_values(X_test)  # [n_test, 7]

# Individual explanation: applicant 0
applicant_idx = 0
shap_dict = dict(zip(X.columns, shap_values[applicant_idx]))
print("\nSHAP values for applicant 0 (positive = increases default risk):")
for feat, val in sorted(shap_dict.items(), key=lambda kv: -abs(kv[1])):
    direction = "↑" if val > 0 else "↓"
    print(f"  {feat:20s}: {val:+.4f}  {direction}")
print(f"  {'TOTAL (excess)':20s}: {sum(shap_dict.values()):+.4f}")
print(f"  {'Base rate':20s}: {explainer.expected_value:.4f}")
print(f"  {'Model P(default)':20s}: {clf.predict_proba(X_test.iloc[[applicant_idx]])[0,1]:.4f}")
```

> [!WARNING]
> SHAP values computed with a background sample that differs from the true training distribution give biased attributions. A common mistake is using the test set as the background for `TreeExplainer`. Always use a representative sample from the *training* distribution. For imbalanced datasets, stratify the background sample to match class proportions.

## In practice

**Tree SHAP is the only method that computes exact Shapley values in polynomial time.** For gradient boosting (XGBoost, LightGBM), Tree SHAP runs in milliseconds per sample and is suitable for real-time explanation serving. For neural networks, Deep SHAP or Kernel SHAP are approximate and much slower (seconds per sample).

**Kernel SHAP with too few samples gives systematically overconfident attributions.** The Kernel SHAP estimator is unbiased but has variance that decreases slowly with sample count. For $M > 50$ features, convergence to accurate Shapley values requires $\gg M^2$ samples. Published SHAP papers often use 2000–5000 samples; anything below 500 should be treated as exploratory only.

**SHAP values depend on the background distribution in ways that practitioners often ignore.** The $\phi_0 = \mathbb{E}[f(x)]$ is computed over the background sample. A background of only "high-risk" patients produces different $\phi_i$ than a representative population background, even for the same model and same patient $x$. The attribution answers "how does this patient differ from the background?" — the answer depends entirely on what "background" means.

> [!TIP]
> For global feature importance (not per-sample), use `shap.summary_plot` with `plot_type="bar"` to show mean $|\phi_i|$ across the test set. This is more honest than standard feature importance from GINI/split-count because it accounts for feature interaction effects and satisfies Completeness.

**SHAP interaction values expose higher-order effects that first-order attributions miss.** If features $i$ and $j$ interact (their joint effect differs from the sum of their individual effects), the first-order SHAP values capture this by distributing the interaction credit between $i$ and $j$ symmetrically. The interaction values (Lundberg et al., 2018) make this explicit at additional computational cost.

**Deep SHAP's accuracy depends on how well DeepLIFT approximates the IG path integral.** For networks with smooth nonlinearities (GELU, SiLU) or complex normalization (LayerNorm, RMSNorm), DeepLIFT's multiplier computation is less accurate, and Deep SHAP attributions diverge from true Shapley values. Use Kernel SHAP on neural networks when faithfulness is critical.

## Pitfalls

- **Wrong belief: SHAP values measure causal feature importance.** Correction: SHAP measures correlational contribution under the model's learned function. Features correlated with the target but causally irrelevant (e.g., a spurious background texture) receive nonzero SHAP values. For causal attributions, use interventional SHAP (Janzing et al., 2020), which conditions on the data-generating process rather than the model.
- **Wrong belief: Summing SHAP values across samples gives a valid global explanation.** Correction: SHAP values are locally meaningful (they sum to $f(x) - \mathbb{E}[f(x)]$ at each $x$), but their absolute values depend on the magnitude of the model output at each sample. Mean $|\phi_i|$ is a valid global importance measure; mean $\phi_i$ is not (signs cancel).
- **Wrong belief: Kernel SHAP is slow only because of the model queries.** Correction: Part of the cost is the weighted regression, which costs $O(d^3)$ in feature dimensionality $d$. For $d > 200$ features, the regression itself can be a bottleneck. Use independent feature approximation (which drops the WLS regression) for high-dimensional inputs at the cost of ignoring feature correlations.
- **Wrong belief: Deep SHAP is exact.** Correction: Deep SHAP is an approximation of SHAP values via DeepLIFT. It can fail on architectures with global pooling, batch normalization, or attention mechanisms because DeepLIFT's multiplier rules are not well-defined for these operations.
- **Wrong belief: SHAP values prove the model is fair.** Correction: A SHAP audit can detect that race or gender is a top-contributing feature, but nonzero SHAP values for a protected attribute prove neither discrimination nor fairness. Proxy variables (zip code, surname) can carry the same causal signal through features with lower SHAP values. SHAP is a forensic tool, not a legal compliance test.

## Open questions

- **Causal SHAP**: Interventional vs. observational conditioning for feature marginalization remains an active area. Janzing et al. (2020) and Heskes et al. (2020) propose interventional and asymmetric Shapley values; no single standard has emerged.
- **SHAP for LLMs**: Computing SHAP values for token-level attribution in transformers with $M = 32{,}768$ context length is computationally prohibitive even with Tree/Deep SHAP. Approximate methods (SHAP on attention, on logit lens activations) are active research.
- **Stability under distribution shift**: SHAP values change when the background distribution changes. Designing SHAP-based monitoring systems that remain informative under gradual covariate shift is unsolved.

## Sources

- Lundberg, S. M. & Lee, S.-I. (2017). *A Unified Approach to Interpreting Model Predictions.* arXiv:1705.07874.
- Lundberg, S. M. et al. (2018). *Consistent Individualized Feature Attribution for Tree Ensembles.* arXiv:1802.03888 (Tree SHAP).
- Shapley, L. S. (1953). *A Value for n-person Games.* Contributions to the Theory of Games, Vol. 2.
- Janzing, D. et al. (2020). *Feature relevance quantification in explainable AI: A causal problem.* arXiv:1910.13413.
- Conversation with user on 2026-05-19.

## Related

- [[1-why-explainability-matters]]
- [[3-attribution-and-gradient-methods]]
- [[4-perturbation-and-sampling-methods]]
- [[6-counterfactuals-and-sanity-checks]]
