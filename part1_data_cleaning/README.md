# Retail Orders Data Cleaning & Reporting
**Part 1 — Business Data Cleaning, Validation & Excel Reporting**\
**June 2026**

---

## 1. Problem Summary

A retail company exports order-level sales data from multiple internal systems (OMS, payment gateway, ERP). The raw export contains a wide range of data quality issues: inconsistent text formatting, date anomalies, duplicate records, missing values, invalid discounts, order status typos, and sales/profit calculation mismatches.

The objective of this project is to:
- Preserve the raw data unchanged for audit purposes
- Clean, validate, and standardise the dataset following documented business rules
- Produce calculated columns to support reliable financial analysis
- Summarise all data quality issues in an evaluator-ready report
- Build pivot summaries that answer key business questions
- Document every decision, assumption, and limitation

---

## 2. Dataset Description

| Property | Detail |
|---|---|
| **Source file** | `data/raw_orders.xlsx` — Sheet: `raw_orders` |
| **Total records** | 900 rows |
| **Columns (raw)** | 21 fields |
| **Columns (cleaned)** | 35 fields (21 original + 8 required calculated + 6 bonus) |
| **Date range** | January 2024 – December 2025 |
| **Geographies** | North, South, East, West, Unknown (5 regions) |
| **Categories** | Electronics, Furniture, Appliances, Clothing, Office Supplies |
| **Customer segments** | Consumer, Corporate, Home Office |
| **Ship modes** | Standard Class, Second Class, First Class, Same Day |
| **Order statuses** | Completed, Cancelled, Processing, Returned, Refunded, Failed |

### Raw Column List
`order_id` · `order_date` · `ship_date` · `customer_id` · `customer_name` · `segment` · `region` · `state` · `city` · `category` · `sub_category` · `product_name` · `ship_mode` · `quantity` · `unit_price` · `discount` · `sales` · `cost` · `profit` · `payment_status` · `order_status`

---

## 3. Tools Used

| Tool | Version / Purpose |
|---|---|
| **Python 3.12** | Primary scripting language for all data tasks |
| **pandas** | Data loading, profiling, cleaning, aggregation, pivoting |
| **numpy** | Numerical operations, random seed for reproducibility |
| **openpyxl** | Excel file creation, formatting, conditional formatting |
| **matplotlib** | Screenshot / preview image generation |
| **dateutil** | Robust date parsing for mixed-format date strings |
| **Microsoft Excel** | Final file review and formula verification |
| **LibreOffice (recalc.py)** | Server-side formula recalculation and error verification |

---

## 4. Cleaning Steps Performed

### Task 1 — Preserve Raw Data
- Raw file copied verbatim to `data/raw_orders.xlsx`; never modified
- Cleaning output: `data/cleaned_orders.xlsx` (separate file)

### Task 2 — Initial Profiling and Standardisation
- Column headers: stripped, lowercased, underscored
- Text fields: `.strip()` across all character columns
- Numeric fields: cast from string to float; non-parseable values flagged
- Date fields: parsed with `dateutil`; malformed strings flagged, not silently dropped

### Task 3 — Validation Flags
Three flag columns added:
- `date_validation_flag` — `SHIP_BEFORE_ORDER`, `UNUSUALLY_LONG_DELAY`, `FUTURE_ORDER`, `NULL_DATE`
- `discount_validation_flag` — `INVALID_NEGATIVE_DISCOUNT`, `INVALID_HIGH_DISCOUNT`, `MISSING_DISCOUNT`
- `status_validation_flag` — `INVALID_STATUS` with nearest valid value noted

Duplicate detection: exact duplicate rows identified; conflicting order_ids flagged.

### Task 4 — Duplicate Handling
- 32 exact duplicate rows: one copy retained; others excluded from analysis (flagged `DUPLICATE_REMOVED`)
- 12 conflicting duplicates: both copies retained with `CONFLICTING_DUPLICATE`; no auto-resolution

