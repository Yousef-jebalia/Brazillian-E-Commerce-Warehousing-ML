# Olist: Building the ML Datasets

This notebook builds the two datasets used by the ML objectives:

1. **Delivery duration (regression)**, target `delivery_total_duration_days`
2. **Late delivery (classification)**, target `is_late` (1 = delivered after the estimated date)

## What we did

**1. Load.** Read the seven cleaned tables: orders, customers, payments, order items, products, sellers and geolocation.

**2. Join.** Started from delivered orders with a known duration and delay (95,736 orders) and attached the other tables one at a time: customer state and zip, item aggregates, main product size, payment aggregates, seller state and zip, and geolocation. Each zip prefix was reduced to one average latitude and longitude. The joined table has 95,736 rows.

**3. Cleaning check.** The data was already clean, so this step only checked for duplicate order IDs (none), orders without a seller (1, removed), and non-positive durations (none). Some coordinates and product sizes are missing and were handled later.

**4. Encoding.** Mapped customer and seller states to numbers using one shared alphabetical mapping, so the same state gets the same number on both sides.

**5. Feature engineering.**
- Purchase month, weekday and hour.
- Haversine distance between customer and seller (km), and a same-state flag.
- Approximate order weight and volume (main product size times quantity).
- Payment shares by card, boleto and voucher.
- Seller history for the classification set only: the seller's past late rate and number of past orders. Only orders delivered before the current purchase are counted, so no future information leaks in.
- Rows with missing core values were dropped (491 rows).

**6. Sanity checks.** Confirmed that `order_id` is unique in both datasets, that the two datasets have the same rows in the same order, and that the regression data has no missing values. The late rate is 8.16%.

**7. Dimension reduction.**
- Removed zip codes and raw coordinates (the distance feature already summarises them).
- Built a correlation heatmap to find redundant features, such as total paid versus price plus freight.
- Ran PCA on standardised numeric features (excluding state codes and the target). 13 components are needed to keep 90% of the variance out of 20 features. The first two components keep only 29.4%. The top 13 components were added as `PC1` to `PC13`.

**8. Visualisation.** Plots of delivery duration distribution, average delivery time and late rate by state and by month, the distance effect, on-time versus late counts, and 2D and 3D PCA scatter plots coloured by delivery days, distance and lateness. All figures are saved in `figures/`.

**9. Save.** Exported `dataset_regression.csv` and `dataset_classification.csv`.

## Outputs

| File | Use |
|---|---|
| `dataset_regression.csv` | ML 1: delivery duration regression |
| `dataset_classification.csv` | ML 2: late delivery classification |
| `figures/*.png` | Plots for the report |

## Notes

- Weight and volume only use the main product in each order, so they can underestimate large orders.
- `main_category` was left out because it is text. Encoding it needs a decision first.
- Seller history is only in the classification dataset. The regression dataset does not include it.
