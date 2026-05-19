---
title: "5 - Deep Architecture Principles"
created: 2026-05-19
updated: 2026-05-19
tags: []
aliases: []
---

# 5 - Deep Architecture Principles

[toc]

> **TL;DR:** Depth in neural networks is not an engineering preference — it has a precise theoretical basis: certain functions require exponentially many units to represent shallowly but only polynomially many with depth. Bengio's research programme argues that the key to AI-level generalisation is learning *distributed hierarchical representations* that disentangle the factors of variation underlying the data, with each level composing representations from the level below. The practical engineering of depth requires solving three interrelated problems: vanishing/exploding gradients (residual connections), internal covariate shift (normalisation), and representation collapse (careful initialisation and regularisation).

---

## Vocabulary

- **Depth of architecture** — the number of nonlinear composition levels. Bengio (2009): this determines whether a function can be represented compactly.
- **Distributed representation** — a vector in which each component participates in representing multiple concepts and each concept is encoded in multiple components jointly. Contrasts with *local* (one-hot, grandmother-cell) representations.
- **Factors of variation** — the underlying independent causes of variability in the data (e.g., pose, lighting, identity for faces; syntax, semantics for text). The goal of deep representation learning is to disentangle these.
- **Greedy layer-wise pretraining** — the 2006 breakthrough technique (Hinton et al., Bengio et al., Ranzato et al.): train one layer at a time unsupervised (with RBMs or autoencoders), stacking results as initialisations for a deep supervised network.
- **Restricted Boltzmann Machine (RBM)** — an energy-based model with visible and hidden units, no within-layer connections. Training via contrastive divergence. Building block of the Deep Belief Network.
- **Deep Belief Network (DBN)** — a hierarchical generative model built by stacking RBMs greedily. The first deep architecture trained successfully (Hinton et al. 2006).
- **Residual connection (skip connection)** — an additive bypass h^{ℓ+2} = F(h^ℓ; θ) + h^ℓ that enables gradient to flow directly from loss to early layers without attenuation through F.
- **Batch Normalisation (BN)** — normalises pre-activations within a mini-batch to zero mean and unit variance, then scales/shifts with learned γ, β. Reduces internal covariate shift, allows higher learning rates, provides mild regularisation.
- **Layer Normalisation (LN)** — normalises across the feature dimension (not the batch dimension), making it batch-size independent. Standard in Transformers.
- **Internal covariate shift** — the change in the distribution of layer inputs as parameters update during training. BN addresses this by re-normalising at each layer.
- **Representation learning** — learning features (representations) that are useful for a task, rather than hand-engineering them. The central goal of deep learning.
- **Transfer learning** — reusing representations learned on a large dataset (e.g., ImageNet) as a starting point for a different task or dataset. Enabled by the generality of learned low-level features.
- **Dropout** — training regularisation that randomly sets a fraction p of activations to zero per mini-batch; at test time, weights are scaled by (1−p). Equivalent to averaging over an exponential ensemble of subnetworks.
- **Sparse activations** — representations where most units are zero for any given input. Brains use ~1–4% active neurons simultaneously; deep networks with ReLU naturally produce sparse activations.

---

## Intuition

Why does depth help? Consider representing the parity function of n bits (output 1 iff an odd number of bits are 1). A single hidden layer needs 2^{n−1} units to express it. A depth-log(n) network of XOR gates expresses it with O(n) units. This exponential gap between shallow and deep representations is the formal motivation for depth.

The representational argument (Bengio 2009): high-level abstractions are *compositional*. The concept "face" is built from parts (eyes, nose, mouth), which are built from edge conjunctions, which are built from oriented edges. A deep hierarchy shares low-level features across many concepts — edge detectors at layer 1 are useful whether you're recognising dogs, cars, or chairs. A shallow model must learn these features independently for each class.

The disentanglement argument (Bengio 2013): the raw pixel space of images is highly entangled — a small change in pose (one underlying factor) changes thousands of pixels. A good representation should be factored: changing one factor should correspond to a simple, ideally linear, change in representation space.

