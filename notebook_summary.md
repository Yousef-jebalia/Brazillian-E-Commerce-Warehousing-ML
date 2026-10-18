# Olist products: what was applied in `data_cleaning_simple.ipynb`

**Files used:** `olist_products_dataset.csv`, `product_category_name_translation.csv`

## 1. Audit (before cleaning)
- Checked missing values, duplicates, ID format and text issues.
- 610 products had no category, name length, description length or photo count.
- 2 products had no weight or dimensions.
- 4 products had a weight of 0.
- 2 categories were missing from the translation file (`pc_gamer`, `portateis_cozinha_...`).

## 2. Cleaning
- Lowercased and stripped the category text.
- Fixed the `lenght` typos in the column names.
- Added the 2 missing translations (translation table: 73 rows).
- Missing category replaced by `unknown`, plus a `category_was_missing` flag.
- Dropped the 2 rows with no dimensions.
- Weight of 0 treated as missing.
- Remaining gaps filled with the category median, then the global median.
- Added `product_volume_cm3` (length x height x width).
- Result: **32,949 rows, 0 missing values**, exported to CSV.

## 3. Outlier detection (IQR rule)
- Fences: Q1 - 1.5 x IQR and Q3 + 1.5 x IQR.
- Applied on the raw columns and again after a log transform.
- Outliers are **flagged, not removed**.
- Rows with at least one outlier: **8,350 -> 830** (about 25% -> 2.5%) after the log.
- Boxplots of the standardised columns to visualise them.

## 4. Correlation matrix
- Size columns (weight, length, width, height) are correlated with each other.
- Listing columns (name length, description length, photos) are only weakly correlated.
- Two separate blocks of variables.

## 5. PCA
- Log transform, then standardisation (volume excluded because it is derived from the dimensions).
- **PC1 (36%)**: product size and weight.
- **PC2 (17.7%)**: listing richness (name, description, photos).
- 80% of the variance needs 4 components; the elbow and Kaiser rule suggest 2.
- Scree plot, loadings heatmap and PC1/PC2 scatter plot.

## Known limitations
- The 609 rows with a missing category get identical imputed name, description and photo values, which slightly distorts the PCA and correlations.
- IQR is a weak rule for `product_photos_qty` (small count, median of 1).
- IQR checks one column at a time, so it can't catch odd combinations (heavy product in a tiny box).
