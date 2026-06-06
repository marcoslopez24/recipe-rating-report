# Thirty Minutes to Five Stars

**Names:** Bora Vanli and Marcos Lopez

This project analyzes Food.com recipes and ratings to understand whether quicker recipes are rated differently from longer recipes, and whether recipe attributes can predict average rating.

## Introduction

The Recipes and Ratings dataset contains recipe-level information from Food.com and user interactions with those recipes. The recipe table includes preparation time, tags, nutrition, ingredients, and number of steps. The interactions table includes user ratings and reviews.

Our main question is: **Do quick recipes receive different average ratings than longer recipes?** We define quick recipes as recipes taking 30 minutes or less. This question matters because recipe users often choose recipes based on time, and recipe creators may want to know whether faster recipes are rewarded differently by reviewers.

The raw recipe table has 83,782 rows and 12 columns. The interaction table has 731,927 rows and 5 columns. The columns most relevant to our question are `minutes`, `n_steps`, `n_ingredients`, `tags`, `nutrition`, `rating`, and `average_rating`, where `average_rating` is computed by averaging user ratings for each recipe.

## Data Cleaning and Exploratory Data Analysis

We treated ratings of 0 as missing because the dataset uses 0 for interactions where a user left a review without assigning a rating, rather than as a true zero-star review. We then computed each recipe's average rating, merged those averages into the recipe table, parsed the list-like `nutrition`, `tags`, and `ingredients` columns, and created time buckets. For analyses involving preparation time, we removed nonpositive times and the top 0.5% of preparation times to reduce the impact of extreme time-entry errors on our visualizations and tests.

Cleaned data preview:

| name | minutes | time_bucket | n_steps | n_ingredients | calories | sugar_pdv | average_rating |
|---|---:|---|---:|---:|---:|---:|---:|
| 1 brownies in the world best ever | 40 | 31-60 min | 10 | 9 | 138.4 | 50.0 | 4.0 |
| 1 in canada chocolate chip cookies | 45 | 31-60 min | 12 | 11 | 595.1 | 211.0 | 5.0 |

Univariate distribution of recipe ratings:

<iframe src="assets/average-rating-distribution.html" width="100%" height="520" frameborder="0"></iframe>

Most recipe average ratings are very high, with many recipes near 5 stars. This concentration makes prediction difficult because the target has limited variation.

Bivariate relationship between cooking time and rating:

<iframe src="assets/rating-by-time-bucket.html" width="100%" height="560" frameborder="0"></iframe>

Quick recipes have a slightly higher mean average rating than longer recipes, though the difference is small on the 1-to-5 rating scale.

Aggregate table by cooking-time bucket:

| time_bucket | recipe_count | mean_rating | median_rating | mean_steps | mean_ingredients |
|---|---:|---:|---:|---:|---:|
| 0-30 min | 36,418 | 4.645 | 5.0 | 7.486 | 7.630 |
| 31-60 min | 24,632 | 4.617 | 5.0 | 10.420 | 9.923 |
| 61-120 min | 12,462 | 4.602 | 5.0 | 13.674 | 11.628 |
| 120+ min | 7,246 | 4.584 | 5.0 | 13.779 | 10.636 |

## Assessment of Missingness

The `review` column may be NMAR because whether a user writes review text may depend on their unobserved motivation, strength of opinion, or experience with the recipe. Additional user-level behavior data could help explain this missingness.

For formal missingness testing, we examined missingness in `average_rating`. A recipe has missing `average_rating` when it has no nonzero ratings. We found that this missingness depends on `n_steps`: recipes missing `average_rating` had about 1.493 more steps on average, with a permutation-test p-value below 0.001. We also found that missingness does not appear to depend on `sodium_pdv`; the p-value was 0.878.

<iframe src="assets/missingness-n-steps.html" width="100%" height="520" frameborder="0"></iframe>

## Hypothesis Testing

Null hypothesis: quick recipes, defined as recipes taking 30 minutes or less, and longer recipes, defined as recipes taking more than 30 minutes, have the same mean average rating.

Alternative hypothesis: quick recipes and longer recipes have different mean average ratings.

We used a two-sided permutation test with the difference in mean average rating as the test statistic. Quick recipes had a mean average rating of 4.6446, while longer recipes had a mean average rating of 4.6096. The observed difference was 0.0351 rating points. In 5,000 permutations, none of the simulated differences were at least as extreme as the observed value, so the p-value was less than 0.0002.

At a 5% significance level, we reject the null hypothesis. The data suggests that quick and longer recipes have different mean average ratings, with quick recipes rated slightly higher. The effect is statistically significant, but it is small in practical terms because the difference is only about 0.035 points on a 1-to-5 rating scale.

<iframe src="assets/hypothesis-test.html" width="100%" height="520" frameborder="0"></iframe>

## Framing a Prediction Problem

We frame a regression problem: predict `average_rating` using recipe information known when the recipe is posted. We use RMSE as the main evaluation metric because larger prediction errors should be penalized more than small errors. We also report MAE for interpretability.

We avoid features that would not be known at prediction time, such as `rating_count`, review text, or user identifiers.

## Baseline Model

The baseline model uses two quantitative features: `minutes` and `n_steps`. It has no nominal or ordinal features. The sklearn pipeline applies median imputation, standard scaling, and Ridge regression.

On the held-out test set, the baseline model had:

| Model | RMSE | MAE | R2 |
|---|---:|---:|---:|
| Baseline | 0.6309 | 0.4617 | 0.0001 |

This model is weak because average ratings are highly concentrated near 5 stars and preparation time plus step count alone do not explain much of the variation.

## Final Model

The final model uses a random forest regressor and adds engineered features that better describe recipe complexity and content. These include log preparation time, number of ingredients, steps per ingredient, calories per ingredient, parsed nutrition values, selected tag indicators, and selected ingredient indicators. These features are reasonable before fitting the model because recipe rating may depend on more than time alone: complexity, recipe type, nutrition, and common ingredients can all influence a user's experience.

We tuned the random forest with `GridSearchCV` using 3-fold cross-validation on the training data. The grid searched `max_depth`, `min_samples_leaf`, and `n_estimators`. The best parameters were `max_depth = 10`, `min_samples_leaf = 30`, and `n_estimators = 80`. After cross-validation selected the model, we evaluated it once on the held-out test set.

| Model | RMSE | MAE | R2 |
|---|---:|---:|---:|
| Baseline | 0.6309 | 0.4617 | 0.0001 |
| Final | 0.6262 | 0.4555 | 0.0148 |

The improvement is modest but real. This makes sense because the target is compressed near 5 stars, but the final model performs better than the baseline on the same train-test split.

<iframe src="assets/model-comparison.html" width="100%" height="520" frameborder="0"></iframe>

## Fairness Analysis

We evaluated whether the final model performs differently for quick recipes and longer recipes. Since this is a regression model, we compare RMSE across groups.

Null hypothesis: the model is fair; its RMSE for quick and longer recipes is roughly the same.

Alternative hypothesis: the model is unfair; its RMSE for quick and longer recipes is different.

The final model had an RMSE of 0.5945 for quick recipes and 0.6508 for longer recipes. The observed difference, RMSE(quick) minus RMSE(longer), was -0.0563. The permutation-test p-value was less than 0.001.

We reject the null hypothesis of equal performance. The model does not appear to perform worse for quick recipes; instead, it performs better for quick recipes and worse for longer recipes.

<iframe src="assets/fairness-test.html" width="100%" height="520" frameborder="0"></iframe>