> [!NOTE]
> The 2006 deep learning renaissance was enabled by unsupervised pretraining as initialisation. By 2012 (AlexNet), researchers found that pretraining was unnecessary if (1) ReLU replaced sigmoid, (2) Dropout was used, (3) large labelled datasets were available, and (4) GPUs were used. Pretraining has since returned in the form of self-supervised pretraining (masked language modeling, contrastive learning) at much larger scale.

---

## How it works

### Why Depth: The Computational Complexity Argument

Bengio & Delalleau (2011) formalise the depth-efficiency argument: there exist families of functions computable by a depth-k circuit of size s(n) that require a depth-(k−1) circuit of size exp(s(n)). Informally: removing one layer of nonlinearity can cost an exponential increase in network size to maintain the same approximation quality.

The informal argument from Bengio (2009): the number of distinct regions a piecewise-linear network can represent in input space scales exponentially in depth but only polynomially in width. A ReLU network of depth d and width w partitions ℝ^n into up to O(w^{dn}) linear regions — depth multiplies the exponent, width only the base.

### The Training Difficulty Problem and its History

Training deep networks before 2006 consistently failed. Bengio (2013) attributes this to:

1. **Vanishing gradients**: gradients of sigmoid/tanh networks decay exponentially with depth.
2. **Poor initialisation**: random small weights near zero cause tiny gradients; random large weights cause saturation.
3. **No intermediate training signal**: lower layers receive only the backpropagated gradient from the top, which is effectively zero for deep sigmoid networks.

The 2006 solution (greedy layer-wise pretraining) solved problem 3 by training each layer separately with a local unsupervised objective (RBM contrastive divergence or autoencoder reconstruction). The resulting initialisations placed the weights in a basin of attraction where fine-tuning with backprop worked.

### Residual Connections

A ResNet block computes the output h^{ℓ+2} = F(h^ℓ; θ) + h^ℓ, where F represents two conv-BN-ReLU operations. The skip connection is the identity function. The gradient of the loss with respect to h^ℓ is:

```math
∂ℒ/∂h^ℓ = ∂ℒ/∂h^{ℓ+2} · (I + ∂F/∂h^ℓ)
```

The identity term I ensures that the gradient can always flow back unchanged, regardless of ∂F/∂h^ℓ. Even if F learns near-zero weights (lazy layers), the gradient highway through the skip connection keeps gradient magnitude O(1) across all layers. This is why ResNets can be trained to 1000+ layers.

The residual learning perspective: instead of learning h^{ℓ+2} = G(h^ℓ), the network learns F = G − I (the residual). If the identity is already close to optimal, F ≈ 0 is easy to learn; the network only needs to learn the small correction.

> [!IMPORTANT]
> The skip connection in ResNets must match the spatial and channel dimensions of the two endpoints. If a strided convolution halves the spatial resolution or a projection increases channels, a 1×1 conv projection must be applied to the shortcut: h^{ℓ+2} = F(h^ℓ) + W_s h^ℓ. Forgetting this projection when dimensions change produces a dimension mismatch error at the addition step.

### Batch Normalisation

The BN transformation for a layer with d-dimensional pre-activation u (computed over a mini-batch of size B) is:

```math
μ_B = (1/B) Σ_b u_b         # batch mean
σ²_B = (1/B) Σ_b (u_b − μ_B)²   # batch variance
û_b = (u_b − μ_B) / sqrt(σ²_B + ε)   # normalise
z_b = γ ⊙ û_b + β            # scale and shift (γ, β are learned)
```

During training, the running exponential moving averages of μ and σ² are tracked. At inference, these running statistics replace the batch statistics, making BN deterministic.

The effect: BN constrains each layer's activations to have roughly zero mean and unit variance at each training step, preventing the distribution from drifting far as earlier layers update. This allows higher learning rates (the loss landscape is smoother), reduces sensitivity to initialisation, and provides a mild regularisation effect (the mini-batch statistics add noise similar to dropout).

### Dropout as Ensemble Regularisation

Dropout with rate p works as follows: during training, each activation is independently zeroed with probability p and scaled by 1/(1−p) (or equivalently, at test time the weights are multiplied by (1−p)). This trains 2^n different subnetworks (for n units), all sharing parameters. The test-time network computes an approximation to the geometric mean of these 2^n networks' predictions.

