---
tags: [ai-ml, ai-ml/machine-learning]
aliases: [Concepts, ML Concepts, Vectors and Neural Networks, One Hot Encoding]
updated: 2026-09-27
---
# ML Core Concepts

> [!tldr] TL;DR
> The small vocabulary you need before anything else in ML: a **vector** is a list of numbers that places an item in a space; **embeddings** are vectors produced by a neural network so that similar things land close together; a **neural network** is layers of weighted nodes with activation functions; **transformers** are the neural network design behind today's LLMs; **one-hot encoding** turns categories into numbers a model can use. Each concept has its own deeper note, linked below.

## What is a vector?

A vector is an ordered list of numbers, for example `[12, 13, 19, 8, 9]`. Each number is a coordinate along one **dimension**, so the list as a whole is a **location in a space**, just like a row and column pick out a cell in a spreadsheet ("B7").

- 2 numbers: a point on a flat map.
- 3 numbers: a point in a room.
- 768 or 3,072 numbers: a point in a space you can't picture, but the maths (distance, angle) works the same.

```python
movie_a = [2.1, 0.9, 0.0]   # e.g. [action, romance, horror] scores
movie_b = [1.9, 1.1, 0.1]
movie_c = [0.0, 0.2, 2.8]
# a and b are close together; c is far away -> a and b are "similar"
```

## How do vector databases work?

Each vector in a vector database stands for one item: a word, a document chunk, an image, a movie. Items are placed along many dimensions (for movies: running time, genre, year, rating, overlapping audiences...). If the vectors are built well, **similar items end up clustered together**, so "find similar things" becomes "find the nearest points".

Uses:
- **Similarity and semantic search**: image search, "more like this", product, song or show recommendations.
- **Machine learning and deep learning**: connecting related items lets models do more complex tasks.
- **LLM apps**: vector search is how [[Retrieval-Augmented Generation (RAG)]] finds relevant text to put into a prompt.

> [!note] Correction to the original note
> The original said LLMs like ChatGPT "rely on vector databases" to understand language. That's not quite right: an LLM's understanding lives in its own weights and internal embeddings. Vector databases are an **external** tool that apps put *around* an LLM (for search, memory and RAG). See [[Embeddings & Semantic Search]] and [[Vector Databases & pgvector]].

## What are embeddings?

**Embeddings** are vectors produced by a trained neural network (an *embedding model*). You don't hand-pick the dimensions; the network learns them. Once trained, it can turn any new text or image into a vector, and those vectors power similarity search, clustering, classification and generative AI.

```python
# conceptual
embed("How do I reset my password?")   # -> [0.012, -0.093, ..., 0.044]  (e.g. 1,536 numbers)
embed("I forgot my login credentials")  # -> a nearby vector, even with no shared words
```

Deep dive: [[Embeddings & Semantic Search]].

## What is a neural network?

A neural network (artificial neural network) is a computing architecture loosely inspired by the brain. It is made of simple processing units called **nodes** (neurons) that pass numbers to each other, the way biological neurons pass electrical signals.

Neural networks are the engine of **deep learning**, the branch of ML that uses networks with many layers. With enough data, a deep network can, for example, recognise objects in photos it has never seen. (Deep learning can learn from labelled *or* unlabelled data; the original note implied it only uses unlabelled data.)

![[image.png]]

## How do neural networks work?

Nodes are arranged in at least three layers:

- an **input layer** (the features),
- one or more **hidden layers**,
- an **output layer** (the prediction).

Each node computes a **weighted sum** of its inputs plus a **bias**, then applies an **activation function**:

```text
output = activation(w1*x1 + w2*x2 + ... + wn*xn + b)
```

The original note described this as "if the result exceeds a threshold, pass data on, otherwise pass nothing". That's exactly how the classic **perceptron** (step activation) works. Modern networks use smooth activations like ReLU or sigmoid, so every node always outputs a number and the network can be trained with gradients. See [[Neural Networks & Deep Learning Basics]].

> [!example] Analogy: the Acme accounting department
> Managers approve expenses below $1,000, directors below $10,000, and the CFO anything above. An expense report enters at the bottom (input layer), each manager or director is a node that decides whether to pass it up (activation), and the final approval is the output. Just as a manager can consult a peer before escalating, networks can be wired in many ways, including connections that go sideways or backwards.

## Types of neural networks

There's no limit on how many nodes and layers a network can have or how they connect, so the list keeps growing. Two broad groups:

| | Shallow | Deep |
|---|---|---|
| Hidden layers | usually one | many |
| Cost | fast, cheap | needs more compute and data |
| Power | simple patterns | complex tasks (vision, language) |

A non-exhaustive list:

**Perceptron**: the simplest network, inputs connected straight to an output node.

