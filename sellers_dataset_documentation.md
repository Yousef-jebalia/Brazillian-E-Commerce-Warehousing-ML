# Olist Sellers Dataset: Cleaning and Preparation

**Source file:** `olist_sellers_dataset.csv`
**Script:** `clean_sellers.py`
**Output:** `clean_seller_0.csv`

## 1. What this dataset is

One row per seller registered on the Olist marketplace (3,095 sellers). It gives the location of each seller. Location matters for the team's goals because seller position drives delivery distance, delivery time and late deliveries.

## 2. Column description

| Column | Type | Description | Use |
|---|---|---|---|
| `seller_id` | text (32 chars) | Unique identifier of the seller | Key to join with orders, order items and closed deals |
| `seller_zip_code_prefix` | text (5 digits) | First 5 digits of the seller's postal code (CEP). Stored as text to keep leading zeros (e.g. `01046`) | Source of truth for the state; key to join with the geolocation table |
| `seller_city` | text (lowercase, no accents) | City where the seller is located | Location analysis, top cities, regional features |
| `seller_state` | text (2 uppercase letters) | Brazilian state code (SP, RJ, MG...) | Regional analysis, distance and delivery features |

## 3. Problems found in the raw data

- **Wrong states:** 35 rows (1.1%) had a state that contradicts the zip prefix. Almost all were labelled `SP` by default.
- **Messy city names:** 611 distinct city strings for what should be far fewer cities. Causes:
  - Uppercase, accents and extra spaces (`são paulo`, `sao  paulo`)
  - State or country written inside the city (`sao paulo - sp`, `auriflama/sp`)
  - Abbreviations (`sbc`)
  - Typos (`sao pauo`, `garulhos`, `floranopolis`)
  - Values that are not cities (an e-mail address, a number, `bahia`, `centro`, `california`, `sp / sp`)
- **No missing values, no duplicate rows, no duplicate `seller_id`, and all zip prefixes are 5 digits.**

## 4. Cleaning steps (in order)

1. **Load as text.** Everything is read as text so zip prefixes keep their leading zeros. Raw copies of the zip, city and state are kept in memory for the before/after comparison.
2. **Zip prefix.** Pad to 5 characters with `zfill` and check for invalid prefixes (none found).
3. **State from zip.** The zip prefix is the source of truth. A table of Correios zip ranges per state is applied, and the state is overwritten where it disagrees. State is stored in uppercase. **35 states corrected.**
4. **City normalization (rules).**
   - Lowercase, remove accents, collapse repeated spaces
   - Keep only the part before `/`, `\`, `,`, `(` or ` - `
   - Remove a trailing state code that matches the seller's own state (`sao paulo sp`)
   - Replace hyphens with spaces (`embu-guacu` becomes `embu guacu`)
   - Expand known abbreviations (`sbc` becomes `sao bernardo do campo`)
   - Two manual fixes, keyed by the raw value (not by row index): the numeric value `04482255` becomes `rio de janeiro`, and the e-mail value becomes `maringa`
   - Values that are not cities are marked as missing for the next steps (11 rows)
5. **City typos (fuzzy matching).** A city that appears once in a state and is very similar (similarity of at least 0.88) to a city appearing twice or more in the same state is corrected. **20 corrections**, all reviewed.
6. **Filling missing cities.**
   - First fallback: the most common city of the same zip prefix (**4 rows**)
   - Second fallback: the most common city of the same state (**7 rows**, an approximation used for rows whose zip prefix is unique)
7. **Validation flag (not applied).** Rows whose city disagrees with the usual city of their zip prefix are counted for review (**9 rows**) but left unchanged by decision of the team.
8. **Integrity checks.** `seller_id` is unique and 32 characters, zip prefixes are 5 characters, all states are valid, and no city is missing.

**Result:** distinct city strings went from **611** (raw) to **573** (after rules) to **553** (final).

## 5. Visualizations

**Noise and outlier diagnostics (6 panels):**
- Distribution of raw city name length (long names are suspicious)
- Noise types in the raw city column
- Distinct cities at each cleaning stage
- Boxplot of zip prefix per original state (dots far from the box are wrong-state sellers)
- Which original states were corrected
- Number of cities by number of sellers (long tail of rare cities)

**Top sellers:**
- Top 10 states with share of sellers: SP has 1,816 sellers (58.7%), then PR (360), MG (250), SC (197), RJ (176), RS (132)
- Top 15 cities: São Paulo (703), Curitiba (127), Rio de Janeiro (100), Belo Horizonte (69), Ribeirão Preto (56)

## 6. Final output

`clean_seller_0.csv`: **3,095 rows, 4 columns**, with the same column names as the original so it joins cleanly with the other tables. No extra audit file or image files are produced.

## 7. Notes and limitations

- The zip-range table should be checked once against the official Correios table (some states have split ranges, such as DF/GO and AM/RR).
- The 7 cities filled from the state fallback are approximations.
- The 9 city/zip disagreements are not corrected.
- Rare cities (about 280 cities have a single seller) may need grouping before encoding for machine learning.
