# Olist Orders: Data Preparation

This notebook cleans and prepares the Olist orders table. It checks for missing values and duplicates, converts dates, creates time-based features, flags impossible values, and saves the prepared file used by the next step.

## What we did

**1. Load and inspect.** Read `olist_orders_dataset.csv` (99,441 rows, 8 columns) and reviewed its summary and types.

**2. Data cleaning.**
- *Missing values.* Missing values appear in the date columns only: 160 approval dates, 1,783 carrier dates, and 2,965 delivery dates. These mark stages an order never reached, so they were kept as missing for now.
- *Duplicates.* None found by row or by `order_id`.
- *Order status.* Stripped and lowercased the text. Counts: delivered 96,478, shipped 1,107, canceled 625, unavailable 609, invoiced 314, processing 301, created 5, approved 2.

**3. Data transformation.**
- *Dates.* Converted the five date columns to datetime, with invalid values set to missing.
- *Status encoding.* Mapped the statuses to numbers: created 0, approved 1, invoiced 2, processing 3, shipped 4, delivered 5, unavailable 6, canceled 7.
- *Study period.* Checked the purchase range (4 Sep 2016 to 17 Oct 2018). The filter removed no rows, so the full period was kept.
- *Impossible date sequences.* Flagged rather than removed: 1,359 rows where the carrier date is before the approval date, 23 where delivery is before the carrier date, and none where approval is before purchase. 1,382 rows have at least one problem.

**4. Feature engineering.**
- Time features in days: `approval_days`, `carrier_days`, `delivery_duration_days`, and `delivery_total_duration_days`.
- Undelivered orders were set to -1 as a placeholder, affecting 2,963 orders. Delivered orders in the analysis set: 96,432.
- Plotted histograms and boxplots of the four features to inspect them.

**5. Outliers.** Applied the IQR rule on a log scale to each time feature, skipping the -1 placeholders. Outliers were flagged in new columns, not removed. Rows with impossible negative values (below -1) were removed: 469 rows. 98,972 rows remain.

**6. Dimensionality reduction.** PCA was not applied because there are only four numeric time features, and reducing them would not help. Instead, computed a Spearman correlation matrix on 95,082 rows where all four features are real values. Spearman is used because the durations are skewed.

**7. Save.** Exported `prepared_orders.csv` with 98,972 rows.

## Output

`prepared_orders.csv`: one row per order, with status codes, time features, impossible-sequence flags, and outlier flags.

## Notes to check

- Missing delivery and approval dates are filled with the order status text. This is applied later in the file, so the date columns end up with mixed types.
- The outlier flags are kept but not used to remove rows, so the outliers stay in the prepared data.
- The KMeans and silhouette imports are not used in this notebook and can be removed.
