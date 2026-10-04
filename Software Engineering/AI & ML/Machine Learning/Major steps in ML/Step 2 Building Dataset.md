---
title: "Step 2 Building Dataset"
tags: [ai-ml, ai-ml/machine-learning, course]
aliases: [Build the Dataset, ML Step 2, Data Preparation]
updated: 2026-09-27
---
# Step 2 Building Dataset

<Note>
**TL;DR**

*Working with data is perhaps the most overlooked, yet most important, step of the ML process.* Building a dataset has four parts: **collect** data that matches the problem, **inspect** it (outliers, missing values, wrong formats), compute **summary statistics** (scope, scale, shape), and **visualise** it. Garbage in, garbage out.
</Note>

![Screenshot_2023-03-29_at_12.50.44_AM](/Attachments/Screenshot_2023-03-29_at_12.50.44_AM.png)

## The four parts of working with data

![Screenshot_2023-03-29_at_12.52.44_AM](/Attachments/Screenshot_2023-03-29_at_12.52.44_AM.png)

### 1. Data collection
Gather data related to the problem (labelled or unlabelled). It can be as simple as running the right SQL queries or as complex as building a custom web scraper. You might even need to run a model over your data to generate labels.

The fundamental question: *does the data you collected match the ML task and problem you defined?*

### 2. Data inspection
Explore the dataset and look for:
- **Outliers**
- **Missing or incomplete** data
- Data that needs **transforming or preprocessing** into the format your model expects (text to numbers, dates to features, units made consistent)

### 3. Summary statistics
Check that your data fits the assumptions of the model you chose. Tools such as the **mean**, **interquartile range (IQR)** and **standard deviation** reveal:
- **trends** in the data
- the **scale** of the data
- the **shape** (distribution) of the data

### 4. Data visualization
Plots help you spot outliers and trends, and communicate findings to stakeholders.

## Worked example with pandas

```python
import pandas as pd

df = pd.read_csv("snow_cone_sales.csv")

df.info()                 # column types, non-null counts -> missing data
df.describe()             # mean, std, min, quartiles, max -> scale and shape
df.isna().sum()           # missing values per column

# impute missing temperatures with the median
df["temp_c"] = df["temp_c"].fillna(df["temp_c"].median())

# flag outliers with the IQR rule
q1, q3 = df["sales"].quantile([0.25, 0.75])
iqr = q3 - q1
outliers = df[(df["sales"] < q1 - 1.5 * iqr) | (df["sales"] > q3 + 1.5 * iqr)]

df.plot.scatter(x="temp_c", y="sales")   # visualise the relationship
```

## Senior perspective
- **Split before you explore too much.** Hold out the test set early so choices you make while cleaning don't leak information from it (data leakage).
- **Leakage** is the most expensive data bug: a feature that won't exist at prediction time (e.g. "delivery date" when predicting late orders) makes offline metrics look great and production terrible.
- **Label quality** matters more than model choice. Measure inter-annotator agreement and audit samples.
- **Representativeness**: does the training data match production traffic (seasons, regions, device types)? Bias in data becomes bias in predictions.
- Version datasets like code (DVC, lakeFS, table snapshots) so a model can always be traced back to the exact data that trained it.

## Key terms
- **Impute**: statistical techniques that fill in missing values (mean, median, mode, model-based).
- **Outliers**: data points that are significantly different from the rest of the sample.

## Recap
1. Good data is the key to answering the problem you defined.
2. To build a good dataset: **collect** the data; **inspect** it for outliers, missing values and needed reformatting; use **summary statistics** to understand scope, scale and shape; and use **visualisations** to check for outliers and trends.

## Interview questions

<Accordion title="What is data leakage and how do you prevent it?">
Information from outside the training period or from the target itself sneaks into the features, inflating offline results. Prevent it by splitting data first (by time for time series), fitting preprocessing only on the training split, and checking each feature is available at prediction time.
</Accordion>

<Accordion title="How would you handle missing values?">
First understand why they're missing. Options: drop rows or columns (if few or uninformative), impute (mean/median/mode, or a model), or add an "is_missing" indicator since missingness can be a signal. Fit imputers on training data only.
</Accordion>

## Related
- [Major steps in ML](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Major%20steps%20in%20ML/Major%20steps%20in%20ML) · [Machine Learning](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Machine%20Learning) (course index) · [AI & ML Index](/Software%20Engineering/AI%20%26%20ML/AI%20%26%20ML%20Index)
- Previous: [Step 1 Define The problem](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Major%20steps%20in%20ML/Step%201%20Define%20The%20problem) · Next: [Step 3 Train the model](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Major%20steps%20in%20ML/Step%203%20Train%20the%20model)

## Sources
- AWS DeepRacer Student: [Introduction to machine learning](https://student.deepracer.com/learn/85a3a216-703b-40c8-ad70-374767eba381)
- scikit-learn: [Outlier detection on a real data set (wine)](https://scikit-learn.org/stable/auto_examples/applications/plot_outlier_detection_wine.html) · [Imputation of missing values](https://scikit-learn.org/stable/modules/impute.html) · [Common pitfalls: data leakage](https://scikit-learn.org/stable/common_pitfalls.html)
- pandas: [DataFrame.describe](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.describe.html)
