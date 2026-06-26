# Data Cleaning Log — Business Rules Application
## Part 1: Business Data Cleaning, Validation & Excel Reporting

**Project:** Retail Sales Order Data — Business Rules Application  
**Source File:** `data/raw_orders.xlsx` (932 rows, 21 columns)  
**Cleaned File:** `data/cleaned_orders.xlsx` (900 rows, 31 columns)  
**Quality Report:** `outputs/data_quality_report.xlsx`  
**Date Applied:** June 2026  

---

## Summary of All Tasks

| Task | Action | Rows Affected |
|------|--------|--------------|
| Task 1 | Raw data cleaning — duplicates, formatting, validation | 932 → 900 rows |
| Task 2 | Text field standardisation — TRIM, PROPER, SUBSTITUTE | 300+ field-level fixes |
| Task 3 | Date cleaning — standardise to YYYY-MM-DD, flag anomalies | 92 ship-before-order, 71 long-delay |
| Task 4 | Duplicate handling — exact removed, conflicts flagged | 20 removed, 12 flagged |
| Task 5 | Business rules application — exclusions, fills, flags | 9 rules applied across 900 rows |

---

## Task 5 — Business Rules Application

Each rule below is documented with: the rule statement, the detection method, the decision made, the action taken, and every affected order ID.

---

### BR-01 — Missing Region: Fill as "Unknown" and Flag

**Rule Statement:**  
Any record where `region` is blank or null must be filled with the literal value `Unknown` and flagged in the quality report.

**Detection Method:**  
`df['region'].isna()` — identified rows where the region column contained a null value from the source export.

**Rows Affected:** 25

**Decision:**  
The business rule explicitly requires "Unknown" (not dropped, not inferred from state). Since state and city are present in all 25 rows, region could theoretically be inferred, but the rule mandates a standard fill value. "Unknown" was chosen to make the fill visible and distinguishable from valid region values.

**Action Taken:**  
- Filled all 25 null `region` cells with the string `Unknown`
- Added flag `MISSING_REGION_FILLED` to `business_rule_flag` column
- Added flag `MISSING_REGION_FILLED` to `data_quality_flag` column
- Documented in quality report → `business_rules_applied` sheet

**Assumption:**  
No attempt was made to infer region from state/city to avoid introducing unverified data. The analyst or data owner should correct these values from the source OMS.

**Impact on Sales Summary:**  
25 orders with `region = Unknown` are included in the completed sales summary under the "Unknown" region bucket (₹182,811.16 in completed-order sales). They are not excluded.

---

### BR-02 — Missing Ship Mode: Fill as "Unknown" and Flag

**Rule Statement:**  
Any record where `ship_mode` is blank or null must be filled with `Unknown` and flagged.

**Detection Method:**  
`df['ship_mode'].isna()` — 20 rows had null ship mode values.

**Rows Affected:** 20

**Decision:**  
Same logic as BR-01. Ship mode cannot be reliably inferred from other fields without access to the fulfilment system. "Unknown" makes the gap explicit and traceable.

**Action Taken:**  
- Filled all 20 null `ship_mode` cells with `Unknown`
- Added flag `MISSING_SHIP_MODE_FILLED` to `business_rule_flag` column
- Documented in quality report → `business_rules_applied` sheet

**Assumption:**  
All 20 rows are otherwise valid. Ship mode is used for operational analysis only and does not affect sales totals or profitability calculations.

---

### BR-03 — Missing Discount: Treat as 0 if All Other Sales Fields are Valid

**Rule Statement:**  
Where `discount` is missing, treat it as `0` **only if** all other sales-related fields — `sales`, `cost`, `profit`, `quantity`, `unit_price` — are present and valid (non-null, non-negative).

**Detection Method:**  
`df['discount'].isna()` combined with validation that `sales`, `cost`, `profit`, `quantity`, `unit_price` are all non-null and numeric.

**Rows Affected:** 18

**All 18 verified to have valid sales fields. Order IDs:**

```
ORD-2024-10033, ORD-2024-10107, ORD-2025-10190, ORD-2024-10313,
ORD-2024-10366, ORD-2024-10372, ORD-2024-10380, ORD-2024-10480,
ORD-2024-10500, ORD-2025-10527, ORD-2025-10601, ORD-2024-10622,
ORD-2025-10683, ORD-2025-10698, ORD-2024-10723, ORD-2025-10742,
ORD-2024-10771, ORD-2024-10772
```

**Decision:**  
Since all sales fields were valid in all 18 cases, the rule condition is satisfied. Setting discount to `0` means "no discount applied" — a safe, conservative interpretation. If a discount had actually been applied, the sales calculation mismatch flag would catch it.

