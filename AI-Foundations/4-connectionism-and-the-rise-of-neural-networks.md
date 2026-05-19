---
title: "4 - Connectionism and the Rise of Neural Networks"
created: 2026-05-19
updated: 2026-05-19
tags: []
aliases: []
---

# 4 - Connectionism and the Rise of Neural Networks

[toc]

> **TL;DR:** Connectionism is the paradigm that models cognition as emergent patterns in networks of simple, neuron-like units rather than as the manipulation of explicit symbols. Its intellectual genealogy runs from McCulloch and Pitts (1943) through Rosenblatt's perceptron (1958), the Minsky-Papert XOR crisis (1969), the PDP revival and backpropagation (1986), and finally the deep learning explosion anchored by AlexNet (2012). Each wave was driven by a new mechanism — logical neurons, adaptive learning, multi-layer backprop, scale and GPUs — and each revealed new limitations. Understanding the oscillation between symbolic and connectionist AI is necessary context for evaluating where the transformer era currently stands.

## Vocabulary

**McCulloch-Pitts neuron** — A formal model of a biological neuron: it receives n binary inputs with associated excitatory or inhibitory weights, sums them, and fires (outputs 1) iff the sum exceeds a threshold. Individual MP neurons can implement AND, OR, and NOT; networks can implement any propositional logic function.

---

**Perceptron** — Rosenblatt's (1957) learning machine: a single-layer McCulloch-Pitts-style network with real-valued weights updated by the Perceptron Learning Rule. Provably converges on linearly separable data.

---

**Perceptron convergence theorem** — If a set of training examples is linearly separable, the Perceptron Learning Rule will find a separating hyperplane in a finite number of updates.

---

**XOR problem** — A binary classification task where the positive class is {(0,1), (1,0)} and the negative class is {(0,0), (1,1)}. No linear decision boundary separates the classes, so single-layer perceptrons cannot solve it.

---

**Backpropagation** — The algorithm for computing gradients of a loss function with respect to all weights in a multi-layer network by applying the chain rule backwards through the computational graph. Enables efficient training of multi-layer networks.

---

**Activation function** — A non-linear function applied element-wise to a neuron's pre-activation sum. Without non-linearity, any stack of linear layers collapses to a single linear map. Common examples: sigmoid, tanh, ReLU.

---

**Hidden layer** — A layer of neurons between the input and output layers. Hidden layers learn internal representations not specified by the programmer. Their existence is what allows multi-layer networks to solve XOR and, more generally, to approximate arbitrary functions.

---

**Universal approximation theorem** — A multi-layer feedforward network with at least one hidden layer and a non-polynomial activation function can approximate any continuous function on a compact domain to arbitrary precision, given sufficient width. (Hornik, Cybenko, 1989)

---

**PDP (Parallel Distributed Processing)** — The connectionist research programme formalized by Rumelhart, McClelland, and the PDP Research Group in two volumes (1986). The core claim: cognitive processes are best understood as the emergent collective behavior of large numbers of simple interacting processing units.

---

**Boltzmann machine** — A stochastic generalization of Hopfield networks with hidden units, allowing unsupervised learning of probability distributions over binary patterns. Proposed by Hinton and Sejnowski in PDP Vol. 1. Trained by minimizing the difference between "waking" and "sleeping" statistics.

---

**Convolutional neural network (CNN)** — A network architecture exploiting translation equivariance by using weight-shared convolutional filters across the spatial input domain. LeCun et al. applied CNNs to handwriting recognition in the 1980s-1990s.

---

**AlexNet** — Krizhevsky, Sutskever, and Hinton's 2012 deep CNN that achieved 16.4% top-5 error on ImageNet-2012 (vs. 26.2% by the runner-up). The event that publicly established deep learning as the dominant paradigm.

---

## Intuition

The connectionist intuition is that intelligence is not in the rules — it is in the weights. A network of simple units, when trained on examples, develops internal representations that capture statistical regularities in the data. No one programs these representations; they emerge from learning.

