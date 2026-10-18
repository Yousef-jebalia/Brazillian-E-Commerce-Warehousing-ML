# Olist Geolocation — Data Cleaning Process

## Objective

Prepare `olist_geolocation_dataset.csv` for geographic analysis by standardizing values, filtering inconsistent records, and removing duplicates.

## 1. Initial inspection and visualization

Load the dataset, preserving ZIP-code prefixes as text to retain leading zeros. Inspect its dimensions and visualize:

- Latitude and longitude distributions.
- The ten states with the most geolocation records, not customers or orders.
- A geographic scatter plot using up to 30,000 sampled records.

## 2. Cleaning rules

| Rule | Action |
| --- | --- |
| **1. ZIP-code prefixes** | Trim spaces, add leading zeros where needed, and retain only five-digit prefixes. |
| **2. City names** | Decode encoded text, repair known text errors, remove accents, convert to lowercase, normalize selected apostrophes, and remove unnecessary punctuation and spaces. |
| **3. Coordinate bounds** | Retain latitude between −33.75° and 5.27° and longitude between −74.0° and −28.8°. |
| **4. State consistency** | Assign each ZIP prefix its most frequent state and remove records with a different state. |
| **5. Geographic outliers** | Calculate distance from each prefix's median coordinate center using the haversine formula. Remove points beyond 50 km when that prefix has at least one point within 50 km. |
| **6. Duplicates** | Remove rows identical across all five fields and reset the index. |

Apply these rules in order and record the number of rows removed at each step. Compare distinct city-name counts before and after normalization.

### Important limits

- The coordinate bounds form a rectangle, not Brazil's exact border.
- The most frequent state is an internal consistency reference, not an official verification.
- Outlier detection is **rule-based**, with a chosen **50 km threshold**. If every point in a ZIP group exceeds that distance, the group is retained. Single-point groups also remain.
- Missing ZIP prefixes, states, and city names need explicit handling; city normalization alone does not resolve missing values.
- Duplicates influence state frequencies and median coordinates because they are removed last.

## 3. Validation

Confirm that the cleaned dataset has:

- No exact duplicates or incorrectly formatted ZIP prefixes.
- No coordinates outside the selected bounds.
- No ZIP prefix linked to multiple states.
- Lowercase city names with no non-ASCII characters or unnecessary spaces.

These checks confirm the selected rules, not official geographic accuracy.

## 4. Export and final review

Save the result as `olist_geolocation_clean.csv` without the row index. Inspect the first ten rows and repeat the initial visualizations to compare distributions and isolated points before and after cleaning.

PCA and a correlation matrix are not required for these cleaning rules. This document describes the process; no execution results are reported.