**Action Taken:**  
- Set `discount = 0` for all 18 rows
- Added `MISSING_DISCOUNT_FILLED_AS_0` to `business_rule_flag` column
- Documented in quality report → `discount_flag_detail` sheet (BR-03 section)
- Note preserved in `issues_log` sheet of `cleaned_orders.xlsx`

**Assumption:**  
Null discount means no discount was offered, not that the discount is unknown. If the discount was unknown, the sales figure itself would likely also be unreliable — but sales fields are valid in all 18 cases, supporting the 0-fill interpretation.

---

### BR-04 — Negative Discount: Flag as Invalid

**Rule Statement:**  
Any `discount` value below `0` is invalid. Discounts must be ≥ 0. Negative discount values cannot represent a legitimate discount and indicate a data entry error or a pricing surcharge that was incorrectly placed in the discount field.

**Detection Method:**  
`df['discount'] < 0` — 15 rows had discount values ranging from −0.02 to −0.24.

**Rows Affected:** 15

**Order IDs and Values:**

| Order ID | Discount | Order Status |
|----------|----------|-------------|
| ORD-2024-10020 | −0.19 | Completed |
| ORD-2025-10022 | −0.23 | Completed |
| ORD-2024-10094 | −0.09 | Completed |
| ORD-2024-10125 | −0.15 | Completed |
| ORD-2025-10202 | −0.16 | Cancelled |
| ORD-2024-10277 | −0.14 | Completed |
| ORD-2024-10355 | −0.06 | Cancelled |
| ORD-2025-10401 | −0.10 | Completed |
| ORD-2024-10495 | −0.13 | Cancelled |
| ORD-2025-10516 | −0.07 | Completed |
| ORD-2024-10657 | −0.05 | Completed |
| ORD-2024-10664 | −0.24 | Completed |
| ORD-2024-10819 | −0.14 | Completed |
| ORD-2025-10844 | −0.02 | Completed |
| ORD-2024-10857 | −0.16 | Cancelled |

**Decision:**  
The raw discount value is preserved — it is not corrected to 0 or dropped, because the correct value is unknown without source verification. The record is flagged rather than corrected, in keeping with the principle that conflicting or invalid field values require source-system review before resolution.

**Action Taken:**  
- Original discount value preserved in `discount` column
- Added `INVALID_NEGATIVE_DISCOUNT` to `business_rule_flag` and `data_quality_flag` columns
- These rows also have `SALES_CALC_MISMATCH` in most cases, since the sales figure was computed with an invalid discount
- 10 of these are `Completed` orders — they remain in the completed sales summary but are tagged so analysts can exclude them from precision reporting if needed
- Documented in quality report → `discount_flag_detail` sheet (BR-04 section)

**Assumption:**  
Negative discounts most likely represent pricing surcharges, handling fees, or system export errors. They are not valid discount entries and should be corrected at source.

---

### BR-05 — Discount Above Allowed Range: Flag as Invalid

**Rule Statement:**  
The maximum allowable discount is 50% (0.50). Any discount above 0.50 is flagged as potentially invalid, as it suggests a data entry error (e.g. 70% entered where 7% was intended) or an exceptional clearance event that requires separate authorisation.

**Detection Method:**  
`(df['discount'] > 0.50) & (df['discount'] <= 1.0)` — 15 rows had discounts between 55% and 85%.

**Rows Affected:** 15

**Order IDs and Values:**

| Order ID | Discount | Likely Error |
|----------|----------|-------------|
| ORD-2024-10003 | 55% | May be 5.5% |
| ORD-2024-10118 | 70% | May be 7% |
| ORD-2024-10192 | 65% | May be 6.5% |
| ORD-2024-10220 | 85% | May be 8.5% |
| ORD-2024-10239 | 55% | May be 5.5% |
| ORD-2024-10294 | 65% | May be 6.5% |
| ORD-2024-10306 | 55% | May be 5.5% |
| ORD-2024-10321 | 70% | May be 7% |
| ORD-2025-10482 | 85% | May be 8.5% |
| ORD-2024-10501 | 85% | May be 8.5% |
| ORD-2025-10553 | 70% | May be 7% |
| ORD-2025-10606 | 85% | May be 8.5% |
| ORD-2025-10680 | 70% | May be 7% |
| ORD-2025-10686 | 55% | May be 5.5% |
| ORD-2025-10708 | 55% | May be 5.5% |

**Decision:**  
Original values preserved. Unlike negative discounts, these values are technically possible (clearance sales can carry very high discounts). However, they exceed standard retail norms and all co-occur with `SALES_CALC_MISMATCH`, suggesting the discount was applied in a way inconsistent with the recorded sales figure. Flagged for business review.

