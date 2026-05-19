---
title: "3 - Convolutional Networks"
created: 2026-05-19
updated: 2026-05-19
tags: []
aliases: []
---

# 3 - Convolutional Networks

[toc]

> **TL;DR:** Convolutional Neural Networks (ConvNets) exploit three structural inductive biases — local receptive fields, weight sharing across spatial positions, and hierarchical pooling — to learn translation-equivariant feature detectors for grid-structured data (images, audio, text). What started as LeNet-5 for digit recognition (LeCun 1989) became the engine of the ImageNet era (AlexNet 2012, ZFNet 2013, VGGNet 2014, ResNet 2015) and revealed through visualisation (Zeiler & Fergus 2014, Simonyan et al. 2014) that CNNs automatically learn a hierarchy from edge detectors in layer 1 to class-discriminative part detectors in higher layers.

---

## Vocabulary

- **Convolution (cross-correlation)** — the operation that slides a learned filter kernel across an input feature map, computing a dot product at each spatial position. In deep learning "convolution" technically means cross-correlation (no kernel flip).
- **Feature map** — the 3D tensor (height × width × channels) produced by a convolutional layer.
- **Kernel / filter** — a small weight tensor (e.g., 3×3×C_in) that is convolved with the input. One filter produces one output channel.
- **Stride** — the step size of the sliding window. Stride = 2 halves the spatial resolution without a pooling layer.
- **Padding** — zeros appended to the input boundary to control output spatial size. "Same" padding keeps H × W constant; "valid" padding reduces it.
- **Receptive field** — the region of the original input that influences a given neuron. Grows with depth: a 3×3 conv on top of another 3×3 conv has a 5×5 effective receptive field.
- **Weight sharing** — all spatial positions in a feature map share the same filter weights. This enforces translation equivariance and reduces parameter count from O(H·W·C) to O(k·k·C).
- **Translation equivariance** — if the input is shifted by (Δx, Δy), the feature map shifts by the same amount. Convolution + ReLU preserve this property; fully-connected layers do not.
- **Max pooling** — takes the maximum value within a local spatial window. Provides a degree of translation invariance and reduces spatial resolution.
- **Global average pooling (GAP)** — averages the entire spatial extent of each feature map to a scalar, producing a channel vector. Replaces fully-connected layers in modern architectures (Network-in-Network, ResNet).
- **Deconvnet (transposed conv)** — a ConvNet run in reverse for visualisation: given a feature map activation, reconstructs the input-space pattern that caused it (Zeiler & Fergus 2014).
- **Saliency map** — the gradient of the class score with respect to input pixels (Simonyan et al. 2014). Shows which pixels most influence the prediction.
- **Guided backpropagation** — a variant of saliency that masks negative gradients at both the upstream gradient and the ReLU activation, producing sharper reconstructions (Springenberg et al. 2015).
- **Locally connected layer** — like convolution but without weight sharing: each spatial position has its own unique filter. Used in DeepFace for aligned faces where spatial stationarity doesn't hold.

---

## Intuition

Flatten a 224×224×3 image and pass it through a fully-connected layer: the weight matrix has 224×224×3 × (number of hidden units) parameters, the model has no concept of proximity, and translation of an object by one pixel produces a completely different activation pattern. Convolution fixes all three issues by tying weights across positions and restricting connectivity to a local window.

Stacking convolution + pooling layers produces a **feature hierarchy**: layer 1 filters become Gabor-like edge detectors (verified by visualisation); layer 2 detects textures and curves; layer 3 captures more complex invariances (mesh patterns, text); layer 4 becomes class-specific (dog faces, bird legs); layer 5 responds to entire objects with pose variation. This is the hierarchy Zeiler & Fergus revealed by projecting feature map activations back to pixel space through the deconvnet.

The key efficiency: a 3×3 convolutional layer with C_in input channels and C_out output channels has only 9·C_in·C_out + C_out parameters regardless of the spatial dimensions. A fully-connected layer over the same feature map with H = W = 56 would require 56²·C_in × 56²·C_out = 9.8M times more parameters for C_in = C_out = 256.