The geometric picture: a single neuron computes a hyperplane decision boundary in the input space. Multiple neurons in one layer tile the space with hyperplanes. A hidden layer applies non-linearities to these tiles, creating curved boundaries. More layers create hierarchical compositions of boundaries. Deep networks can represent exponentially complex functions (in depth) that would require exponentially wide shallow networks.

The biological picture is rougher: real neurons fire asynchronously, use spike rates rather than clean binary outputs, have dendritic computation, synaptic dynamics, and neuromodulation that none of these models capture. McCulloch and Pitts knew this and were explicit that their model captured only the logical structure, not the biophysics.

## How it Works

### McCulloch-Pitts Neurons (1943)

McCulloch and Pitts modelled the neuron as a threshold logic unit. Each neuron c has excitatory inputs (which add to the net input) and inhibitory inputs (which absolutely prevent firing). The neuron fires at time t if: the net excitatory input at t-1 exceeds the threshold, and no inhibitory input fired at t-1.

The key theorems of the 1943 paper are (expressed informally):
1. Any network without cycles (acyclic net, order 0) is equivalent to a temporal propositional expression (TPE).
2. Any TPE can be realized by a net of order 0.
3. Any net with cycles can be described in terms of recursive predicates over time.

This means MP neuron networks are exactly as powerful as propositional logic (for acyclic nets) and can realize any finitely describable temporal behavior (for nets with cycles). It also means they are Turing complete when combined with feedback.

```python
import numpy as np

def mp_neuron(inputs: list[int], weights: list[int], threshold: int,
              inhibitory: list[int] | None = None) -> int:
    """
    McCulloch-Pitts neuron (1943).

    inputs:     binary input values (0 or 1)
    weights:    +1 for excitatory, treated as multiplier
    threshold:  minimum net input to fire
    inhibitory: list of input indices that absolutely prevent firing

    Returns 1 if neuron fires, 0 otherwise.
    """
    # Absolute inhibition: any inhibitory input prevents firing
    if inhibitory:
        for i in inhibitory:
            if inputs[i] == 1:
                return 0

    net = sum(w * x for w, x in zip(weights, inputs))
    return 1 if net >= threshold else 0

# AND gate: fires only when both inputs are 1
and_gate = lambda x1, x2: mp_neuron([x1, x2], [1, 1], threshold=2)

# OR gate: fires when either input is 1
or_gate = lambda x1, x2: mp_neuron([x1, x2], [1, 1], threshold=1)

# NOT gate: fires when input is 0 (inhibitory connection)
not_gate = lambda x: mp_neuron([x], [1], threshold=1, inhibitory=[0])

# Verify truth tables
for x1, x2 in [(0,0),(0,1),(1,0),(1,1)]:
    print(f"AND({x1},{x2})={and_gate(x1,x2)}  OR({x1},{x2})={or_gate(x1,x2)}")
```

### The Perceptron and Learning (1957–1969)

Rosenblatt added the critical ingredient that McCulloch and Pitts lacked: *learning*. The Perceptron Learning Rule adjusts weights by the error signal — if the output is wrong, the weights move in the direction that would have produced the correct output.

The Perceptron convergence theorem guarantees that if a linear decision boundary exists, the algorithm will find it. This gave early AI enormous optimism about learning machines. The convergence theorem does not guarantee rate of convergence or generalization to new examples.

```python
import numpy as np

class Perceptron:
    """
    Rosenblatt 1957 perceptron with binary threshold output.
    Convergence theorem: if data is linearly separable, this halts.
    """
    def __init__(self, n_inputs: int, lr: float = 0.1):
        self.weights = np.zeros(n_inputs)
        self.bias = 0.0
        self.lr = lr

    def predict(self, x: np.ndarray) -> int:
        return 1 if np.dot(self.weights, x) + self.bias > 0 else 0

    def fit(self, X: np.ndarray, y: np.ndarray, epochs: int = 100) -> None:
        for epoch in range(epochs):
            errors = 0
            for xi, yi in zip(X, y):
                pred = self.predict(xi)
                err = yi - pred
                if err != 0:
                    self.weights += self.lr * err * xi
                    self.bias += self.lr * err
                    errors += 1
            if errors == 0:
                print(f"Converged at epoch {epoch+1}")
                return
        print("Did not converge")

# AND gate: linearly separable — converges
X = np.array([[0,0],[0,1],[1,0],[1,1]])
y_and = np.array([0, 0, 0, 1])
p_and = Perceptron(n_inputs=2)
p_and.fit(X, y_and)
print("AND gate weights:", p_and.weights, "bias:", p_and.bias)

# XOR gate: NOT linearly separable — will not converge
y_xor = np.array([0, 1, 1, 0])
p_xor = Perceptron(n_inputs=2)
p_xor.fit(X, y_xor, epochs=1000)
print("XOR predictions:", [p_xor.predict(x) for x in X])  # Wrong
```