### Task 5 — Business Rules Applied
Nine rules (BR-01 to BR-09) applied; `business_rule_flag` column added. See Section 5.

### Task 6 — Calculated Columns
Eight required + six bonus columns added. Key columns:

| Column | Formula |
|---|---|
| `cleaned_discount` | Standardised decimal; null → 0 (BR-03); negative preserved & flagged |
| `calculated_sales` | `quantity × unit_price × (1 − MAX(0, cleaned_discount))` |
| `calculated_profit` | `calculated_sales − cost` |
| `profit_margin` | `calculated_profit / calculated_sales` |
| `shipping_delay_days` | `ship_date − order_date` (integer days) |
| `order_month` | Month name from `order_date` |
| `order_year` | Year from `order_date` |
| `data_quality_flag` | 6-check priority chain → `CLEAN` or named issue |

---

## 5. Business Rules Applied

| Rule | Condition | Action | Rows |
|---|---|---|---|
| BR-01 | `region` is null | Fill `'Unknown'` + flag | 25 |
| BR-02 | `ship_mode` is null | Fill `'Unknown'` + flag | 20 |
| BR-03 | `discount` is null (sales fields valid) | Set to 0 + flag | 18 |
| BR-04 | `discount < 0` | Preserve + flag `INVALID_NEGATIVE_DISCOUNT` | 15 |
| BR-05 | `discount > 0.50` | Preserve + flag `INVALID_HIGH_DISCOUNT` | 15 |
| BR-06 | `order_status = 'Cancelled'` | Exclude from revenue + flag | 145 |
| BR-07 | `payment_status = 'Failed'` | Exclude from revenue + flag | 67 |
| BR-08 | `order_status IN ('Returned','Refunded')` | Track separately + flag | 154 |
| BR-09 | `ship_date < order_date` | Flag `INVALID_SHIP_DATE`; sale valid | 92 |

---

## 6. Summary of Data Quality Issues Found

| # | Issue Type | Issues Found | Rows Affected | Severity |
|---|---|---|---|---|
| 1 | Missing Values | 8 columns | 90 total | Medium–High |
| 2 | Duplicate Records | 44 inflated rows | 44 | High |
| 3 | Invalid Discount | 30 records | 30 | High |
| 4 | Date Issues | 7 issue types | 114 | High |
| 5 | Order Status Issues | 16 invalid values | 20 | Medium |
| 6 | Sales/Profit Calc Mismatch | 5 mismatch types | 75 | High |
| 7 | Business Rule Flags | 9 flag types | 551 (overlap) | Various |

### Final Record Count
| Category | Count | % | Revenue |
|---|---|---|---|
| Total raw records | 900 | 100% | — |
| Exact duplicates removed | 32 | 3.6% | — |
| **Completed + clean (revenue base)** | **392** | **43.6%** | **₹59,64,435** |
| Cancelled (excluded) | 145 | 16.1% | — |
| Failed payment (excluded) | 67 | 7.4% | — |
| Returned / Refund eligible | 154 | 17.1% | ₹15,33,037 at risk |
| Processing / In-flight | 108 | 12.0% | Pending |
| Flagged (any flag) | 508 | 56.4% | See report |

**Overall data quality score: 43.6%** (392 fully clean of 900 total records)

---

## 7. Summary of Final Pivot Reports

All pivots in `outputs/pivot_summary.xlsx`. Revenue figures are from Completed orders unless stated.

### P1 — Sales & Profit by Region *(sorted by Sales ↓)*
| Region | Orders | Total Sales (₹) | Profit Margin % |
|---|---|---|---|
| North | 255 | 38,24,524 | 13.15% ← **Highest** |
| West | 229 | 31,37,145 | 12.36% |
| South | 213 | 27,36,672 | 10.23% |
| East | 171 | 23,23,755 | 9.29% |
| Unknown | 32 | 3,54,337 | 9.26% ← **Lowest** |
| **TOTAL** | **900** | **1,23,76,432** | **11.47%** |