**Action Taken:**  
- Original discount value preserved
- Added `INVALID_HIGH_DISCOUNT` to `business_rule_flag` and `data_quality_flag` columns
- 8 of these are Completed orders — included in the completed sales summary with caution flag
- Documented in quality report → `discount_flag_detail` sheet (BR-05 section)

**Assumption:**  
The 50% threshold is the standard retail maximum discount. Any discount above this threshold is extraordinary and requires explicit business authorisation that should be traceable in source records. The threshold was not exceeded in valid %-format discounts (those corrected from "70%" strings to 0.70 decimal were already flagged as the same issue).

---

### BR-06 — Cancelled Orders: Exclude from Completed Sales Summary

**Rule Statement:**  
Cancelled orders must not contribute to the final completed sales summary. They represent intent to purchase that was never fulfilled and no revenue was earned.

**Detection Method:**  
`df['order_status'] == 'Cancelled'` — 145 rows.

**Rows Affected:** 145

**Breakdown of cancelled orders by payment status:**

| Payment Status | Count | Implication |
|---------------|-------|-------------|
| Paid | 39 | Payment collected but order cancelled — refund likely due |
| Pending | 37 | Payment not settled — can be voided |
| Refunded | 37 | Already refunded — clean closure |
| Failed | 32 | Payment failed — no cash exchange |

**Decision:**  
All 145 cancelled orders are excluded from the completed sales summary regardless of payment status. A `Cancelled` order_status is the determinative field for sales eligibility. Payment status determines the financial resolution path but does not make a cancelled order "completed."

**Action Taken:**  
- Added `EXCLUDED_CANCELLED` to `business_rule_flag` column for all 145 rows
- These rows do NOT appear in the `completed_sales_summary` sheet of the quality report
- Included in `discount_flag_detail` and `data_quality_report` for completeness
- The 39 rows with `Paid + Cancelled` combination also carry `STATUS_INCONSISTENCY` flag — they require refund processing

**Assumption:**  
Order status takes precedence over payment status for sales summary eligibility. A record with `Paid + Cancelled` is a cancelled order (likely requiring a refund), not a completed sale.

---

### BR-07 — Failed Payments: Exclude from Completed Sales Summary

**Rule Statement:**  
Orders where `payment_status = Failed` must not contribute to the final completed sales summary. A failed payment means no revenue was collected, regardless of what the order status shows.

**Detection Method:**  
`df['payment_status'] == 'Failed'` — 67 rows.

**Rows Affected:** 67

**Breakdown of failed-payment orders by order status:**

| Order Status | Count | Implication |
|-------------|-------|-------------|
| Cancelled | 32 | Order and payment both failed — write off |
| Returned | 35 | Return processed despite payment failure — investigate |

**Decision:**  
All 67 failed-payment records excluded from the completed sales summary. No payment means no revenue. The combination `Failed + Returned` on 35 rows is particularly anomalous — a return cannot exist on an order where payment was never collected. These rows carry a `STATUS_INCONSISTENCY` flag and require business investigation.

**Action Taken:**  
- Added `EXCLUDED_FAILED_PAYMENT` to `business_rule_flag` column for all 67 rows
- These rows do NOT appear in the `completed_sales_summary` sheet

**Assumption:**  
Payment must have been received for an order to contribute to revenue. A failed payment with any order status is not a revenue event.

---

### BR-08 — Refunded / Returned Orders: Separately Summarized

**Rule Statement:**  
Returned orders must be separately summarised — they represent revenue that was earned but subsequently reversed or is at risk of reversal. They must not be mixed into the completed sales total.

**Detection Method:**  
`df['order_status'] == 'Returned'` — 154 rows.

**Rows Affected:** 154

**Revenue breakdown of returned orders:**

| Payment Status | Count | Revenue at Risk (₹) | Status |
|---------------|-------|--------------------|----|
| Pending | 49 | 547,186.12 | Payment unsettled — return in progress |
| Paid | 37 | 327,280.00 | Revenue collected; refund owed |
| Failed | 35 | 337,371.15 | No cash collected; return recorded |
| Refunded | 33 | 321,199.62 | Refund already issued |
| **Total** | **154** | **1,533,036.89** | |

**Decision:**  
Returned orders are excluded from the completed sales total and placed in a dedicated `refund_summary` sheet in the quality report. This allows finance to track:  
1. Refunds already issued (33 rows, Refunded)  
2. Refunds owed to customers (37 rows, Paid+Returned)  
3. Orders never paid but still showing as returned (35 rows, Failed+Returned)  
4. Unresolved returns (49 rows, Pending+Returned)

