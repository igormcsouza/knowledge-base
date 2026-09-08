---
tags:

- machine-learning
- nlp
- transformers
- llms
- deep-learning
- intermediate

---

# From NLP to LLMs: the Mathematical and Architectural Path to GPT-3.5

Large language models did not appear out of nowhere. Every ingredient — token
representations, sequence modeling, attention, scale, and alignment — was built on top of
the ingredient before it. This article reconstructs that chain step by step, aimed at
someone who already knows attention/Transformers at a conceptual level but wants the
mathematical bridge connecting classical NLP all the way to ChatGPT.

The narrative in one line:

```text
classical NLP → word representations → RNN/LSTM → attention → Transformer →
self-supervised pretraining → GPT → scaling (GPT-2 → GPT-3) →
instruction tuning → RLHF → GPT-3.5 / ChatGPT
```

## 1. Classical NLP Foundations

Before learned representations, NLP relied on **sparse, hand-built features** counting how
words appear in documents.

**Term Frequency–Inverse Document Frequency (TF-IDF)** weights a term by how often it
appears in a document, discounted by how common it is across all documents:

$$
TF(t,d) = \text{count of term } t \text{ in document } d
$$

$$
IDF(t) = \log\frac{N}{df(t)}
$$

$$
TFIDF(t,d) = TF(t,d) \cdot IDF(t)
$$

- $N$ — total number of documents.
- $df(t)$ — number of documents containing term $t$.

A document becomes a vector over the whole vocabulary, mostly zeros (**sparse**), with
non-zero TF-IDF weights for the words it actually uses (Bag-of-Words, with TF-IDF as a
smarter weighting on top). The limitation: the vector for "car" and the vector for
"automobile" share no dimensions and look completely unrelated, even though the words are
near-synonyms. Sparse representations carry no notion of semantic similarity — they only
capture *which* words occurred.

