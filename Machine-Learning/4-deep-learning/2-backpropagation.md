---
title: "2 - Backpropagation"
created: 2026-05-19
updated: 2026-05-19
tags: []
aliases: []
---

# 2 - Backpropagation

[toc]

> **TL;DR:** Backpropagation is dynamic programming applied to a computation graph: it propagates error signals from the loss back through each layer by the chain rule, computing gradients of all parameters in a single reverse pass at a cost proportional to the forward pass. The modular formulation — each layer specifies a forward function and a backward function — is how PyTorch's autograd engine works, and it is why you can compose arbitrary differentiable building blocks without deriving new gradient formulas by hand.

---

## Vocabulary

- **Computational graph** — a directed acyclic graph (DAG) where nodes are operations and edges are tensor values. Backprop is a reverse traversal of this graph.
- **Chain rule** — if z = f(y) and y = g(x), then dz/dx = (dz/dy)(dy/dx). The engine of backprop.
- **Delta (δ^ℓ)** — the error signal at layer ℓ: δ^ℓ = ∂ℒ/∂u^ℓ, the gradient of the loss with respect to the pre-activation. This is the core quantity propagated backward.
- **Jacobian** — the matrix of all partial derivatives of a vector-valued function. The backward pass for a layer is a Jacobian-vector product (JVP) operation.
- **JVP (Jacobian-vector product)** — given a function f: ℝ^n → ℝ^m with Jacobian J ∈ ℝ^{m×n}, a JVP is Jv for a vector v ∈ ℝ^n. Forward-mode AD computes these.
- **VJP (vector-Jacobian product)** — vᵀJ for a row-vector v ∈ ℝ^m. Reverse-mode AD (backprop) computes VJPs. For scalar losses, this is the gradient.
- **Reverse-mode autodiff** — the algorithmic framework behind backprop. Efficient when the output dimensionality ≪ input dimensionality (typical in ML: scalar loss, millions of parameters).
- **Forward pass** — computes the loss and stores intermediate activations needed for the backward pass.
- **Backward pass** — traverses the graph in reverse, accumulating parameter gradients.
- **Modular backprop** — the design pattern in which each module (linear layer, activation, loss) independently implements a backward method, accepting the upstream gradient and returning the downstream gradient.
- **Gradient tape** — TensorFlow's name for the dynamic computation graph that records operations for later differentiation; PyTorch's equivalent is `autograd`.
- **Vanishing gradient** — exponential decay of δ^ℓ as ℓ decreases, caused by chains of small derivative values. The core failure mode of deep sigmoid networks.
- **Exploding gradient** — exponential growth of δ^ℓ; handled by gradient clipping.

---

## Intuition

Imagine tracing how a small perturbation in weight W^ℓ_ij propagates forward to change the loss. It first changes u^ℓ_i, then h^ℓ_i, then every downstream activation, finally ℒ. The path is a product of derivatives along edges of the computation graph. Backprop collects all those path products efficiently: instead of computing one derivative per weight (which would require N forward passes for N parameters), it computes all of them in a single reverse pass by sharing intermediate computations.

The DP insight: the gradient ∂ℒ/∂h^ℓ at layer ℓ depends only on the gradient ∂ℒ/∂h^{ℓ+1} and the local derivative ∂h^{ℓ+1}/∂h^ℓ. Compute these quantities once, left-to-right in the forward pass; then right-to-left in the backward pass. This is exactly how the Viterbi algorithm works on a chain HMM, or how the Bellman equation propagates rewards backward in time.

---

## How it works

The backpropagation algorithm operates in two phases: a forward pass that computes and caches activations, and a backward pass that propagates gradients from the loss back to each parameter.

### Phase 1 — Forward Pass

The forward pass evaluates the network's prediction and stores the pre-activations u^ℓ and post-activations h^ℓ at every layer, because the backward pass needs them to compute local derivatives. For an L-layer MLP:

```math
for ℓ = 1 to L:
    u^ℓ = W^ℓ h^{ℓ−1} + b^ℓ      # cache u^ℓ
    h^ℓ = σ^ℓ(u^ℓ)                # cache h^ℓ

ŷ = h^L (last layer, with softmax if classification)
ℒ = cross_entropy(ŷ, y)
```

### Phase 2 — Backward Pass (delta propagation)

The backward pass initialises the error signal at the output layer, then propagates it backward layer by layer. The key quantity is the delta δ^ℓ = ∂ℒ/∂u^ℓ — the gradient with respect to pre-activations.

