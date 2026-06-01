# Recipe Ratings and Preparation Time

**Names:** Marcos Lopez and Bora Vanli

This project investigates whether recipe characteristics, especially preparation time, relate to user ratings on Food.com.

## Checkpoint 2 Progress

We are using the Recipes and Ratings dataset. Our current research question asks whether quick recipes, defined as recipes taking 30 minutes or less, receive different average ratings than longer recipes.

Our preliminary permutation test compares the mean average rating for quick recipes and longer recipes. Quick recipes had a mean average rating of about 4.645, while longer recipes had a mean average rating of about 4.610. The observed difference was about 0.035 rating points, and in 5,000 simulated permutations, none produced a difference at least as extreme as the observed result. This gives an approximate p-value less than 0.0002.

For prediction, we are building a regression model to predict `average_rating` from recipe characteristics known when the recipe is posted, such as preparation time and number of steps. Our baseline model uses `minutes` and `n_steps` in a sklearn pipeline with imputation, scaling, and Ridge regression. We plan to improve the model by adding parsed nutrition features, number of ingredients, tag-derived features, and a more flexible model such as a random forest with tuned hyperparameters.

