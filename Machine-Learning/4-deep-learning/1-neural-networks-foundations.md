---
title: "1 - Neural Networks Foundations"
created: 2026-05-19
updated: 2026-05-19
tags: []
aliases: []
---

# 1 - Neural Networks Foundations

[toc]

> **TL;DR:** A feedforward neural network is a directed acyclic graph of parameterised affine-plus-nonlinearity modules, composing them to produce a function arbitrarily complex enough to approximate any continuous map (the Universal Approximation Theorem). It replaces hand-crafted features with learned representations: from perceptrons that can only separate linearly separable classes, through single-hidden-layer networks that solve XOR, to deep multilayer perceptrons (MLPs) that factorize a highly varying function into a hierarchy of progressively more abstract representations.

---

## Vocabulary

- **Perceptron** — Rosenblatt (1959) binary classifier: output = sign(wᵀx + b). Linearly separable inputs only.
- **Multilayer Perceptron (MLP)** — a feedforward network with ≥ 1 hidden layers of nonlinear units.
- **Pre-activation (u)** — the raw linear combination at layer ℓ: u = Wh + b, before the nonlinearity is applied.
- **Post-activation (h)** — u after applying elementwise activation σ: h = σ(u).
- **Activation function (σ)** — elementwise nonlinearity. Classical: sigmoid, tanh. Modern default: ReLU = max(0, x).
- **Softmax** — output-layer normalisation for C-class classification: ŷ_c = exp(a_c) / Σ_k exp(a_k), ensuring Σ ŷ_c = 1.
- **Cross-entropy loss** — supervised classification objective: L = −Σ_c y_c log ŷ_c. Minimising it is equivalent to maximising log-likelihood under a categorical model.
- **MSE loss** — regression objective: L = (1/N) Σ (y − ŷ)². The MLE objective for Gaussian noise.
- **Universal Approximation Theorem (UAT)** — a single hidden layer of sufficient width with any non-polynomial activation can approximate any continuous function on a compact domain to arbitrary precision.
- **Depth** (of architecture) — the number of composed nonlinear layers. Bengio (2009): deeper architectures can represent highly varying functions exponentially more compactly than shallow ones.
- **Representational power** — the set of functions a given architecture can express; capacity increases with both width and depth.
- **Distributed representation** — each neuron participates in representing multiple concepts; each concept is represented by the joint activation of many neurons. Contrasts with local (grandmother cell) representation.
- **Weight sharing** — multiple units use the same weight vector (generalised in ConvNets); reduces parameter count and injects inductive bias.
- **Bias unit** — constant input of 1 to each layer, allowing the affine offset b in the linear layer.

---

## Intuition

A linear classifier carves input space with a single hyperplane. Composing a linear map with a pointwise nonlinearity bends and folds that hyperplane: two composed layers can solve XOR (which lies in two half-spaces, not one). Each additional hidden layer folds the input space one more time, allowing the final output layer to draw arbitrarily complex decision boundaries with simple linear thresholds. The UAT tells us one layer of sufficient width is enough in theory — but depth is more *efficient*: representing a highly varying function shallowly can require exponentially more units than representing it with a deep hierarchy where lower layers detect edges, middle layers detect parts, and upper layers detect objects.

The biological analogy (de Freitas lectures, UvA Gavves): visual cortex has staged processing — V1 detects oriented edges at specific positions, V2 combines them into curves, IT cortex responds to entire faces. Deep networks replicate this inductive structure without hard-coding it, learning the hierarchy from data.

> [!NOTE]
> The UAT guarantees expressibility, not learnability. A width-1000 single-hidden-layer network can represent any function on paper, but gradient descent may not find the weights. Depth provides an implicit curriculum: lower layers learn reusable primitives that higher layers combine.

---

## How it works

A feedforward pass sends an input x through L layers, alternating between affine transformations and elementwise nonlinearities, to produce a prediction ŷ. The network is then trained by minimising a loss ℒ(ŷ, y) with stochastic gradient descent (SGD), where gradients are computed by backpropagation (see note 2).

### Step 1 — Architecture specification

