---
title: "1 - Why Explainability Matters"
created: 2026-05-19
updated: 2026-05-19
tags: []
aliases: []
---

# 1 - Why Explainability Matters

[toc]

> **TL;DR:** Interpretability in ML is not a cosmetic feature — it is a precondition for safe deployment when our formalization of the real-world objective is incomplete. Doshi-Velez & Kim (2017) formalize this "incompleteness argument" and propose a three-tier evaluation hierarchy; Gilpin et al. (2018) independently categorize the entire space of explanation methods into processing, representation, and explanation-producing systems. Together these two papers form the theoretical bedrock for the field.

## Vocabulary

**Interpretability**: The degree to which a human can understand the cause of a model's decision. Distinct from accuracy — a model can be perfectly accurate on a test set and still be uninterpretable.

---

**Explainability**: Often used interchangeably with interpretability, but Gilpin et al. draw a subtle distinction: an *explainable* system can summarize its reasoning in terms a human can engage with, whereas an *interpretable* model exposes its internal mechanics directly. In this series of notes the two terms are used interchangeably unless otherwise stated.

---

**Incompleteness argument**: The core justification for requiring interpretability. When the real-world objective cannot be perfectly specified as a loss function (cost mis-specification, distributional shift, proxies for unmeasurable quantities), a human must retain the ability to audit and correct the system. Doshi-Velez & Kim argue this condition holds for essentially all safety-critical applications.

---

**Application-grounded evaluation**: Testing explanations with domain experts on the real task (e.g., radiologists judging whether a saliency map would help them catch a misdiagnosis). Gold standard; expensive and slow.

---

**Human-grounded evaluation**: Testing with lay humans on a simplified proxy task (e.g., forward-simulation tests: "given this explanation, predict the model's output on this new input"). Cheaper than application-grounded; still measures a real human cognitive signal.

---

**Functionally-grounded evaluation**: Testing without any humans — using a proxy definition of "good explanation" (e.g., depth of the decision tree, or whether the explanation satisfies an axiomatic desideratum). Fastest; most disconnected from true utility.

---

**Processing explanation**: A system that makes a black-box model more understandable by describing *what it does* — e.g., extracting intermediate representations or distilling it into a simpler surrogate. (Gilpin et al. taxonomy, Type 1.)

---

**Representation explanation**: A method that exposes *what the model has learned* — embedding visualizations, probing classifiers, concept activation vectors. (Gilpin et al. taxonomy, Type 2.)

---

**Explanation-producing system**: A method that generates *natural-language or visual artifacts* as its primary output — e.g., attention overlays, saliency maps, textual rationales. (Gilpin et al. taxonomy, Type 3.)

---

**Transparency vs. accuracy tradeoff**: The empirical observation that more interpretable model families (linear models, shallow decision trees) typically underperform deep neural networks on complex tasks. Not a mathematical law — FlashAttention-equipped transformers with mechanistic interpretability tools are beginning to challenge this.

---

**Stakeholder diversity**: Different users of explanations have fundamentally different requirements. A regulator needs audit trails; a domain expert needs actionable guidance; an ML engineer needs debugging signal; an end user needs trust calibration. One explanation format rarely serves all four.

---

**Proxy task**: A simpler stand-in for the real objective used to evaluate explanations without running a full application-grounded study. Common proxy tasks: forward simulation, binary choice between real and counterfactual, and pointing game (does the saliency peak land on the correct object?).

---

**Simplicity prior**: The assumption, common in early XAI work, that shorter/sparser explanations are better. Valid as a cognitive load heuristic but can conflict with faithfulness — a two-feature LIME approximation may be easier to read but badly misrepresents a 512-dimensional feature interaction.

---

## Intuition

Think of model interpretability as the analog of a medical audit trail. A doctor's treatment decision is explainable not because we can read their neurons, but because they can give a structured account of which symptoms drove which conclusions — an account we can check against medical knowledge, challenge with new evidence, and use to detect bias or error. Deep neural networks currently lack this property: they produce answers without accounts. The field of XAI is building those accounts, and the central question is whether the accounts are *faithful* (they reflect what the model actually did) or merely *plausible* (they sound reasonable to a human but are post-hoc rationalizations).