**Word embeddings** fix this by learning a dense vector $x \in \mathbb{R}^d$ (typically
$d \approx 100$–$300$) per token, trained so that words used in similar contexts end up
with similar vectors (the distributional hypothesis: "a word is characterized by the
company it keeps"). Because the vectors are dense and learned, semantic relationships
become geometric — synonyms cluster, and directions in the space can even encode
relationships (`king - man + woman ≈ queen` was the canonical Word2Vec demonstration).

Similarity between two embeddings is measured with **cosine similarity**, which compares
direction rather than magnitude:

$$
\cos(x,y) = \frac{x^Ty}{\|x\|\|y\|}
$$

This is the first big shift: **from counting to learning a geometry of meaning.**
Everything downstream — RNNs, attention, Transformers — operates on dense vectors like
these rather than sparse counts.

## 2. Neural Sequence Models: RNNs/LSTMs

Bag-of-Words representations throw away word order entirely. Language is sequential —
"dog bites man" and "man bites dog" have the same bag of words but opposite meaning — so
the next step was models that consume a sequence and carry information forward through it.

A **Recurrent Neural Network (RNN)** keeps a hidden state $h_t$ that summarizes everything
seen so far, updated at each time step from the current input and the previous state:

$$
h_t = \phi(W_x x_t + W_h h_{t-1} + b)
$$

$$
y_t = \text{softmax}(W_y h_t + b_y)
$$

- $x_t$ — the input at step $t$ (e.g. a word embedding).
- $h_{t-1}$ — the hidden state carried over from the previous step (the model's "memory").
- $\phi$ — a non-linearity (typically $\tanh$).
- $W_x, W_h, W_y$ — weight matrices shared across *every* time step, which is what makes
  this recurrent rather than a separate network per position.

The same weights are reused at every step, so an RNN can in principle process a sequence
of any length. In practice, training runs into **vanishing/exploding gradients**:
backpropagating the loss through $t$ time steps means multiplying $t$ Jacobians together
(this repeated application is called *backpropagation through time*), and if their norms
are consistently below or above 1, the product shrinks to ~0 or blows up exponentially in
$t$. Vanishing gradients mean the network effectively can't learn dependencies that span
more than a handful of steps — a pronoun 40 words back has almost no gradient signal
reaching it.

**LSTM** (Long Short-Term Memory) and **GRU** (Gated Recurrent Unit) address this with
*gates* — learned vectors in $(0,1)$ that control how much of the previous state to keep,
how much new information to write, and how much to expose as output. This gives the
network a more direct path for gradients to flow through the memory cell across many time
steps, mitigating (not eliminating) vanishing gradients. The mechanism matters less here
than the motivation: **controlling information flow through a sequence was already the
central problem attention would later solve more directly.**

## 3. Sequence-to-Sequence Models and the Need for Attention

Tasks like machine translation need to map an input sequence to an output sequence of
possibly different length. The classical solution was **encoder-decoder**: an encoder RNN
reads the source sentence into a single fixed-size vector (the final hidden state), and a
decoder RNN generates the target sentence conditioned on that one vector.

The problem is the bottleneck: a 50-word sentence and a 3-word sentence get compressed into
the *same-sized* vector. Long sequences lose information — the decoder has to reconstruct
everything from one summary, no matter how much it needed to remember.

**Attention** removes the bottleneck by letting the decoder look back at *all* encoder
hidden states at every generation step, dynamically deciding which ones are relevant right
now, rather than relying on one fixed summary:

$$
e_{ij} = \text{score}(s_{i-1}, h_j)
$$

$$
\alpha_{ij} = \frac{\exp(e_{ij})}{\sum_k \exp(e_{ik})}
$$

$$
c_i = \sum_j \alpha_{ij} h_j
$$

- $s_{i-1}$ — the decoder's hidden state from the previous step (what it's currently
  trying to produce).
- $h_j$ — the encoder's hidden state for source position $j$.
- $e_{ij}$ — a raw compatibility score between "what the decoder wants" and "what encoder
  position $j$ offers."
- $\alpha_{ij}$ — that score normalized into a probability distribution over source
  positions (softmax, so all $\alpha_{ij}$ for a given $i$ sum to 1).
- $c_i$ — the **context vector**: a weighted average of all encoder states, where the
  weights are the attention distribution.

Intuitively, $\alpha_{ij}$ answers "how much should output position $i$ care about input
position $j$ right now?" When translating a word, attention lets the decoder focus mostly
on the corresponding source word (and a bit of surrounding context), rather than fishing
it out of one compressed vector. This is **dynamic, content-based retrieval** from the
encoder's memory — a preview of everything that follows.

## 4. Self-Attention

The attention above relates a decoder state to encoder states — two *different*
sequences. **Self-attention** applies the same idea *within* one sequence: every token
attends to every other token in the same input, including itself.

Instead of hand-designing the "query," self-attention learns three linear projections of
the input matrix $X$ (one row per token):

$$
Q = XW_Q, \quad K = XW_K, \quad V = XW_V
$$

- $Q$ (query) — "what am I looking for?"
- $K$ (key) — "what do I contain, for others to match against?"
- $V$ (value) — "what do I actually offer, once someone attends to me?"

Then **scaled dot-product attention**:

$$
\text{Attention}(Q,K,V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V
$$

Working through the shapes makes this concrete. For a sequence of length $n$ and key/query
dimension $d_k$: $Q \in \mathbb{R}^{n\times d_k}$, $K \in \mathbb{R}^{n\times d_k}$, so
$QK^T \in \mathbb{R}^{n\times n}$ — one raw compatibility score between every pair of
tokens. Softmax (applied row-wise) turns each row into a probability distribution over
*all* tokens — this replaces the hand-derived $\alpha_{ij}$ from the previous section with
a learned equivalent. Multiplying by $V \in \mathbb{R}^{n\times d_v}$ produces an output
in $\mathbb{R}^{n\times d_v}$: for each token, a weighted blend of every other token's
value vector, weighted by how relevant that token's key was to this token's query.

**Why divide by $\sqrt{d_k}$?** Assume $Q$ and $K$ entries are roughly independent with
mean 0, variance 1. The dot product $q \cdot k = \sum_{i=1}^{d_k} q_i k_i$ sums $d_k$
such terms, so its variance grows proportionally to $d_k$ — for large $d_k$, raw dot
products can become very large in magnitude. Feeding large values into softmax pushes it
into regions where the gradient is nearly flat (the largest logit dominates and the
softmax saturates), which makes learning unstable. Dividing by $\sqrt{d_k}$ rescales the
dot product back to roughly unit variance regardless of $d_k$, keeping softmax in a
well-behaved range.

The key structural result: **every token can attend directly to every other token in a
single operation**, with a path length of 1 between any two positions — no matter how far
apart they are in the sequence. Compare this to an RNN, where information from position 1
has to pass through $t-1$ recurrent steps to reach position $t$. This is exactly what
solves the vanishing-gradient/long-range-dependency problem from Section 2, and it does so
without any recurrence at all.

## 5. Multi-Head Attention

A single attention operation computes one weighted average per token — one "view" of
which tokens matter to which. But relevance is multi-faceted: one relationship might be
syntactic (subject-verb agreement), another semantic (coreference — which noun a pronoun
refers to), another positional (nearby words). A single softmax distribution can't
represent all of these at once — it's one weighting scheme, and averaging together
signals with different purposes tends to blur them into a single, unhelpfully generic
pattern.

**Multi-head attention** runs several attention operations ("heads") in parallel, each
with its own learned projections, so each head is free to specialize in different
relational patterns:

$$
head_i = \text{Attention}(QW_i^Q, KW_i^K, VW_i^V)
$$

$$
\text{MultiHead}(Q,K,V) = \text{Concat}(head_1,\ldots,head_h)W^O
$$

Each head projects into a smaller subspace (if the model dimension is $d_{model}$ and
there are $h$ heads, each head typically uses $d_k = d_{model}/h$), so the total compute is
comparable to one full-size attention operation, just split across $h$ independent
"perspectives." The outputs are concatenated back to $d_{model}$ and mixed by output
projection $W^O$.

In practice, some heads do turn out to specialize in interpretable ways (e.g. attending to
the previous token, or to a syntactic dependency), but this isn't guaranteed or uniform
across a whole model — treat "heads learn distinct roles" as a tendency the architecture
enables, not a designed guarantee for every head.

## 6. The Transformer Architecture (2017)

Vaswani et al.'s "Attention Is All You Need" assembled self-attention, multi-head
attention, and a feed-forward network into a repeatable block, and showed that stacking
these blocks — with no recurrence at all — was enough to beat RNN-based
sequence-to-sequence models.

One Transformer block, in order:

```text
input embeddings + positional encoding
        → multi-head self-attention
        → residual connection + layer normalization
        → position-wise feed-forward network
        → residual connection + layer normalization
```

The **feed-forward network** is applied independently to each position (hence
"position-wise") — it doesn't mix information across tokens (attention already did that);
it just transforms each token's representation:

$$
FFN(x) = \sigma(xW_1 + b_1)W_2 + b_2
$$

Typically $W_1$ expands to a larger inner dimension (e.g. $4\times d_{model}$) and $W_2$
projects back down, with $\sigma$ commonly ReLU or GELU.

Residual connections ($x + \text{Sublayer}(x)$, wrapped by layer normalization) let
gradients skip past each sublayer, which is what makes stacking many blocks (dozens, in
large models) trainable at all — without them, deep stacks would suffer their own version
of the vanishing-gradient problem.

**Positional encoding** is necessary because self-attention itself has no notion of order
— $QK^T$ treats the sequence as a *set*; permuting the tokens permutes the attention
matrix's rows/columns but changes nothing about how any single token is computed relative
to the others. The original paper injects a fixed sinusoidal pattern per position, added
to the token embedding before the first block:

$$
PE(pos, 2i) = \sin\left(pos / 10000^{2i/d_{model}}\right)
$$

$$
PE(pos, 2i+1) = \cos\left(pos / 10000^{2i/d_{model}}\right)
$$

Each dimension pair oscillates at a different frequency, so the combination of all
dimensions gives every position a unique, smoothly-varying signature — and because sine
and cosine are periodic, relative offsets between positions can be expressed as linear
functions of the encoding, which the theory suggested would help the model generalize to
relative positions. (Many modern models use learned or relative positional schemes
instead, but the goal — giving otherwise order-blind self-attention some sense of
sequence position — is the same.)

**Why this mattered practically:** removing recurrence means every token's representation
at a given layer can be computed *independently and in parallel* — there's no "wait for
$h_{t-1}$" dependency. On a GPU/TPU, an entire sequence's self-attention is one batched
matrix multiplication. This is what made training on vastly larger datasets and models
computationally feasible: Transformers parallelize across the sequence dimension in a way
RNNs fundamentally cannot.

## 7. Encoder-Only, Decoder-Only, and Encoder-Decoder Transformers

The Transformer block from Section 6 is a reusable unit; models differ in how they arrange
it and whether attention is allowed to look at the whole sequence or only the past:

- **Encoder-only (BERT-style):** every token attends to every other token, in both
  directions. Good for building representations of a full input (classification, span
  extraction) but not naturally suited to generating text left-to-right.
- **Decoder-only (GPT-style):** every token can only attend to itself and earlier tokens —
  **causal** (masked) self-attention. Naturally suited to autoregressive generation, since
  at inference time future tokens don't exist yet anyway.
- **Encoder-decoder (T5-style):** an encoder processes the full input bidirectionally, and
  a decoder generates the output autoregressively, attending both to its own previous
  tokens (causal self-attention) and to the encoder's output (cross-attention, structurally
  the same mechanism as Section 3's attention).

Causal masking is implemented by adding a mask matrix $M$ to the raw attention scores
before softmax:

$$
M_{ij} = \begin{cases} 0, & j \le i \\ -\infty, & j > i \end{cases}
$$

$$
\text{Attention}(Q,K,V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}} + M\right)V
$$

