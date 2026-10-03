---
tags: [ai-ml, ai-ml/machine-learning, course]
aliases: [Book Micro-genres, K-means Example, Bag of Words]
updated: 2026-09-27
---
# Example 2 Micro-genre Book Genre

> [!tldr] TL;DR
> An **unsupervised clustering** case study: find hidden "micro-genres" among 800 romance books using only their back-cover descriptions. Clean the text (lowercase, normalise verb tense, drop punctuation and **stop words**), **vectorise** it as a **bag of words**, run **k-means** for many values of k, pick k with the **silhouette coefficient** (k = 19 here), then inspect clusters by hand.

You do editorial work for a book recommendation company and want to write about the year's biggest book trends. You believe "micro-genres" exist and that description text can reveal them. **Clustering** lets you test that hypothesis without any labels.

## Step 1: Define the problem
*Is it possible to find clusters of similar books based on the presence of common words in the book descriptions?*

![[Screenshot_2023-03-31_at_6.11.45_PM.png]]

There are no labels (nobody has tagged micro-genres), so this is **unsupervised learning**.

## Step 2: Build the dataset
You gather the description text of **800 romance books** published this year.

![[Screenshot_2023-03-31_at_6.12.48_PM.png]]

Dataset: collected → explored → cleaned → **vectorised** (transformed into numbers).

**Cleaning and exploration**: capitalisation and verb tense don't matter here, so you lowercase everything and convert verbs to the same tense with an NLP library (lemmatisation). You remove punctuation and low-meaning words like *a* and *the*, called **stop words**.

**Preprocessing (vectorisation)**: models need numbers, so you turn each description into a **bag of words**: one column per word in the vocabulary, the value is how often the word appears.

```python
from sklearn.feature_extraction.text import CountVectorizer
docs = ["A vampire falls for a teen girl", "The duke and the governess fall in love"]
vec = CountVectorizer(stop_words="english")
X = vec.fit_transform(docs)
vec.get_feature_names_out()   # ['duke', 'fall', 'falls', 'girl', 'governess', 'love', 'teen', 'vampire']
X.toarray()                   # word counts per book
```

## Step 3: Train the model
You pick **k-means**, a common clustering model. Its hyperparameter **k** is how many clusters it will look for. You don't know how many micro-genres exist, so you train many times with different k. The graphs show k = 2 and k = 3.

![[Untitled 8.png]]

```python
from sklearn.cluster import KMeans
from sklearn.metrics import silhouette_score

scores = {}
for k in range(2, 30):
    km = KMeans(n_clusters=k, n_init="auto", random_state=0).fit(X)
    scores[k] = silhouette_score(X, km.labels_)
best_k = max(scores, key=scores.get)
```

## Step 4: Evaluate the model
The **silhouette coefficient** describes how well the data was clustered. Plotting it for each k shows the best value at **k = 19**.

![[Untitled 9.png]]

**Manual inspection** matters too: one cluster is clearly "paranormal teen romance", a known industry trend, which gives you some confidence in the approach. Not every cluster will be that clean, but it's good enough to look for article material.

## Step 5: Use the model
Use the clusters to explore trends for the article, continuing to inspect them by hand to make sure they're meaningful.

## Takeaways
1. Some problems need the data converted into a machine-readable format; here words became numbers through **data vectorisation**.
2. ML needs iteration: the model was trained many times with different k, and the silhouette coefficient picked the best one.
3. During inference you keep inspecting clusters to confirm the model produces useful results.

> [!info] How you'd do this today
> Instead of bag of words, you'd embed each description with a text **embedding model** and cluster the vectors. Embeddings capture meaning ("vampire" and "undead" end up close), so clusters are much better. An LLM can then name each cluster. See [[Embeddings & Semantic Search]].

## Terminology
- **Bag of words**: extracts features from text by counting how many times each word appears in a document, producing a numeric dataset. Word order is lost.
- **Data vectorisation**: converting non-numeric data into numbers a model can use.
- **Silhouette coefficient**: a score from −1 to 1 describing clusters. Near 1: well-separated, distinct clusters. Near 0: overlapping clusters. Below 0: points probably assigned to the wrong cluster.
- **Stop words**: words removed by NLP tools when building a dataset. There's no single universal list.

## Interview questions

> [!question]- How do you choose k in k-means?
> Try a range of k and compare a metric: the silhouette score (higher is better), or the elbow in within-cluster sum of squares. Then sanity-check the clusters with domain knowledge; the "best" k mathematically may not be the most useful.

> [!question]- Limitations of bag of words?
> It ignores word order and meaning, treats synonyms as unrelated, and produces huge sparse vectors. TF-IDF helps weight rare words; embeddings solve the meaning problem.

## Related
- [[Case Studies that use ML]] · [[Machine Learning]] (course index) · [[AI & ML Index]]
- Previous: [[Example 1 Predicting Home Prices]] · Next: [[Example 3 Detecting Spills]]
- [[Supervised vs Unsupervised Machine learning]] · [[Embeddings & Semantic Search]]

## Sources
- AWS DeepRacer Student: [Introduction to machine learning](https://student.deepracer.com/learn/85a3a216-703b-40c8-ad70-374767eba381)
- scikit-learn: [K-means](https://scikit-learn.org/stable/modules/clustering.html#k-means) · [Silhouette analysis](https://scikit-learn.org/stable/auto_examples/cluster/plot_kmeans_silhouette_analysis.html) · [Text feature extraction](https://scikit-learn.org/stable/modules/feature_extraction.html#text-feature-extraction)
