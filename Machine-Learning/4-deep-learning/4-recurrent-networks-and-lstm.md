---
title: "4 - Recurrent Networks and LSTM"
created: 2026-05-19
updated: 2026-05-19
tags: []
aliases: []
---

# 4 - Recurrent Networks and LSTM

[toc]

> **TL;DR:** Recurrent Neural Networks (RNNs) process sequences by maintaining a hidden state that summarises history, making them the first neural approach to variable-length sequential modelling. Vanilla RNNs fail on long-range dependencies because backpropagation through time (BPTT) multiplies gradients by the recurrent weight matrix at every step, causing them to vanish or explode exponentially. The Long Short-Term Memory (LSTM) architecture solves this with gated memory cells whose cell state provides a near-gradient-preserving highway, enabling learning of dependencies spanning hundreds of steps.

---

## Vocabulary

- **Recurrent Neural Network (RNN)** — a network with a directed cycle: the hidden state h_t depends on the current input x_t and the previous hidden state h_{t−1}.
- **Hidden state (h_t)** — the RNN's internal summary of the sequence up to time t. Analogous to a memory register.
- **Backpropagation Through Time (BPTT)** — the algorithm for computing gradients in an RNN by unrolling the recurrence over T timesteps and applying standard backprop to the resulting computation graph.
- **Vanishing gradient** — in BPTT, the gradient of the loss at step t with respect to h_k (k ≪ t) decays as (γ_θ · γ_φ)^{t−k}, where γ_θ, γ_φ are the spectral norms of the recurrent weight matrix and the activation derivative. For γ < 1, this becomes negligibly small.
- **Exploding gradient** — when γ_θ > 1, gradients grow exponentially. Addressed by gradient clipping.
- **LSTM** — Long Short-Term Memory (Hochreiter & Schmidhuber 1997). An RNN architecture with four gated components: input gate i_t, forget gate f_t, output gate o_t, and cell input g_t, operating on a cell state c_t.
- **Cell state (c_t)** — the LSTM's "conveyor belt": a linear combination of the previous cell state (scaled by f_t) and new candidate values (scaled by i_t). The gradient highway that prevents vanishing.
- **Forget gate (f_t)** — a sigmoid gate that decides what fraction of c_{t−1} to retain. f_t = 1 means perfect memory; f_t = 0 means complete forgetting.
- **Input gate (i_t)** — controls what fraction of the new candidate g_t to write into the cell state.
- **Output gate (o_t)** — controls what part of the cell state is exposed as the hidden state h_t.
- **GRU (Gated Recurrent Unit)** — a simplified LSTM variant (Cho et al. 2014) with only two gates: reset and update. Fewer parameters than LSTM; competitive performance on many tasks.
- **Seq2seq** — the encoder-decoder architecture (Sutskever et al. 2014) that maps a variable-length input sequence to a variable-length output sequence using two RNNs. The foundation of neural machine translation.
- **Gradient clipping** — if ||∇θ||₂ > τ, rescale the gradient to τ · ∇θ / ||∇θ||₂. Prevents exploding gradients from destabilising training.
- **Teacher forcing** — during training, feeding the ground-truth token y_{t−1} as the decoder input at step t, rather than the model's own prediction. Accelerates convergence but can cause exposure bias at test time.

---

## Intuition

Imagine reading this sentence and needing to predict the next word. You are not starting from scratch at each word — you have a running internal representation of everything you've read. The RNN formalises this as a vector h_t, updated at each step as a function of the current word and the previous state. The problem: after 50 steps, updating h_t requires differentiating through 50 nested applications of the same function. If that function contracts slightly (derivative < 1), the gradient from step 50 to step 1 is essentially zero. The LSTM sidesteps this by maintaining a *separate* cell state c_t that is updated by addition (c_t = f_t ⊙ c_{t−1} + i_t ⊙ g_t), not by a saturating nonlinearity. Addition is linear; its derivative is 1. So the gradient of the loss with respect to c_{t−100} can still be O(1) if the forget gate f_t ≈ 1 throughout.

