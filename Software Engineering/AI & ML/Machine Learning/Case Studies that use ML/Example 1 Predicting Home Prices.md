---
tags: [ai-ml, ai-ml/machine-learning, course]
aliases: [Predicting Home Prices, House Price Prediction]
updated: 2026-09-27
---
# Example 1 Predicting Home Prices

> [!tldr] TL;DR
> A **supervised regression** case study: predict a house's price from quantifiable features (lot size, bedrooms). Collect recent sales, clean out outliers, split 80/20, fit a **linear model**, evaluate with **root mean square error** plus a manual "how many were off by more than $50k?" check, then use it for inference on new homes.

House price prediction is one of the most common introductions to ML. Real estate appraisers already use quantifiable details about a home (number of rooms, lot size, year built) to estimate its value. You notice that relationship and believe ML could predict prices.

![[Untitled 4.png]]

## Step 1: Define the problem
*Can we estimate the price of a house based on lot size or the number of bedrooms?*

You can get sale prices for recently sold homes (or have them appraised), so you have **labels**: this is a **supervised** task. The target is a **continuous number**, so it's a **regression** task.

![[Untitled 5.png]]

## Step 2: Build the dataset
- **Collect**: examples of homes sold in the neighbourhood in the past year; pay an appraiser for homes whose price isn't known.
- **Explore**: confirm the data is numerical (most models work on numbers; convert any text).
- **Clean**: look for missing information and outliers, such as the 10-room mansion. You can handle outliers in several ways, or just remove them.
- **Visualise**: plot price against lot size to see the relationship.
- **Split**: the standard practice is 80% training, 20% test.

## Step 3: Train the model
As lot size increases, home value increases too. The relationship is simple enough for a **linear model**. With one input it's a **line**, with two a **plane**, with more a **hyperplane**; the intuition (a constant slope per input) stays the same.

```python
from sklearn.linear_model import LinearRegression
from sklearn.model_selection import train_test_split
from sklearn.metrics import root_mean_squared_error

X = df[["lot_size_m2", "bedrooms"]]
y = df["price"]
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=0)

model = LinearRegression().fit(X_train, y_train)
print(model.coef_, model.intercept_)     # $ per m², $ per bedroom, base price
```

## Step 4: Evaluate the model
- **Root mean square (RMS) error**: roughly the "average error" across the test set (it squares the errors first, so big misses weigh more). Lower is better. You compute it between the model's prediction and the true value for each test point.

![[Untitled 6.png]]

You want the data points as close to the fitted line as possible, meaning less net error.

![[Untitled 7.png]]

- **Interpreting results / manual threshold check**: a single RMS number is hard to judge. Many engineers count how many predictions were off by more than a threshold (say $50,000) to decide if the model is good enough.

```python
pred = model.predict(X_test)
rmse = root_mean_squared_error(y_test, pred)
off_by_50k = (abs(pred - y_test) > 50_000).mean()   # share of bad predictions
```

## Step 5: Use the model

![[Screenshot_2023-03-29_at_11.12.48_PM.png]]

- **Inference**: feed in a new home's features to get a price estimate. Keep monitoring: when the market moves (interest rates, a new school), retrain.

## Takeaways
1. Solving problems with ML is an evolving and iterative process.
2. High-quality data is essential.
3. Metrics are chosen for the specific use case.
4. Senior note: real price models add many more features (location is usually the strongest), check residuals for patterns, and watch for **concept drift** as the market changes.

## Terminology
- **Regression**: a supervised task that models the relationship between input variables and a continuous output.
- **Continuous**: floating-point values with an infinite range of possible values (as opposed to categorical/discrete).
- **Plane**: a flat 2-D surface (like a sheet of paper) that extends infinitely.
- **Hyperplane**: the generalisation of a line (2-D) or plane (3-D) to higher dimensions: a flat surface with one dimension fewer than the space it lives in. *(The original note said "a surface that contains more than two planes", which isn't right.)*

## Interview questions

> [!question]- What assumptions does linear regression make?
> A linear relationship between features and target, independent errors, constant error variance (homoscedasticity), roughly normal residuals for inference, and features that aren't highly collinear.

> [!question]- Why remove or treat outliers here?
> A few extreme values (a 10-room mansion) pull a least-squares line strongly because errors are squared, hurting predictions for typical homes. Remove them, cap them, model them separately, or use a robust loss.

## Related
- [[Case Studies that use ML]] · [[Machine Learning]] (course index) · [[AI & ML Index]]
- [[Step 3 Train the model]] · [[Step 4 Evaluate the model]]
- Next: [[Example 2 Micro-genre Book Genre]]

## Sources
- AWS DeepRacer Student: [Introduction to machine learning](https://student.deepracer.com/learn/85a3a216-703b-40c8-ad70-374767eba381)
- scikit-learn: [Ordinary Least Squares example](https://scikit-learn.org/stable/auto_examples/linear_model/plot_ols.html) · [root_mean_squared_error](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.root_mean_squared_error.html)