> [!NOTE]
> Springenberg et al. (2015) showed that max-pooling is not strictly necessary: replacing pooling with a strided convolutional layer (stride 2) achieves the same accuracy on CIFAR-10 and CIFAR-100 while reducing model complexity. The spatial dimension reduction is what matters, not the max-pooling operation per se.

---

## How it works

### Forward Pass: 2D Convolution

The 2D convolution (cross-correlation) operation for input X ∈ ℝ^{H×W×C_in} with filter θ ∈ ℝ^{k×k×C_in×C_out} and bias b ∈ ℝ^{C_out} produces output Y ∈ ℝ^{H'×W'×C_out}:

```math
Y_{i',j',f'} = b_{f'} + Σ_{i=1}^{k} Σ_{j=1}^{k} Σ_{f=1}^{C_in} X_{(i'+i−1),(j'+j−1),f} · θ_{i,j,f,f'}
```

The output spatial dimensions with padding P and stride S are H' = (H + 2P − k)/S + 1 and W' = (W + 2P − k)/S + 1.

For the standard 3×3 conv with stride 1 and "same" padding (P=1): H' = H, W' = W.

### Backward Pass: Gradient through Convolution

The gradient of the loss with respect to the input X is a convolution of the upstream gradient δ ∈ ℝ^{H'×W'×C_out} with the filter rotated 180°. The gradient with respect to the filter weights θ is a convolution of the input with the upstream gradient (de Freitas oxf(13)):

```math
∂ℒ/∂X = δ ★ θ^{flip}        # "full" convolution (deconvolution)
∂ℒ/∂θ = X ★ δ               # valid cross-correlation
```

### Max-Pooling Layer

Max-pooling with window k×k and stride S selects the maximum activation in each local window. The forward pass is a simple argmax. The backward pass ("switch" mechanism in ZFNet) routes the upstream gradient to the location of the maximum, zeroing out all other positions:

```math
∂ℒ/∂X_{ij} = δ_{i',j'} · 𝟏[(i,j) = argmax of pool window at (i',j')]
```

### Translation Equivariance

A function f is equivariant to translation T_Δ if f(T_Δ(x)) = T_Δ(f(x)). Convolution satisfies this: shifting the input shifts the feature map by the same amount. Max-pooling provides approximate *invariance* (the output changes less than the input shift for shifts smaller than the pool stride). The combination of equivariance (detecting where a feature is) followed by invariance (suppressing exact position) is the inductive bias that makes ConvNets so effective on natural images.

> [!IMPORTANT]
> Equivariance and invariance are not the same. Convolutional layers are equivariant (the location of a detected feature moves with the image). Pooling provides invariance (small shifts are absorbed). Classification requires invariance; localisation requires equivariance. This is why detection architectures (YOLO, Faster R-CNN) preserve spatial feature maps rather than collapsing them.

### Architecture Evolution: LeNet to ResNet

**Figure:** Evolution of ConvNet architectures.

```
LeNet-5 (1989)
  Input 32×32 → Conv 5×5 → Pool → Conv 5×5 → Pool → FC → FC → Softmax
  60K parameters. Trained on MNIST/digits.

AlexNet (2012)  — ImageNet breakthrough
  Input 224×224×3 → [Conv 11×11/s4, Pool] → [Conv 5×5, Pool] →
  [3× Conv 3×3] → Pool → [FC 4096] → [FC 4096] → FC 1000
  60M parameters. ReLU, Dropout, LRN, GPU training.
  Top-5 error: 15.3% (2nd: 26.2%)

ZFNet / VGGNet (2013/2014)
  ZFNet: refined AlexNet (7×7 stride-2 layer 1 → smaller stride, better features)
  VGGNet: all 3×3 convs, depth 16–19 layers. Simple, highly regularised.

ResNet (2015)
  Skip connections h^{ℓ+2} = F(h^ℓ; θ) + h^ℓ.
  Solves gradient vanishing for 50–152 layers. Top-5: 3.57%.
```

### ConvNets for Language (1D Convolution)

The same operation extends to sequences: a 1D conv with kernel width k slides over a sequence of token embeddings, detecting local n-gram patterns. Collobert & Weston (2011) and Kalchbrenner et al. (2014) used 1D ConvNets with temporal max-pooling for sentence classification — the temporal pooling selects the most salient n-gram feature regardless of its position in the sentence.

