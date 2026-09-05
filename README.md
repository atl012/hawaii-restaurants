# What Makes a Hawaii Restaurant a 5-Star Success?

**By**: (add your name(s) here)

*DSC 80 Final Project*

---

## Introduction

Imagine you want to open a restaurant in Hawaii. Before committing anything, you would want to know: **what business attributes are associated with a higher Google rating, and does location matter?**

This project investigates: *if I am a prospective restaurant owner in Hawaii, what makes a restaurant successful (measured by average Google rating), and how does this influence by price point, cuisine style, and zip code?* Anyone thinking about opening a restaurant should care about this question, since it turns scattered Google reviews into concrete patterns.

I use the **Hawaii Google Maps Reviews** dataset (`meta-Hawaii.json`), which contains **21,507 businesses** across Hawaii from Google Maps. I didn't use the review-level file in this analysis because the business-level file already contains everything my question and prediction need (category, price, location, amenities, and the outcome, `avg_rating`, itself).

Restaurants are the largest business type in the dataset (4,301 of 21,507 businesses, 20%), so I restrict the analysis to businesses whose category tags contain the word "restaurant".

**Relevant columns:**

| Column | Description |
|---|---|
| `name` | Business name |
| `category` | List of Google category tags (used to identify restaurants and cuisine style) |
| `address` | Free-text address (to extract the zip code) |
| `avg_rating` | The business's average Google rating (the outcome of interest) |
| `num_of_reviews` | Number of Google reviews |
| `price` | Google's price-tier symbol (e.g. `$`, `$$`) |
| `hours` | Weekly operating hours |
| `MISC` | Nested dictionary of offerings |
| `latitude`, `longitude` | Used to derive which Hawaiian island a business is on |

---

## Data Cleaning and Exploratory Data Analysis

**Cleaning steps** (data-generating process):