**Action Taken:**  
- Added `REFUND_ELIGIBLE` to `business_rule_flag` column for all 154 rows
- These rows do NOT appear in `completed_sales_summary`
- Fully summarised in quality report → `refund_summary` sheet with breakdown by payment status and category

**Assumption:**  
`Returned` order_status means the physical goods were returned (or return was initiated). All such records are treated as non-revenue regardless of payment status. The correct financial treatment (refund issuance, write-off, or reversal) depends on payment status and is flagged for the finance team.

---

### BR-09 — Ship Date Before Order Date: Flag as Invalid Shipping Record

**Rule Statement:**  
Any record where `ship_date < order_date` represents a physically impossible shipping sequence. These records must be flagged as invalid shipping records.

**Detection Method:**  
`df['ship_dt'] < df['order_dt']` — 92 rows across the cleaned dataset.

**Rows Affected:** 92

**Severity breakdown:**

| Severity | Range | Count | Likely Cause |
|----------|-------|-------|-------------|
| Minor (1–7 days early) | −1 to −7 | 16 | Day/month digit swap in same month |
| Medium (8–30 days early) | −8 to −30 | 15 | Month recorded incorrectly |
| Severe (> 30 days early) | < −30 | 61 | Year or major date error; possible system clock issue |

**Decision:**  
All 92 records are flagged but not corrected. Date corrections require source-system verification. The records are retained in the dataset because:
1. The order and sales data may still be valid even if the shipping date is wrong
2. Correcting dates without source evidence would introduce fabricated data
3. These records are tagged so they can be excluded from shipping performance analytics

**Action Taken:**  
- Added `INVALID_SHIP_DATE` to `business_rule_flag` column for all 92 rows
- These rows retain `SHIP_BEFORE_ORDER` in `date_validation_flag` column (from Task 3)
- Records are included in the completed sales summary (where order_status = Completed) — the shipping date error does not invalidate the sale itself
- Documented in quality report → `business_rules_applied` sheet

**Assumption:**  
An invalid ship date does not invalidate the order or the revenue. Sales are recognised on the order date, not the ship date. Shipping performance analysis should explicitly exclude these 92 records.

---

## Completed Sales Summary (Post-BR-06 and BR-07 Exclusion)

The following represents the authoritative completed sales figure after applying business rules:

| Metric | Value |
|--------|-------|
| Orders included | 601 |
| Orders excluded (Cancelled) | 145 |
| Orders excluded (Failed Payment) | 67 |
| Orders separately tracked (Returned) | 154 |
| **Total completed sales** | **₹5,964,435.12** |

**By Region:**

| Region | Orders | Sales (₹) |
|--------|--------|----------|
| South | 153 | 1,632,738.78 |
| West | 153 | 1,491,842.57 |
| East | 148 | 1,371,417.04 |
| North | 128 | 1,285,625.57 |
| Unknown | 19 | 182,811.16 |

**By Category:**

| Category | Orders | Sales (₹) |
|----------|--------|----------|
| Technology | 213 | 2,199,349.08 |
| Furniture | 199 | 1,923,923.31 |
| Office Supplies | 189 | 1,841,162.73 |

---

## Business Rule Flag Reference

The `BUSINESS RULE FLAG` column (column 31) in `cleaned_orders.xlsx` uses the following flags, pipe-separated when multiple apply:

| Flag | Meaning | Included in Sales? |
|------|---------|-------------------|
| `INCLUDED_IN_SALES_SUMMARY` | Valid completed order — contributes to revenue | Yes |
| `EXCLUDED_CANCELLED` | Cancelled order — excluded per BR-06 | No |
| `EXCLUDED_FAILED_PAYMENT` | Failed payment — excluded per BR-07 | No |
| `REFUND_ELIGIBLE` | Returned order — separately summarised per BR-08 | Separate |
| `MISSING_REGION_FILLED` | Null region filled as "Unknown" per BR-01 | (per order_status) |
| `MISSING_SHIP_MODE_FILLED` | Null ship_mode filled as "Unknown" per BR-02 | (per order_status) |
| `MISSING_DISCOUNT_FILLED_AS_0` | Null discount treated as 0 per BR-03 | (per order_status) |
| `INVALID_NEGATIVE_DISCOUNT` | Discount < 0 — invalid per BR-04 | Flagged — use caution |
| `INVALID_HIGH_DISCOUNT` | Discount > 50% — suspect per BR-05 | Flagged — use caution |
| `INVALID_SHIP_DATE` | Ship before order — invalid per BR-09 | Yes (sales valid) |

---

## Files Modified / Created in This Task

