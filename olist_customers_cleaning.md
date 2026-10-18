# Olist Customers Dataset: Cleaning

**Notebook:** `olist_customers_cleaning.ipynb`
**Input:** `olist_customers_dataset.csv` (99,441 rows, 5 columns)
**Output:** `olist_customers_clean.csv` (99,441 rows, 5 columns)

## Columns

| Column | Description |
|---|---|
| `customer_id` | ID created for each order (one per order, not per person) |
| `customer_unique_id` | The real person. One person can have several `customer_id` |
| `customer_zip_code_prefix` | First 5 digits of the customer's zip code |
| `customer_city` | Customer city |
| `customer_state` | Customer state (2-letter code) |

## Rules and results

| # | Rule | Result |
|---|---|---|
| 1 | No missing values | ✅ 0 |
| 2 | No duplicate rows | ✅ 0 |
| 3 | `customer_id` is unique and is a 32-character hex code | ✅ 0 invalid, 0 duplicates |
| 4 | `customer_unique_id` is a 32-character hex code and **may repeat** | ✅ 96,096 people, of whom 2,997 have 2 or more orders |
| 5 | Zip is a 5-digit string with its leading zeros | ✅ 0 invalid |
| 6 | State is one of the 27 Brazilian codes | ✅ 0 invalid |
| 7 | Each zip belongs to one state | ✅ 0 conflicts |
| 8 | City is lowercase, with no accents and no extra spaces | ✅ 0 problems |

The file was already clean, so **no rows were removed**.

## Cleaning steps

1. **Load with the zip as text:** `dtype={'customer_zip_code_prefix': str}`, so `01151` doesn't become `1151`.
2. **Zip:** `strip()`, then `zfill(5)`, then keep only 5-digit values.
3. **Text format:** city gets `strip()`, `lower()` and single spaces; state gets `strip()` and `upper()`.
4. **State:** keep only the 27 valid codes.
5. **IDs:** keep only 32-character hex codes, and drop duplicates by `customer_id`.
6. **Final check** of every rule, then save to `olist_customers_clean.csv`.

## Important notes

- **Don't delete repeated `customer_unique_id` values.** They're the same person placing several orders.
- Use **`customer_unique_id`** for segmentation, recommendation and retention. With `customer_id`, every customer looks like a one-time buyer.
- When reading `olist_customers_clean.csv` again, load the zip as text: `dtype={'customer_zip_code_prefix': str}`.
- **279 customers have no coordinates**, because their zip isn't in `olist_geolocation_clean.csv`. After the merge, fill their lat/lng with the state average.

## Visualizations

| # | Chart | Key finding |
|---|---|---|
| 1 | Customers per state | SP 42.0%, RJ 12.9%, MG 11.7% |
| 2 | Top 15 cities | São Paulo and Rio de Janeiro lead |
| 3 | One-time vs repeat customers | 93,099 one-time, 2,997 repeat (**about 3.1%** come back) |
| 4 | Customer map (uses the clean geolocation file) | Customers are concentrated in the southeast |
| 5 | Repeat rate per state | AC 5.19%, RO 4.17%, RJ 3.40% (small states vary by chance) |