The architectural analogy: the cell state is a resistor-less wire connecting t=0 to t=T; the gates are switches that selectively write to and read from that wire.

---

## How it works

### Vanilla RNN Forward Pass

The vanilla RNN updates its hidden state at each step using a single affine transformation followed by a nonlinearity. Both the input and the previous hidden state are combined in the same matrix operation:

```math
h_t = tanh(W_hh · h_{t−1} + W_xh · x_t + b_h)
ŷ_t = softmax(W_hy · h_t + b_y)
```

Here W_hh ∈ ℝ^{d×d}, W_xh ∈ ℝ^{d×p}, W_hy ∈ ℝ^{C×d}. All timesteps share the same weights — this is the parameter sharing that allows generalisation across sequence lengths.

### BPTT and the Vanishing Gradient

To train the RNN with gradient-based methods, the recurrence must be unrolled over T timesteps. The gradient of the loss at step t with respect to the hidden state at step k < t is:

```math
∂h_t/∂h_k = Π_{j=k+1}^{t} (W_hh)ᵀ · diag(σ'(u_j))
```

This is a product of T−k matrices. If the spectral norm γ_θ of W_hh and the maximum derivative γ_φ of tanh satisfy γ_θ · γ_φ < 1, then:

```math
||∂h_t/∂h_k|| ≤ (γ_θ · γ_φ)^{t−k}  → 0 exponentially as t−k grows
```

For tanh, γ_φ ≤ 1. For W_hh with spectral norm γ_θ < 1, the product decays geometrically. This is the vanishing gradient — dependencies more than ~10 steps in the past produce negligible gradients.

> [!IMPORTANT]
> Vanishing gradient is a problem of gradient flow through time, not through depth. A 1-layer RNN unrolled over 100 steps has the same vanishing-gradient problem as a 100-layer MLP with tied weights. The fix (LSTM) addresses the time direction, not the layer direction (that is addressed by ResNets).

### LSTM: the Six Equations

The LSTM computes at each timestep t (given x_t and the previous state (h_{t−1}, c_{t−1})):

```math
i_t = σ(W_i · [h_{t−1}, x_t] + b_i)        # input gate
f_t = σ(W_f · [h_{t−1}, x_t] + b_f)        # forget gate
o_t = σ(W_o · [h_{t−1}, x_t] + b_o)        # output gate
g_t = tanh(W_g · [h_{t−1}, x_t] + b_g)     # cell input (candidate values)
c_t = f_t ⊙ c_{t−1} + i_t ⊙ g_t           # cell state update
h_t = o_t ⊙ tanh(c_t)                       # hidden state output
```

Here σ is the sigmoid function, ⊙ is elementwise multiplication, and [h_{t−1}, x_t] is the concatenated vector. There are 4 weight matrices (each ℝ^{d×(d+p)}) and 4 bias vectors — roughly 4× more parameters than a vanilla RNN of the same hidden dim.

The critical equation is c_t = f_t ⊙ c_{t−1} + i_t ⊙ g_t. This is a *linear recurrence* in c_t: the cell state changes by addition. Its gradient with respect to c_{t−1} is simply diag(f_t), which is a scaled identity — no matrix multiply, no saturating nonlinearity in the path of gradient flow.

### Backward Through the Entry-wise Multiplication Layer

The gradient through the elementwise product c_t = f_t ⊙ c_{t−1} + i_t ⊙ g_t is:

```math
∂ℒ/∂c_{t−1} = (∂ℒ/∂c_t) ⊙ f_t
```

If f_t ≈ 1 (the forget gate is open), this is approximately the identity operation — the gradient passes unchanged from c_t to c_{t−1}. This is the "gradient highway."

### GRU: Simplified Gating

The GRU (Cho et al. 2014) uses two gates: a reset gate r_t (controls how much of h_{t−1} to ignore when computing the candidate) and an update gate z_t (interpolates between h_{t−1} and the candidate):

