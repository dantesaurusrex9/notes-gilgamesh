---
title: "6 - Language Modeling Foundations"
created: 2026-05-19
updated: 2026-05-19
tags: []
aliases: []
---

# 6 - Language Modeling Foundations

[toc]

> **TL;DR:** Statistical language modeling — learning the joint probability distribution P(w_1, …, w_T) — progressed from n-gram tables (count-based, plagued by data sparsity and the curse of dimensionality) to neural probabilistic language models (Bengio et al. 2003) that learn *distributed word representations* (embeddings) jointly with the probability function, enabling generalisation to unseen n-grams by exploiting semantic similarity. This is the direct ancestor of word2vec, ELMo, GPT, and every modern autoregressive LLM: the architecture, objectives, and vocabulary (perplexity, embedding, softmax bottleneck) are all established in Bengio 2003.

---

## Vocabulary

- **Language model (LM)** — a probability distribution over sequences: P(w_1, …, w_T) = Π_t P(w_t | w_{t−1}, …, w_1). Statistical LMs model this with approximations.
- **n-gram model** — approximates the conditional with a finite context of n−1 words: P(w_t | w_{t−1}, …, w_1) ≈ P(w_t | w_{t−n+1}, …, w_{t−1}). Stored as count tables; requires smoothing for unseen n-grams.
- **Curse of dimensionality** — for a vocabulary V of size |V| = 100K, the number of distinct 10-grams is |V|^{10} = 10^{50}. No training corpus can cover this space; n-gram models generalise only by gluing short overlapping seen sequences.
- **Distributed word representation (embedding)** — a real-valued vector C(w) ∈ ℝ^m that encodes the semantic/syntactic properties of word w. The embedding matrix C ∈ ℝ^{|V|×m} is a parameter of the model.
- **Shared parameters across positions** — the Bengio 2003 model uses the same C matrix at all context positions (w_{t−1}, …, w_{t−n+1}), analogous to weight sharing in ConvNets. This is the prototype for parameter-efficient language modelling.
- **Perplexity (PPL)** — the exponential of the average negative log-likelihood per word: PPL = exp(−(1/T) Σ_t log P(w_t | context)). Lower is better. A uniform model over V has PPL = |V|. A trigram model on Brown corpus has PPL ≈ 336; the Bengio 2003 MLP model achieves ≈ 252.
- **Softmax bottleneck** — the output softmax projects from a fixed-dimensional hidden state to a distribution over |V| classes. If |V| ≫ d_hidden, the rank of the log-probability matrix is bounded by d_hidden — limiting the expressivity of the model over the vocabulary. Addressed in modern LLMs by using very large d_hidden (4096–8192).
- **Negative log-likelihood (NLL)** — the training objective: L = −(1/T) Σ_t log P(w_t | w_{t−n+1}, …, w_{t−1}; θ). Equivalent to cross-entropy between the model distribution and the empirical distribution.
- **Context window** — the number of preceding tokens the model conditions on. Bengio 2003: n−1 = 4–5. GPT-3: 2048. GPT-4: 128K. LSTM: theoretically unbounded but practically ~100–200 tokens.
- **Causal / autoregressive LM** — predicts each token from all preceding tokens. Enables generation by sampling. The paradigm of GPT, Llama, Claude.
- **Masked LM** — predicts masked tokens from surrounding context in both directions. Enables richer representations but not generation. The paradigm of BERT.
- **Next-token prediction** — the training objective of autoregressive LMs: predict w_t from (w_1, …, w_{t−1}). Self-supervised: no labels needed beyond the text itself.
- **Teacher forcing** — feeding the ground-truth previous token w_{t−1} during training (not the model's prediction). Analogous to the RNN training regime.
- **Weight tying** — sharing the embedding matrix C with the output projection W_out (C = W_out^T). Reduces parameters by |V|·d without loss of accuracy; standard in modern LLMs.

---

## Intuition

A trigram model knows P("cat is walking" | "the") from counting, but assigns probability zero to "dog is walking | the" if that exact trigram was not in training data — even though "dog" and "cat" play the same grammatical role and the sentence is clearly valid. The model treats words as atomic, discrete symbols with no notion of similarity.

The Bengio 2003 insight: words have smooth geometry in a continuous space. If C("cat") ≈ C("dog") ≈ C("animal"), then the probability function g, which is a smooth function of its embedding inputs, will automatically assign similar probabilities to "A cat is walking" and "A dog is running" even if only the first was in training data. One observed sentence updates the probability of an exponential number of semantically related sentences — this is what defeats the curse of dimensionality.

The architecture is simple: look up embeddings for the context words, concatenate them into a single feature vector, pass through one hidden layer with tanh, project to a |V|-dimensional softmax. The parameters are learned end-to-end by maximising the log-likelihood of the training corpus. The embeddings are not hand-crafted — they emerge from the gradient updates as the model learns that words appearing in similar contexts should have similar vectors.

---

## How it works

### The Bengio 2003 Architecture

The neural probabilistic language model (Bengio et al. JMLR 2003) decomposes the probability function f(w_t, w_{t−1}, …, w_{t−n+1}) = P̂(w_t | w_{t−1}, …, w_{t−n+1}) into two stages:

**Stage 1 — Embedding lookup.** Each context word w_{t−k} is mapped to its embedding vector C(w_{t−k}) = the w_{t−k}-th row of the matrix C ∈ ℝ^{|V|×m}. The context embeddings are concatenated to form the feature vector:

```math
x = (C(w_{t−1}), C(w_{t−2}), …, C(w_{t−n+1})) ∈ ℝ^{(n−1)·m}
```

**Stage 2 — Probability function.** The probability is computed by a feedforward network with one hidden layer:

```math
y = b + W·x + U · tanh(d + H·x)
P̂(w_t | context) = softmax(y)_t = exp(y_t) / Σ_k exp(y_k)
```

where b ∈ ℝ^{|V|} (output biases), W ∈ ℝ^{|V|×(n−1)m} (optional direct connections from embeddings to output — set to 0 for a pure 2-layer network), U ∈ ℝ^{|V|×h} (hidden-to-output weights), d ∈ ℝ^h (hidden biases), H ∈ ℝ^{h×(n−1)m} (input-to-hidden weights).

**Figure:** Bengio 2003 neural LM architecture.

```
i-th output = P(w_t = i | context)
        [softmax]
            ↑
   [b + Wx + U·tanh(d+Hx)]  ← "most computation here"
            ↑
        [tanh]
            ↑
   [C(w_{t-1}), C(w_{t-2}), ..., C(w_{t-n+1})]
            ↑
     [embedding table C]
            ↑
   w_{t-1}  w_{t-2}  ...  w_{t-n+1}
```

The parameter set is θ = (b, W, U, d, H, C). The embedding C is trained end-to-end. The number of free parameters is |V|(1 + nm + h) + h(1 + (n−1)m) — dominated by |V|(nm + h), scaling linearly with vocabulary size.

### Training Objective

Training maximises the penalised log-likelihood over the training corpus of T words:

```math
L = (1/T) Σ_t log f(w_t, w_{t−1}, …, w_{t−n+1}; θ) + R(θ)
```

where R(θ) is weight decay (applied to W, H, U but not C in the original paper). Stochastic gradient ascent with learning rate ε_t = ε_0 / (1 + r·t) where r is a decay factor (typically r = 10^{−8}).

### The Computational Bottleneck: Output Softmax

Computing the output softmax requires evaluating exp(y_k) for all |V| words to normalise. For |V| = 17,964 and h = 60, n = 6, m = 100 (the AP News experiment), approximately 99.7% of total operations are in computing the |V|-dimensional output layer. This is the *softmax bottleneck* in its computational manifestation.

The Bengio 2003 paper parallelises this by partitioning the output vocabulary across CPUs (parameter-parallel processing): each CPU handles a block of |V|/M output units, and an AllReduce computes the softmax normalisation constant.

Modern solutions:
- **Hierarchical softmax**: represent the vocabulary as a tree; predict log(|V|) binary decisions per step instead of |V| logits.
- **Negative sampling (word2vec)**: train a binary classifier instead of a softmax; sample k "negative" words per positive.
- **Adaptive softmax** (Grave et al. 2017): bin vocabulary by frequency; use a smaller head for rare words.

### From Bengio 2003 to GPT

The architecture evolved but the core ideas are unchanged:

```
Bengio 2003:  Embedding → MLP(tanh) → Softmax(|V|)     fixed context n−1
              n = 5, m = 100, h = 60. Brown corpus PPL: 252 (best trigram: 336)

Word2Vec:     Embedding → dot product → Softmax          no hidden layer
(2013)        Negative sampling objective. 300-d embeddings.

ELMo (2018):  Embedding → BiLSTM → Linear → Softmax     contextual embeddings

GPT (2018):   Embedding + Position → Transformer decoder → Softmax(|V|)
              117M params, 1024 context. Next-token prediction.

GPT-3 (2020): Same objective, 175B params, 2048 context. Few-shot learning emerges.

Llama 3 (2024): Same objective, 70–405B params, 128K context. RoPE + GQA + SwiGLU.
```

The jump from Bengio 2003 to GPT is: (1) replace MLP with Transformer decoder (Vaswani et al. 2017), (2) extend context from n−1 words to thousands of tokens with attention, (3) scale parameters from millions to billions. The objective (next-token prediction on text) is identical.

### n-gram Baselines and Why Neural LMs Win

The Bengio 2003 paper's experimental result on Brown corpus:

| Model | Context | PPL (test) |
| :--- | :---: | ---: |
| Deleted Interpolation trigram | 2 | 336 |
| Kneser-Ney back-off (best n-gram) | 4 | 312 |
| MLP3 (neural, n=5, h=50, m=60) | 4 | 310 |
| MLP1 (neural, n=5, h=50, m=60, mix w/ trigram) | 4 | **252** |

The neural model achieves a 24% reduction in PPL over the best n-gram. Mixing the neural model with the trigram (equal weights) helps further because they make errors in different places — the n-gram is excellent at memorising high-frequency phrases; the neural model generalises better to novel contexts.

> [!NOTE]
> The Bengio 2003 model took 3 weeks to train on 40 CPUs for 5 epochs on the AP News corpus (14M words). The same model today trains in minutes on a modern GPU. The computational bottleneck at the time was the |V|-dimensional output layer — a problem that is still present in modern LLMs (GPT-3's vocabulary is 50,257) but is handled by hardware (A100 GPUs with fast matrix operations) rather than algorithmic tricks.

---

## Math

### Perplexity as inverse probability

Perplexity is the geometric mean of inverse probabilities:

```math
PPL = exp(−(1/T) Σ_t log P(w_t | context))
    = (Π_t P(w_t | context))^{−1/T}
    = 2^{H}   (in bits, where H is the cross-entropy)
```

A uniform model assigns P(w) = 1/|V|, giving PPL = |V| = the vocabulary size. A perfect model (PPL = 1) assigns probability 1 to every word — impossible except for deterministic sequences. A model with PPL = k is equivalent to being "as confused as if randomly choosing among k words."

### Softmax bottleneck

The log-probability matrix A ∈ ℝ^{|V|×d_{context}} where A_{w,c} = log P(w | context c) has rank at most min(|V|, d_{context}). If the true log-probability matrix has rank r > d_{context}, the model cannot express it regardless of the number of parameters in C and W_out. This rank constraint is the softmax bottleneck (Yang et al. 2018). Resolution: increase d_hidden, or use a mixture-of-softmaxes output.

### Embedding dimensionality tradeoff

The parameter count of the embedding table is |V| × m. For m = 100 and |V| = 50,000: 5M parameters just for embeddings. A general rule: m ≈ 100–300 for word2vec-style embeddings; m = 512–4096 for modern LLM residual stream width. The representation needs to be rich enough to capture semantic similarity but small enough to be trainable without excessive parameters.

---

## Real-world Example

A PyTorch implementation of the Bengio 2003 architecture (fixed-context neural LM), using weight tying between the embedding and output layer.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from torch.utils.data import Dataset, DataLoader
from typing import Iterator

class NGramDataset(Dataset):
    """Creates (context, target) pairs from a flat token sequence."""

    def __init__(self, tokens: list[int], context_size: int) -> None:
        self.data: list[tuple[list[int], int]] = [
            (tokens[i : i + context_size], tokens[i + context_size])
            for i in range(len(tokens) - context_size)
        ]

    def __len__(self) -> int:
        return len(self.data)

    def __getitem__(self, idx: int) -> tuple[torch.Tensor, torch.Tensor]:
        ctx, tgt = self.data[idx]
        return torch.tensor(ctx, dtype=torch.long), torch.tensor(tgt, dtype=torch.long)


class BengioLM(nn.Module):
    """
    Bengio et al. 2003 neural probabilistic language model.
    y = b + W·x + U·tanh(d + H·x)
    with weight tying: embedding matrix C = output projection W_out^T.
    """

    def __init__(self, vocab_size: int, embed_dim: int,
                 hidden_dim: int, context_size: int) -> None:
        super().__init__()
        self.context_size = context_size
        self.embed_dim    = embed_dim
        # Embedding table C ∈ R^{|V| × m}
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        # Hidden layer: H ∈ R^{h × (n-1)m}, d ∈ R^h
        self.hidden = nn.Linear(context_size * embed_dim, hidden_dim)
        # Output layer: U ∈ R^{|V| × h}, b ∈ R^{|V|}
        self.output = nn.Linear(hidden_dim, vocab_size)
        # Weight tying: W_out shares weights with the embedding matrix
        self.output.weight = self.embedding.weight  # shape [|V|, embed_dim] — only works if hidden_dim == embed_dim
        # Optional: if hidden_dim != embed_dim, add a linear projection before output
        if hidden_dim != embed_dim:
            self.project = nn.Linear(hidden_dim, embed_dim, bias=False)
        else:
            self.project = None

    def forward(self, context: torch.Tensor) -> torch.Tensor:
        # context: [B, n-1]  token indices
        x = self.embedding(context)                         # [B, n-1, m]
        x = x.view(x.size(0), -1)                          # [B, (n-1)*m]
        h = torch.tanh(self.hidden(x))                     # [B, hidden_dim]
        if self.project is not None:
            h = self.project(h)                             # [B, embed_dim]
        logits = self.output(h)                             # [B, |V|]
        return logits                                       # CrossEntropyLoss expects logits


# Quick smoke test
torch.manual_seed(42)
V, m, h, n = 1000, 64, 128, 5
model = BengioLM(vocab_size=V, embed_dim=m, hidden_dim=h, context_size=n - 1)

# Fake training batch
context_batch = torch.randint(0, V, (32, n - 1))
target_batch  = torch.randint(0, V, (32,))

optimizer = torch.optim.SGD(model.parameters(), lr=1e-3, momentum=0.9)
for step in range(5):
    optimizer.zero_grad()
    logits = model(context_batch)
    loss   = F.cross_entropy(logits, target_batch)
    loss.backward()
    optimizer.step()
    ppl = torch.exp(loss).item()
    print(f"step {step}  loss={loss.item():.4f}  PPL={ppl:.1f}")
```

> [!TIP]
> Weight tying (sharing the embedding matrix with the output projection) is a standard trick in language models that reduces parameters by |V|·embed_dim (5M for |V|=50K, m=100) and often *improves* generalisation because it forces the embedding and output representations to be aligned. Press & Wolf (2017) formalised this and showed it works across many LM architectures.

---

## In Practice

**The output softmax is the bottleneck.** In any language model over a large vocabulary, computing the normalisation constant Σ_k exp(y_k) requires evaluating all |V| output logits. For |V| = 50,257 (GPT-2) and a batch of 512 sequences of length 1024, that is 512 × 1024 × 50,257 ≈ 26B multiplications per forward step — just for the output projection. Modern LLMs mitigate this with tensor parallelism: the output vocabulary is sharded across GPU devices, with each device computing a slice of the logits and an AllReduce for the normalisation.

**Context window and the memory–capacity tradeoff.** Bengio 2003 uses n−1 = 4 context tokens; GPT-3 uses 2048; Llama 3 supports 128K. The increase from n-gram to Transformer is not just architectural — it fundamentally changes what can be learned. With a 4-token context, the model cannot capture sentence-level subject-verb agreement. With a 128K-token context, it can condition on an entire book. The price: attention's O(T²) complexity in the context window length T.

**Perplexity as a sufficient summary statistic.** Perplexity measures average uncertainty per word. A 1 PPL reduction from 250 to 249 sounds small but is a consistent, measurable improvement that transfers to downstream task performance. However, perplexity does not capture all dimensions of language quality: a model can have excellent PPL while producing incoherent long-range text (e.g., an LSTM trained on shuffled sentences). GPT-era evaluation therefore supplemented PPL with downstream tasks (SuperGLUE, HellaSwag, etc.).

**Embedding geometry and analogical reasoning.** The finding that embeddings trained for next-word prediction capture semantic relationships (king − man + woman ≈ queen, from word2vec 2013) is a direct consequence of the Bengio 2003 objective: words appearing in similar contexts get similar embeddings because the probability function must produce similar outputs for them. This is the practical consequence of "fighting the curse of dimensionality with distributed representations."

> [!CAUTION]
> Sampling from language models without temperature control produces degenerate outputs. At temperature T = 0 (argmax), the model greedily picks the highest-probability token at every step, producing repetitive, high-confidence but often semantically shallow text. At T → ∞ (uniform), outputs are random gibberish. The practical operating range is T ∈ [0.6, 1.2], but optimal T is task-dependent. Never deploy an autoregressive LM without considering the sampling hyperparameters (temperature, top-p / nucleus sampling, top-k).

---

## Pitfalls

- **"Perplexity is a language model's quality metric."** Perplexity measures language modelling quality on the *evaluation corpus*. It does not measure instruction following, factual accuracy, reasoning, or safety. A model can have PPL = 3.5 (excellent) and hallucinate confidently. Modern evaluation uses perplexity as one signal among many.
- **"n-gram models are strictly worse than neural LMs."** Kneser-Ney n-gram models remain competitive for small corpora and are extremely fast to compute. For very low-resource languages with < 1M words of text, an n-gram model often outperforms a neural model because there is insufficient data to train embeddings. The Bengio 2003 paper's 8% PPL advantage on AP News (14M words) is precisely because the larger corpus gives the neural model enough signal to learn useful embeddings.
- **"Embeddings are fixed after training."** In modern LLMs, the embedding table is fine-tuned along with all other parameters. Only in old word2vec-style pipelines were embeddings treated as frozen pretrained features. In GPT, the embedding is just the first layer of a continuous function and receives gradients during both pretraining and fine-tuning.
- **"Weight tying forces the input and output representations to be identical."** Weight tying shares the embedding matrix between the input embedding and the output linear projection. The input embedding maps word index → vector; the output projection maps vector → unnormalised logit for each word. They use the same parameters, but the computations are different (embedding lookup = select row; output projection = matrix-vector multiply). The shared parameters are trained to be useful for both, which in practice means they capture semantic similarity (useful for input) and distinguishability (useful for output).
- **"The Transformer replaced the objective."** GPT uses the exact same training objective as Bengio 2003 — maximise log P(w_t | w_{t−1}, …, w_1). The Transformer replaced the architecture (MLP with fixed context → attention with full context), but the self-supervised next-word prediction objective is 20+ years old.

---

## Exercises

### Exercise 1

A unigram language model assigns P(w) = freq(w)/N for each word (ignoring context). Compute its perplexity on a test set where the true distribution is uniform over a vocabulary of size |V| = 10,000, and the model assigns P(w) = 1/10,000 to every word. What does this tell you about the baseline difficulty of language modelling?

#### Solution 1

For a uniform model, P(w_t | context) = 1/|V| for all t. The per-word log-probability is log(1/|V|) = −log|V| = −log(10,000) ≈ −9.21 nats.

Perplexity = exp(9.21) = 10,000 = |V|.

Interpretation: a uniform model has perplexity equal to the vocabulary size — it is "as confused as if randomly choosing among 10,000 words." Any useful language model must achieve PPL << |V|. The Bengio 2003 trigram baseline on Brown corpus has PPL ≈ 336 (out of ~16K unique words) — already 48× better than uniform. A well-trained GPT-2 on the same domain would achieve PPL ≈ 20–40.

### Exercise 2

Derive the gradient of the NLL loss with respect to the embedding vector C(w_{t−1}) for one training example in the Bengio 2003 model. At what points in the computation does the gradient become sparse (affecting only some embeddings)?

#### Solution 2

The loss for one example is L = −log P̂(w_t | context). Let x = [C(w_{t−1}), …, C(w_{t−n+1})] be the concatenated context embeddings, h = tanh(d + Hx), y = b + Ux (with no direct connections), and ŷ_k = exp(y_k)/Z.

Gradient of L with respect to y: ∂L/∂y = ŷ − e_{w_t} (one-hot error vector).

Gradient with respect to h: ∂L/∂h = Uᵀ (ŷ − e_{w_t}).

Gradient with respect to x: ∂L/∂x = Hᵀ · diag(1 − h²) · Uᵀ (ŷ − e_{w_t}).

Since x = [C(w_{t−1}), …, C(w_{t−n+1})], the gradient ∂L/∂C(w_{t−k}) is the k-th block of ∂L/∂x of size m. Sparsity: only the embeddings for the n−1 context words and the target word receive gradient updates per training step. All other |V| − n embeddings have zero gradient. This is the sparse update property mentioned in Bengio (2013) as an early instance of conditional computation — per-step, only O(n) rows of the |V| × m embedding matrix are touched.

### Exercise 3

Compute the perplexity ratio of a model that achieves average cross-entropy loss of 2.1 nats/token versus a model achieving 2.8 nats/token. Express the result as both a perplexity difference and a perplexity ratio.

#### Solution 3

PPL₁ = exp(2.1) ≈ 8.16
PPL₂ = exp(2.8) ≈ 16.44

Perplexity difference: 16.44 − 8.16 = 8.28.
Perplexity ratio: 16.44 / 8.16 ≈ 2.01.

The model with loss 2.1 is approximately twice as "certain" as the model with loss 2.8 in terms of the effective number of choices per token. In practice, this difference (0.7 nats/token = ~1.0 bits/token) is a large gap in language modelling quality — roughly the difference between a fine-tuned model and a poorly pretrained baseline on the same domain.

---

## Sources

- Bengio, Y., Ducharme, R., Vincent, P. & Jauvin, C. (2003). A Neural Probabilistic Language Model. *Journal of Machine Learning Research*, 3, 1137–1155.
- Bengio, Y. (2013). Deep Learning of Representations: Looking Forward. arXiv:1305.0445.
- Mikolov, T. et al. (2013). Efficient Estimation of Word Representations in Vector Space (word2vec). arXiv:1301.3781.
- Radford, A. et al. (2018). Improving Language Understanding by Generative Pretraining (GPT). https://cdn.openai.com/research-covers/language-unsupervised/language_understanding_paper.pdf
- Radford, A. et al. (2019). Language Models are Unsupervised Multitask Learners (GPT-2).
- Press, O. & Wolf, L. (2017). Using the Output Embedding to Improve Language Models. *EACL*.
- Yang, Z. et al. (2018). Breaking the Softmax Bottleneck: A High-Rank RNN Language Model. *ICLR*.
- Conversation with user on 2026-05-19.

---

## Related

- [4 - Recurrent Networks and LSTM](./4-recurrent-networks-and-lstm.md)
- [5 - Deep Architecture Principles](./5-deep-architecture-principles.md)
- [2 - Language Models: Autoregressive vs Masked](../../AI-Engineering/1-foundations/2-language-models.md)
- [3 - Generative AI Fundamentals](../../AI-Engineering/1-foundations/3-generative-ai-fundamentals.md)
- [2 - Entropy, Cross-Entropy, and Perplexity](../../AI-Engineering/3-evaluation/2-entropy-cross-entropy-perplexity.md)