![[Perceptron.png]]

**Multilayer perceptron (MLP)**: adds one or more hidden layers to the perceptron.

![[multilayer-perceptron-neural-network.png]]

**Feed-forward**: information only flows forward, input to output, no loops.

![[feed-forward-neural-network-diagram.png]]

**Recurrent (RNN)**: outputs can loop back and affect earlier nodes, giving the network a memory of previous steps. Used for sequences before transformers took over.

![[recurrent-neural-network-diagram.png]]

**Modular**: two or more networks combined to produce the output.

![[modular-neural-network.png]]

**Radial basis function (RBF)**: nodes use a radial basis function (output depends on distance from a centre point).

![[radial-basis-function-neural-network.png]]

**Liquid state machine**: nodes are randomly connected to each other (a form of reservoir computing).

![[liquid-state-machine-neural-network-diagram.png]]

**Residual (ResNet)**: "skip connections" let data jump ahead (identity mapping), adding early-layer output to later-layer output. This is what made very deep networks trainable, and transformers use the same trick.

![[residual-neural-network-diagram.png]]

Also important but not pictured: **convolutional networks (CNNs)** for images; see [[Neural Networks & Deep Learning Basics]].

## What is a transformer?

Transformers deserve their own mention because nearly every modern LLM is one. Introduced by Google researchers in 2017 ("Attention Is All You Need"), they use **self-attention**: every token in a sequence looks at every other token and decides how much each one matters for its own meaning.

Earlier text models (RNNs) processed words one at a time and tended to "forget" the start of a long sentence by the time they reached the end. Attention looks at the whole sequence at once, so transformers capture long-range relationships and can be trained in parallel on huge datasets. They are the core of generative AI and LLMs. Deep dive: [[Transformers & How LLMs Work]].

## What is one-hot encoding?

**One-hot encoding** converts a categorical column into binary columns, one per category. A `1` marks the category that applies, `0` everything else.

| colour | → | colour_red | colour_green | colour_blue |
|---|---|---|---|---|
| red | | 1 | 0 | 0 |
| blue | | 0 | 0 | 1 |
| green | | 0 | 1 | 0 |

```python
import pandas as pd
df = pd.DataFrame({"colour": ["red", "blue", "green"]})
pd.get_dummies(df, columns=["colour"])
```

Why not just use `red=1, green=2, blue=3`? Because the model would assume an order and distance (blue is "3 times" red), which is meaningless. Downside: many categories create many sparse columns; for high-cardinality data use embeddings or target encoding instead.

## Interview questions

> [!question]- What is the difference between a vector and an embedding?
> A vector is any list of numbers. An embedding is a vector *learned* by a model so that distance in the space reflects similarity in meaning. All embeddings are vectors; not all vectors are embeddings (a one-hot vector isn't).

> [!question]- Why do neural networks need activation functions?
> Without a non-linear activation, stacking layers just produces another linear function, so the network could only learn straight-line relationships. Non-linear activations (ReLU, sigmoid, tanh) let it approximate complex functions.

> [!question]- Why did transformers replace RNNs for language?
> Attention sees the whole sequence at once, captures long-range dependencies without the vanishing-gradient "forgetting" of RNNs, and can be trained in parallel on GPUs, which made scaling to huge datasets practical.

> [!question]- When would you use one-hot encoding and when not?
> Use it for nominal categories with a small number of values. Avoid it for high-cardinality features (thousands of IDs), where it creates huge sparse matrices; use embeddings, hashing or target encoding instead.

## Related
- [[Machine Learning]] (course index) · [[AI & ML Index]]
- [[Neural Networks & Deep Learning Basics]] · [[Transformers & How LLMs Work]]
- [[Embeddings & Semantic Search]] · [[Vector Databases & pgvector]] · [[Retrieval-Augmented Generation (RAG)]]
- [[Supervised vs Unsupervised Machine learning]]

## Sources
- Cloudflare Learning Center: [What is a vector database?](https://www.cloudflare.com/learning/ai/what-is-vector-database/) · [What is a neural network?](https://www.cloudflare.com/learning/ai/what-is-neural-network/) · [What are embeddings?](https://www.cloudflare.com/learning/ai/what-are-embeddings/)
- Google Research blog (2017): [Transformer: A Novel Neural Network Architecture for Language Understanding](https://blog.research.google/2017/08/transformer-novel-neural-network.html)
- Vaswani et al. (2017), [Attention Is All You Need](https://arxiv.org/abs/1706.03762)
- He et al. (2015), [Deep Residual Learning for Image Recognition](https://arxiv.org/abs/1512.03385)
- scikit-learn: [OneHotEncoder](https://scikit-learn.org/stable/modules/generated/sklearn.preprocessing.OneHotEncoder.html)