---

## Math

### Parameter count comparison

Fully-connected layer mapping H×W×C_in to H'×W'×C_out: (H·W·C_in)·(H'·W'·C_out) parameters.

Convolutional layer: k·k·C_in·C_out parameters (+ C_out biases).

For AlexNet layer 3 (C_in = 256, C_out = 384, k = 3, H = W = 13): the convolutional layer has 3·3·256·384 ≈ 884K parameters. A fully-connected replacement would require 13²·256 × 13²·384 ≈ 4.3B parameters — ~5000× more.

### Saliency map derivation (Simonyan et al. 2014)

Given class score S_c(I) and image I_0, the first-order Taylor approximation is S_c(I) ≈ wᵀI + b where:

```math
w = ∂S_c/∂I|_{I_0}
```

The magnitude |w_ij| at pixel (i,j) indicates how much a small change to that pixel affects the class score — this is the saliency map. It requires exactly one backward pass through the classification network.

For multichannel (RGB) images, take the max across colour channels: M_ij = max_c |w_{h(i,j,c)}|.

---

## Real-world Example

A minimal ResNet-style block on CIFAR-10, with visualisation of the learned first-layer filters.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import matplotlib.pyplot as plt
from torchvision import datasets, transforms
from torch.utils.data import DataLoader
from typing import Optional

class ResBlock(nn.Module):
    """Basic residual block with two 3×3 convs and a skip connection."""

    def __init__(self, channels: int) -> None:
        super().__init__()
        self.conv1 = nn.Conv2d(channels, channels, 3, padding=1, bias=False)
        self.bn1   = nn.BatchNorm2d(channels)
        self.conv2 = nn.Conv2d(channels, channels, 3, padding=1, bias=False)
        self.bn2   = nn.BatchNorm2d(channels)

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        residual = x
        out = F.relu(self.bn1(self.conv1(x)))
        out = self.bn2(self.conv2(out))
        return F.relu(out + residual)   # skip connection


class SmallResNet(nn.Module):
    def __init__(self, num_classes: int = 10) -> None:
        super().__init__()
        self.stem = nn.Sequential(
            nn.Conv2d(3, 64, 3, padding=1, bias=False),
            nn.BatchNorm2d(64),
            nn.ReLU(),
        )
        self.layer1 = nn.Sequential(ResBlock(64), ResBlock(64))
        self.layer2 = nn.Sequential(
            nn.Conv2d(64, 128, 3, stride=2, padding=1, bias=False),
            nn.BatchNorm2d(128), nn.ReLU(),
            ResBlock(128),
        )
        self.pool   = nn.AdaptiveAvgPool2d(1)   # global average pooling
        self.fc     = nn.Linear(128, num_classes)

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        x = self.stem(x)
        x = self.layer1(x)
        x = self.layer2(x)
        x = self.pool(x).flatten(1)  # [B, 128]
        return self.fc(x)


# Training setup
transform = transforms.Compose([
    transforms.RandomHorizontalFlip(),
    transforms.RandomCrop(32, padding=4),
    transforms.ToTensor(),
    transforms.Normalize((0.4914, 0.4822, 0.4465), (0.2023, 0.1994, 0.2010)),
])
train_loader = DataLoader(
    datasets.CIFAR10("/tmp/cifar", train=True, download=True, transform=transform),
    batch_size=128, shuffle=True, num_workers=2,
)

model = SmallResNet()
optimizer = torch.optim.SGD(model.parameters(), lr=0.1, momentum=0.9, weight_decay=5e-4)
scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=100)

for epoch in range(5):  # 5 epochs for illustration; 100 needed for convergence
    for x, y in train_loader:
        optimizer.zero_grad()
        loss = F.cross_entropy(model(x), y)
        loss.backward()
        optimizer.step()
    scheduler.step()
    print(f"epoch {epoch+1}")
