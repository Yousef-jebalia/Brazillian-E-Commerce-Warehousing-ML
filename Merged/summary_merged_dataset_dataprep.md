# Merged Dataset: Data Preparation (Orders + Reviews)

This notebook merges the prepared orders table with the cleaned review table and prepares the combined dataset for analysis.

## What we did

**1. Merge.** Loaded `prepared_orders.csv` (98,972 rows, 20 columns) and `clean_order_reviews.csv` (98,673 rows, 9 columns). Left-joined reviews onto orders on `order_id` with a one-to-one check. The result has 98,972 rows and 28 columns.

**2. Handle the difference between the two datasets.**
- 765 orders have no review (kept, with empty review fields).
- 466 reviews have no matching order. These were removed, leaving 98,207 reviews.

**3. Key and duplicate checks.** Cast `order_id` to a clean string and stripped spaces. Found no duplicate `order_id`, no duplicate full rows, and no review scores outside 1 to 5.

**4. Null check.** The merged table had 6,167 missing values before any treatment, concentrated in the review fields and a few duration columns (8 to 16 rows each).

**5. Date conversion and study period.** Converted all seven date columns to datetime. Kept only orders purchased between 1 Jan 2017 and 31 Aug 2018, removing 347 orders.

**6. Flags.**
- `has_review`: 1 if the order has a review score.
- `has_comment`: missing values filled with 0, then cast to integer.

**7. Feature engineering.**
- `estimated_days`: promised delivery time (estimated date minus purchase date, in days).
- `delivery_delay_days`: actual minus estimated delivery date, in days, only for delivered orders.
- `is_delivered`: 1 when the order status is 5 (delivered).

**8. Null treatment for the final export.** Filled missing values in three groups:
- Date columns: missing values replaced with the order status text.
- Duration columns: missing values replaced with -1.
- Review fields: missing values replaced with "N/A".

After this step, 759 nulls remain.

**9. Correlation and PCA.** Built a correlation heatmap on delivered orders for the duration, estimate, review and comment variables. Ran PCA on five duration and estimate columns. The first component explains 45.3% of the variance, and the first three explain 84.8%. The last component explains almost nothing, which shows these columns are close to linearly dependent (the total duration is roughly the sum of its parts).

**10. Plots.** Scree plot, estimated delivery days and delay histograms, review score distribution, orders per month, and orders by status code.

**11. Export.** Saved `orders_master.csv` and triggered a Colab download.

## Output

`orders_master.csv`: one row per order, combining order details, delivery durations, flags, and review information.

## Notes to check

- Writing the status text into date columns and "N/A" into numeric columns makes those columns mixed-type. This is fine for export, but any model code must handle it.
- `delivery_delay_days` is in the list of review columns that get "N/A" filled, although it is numeric. Check this before modelling, as it may turn a numeric column into text.
- The review notebook saves its output as a Parquet file, but this notebook reads `clean_order_reviews.csv`. Make sure the file name and format match.