```math
r_t = σ(W_r · [h_{t−1}, x_t])
z_t = σ(W_z · [h_{t−1}, x_t])
n_t = tanh(W_n · [r_t ⊙ h_{t−1}, x_t])   # candidate
h_t = (1 − z_t) ⊙ h_{t−1} + z_t ⊙ n_t
```

The GRU has no separate cell state — the hidden state itself serves both roles. It performs comparably to LSTM on most tasks with fewer parameters.

### Seq2seq Architecture

Sutskever et al. (2014) demonstrated that two LSTMs could learn to map variable-length input sequences to variable-length output sequences. The encoder LSTM reads the input sequence and compresses it into a fixed-length context vector (the final hidden and cell states). The decoder LSTM generates the output sequence token by token, conditioned on that context vector:

```
Encoder: reads (x_1, …, x_T) → produces context (h_T, c_T)
Decoder: generates (y_1, …, y_{T'}) one token at a time, initialised with (h_T, c_T)
```

This architecture achieved a 4.5-point BLEU improvement on WMT English-to-French translation, establishing the neural MT paradigm. Reversing the input sequence (a simple trick) improved performance further by putting the most recent input tokens closer in time to the start of decoding.

---

## Math

### Truncated BPTT

For sequences of length T, full BPTT requires O(T) memory for storing all hidden states. Truncated BPTT divides the sequence into chunks of length k and only backpropagates through the most recent k steps. Gradients from earlier steps are not propagated, trading exact gradients for feasibility. In practice k = 30–50 is commonly used.

### Initialisation trick for LSTM forget gate

A common initialisation trick (recommended by Jozefowicz et al. 2015): initialise the forget gate bias b_f ≈ 1 rather than 0. This starts the forget gate near 1 (open), encouraging the cell state to remember more of its history at the start of training, which helps the LSTM learn long-range dependencies early.

### Gradient clipping formula

```math
if ||g||₂ > τ:
    g ← τ · g / ||g||₂
```

where g = ∇_θ ℒ is the gradient vector and τ is the threshold. This is global norm clipping: all gradients are rescaled together, preserving the relative direction of the gradient vector while bounding its magnitude.

---

## Real-world Example

An LSTM language model on a character-level sequence prediction task, demonstrating the vanilla RNN's training failure vs LSTM's successful convergence.

```python
import torch
import torch.nn as nn
from torch.utils.data import DataLoader, TensorDataset
from typing import Tuple

class CharLSTM(nn.Module):
    """Character-level LSTM language model."""

    def __init__(self, vocab_size: int, embed_dim: int, hidden_dim: int,
                 num_layers: int = 2, dropout: float = 0.3) -> None:
        super().__init__()
        self.embed = nn.Embedding(vocab_size, embed_dim)
        self.lstm  = nn.LSTM(embed_dim, hidden_dim, num_layers,
                             batch_first=True, dropout=dropout)
        self.head  = nn.Linear(hidden_dim, vocab_size)
        self.hidden_dim  = hidden_dim
        self.num_layers  = num_layers

    def forward(
        self,
        x: torch.Tensor,                                   # [B, T] token indices
        state: Tuple[torch.Tensor, torch.Tensor] | None = None,
    ) -> Tuple[torch.Tensor, Tuple[torch.Tensor, torch.Tensor]]:
        emb = self.embed(x)                                # [B, T, embed_dim]
        out, state = self.lstm(emb, state)                 # out: [B, T, hidden_dim]
        logits = self.head(out)                            # [B, T, vocab_size]
        return logits, state


def train_epoch(
    model: CharLSTM,
    loader: DataLoader,
    optimizer: torch.optim.Optimizer,
    max_grad_norm: float = 1.0,
) -> float:
    model.train()
    total_loss = 0.0
    for x_batch, y_batch in loader:                        # x_batch, y_batch: [B, T]
        optimizer.zero_grad()
        logits, _ = model(x_batch)                         # [B, T, V]
        # Flatten for cross-entropy: [B*T, V] vs [B*T]
        loss = nn.functional.cross_entropy(
            logits.reshape(-1, logits.size(-1)), y_batch.reshape(-1)
        )
        loss.backward()
        # Gradient clipping — essential for RNN/LSTM training
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_grad_norm)
        optimizer.step()
        total_loss += loss.item()
    return total_loss / len(loader)
```