```

> [!TIP]
> The skip connection `out + residual` in `ResBlock.forward` is the key insight of ResNets. During backpropagation, the gradient flows through the addition gate *unchanged* — ∂(out + residual)/∂residual = 1 — creating a direct gradient highway from the loss to early layers. This is why ResNets can be trained to 152 layers without vanishing gradients.

---

## In Practice

**Stride 2 vs max-pooling for downsampling.** Springenberg et al. (2015) showed max-pooling can be replaced by strided convolution without accuracy loss on CIFAR-10/100. The "All-CNN" result (9.08% error with ~1.3M parameters, vs 9.69% for the ConvPool baseline) demonstrates that the spatial dimension reduction is the operative mechanism, not the max operation itself. Modern architectures (ConvNeXt, EfficientNet) largely follow this.

**Feature map visualisation as a debugging tool.** Zeiler & Fergus (2014) used the deconvnet to identify that AlexNet's first-layer filters were learning mixed high/low frequency patterns due to the large stride (4) — a dead giveaway from the visualisation. Reducing the stride to 2 and the filter size from 11×11 to 7×7 eliminated aliasing artifacts in layer 2 and improved accuracy by 1.7% top-5.

**Locally-connected layers for aligned faces (DeepFace).** When the input is geometrically normalised (e.g., 3D-aligned frontal face images), spatial stationarity does not hold: eye regions and nose regions have systematically different statistics. DeepFace (Taigman et al. 2014) uses locally-connected layers (no weight sharing) for layers 4–6 after the initial convolutions, achieving 97.35% on LFW. The trade-off is a large parameter increase (>95% of DeepFace's 120M parameters are in the local and FC layers).

**Softmax sparsity.** An interesting empirical observation from DeepFace: about 75% of feature components in the topmost fully-connected layer are exactly zero (due to ReLU), making the representation very sparse. Sparse activations reduce inference compute and improve generalisation by reducing co-adaptation between features.

> [!CAUTION]
> Batch normalisation during fine-tuning from a pretrained model: if you freeze BN statistics (running mean/variance from ImageNet pretraining) but fine-tune on a small dataset with very different distribution, the mismatch can cause catastrophic accuracy degradation. Set `model.eval()` to freeze BN (uses running stats) or ensure your fine-tuning dataset is large enough for BN to re-estimate meaningful statistics. The same applies when batch size < 8 — BN statistics become noisy and hurt performance.

---

## Pitfalls

- **"Convolution and cross-correlation are the same operation."** Cross-correlation slides the filter without flipping; convolution flips first. In deep learning, "convolution" means cross-correlation. This matters only if you're writing signal-processing code that must be mathematically exact.
- **"More pooling = more invariance = better."** Over-pooling loses spatial information. Several levels of max-pooling collapse the spatial dimensions to 1×1 too early, preventing the network from learning precise spatial relationships. DeepFace explicitly avoids multiple pooling to preserve fine-grained spatial structure.
- **"The deconvnet reconstructs what the network represents."** As Simonyan et al. (2014) showed, deconvnet reconstructions are *not* samples from a generative model — they are input-conditioned projections. They show which image patches most activate a feature, not the feature "in isolation."
- **"1×1 convolutions don't do anything."** A 1×1 conv with C_out filters is equivalent to a per-position linear projection across channels — the same operation as a fully-connected layer applied independently at each spatial location. It is a computationally cheap channel mixing operation used heavily in Inception modules and ResNets.
- **"Validation loss is the only metric that matters."** For ConvNets on ImageNet, watching the feature visualisations evolve during training (as Zeiler & Fergus showed for epochs 1–64) is a diagnostic signal independent of validation loss. Dead filters (grey/noisy patches with no structure) or dominated features (one filter with very high activation) indicate training problems before the loss curve shows them.

---

## Exercises

### Exercise 1

A convolutional layer has input shape [3, 224, 224] (C×H×W), uses 64 filters of size 7×7 with stride 2 and padding 3. (a) Compute the output spatial dimensions. (b) Count the parameters. (c) What is the effective receptive field of a neuron in the output feature map?

#### Solution 1

**(a)** H_out = (224 + 2·3 − 7)/2 + 1 = (224 + 6 − 7)/2 + 1 = 223/2 + 1 = 112 (integer division). So output shape: [64, 112, 112].

**(b)** Parameters = 7 × 7 × 3 × 64 + 64 (biases) = 9,408 + 64 = **9,472 parameters**.

**(c)** Each output neuron "sees" a 7×7 patch of the original input — this is the receptive field for a single convolution layer with no further layers.

### Exercise 2

Explain why AlexNet achieved 15.3% top-5 error on ImageNet 2012 while the second-place hand-crafted method achieved 26.2%. List the three architectural and training innovations that made this possible.

#### Solution 2

The three key innovations (from Krizhevsky et al. 2012 and contextualised by Zeiler & Fergus 2013):

1. **ReLU activations**: eliminate the vanishing-gradient problem of sigmoid/tanh, enabling training of 8-layer deep networks within a practical timeframe. The paper reports 6× faster training than the equivalent tanh network.
2. **GPU training with dropout regularisation**: training on 2 GTX 580 GPUs with 3GB RAM enabled using a much larger model (60M parameters). Dropout (rate 0.5 in FC layers) prevented overfitting on the 1.2M image training set.
3. **Large-scale data augmentation**: horizontal flipping, random 224×224 crops from 256×256 images (5 crops + 5 flips = 10 test variants per image), PCA-based colour jitter. Together these effectively multiplied the training set size by orders of magnitude.

The architectural depth (8 learned layers vs 2–3 in prior work) combined with the training infrastructure and the scale of ImageNet (1.2M images, 1000 classes) created the inflection point.

### Exercise 3

Derive the saliency map for an AlexNet-class network and write the PyTorch code to compute it for a given image and target class.

#### Solution 3

The saliency map is the gradient of the class score with respect to the input image, which requires only one backward pass with respect to the input rather than the parameters.

```python
import torch
import torchvision.models as models
import torchvision.transforms as transforms
from PIL import Image
import numpy as np