From Bengio's (2013) conditional computation perspective, dropout also encourages sparse gradient updates and prevents co-adaptation: each unit must be useful on its own because it cannot rely on specific co-activating units.

---

## Math

### Local vs Non-local Generalisation

A learning algorithm generalises *locally* if it interpolates between nearby training examples: changing the test input changes the output only if the new input is near a training example. K-nearest-neighbours, Gaussian process regression, and RBF kernel SVMs are local.

A distributed representation provides *non-local* generalisation: the 2^n possible activation patterns of n binary hidden units can distinguish up to 2^n distinct input regions, each corresponding to a different concept. A local representation with n units can only distinguish n regions. This exponential advantage is the fundamental theoretical justification for representation learning:

```math
local capacity: O(n) input regions
distributed capacity: O(2^n) input regions (with n binary features)
```

### Glorot/He Initialisation Derivation

For a layer h = σ(W x + b) with x ∈ ℝ^{fan_in}, the variance of the output should equal the variance of the input to prevent explosion or vanishing. Assuming W_ij ~ N(0, σ_w²), x_j ~ N(0, σ_x²), and linear activation (for the derivation):

```math
Var(h_i) = fan_in · σ_w² · Var(x_j)
```

Setting Var(h_i) = Var(x_j) requires σ_w² = 1/fan_in. Xavier (Glorot & Bengio 2010) symmetrises this for both forward and backward passes:

```math
σ_w² = 2 / (fan_in + fan_out)
```

For ReLU (which zeros half the outputs), the effective fan_in is halved, so He initialisation (He et al. 2015) uses:

```math
σ_w² = 2 / fan_in
```

---

## Real-world Example

Comparing training dynamics of a plain deep network vs a residual network on CIFAR-10 — the classic experiment from He et al. (2015).

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class PlainBlock(nn.Module):
    """Two conv layers, no skip connection."""

    def __init__(self, channels: int) -> None:
        super().__init__()
        self.layers = nn.Sequential(
            nn.Conv2d(channels, channels, 3, padding=1, bias=False),
            nn.BatchNorm2d(channels),
            nn.ReLU(),
            nn.Conv2d(channels, channels, 3, padding=1, bias=False),
            nn.BatchNorm2d(channels),
        )

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        return F.relu(self.layers(x))   # no skip


class ResidualBlock(nn.Module):
    """Same as PlainBlock but with additive skip connection."""

    def __init__(self, channels: int) -> None:
        super().__init__()
        self.layers = nn.Sequential(
            nn.Conv2d(channels, channels, 3, padding=1, bias=False),
            nn.BatchNorm2d(channels),
            nn.ReLU(),
            nn.Conv2d(channels, channels, 3, padding=1, bias=False),
            nn.BatchNorm2d(channels),
        )

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        return F.relu(self.layers(x) + x)   # skip: h^{l+2} = F(h^l) + h^l


def build_model(block_class: type, n_blocks: int, use_residual: bool = True) -> nn.Module:
    """Build a 3-stage model stacking n_blocks of the given block type per stage."""
    def make_stage(c_in: int, c_out: int, n: int, stride: int) -> nn.Sequential:
        layers: list[nn.Module] = [
            nn.Conv2d(c_in, c_out, 3, stride=stride, padding=1, bias=False),
            nn.BatchNorm2d(c_out), nn.ReLU(),
        ]
        for _ in range(n):
            layers.append(block_class(c_out))
        return nn.Sequential(*layers)

    return nn.Sequential(
        make_stage(3, 64, n_blocks, stride=1),
        make_stage(64, 128, n_blocks, stride=2),
        make_stage(128, 256, n_blocks, stride=2),
        nn.AdaptiveAvgPool2d(1),
        nn.Flatten(),
        nn.Linear(256, 10),
    )

# The He et al. 2015 experiment: 20-block plain vs residual
plain_20  = build_model(PlainBlock, n_blocks=10)
resnet_20 = build_model(ResidualBlock, n_blocks=10)