> [!WARNING]
> When using `nn.LSTM` with `batch_first=True`, the hidden and cell state tensors have shape `[num_layers, B, hidden_dim]` (not `[B, num_layers, hidden_dim]`). The first dimension is *layers*, not batch. Getting this wrong when manually initialising the state produces no error — it silently feeds the wrong initial states and the model converges to lower accuracy.

Training a vanilla RNN (`nn.RNN`) on the same task with sequences of length > 50 will show loss plateauing ~3–5 epochs earlier than the LSTM, illustrating the vanishing gradient problem empirically.

---

## In Practice

**Gradient clipping is non-optional for RNNs.** Unlike MLPs and ConvNets where gradient explosion is uncommon, vanilla RNNs routinely explode at the start of training. Threshold τ = 1.0 or τ = 5.0 is the standard setting. Always log the global gradient norm (`torch.nn.utils.clip_grad_norm_` returns the pre-clipping norm) to verify clipping is not triggering every step — frequent clipping indicates a learning rate or initialisation problem.

**Forget gate initialisation matters.** Initialise LSTM forget gate bias to 1 (not 0). PyTorch's `nn.LSTM` initialises all biases to 0 by default. Fix it manually:

```python
for name, param in model.lstm.named_parameters():
    if "bias_hh" in name:
        # bias layout: [input_gate | forget_gate | cell | output_gate]
        n = param.shape[0] // 4
        nn.init.constant_(param[n:2*n], 1.0)   # forget gate bias = 1
```

**Seq2seq reversal trick.** Sutskever et al. (2014) found that reversing the source sequence (feeding it to the encoder in reverse order) improved BLEU by ~1.5 points on English→French. The intuition: the beginning of the source sequence is temporally closest to the beginning of the target sequence after reversal, reducing the "minimum time lag" between correlated input and output tokens.

**BPTT truncation length.** Full BPTT is O(T²) memory and O(T²) compute for sequence length T (because of the recurrence). Truncated BPTT with chunk size k reduces memory to O(k) and compute to O(T·k), at the cost of not propagating gradients beyond k steps. In practice, for language modelling k = 35–70 is standard; for speech k = 200–300. Setting k too small means the model can't learn dependencies longer than k.

> [!NOTE]
> By 2017, attention mechanisms (Bahdanau et al. 2015) had largely superseded the fixed-context-vector bottleneck in seq2seq, allowing the decoder to attend to all encoder hidden states at each step. By 2018, Transformers eliminated the recurrence entirely. However, LSTMs remain the default for streaming inference, time series forecasting, and tasks where causal (online) processing is required, because their per-step compute is O(d²) rather than O(T·d²) for attention over a growing context.

---

## Pitfalls

- **"LSTM solves the vanishing gradient problem completely."** LSTM drastically mitigates it via the cell state gradient highway, but very long dependencies (thousands of steps) can still be problematic if the forget gate is near 0 throughout. Attention mechanisms are the proper solution for very long range.
- **"Gradient clipping is equivalent to changing the learning rate."** Clipping bounds the norm of the gradient update; it does not scale the learning rate. Clipping triggers only when the gradient is anomalously large (an instability), while learning rate scaling applies uniformly to all steps. They have different effects on convergence.
- **"The hidden state h_t and cell state c_t are both outputs of the LSTM."** Both exist, but h_t is the visible output (used for prediction, passed to the next layer); c_t is the internal memory (not passed to subsequent layers in a stacked LSTM, though h_t is). When initialising or resetting LSTM state between sequences, both (h_0, c_0) must be reset to zero.
- **"Bidirectional LSTMs can be used for generation."** BiLSTM reads the sequence in both directions and concatenates hidden states. It requires the full sequence before producing any output — there is no causal constraint. It is appropriate for encoding tasks (text classification, NER) but cannot be used for autoregressive generation.
- **"Teacher forcing eliminates all training instability."** Teacher forcing speeds convergence but creates exposure bias: the model is never trained to condition on its own (possibly wrong) predictions. At test time, errors compound. Scheduled sampling (gradually replacing teacher-forced inputs with model predictions) partially addresses this but adds training complexity.