- **Zip code**: extracted directly from the free-text `address` field through regex (Hawaii zip codes all start with `967` or `968`), successful for 98% of restaurants.
- **Price level**: Google's `price` field is a string of repeated symbols (`$`, `$$`, etc.). In this scrape, the symbol is occasionally replaced by `₩` sign instead of `$` for the same tiers (not a real price difference, only related to how the page was scraped). Since only the *number* of symbols carries meaning, we converted `price` into an ordinal 1–4 `price_level` using string length, regardless of which symbol was used.
- **Amenities**: flattened the nested `MISC` dictionary into a single `num_amenities`.
- **Hours**: converted the weekly `hours` list into a `days_open` (number of days per week the restaurant is open).
- **Cuisine grouping**: mapped each restaurant's specific category tag into one of five broader cuisine groups (Hawaiian/Local, Asian, Cafe/Bakery, Fast/Casual, Other/Western), since the raw category tags are too sparse individually to analyze or model directly.
- **Island**: derived which Hawaiian island a restaurant is on from `latitude`/`longitude` using geographic bounding boxes (Hawaii's islands are well-separated).

The head of the cleaned DataFrame:

| name | zip_code | island | price_level | num_amenities | days_open | cuisine_group | avg_rating | num_of_reviews |
|:---|:---|:---|---:|---:|---:|:---|---:|---:|
| Hale Pops | 96762 | Oahu | NaN | 13 | 6 | Other/Western | 4.4 | 18 |
| Akasatana Ramen Kyoto | 96814 | Oahu | NaN | 9 | 7 | Asian | 5.0 | 1 |
| Grill City | 96797 | Oahu | NaN | 13 | 7 | Other/Western | 3.5 | 8 |
| Buona Sera | 96734 | Oahu | 2.0 | 15 | 6 | Other/Western | 3.7 | 24 |
| Tucker & Bevvy Breakfast | 96815 | Oahu | NaN | 16 | 7 | Fast/Casual | 4.2 | 57 |

### Univariate Analysis

<iframe
  src="assets/fig1-rating-distribution.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

Restaurant ratings are heavily left-skewed and cluster tightly between about 4.0 and 4.5. Only few restaurants are below 3.5. This is consistent with the positivity bias of online review platforms (most reviewers are visitors and liked what they got).

<iframe
  src="assets/fig2-price-tier-counts.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

About 41% of restaurants (1,757 of 4,301) have no price tier listed on Google. Among the ones that do, the large majority are budget-friendly (`$`/`$$`); upscale (`$$$`/`$$$$`) restaurants are rare (173 total, 4% of all restaurants).

### Bivariate Analysis

<iframe
  src="assets/fig3-rating-by-price-tier.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

Mean rating rises along with price tier: 4.08 (`$`), 4.25 (`$$`), 4.27 (`$$$`), 4.48 (`$$$$`). This pattern is exactly what being tested in the Hypothesis Testing section below.

<iframe
  src="assets/fig4-rating-vs-reviews.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

The correlation between number of reviews and average rating is essentially zero (r ≈ 0.06). A restaurant with a handful of reviews and one with thousands are almostly equally likely to be highly rated, implying popularity is not a proxy for quality in this dataset. This is also one of the reasons I exclude `num_of_reviews` from the prediction task below.

### Interesting Aggregates

**Top 10 zip codes by restaurant count:**

| zip_code | n_restaurants | mean_rating | mean_reviews |
|---:|---:|---:|---:|
| 96815 | 378 | 4.12 | 503.91 |
| 96814 | 333 | 4.21 | 196.99 |
| 96817 | 206 | 4.32 | 224.24 |
| 96761 | 188 | 4.29 | 453.46 |
| 96740 | 185 | 4.29 | 326.41 |
| 96720 | 174 | 4.30 | 249.43 |
| 96753 | 172 | 4.34 | 320.97 |
| 96816 | 161 | 4.31 | 169.29 |
| 96813 | 146 | 4.37 | 157.66 |
| 96707 | 141 | 4.16 | 314.76 |

Zip 96815 (Waikiki) has by far the most restaurants (378) and the highest average review count (504), but is also has the **lowest** average rating (4.12) of the top 10 zips. Zip 96813 (downtown Honolulu, 146 restaurants) has the **highest** average rating (4.37) among the top zips. This suggests that the most saturated, most-touristed zip code isn't necessarily where restaurants are rated best. Thus, high foot traffic and high rating don't automatically go together.

**Rating and price by cuisine group:**

| cuisine_group | n | mean_rating | mean_price_level |
|:---|---:|---:|---:|
| Other/Western | 1803 | 4.326 | 1.862 |
| Asian | 985 | 4.245 | 1.742 |
| Fast/Casual | 590 | 4.000 | 1.305 |
| Cafe/Bakery | 550 | 4.174 | 1.496 |
| Hawaiian/Local | 373 | 4.285 | 1.716 |

"Other/Western"-style restaurants have both the highest average rating (4.33) and the highest average price level (1.86) among the five cuisine groups, while "Fast/Casual" restaurants have the lowest average rating (4.00) and lowest average price level (1.31). This is consistent with the price/rating relationship above.

---

## Assessment of Missingness

We believe `price` is reasonably **MNAR**. Google's price-tier field is filled by users who volunteer a price impression when they visit a business, not information the business itself is required to submit. Very informal food establishments (such as food trucks etc.) may have prices that can't be categorized in Google's fixed `$`–`$$$$` scale, or may simply not attract the kind of reviewer who will fill it in. In other words, the *chance that price is missing* plausibly depend on the (unobserved) price/format of the business itself, not only on other recorded columns, which is exactly the definition of MNAR. Additional data that could change this to MAR would be an explicit "food truck / mobile vendor" flag, or a scraped menu, so that the missingness can be explained by an observed column instead of the unobserved price itself.

**Does missingness of `price` depend on `num_of_reviews`?**

<iframe
  src="assets/fig5-reviews-by-price-missing.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

<iframe
  src="assets/fig6-missingness-permutation.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

Permutation test: shuffle the price-missingness label 10,000 times and compute the difference in mean `num_of_reviews` between the "price listed" and "price missing" groups. The observed difference (restaurants with price listed average 388 reviews vs. 85 for restaurants missing price) is far more extreme than anything in the permuted distribution (**p < 0.0001**), so there is a strong evidence that missingness of `price` **depends on** `num_of_reviews`: less-reviewed restaurants are far less likely to have a price tier on Google. This missingness is **MAR** depending on `num_of_reviews`.

**Does missingness of `price` depend on `latitude`?**

Using the same permutation approach on `latitude` (a restaurant's north/south position within Hawaii), the observed difference in means (0.0099°) gives a two-sided **p-value of about 0.61** (not significant at all). No evidence is found that price-missingness depends on latitude. This implies that one column price-missingness clearly depends on (`num_of_reviews`) and one it does not appear to depend on (`latitude`).

---

## Hypothesis Testing

**Null hypothesis**: Among Hawaii restaurants, the average Google rating is the same for "budget" restaurants (`$`/`$$`) and "upscale" restaurants (`$$$`/`$$$$`); any observed difference is due to random chance.

**Alternative hypothesis**: Upscale restaurants (`$$$`/`$$$$`) have a *higher* average rating than budget restaurants (`$`/`$$`).

**Test statistic**: difference in group means, `mean(rating | upscale) − mean(rating | budget)`.

**Significance level**: α = 0.05.

I use a permutation test because I don't want to assume normally distributed ratings, and the size of two groups are very different (173 upscale and 2,371 budget restaurants) with different variances. Both assumptions are not required in a permutation test.

<iframe
  src="assets/fig7-hypothesis-permutation.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

The observed difference in means (0.1495 rating points) is more extreme than all of the 10,000 permuted differences (**p < 0.0001**), so I reject the null hypothesis at α = 0.05. This is consistent with upscale restaurants earning higher ratings than budget restaurants in Hawaii. This test alone does not prove that price *causes* higher ratings: both price and rating could be jointly driven by some unobserved factor that upscale restaurants tend to include.

---

## Framing a Prediction Problem

**Prediction problem**: predict a Hawaii restaurant's average Google rating (`avg_rating`). This is a **regression** problem, since the response is continuous (ranging from 1.0 to 5.0).

**Response variable**: `avg_rating`. This is the outcome a prospective restaurant owner wants to estimate before investing money and time, essntially "given the choices I'm about to make, what rating can I expect?"

**Evaluation metric**: RMSE. I use RMSE rather than R² or MAE because it's expressed in the same units as the response and penalizes large misses more heavily. This is suitable when a large error could mean the difference between a viable and non-viable business plan.

**Time-of-prediction consideration**: a prospective owner is deciding *before opening*, so I only use features they can plan/ control ahead of time, such as which zip code to locate in, what price tier to target, what cuisine to serve, what amenities to offer, and how many days a week to be open. I deliberately **exclude `num_of_reviews`**, since review volume can only be observed after a restaurant has been open and operating for a while.

---

## Baseline Model

The baseline model uses two features, both are known at planning time:

- **`zip_group`** (*nominal*): the restaurant's zip code, grouped into the top 20 most common Hawaii restaurant zip codes plus an "Other" bucket, one-hot encoded.
- **`price_level`** (*ordinal*, 1–4 for `$`–`$$$$`): missing values (~41% of restaurants) are median-imputed.

Model: `LinearRegression` (wrapped in a single `sklearn` `Pipeline` with a `ColumnTransformer` for preprocessing)

**Performance**: RMSE of **0.4346** on training data, **0.4569** on test data. Since restaurant ratings only have a standard deviation of about 0.45 to begin with, this baseline is only a little improvement than purely guessing the overall mean rating for every restaurant. I don't consider this "good" on its own because there's clear room to do better, which will be addressed next.

---

## Final Model

I add three new engineered features on top of `zip_group` and `price_level`:

- **`num_amenities`** (*quantitative*): total count of listed offerings from the `MISC` field. This shows the "richness" of a restaurant's planned offering, which an owner decides before opening.
- **`days_open`** (*quantitative*): number of days per week the restaurant is open, derived from `hours` (operating decision).
- **`cuisine_group`** (*nominal*): the restaurant's specific category tag sorted into 5 cuisine groups, one-hot encoded. Individual raw categories are too sparse to one-hot encode directly, but cuisine style may correlates with typical service style and ambiance.

**Modeling algorithm**: moved from `LinearRegression` to a `RandomForestRegressor`, because the exploratory analysis showed the price/rating relationship isn't perfectly linear (the jump from `$` to `$$` is bigger than from `$$` to `$$$`). A tree-based model can capture this non-linearities and feature interactions without being directly specified.

**Hyperparameter tuning**: tuned `max_depth` (over `[3, 5, 8, 12, None]`) and `n_estimators` (over `[100, 200]`) through 5-fold `GridSearchCV`, using negative RMSE as the scoring metric. The best combination was **`max_depth=5, n_estimators=200`** (best cross-validated RMSE: 0.4099).

**Performance**: the final model achieves an RMSE of **0.3929** on training data and **0.4327** on test data. There is about a 5% reduction in typical prediction error (an improvement of **0.0241** over the baseline's test RMSE of 0.4569). The improvement comes mainly from the cuisine-group and amenity-count features. The improvement is genuine and justified, and is achieved by adding features that are plausibly connected to how customers actually experience a restaurant, not by blindly transforming existing ones.

---

## Fairness Analysis

**Group X**: restaurants on **Oahu** (the most populous, most-visited island; the largest group in the training data).

**Group Y**: restaurants on the **Neighbor Islands** (Maui, Big Island, Kauai, Molokai, and Lanai combined).

**Evaluation metric**: RMSE of the final model's predicted `avg_rating`, computed separately on each group within the test set.

**Null hypothesis**: the model is fair. Its RMSE on Neighbor Island restaurants and on Oahu restaurants is roughly the same, and any difference seen is due to random chance.

**Alternative hypothesis**: the model is unfair. Its RMSE is *higher* for Neighbor Island restaurants than for Oahu restaurants (plausible, since the majority of the training data are Oahu restaurants, so the model may have learned Oahu's patterns better).

**Test statistic**: `RMSE(Neighbor Islands) − RMSE(Oahu)`.

**Significance level**: α = 0.05.

<iframe
  src="assets/fig8-fairness-permutation.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

RMSE was 0.4361 for Oahu restaurants (n=551) and 0.4267 for Neighbor Island restaurants (n=310). They are essentially the same, and in fact slightly *better* for the Neighbor Islands. The permutation test gives **p ≈ 0.57**, so I fail to reject the null hypothesis at α = 0.05. Despite Oahu having roughly triple the training examples, no evidence is found that the model performs worse for Neighbor Island restaurants, at least by this metric.
