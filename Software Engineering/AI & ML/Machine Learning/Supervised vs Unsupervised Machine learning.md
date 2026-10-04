---
title: "Supervised vs Unsupervised Machine learning"
tags: [ai-ml, ai-ml/machine-learning, course]
aliases: [Supervised Learning, Unsupervised Learning, Regression vs Classification, Clustering]
updated: 2026-09-27
---
# Supervised vs Unsupervised Machine learning

<Note>
**TL;DR**

**Supervised learning** learns a mapping from inputs `X` to known answers `y` (labels); it splits into **regression** (predict a number) and **classification** (predict a category). **Unsupervised learning** gets no labels and must find structure by itself: **clustering**, **anomaly detection** and **dimensionality reduction**. Supervised learning is by far the most used in practice. (Notes from the DeepLearning.AI / Coursera Machine Learning Specialization.)
</Note>

> The more runs the better (more time to learn). *Within limits: past a point extra training overfits; see [Step 4 Evaluate the model](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Major%20steps%20in%20ML/Step%204%20Evaluate%20the%20model).*

## Types of machine learning
- **Supervised learning** (most used in real-world applications)
- **Unsupervised learning**
- Others covered in the course: **recommender systems** and **reinforcement learning**

## Supervised learning

Algorithms that learn from examples **that include the correct answers**. The model learns to map input `X` to output label `y`, then predicts `y` for inputs it has never seen.

| Input (X) | Output (Y) | Application |
| --- | --- | --- |
| Email | Spam? (0/1) | Spam filter |
| Audio | Text transcript | Speech recognition |
| English | Spanish | Machine translation |
| Image, radar info | Position of other cars | Self-driving cars |
| Ad, user info | Click? (0/1) | Online advertising |
| Image of a phone | Defect? (0/1) | Visual inspection |

### Regression
Predict a **number** from infinitely many possible outputs (e.g. house price from size).

![image 1](/Attachments/image%201.png)

### Classification
Predict a **class/category** from a small, finite set of possible outputs (e.g. tumour benign or malignant). The classes can be numbers, strings or anything else; what makes it classification is the small number of possible outputs. With 2+ inputs the model learns a **decision boundary** that separates the classes.

![image 2](/Attachments/image%202.png)

| | Regression | Classification |
|---|---|---|
| Output | continuous number | one of a few categories |
| Example | price, temperature, sales | spam/not spam, cat/dog/bird |
| Typical loss | squared error (MSE) | cross-entropy / log loss |
| Typical metrics | MAE, RMSE, R² | accuracy, precision, recall, F1 |

## Unsupervised learning

Algorithms that learn from data **without labels**: there is no "right answer" in the data, so the algorithm has to find structure on its own.

![image 3](/Attachments/image%203.png)

Types of unsupervised learning:
- **Clustering**: group similar data points together (a cluster is a group of points that are close to each other).
- **Anomaly detection**: find unusual data points (fraud, faulty machines).
- **Dimensionality reduction**: compress data into fewer numbers while losing as little information as possible (e.g. PCA).

![image 4](/Attachments/image%204.png)

Example: **Google News** clusters thousands of articles about the same story (shared words like "panda", "twin", "zoo") without anyone labelling them.

![image 5](/Attachments/image%205.png)

Other examples: grouping customers into market segments, grouping genes with similar expression patterns.

```python
from sklearn.cluster import KMeans
X = [[1, 2], [1, 4], [1, 0], [10, 2], [10, 4], [10, 0]]
km = KMeans(n_clusters=2, n_init="auto").fit(X)
km.labels_   # e.g. [1 1 1 0 0 0]: two groups found without labels
```

## Where LLMs fit
LLMs are pretrained with **self-supervised** learning: the "label" is simply the next token of the text itself, so no human labelling is needed. They are then refined with supervised fine-tuning and reinforcement learning. See [Transformers & How LLMs Work](/Software%20Engineering/AI%20%26%20ML/Transformers%20%26%20How%20LLMs%20Work).

## Interview questions

<Accordion title="Is predicting tomorrow's temperature regression or classification?">
Regression (a continuous number). Predicting "hot / mild / cold" would be classification.
</Accordion>

<Accordion title="You have millions of support tickets and no labels. What can you do?">
Unsupervised clustering (or topic modelling, or clustering embeddings) to discover groups; then label a sample of each cluster and train a supervised classifier. Today you might also use an LLM to label a sample (weak supervision) and check it by hand.
</Accordion>

<Accordion title="What is self-supervised learning?">
Labels are generated automatically from the data itself, for example masking a word or predicting the next token. It's how LLMs and embedding models are pretrained on huge unlabelled corpora.
</Accordion>

## Related
- [Machine Learning](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Machine%20Learning) (course index) · [AI & ML Index](/Software%20Engineering/AI%20%26%20ML/AI%20%26%20ML%20Index)
- [Introduction to Machine Learning](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Introduction%20to%20Machine%20Learning) · [Step 1 Define The problem](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Major%20steps%20in%20ML/Step%201%20Define%20The%20problem)
- [Example 1 Predicting Home Prices](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Case%20Studies%20that%20use%20ML/Example%201%20Predicting%20Home%20Prices) (regression) · [Example 2 Micro-genre Book Genre](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Case%20Studies%20that%20use%20ML/Example%202%20Micro-genre%20Book%20Genre) (clustering)

## Sources
- DeepLearning.AI / Coursera: [Machine Learning Specialization](https://www.coursera.org/specializations/machine-learning-introduction) (Course 1: Supervised Machine Learning: Regression and Classification)
- scikit-learn: [Supervised learning](https://scikit-learn.org/stable/supervised_learning.html) · [Clustering](https://scikit-learn.org/stable/modules/clustering.html)