# Parameter counts
n_plain  = sum(p.numel() for p in plain_20.parameters())
n_resnet = sum(p.numel() for p in resnet_20.parameters())
print(f"Plain-20:  {n_plain/1e6:.2f}M params")
print(f"ResNet-20: {n_resnet/1e6:.2f}M params")
# They are essentially equal — the skip connection adds zero parameters.
# Yet ResNet-20 trains to ~91% on CIFAR-10; Plain-20 degrades to ~87%.
```

> [!TIP]
> He initialisation (`torch.nn.init.kaiming_normal_(weight, mode='fan_in', nonlinearity='relu')`) is the appropriate default for any ReLU network. Xavier initialisation (the PyTorch default, `nn.Linear` uses uniform Xavier) is designed for sigmoid/tanh and will cause gradients to vanish in deep ReLU networks. Always explicitly set initialisation for deep networks rather than relying on defaults.

---

## In Practice

**Pretraining regime shift.** In 2006–2012, deep networks required unsupervised pretraining (RBMs, autoencoders) to initialise weights in a good basin before supervised fine-tuning. By 2012, the combination of ReLU, Dropout, data augmentation, and GPU-scale training made pretraining unnecessary for supervised tasks. By 2018–2020, pretraining returned in a new form: massive self-supervised pretraining (BERT, GPT, CLIP) on internet-scale data, followed by fine-tuning. The mechanism is the same — initialise in a good region of weight space — but the scale is orders of magnitude larger.

**Batch normalisation and batch size coupling.** BN's statistics are computed over the current mini-batch. Very small batch sizes (< 8) produce noisy statistics and unstable training. This is a problem for memory-constrained training (large images, long sequences). Alternatives: Layer Norm (normalise across features, batch-independent), Group Norm (normalise within channel groups), or Ghost BN. For sequence models (Transformers), Layer Norm is the standard.

**Residual connections and the identity mapping.** He et al. (2016) showed that placing BN and ReLU *before* the convolution (pre-activation ResNet: BN → ReLU → Conv → BN → ReLU → Conv + skip) produces a pure identity shortcut with no activation in the skip path, enabling gradient flow without any transformation. This "pre-act ResNet" is slightly more principled theoretically and is used in very deep networks (ResNet-1001).

**Conditional computation as a scaling path.** Bengio (2013) proposed conditional computation — activating only a subset of parameters per input — as a way to scale models while keeping per-sample compute constant. This idea materialised as Mixture of Experts (MoE) architectures in which a gating network routes each token to one of K expert FFN layers. GShard (Lepikhin et al. 2021), Switch Transformer (Fedus et al. 2022), and Mixtral (Mistral 2024) are direct realisations of Bengio's 2013 proposal.

> [!WARNING]
> Batch normalisation has different behaviour in `model.train()` (uses batch stats, updates running estimates) vs `model.eval()` (uses frozen running estimates). Forgetting to call `model.eval()` before inference causes non-deterministic outputs that vary with batch size and content — a subtle bug that manifests as surprisingly bad validation accuracy when you test one example at a time (batch size = 1 makes BN degenerate).

---

## Pitfalls

- **"Deeper networks always perform better."** Beyond a certain depth, without residual connections or normalisation, deeper networks exhibit higher *training* error than shallower ones — not just test error. This is the degradation problem that He et al. 2015 demonstrated: a 56-layer plain network had higher training error than a 20-layer plain network on CIFAR-10. Adding residual connections eliminates this degradation.
- **"Pretraining is obsolete."** Pretraining is central to modern ML — it just operates at web scale with self-supervision rather than on task-specific unsupervised objectives. GPT-4, Llama, CLIP, DINOv2 are all heavily pretrained models.
- **"Distributed representations are always better than local ones."** For small datasets or tasks requiring explicit interpretability, local representations (sparse codes, prototype-based models, decision trees) can be preferable. Distributed representations require sufficient data to regularise the exponentially large capacity.
- **"Batch normalisation is a regulariser."** BN has a regularisation effect (adds noise via batch statistics), but it was designed to solve internal covariate shift. Using BN as a substitute for Dropout is possible but unreliable — on small datasets BN's regularisation is weaker.
- **"Layer normalisation and batch normalisation are interchangeable."** BN normalises over the batch for each feature; LN normalises over features for each sample. Their inductive biases differ: BN ties the statistics of different samples in a batch; LN is completely per-sample. For Transformers (variable batch size, sequences of varying length), LN is the right choice. For ConvNets on fixed-size images, BN is standard.

---

## Exercises

### Exercise 1

An L-layer network of width w uses ReLU activations. Approximately how many linear regions can it represent in an n-dimensional input space? How does this compare to a single-hidden-layer network of the same total parameter count?

#### Solution 1

Using the result from Montufar et al. (2014): a depth-L ReLU network of width w can create up to O((w/n)^{n(L−1)} · w^n) linear regions — this grows as w^{nL}, exponential in both width and depth.

A single-hidden-layer network with the same total parameters W = L · w^2 has width w' = √(W/1) = w√L. Its linear regions are O((w')^n) = O((w√L)^n) = O(w^n · L^{n/2}). 

The deep network creates exponentially more regions in depth L than the shallow network: the gap is roughly (w/n)^{n(L−1)} ≈ w^{n(L−2)} times more for large L. Depth multiplies the *exponent* of the capacity, not just the base — this is the formal statement of why depth is more than just extra parameters.

### Exercise 2

Derive the gradient flow through a residual block h^{ℓ+2} = F(h^ℓ) + h^ℓ and show why it prevents vanishing gradients.

#### Solution 2

By the chain rule:

∂ℒ/∂h^ℓ = ∂ℒ/∂h^{ℓ+2} · ∂h^{ℓ+2}/∂h^ℓ = ∂ℒ/∂h^{ℓ+2} · (I + ∂F(h^ℓ)/∂h^ℓ)

The term I is the identity matrix — it contributes a direct additive gradient of magnitude ||∂ℒ/∂h^{ℓ+2}|| regardless of what F learns. Even if ∂F/∂h^ℓ → 0 (the residual branch contributes nothing), the gradient is:

∂ℒ/∂h^ℓ ≈ ∂ℒ/∂h^{ℓ+2}

This means the gradient at layer ℓ is at least as large as at layer ℓ+2. Over L blocks, the gradient at layer 0 includes a direct path of magnitude ||∂ℒ/∂h^L|| with no attenuation. This is why a ResNet-152 trains as well as a ResNet-20 without gradient vanishing.

---

## Sources

- Bengio, Y. (2009). Learning Deep Architectures for AI. *Foundations and Trends in Machine Learning*, 2(1), 1–127.
- Bengio, Y. (2013). Deep Learning of Representations: Looking Forward. arXiv:1305.0445.
- Hinton, G.E., Osindero, S. & Teh, Y.W. (2006). A Fast Learning Algorithm for Deep Belief Nets. *Neural Computation*, 18(7), 1527–1554.
- He, K., Zhang, X., Ren, S. & Sun, J. (2015). Deep Residual Learning for Image Recognition. arXiv:1512.03385. *CVPR 2016*.
- He, K. et al. (2015). Delving Deep into Rectifiers. arXiv:1502.01852.
- Ioffe, S. & Szegedy, C. (2015). Batch Normalization: Accelerating Deep Network Training. *ICML*.
- Glorot, X. & Bengio, Y. (2010). Understanding the Difficulty of Training Deep Feedforward Neural Networks. *AISTATS*.
- Srivastava, N., Hinton, G., Krizhevsky, A., Sutskever, I. & Salakhutdinov, R. (2014). Dropout. *JMLR*, 15, 1929–1958.
- Conversation with user on 2026-05-19.

---

## Related

- [1 - Neural Networks Foundations](./1-neural-networks-foundations.md)
- [2 - Backpropagation](./2-backpropagation.md)
- [3 - Convolutional Networks](./3-convolutional-networks.md)
- [4 - Recurrent Networks and LSTM](./4-recurrent-networks-and-lstm.md)
- [6 - Language Modeling Foundations](./6-language-modeling-foundations.md)
- [4 - Optimization and KKT](../1-foundations/4-optimization-and-kkt.md)
- [3 - Generative AI Fundamentals](../../AI-Engineering/1-foundations/3-generative-ai-fundamentals.md)