### P2 — Sales & Profit by Category & Sub-Category *(sorted within category)*
Top categories by completed sales: Appliances (₹27.9L) › Electronics (₹16.6L) › Furniture (₹11.7L) › Clothing (₹2.3L) › Office Supplies (₹1.2L)

### P3 — Order Count by Ship Mode *(filtered: Completed only)*
Standard Class dominates at 49.9% of completed orders (227 of 455). Same Day is smallest at 7.5% (34 orders).

### P4 — Profit Margin by Customer Segment *(sorted by Margin ↓)*
| Segment | Avg Margin % | Total Profit (₹) |
|---|---|---|
| Corporate | 12.75% ← **Best** | 2,44,396 |
| Consumer | 11.30% | 2,26,624 |
| Home Office | 9.91% ← **Lowest** | 2,10,359 |

### P5 — Non-Revenue Orders by Region *(filtered: non-revenue statuses)*
Total sales at risk across Cancelled + Failed + Returned + Refunded = **₹48,04,020** across 333 orders. North has the highest cancelled value at ₹7,00,429 (54 orders).

### P6 — Monthly Sales Trend *(sorted by date ↑)*
- **2024 total:** ₹63,78,906 across 487 orders | Best month: April (+30.1% MoM)
- **2025 total:** ₹59,97,526 across 413 orders | Best month: January 2025 (₹8,88,260)
- Most volatile month: October 2025 (+88.6% MoM); November 2025 worst decline (−66.9%)

---

## 8. Key Business Insights

1. **North is the highest-performing region** in both total sales (₹38.2L) and profit margin (13.15%), contributing 28.3% of all-order revenue. Resources and marketing investment should be weighted here.

2. **East and Unknown regions have the lowest margins** (9.29% and 9.26%) — both below the dataset average of 11.47%. East should be investigated for cost-reduction opportunities.

3. **Corporate segment yields the best margins** (12.75% vs. 9.91% for Home Office). Targeting B2B corporate customers provides a meaningful margin advantage.

4. **56.4% of records carry at least one quality flag** — a significant data quality debt. Of particular concern: 145 cancelled orders and 154 returned/refunded orders together represent ₹48L+ in unrecognised revenue risk.

5. **92 records have impossible ship dates** (ship before order) — indicating a systemic data entry problem in the logistics system. These records must be corrected at source before shipping performance KPIs can be trusted.

6. **Standard Class is the dominant shipping method** (49.9% of completed orders). Reviewing whether First Class (16.9%) and Same Day (7.5%) deliver incremental margin or are cost centres would inform logistics strategy.

7. **Appliances category drives the most completed-order revenue** (₹27.9L, avg ticket ~₹29,630), making it the critical category for both revenue growth and discounting controls.

8. **Q4 seasonality is visible:** December 2024 (+25.4% MoM) and January 2025 (+24.4% MoM) show a strong year-end/new-year spike, which should inform inventory and staffing planning.

---

## 9. Assumptions and Limitations

### Assumptions
1. **50% discount ceiling** — no company policy available; threshold is assumed retail-standard
2. **Null discount = 0** — only where all other sales fields are valid (BR-03)
3. **Invalid ship dates do not invalidate the sale** — treated as logistics system error
4. **Cancelled = no revenue** — regardless of payment status; refund liability assumed to exist elsewhere
5. **Processing = in-flight** — not included in or excluded from revenue; held in limbo
6. **Conflicting duplicate resolution deferred** — conservative choice (lower sales) recommended but not enforced
7. **Calculation tolerance ±₹1** — smaller differences are rounding artefacts, not errors
8. **Profit margin outliers at −10% / +90%** — analyst-defined heuristics; should be confirmed with finance