Adding $-\infty$ to the score for any future position $j > i$ makes that entry's
contribution to the softmax exactly zero (since $e^{-\infty} = 0$) — token $i$'s output can
only be a weighted combination of positions $\le i$. This is the mechanism that turns
self-attention into something compatible with next-token prediction: at training time the
model sees the whole sequence at once (for parallelism), but the mask ensures the
prediction for position $i$ never has access to positions after it, which is exactly the
information it wouldn't have at generation time either. GPT is built entirely from this
decoder-only, causally-masked stack.

## 8. The Key Paradigm Shift: Self-Supervised Language-Model Pretraining

Pre-Transformer NLP mostly followed a **task-specific** pattern: label a dataset for
sentiment analysis, train a model for sentiment analysis; label a dataset for named-entity
recognition, train a separate model for that. Every task needed its own labeled data and
its own model.

The shift: train one large model on a task that requires *no labels at all* —
**predicting the next token** — over enormous amounts of raw, naturally-occurring text.
This is **autoregressive language modeling**: the model learns the joint probability of a
sequence by factoring it into a product of conditional next-token probabilities (the chain
rule of probability):

$$
P(x_1,\ldots,x_T) = \prod_{t=1}^{T} P(x_t \mid x_{<t})
$$

The training objective is the negative log-likelihood of the actual next token under the
model's predicted distribution, summed over the sequence:

$$
\mathcal{L} = -\sum_{t=1}^{T} \log P_\theta(x_t \mid x_{<t})
$$

Minimizing this is equivalent to making the model assign as much probability mass as
possible to whatever token actually comes next, everywhere in the training corpus.

Predicting the next token sounds almost too simple to matter, but at the scale of a large
corpus, doing it well is extremely demanding: to predict the next token accurately, the
model has to implicitly learn grammar (what can follow syntactically), facts (what word
completes a factual statement), reasoning patterns (what follows logically), coding
conventions, discourse structure, and style — because all of these show up as regularities
in what token comes next across billions of examples. Next-token prediction is the *task*;
everything else is a side effect the model is forced to learn in order to get good at it.

## 9. GPT: Generative Pre-Trained Transformer

GPT combines the decoder-only, causally-masked Transformer stack (Section 7) with the
autoregressive pretraining objective (Section 8). The training pipeline, conceptually:

```text
raw text → tokenization → token embeddings → Transformer stack (causal self-attention +
FFN, stacked N times) → logits → softmax → next-token probabilities → loss (cross-entropy)
→ backpropagation → parameter update
```

At the output of the Transformer stack, each position has a vector; a final linear layer
projects it to a vector of **logits** — one raw score per vocabulary entry — and softmax
turns those into a probability distribution over the vocabulary:

$$
P(y_i \mid x) = \frac{e^{z_i}}{\sum_j e^{z_j}}
$$

- $z_i$ — the logit for vocabulary entry $i$ (an unnormalized score; can be any real
  number).
- The exponential makes every term positive, and dividing by the sum normalizes them into
  a valid probability distribution.

