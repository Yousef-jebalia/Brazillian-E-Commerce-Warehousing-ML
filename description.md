# ML3 – Customer Segmentation (Olist): Summary of the Notebook

**Goal:** segment customers of the Olist Brazilian e-commerce dataset with **K-Means** and **DBSCAN**, then compare the two.

**Structure:** 4 parts, Steps 0 to 6.

| Part | Steps | Content |
|---|---|---|
| 1 | 0 – 3 | Setup, load and merge tables, clean, aggregate to one row per customer |
| 2 | 4 | Feature transformation, feature sets, standardization, correlation check |
| 3 | 5 | K-Means on original features vs. K-Means on PCA components |
| 4 | 6 | DBSCAN and its comparison with K-Means |

**Input actually used in this run:** `ml3_features_A_delivered.csv` (`START_FROM = "features"`). Steps 1 to 4.1 were therefore **skipped** (their cells print "Skipped"), and the notebook started at Step 4.2. The code of the skipped steps is still described below.

---

## Part 1 – Setup, merge, cleaning, customer table (Steps 0–3)

### Step 0 – Imports and settings
- Imports all libraries (pandas, numpy, matplotlib, seaborn, scikit-learn, scipy).
- Fixes the key settings:
  - `OBS_DATE = 2018-09-01` (fixed observation date, avoids leakage)
  - `INCLUDE_UNDELIVERED = False` → **version A: delivered orders only**
  - `RANDOM_STATE = 42`
  - names of the input and output files.

### Step 0b – Detect the starting step
- Looks at the CSV files in `DATA_DIR` and recognizes what each one is **from its columns** (raw tables, order-level, customer-level or features).
- Starts from the furthest step possible and skips everything before it.
- Result in this run: `Starting from: features (file: ml3_features_A_delivered.csv)`.

### Step 1 – Load and merge tables (one row = one order)
- Loads 5 tables: orders, customers, payments, items, products.
- Checks that the join keys are unique so merges cannot duplicate orders.
- Keeps delivered orders only (status code 5).
- Merges step by step (orders → items inner → customers inner → payments left → products left) with a row count after each merge, and checks that there is still one row per order.
- Plots orders per month with the observation date marked.

### Step 2 – Clean the merged data
- Removes orders after the observation date.
- Fills missing values: category → `"unknown"`, payment total → item total, installments → 1, payment values → 0.
- Delivery delay: median fill, then clip to the 1st–99th percentile. Freight ratio: capped at the 99th percentile.
- Drops unused columns and compares missing values before and after.
- Plots: delivery delay before/after cleaning, top 15 categories.

### Step 3 – Aggregate to one row per customer (`customer_unique_id`)
- Builds the customer table: total spent, average order value, average item price, freight ratio, installments, review score, delivery delay, product weight and volume, plus profiling columns (number of orders, items, categories).
- Computes **recency** (days between last purchase and observation date) and the **payment shares** (boleto, credit card, debit card, voucher).
- Adds profiling columns: state (of the latest order) and main category (most frequent).
- Fills the remaining missing values with the median and runs final checks (unique IDs, no NaN).
- Plots: orders per customer (shows why frequency is not used for clustering), raw feature distributions with skew, top categories and states, spending vs. item price.
- Saves `ml3_customers_A_delivered.csv` (and offers downloads in Colab).

---

## Part 2 – Transformations and feature sets (Step 4)

### 4.0 – Load the customer table
- Uses the customer table as a readable `base`. Not available in this run (we started from the features file), so `profile = None`.

### 4.1 – Transform features
- K-Means is distance-based, so heavily skewed features are reduced with `log1p`.
- Logged: `total_spent` (skew 9.2), `freight_ratio` (2.1), `avg_installments` (1.6), `avg_product_volume_cm3` (4.1).
- Not transformed: recency (skew 0.4), shares, review score, delivery delay.
- Also creates `log_recency_days`, used only for the baseline set D.
- Skew before/after table and before/after histograms.