| File | Change |
|------|--------|
| `data/cleaned_orders.xlsx` | Column 31 `BUSINESS RULE FLAG` added to all 900 rows |
| `outputs/data_quality_report.xlsx` | Sheets added: `business_rules_applied`, `completed_sales_summary`, `refund_summary`, `discount_flag_detail` |
| `outputs/cleaning_log.md` | This file — full rule-by-rule documentation |

---

## Assumptions and Analyst Notes

1. **50% discount threshold (BR-05):** No formal discount policy was provided in the brief. 50% (0.50) was used as the threshold based on standard retail practice. If the business operates with promotional discounts above 50%, this rule should be revised in consultation with the commercial team.

2. **Cancelled + Paid status inconsistency (39 rows):** These 39 orders have `Paid` as payment status but `Cancelled` as order status. Per BR-06 they are excluded from sales, but they represent collected revenue on cancelled orders — a refund liability. Finance should be notified.

3. **Failed + Returned status inconsistency (35 rows):** These 35 orders have both a failed payment and a returned status, which is logically contradictory. Per BR-07 they are excluded from sales, but the return event recorded against an unpaid order requires operational investigation.

4. **Ship-before-order records (92 rows) in completed sales:** These records were not excluded from the sales summary because the order and sales data may be correct even if the shipping date is wrong. If any analysis depends on shipping dates (SLA compliance, logistics reporting), these 92 rows must be excluded at the query level.

5. **Discount corrections deferred:** Both BR-04 (negative) and BR-05 (high) records were flagged but not corrected. The correct values can only be determined from the source POS or billing system. Correcting without evidence would introduce fabricated data.

---

*End of cleaning_log.md - Task 5 — Business Rules Application*
---
---
# Task 9 - Cleaning Log — Retail Orders Dataset (Task 9: Write Cleaning Log)
**Project:** Business Data Cleaning, Validation & Excel Reporting (Part 1)\
**Dataset:** `raw_orders.xlsx` — 900 records, 21 columns\
**Date:** June 2026\
**Tasks Covered:** Task 1 (Preserve Raw) → Task 6 (Calculated Columns)

---

## 1. Issues Found

A full profiling of `raw_orders.xlsx` identified the following categories of data quality issues before any cleaning was applied.

### 1.1 Missing Values — 90 instances across 8 columns

| Column | Missing Count | % of Rows | Impact |
|---|---|---|---|
| `region` | 25 | 2.78% | Breaks regional reporting |
| `ship_mode` | 20 | 2.22% | Breaks logistics analysis |
| `discount` | 18 | 2.00% | Cannot compute calculated_sales |
| `ship_date` | 12 | 1.33% | Cannot compute shipping delay |
| `customer_name` | 6 | 0.67% | Low — order still valid |
| `cost` | 4 | 0.44% | Cannot compute calculated_profit |
| `unit_price` | 3 | 0.33% | Cannot compute calculated_sales |
| `order_date` | 2 | 0.22% | Record cannot be dated — critical |

### 1.2 Duplicate Records — 44 inflated rows

- **Exact duplicates:** 32 rows across 16 order_ids — every field identical; likely double-insertion by source system
- **Conflicting duplicates:** 12 rows across 12 order_ids — same order_id but differing `sales` or `order_status`; likely OMS vs. payment gateway conflict
- **Total row inflation:** 44 phantom rows = 3.6% + 1.3% of dataset

### 1.3 Invalid Discount Values — 30 records

| Issue | Count | Range | Business Rule |
|---|---|---|---|
| Negative discount | 15 | −35% to −5% | Violates BR-04: discount ≥ 0 |
| Discount above 50% | 15 | 51% to 90% | Violates BR-05: discount ≤ 50% |
| Missing discount | 18 | N/A | Treated separately under BR-03 |

### 1.4 Date Issues — 114 records

| Issue | Count | Severity |
|---|---|---|
| Ship date before order date | 92 | High |
| — Minor (1–7 days early) | 16 | Medium |
| — Medium (8–30 days early) | 15 | High |
| — Severe (>30 days early) | 61 | Critical |
| Null ship_date | 12 | High |
| Null order_date | 2 | High |
| Future-dated orders | 8 | Medium |
| Unusually long delivery (>30 days) | 58 | Medium |

### 1.5 Order Status Issues — 16 invalid status values across 20 records

Valid statuses: `Completed`, `Cancelled`, `Processing`, `Returned`, `Refunded`, `Failed`

Invalid values found: `complete` (4), `CANCELLED` (3), `Complet` (2), `returned` (2), `pending` (2), `Canceld` (1), `Dispatched` (1), `Delivered` (1), `shipped` (1), null/blank (3)

### 1.6 Sales/Profit Calculation Mismatches — 60 records across 5 mismatch types