### The XOR Crisis and First Neural Network Winter (1969)

Minsky and Papert's 1969 *Perceptrons* proved rigorously that single-layer perceptrons cannot compute XOR, cannot recognize connectivity of patterns, and fail on many geometric tasks. Their analysis was mathematically sound. The policy response — effectively halting neural network funding — was an overreaction. Their book noted that multi-layer networks could, in principle, solve these problems, but that no training algorithm was known.

The key gap: backpropagation had not been derived yet for multi-layer networks. The credit assignment problem — how to attribute error to weights in hidden layers — was unsolved for the general case.

> [!IMPORTANT]
> The XOR failure is a property of *single-layer* linear classifiers. The correct lesson is: any learning machine must have sufficient capacity to represent the hypothesis class of the target function. XOR requires at minimum one hidden layer; it does not require deep networks. The Minsky-Papert book itself acknowledged this on page 232. The defunding was a consequence of the political climate around AI's failed promises, not a logical deduction from the math.

### Backpropagation and the PDP Revival (1986)

The PDP volumes (1986) brought connectionism back from the margins. The key paper is Rumelhart, Hinton, and Williams' "Learning Representations by Back-Propagating Errors." The algorithm applies the chain rule of calculus to compute the gradient of the loss with respect to every weight in the network, enabling gradient descent on multi-layer architectures.

```python
import numpy as np

def sigmoid(x: np.ndarray) -> np.ndarray:
    return 1.0 / (1.0 + np.exp(-x))

def sigmoid_deriv(x: np.ndarray) -> np.ndarray:
    s = sigmoid(x)
    return s * (1.0 - s)

class TwoLayerMLP:
    """
    Two-layer MLP solving XOR. Demonstrates backpropagation.
    Architecture: input(2) -> hidden(4) -> output(1)
    """
    def __init__(self, lr: float = 0.5, seed: int = 42):
        rng = np.random.default_rng(seed)
        self.W1 = rng.normal(0, 0.5, (4, 2))   # (hidden, input)
        self.b1 = np.zeros(4)
        self.W2 = rng.normal(0, 0.5, (1, 4))   # (output, hidden)
        self.b2 = np.zeros(1)
        self.lr = lr

    def forward(self, x: np.ndarray) -> tuple[np.ndarray, ...]:
        z1 = self.W1 @ x + self.b1      # (4,)
        a1 = sigmoid(z1)                # (4,)
        z2 = self.W2 @ a1 + self.b2     # (1,)
        a2 = sigmoid(z2)                # (1,) — prediction
        return z1, a1, z2, a2

    def backward(self, x: np.ndarray, y: float,
                 z1: np.ndarray, a1: np.ndarray,
                 z2: np.ndarray, a2: np.ndarray) -> None:
        # Output layer gradient
        delta2 = (a2 - y) * sigmoid_deriv(z2)   # (1,)
        # Hidden layer gradient (chain rule through W2)
        delta1 = (self.W2.T @ delta2) * sigmoid_deriv(z1)   # (4,)

        # Weight updates (gradient descent)
        self.W2 -= self.lr * np.outer(delta2, a1)
        self.b2 -= self.lr * delta2
        self.W1 -= self.lr * np.outer(delta1, x)
        self.b1 -= self.lr * delta1

    def train(self, X: np.ndarray, y: np.ndarray, epochs: int = 5000) -> None:
        for epoch in range(epochs):
            for xi, yi in zip(X, y):
                z1, a1, z2, a2 = self.forward(xi)
                self.backward(xi, yi, z1, a1, z2, a2)

# XOR dataset
X_xor = np.array([[0,0],[0,1],[1,0],[1,1]], dtype=float)
y_xor = np.array([0, 1, 1, 0], dtype=float)

mlp = TwoLayerMLP(lr=0.5)
mlp.train(X_xor, y_xor, epochs=10000)

print("XOR predictions after training:")
for xi, yi in zip(X_xor, y_xor):
    _, _, _, pred = mlp.forward(xi)
    print(f"  {xi} -> pred={pred[0]:.3f}  true={yi:.0f}  "
          f"{'CORRECT' if round(pred[0]) == yi else 'WRONG'}")
```