### 4.2 – Define four feature sets
| Set | Features | Question |
|---|---|---|
| **A_core** | 7 core features: recency, log total spent, log freight ratio, log installments, boleto share, review score, delivery delay | Main candidate |
| **B_core_plus_volume** | A + product volume (8) | Does product size help? |
| **C_core_plus_rare_payments** | A + voucher share + debit share (9) | Do rare payment types help? |
| **D_baseline** | 6 features of the earlier baseline test | Reference point |

### 4.3 – Standardize
- `StandardScaler` per set (mean 0, std 1) so large-scale features (recency in days) do not dominate.
- Checks mean ≈ 0 and std ≈ 1.
- Shows the largest |z-score| per feature. Rare features are extreme (`debit_card_share` 8.0, `voucher_share` 6.1), which can distort distances. This is a warning for set C.

### 4.4 – Correlation check
- Heatmap per set, with |r| > 0.8 as the redundancy warning level.
- Sets A, B and C are fine (strongest pair: log total spent vs. log freight ratio, r = −0.61).
- **D_baseline gets a warning:** credit card share and boleto share have r = −0.89.

### 4.5 – Final checks and saving
- All four matrices are clean (no NaN or infinite values), with the same customers in every set.
- The transformed features are saved only when built in this run (skipped here).

---

## Part 3 – K-Means: original features vs. PCA (Step 5)

### 5.0 – Checks, PCA matrices, settings
- Builds 2D PCA coordinates (for maps) and PCA components keeping **90% of the variance** (A: 6 components, B: 7, C: 7, D: 4).
- Fixes a 10,000-customer sample for plots.

### 5.1 – Run K-Means
- Settings: `k = 3, 4, 5, 6`; 5 seeds (42, 7, 21, 99, 123); `n_init = 10`; silhouette on a fixed 15,000-customer sample.
- 4 sets × 2 variants × 4 values of k × 5 seeds = **160 fits** (about 230 s).
- Seed 42 is the main run; the other seeds measure **stability**.
- **Both variants are evaluated in the original standardized space**, so the metrics are comparable.

| Metric | Meaning |
|---|---|
| Silhouette | Cluster separation (higher is better) |
| Inertia | Used for the elbow plot |
| Stability (ARI) | Agreement between seeds |
| Min cluster % | Sanity check on tiny clusters |

### 5.2 – Results table
- One row per set, variant and k (32 configurations).

### 5.3 – Metric plots
- Elbow plot (inertia relative to k = 3), plus silhouette and stability for each set, original vs. PCA.

### 5.4 – Similarity between variants
- ARI heatmap between the original-feature clusters and the PCA clusters.

### 5.5 – Cluster maps
- Side-by-side 2D maps (PC1 vs PC2) for A_core, one row per k.
- PCA cluster labels are re-matched to the original ones (Hungarian method) so colors are comparable.
- Interactive 3D maps for k = 4, 5 and 6.

### 5.6 – Summary
- Average difference (PCA minus original) of silhouette and stability over all sets and k.
- The notebook later treats **A_core with k = 4** as the best K-Means result (stated in the Part 4 settings).

---

## Part 4 – DBSCAN and comparison with K-Means (Step 6)

DBSCAN finds dense regions and labels sparse points as **noise (−1)**. It needs `eps` (neighborhood radius) and `min_samples`. It uses the same standardized matrix as K-Means (A_core).

### 6.0 – Settings
- `DB_SET = A_core`, compared with K-Means `k = 4`.
- `min_samples` ∈ {7, 14, 28}; `eps` = knee value × {0.6 … 1.4}.
- Selection limits: noise ≤ 20%, largest cluster ≤ 95%. Memory guard on neighbor pairs (50M).

### 6.1 – k-distance plot
- Sorted distance to the `min_samples`-th neighbor; the knee is found automatically.
- Resulting `eps` grid: **0.687, 0.916, 1.145, 1.374, 1.603**.

### 6.2 – Run DBSCAN over the grid
- 15 solutions. For each: number of clusters, noise %, largest cluster %, silhouette (non-noise points), ARI vs. K-Means.
- Notable results:
  - `eps = 0.687`: many clusters and high noise (16–39%), silhouette near 0 or negative.
  - `eps = 0.916 – 1.145`: 2–4 clusters, low noise, silhouette ≈ 0.23–0.26.
  - `eps ≥ 1.374`: everything collapses into **one cluster** (> 98%), silhouette undefined.