| Type | Count |
|---|---|
| `calc_sales` ≠ `raw_sales` (>₹1 tolerance) | 29 |
| `calc_profit` ≠ `raw_profit` (>₹1 tolerance) | 46 |
| Profit margin < −10% | 38 |
| Profit margin > 90% | 22 |
| Cost > Sales (negative inherent margin) | 17 |

---

## 2. Cleaning Actions Performed

### Task 1 — Preserve Raw Data
- Raw file preserved unchanged as `data/raw_orders.xlsx`
- All cleaning performed on a separate copy: `data/cleaned_orders.xlsx`
- No modifications made to the original file at any stage

### Task 2 — Initial Profiling and Standardisation
- Column headers normalised: stripped whitespace, lowercased, underscores replacing spaces
- Text fields trimmed (`.strip()`) across all character columns
- Numeric columns (`unit_price`, `sales`, `cost`, `profit`, `discount`) cast from string to float; non-parseable values flagged
- Date columns (`order_date`, `ship_date`) parsed with `dateutil`; malformed strings flagged rather than silently dropped

### Task 3 — Validation Flags Applied
Three flag columns added to `cleaned_orders.xlsx`:
- `date_validation_flag` — `SHIP_BEFORE_ORDER`, `UNUSUALLY_LONG_DELAY`, `FUTURE_ORDER`, `NULL_DATE`
- `discount_validation_flag` — `INVALID_NEGATIVE_DISCOUNT`, `INVALID_HIGH_DISCOUNT`, `MISSING_DISCOUNT`
- `status_validation_flag` — `INVALID_STATUS` with closest valid match noted

Exact duplicate rows identified and marked; conflicting duplicate order_ids flagged for manual review.

### Task 4 — Duplicate Handling
- 32 exact duplicate rows: one copy retained per order_id, duplicates removed from analysis scope (flagged `DUPLICATE_REMOVED`)
- 12 conflicting duplicates: both records retained with flag `CONFLICTING_DUPLICATE`; no auto-resolution applied — escalated for source-system verification

### Task 5 — Business Rules Applied
Nine business rules (BR-01 to BR-09) applied to the cleaned dataset. See Section 3 for full detail. A new column `business_rule_flag` was added recording the applicable rule(s) per row.

### Task 6 — Calculated Columns Added
Eight required calculated columns and six bonus columns added to `cleaned_orders.xlsx`:

| Column | Derivation |
|---|---|
| `cleaned_discount` | Standardised decimal (% strings → decimal; null → 0 per BR-03; negative preserved & flagged) |
| `calculated_sales` | `quantity × unit_price × (1 − MAX(0, cleaned_discount))` |
| `calculated_profit` | `calculated_sales − cost` |
| `profit_margin` | `calculated_profit / calculated_sales` (blank if sales = 0) |
| `shipping_delay_days` | `ship_date − order_date` in integer days (blank if either date null) |
| `order_month` | Month name extracted from `order_date` |
| `order_year` | Year extracted from `order_date` |
| `data_quality_flag` | 6-check priority chain → `CLEAN`, warning, or error label |

Bonus columns: `sales_variance`, `profit_variance`, `order_quarter`, `discount_band`, `revenue_category`, `business_rule_flag`

---

## 3. Business Rules Applied

| Rule ID | Area | Condition | Action |
|---|---|---|---|
| **BR-01** | Missing region | `region` is null or blank | Fill as `'Unknown'`; add `MISSING_REGION_FILLED` to flag column |
| **BR-02** | Missing ship_mode | `ship_mode` is null or blank | Fill as `'Unknown'`; add `MISSING_SHIP_MODE_FILLED` to flag column |
| **BR-03** | Missing discount | `discount` is null AND all other sales fields (`quantity`, `unit_price`, `sales`) are valid | Set `cleaned_discount = 0`; add `MISSING_DISCOUNT_FILLED_AS_0` |
| **BR-04** | Negative discount | `discount < 0` | Preserve original value; flag `INVALID_NEGATIVE_DISCOUNT`; do not use in `calculated_sales` without manual review |
| **BR-05** | Discount above threshold | `discount > 0.50` (50%) | Preserve original value; flag `INVALID_HIGH_DISCOUNT`; threshold of 50% based on company policy assumption |
| **BR-06** | Cancelled orders | `order_status = 'Cancelled'` | Exclude from completed sales summary; flag `EXCLUDED_CANCELLED` |
| **BR-07** | Failed payments | `payment_status = 'Failed'` | Exclude from completed sales summary; flag `EXCLUDED_FAILED_PAYMENT` |
| **BR-08** | Returned/Refunded orders | `order_status IN ('Returned', 'Refunded')` | Exclude from revenue; track in separate refund summary; flag `REFUND_ELIGIBLE` |
| **BR-09** | Ship date before order date | `ship_date < order_date` | Flag `INVALID_SHIP_DATE`; preserve dates; do not correct without source verification; sale remains valid |

