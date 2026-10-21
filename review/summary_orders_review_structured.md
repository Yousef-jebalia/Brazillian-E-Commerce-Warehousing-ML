# Data Preparation: Olist Order Reviews

This notebook cleans and prepares the Olist order reviews table, following the order of the validation session: data cleaning, data transformation, feature engineering, and dimensionality reduction.

## What we did

**1. Load and inspect.** Read `olist_order_reviews_dataset.csv` (99,224 rows, 7 columns) and checked its types and summary statistics.

**2. Data cleaning.**
- *Missing values and duplicates.* Comment title is missing in 87,656 rows and comment message in 58,247. Both are optional, so this is expected. Found 814 duplicate `review_id` values and some orders with more than one review.
- *Score validation.* All review scores are between 1 and 5, so no rows were removed.
- *Data types.* Converted the two timestamp columns from text to datetime.
- *Date sanity check.* No review has an answer timestamp before its creation date.

**3. Data transformation.**
- *One review per order.* Sorted by creation date and kept the latest review for each order. This removed 551 rows, leaving 98,673 reviews with a unique `order_id`.

**4. Feature engineering.**
- `has_comment`: True if either the title or the message has a value. The two text columns and their Portuguese content were then dropped, because the current models do not use them. About 42,500 reviews have a comment.
- `days_to_answer`: the time between the review and the seller's answer, in fractional days (total seconds divided by 86,400, so gaps under one day are not truncated to zero). Median is about 1.7 days, with a long tail up to about 519 days.
- `is_low_review`: True for scores of 2 or below. About 14.7% of reviews are low.

**5. Dimensionality reduction.** Checked the correlations between the numeric review features. The strongest relationship is between `review_score` and `is_low_review` (-0.88), which is expected because one is derived from the other. The features are otherwise only weakly related.

**6. Output.** Saved `clean_order_reviews.parquet` with 98,673 rows and 9 columns.

## Output

`clean_order_reviews.parquet`: one review per order, with the score, response time, comment flag, and low-review flag.

## Notes to check

- Keeping only the latest review per order is a design choice. Orders with several reviews lose the earlier scores.
- The output is Parquet, but the merged dataset notebook reads a CSV file with a similar name. Align the format before running the pipeline.