---

## Exercises

### Exercise 1

Write out the six LSTM equations for a single timestep and annotate each gate with its function and why it helps with vanishing gradients.

#### Solution 1

See the "How it works" section equations. Annotation:

- **i_t (input gate)**: σ(·) ∈ (0,1). Controls "how much new information to write." Multiplied with g_t to modulate the write amplitude.
- **f_t (forget gate)**: σ(·) ∈ (0,1). Controls "how much of the past to retain." The gradient ∂c_t/∂c_{t−1} = diag(f_t): when f_t ≈ 1, gradient passes with magnitude ≈ 1 — the highway.
- **o_t (output gate)**: σ(·) ∈ (0,1). Controls "how much of the cell state to expose." The cell state c_t is hidden from the outside world except through tanh(c_t) ⊙ o_t.
- **g_t (cell input)**: tanh(·) ∈ (−1,1). The candidate values to potentially write into the cell state.
- **c_t (cell state update)**: f_t ⊙ c_{t−1} + i_t ⊙ g_t. The additive update (not multiplicative-through-tanh) is the vanishing-gradient fix. Gradient flows through addition unchanged.
- **h_t (hidden state)**: o_t ⊙ tanh(c_t). The "read" operation: expose a filtered version of the cell state as the visible output.

### Exercise 2

The vanilla RNN gradient bound is ||∂h_t/∂h_k|| ≤ (γ_θ γ_φ)^{t−k}. Suppose γ_θ = 1.05 and γ_φ = 1 (e.g., ReLU activations). What happens to the gradient over 50 steps? What does this mean for training?

#### Solution 2

With γ_θ = 1.05 and γ_φ = 1: (1.05)^{50} ≈ 11.5. The gradient magnitude at step t with respect to step k = t−50 is amplified by a factor of ~11.5. With longer sequences (t−k = 100), the factor is (1.05)^{100} ≈ 131. Gradients grow exponentially — this is exploding gradients.

Consequence: the gradient update step ||η · ∇_θ ℒ|| becomes very large, causing the parameter update to overshoot, and training diverges. The fix: gradient clipping caps the update norm at τ regardless of how much the gradient has exploded, preventing divergence while still moving in the gradient direction.

---

## Sources

- de Freitas, N. Oxford Lecture oxf(15): Recurrent Neural Networks and LSTM. 2014.
- Hochreiter, S. & Schmidhuber, J. (1997). Long Short-Term Memory. *Neural Computation*, 9(8), 1735–1780.
- Sutskever, I., Vinyals, O. & Le, Q.V. (2014). Sequence to Sequence Learning with Neural Networks. *NeurIPS*.
- Bengio, Y., Simard, P. & Frasconi, P. (1994). Learning Long-Term Dependencies with Gradient Descent Is Difficult. *IEEE Transactions on Neural Networks*, 5(2), 157–166.
- Cho, K. et al. (2014). Learning Phrase Representations using RNN Encoder-Decoder for Statistical Machine Translation. *EMNLP*. arXiv:1406.1078.
- Jozefowicz, R., Zaremba, W. & Sutskever, I. (2015). An Empirical Evaluation of Recurrent Network Architectures. *ICML*.
- Conversation with user on 2026-05-19.

---

## Related

- [2 - Backpropagation](./2-backpropagation.md)
- [5 - Deep Architecture Principles](./5-deep-architecture-principles.md)
- [6 - Language Modeling Foundations](./6-language-modeling-foundations.md)
- [2 - Language Models: Autoregressive vs Masked](../../AI-Engineering/1-foundations/2-language-models.md)