At the output layer (softmax + cross-entropy), the delta has the clean form derived in note 1:

```math
δ^L = ŷ − y
```

For intermediate layers ℓ < L, the delta is propagated through the activation derivative and weight matrix using the chain rule:

```math
δ^ℓ = (W^{ℓ+1}ᵀ δ^{ℓ+1}) ⊙ σ'(u^ℓ)
```

Here ⊙ is elementwise multiplication and σ'(u^ℓ) is the elementwise derivative of the activation at the cached pre-activation.

### Phase 3 — Parameter Gradients

Once the deltas are computed, the parameter gradients follow directly by applying the chain rule to the affine transformation in each layer:

```math
∂ℒ/∂W^ℓ = δ^ℓ (h^{ℓ−1})ᵀ
∂ℒ/∂b^ℓ = δ^ℓ
```

These are the gradients that SGD (or Adam) uses to update W^ℓ and b^ℓ.

> [!IMPORTANT]
> The backward formula δ^ℓ = (W^{ℓ+1}ᵀ δ^{ℓ+1}) ⊙ σ'(u^ℓ) requires the *cached* pre-activation u^ℓ from the forward pass. If you free activations early (to save memory), you must recompute them — this is the trade-off exploited by gradient checkpointing.

### Phase 4 — Modular Layer Interface

The de Freitas lecture (oxf(11)) formalises each layer as an object with two methods. This is the design every modern autodiff framework follows:

```python
class Layer:
    def forward(self, z_in):
        # z_in: input activations
        # Returns: z_out (output activations), caches what backward needs
        raise NotImplementedError

    def backward(self, delta_out):
        # delta_out: upstream gradient ∂ℒ/∂z_out
        # Returns: delta_in ∂ℒ/∂z_in, also computes ∂ℒ/∂θ internally
        raise NotImplementedError
```

A network is then a composition of layers. The forward pass runs them left to right; the backward pass runs them right to left.

---

## Math

### Full backprop derivation for a linear layer

A linear layer computes z_out = W z_in + b. The Jacobian ∂z_out/∂z_in = W (an m×n matrix). Given upstream gradient δ_out = ∂ℒ/∂z_out ∈ ℝ^m, the downstream gradient is:

```math
δ_in = ∂ℒ/∂z_in = Wᵀ δ_out   ∈ ℝ^n
```

The weight gradient is the outer product of the upstream gradient with the input:

```math
∂ℒ/∂W = δ_out (z_in)ᵀ   ∈ ℝ^{m×n}
∂ℒ/∂b = δ_out            ∈ ℝ^m
```

### Elementwise activation layer

