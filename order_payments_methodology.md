# Olist Order Payments: Cleaning, Transformation and Dataset Description

Notebook: `order_payment.ipynb` (Google Colab, pandas)
Input: `olist_order_payments_dataset.csv`
Outputs: `olist_order_payments_clean.csv` (cleaned, one row per payment) and `order_payments_clean.csv` (aggregated, one row per order)

---

## 1. Goal

The payments table has to feed the ML objectives (unusual-order detection, customer segmentation, and possibly delivery prediction). Models need one row per order, so the work has two stages:

1. **Cleaning:** make the payment-level table consistent and valid.
2. **Transformation:** turn it into one row per order without losing the payment information that matters.

---

## 2. Original dataset

One row per payment. An order can be paid in several payments, for example a voucher plus a credit card.

| Column | Description |
|---|---|
| `order_id` | Order identifier (key to the other Olist tables) |
| `payment_sequential` | Position of the payment inside the order (1, 2, 3, ...) |
| `payment_type` | `credit_card`, `boleto`, `voucher`, `debit_card` or `not_defined` |
| `payment_installments` | Number of installments chosen |
| `payment_value` | Amount of the payment |

- **Rows:** 103,886
- **Grain:** `order_id` + `payment_sequential`

---

## 3. Cleaning (payment level)

| # | Step | Why |
|---|---|---|
| 1 | Normalize `order_id` and `payment_type`: strip spaces, lowercase, replace inner whitespace with `_` | Keys and categories must be written the same way in every table, or joins and encodings silently split one value into two |
| 2 | Map `not_defined` with a non-zero value to `credit_card` | A payment with money in it has a real method, and credit card is the one that fits. This changed 0 rows in this dataset, so the rule is a safeguard |
| 3 | Delete the 9 payments with `payment_value = 0` | A payment of zero carries no information and would distort counts, minimums and means. The `not_defined` rows were all zero-value, so this step also removes the whole `not_defined` category |
| 4 | Renumber `payment_sequential` within each order after the deletions | Without it, an order could start at 2 or have gaps, which breaks the key logic and any "first payment" feature |
| 5 | Set `payment_installments = 0` to 1 (2 rows) | A payment is at least one installment. Zero is an encoding artifact, and it would distort averages and maxima |
| 6 | Final asserts: shape, no `not_defined`, installments >= 1, values > 0 | Fail loudly if a later rerun produces something unexpected |

**Result:** 103,877 rows (103,886 minus the 9 zero-value rows).

**What was deliberately not changed**
- **Large payments were kept.** They are likely real high-value orders, and removing them would hide exactly what unusual-order detection looks for. The maximum order total is 13,664.08.
- **Split payments were kept as they are.** They are real behavior (for example several vouchers in one order).

**Side effect to know about:** the 9 deleted rows belong to 8 orders. 3 of those orders had only a zero-value payment, so they no longer appear in the payments tables. After joining with items or orders, these 3 will show as having no payment record, which is correct.

### Exploration used to check the data (visualisation section of the notebook)
- Histogram of `payment_value` on a log scale: most payments fall between 0 and about 3,000, with a long tail.
- Payments per order: most orders are paid with a single payment.
- Boxplots of value by payment type and of credit-card value by number of installments.
- Scatter plot of value vs installments, colored by payment type (jittered installments, log value axis).

---

## 4. Transformation (order level)

The aggregation is split into separate notebook cells so each step can be checked and rerun on its own.

### 4.1 One-hot encoding of the payment type
`pd.get_dummies` creates one column per payment type (`n_boleto`, `n_credit_card`, `n_debit_card`, `n_voucher`), added to the payment-level table so the encoding can be inspected. At this stage each row has exactly one 1.

**Why:** models can't use text categories directly, and a payment type column has no meaning once there are several payments per order.

### 4.2 Counts and totals per type
- **Counts:** the one-hot columns are summed per order. The encoding is no longer 1/0: each column holds **how many times that payment type was used in the order**.
- **Totals:** `value_<type>` is the total paid with each type in the order.

**Why counts instead of 1/0:** an order paid with three vouchers is different from one paid with a single voucher, and 1/0 would hide it. 800 orders have more than one voucher payment.

**Why totals per type:** counts alone say how often a type was used, not how much was paid with it. "A small voucher plus a large card payment" is different from "mostly voucher". Per-type max and min were considered and left out, since most orders have one payment per type and they would mostly repeat the total.