The incompleteness argument makes this concrete. In any real deployment, the loss function is a proxy: we measure cross-entropy on labeled data, not directly whether the model will generalize to the distribution shift induced by a new hospital, a new camera, or an adversary. When the proxy diverges from the true objective, accuracy is not enough — we need a way to inspect whether the model has learned the right *generalization*, not just the right *statistics* of the training set.

## How it works

The "how" of *why we need* explainability is a philosophical and empirical argument, not an algorithm. This section traces the logical structure of that argument, and maps it to the evaluation hierarchy that Doshi-Velez & Kim propose as the operational framework for measuring progress.

### The incompleteness argument in full

The argument has four premises: (1) we cannot perfectly formalize our true objective; (2) when formalization is imperfect, a model can optimize the stated loss while violating the unstated desiderata; (3) the only way to catch and correct such violations is to inspect the model's reasoning; and (4) inspection requires interpretability. The conclusion is that interpretability is *not* a luxury for post-hoc communication — it is a *safety precondition* for any deployment where consequences matter.

Doshi-Velez & Kim list four concrete situations where incompleteness manifests: (a) the cost of errors is asymmetric and unknown, (b) the system will be deployed on a distribution different from training, (c) the system interacts with social or legal systems that embed ethical norms not captured in any loss, and (d) the deployment requires ongoing human oversight or correction. All four characterize the majority of ML deployments in medicine, criminal justice, finance, and autonomous systems.

### The three evaluation tiers

Doshi-Velez & Kim organize evaluation into three tiers based on cost and validity. The figure below maps each tier to its defining property.

**Figure:** Three-tier evaluation hierarchy for interpretability methods (Doshi-Velez & Kim, 2017).

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Tier                    │  Who evaluates    │  Task             │ Cost  │
├─────────────────────────────────────────────────────────────────────────┤
│  Application-grounded    │  Domain experts   │  Real task        │ High  │
│  Human-grounded          │  Lay humans       │  Simplified proxy │ Med   │
│  Functionally-grounded   │  Automated        │  Axiomatic proxy  │ Low   │
└─────────────────────────────────────────────────────────────────────────┘
```

The tiers are not alternatives — they form a validation chain. A new explanation method should first pass functionally-grounded unit tests (does it satisfy the axioms it claims?), then graduate to human-grounded studies (does it improve human task performance on a proxy?), and ultimately be validated application-grounded (does it improve outcomes in the real deployment context?). Most published papers stop at tier 1 or 2; tier 3 evaluations remain rare.

> [!IMPORTANT]
> Functionally-grounded metrics (e.g., "the saliency map covers the correct bounding box") are necessary but not sufficient. They can be satisfied by an explanation that looks visually appealing but bears no causal relationship to the model's actual computation. Always trace the validation chain upward toward application-grounded evidence.

### Gilpin et al. taxonomy

Gilpin et al. approach the same problem from the system-design perspective rather than the evaluation perspective. They observe that explanations can arise at three points in the ML pipeline:

1. **Processing explanation** — modify or probe the model to reveal its internal computation. Examples: attention weight visualization, probing classifiers on hidden states, concept activation vectors (CAVs). The model itself is the subject.
2. **Representation explanation** — analyze the learned feature space. Examples: t-SNE/UMAP of embeddings, activation atlas, feature inversion. The representations learned by the model are the subject.
3. **Explanation-producing system** — build a separate module whose output is an explanation artifact consumed by a human. Examples: LIME, SHAP, saliency maps, textual rationale generators. The explanation artifact is the product.

The taxonomy matters for engineering: a processing explanation can be invalidated by an architectural change (e.g., adding dropout changes which attention heads are active); a representation explanation requires access to intermediate activations; an explanation-producing system can be attached to any black box but risks faithfulness failures.

> [!NOTE]
> Gilpin et al. note that the most practically valuable explanations are often *hybrid* — they combine processing analysis (e.g., which layer activates) with an explanation-producing output (e.g., a saliency overlay). GRAD-CAM is the canonical example of this hybrid design.

### Stakeholder requirements

Different stakeholders impose different constraints on what counts as a "good" explanation. A radiologist interpreting a chest X-ray classification needs high spatial precision (which pixels drove the decision) and clinical vocabulary (the explanation should map to anatomical structures). A compliance officer needs a decision log that can be audited after the fact without re-running the model. An ML researcher needs a signal that is faithful enough to diagnose failure modes (memorization vs. generalization). An end user needs a summary that supports trust calibration (when should I trust this system and when should I override it?).

These requirements are often in tension. High-fidelity gradient-based attributions satisfy the researcher but are unreadable to the radiologist. Simple linear surrogates (LIME) are readable to the radiologist but may be unfaithful to the model's actual computation. Stakeholder analysis should precede method selection.

![Gilpin et al. taxonomy figure — processing, representation, and explanation-producing systems.](./assets/1-why-explainability-matters/gilpin-003.png)

## Math

The incompleteness argument can be stated precisely. Let the true objective be a functional $U: \mathcal{H} \to \mathbb{R}$ over the hypothesis class, and let the training objective be the empirical risk $\hat{R}(h) = \frac{1}{n}\sum_{i=1}^{n} \ell(h(x_i), y_i)$. The gap is:

```math
U(h^*) - U(\hat{h}) \quad \text{where} \quad \hat{h} = \arg\min_h \hat{R}(h)
```

When this gap is nonzero due to mis-specification, no amount of test-set accuracy improvement closes it. Interpretability is the mechanism by which humans detect and reduce this gap by inspecting $\hat{h}$'s behavior outside the training distribution.

For evaluation, Doshi-Velez & Kim formalize the human-grounded forward simulation task: given an explanation $e = E(h, x)$ and a new input $x'$, measure whether a human can predict $h(x')$ more accurately with $e$ than without it. The metric is:

```math
\Delta_{\text{acc}} = \Pr[\hat{y}_{\text{human}} = h(x') \mid e] - \Pr[\hat{y}_{\text{human}} = h(x') \mid \varnothing]
```

A positive $\Delta_{\text{acc}}$ means the explanation carries genuine information about the model's behavior — it is not merely decorative.

## Real-world example

A canonical deployment failure illustrates the incompleteness argument in action. Consider a skin lesion classifier trained on a dermatology dataset where cancerous lesions frequently appear alongside surgical rulers (placed for scale during professional photography). The model achieves 95% accuracy on held-out data — but when deployed in primary care settings where rulers are absent, accuracy drops to 78%. A saliency-map inspection of the training-distribution predictions reveals that the ruler edge is one of the top-attributed features for the "melanoma" class. The loss function encoded "label agreement on training images" faithfully; it failed to encode "generalize to ruler-absent images."

```python
import torch
import torch.nn.functional as F
from torchvision import models, transforms
from PIL import Image
import numpy as np
import matplotlib.pyplot as plt