Each layer ℓ ∈ {1, …, L} is specified by a weight matrix W^ℓ ∈ ℝ^{d_ℓ × d_{ℓ−1}}, a bias b^ℓ ∈ ℝ^{d_ℓ}, and an activation σ^ℓ. The input layer is h^0 = x. The output layer typically uses a task-specific activation (softmax for classification, identity for regression).

### Step 2 — Forward pass

The forward pass is a sequential left-to-right evaluation. At each layer the pre-activation and post-activation are computed and stored (they are needed during the backward pass). The computation at layer ℓ is:

```math
u^ℓ = W^ℓ h^{ℓ−1} + b^ℓ
h^ℓ = σ^ℓ(u^ℓ)
```

For multiclass classification with C classes, the final layer uses the softmax function to convert raw scores (logits) into a probability distribution over classes:

```math
ŷ_c = softmax(u^L)_c = exp(u^L_c) / Σ_{k=1}^{C} exp(u^L_k)
```

### Step 3 — Loss computation

The scalar loss value quantifies the prediction error. For classification with one-hot target y, the cross-entropy loss is the negative log-probability assigned to the correct class:

```math
ℒ = − Σ_{c=1}^{C} y_c log ŷ_c = −log ŷ_{y_true}
```

For regression with continuous target y, the mean-squared-error loss corresponds to a Gaussian noise assumption on the outputs:

```math
ℒ = (1/2) ||y − ŷ||²
```

### Step 4 — Parameter update

Weights are updated by SGD (or Adam / RMSProp in practice). The gradient ∂ℒ/∂θ is computed by backpropagation. The parameter update rule is:

```math
θ ← θ − η · ∇_θ ℒ
```

where η is the learning rate and ∇_θ ℒ is the gradient of the mini-batch loss. The full backpropagation algorithm is the subject of note 2.

### Activation functions

The choice of activation function governs the gradient flow and the representational geometry. Understanding the strengths and weaknesses of each is critical for deep architectures.

```math
sigmoid: σ(x) = 1 / (1 + exp(−x)),  range (0,1),  saturates for |x| ≫ 0
tanh:    σ(x) = (exp(x) − exp(−x)) / (exp(x) + exp(−x)),  range (−1,1),  zero-centred but still saturates
ReLU:    σ(x) = max(0, x),  no saturation for x > 0,  gradient 1 or 0 (dead-neuron risk for x < 0)
```

> [!IMPORTANT]
> Stacking affine layers without a nonlinearity collapses to a single affine map: W_L W_{L−1} … W_1 x + bias. No depth of purely linear layers adds representational power. The activation function is the essential ingredient that makes depth useful.

---

## Math

### Universal Approximation

The classical UAT (Cybenko 1989, Hornik 1991) states: for any continuous function f: [0,1]^n → ℝ and any ε > 0, there exists a single-hidden-layer network g with a sufficient number of sigmoid units such that max_x |f(x) − g(x)| < ε.

The modern statement (Barron 1993) replaces "any" with "functions with bounded spectral norm," and gives sample complexity bounds. For deep networks, the complexity argument is stronger: certain functions that require exponentially many neurons to express shallowly can be expressed polynomially efficiently with depth (Bengio & Delalleau 2011).

### Softmax and cross-entropy are conjugates

The softmax outputs ŷ are in the (C−1)-simplex. The cross-entropy loss gradient with respect to the pre-activation logits u^L has a particularly clean form that drops out from the chain rule:

```math
∂ℒ/∂u^L = ŷ − y
```

This is the gradient of softmax + cross-entropy with respect to the input logits — a vector of prediction errors. This clean form is why the softmax-cross-entropy combination is the standard output head for classification.

### Parameter count

An MLP with layers of sizes [d_0, d_1, …, d_L] has a total parameter count of:

```math
|θ| = Σ_{ℓ=1}^{L} (d_{ℓ−1} · d_ℓ + d_ℓ)
```

The dominant terms grow quadratically in layer width. A 4-layer MLP with all hidden dims = 1024 and input/output dims 784/10 has ≈ 3M parameters — already large enough to overfit MNIST if not regularised.

---

## Real-world Example

Consider training a 2-layer MLP on MNIST (784-dimensional pixel inputs, 10 classes). The architecture is: Linear(784→512) → ReLU → Linear(512→256) → ReLU → Linear(256→10) → Softmax.

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torchvision import datasets, transforms
from torch.utils.data import DataLoader

