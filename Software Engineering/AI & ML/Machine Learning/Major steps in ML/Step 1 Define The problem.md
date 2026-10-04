---
title: "Step 1 Define The problem"
tags: [ai-ml, ai-ml/machine-learning, course]
aliases: [Define the ML Problem, ML Step 1]
updated: 2026-09-27
---
# Step 1 Define The problem

<Note>
**TL;DR**

Start every ML project by (A) turning a vague business question into a **very specific, measurable** prediction task, and (B) identifying the **ML task type**. Whether your data has **labels** decides supervised vs unsupervised; whether the label is a **category or a number** decides classification vs regression.
</Note>

## Key terms
- **Label**: the answer already present in the data (the thing you want to predict).
- **Labelled data**: data where you already know the target answer.
- **Unlabelled data**: data with no labels; the model gets no solution during training.
- **Categorical label**: a discrete set of possible values, e.g. "is a cat" / "is not a cat".
- **Continuous (regression) label**: no fixed set of values; potentially unlimited possibilities (a number).
- **Discrete**: a statistics term for an outcome that takes only a finite number of values (days of the week).
- **Clustering**: an unsupervised task that finds naturally occurring groups in the data.

## How do you start a machine learning task?

### Step A: define a very specific task
"How do I increase sales?" is a valid question, but the opposite of a specific task. An ML practitioner would narrow it down to something testable, like:

> *Does adding a $0.50 charge for organic flavours to a snow cone increase snow cone sales?*

A good problem statement names **the prediction**, **the input data**, **the decision it drives**, and **how you'll measure success**.

| Vague | Specific |
|---|---|
| Increase sales | Predict daily snow cone sales from weather and price, to plan stock |
| Reduce churn | Predict which subscribers will cancel in the next 30 days, so support can call them |
| Find good books | Group this year's romance titles into micro-genres from their descriptions |

Also ask early: is ML even needed? A simple rule, a SQL query, or (today) a prompted LLM may be enough. See [Fine-tuning vs RAG vs Prompting](/Software%20Engineering/AI%20%26%20ML/Fine-tuning%20vs%20RAG%20vs%20Prompting).

### Step B: identify the ML task
All models take data as input, but their outputs differ, and the type of output groups them into tasks. The **presence or absence of labels** is the first split.

![Screenshot_2023-03-29_at_12.31.50_AM](/Attachments/Screenshot_2023-03-29_at_12.31.50_AM.png)

{/* diagram source: /diagrams/diagram-016.mmd */}
![diagram](/diagrams/diagram-016.svg)

## Supervised learning
A task is **supervised** if you use **labelled data**: data that already contains the solutions (labels).

![Screenshot_2023-03-29_at_12.38.22_AM](/Attachments/Screenshot_2023-03-29_at_12.38.22_AM.png)

- **Classification (categorical label)**: e.g. identify the type of flower from a picture. You train on images labelled with the flower categories. Categorical labels usually mean a classification task.
- **Regression (continuous label)**: e.g. predict how many snow cones will be sold on a day. The label is a number that could, in theory, take any value.

## Unsupervised learning
A task is **unsupervised** if you use **unlabelled data**: the model gets no labels or solutions during training.

![Screenshot_2023-03-29_at_12.38.59_AM](/Attachments/Screenshot_2023-03-29_at_12.38.59_AM.png)

- **Clustering** finds natural groupings when you have no labels but want to extract structure.

<Info>
**Identifying book micro-genres**

You work for a book recommendation company. You're fairly confident micro-genres exist (say, *Teen Vampire Romance*), but you don't know which ones. Without labels you can't use supervised learning, but clustering on the words in book descriptions may reveal the groups. Full walkthrough: [Example 2 Micro-genre Book Genre](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Case%20Studies%20that%20use%20ML/Example%202%20Micro-genre%20Book%20Genre).
</Info>

## Senior perspective
- Most failed ML projects fail at step 1: the wrong target, a metric that doesn't match the business goal, or a label that leaks the answer (for example using "refund issued" to predict "complaint filed").
- Decide the **baseline** up front (a rule, the current process, or a trivial model like "predict the average") so you can prove the model adds value.
- Ask how predictions will be **used**, what latency and cost are acceptable, and what an error costs (false positive vs false negative). This drives the metric in [Step 4 Evaluate the model](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Major%20steps%20in%20ML/Step%204%20Evaluate%20the%20model).

## Interview questions

<Accordion title="How do you decide between classification and regression?">
By the label: a small set of discrete categories means classification; a continuous number means regression. Some problems can be framed either way (predict price, or predict a price band); pick the framing that matches the decision being made.
</Accordion>

<Accordion title="What makes a good ML problem statement?">
Specific target, known inputs available at prediction time, a clear decision that uses the prediction, a success metric tied to business value, and a baseline to beat.
</Accordion>

## Related
- [Major steps in ML](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Major%20steps%20in%20ML/Major%20steps%20in%20ML) · [Machine Learning](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Machine%20Learning) (course index) · [AI & ML Index](/Software%20Engineering/AI%20%26%20ML/AI%20%26%20ML%20Index)
- Next: [Step 2 Building Dataset](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Major%20steps%20in%20ML/Step%202%20Building%20Dataset)
- [Supervised vs Unsupervised Machine learning](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Supervised%20vs%20Unsupervised%20Machine%20learning)

## Sources
- AWS DeepRacer Student: [Introduction to machine learning](https://student.deepracer.com/learn/85a3a216-703b-40c8-ad70-374767eba381)
- Google: [Introduction to Machine Learning Problem Framing](https://developers.google.com/machine-learning/problem-framing)