**Cross-entropy loss** (Section 8's $\mathcal{L}$) compares this predicted distribution to
the actual next token (a one-hot target); minimizing it is exactly maximum-likelihood
estimation of the model's parameters. **Backpropagation** computes
$\partial \mathcal{L}/\partial \theta$ for every parameter $\theta$ in the network by the
chain rule (the same mechanism as any neural network — see
[Neural Networks: Math Fundamentals](neural-networks-math-fundamentals.md) for the
worked-out derivation on a small network), and **gradient descent** (or a variant like
Adam) nudges every parameter slightly opposite its gradient. Repeated over trillions of
tokens, this is the entire training loop — no additional machinery is needed to get from
"random weights" to "a language model."

## 10. Tokenization and Subword Units

Models don't operate on raw characters (too fine-grained — sequences would be enormous and
long-range structure harder to capture) or whole words (too coarse — vocabularies would be
unbounded, and rare/novel words like "unfriendliness" or a typo would have no
representation at all). The standard solution is **subword tokenization** — commonly
**Byte-Pair Encoding (BPE)** or a variant — which starts from individual
characters/bytes and iteratively merges the most frequent adjacent pairs into new vocabulary
entries, producing a fixed-size vocabulary of common words, word-pieces, and, as a
fallback, individual bytes/characters (so *any* string can be tokenized, even one the
tokenizer has never seen).

The pipeline from text to vectors:

- Raw text is split into a sequence of **token IDs** — integers indexing into a fixed
  **vocabulary** $V$ (typically tens of thousands of entries).
- An **embedding matrix** $E \in \mathbb{R}^{|V| \times d}$ holds one learned row per
  vocabulary entry.