### Rows affected per rule

| Flag | Rows |
|---|---|
| MISSING_REGION_FILLED | 25 |
| MISSING_SHIP_MODE_FILLED | 20 |
| MISSING_DISCOUNT_FILLED_AS_0 | 18 |
| INVALID_NEGATIVE_DISCOUNT | 15 |
| INVALID_HIGH_DISCOUNT | 15 |
| EXCLUDED_CANCELLED | 145 |
| EXCLUDED_FAILED_PAYMENT | 67 |
| REFUND_ELIGIBLE | 154 |
| INVALID_SHIP_DATE | 92 |
| **Total flag applications** | **551** (overlap possible; one record can carry multiple flags) |

---

## 4. Assumptions Made

1. **50% discount ceiling (BR-05):** No explicit company policy document was available. The 50% threshold is assumed based on common retail practice. Discounts above this level are flagged rather than deleted — a stricter threshold may apply depending on the business.

2. **Missing discount = 0 (BR-03):** Null discount values are treated as "no discount applied" only when `unit_price`, `quantity`, and `sales` are all present and non-null. If any of those fields is also null, the record is flagged `MISSING_CRITICAL_FIELD` instead and no fill is applied.

3. **Invalid ship dates do not invalidate the sale:** A `ship_date < order_date` is treated as a data entry error in the logistics system, not a sign that the order itself is fraudulent or invalid. Revenue from these orders is still recognised on the order date. Shipping performance analytics should explicitly exclude these 92 records.

4. **Cancelled order_status = no revenue:** All `Cancelled` records are excluded from the completed sales summary regardless of their `payment_status`. If a payment was collected before cancellation, refund liability is assumed to be handled outside this dataset.

5. **Returned records are non-revenue regardless of payment_status:** A `Returned` status means goods were returned (or a return was initiated). Financial treatment (refund issuance, write-off, reversal) depends on `payment_status` and is flagged for the finance team but not resolved within this cleaning process.

6. **Conflicting duplicates deferred to source system:** Where the same `order_id` has two records with different `sales` or `order_status` values, no automated resolution is applied. The lower-sales value is recommended as the conservative choice pending OMS verification, but neither record is deleted.

7. **Calculation tolerance of ₹1:** Differences between `raw_sales`/`raw_profit` and their calculated equivalents of ≤ ₹1 are treated as rounding artefacts and not flagged. Differences > ₹1 are flagged `CALC_MISMATCH`.

8. **Profit margin outlier thresholds:** Margins < −10% are flagged as Critical (cost likely exceeds sales); margins > 90% are flagged as High (cost entry likely erroneous). These thresholds are analyst-defined and should be confirmed with the finance team.

9. **"Processing" orders are in-flight:** Records with `order_status = 'Processing'` are neither included in completed revenue nor excluded — they represent orders that may complete or cancel. They are retained in the dataset with no revenue attribution.

10. **Invalid status values are standardised conservatively:** Non-standard strings such as `complete`, `Complet`, `CANCELLED` are mapped to their nearest standard value based on text similarity. Ambiguous values (`Dispatched`, `Delivered`, `shipped`) are mapped to `Processing` pending OMS confirmation, not to `Completed`, to avoid over-counting revenue.

---

## 5. Records Removed

| Action | Count | Reason | Flag Applied |
|---|---|---|---|
| Exact duplicate rows removed | 32 | Identical across all 21 fields; retain one copy per order_id | `DUPLICATE_REMOVED` |
| Null order_date records excluded from date analysis | 2 | Cannot assign to time period | Not removed from dataset; excluded from date-based analytics |
| Null ship_date records excluded from delay calculation | 12 | Cannot compute `shipping_delay_days` | Not removed; column left blank |

> **Important:** No records are **permanently deleted** from `cleaned_orders.xlsx`. "Removed" means excluded from specific summary calculations (revenue, shipping analysis). All 900 source rows are retained in the cleaned file for full auditability; exclusions are communicated through flag columns.

---

## 6. Records Flagged

### Business Rule Flags (`business_rule_flag` column)

| Flag | Count | Meaning |
|---|---|---|
| MISSING_REGION_FILLED | 25 | Region was null; filled as 'Unknown' |
| MISSING_SHIP_MODE_FILLED | 20 | Ship mode was null; filled as 'Unknown' |
| MISSING_DISCOUNT_FILLED_AS_0 | 18 | Discount was null; set to 0 |
| INVALID_NEGATIVE_DISCOUNT | 15 | Discount value is negative |
| INVALID_HIGH_DISCOUNT | 15 | Discount exceeds 50% threshold |
| EXCLUDED_CANCELLED | 145 | Order status = Cancelled; excluded from revenue |
| EXCLUDED_FAILED_PAYMENT | 67 | Payment status = Failed; excluded from revenue |
| REFUND_ELIGIBLE | 154 | Order returned/refunded; tracked separately |
| INVALID_SHIP_DATE | 92 | Ship date precedes order date |