> [!TIP]
> In PyTorch, the entire backpropagation above is replaced by `loss.backward()`. PyTorch builds a dynamic computation graph during the forward pass and traverses it in reverse. The delta2 and delta1 calculations above are exactly what `autograd` computes. Understanding the manual derivation makes debugging gradient flow issues — vanishing gradients, dead ReLUs, exploding gradients — tractable.

### Boltzmann Machines and Unsupervised Learning

Hinton and Sejnowski's Boltzmann machine (PDP Vol. 1, Ch. 7) extended the connectionist programme to unsupervised learning. A Boltzmann machine is a stochastic network of binary units where the joint distribution follows a Boltzmann distribution parameterized by the pairwise weights.

The energy of a state s is:

```math
E(\mathbf{s}) = -\sum_{i < j} w_{ij} s_i s_j - \sum_i b_i s_i
```

The probability of state s is:

```math
P(\mathbf{s}) = \frac{1}{Z} \exp(-E(\mathbf{s}))
```

where Z is the partition function. Training by contrastive divergence minimizes the difference between data statistics and model statistics. The Boltzmann machine with hidden units (Restricted Boltzmann Machine, Smolensky 1986 / Hinton & Sejnowski 1986) became the workhorse of the 2006 deep belief network revival.

## Math

The universal approximation theorem (Cybenko 1989, Hornik 1991): For any continuous function f on a compact subset of ℝⁿ and any ε > 0, there exists a single-hidden-layer feedforward network with sigmoid activations that approximates f to within ε in the sup norm. The key parameters:

```math
\hat{f}(\mathbf{x}) = \sum_{j=1}^{N} \alpha_j \sigma\!\left(\mathbf{w}_j^\top \mathbf{x} + b_j\right)
```

where N is the (possibly large) number of hidden units. The theorem guarantees existence, not learnability or sample efficiency. Deep networks achieve the same approximation power with exponentially fewer units in many function classes — this is the expressiveness argument for depth.

The backpropagation update rule for weight w connecting hidden unit j to output unit k:

```math
\frac{\partial \mathcal{L}}{\partial w_{jk}} = \delta_k \cdot a_j
```

where delta_k is the error signal at unit k and a_j is the activation of unit j. The chain rule propagates delta back through the network: for hidden unit j, delta_j = (sum_k w_jk * delta_k) * sigma'(z_j).

## Real-world Example

A modern PyTorch implementation of a deep MLP trained on a non-linear classification problem, showing how the ideas of the PDP era become production code.

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader, TensorDataset
import numpy as np