def compute_saliency(
    model: torch.nn.Module,
    image_tensor: torch.Tensor,   # [1, 3, H, W], already normalised
    target_class: int,
) -> np.ndarray:
    """Return saliency map shape [H, W] for the given image and class."""
    model.eval()
    x = image_tensor.requires_grad_(True)  # enable gradient w.r.t. input

    logits = model(x)                       # [1, 1000]
    score = logits[0, target_class]
    model.zero_grad()
    score.backward()

    # gradient shape: [1, 3, H, W]
    saliency = x.grad.data.abs()            # absolute gradient
    saliency, _ = saliency.max(dim=1)       # max over colour channels
    return saliency.squeeze().cpu().numpy()  # [H, W]
```

This is the Simonyan et al. (2014) "image-specific class saliency" approach. The resulting heatmap highlights pixels whose perturbation would most change the class score, providing a form of weakly-supervised object localisation.

---

## Sources

- de Freitas, N. Oxford Lecture oxf(13): Convolutional Neural Networks. 2014.
- LeCun, Y., Boser, B., Denker, J.S. et al. (1989). Backpropagation Applied to Handwritten Zip Code Recognition. *Neural Computation*, 1(4), 541–551.
- Krizhevsky, A., Sutskever, I. & Hinton, G.E. (2012). ImageNet Classification with Deep Convolutional Neural Networks. *NeurIPS*. (AlexNet)
- Zeiler, M.D. & Fergus, R. (2014). Visualizing and Understanding Convolutional Networks. *ECCV*. arXiv:1311.2901.
- Simonyan, K., Vedaldi, A. & Zisserman, A. (2014). Deep Inside Convolutional Networks: Visualising Image Classification Models and Saliency Maps. arXiv:1312.6034.
- Springenberg, J.T., Dosovitskiy, A., Brox, T. & Riedmiller, M. (2015). Striving for Simplicity: The All Convolutional Net. arXiv:1412.6806. ICLR 2015 workshop.
- Taigman, Y., Yang, M., Ranzato, M.A. & Wolf, L. (2014). DeepFace: Closing the Gap to Human-Level Performance in Face Verification. *CVPR*.
- Conversation with user on 2026-05-19.

---

## Related

- [1 - Neural Networks Foundations](./1-neural-networks-foundations.md)
- [2 - Backpropagation](./2-backpropagation.md)
- [5 - Deep Architecture Principles](./5-deep-architecture-principles.md)
- [2 - Language Models: Autoregressive vs Masked](../../AI-Engineering/1-foundations/2-language-models.md)