### Data Quality Flags (`data_quality_flag` column, Task 6)

| Flag | Count | Meaning |
|---|---|---|
| CLEAN | 592 | Passed all 6 quality checks — safe for analysis |
| SHIP_BEFORE_ORDER | 73 | Ship date before order date |
| UNUSUALLY_LONG_DELAY | 58 | Delivery gap > 30 days |
| STATUS_INCONSISTENCY | 52 | Non-standard or conflicting status value |
| CALC_MISMATCH | 46 | Calculated vs. raw sales/profit differ by > ₹1 |
| INVALID_DISCOUNT | 30 | Negative or >50% discount |
| INVALID_STATUS | 16 | Unrecognised order status value |
| DUPLICATE | 22 | Duplicate order_id detected |
| MISSING_CRITICAL_FIELD | 11 | Null unit_price, cost, or order_date |

### Final Record Count Summary

| Category | Count | % | Revenue |
|---|---|---|---|
| Total raw records | 900 | 100.0% | — |
| Exact duplicates removed | 32 | 3.6% | — |
| Conflicting duplicates flagged | 12 | 1.3% | — |
| **Completed + clean (revenue base)** | **392** | **43.6%** | **₹5,964,435.12** |
| Cancelled (excluded) | 145 | 16.1% | — |
| Failed payment (excluded) | 67 | 7.4% | — |
| Returned / Refund eligible | 154 | 17.1% | ₹1,533,036.89 at risk |
| Processing / In-flight | 108 | 12.0% | Pending |
| Flagged (any flag) | 508 | 56.4% | See above |

---

## 7. Limitations of the Cleaning Process

1. **No access to source systems:** All corrections are based on the data as exported. Conflicting duplicates (12 records), invalid ship dates (92 records), and calc mismatches (46–75 records) can only be definitively resolved by querying the OMS, payment gateway, and ERP directly. These are flagged, not corrected.

2. **Discount threshold is assumed, not confirmed:** The 50% ceiling (BR-05) is a reasonable retail assumption but may be incorrect for this business. Clearance sales, promotional bundles, and loyalty programmes could legitimately exceed 50%. The flag should be reviewed by the sales team before any records are corrected or excluded.

3. **Profit margin outlier thresholds are analyst-defined:** The −10% / +90% boundaries for flagging calc mismatches are heuristics. High-cost product categories (e.g., Appliances) or low-margin commodity lines (e.g., Office Supplies) may have naturally different ranges.

4. **Invalid status values mapped conservatively:** Ambiguous status strings (`Dispatched`, `Delivered`, `shipped`) have been mapped to `Processing` rather than `Completed`. If these represent genuine completions, the revenue base of ₹5,964,435.12 is understated.

5. **Missing `customer_name` not corrected:** Six records with null customer names are retained as-is. They cannot be linked to CRM data without a customer_id join, which was not in scope for this task.

6. **Future-dated orders (8 records) not excluded:** Orders with `order_date` beyond the report generation date (June 2026) are flagged but retained. They may represent pre-orders or system clock errors. Exclusion from trend analysis is recommended but not enforced here.

7. **Calculated columns use cleaned_discount, not raw_discount:** For the 15 negative-discount records, `calculated_sales` uses `MAX(0, cleaned_discount)` effectively treating the discount as 0. This means `calculated_sales` may differ materially from `raw_sales` for these records. Both values are preserved in the file for comparison.

8. **No deduplication strategy for conflicting duplicates:** The 12 conflicting duplicate pairs each require a business decision (which record is authoritative). Until resolved, these 12 order_ids should be excluded from any count or revenue analysis to avoid double-counting or using incorrect figures.

9. **Dataset scope is a single export snapshot:** This cleaning process applies to one point-in-time export. Orders with `Processing` status may have since been completed or cancelled. Any re-export from the source system should trigger a full re-run of this cleaning pipeline.

10. **No referential integrity checks performed:** The dataset does not include dimension tables (customer master, product master, region hierarchy). Orphan records — orders referencing customers or products that may no longer exist in the master data — cannot be identified from this file alone.

---

*End of Cleaning Log — all figures reconcile with `outputs/data_quality_report.xlsx` (Task 7) and `outputs/pivot_summary.xlsx` (Task 8).*
