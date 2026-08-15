---
tags:

- machine-learning
- neural-networks
- math
- fundamentals

---

# Neural Networks: Math Fundamentals

The math behind neural networks isn't exotic — it's linear algebra (how a layer transforms
its input) plus calculus (how we figure out which direction to nudge the weights). This is
a living note: the goal is to build the chain from "a single neuron" up to "backpropagation
updates every weight in the network," with enough detail that each step is derivable, not
just quotable.

## 1. The Building Block: A Single Neuron

A neuron takes an input vector, computes a weighted sum, adds a bias, and passes the
result through a non-linear **activation function**:

$$
z = \mathbf{w} \cdot \mathbf{x} + b, \qquad a = \sigma(z)
$$

- $\mathbf{x} \in \mathbb{R}^n$ — the input vector (features, or the previous layer's
  output).
- $\mathbf{w} \in \mathbb{R}^n$ — learned weights, one per input.
- $b$ — a learned bias (a shift term).
- $\mathbf{w} \cdot \mathbf{x}$ — the **dot product**: $\sum_i w_i x_i$. This is where the
  linear algebra comes in — it's a weighted vote over the inputs.
- $\sigma$ — the activation function (see below), which introduces non-linearity. Without
  it, stacking layers would just collapse into one big linear function, no matter how many
  layers you add.

## 2. Layers as Matrix Multiplication

A layer is just many neurons applied to the same input, and stacking their weight vectors
into a matrix turns the whole layer into one matrix multiplication:

$$
\mathbf{z} = W\mathbf{x} + \mathbf{b}, \qquad \mathbf{a} = \sigma(\mathbf{z})
$$

- $W \in \mathbb{R}^{m \times n}$ — row $i$ is the weight vector of neuron $i$; $m$ is the
  number of neurons in the layer, $n$ is the number of inputs.
- $\mathbf{b} \in \mathbb{R}^m$ — one bias per neuron.
- $\sigma$ is applied element-wise to each entry of $\mathbf{z}$.

This is why frameworks like NumPy/PyTorch lean so heavily on matrix multiplication: a full
forward pass through a network is just a sequence of `matmul → add bias → activation`
steps, one per layer, batched across many examples at once.

## 3. Common Activation Functions

| Name | Formula | Notes |
|---|---|---|
| Sigmoid | $\sigma(z) = \dfrac{1}{1 + e^{-z}}$ | Squashes to $(0, 1)$; saturates for large $\lvert z \rvert$, which slows learning (vanishing gradient). |
| Tanh | $\tanh(z) = \dfrac{e^z - e^{-z}}{e^z + e^{-z}}$ | Squashes to $(-1, 1)$; zero-centered, same saturation issue as sigmoid. |
| ReLU | $\text{ReLU}(z) = \max(0, z)$ | Cheap, doesn't saturate for $z > 0$; default choice for hidden layers in most modern networks. |
| Softmax | $\text{softmax}(\mathbf{z})_i = \dfrac{e^{z_i}}{\sum_j e^{z_j}}$ | Turns a vector of scores into a probability distribution; used on the output layer for multi-class classification. |

## 4. Loss Functions

The loss function measures how wrong the network's prediction $\hat{y}$ is compared to the
true label $y$. It's the quantity we're trying to minimize.

**Mean Squared Error** (regression):

$$
\mathcal{L}(y, \hat{y}) = \frac{1}{N}\sum_{i=1}^N (y_i - \hat{y}_i)^2
$$

**Binary Cross-Entropy** (binary classification, $\hat{y} \in (0, 1)$ from a sigmoid
output):

$$
\mathcal{L}(y, \hat{y}) = -\big[y \log(\hat{y}) + (1 - y)\log(1 - \hat{y})\big]
$$

Cross-entropy is preferred over MSE for classification because it penalizes confident
wrong predictions much more sharply, and pairs well with sigmoid/softmax outputs (their
gradients simplify nicely together, as shown below).

## 5. The Calculus: Gradients and the Chain Rule

Training a network means finding weights that minimize the loss. We do this with
**gradient descent**: repeatedly nudge each weight in the direction that decreases the
loss the fastest, i.e. opposite to the gradient.

$$
w \leftarrow w - \eta \frac{\partial \mathcal{L}}{\partial w}
$$

- $\eta$ — the **learning rate**, how big a step to take.
- $\dfrac{\partial \mathcal{L}}{\partial w}$ — how much the loss changes if we nudge $w$
  slightly. This is the derivative we need for *every* weight in the network.

A network is a composition of functions (layer after layer), so computing
$\partial \mathcal{L} / \partial w$ for an early layer means differentiating through every
layer that comes after it. That's exactly what the **chain rule** is for:

$$
\frac{d}{dx} f(g(x)) = f'(g(x)) \cdot g'(x)
$$

Applied to a network, if $\mathcal{L}$ depends on $w$ only through $z = wx + b$ and
$a = \sigma(z)$:

$$
\frac{\partial \mathcal{L}}{\partial w} = \frac{\partial \mathcal{L}}{\partial a} \cdot \frac{\partial a}{\partial z} \cdot \frac{\partial z}{\partial w}
$$

