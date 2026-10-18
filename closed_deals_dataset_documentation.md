# Olist Closed Deals Dataset: Cleaning and Preparation

**Source file:** `olist_closed_deals_dataset.csv`
**Script:** `clean_closed_deals.py`
**Output:** `clean_closed_deals_0.csv`

## 1. What this dataset is

One row per **closed deal** in Olist's sales funnel (842 deals, won between December 2017 and November 2018). A closed deal means a lead (a prospective seller) was converted into an Olist seller. The table says who handled the deal, when it was won, and what kind of business the seller has. It links to the sellers table through `seller_id` and to the marketing qualified leads file through `mql_id`.

## 2. Column description

### Columns kept in the final file

| Column | Type | Description |
|---|---|---|
| `mql_id` | text (32 chars) | Marketing qualified lead ID, unique per deal. Key to the marketing leads file |
| `seller_id` | text (32 chars) | Unique ID of the seller created by the deal. Key to the sellers and orders tables |
| `sdr_id` | text (32 chars) | Sales Development Representative: the person who contacted and qualified the lead. One person handles many deals |
| `sr_id` | text (32 chars) | Sales Representative: the person who closed the deal. Some IDs appear in both `sdr_id` and `sr_id` (same people in both roles) |
| `won_date` | datetime | Date and time the deal was won |
| `business_segment` | text | Product segment of the seller's business (e.g. `home_decor`, `health_beauty`, `pet`). 33 levels plus `unknown` |
| `lead_type` | text | Type or size of the lead's business (`online_medium`, `online_big`, `industry`, `offline`, `online_small`, `online_beginner`, `online_top`, `other`, `unknown`) |
| `business_type` | text | `reseller`, `manufacturer`, `other` or `unknown` |
| `behaviour_cat` | 0/1 | Lead's behaviour profile includes "cat" (likely calm, patient style) |
| `behaviour_eagle` | 0/1 | Profile includes "eagle" (likely sociable, big-picture style) |
| `behaviour_wolf` | 0/1 | Profile includes "wolf" (likely analytical, detail-oriented style) |
| `behaviour_shark` | 0/1 | Profile includes "shark" (likely direct, results-driven style) |
| `behaviour_profile_missing` | 0/1 | 1 if no behaviour profile was recorded (field not collected before February 2018) |
| `won_year_month` | text | Month of the deal (e.g. `2018-04`) |
| `won_quarter` | text | Quarter of the deal (e.g. `2018Q2`) |
| `won_week` | integer | ISO week number of the deal (does not include the year) |

The four behaviour labels are not documented in the dataset. The interpretation above follows the common animal version of the DISC personality model and should be confirmed with the data source.

### Columns removed from the raw file

| Column | Description | Why removed |
|---|---|---|
| `declared_monthly_revenue` | Monthly revenue declared by the lead | 94.7% are `0` (meaning "not asked") |
| `average_stock` | Declared average stock range (`1-5`, `5-20`, ..., `200+`) | 92.6% missing or `unknown` |
| `has_company` | Whether the lead has a registered company | 92.5% missing |
| `has_gtin` | Whether the products have a GTIN (barcode) | 92.4% missing |
| `declared_product_catalog_size` | Number of products declared in the catalog | 91.8% missing |

## 3. Problems found in the raw data

- **Structural missing data:** the five columns above were only collected from about July 2018. A missing value means "not asked", not "unknown", so filling it with a median would invent data.
- **Behaviour profile:** 177 rows empty (collected only from February 2018), and 16 rows hold two labels (e.g. `cat, wolf`).
- **Categories:** 1 missing `business_segment`, 6 missing `lead_type`, 10 missing `business_type`, a misspelling (`jewerly`), and a separate `other` value.
- **`average_stock`:** contained a fake category `unknown`.
- **No duplicate rows and no unparseable dates.**

## 4. Cleaning steps (in order)

1. **Load as text** so IDs are never altered, then parse `won_date` to a datetime.
2. **Measure "not asked" values per column.** An empty cell, a numeric `0` or the word `unknown` all count as "not asked".
3. **Drop columns above 70% not asked.** The rule is computed on every column (not a hardcoded list), and ID columns are protected. It removed the five columns listed above.
4. **IDs.** Only checked (length 32, no spaces, uniqueness). `mql_id` and `seller_id` are unique, and nothing was changed or removed.
5. **Text normalization.** Categorical columns are lowercased and trimmed, spaces become underscores, and `jewerly` becomes `jewelry`.
6. **Missing categories** are filled with `unknown`, kept separate from `other`.
7. **Behaviour profile.** Split into four 0/1 columns (multi-hot encoding) so combined values keep both labels, plus a `behaviour_profile_missing` flag (missing-value indicator). The original text column is dropped.
8. **Date features.** `won_year_month`, `won_quarter` and `won_week` are added.
9. **Integrity checks.** No missing IDs, dates or categories, and `mql_id` and `seller_id` are unique.

## 5. Visualizations

- **Evidence for the drop decision:** a bar chart of the missing percentage per column with the 70% rule line, and a line chart of the percentage of rows with a real value by month. The second chart shows the detail fields only appear from mid-2018.
- **Category noise:** frequency bars for `business_segment`, `lead_type` and `business_type`, with rare levels visible at the bottom.
- **Time and concentration:** deals per month (peak in April 2018 with 207 deals, thin tail at the end), and the top 15 SDRs and SRs by deals closed.

## 6. Final output

`clean_closed_deals_0.csv`: **842 rows, 16 columns**, saved with `index=False`. No audit file or image files are produced.

Behaviour columns in the final file: cat 419, eagle 129, wolf 107, shark 26, missing 177. Of the 842 rows, 649 have one label, 16 have two labels, and 177 have none.

## 7. Notes and limitations

- Only about 380 of the 842 closed-deal sellers appear in the sellers table, so these features will be empty for most sellers after the merge. Use a "has closed-deal info" flag when joining.
- `won_date` is saved as text in the CSV. Reload it with `parse_dates=['won_date']`.
- `won_week` has no year, so the same week of different years looks identical. `won_year_month` carries the year.
- Rare category levels (e.g. `religious`, `perfume`, `online_top`) may need grouping when encoding features after the merge.
- Mapping `business_segment` to the product category names used in the other tables is left for after the merge.
- Scaling and further encoding should be done after the train/test split is defined.