### 6.3 – Heatmaps
- Silhouette, ARI, noise % and number of clusters over the `eps` × `min_samples` grid. Silhouette must be read together with noise %.

### 6.4 – Final DBSCAN solution
- Automatic rule: highest silhouette among solutions with ≥ 2 clusters, noise ≤ 20% and largest cluster ≤ 95%.
- **Chosen: `eps = 0.916`, `min_samples = 14`** → 2 clusters, 7.7% noise, largest cluster 74.5%, silhouette 0.258, ARI vs. K-Means 0.317.
- Sizes: D1 = 74.5%, D2 = 17.8%, noise = 7.7%.

### 6.5 – Metrics: DBSCAN vs. K-Means
| Model | Customers scored | Silhouette |
|---|---|---|
| K-Means k = 4 | all | 0.211 |
| K-Means k = 4 | DBSCAN non-noise only | 0.226 |
| DBSCAN (0.916, 14) | non-noise only | 0.258 |

- ARI of DBSCAN against each K-Means solution: k=3 **0.411**, k=4 0.317, k=5 0.242, k=6 0.177. DBSCAN is closest to k = 3.

### 6.6 – Cross-table (K-Means clusters vs. DBSCAN clusters)
- K1 and K2 both fall almost entirely into D1, so DBSCAN **merges** them.
- K3 maps almost entirely to D2.
- K0 is split (D1, D2 and noise).
- Each K-Means cluster loses roughly 5–23% of its customers to noise.

### 6.7 / 6.8 – 2D and 3D side-by-side maps
- Same 10,000 customers and same PCA axes. DBSCAN clusters take the color of their best-matching K-Means cluster; noise is drawn in black.
- The 3D version is interactive (legend entries can be toggled).

### 6.9 – Summary of the comparison
- DBSCAN's silhouette is higher (0.258 vs. 0.226 on the same customers).
- DBSCAN leaves 7.7% of customers unsegmented and produces one dominant group (74.5%), i.e. **little real segmentation**.
- ARI 0.32 → the two models are only partly similar.

### Extra experiment cells (user-adjustable settings)
Three runs of K-Means vs. DBSCAN on A_core with different parameters:

| K-Means k | DBSCAN eps / min_samples | DBSCAN result | ARI |
|---|---|---|---|
| 2 | 0.99 / 14 | 2 clusters, 5.0% noise, silhouette 0.253 (K-Means 0.179) | 0.105 |
| 3 | 1.5 / 20 | **1 cluster only**, silhouette undefined (eps too large) | 0.000 |
| 3 | 0.99 / 15 | 2 clusters, 5.2% noise, silhouette 0.253 (K-Means 0.210) | 0.411 |

Each run is followed by 2D and 3D comparison maps.

---

## Overall takeaways
1. The pipeline goes from raw tables to **one row per customer**, with log-transforms for skewed features and standardization for distance-based models.
2. Four feature sets were compared. **A_core (7 features)** was the main candidate; the baseline D has a redundant pair (credit card vs. boleto, r = −0.89); set C contains very rare features with extreme z-scores.
3. **K-Means** was tested with and without PCA (90% variance), judged by silhouette, seed stability and cluster size. **k = 4 on A_core** was selected as the K-Means reference.
4. **DBSCAN** gives a slightly higher silhouette, but it mostly finds one large dense group plus a smaller one, and it discards 5–8% of customers as noise. It is sensitive to `eps`: above about 1.3 everything becomes a single cluster.
5. For actual segmentation, **K-Means gives more usable, balanced segments**; DBSCAN is more useful as a check on the density structure and on outliers.

## Note on the notebook
The `DBSCAN input` line printed in 6.0 shows shape `(25934, 7)`, while the features file has 92,661 customers (Part 2). The saved outputs may come from different runs or a subsample, so it is worth re-running the notebook top to bottom before reporting final numbers.