def vanilla_gradient_saliency(
    model: torch.nn.Module,
    image_tensor: torch.Tensor,   # shape: [1, 3, H, W]
    target_class: int,
) -> np.ndarray:
    """
    Compute vanilla gradient saliency map for a single image.
    Returns absolute-value saliency summed over colour channels, shape [H, W].
    """
    model.eval()
    x = image_tensor.clone().requires_grad_(True)
    logits = model(x)                        # [1, num_classes]
    score = logits[0, target_class]
    model.zero_grad()
    score.backward()
    # grad shape: [1, 3, H, W] → collapse channels
    saliency = x.grad.data.abs().squeeze(0).max(dim=0).values
    return saliency.cpu().numpy()

# --- usage ---
model = models.resnet50(weights=models.ResNet50_Weights.IMAGENET1K_V2)
preprocess = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406],
                         std=[0.229, 0.224, 0.225]),
])

img = Image.open("lesion_with_ruler.jpg").convert("RGB")
x = preprocess(img).unsqueeze(0)            # [1, 3, 224, 224]
sal = vanilla_gradient_saliency(model, x, target_class=207)

plt.imshow(sal, cmap="hot")
plt.title("Saliency: high activation → ruler edge detected")
plt.colorbar()
plt.savefig("ruler_saliency.png", dpi=150, bbox_inches="tight")
```

> [!WARNING]
> Vanilla gradient saliency is a fast diagnostic tool but is known to be noisy and sensitive to irrelevant input variation. It demonstrates the *existence* of a shortcut feature but should not be used alone as evidence that the model is safe. Use it alongside application-grounded evaluation (clinician review) and data-randomization sanity checks (see [[6-counterfactuals-and-sanity-checks]]).

## In practice

**Regulatory pressure is the primary driver of XAI adoption.** The EU AI Act (2024), GDPR Article 22, and the US FDA's predetermined change control plans for medical AI all create legal obligations for decision explanations. This means application-grounded evaluation is no longer optional for regulated deployments — it is a compliance requirement. Methods that score well on functionally-grounded benchmarks but have no human-grounded validation will face regulatory challenge.

**The field has not converged on a single faithfulness metric.** Dozens of proxy metrics exist (pointing game accuracy, insertion/deletion curves, infidelity, sensitivity). Different metrics rank the same method differently. Before comparing XAI papers, check which metric they report — a method that wins on pointing game may lose on insertion/deletion.

**Cost of explanation is often ignored.** LIME and SHAP require hundreds to thousands of model forward passes per example. At inference serving scale (100k requests/second), this can be a 100x latency multiplier. In practice, XAI is often deployed as an offline audit tool rather than a real-time feature, or approximated with faster gradient-based proxies.

> [!TIP]
> For latency-sensitive applications, gradient-based methods (vanilla gradient, Integrated Gradients with 50 steps) are 2–3 orders of magnitude faster than perturbation-based methods (LIME, RISE) per explanation. Reserve SHAP and LIME for post-hoc audits, not real-time explanations.

**Explanation faithfulness and human interpretability are orthogonal axes.** A gradient attribution map is faithful by construction (it measures exactly what the model responds to) but may be spatially noisy and hard for humans to interpret. A LIME linear model is easy to read but may poorly approximate the model in the region of interest. No current method achieves both simultaneously on all tasks.

## Pitfalls

- **Wrong belief: A high-accuracy model needs no explanation.** Correction: accuracy on the training distribution says nothing about behaviour under distribution shift, adversarial inputs, or in demographic subgroups. The skin lesion/ruler example is a documented real case.
- **Wrong belief: Explanations are for end users.** Correction: the highest-value consumers of explanations are often ML engineers and domain experts during model development, not end users at inference time. Stakeholder analysis precedes deployment.
- **Wrong belief: Any proxy for "good explanation" is adequate.** Correction: functionally-grounded metrics are unit tests, not validation. They can be gamed — a method can be explicitly optimized to score well on a pointing-game metric without being faithful to model computation.
- **Wrong belief: Interpretability reduces accuracy.** Correction: this is an empirical observation about model families, not a theorem. It does not apply when the same model is used and explanations are generated post-hoc. For intrinsically interpretable model families, the tradeoff is real but its magnitude is task-dependent.
- **Wrong belief: Attention weights are explanations.** Correction: Jain & Wallace (2019) show that attention weights are not causally linked to output in the sense required for explanation. They can be permuted without changing the output in many models. Attention is a *routing mechanism*, not an attribution.

## Open questions

- **Faithfulness vs. interpretability Pareto front**: Can we characterize which model architectures admit both faithful and human-readable explanations, and what the fundamental tradeoff looks like?
- **Distributional shift detection via XAI**: Can routine saliency monitoring at deployment time serve as an early-warning system for covariate shift, without labeled data from the new distribution?
- **Causal vs. correlational explanations**: Current methods (gradient, LIME, SHAP) identify correlations between features and outputs. The next generation of XAI work (e.g., FIDO, causal Shapley) attempts to identify *causal* relationships. The gap between correlation and causation in explanations is an open and active research area.
- **Multi-modal explanations**: As VLMs (CLIP, GPT-4V, Gemini) become deployment targets, explanations must span text and image jointly. The evaluation frameworks developed for unimodal classifiers do not transfer cleanly.

## Sources

- Doshi-Velez, F. & Kim, B. (2017). *Towards a rigorous science of interpretable machine learning.* arXiv:1702.08608.
- Gilpin, L. H., Bau, D., Yuan, B. Z., Bajwa, A., Specter, M., & Kagal, L. (2018). *Explaining Explanations: An Overview of Interpretability of Machine Learning.* arXiv:1806.00069.
- Jain, S. & Wallace, B. C. (2019). *Attention is not Explanation.* arXiv:1902.10186.
- EU AI Act (2024). Regulation (EU) 2024/1689. https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32024R1689
- Conversation with user on 2026-05-19.

## Related

- [[2-feature-visualization]]
- [[3-attribution-and-gradient-methods]]
- [[4-perturbation-and-sampling-methods]]
- [[5-shap-and-shapley-values]]
- [[6-counterfactuals-and-sanity-checks]]