**Why not `payment_type1/2` columns:** an earlier design used these. Per-type columns don't depend on which payment came first and don't need a new column if a third type appears.

### 4.3 Order-level aggregates

| Column | Calculation |
|---|---|
| `n_payments` | number of payments in the order |
| `total_paid` | sum of `payment_value` |
| `max_payment` | largest payment |
| `min_payment` | smallest payment |
| `max_installments` | largest number of installments |
| `n_payment_types` | number of distinct payment types |

### 4.4 Combine and verify
The two index-aligned tables (type features and order aggregates) are joined on `order_id` and `order_id` is restored as a column.

Checks that must pass:
- `order_id` is unique.
- Number of rows equals the number of distinct orders in the cleaned payments.
- For every order, the type counts sum to `n_payments`.
- For every order, the `value_` columns sum to `total_paid`.
- The grand total of `total_paid` equals the grand total of the payment-level `payment_value`.

Together these checks show that the aggregation lost no money and no payments.

---

## 5. Clean aggregated dataset: `order_payments_clean.csv`

- **Rows:** 99,437 (one per order)
- **Columns:** 15
- **Grain:** `order_id` (unique)
- **Missing values:** none
- **Payments represented:** 103,877 (the sum of `n_payments`)

| Column | Type | Description |
|---|---|---|
| `order_id` | text | Order identifier, unique |
| `n_payments` | int | Number of payments in the order |
| `total_paid` | float | Total amount paid |
| `max_payment` | float | Largest single payment |
| `min_payment` | float | Smallest single payment |
| `max_installments` | int | Largest installment count among the order's payments |
| `n_payment_types` | int | Number of distinct payment types (1 or 2) |
| `n_boleto` | int | Number of boleto payments |
| `n_credit_card` | int | Number of credit card payments |
| `n_debit_card` | int | Number of debit card payments |
| `n_voucher` | int | Number of voucher payments |
| `value_boleto` | float | Amount paid by boleto |
| `value_credit_card` | float | Amount paid by credit card |
| `value_debit_card` | float | Amount paid by debit card |
| `value_voucher` | float | Amount paid by voucher |

### Profile of the data

**Payments per order**
- 97.0% of orders have a single payment. 2,961 orders have more than one and 578 have more than two (maximum 27, many of them vouchers).
- 2,246 orders use two payment types. No order uses more than two.

**Payment types** (share of orders using each type; these add up to more than 100% because some orders mix types)

| Type | Orders | Share |
|---|---|---|
| Credit card | 76,505 | 76.9% |
| Boleto | 19,784 | 19.9% |
| Voucher | 3,866 | 3.9% |
| Debit card | 1,528 | 1.5% |

**Amounts**
- `total_paid`: median 105.29, mean 161.00, 95th percentile 453.01, 99th percentile 1,075.80, maximum 13,664.08, minimum 9.59.
- The smallest single payment is 0.01.

**Installments**
- `max_installments` ranges from 1 to 24 and 48.5% of orders have 1.
- Values of 8 and 10 are more common than their neighbors, so installments don't follow a smooth pattern.

---

## 6. Limitations and notes

- **Orders without payments are absent.** Only orders with at least one payment appear. After joining with items or orders, check for missing payment features, including the 3 orders whose only payment was zero-value, and the delivered order that has items but no payment.
- **Unused types are zero, not missing.** `n_` and `value_` columns are 0 when a type wasn't used, which is correct for these columns, but a zero in a value column doesn't mean a zero-value payment.
- **Redundant columns.** The `n_` columns sum to `n_payments` and the `value_` columns sum to `total_paid`. This is harmless for tree-based models and Isolation Forest, but for linear models or clustering one column of each group should be dropped or the redundancy kept in mind.
- **Payments don't always equal the order's items total.** Payments can differ from item price plus freight (installment interest or discounts are a likely cause, not verified), so use `total_paid` as a payment feature and don't treat it as the order's revenue.
- **Notebook housekeeping.** The cleaning logic appears twice in the notebook: first as exploratory cells, then as one consolidated cell. The consolidated cell is the version that matters. Its `assert shape == (103877, 5)` is only valid on the original data, so it should be removed if the notebook is run on different data.
- **Next step.** Aggregate the order items table the same way and join both on `order_id`, with `orders` as the base table once it is ready.
