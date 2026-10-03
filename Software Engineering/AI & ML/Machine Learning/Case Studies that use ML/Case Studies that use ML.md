---
tags: [ai-ml, ai-ml/machine-learning, course]
aliases: [ML Case Studies]
updated: 2026-09-27
---
# Case Studies that use ML

> [!tldr] TL;DR
> Three worked examples run the same [[Major steps in ML|five-step ML process]] on very different problems: **supervised regression** (predict house prices from lot size and bedrooms), **unsupervised clustering** (discover book micro-genres from back-cover text) and **deep learning** (detect chemical spills in security camera footage with a CNN).

| | Example 1 | Example 2 | Example 3 |
|---|---|---|---|
| Task | Supervised · regression | Unsupervised · clustering | Supervised · image classification |
| Data | numeric house features + sale prices | 800 romance book descriptions (text) | camera images, spill / no spill |
| Preprocessing | clean, remove outliers, split | stop words, bag of words (vectorise) | pixels → numbers, split |
| Model | linear regression | k-means | convolutional neural network |
| Metric | RMS(E), manual threshold check | silhouette coefficient, manual review | precision & recall (not accuracy) |
| Inference | price estimate for new homes | clusters for an article | alert the clean-up team |

## The three cases

1. **Supervised learning**: use ML to predict housing prices in a neighbourhood from lot size and number of bedrooms → [[Example 1 Predicting Home Prices]]
2. **Unsupervised learning**: use ML to isolate book micro-genres by analysing the wording of the back-cover description → [[Example 2 Micro-genre Book Genre]]
3. **Deep neural network**: beyond the course's scope, but it shows how modern ML can analyse raw images from lab security cameras to detect chemical spills → [[Example 3 Detecting Spills]]

## Interview questions

> [!question]- Given a new problem, how do you choose between these three approaches?
> Look at the data and labels. Numeric target with labelled history → supervised regression. No labels, want groups → clustering. Unstructured data like images, audio or text with enough labelled examples → deep learning (often fine-tuning a pretrained model). Always start with the simplest approach that could work.

## Related
- [[Machine Learning]] (course index) · [[Major steps in ML]] · [[AI & ML Index]]

## Sources
- AWS DeepRacer Student: [Introduction to machine learning](https://student.deepracer.com/learn/85a3a216-703b-40c8-ad70-374767eba381)