- Looking up a token ID's row in $E$ converts it into the dense vector $x \in
  \mathbb{R}^d$ that the Transformer stack actually operates on (this is literally the $X$
  from Section 4's $Q = XW_Q$ etc., after positional encoding is added).

The consequence worth internalizing: the model predicts the next **token**, not the next
*word* — a token might be a whole common word, a word fragment ("token" + "ization"), or
even a single byte for unusual input. Context length, cost, and "how many words fit in the
prompt" are all really about token counts, not word counts.

## 11. Scaling: GPT-2 → GPT-3

Once the recipe (decoder-only Transformer + autoregressive pretraining) was established,
the next major finding was that making the same recipe **bigger** kept improving results,
predictably, well past what researchers initially expected — with no fundamentally new
architecture needed.

The axes that scale together:

- **Parameter count** ($N$) — more layers, wider layers, more attention heads.
- **Dataset size** ($D$) — vastly more (and more diverse) raw text.
- **Compute** ($C$) — a rough product of $N$, $D$, and training time; larger models trained
  on more data need proportionally more compute.
- **Context length** — how many previous tokens the causal self-attention can look back
  over when predicting the next one.
- **Training stability** — larger models are more sensitive to learning-rate schedules,
  initialization, and normalization choices; a substantial part of scaling work is
  engineering training to *not* diverge at larger sizes.

**Scaling laws** describe an empirical (not derived from first principles) relationship
between loss and these axes — as an explanatory abstraction rather than any specific
paper's exact fitted curve:

$$
L(N,D,C) \approx A N^{-\alpha} + B D^{-\beta} + C_0
$$

The intuition: loss decreases smoothly as a power law in model size and data size (each
term shrinks as $N$ or $D$ grows), flattening out toward some irreducible floor $C_0$. This
predictability is what let labs plan training runs — you could estimate, before spending
the compute, roughly how much a bigger model or bigger dataset would help.

GPT-3's significance wasn't a new architecture over GPT-2 — it was evidence that a
sufficiently large autoregressive Transformer, trained only with the next-token objective,
develops **in-context learning**: given a handful of examples in the prompt (or even just
an instruction), it can perform a task it was never explicitly fine-tuned for, without any
gradient update. This was a qualitative shift in how these models were used — from
"fine-tune a model per task" back toward "one model, many tasks via prompting" — a return,
at much larger scale, to the "one general model" idea, but now achieved by scale rather
than by task-specific architecture.

## 12. Why GPT-3 Was Not Yet ChatGPT

A raw pretrained model like GPT-3 is optimized purely to predict plausible continuations of
text — it has no built-in notion of "answer the user's question helpfully." Given a prompt
that looks like the start of some document, it continues that document, which is not the
same behavior as an assistant.

- **Base LLM:** given `"The capital of France is"`, produces a plausible continuation —
  likely `" Paris."` — because that's what usually follows in its training data.
  Given `"Write a poem about the ocean."`, a base model might just as easily continue with
  *more instructions* (`"Write a poem about the mountains."`) if that's what commonly
  follows similar text in its training data, rather than actually writing a poem — it has
  no learned preference for "comply with the instruction" over "continue the pattern of
  text that contains instructions."
- **Chat assistant:** the same underlying architecture, additionally trained (Sections
  13–14) to recognize "this looks like an instruction" and produce a helpful, on-task
  response rather than an arbitrary plausible continuation.

The gap between these two behaviors is exactly what Sections 13 and 14 close.

## 13. Instruction Tuning / Supervised Fine-Tuning (SFT)

**Instruction tuning** takes the pretrained base model and continues training it — with
the same architecture and the same kind of next-token loss — but now on a curated dataset
of (instruction, ideal response) pairs written or selected specifically to demonstrate
helpful, on-task behavior, rather than on generic web text.

$$
\mathcal{L}_{SFT} = -\sum_t \log P_\theta(y_t \mid y_{<t}, x)
$$

- $x$ — the user's instruction/context.
- $y$ — the target (ideal) response, generated token by token.

This is mechanically the same cross-entropy objective as pretraining (Section 8) — the
only thing that changed is *what data* the loss is computed over. Instead of "predict the
next token of arbitrary internet text," it's "predict the next token of a response a human
considered a good answer to this instruction." A relatively modest amount of high-quality
instruction data (tiny compared to the pretraining corpus) is enough to substantially shift
the model's behavior toward instruction-following, because the underlying capability
(language, facts, reasoning patterns) was already learned during pretraining — SFT mostly
teaches the model *which mode to be in*, not new knowledge.

## 14. RLHF and Preference Optimization

Instruction tuning teaches the model to imitate example responses, but "imitate this
specific answer" doesn't fully capture what makes a response *good* — helpfulness,
relevance, safety, tone, and appropriate refusal are more naturally expressed as
*comparisons* ("response A is better than response B") than as single correct targets.
**Reinforcement Learning from Human Feedback (RLHF)** is built around exactly that kind of
data.

The high-level pipeline:

```text
pretrained model → SFT model → collect preference data (humans rank/compare model outputs)
→ train a reward model on those preferences → optimize the SFT model's policy against the
reward model (reinforcement learning) → aligned model
```

- **Preference data:** humans are shown multiple model outputs for the same prompt and
  rank them (or pick the better one).
- **Reward model:** a separate model trained to take (prompt, response) and output a
  scalar score, fit so that it ranks the human-preferred response higher than the
  rejected one — turning subjective human judgment into a differentiable signal.
- **Policy optimization:** the SFT model (now the "policy") is further trained to produce
  responses that the reward model scores highly, using a reinforcement-learning algorithm
  (the original InstructGPT/ChatGPT work used PPO — Proximal Policy Optimization) — while
  typically penalizing large deviations from the SFT model, so it improves preference
  scores without collapsing into degenerate outputs that merely exploit the reward model.

At a high level, the policy is nudged to increase the reward model's score while a
KL-divergence penalty keeps it close to the original SFT model:

$$
\text{objective} \approx \mathbb{E}\big[R_\phi(x,y)\big] - \beta \cdot \text{KL}\big(\pi_\theta \,\|\, \pi_{SFT}\big)
$$

where $R_\phi$ is the reward model's score and $\pi_\theta$ is the policy being optimized —
presented here as the general shape of the objective rather than a claim about the exact
formula or hyperparameters used for any specific released model.

Why human preferences matter as a training signal at all: helpfulness, relevance, safety,
style, correctly following instructions, and knowing when to refuse are all things that
are far easier for a human to *recognize* when comparing two candidate responses than to
*specify* as a single target output for supervised learning to imitate. RLHF turns "humans
can tell good from bad" into a trainable objective.

It's worth being precise that RLHF-with-PPO is the *historical* recipe associated with the
original InstructGPT/ChatGPT work; modern post-training has evolved considerably (e.g.
Direct Preference Optimization and other preference-optimization methods that skip
training an explicit separate reward model). The general concept — **align a pretrained
model to human preferences using comparison data** — is the durable idea; the specific
algorithm is not.

## 15. GPT-3.5 and ChatGPT

Putting the historical pieces in order:

- **GPT-3 (2020)** demonstrated that a large-scale, decoder-only, autoregressively
  pretrained Transformer exhibits strong few-shot/in-context learning — a base model, not
  yet an assistant.
- **Instruction tuning and preference-based alignment** (Sections 13–14) were applied on
  top of GPT-3-family models to make them substantially more useful and controllable as
  assistants — this is the work generally described as producing the "GPT-3.5 series."
- **ChatGPT launched in November 2022**, built on a GPT-3.5-series model, packaged as a
  conversational product (with additional product-level scaffolding — conversation
  history/turn management, safety mitigations, and a chat-specific interface — beyond just
  the model itself).

It's important to be precise here: **GPT-3, GPT-3.5, and ChatGPT are related but not
identical labels.** GPT-3 names a base pretrained model family; GPT-3.5 names a series of
models built on that lineage with additional instruction-tuning/alignment training;
ChatGPT is a *product* — a system built around an aligned model, with its own product
decisions layered on top — not simply "GPT-3 with a chat interface bolted on." OpenAI has
not published every detail of GPT-3.5's exact training recipe, so specifics beyond the
public high-level description (instruction tuning plus preference-based alignment, in the
lineage of the publicly described InstructGPT approach) should be treated as reasonable
architectural inference rather than confirmed fact.

## 16. The Complete Mathematical Pipeline

Tying every section together, training end-to-end:

$$
\text{text} \rightarrow \text{tokens} \rightarrow X \rightarrow Q,K,V \rightarrow
\text{Attention} \rightarrow \text{Transformer blocks} \rightarrow \text{logits}
\rightarrow \text{softmax} \rightarrow P(token_t \mid token_{<t}) \rightarrow
\text{cross-entropy} \rightarrow \text{gradients} \rightarrow \text{parameter updates}
$$

Each arrow is a section above: tokenization (10) turns text into token IDs; the embedding
matrix (10) turns IDs into $X$; linear projections (4) produce $Q,K,V$; scaled dot-product
and multi-head attention (4, 5) inside stacked, causally-masked Transformer blocks (6, 7)
produce contextualized representations; a final linear layer and softmax (9) produce
next-token probabilities; cross-entropy (8, 9) measures the error; backpropagation and
gradient descent (9) update every parameter.

**Generation** (inference) runs the same forward pass, but one token at a time, feeding
each output back in as input for the next step — this is why it's called **autoregressive**
generation:

$$
x_{1:t} \rightarrow P(x_{t+1} \mid x_{1:t}) \rightarrow \text{sample/select token}
\rightarrow x_{1:t+1}
$$

"Sample/select" can mean greedy decoding (always take the highest-probability token),
sampling (draw from the distribution, optionally reshaped by temperature/top-k/top-p), or
more elaborate search strategies — a decoding-strategy choice sitting on top of the
probability distribution the model produces, not something the training objective
dictates by itself.

## 17. What Changed from "NLP" to "LLM Engineering"

The old paradigm:

```text
one task → one dataset → one specialized model
```

The modern foundation-model paradigm:

```text
huge corpus → one general model (pretraining) → prompting / fine-tuning / tools → many tasks
```

A single pretrained-and-aligned model now serves as a general-purpose component that
downstream applications *compose with* rather than retrain from scratch for every task.
This is precisely why the discipline around LLMs looks less like classical ML research and
more like systems engineering — the model is closer to a database or a search index than
to a bespoke per-task classifier. A backend/software engineer's existing systems
intuition maps directly onto this new layer:

- **Tokenization** — understanding cost/latency/context-length in terms of tokens, not
  words or characters (Section 10).
- **Inference** — running the forward pass (Section 9) efficiently at request time, as
  opposed to training.
- **Batching** — grouping concurrent requests to use GPU compute efficiently, the same
  throughput-vs-latency trade-off as any high-throughput service.
- **KV cache** — reusing already-computed key/value vectors (Section 4) from previous
  tokens during autoregressive generation, instead of recomputing attention over the whole
  prefix at every step — a caching strategy in the same family as any other.
- **Latency/throughput trade-offs** — classic systems trade-offs, now applied to
  token-by-token generation.
- **GPU memory** — model weights, KV cache, and activations all compete for a fixed memory
  budget, much like capacity planning for any memory-bound service.
- **Quantization** — reducing numerical precision of weights/activations to fit more model
  in less memory/bandwidth, at some accuracy cost — a compression trade-off.
- **RAG (Retrieval-Augmented Generation)** — fetching relevant external documents into the
  prompt at request time, rather than relying solely on what was baked in during
  pretraining — conceptually a cache/index sitting in front of the model.
- **Tool calling** — the model emitting a structured request that a surrounding system
  executes and feeds back in, i.e. the model as one component in a larger orchestrated
  pipeline.
- **Evaluation** — measuring whether outputs are actually good, an ongoing systems
  problem (metrics, regression testing, A/B testing) rather than a one-time research
  result.
- **Observability** — logging, tracing, and monitoring model behavior in production, same
  as for any other service with non-deterministic, hard-to-fully-specify outputs.

The point isn't to turn LLM work into pure infrastructure — the modeling ideas from
Sections 1–16 are what make any of this worth building on. But once a model is
trained, aligned, and deployed, everything an engineer does with it — serving it, scaling
it, composing it with other systems — is systems engineering with a probabilistic
component in the middle, and that's exactly the part a backend background already prepares
someone for.

## Summary

- Classical NLP moved from sparse counts (TF-IDF) to dense, geometry-respecting embeddings.
- RNNs/LSTMs modeled sequences with a carried hidden state, but suffered vanishing
  gradients over long sequences.
- Attention removed the fixed-size bottleneck of encoder-decoder models by letting a
  decoder dynamically retrieve relevant encoder states.
- Self-attention generalized this within a single sequence via learned $Q, K, V$
  projections and scaled dot-product attention; multi-head attention lets several such
  operations specialize in parallel.
- The Transformer assembled self-attention, residual connections, and feed-forward layers
  into a fully parallelizable block, with positional encoding restoring order information.
- Causal masking turns self-attention into a next-token predictor; autoregressive,
  self-supervised pretraining on raw text (no labels needed) is what let one general model
  replace many task-specific ones.
- GPT-2 → GPT-3 showed this recipe keeps improving predictably with scale (parameters,
  data, compute), and that scale alone unlocks in-context learning.
- A base LLM predicts plausible continuations; instruction tuning (SFT) and RLHF/preference
  optimization are what turn it into an assistant that follows instructions and reflects
  human preferences.
- GPT-3.5 and ChatGPT sit at the end of that alignment pipeline — related to, but distinct
  from, the base GPT-3 model.
- Attention was the architectural breakthrough; the Transformer made it scalable;
  autoregressive pretraining made it general; scale unlocked broad capability; instruction
  tuning and preference optimization turned it into a useful conversational assistant.

## Related Articles

- [Neural Networks: Math Fundamentals](neural-networks-math-fundamentals.md) — the
  backpropagation/chain-rule machinery underlying every training step described here.

## Additional Resources

- Vaswani et al., ["Attention Is All You Need"](https://arxiv.org/abs/1706.03762) (2017).
- Devlin et al., ["BERT: Pre-training of Deep Bidirectional Transformers for Language
  Understanding"](https://arxiv.org/abs/1810.04805) (2018).
- Radford et al., ["Improving Language Understanding by Generative
  Pre-Training"](https://openai.com/research/language-unsupervised) (GPT, 2018) and
  ["Language Models are Unsupervised Multitask Learners"](https://openai.com/research/better-language-models)
  (GPT-2, 2019).
- Brown et al., ["Language Models are Few-Shot
  Learners"](https://arxiv.org/abs/2005.14165) (GPT-3, 2020).
- Ouyang et al., ["Training language models to follow instructions with human
  feedback"](https://arxiv.org/abs/2203.02155) (InstructGPT, 2022).
- Kaplan et al., ["Scaling Laws for Neural Language Models"](https://arxiv.org/abs/2001.08361)
  (2020); Hoffmann et al., ["Training Compute-Optimal Large Language Models"](https://arxiv.org/abs/2203.15556)
  (Chinchilla, 2022).
- [OpenAI: Introducing ChatGPT](https://openai.com/index/chatgpt/) — the original
  announcement, for historical context on the GPT-3.5/ChatGPT relationship.