# Architecture
class MLP(nn.Module):
    def __init__(self) -> None:
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(784, 512),
            nn.ReLU(),
            nn.Linear(512, 256),
            nn.ReLU(),
            nn.Linear(256, 10),
        )

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        # x: [B, 1, 28, 28]
        return self.net(x.view(x.size(0), -1))  # flatten to [B, 784]

# Data
transform = transforms.Compose([transforms.ToTensor(),
                                  transforms.Normalize((0.1307,), (0.3081,))])
train_loader = DataLoader(
    datasets.MNIST("/tmp/mnist", train=True, download=True, transform=transform),
    batch_size=256, shuffle=True,
)

model = MLP()
# CrossEntropyLoss = log-softmax + NLL — numerically stable; no need for explicit softmax
criterion = nn.CrossEntropyLoss()
optimizer = optim.Adam(model.parameters(), lr=1e-3)

for epoch in range(5):
    for x, y in train_loader:
        optimizer.zero_grad()
        logits = model(x)          # [B, 10] unnormalised scores (logits)
        loss = criterion(logits, y)
        loss.backward()
        optimizer.step()
    print(f"epoch {epoch+1}  loss={loss.item():.4f}")
```

> [!TIP]
> Use `nn.CrossEntropyLoss()` (which combines log-softmax and NLL) rather than applying `nn.Softmax()` manually followed by `nn.NLLLoss()`. The fused implementation is numerically more stable — it subtracts max(logits) before computing the exponential to prevent overflow, which is equivalent to the log-sum-exp trick.

After 5 epochs with Adam, this model reaches ~98% test accuracy on MNIST — a textbook sanity check for any new training pipeline.

---

## In Practice

**Dead ReLU neurons.** If a neuron's pre-activation is consistently negative (e.g., due to a large negative bias), its gradient is permanently zero and it never updates. This is the "dying ReLU" problem. Mitigation: careful weight initialisation (He initialisation scales W by √(2/fan_in) for ReLU), LeakyReLU (small negative slope for x < 0), or GELU.

**Vanishing gradients with sigmoid/tanh.** These activations saturate when |x| is large, pushing gradient toward 0. In a 10-layer sigmoid network, the gradient at layer 1 is roughly (0.25)^10 ≈ 10^{−6} of the gradient at layer 10 — effectively zero. This is why ReLU became the default: its derivative is 1 wherever the unit is active.

**Initialisation matters critically.** Xavier (Glorot) initialisation for sigmoid/tanh networks sets Var(W) = 2/(fan_in + fan_out); He initialisation for ReLU sets Var(W) = 2/fan_in. Without this, gradients either explode or vanish at initialisation before training even starts (shown empirically by Glorot & Bengio 2010).

**The cascade of affine maps.** A common mistake is to build a "deep" network of linear layers (nn.Linear) without activations. As shown in the Math section, W_L …W_1 x is still a single linear map. Depth only buys you expressibility when nonlinearities are interleaved.

> [!WARNING]
> `nn.CrossEntropyLoss` expects raw logits, not probabilities. If you pass `softmax(logits)` into `CrossEntropyLoss`, you are computing log(softmax(softmax(x))), which compresses the distribution doubly and produces incorrect gradients. This mistake is silent — the code runs, loss decreases, but accuracy plateaus far below optimal.

---

## Pitfalls

- **"Deeper is always better."** Not without appropriate regularisation. Adding hidden layers without batch normalisation or dropout typically worsens generalisation on small datasets because the extra capacity memorises training noise.
- **"The UAT means one hidden layer is sufficient."** In theory yes, but the required width can be exponential in the input dimension. The UAT is an existence result, not a learnability result.
- **"ReLU networks are piecewise-linear, so they can't fit smooth functions."** They can approximate smooth functions arbitrarily well with enough units — a piecewise-linear approximation converges to any smooth function as the number of pieces grows.
- **"A wider hidden layer is always a substitute for a deeper one."** Not for compositional functions. Depth enables parameter sharing across levels of abstraction; width alone cannot replicate hierarchical composition.
- **"Softmax outputs are probabilities."** They are calibrated to sum to 1 and lie in (0,1), but they are not well-calibrated probabilities unless the model is explicitly calibrated (temperature scaling, Platt scaling). A model can be confidently wrong.

---

## Exercises

### Exercise 1

A 3-layer MLP has hidden dims [d_0=784, d_1=512, d_2=256, d_3=10]. (a) Compute the total parameter count. (b) If you removed all activation functions and collapsed it into a single affine layer d_0 → d_3, how many parameters would that single layer have? What does the comparison tell you?

#### Solution 1

**(a)** Layer 1: 784 × 512 + 512 = 401,920. Layer 2: 512 × 256 + 256 = 131,328. Layer 3: 256 × 10 + 10 = 2,570. Total: **535,818 parameters**.

**(b)** A single linear layer 784 → 10 has 784 × 10 + 10 = 7,850 parameters. The 3-layer MLP uses ≈ 68× more parameters to represent the same kind of function class — except the deep network is *not* the same function class! With activations the MLP can express nonlinear functions; without them it is still just a linear map, so those extra parameters are wasted. The punchline: activation functions are the reason depth matters.

### Exercise 2

The softmax + cross-entropy gradient with respect to logits u^L is ŷ − y. Derive this from first principles using the chain rule.

#### Solution 2

Let a_c = u^L_c (logits), ŷ_c = exp(a_c)/Z where Z = Σ_k exp(a_k), and ℒ = −log ŷ_{y_true} = −a_{y_true} + log Z.

∂ℒ/∂a_c = −𝟏[c = y_true] + exp(a_c)/Z = −y_c + ŷ_c.

In vector form: **∂ℒ/∂a = ŷ − y**. This is why the gradient at the output layer of a classification network is simply the prediction error — an elegant consequence of the softmax-cross-entropy pairing.

### Exercise 3

Explain why a sigmoid-activated network of depth 20 fails to train via standard backpropagation, referencing the vanishing-gradient phenomenon quantitatively.

#### Solution 3

The sigmoid derivative is σ'(x) = σ(x)(1 − σ(x)) ≤ 0.25, with the maximum at x = 0. During backpropagation the gradient at layer ℓ is multiplied by W^ℓ · diag(σ'(u^ℓ)) at each layer. If the spectral norm of W^ℓ is of order 1 and the saturated derivative is ≈ 0.25, the gradient magnitude at layer 1 is at most (0.25)^{20} ≈ 9 × 10^{−13} relative to the output gradient. In practice, random initialisation puts most pre-activations away from zero, so σ' is even smaller (closer to 0), making the product vanishingly small. The network's lower layers receive effectively zero gradient and do not train, which is why ReLU (derivative exactly 1 for positive pre-activations) or residual connections are required for deep networks.

---

## Sources

- de Freitas, N. Oxford Lecture oxf(12): MLP Foundations, Probabilistic Interpretation, Softmax. 2014.
- Gavves, E. UvA Lecture ail(6): Introduction to Neural Networks and Deep Learning. 2015.
- AI Neural Networks Lecture air(15): Perceptron history, LeCun 1989, backpropagation revival.
- Rosenblatt, F. (1958). The Perceptron: A Probabilistic Model for Information Storage and Organisation in the Brain. *Psychological Review*, 65(6), 386–408.
- Cybenko, G. (1989). Approximation by Superpositions of a Sigmoidal Function. *Mathematics of Control, Signals and Systems*, 2(4), 303–314.
- Glorot, X. & Bengio, Y. (2010). Understanding the Difficulty of Training Deep Feedforward Neural Networks. *AISTATS*. http://proceedings.mlr.press/v9/glorot10a.html
- Bengio, Y. (2009). Learning Deep Architectures for AI. *Foundations and Trends in Machine Learning*, 2(1), 1–127.
- Conversation with user on 2026-05-19.

---

## Related

- [2 - Backpropagation](./2-backpropagation.md)
- [3 - Convolutional Networks](./3-convolutional-networks.md)
- [5 - Deep Architecture Principles](./5-deep-architecture-principles.md)
- [5 - Perceptron and Logistic Regression](../2-supervised-learning/5-perceptron-and-logistic-regression.md)
- [4 - Optimization and KKT](../1-foundations/4-optimization-and-kkt.md)
- [12 - Affine Spaces and Affine Mappings](../../Mathematics/Linear-Algebra/12-affine-spaces-and-affine-mappings.md)