# Generate a non-linearly separable dataset (two concentric circles)
np.random.seed(0)
N = 500
theta = np.random.uniform(0, 2 * np.pi, N)
r_inner = 0.5 + 0.1 * np.random.randn(N // 2)
r_outer = 1.5 + 0.1 * np.random.randn(N // 2)
r = np.concatenate([r_inner, r_outer])
theta_all = np.concatenate([theta[:N//2], theta[N//2:]])
X = np.stack([r * np.cos(theta_all), r * np.sin(theta_all)], axis=1)
y = np.concatenate([np.zeros(N//2), np.ones(N//2)])

X_t = torch.tensor(X, dtype=torch.float32)
y_t = torch.tensor(y, dtype=torch.float32).unsqueeze(1)
loader = DataLoader(TensorDataset(X_t, y_t), batch_size=64, shuffle=True)

# Deep MLP: same topology as PDP-era networks, modern activation + optimizer
class MLP(nn.Module):
    def __init__(self) -> None:
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(2, 64),
            nn.ReLU(),
            nn.Linear(64, 64),
            nn.ReLU(),
            nn.Linear(64, 1),
            nn.Sigmoid(),
        )

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        return self.net(x)

model = MLP()
optimizer = optim.Adam(model.parameters(), lr=1e-3)
criterion = nn.BCELoss()

for epoch in range(50):
    total_loss = 0.0
    for X_batch, y_batch in loader:
        optimizer.zero_grad()
        preds = model(X_batch)
        loss = criterion(preds, y_batch)
        loss.backward()          # backpropagation — chain rule through the graph
        optimizer.step()
        total_loss += loss.item()
    if (epoch + 1) % 10 == 0:
        print(f"Epoch {epoch+1:3d}  loss={total_loss/len(loader):.4f}")

# Evaluate
with torch.no_grad():
    preds = (model(X_t) > 0.5).float()
    acc = (preds == y_t).float().mean().item()
    print(f"Accuracy: {acc:.3f}")
```

> [!NOTE]
> ReLU (Rectified Linear Unit) replaced sigmoid as the dominant hidden-layer activation after 2012 because sigmoid suffers from vanishing gradients in deep networks — the sigmoid derivative is at most 0.25, so gradients shrink exponentially with depth. ReLU's derivative is 1 for positive pre-activations, preventing this. The PDP-era networks with sigmoid are conceptually equivalent but practically unstable beyond about 4 layers.

## In Practice

**The compute hypothesis.** Each wave of connectionism was gated by available compute. McCulloch-Pitts neurons were analyzed mathematically, not simulated. Rosenblatt's Mark I Perceptron was hardware (photocells + potentiometers on a 20x20 grid). The PDP revival could only demonstrate small networks on toy cognitive tasks. AlexNet ran on two GTX 580 GPUs (3 GB VRAM each). GPT-3 required roughly 3 × 10²³ FLOP of training compute. The empirical scaling laws (Kaplan et al. 2020) formalized the observation that loss decreases as a power law in compute, model size, and data — making future capability gains partially predictable.

**Vanishing and exploding gradients.** Deep networks trained with backpropagation suffer from gradients that either shrink to zero (vanishing) or grow without bound (exploding) as they propagate through many layers. Solutions developed incrementally: ReLU activations (2010), batch normalization (2015), residual connections (He et al., 2016), layer normalization (Ba et al., 2016). The transformer's residual stream + layer normalization directly addresses this.

**The representation learning hypothesis.** The PDP programme's central claim — that hidden layers learn useful internal representations — is now empirically well-supported. Learned representations transfer across tasks (ImageNet → detection, NLP → question answering), suggesting that the representations encode genuinely domain-relevant structure rather than overfitted task-specific features.

> [!WARNING]
> Universal approximation says a wide enough shallow network can approximate any continuous function, but says nothing about how to find the weights or how many training examples are needed. In practice, deep networks with appropriate architecture achieve the same representational power with far fewer parameters and far better generalization than the theoretically equivalent wide shallow network. Citing universal approximation as evidence that "one hidden layer is enough" is a misuse of the theorem.

> [!CAUTION]
> Backpropagation computes exact gradients only for the declared computational graph. Any operation that is not differentiable (argmax, sampling without reparameterization, discrete selection) silently breaks the gradient flow. The gradient of zero will be computed with no error — the weights simply will not update. Always verify gradient flow on new architectures with a gradient check or by confirming that loss decreases.

## Pitfalls

- **"The PDP book invented backpropagation."** — Backpropagation's chain-rule derivation for multi-layer networks was published independently by Werbos (1974, in his PhD thesis), by Bryson and Ho (1969) in control theory, and by Dreyfus (1962). The PDP group popularized it, demonstrated it on cognitive tasks, and sparked the revival — but did not invent it.
- **"Deep learning works because it mimics the brain."** — Biological neurons are far more complex than ReLU units. Biological learning uses spike-timing-dependent plasticity, neuromodulators, and local learning rules, not global backpropagation via a global error signal. Deep learning is *inspired by* neuroscience in its architecture metaphors; its actual mechanisms have no known biological substrate.
- **"More layers is always better."** — Depth helps when the function class has hierarchical structure (which natural images and language do). For tabular data with no hierarchical structure, gradient boosted trees (XGBoost) routinely outperform deep networks. Architecture choice should match the inductive biases of the problem.
- **"The 1986 backpropagation paper was a theoretical breakthrough."** — It was primarily an empirical demonstration: multi-layer networks with backpropagation learn useful internal representations on real cognitive tasks. The theoretical analysis (convergence, generalization) came later and remains incomplete.

## Exercises

### Exercise 1 — McCulloch-Pitts logic gates

Implement NAND as a McCulloch-Pitts neuron (weights and threshold). Then argue why NAND is functionally complete.

#### Solution 1

NAND fires unless both inputs are 1. Using the MP model with inhibition: set both inputs as inhibitory with threshold 0. Alternatively, using excitatory weights: NAND(x1, x2) = NOT(AND(x1, x2)).

With excitatory weights and a threshold-based model: weights = [-1, -1], threshold = -1. If x1 = x2 = 1, net = -2 < -1, does not fire (output 0). Otherwise net ≥ -1, fires (output 1).

NAND is functionally complete because: NOT(x) = NAND(x, x); AND(x,y) = NOT(NAND(x,y)) = NAND(NAND(x,y), NAND(x,y)); OR(x,y) = NAND(NOT(x), NOT(y)) = NAND(NAND(x,x), NAND(y,y)). Since {NOT, AND, OR} is a complete basis for Boolean algebra, and all three are constructible from NAND, NAND is complete.

### Exercise 2 — Perceptron convergence

The Perceptron convergence theorem guarantees termination on linearly separable data. Give a counterexample or explain why convergence is not guaranteed when data is not linearly separable.

#### Solution 2

If data is not linearly separable, the perceptron can cycle indefinitely. Consider XOR: any hyperplane that correctly classifies (0,0) and (1,1) as negative will incorrectly classify some positive example. Updating the weights to fix that error will break the previously correct classifications. The algorithm will oscillate without ever reaching a state where all examples are correctly classified. There is no weight vector w that satisfies all constraints simultaneously, so the guarantee (which assumes such a vector exists) fails.

In practice, you observe oscillating weights, non-decreasing cumulative error over epochs, and loss that refuses to converge to zero. The fix is either a non-linear model (multi-layer network) or a softer objective (logistic regression, which finds the best linear separator without requiring perfect classification).

### Exercise 3 — Backpropagation chain rule

For a two-layer network with loss L = (a2 - y)^2 / 2, a2 = sigmoid(z2), z2 = w2 * a1 + b2, a1 = sigmoid(z1), z1 = w1 * x + b1, derive dL/dw1 using the chain rule.

#### Solution 3

Applying the chain rule step by step:

```math
\frac{dL}{dw_1} = \frac{dL}{da_2} \cdot \frac{da_2}{dz_2} \cdot \frac{dz_2}{da_1} \cdot \frac{da_1}{dz_1} \cdot \frac{dz_1}{dw_1}
```

Each factor:
- dL/da2 = (a2 - y)
- da2/dz2 = sigmoid'(z2) = a2(1 - a2)
- dz2/da1 = w2
- da1/dz1 = sigmoid'(z1) = a1(1 - a1)
- dz1/dw1 = x

Combined:

```math
\frac{dL}{dw_1} = (a_2 - y) \cdot a_2(1 - a_2) \cdot w_2 \cdot a_1(1 - a_1) \cdot x
```

The (a2 - y) * a2(1-a2) term is the output error delta2. The w2 * a1(1-a1) factor is how the output error propagates back through w2 into the hidden layer — this is the "back-propagated error" that gives the algorithm its name.

### Exercise 4 — Vanishing gradients with sigmoid

For a 5-layer network with sigmoid activations, estimate the magnitude of the gradient at layer 1 relative to the gradient at layer 5, assuming all activations are near 0.5 (maximum sigmoid derivative).

#### Solution 4

The sigmoid derivative sigma'(z) = sigma(z)(1-sigma(z)) is maximized at z=0 where sigma(0) = 0.5, giving sigma'(0) = 0.25. The backpropagation gradient for layer l is multiplied by sigma'(z_l) at each backward step. Through 4 hidden layers: gradient_at_layer_1 ≈ gradient_at_layer_5 * (0.25)^4 = gradient_at_layer_5 * 0.0039, roughly 1/256th the magnitude.

At realistic operating points (e.g., sigmoid(z) ≈ 0.9 for a confidently active unit), sigma'(z) = 0.9 * 0.1 = 0.09. Through 4 layers: 0.09^4 ≈ 6.6 × 10^-5 — the first layer receives a gradient nearly 15,000 times smaller than the last layer. This makes early layers learn extremely slowly, which is why deep sigmoid networks were essentially untrainable before ReLU and residual connections.

### Exercise 5 — PDP and modern LLMs

The PDP programme argued for distributed representations over local representations. How is this principle reflected in the transformer architecture?

#### Solution 5

In a local representation, each concept is encoded in a single unit (neuron or symbol). In a distributed representation, each concept is encoded across a pattern of activation over many units, and each unit participates in many concepts. PDP argued that distributed representations enable graceful degradation, generalization, and efficient use of representational capacity.

Transformer token embeddings are distributed: a single token like "bank" is not a single neuron but a 768- or 4096-dimensional dense vector that encodes its meaning distributed across all dimensions. Attention heads implement distributed feature selection: each head is a different "perspective" on the input, and the output is a weighted sum over all positions (a distributed aggregation). The feed-forward network in each transformer block acts as a key-value memory (Geva et al. 2021) where the first layer "looks up" patterns distributed across neurons and the second layer writes them into the residual stream.

The PDP research group would recognize the distributed representation principle in transformers, but the scale — trillions of parameters, quadratic attention, RLHF fine-tuning — would be entirely unfamiliar.

## Sources

- McCulloch, W. S., & Pitts, W. (1943). A Logical Calculus of the Ideas Immanent in Nervous Activity. *Bulletin of Mathematical Biophysics* 5, 115–133.
- Rosenblatt, F. (1958). The Perceptron: A Probabilistic Model for Information Storage and Organization in the Brain. *Psychological Review* 65(6), 386–408.
- Minsky, M., & Papert, S. (1969). *Perceptrons*. MIT Press.
- Rumelhart, D. E., Hinton, G. E., & Williams, R. J. (1986). Learning Representations by Back-Propagating Errors. *Nature* 323, 533–536.
- Rumelhart, D. E., & McClelland, J. L. (1986). *Parallel Distributed Processing*, Vol. 1: Foundations. MIT Press. [Including Hinton & Sejnowski, Ch. 7: Boltzmann Machines]
- Hinton, G. E., Osindero, S., & Teh, Y.-W. (2006). A Fast Learning Algorithm for Deep Belief Nets. *Neural Computation* 18(7), 1527–1554.
- Krizhevsky, A., Sutskever, I., & Hinton, G. E. (2012). ImageNet Classification with Deep Convolutional Neural Networks. *NeurIPS* 25.
- Cybenko, G. (1989). Approximation by Superpositions of a Sigmoidal Function. *Mathematics of Control, Signals and Systems* 2(4), 303–314.
- Privacy Foundation (University of Denver). History of Artificial Intelligence — 1969 backpropagation entry.

## Related

- [1 - History of AI](./1-history-of-ai.md)
- [3 - Symbolic AI and Knowledge Representation](./3-symbolic-ai-and-knowledge-representation.md)
- [5 - Embodiment and the Brooks Revolution](./5-embodiment-and-the-brooks-revolution.md)
- [1 - What is ML and Version Space](../Machine-Learning/1-foundations/1-what-is-ml-and-version-space.md)
- [1 - What is Linear Algebra](../Mathematics/Linear-Algebra/1-what-is-linear-algebra.md)
- [Generative AI Fundamentals](../AI-Engineering/1-foundations/3-generative-ai-fundamentals.md)