For z_out = σ(z_in) where σ is applied elementwise, the Jacobian is diagonal: ∂z_out/∂z_in = diag(σ'(z_in)). The downstream gradient is:

```math
δ_in = σ'(z_in) ⊙ δ_out
```

### Vanishing gradient: quantitative bound

For an L-layer sigmoid network with weights having spectral norm ≈ 1, the gradient magnitude at layer ℓ is bounded by:

```math
||δ^ℓ|| ≤ (0.25)^{L−ℓ} · ||δ^L||
```

since max σ'(x) = 0.25 for sigmoid. For L = 20 and ℓ = 1: the gradient at layer 1 is at most (0.25)^{19} ≈ 4 × 10^{−12} of the output gradient. Effectively zero.

### Computational complexity

For an MLP with layers of width d, the forward pass costs O(L d²) FLOPs. The backward pass costs the same asymptotic amount — it is also a sequence of matrix-vector products. So backprop doubles the compute of the forward pass, not squares it. Memory cost is O(L d) for storing the activations.

---

## Real-world Example

Implementing a modular linear layer with explicit forward and backward, matching the PyTorch convention, to verify the gradient against torch autograd.

```python
import torch
import torch.nn.functional as F
from typing import Optional

class LinearLayer:
    """Manual linear layer with explicit forward/backward for pedagogic clarity."""

    def __init__(self, in_features: int, out_features: int) -> None:
        # He initialisation for ReLU networks
        std = (2.0 / in_features) ** 0.5
        self.W = torch.randn(out_features, in_features) * std
        self.b = torch.zeros(out_features)
        self.grad_W: Optional[torch.Tensor] = None
        self.grad_b: Optional[torch.Tensor] = None
        self._z_in: Optional[torch.Tensor] = None

    def forward(self, z_in: torch.Tensor) -> torch.Tensor:
        # z_in: [B, in_features]
        self._z_in = z_in                        # cache for backward
        return z_in @ self.W.T + self.b          # [B, out_features]

    def backward(self, delta_out: torch.Tensor) -> torch.Tensor:
        # delta_out: [B, out_features]  upstream gradient ∂L/∂z_out
        assert self._z_in is not None, "call forward() before backward()"
        # ∂L/∂W = delta_outᵀ z_in  (summed over batch)
        self.grad_W = delta_out.T @ self._z_in           # [out, in]
        self.grad_b = delta_out.sum(dim=0)               # [out]
        # ∂L/∂z_in = delta_out @ W
        return delta_out @ self.W                        # [B, in_features]


# Verify against PyTorch autograd
torch.manual_seed(0)
B, D_in, D_out = 4, 8, 6
layer = LinearLayer(D_in, D_out)

x = torch.randn(B, D_in)
y_target = torch.randn(B, D_out)

# Manual forward + backward
z_out = layer.forward(x)
loss_manual = 0.5 * ((z_out - y_target) ** 2).mean()
delta = (z_out - y_target) / (B * D_out)  # gradient of MSE w.r.t. z_out
layer.backward(delta)

# PyTorch autograd
W_pt = layer.W.detach().clone().requires_grad_(True)
b_pt = layer.b.detach().clone().requires_grad_(True)
z_pt = x @ W_pt.T + b_pt
loss_pt = 0.5 * ((z_pt - y_target) ** 2).mean()
loss_pt.backward()

print("grad_W match:", torch.allclose(layer.grad_W, W_pt.grad, atol=1e-5))
print("grad_b match:", torch.allclose(layer.grad_b, b_pt.grad, atol=1e-5))
```

> [!TIP]
> Use `torch.autograd.gradcheck(fn, inputs)` in unit tests for custom autograd functions. It evaluates the analytical Jacobian against a finite-difference numerical Jacobian. If `gradcheck` passes to default tolerance (rtol=1e-3, atol=1e-3), the backward implementation is correct.

---

## In Practice

**Memory bottleneck during training.** Backpropagation requires storing all intermediate activations h^ℓ from the forward pass. For a transformer with sequence length 8192, the activation memory can exceed the HBM capacity. *Gradient checkpointing* (also called activation recomputation) trades memory for compute: only a subset of activations is stored; the rest are recomputed on demand during the backward pass. PyTorch provides `torch.utils.checkpoint.checkpoint()` for this.

**Gradient clipping.** When gradients explode (common in RNNs, early in training of deep ResNets), training destabilises. The standard fix is global norm clipping: if ||∇θ ℒ||₂ > τ, scale the gradient by τ/||∇θ ℒ||₂. PyTorch: `torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)`. Note this distorts the gradient direction slightly.

**Numerical stability of log-softmax.** The naive softmax exp(a_c)/Σ exp(a_k) overflows for large logits. The stable version subtracts max(a): exp(a_c − max) / Σ exp(a_k − max). PyTorch's `F.cross_entropy` does this internally. Never implement softmax naively.

**Sparse gradient updates.** In Bengio's 2003 language model, only the embedding rows corresponding to words in the current context receive gradient updates per example. This is the first instance of the "sparse updates" pattern that Bengio (2013) later proposed as a scaling mechanism: when the input is sparse, most parameters are untouched per step, making asynchronous SGD across CPUs/GPUs communication-efficient.

> [!WARNING]
> Calling `loss.backward()` in PyTorch *accumulates* gradients (adds to `.grad` rather than overwriting). Forgetting `optimizer.zero_grad()` before each training step accumulates gradients across mini-batches, effectively simulating a larger batch size with incorrect scaling. The resulting gradients are wrong and training will diverge or learn incorrectly. Always call `zero_grad()` at the start of each step.

---

## Pitfalls

- **"Backprop is slow because it requires a second forward pass."** The backward pass costs roughly the same FLOPs as one forward pass — not a whole second forward pass. Total training cost is approximately 2× the forward pass cost (plus optimizer overhead).
- **"Backprop requires explicit Jacobian computation."** Full Jacobian materialisation is O(m·n) and is never done in practice. Backprop computes VJPs (vector-Jacobian products), which cost O(n + m) for each layer — the same order as the forward pass.
- **"Gradient checkpointing is always a good idea."** It trades memory for compute. On small models that fit comfortably in GPU memory, the recomputation overhead (typically 1.33× more forward FLOPs per training step) is an unnecessary cost.
- **"The chain rule only applies to scalar-valued compositions."** The chain rule generalises to vector-valued functions via Jacobians. Backprop is a sequence of VJPs on the full Jacobians, which happen to be efficiently computable (often diagonal or structured) for standard layers.
- **"In-place operations are fine in autograd."** PyTorch autograd tracks versions of tensors. In-place operations (tensor.add_(other), tensor[...] = value) can corrupt the computation graph by modifying a tensor that was saved for the backward pass. This produces a cryptic RuntimeError or, worse, silently incorrect gradients.

---

## Exercises

### Exercise 1

Derive the backward pass (gradient formulas for W, b, and δ_in) for a *batch normalisation* layer in inference mode (fixed running mean μ and variance σ²): z_out = (z_in − μ) / sqrt(σ² + ε) · γ + β.

#### Solution 1

Let x_hat = (z_in − μ) / sqrt(σ² + ε), so z_out = γ ⊙ x_hat + β. Given upstream gradient δ_out:

- ∂ℒ/∂γ = Σ_B (δ_out ⊙ x_hat), summed over the batch dimension.
- ∂ℒ/∂β = Σ_B δ_out.
- ∂ℒ/∂x_hat = δ_out ⊙ γ.
- ∂ℒ/∂z_in = (δ_out ⊙ γ) / sqrt(σ² + ε).

In training mode the backward pass is more complex because the mean and variance depend on the batch, introducing additional terms from the chain rule through μ and σ².

### Exercise 2

A network has 3 layers with pre-activations u^1, u^2, u^3. Using ReLU activations and cross-entropy + softmax at the output, write the explicit δ formulas for all three layers.

#### Solution 2

Output layer: δ^3 = ŷ − y (clean softmax-CE gradient).

Layer 2: δ^2 = (W^3ᵀ δ^3) ⊙ 𝟏[u^2 > 0], where 𝟏[u^2 > 0] is the elementwise indicator of positive pre-activations (ReLU derivative).

Layer 1: δ^1 = (W^2ᵀ δ^2) ⊙ 𝟏[u^1 > 0].

Note that any unit in layer ℓ with u^ℓ_i ≤ 0 has δ^ℓ_i = 0 regardless of the upstream signal — these are the "dead" units that receive no gradient.

### Exercise 3

Explain gradient checkpointing. If a 12-layer transformer's activations consume 24 GB with no checkpointing, and you checkpoint every 4 layers (storing only activations at layers 4, 8, 12), how many extra FLOPs does the backward pass cost?

#### Solution 3

With checkpointing every 4 layers, during the backward pass through layers 1–4, activations for layers 1–3 must be recomputed from the checkpoint at layer 4's input. This recomputation is one extra forward pass through layers 1–4 (roughly 4/12 = 1/3 of the full forward cost). The same applies to layers 5–8. Layers 9–12 use the stored final activations.

Total extra FLOPs ≈ 2 × (1/3 forward pass) = 2/3 of a forward pass — roughly 33% overhead. Memory savings: instead of storing activations for all 12 layers (~24 GB), you store 3 checkpoints + activations for the current 4-layer segment, reducing peak memory to roughly ~8 GB (3× reduction). The general rule: checkpointing every k layers reduces activation memory by a factor of roughly k at the cost of O(1/k) additional FLOPs.

---

## Sources

- de Freitas, N. Oxford Lecture oxf(11): Backpropagation — Modular Approach, Chain Rule, Layer Specification. 2014.
- de Freitas, N. Oxford Lecture oxf(12): Backpropagation Algorithm Details. 2014.
- Rumelhart, D.E., Hinton, G.E. & Williams, R.J. (1986). Learning Representations by Back-propagating Errors. *Nature*, 323, 533–536.
- Bengio, Y., Ducharme, R., Vincent, P. & Jauvin, C. (2003). A Neural Probabilistic Language Model. *JMLR*, 3, 1137–1155.
- Conversation with user on 2026-05-19.

---

## Related

- [1 - Neural Networks Foundations](./1-neural-networks-foundations.md)
- [4 - Recurrent Networks and LSTM](./4-recurrent-networks-and-lstm.md)
- [5 - Deep Architecture Principles](./5-deep-architecture-principles.md)
- [4 - Optimization and KKT](../1-foundations/4-optimization-and-kkt.md)