### Limitations
1. No access to source OMS, payment gateway, or ERP — 92 date errors, 12 conflicting duplicates, and 46 calc mismatches cannot be definitively resolved
2. Discount threshold is assumed — may be wrong for clearance/loyalty categories
3. Invalid status strings mapped conservatively (→ `Processing`, not `Completed`) — may understate completed revenue
4. No CRM join possible — 6 null customer names cannot be resolved
5. Dataset is a single point-in-time snapshot — `Processing` orders may have since resolved
6. No referential integrity check — orphan records cannot be identified without dimension tables
7. All 900 source rows retained in cleaned file — "removed" means excluded from analytics, not deleted

---

## 10. Screenshots

All screenshots are rendered from actual dataset values and are located in `outputs/`.

### `raw_data_preview.png` — Raw dataset before cleaning
![Raw Data Preview](raw_data_preview.png)

Shows `raw_orders.xlsx` with highlighted data quality issues:
- 🔴 Red cells — invalid values (negative discount, bad status, ship before order, duplicate rows)
- 🟡 Amber cells — null/missing values (NULL customer, NULL region, NULL ship_date, NULL discount)
- Row ORD-0005 appears twice — exact duplicate
- Row ORD-0003 — negative discount (−15%) and invalid status "Complet"

---

### `cleaned_data_preview.png` — Cleaned dataset with calculated columns
![Cleaned Data Preview](cleaned_data_preview.png)

Shows `cleaned_orders.xlsx` after all 6 tasks:
- **Navy columns** — original 21 columns, standardised
- **Green columns** — 8 required calculated columns (`cleaned_discount`, `calculated_sales`, etc.)
- **Amber columns** — business rule and data quality flag columns
- Flagged cells shown in context (red = error, amber = warning, green = clean)

---

### `pivot_summary_1.png` — P1: Sales by Region + P4: Margin by Segment
![Pivot Summary 1](pivot_summary_1.png)

Left panel: P1 — 5 regions sorted by total sales descending; colour-scaled profit margin column (green = highest, red = lowest). Bar chart inset showing relative regional sales. Right panel: P4 — 3 customer segments sorted by average margin descending; donut chart showing order share per segment.

---

### `pivot_summary_2.png` — P6: Monthly Trend + P5: Non-Revenue by Region
![Pivot Summary 2](pivot_summary_2.png)

Left panel: P6 — 24-month sales trend (Jan 2024 – Dec 2025) with MoM growth % colour-coded green (gain) / red (decline). Right panel: P5 — non-revenue orders filtered to Cancelled, Failed, Returned, Refunded; broken down by region and status with business rule references.

---

## Output Files

| File | Description | Task |
|---|---|---|
| `data/raw_orders.xlsx` | Original raw data — never modified | Task 1 |
| `data/cleaned_orders.xlsx` | Cleaned + validated + calculated columns (900 rows × 35 cols) | Tasks 2–6 |
| `outputs/data_quality_report.xlsx` | 8-sheet quality report (Dashboard + 7 issue summaries) | Task 7 |
| `outputs/pivot_summary.xlsx` | 7-sheet pivot report (Index + 6 pivot tables) | Task 8 |
| `outputs/cleaning_log.md` | Full cleaning log (issues, rules, assumptions, limitations) | Task 9 |
| `outputs/README.md` | This file | Task 9 |
| `outputs/raw_data_preview.png` | Screenshot of raw data with issues highlighted | Task 9 |
| `outputs/cleaned_data_preview.png` | Screenshot of cleaned data with calculated columns | Task 9 |
| `outputs/pivot_summary_1.png` | Screenshot of P1 (Region) + P4 (Segment) pivots | Task 9 |
| `outputs/pivot_summary_2.png` | Screenshot of P6 (Monthly Trend) + P5 (Non-Revenue) pivots | Task 9 |

---

*All figures in this README reconcile directly with `outputs/data_quality_report.xlsx` (Task 7) and `outputs/pivot_summary.xlsx` (Task 8). Any re-run of the cleaning pipeline from `raw_orders.xlsx` should reproduce the same results given the same business rules and random seed (seed=7 for synthetic augmentation).*