Each factor is a *local* derivative — easy to compute on its own. The chain rule is what
lets us chain those local derivatives together across an arbitrarily deep network.

## 6. Backpropagation, Derived on a Tiny Network

Take the smallest useful example: one input, one hidden neuron, one output neuron, sigmoid
activations, MSE loss.

**Forward pass:**

$$
z_1 = w_1 x + b_1, \quad a_1 = \sigma(z_1)
$$

$$
z_2 = w_2 a_1 + b_2, \quad \hat{y} = \sigma(z_2)
$$

$$
\mathcal{L} = (y - \hat{y})^2
$$

**Backward pass** — we want $\partial \mathcal{L}/\partial w_2$ and
$\partial \mathcal{L}/\partial w_1$, working from the output backward (hence
"backpropagation").

Step 1 — output layer, apply the chain rule through $\mathcal{L} \to \hat{y} \to z_2$:

$$
\frac{\partial \mathcal{L}}{\partial z_2} = \frac{\partial \mathcal{L}}{\partial \hat{y}} \cdot \frac{\partial \hat{y}}{\partial z_2} = -2(y - \hat{y}) \cdot \sigma(z_2)\big(1 - \sigma(z_2)\big)
$$

using $\sigma'(z) = \sigma(z)(1 - \sigma(z))$, a convenient property of the sigmoid.

Then, since $z_2 = w_2 a_1 + b_2$:

$$
\frac{\partial \mathcal{L}}{\partial w_2} = \frac{\partial \mathcal{L}}{\partial z_2} \cdot \frac{\partial z_2}{\partial w_2} = \frac{\partial \mathcal{L}}{\partial z_2} \cdot a_1
$$

Step 2 — hidden layer. The key idea of backprop: $\partial \mathcal{L}/\partial z_2$
(already computed) tells us how the loss flows back into $a_1$, and from there into $z_1$
and $w_1$:

$$
\frac{\partial \mathcal{L}}{\partial a_1} = \frac{\partial \mathcal{L}}{\partial z_2} \cdot \frac{\partial z_2}{\partial a_1} = \frac{\partial \mathcal{L}}{\partial z_2} \cdot w_2
$$

$$
\frac{\partial \mathcal{L}}{\partial z_1} = \frac{\partial \mathcal{L}}{\partial a_1} \cdot \sigma(z_1)\big(1 - \sigma(z_1)\big)
$$

$$
\frac{\partial \mathcal{L}}{\partial w_1} = \frac{\partial \mathcal{L}}{\partial z_1} \cdot x
$$

Notice the pattern: each layer's gradient is "the gradient flowing back from the next
layer" multiplied by "how this layer's output depends on its own input/weights." That
pattern is exactly what generalizes to networks with arbitrarily many layers — it's just
the chain rule applied once per layer, propagated backward from the loss.

## 7. Worked Numeric Example

Using the tiny network above with $x = 1$, $w_1 = 0.5$, $b_1 = 0$, $w_2 = 0.5$, $b_2 = 0$,
target $y = 1$:

```python
import math


def sigmoid(z: float) -> float:
    return 1 / (1 + math.exp(-z))


x, y = 1.0, 1.0
w1, b1 = 0.5, 0.0
w2, b2 = 0.5, 0.0

# Forward pass
z1 = w1 * x + b1
a1 = sigmoid(z1)
z2 = w2 * a1 + b2
y_hat = sigmoid(z2)
loss = (y - y_hat) ** 2

# Backward pass
d_loss_d_yhat = -2 * (y - y_hat)
d_yhat_d_z2 = y_hat * (1 - y_hat)
d_loss_d_z2 = d_loss_d_yhat * d_yhat_d_z2

d_loss_d_w2 = d_loss_d_z2 * a1

d_loss_d_a1 = d_loss_d_z2 * w2
d_a1_d_z1 = a1 * (1 - a1)
d_loss_d_z1 = d_loss_d_a1 * d_a1_d_z1

d_loss_d_w1 = d_loss_d_z1 * x

print(f"loss={loss:.4f}  dL/dw1={d_loss_d_w1:.4f}  dL/dw2={d_loss_d_w2:.4f}")
```

Running this prints the loss and both weight gradients — the exact quantities a gradient
descent step (`w -= learning_rate * gradient`) would use to update `w1` and `w2`. Every
real framework (PyTorch, TensorFlow) does the same chain-rule bookkeeping automatically via
autodiff; this by-hand version is what's happening underneath.

## Summary

- A neuron is a dot product plus a bias, squashed by a non-linear activation.
- A layer is a matrix multiplication — this is why linear algebra matters so much for
  performance (GPUs are matrix-multiplication machines).
- The loss function measures error; gradient descent minimizes it by stepping opposite the
  gradient.
- Backpropagation is the chain rule applied once per layer, computing gradients from the
  output backward to the input.

## Related Articles

- More machine learning notes will land in this section as they're written — see
  [Machine Learning & AI](index.md).

## Additional Resources

- [3Blue1Brown: Neural Networks series](https://www.3blue1brown.com/topics/neural-networks) — excellent visual intuition for everything above.
- [CS231n: Backpropagation notes](https://cs231n.github.io/optimization-2/) — a deeper, more rigorous derivation.
