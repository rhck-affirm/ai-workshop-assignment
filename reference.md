# Reference — Self-Reported vs Bureau Income Monitor (UK)

Context and canonical recipes for the income monitor. The `SKILL.md` covers
*how* to work; this file covers *what* to use (tables, fields, definitions,
canonical SQL fragments). Compose these fragments — do not hard-code full
end-to-end queries.

## Tables

| Role                     | Fully-qualified name                                       | Notes                                                |
| ------------------------ | --------------------------------------------------------- | ---------------------------------------------------- |
| Primary (POS)            | `PROD__UK.DBT_ANALYTICS.UNDERWRITING_DECIDERRUN_EVENT_POS` | Income + decision; base for everything               |
| FX rates                 | `PROD__UK.DBT_ANALYTICS.STG_BASE_EXCHANGE_RATES`           | Daily rates; stored X→USD only                       |
| Enrichment (cart value)  | `PROD__UK.DBT_ANALYTICS.CHECKOUT_FUNNEL_V5`                | Cart value, merchant, charge-off; join on `CHARGE_ARI` |
| Avoid                    | `UNDERWRITING_DECIDERRUN_FACT`                             | Transient, often empty                               |

**Warehouse:** `SHARED`

## Key fields

| Field | Meaning |
| ----- | ------- |
| `AFFIRM__USER__RAW_ANNUAL_INCOME_DOLLARS__V1` | Self-reported income — annual, **USD** |
| `EXPERIAN_UK__CREDIT_REPORT__GROSS_INCOME_MONTHLY_IN_POUNDS__V1` | Experian bureau income — monthly, **GBP** (modeled estimate, capped ~£10k/mo) |
| `DECISION_ACTION` | `'17'` = approved, `'16'` = declined |
| `DECIDER_RUN_DATE` | Decision date; used for time filtering + FX join |
| `DECIDER_RUN_CREATED_DT` | Decider-run timestamp; used to pick latest run per charge |
| `DECIDER_RUN_ARI` | **Table primary key** — one row per decider run |
| `CHARGE_ARI` | Checkout/loan key; NULL for `LOAN_MODIFICATION` |
| `DECIDER_NAME` | `LOAN_TERMS` (original) / `LOAN_MODIFICATION` (re-decision) |
| `USER_ARI` | User id; one user → many charges |

Notes on the bureau field: it is a **modeled** estimate (capped ~£10k/mo), not
raw CATO transaction income — treat it as the operational benchmark, not ground
truth. No separate CATO/BIM fields exist in this table.

## Grain & checkout dedup (apply to EVERY step)

Grain is **one row per decider run**, not per checkout. A `CHARGE_ARI` can have
several runs (re-decisions, ~98% within 1 hour = same checkout). Reduce to one
row per charge (latest run = final terms seen):

```sql
WHERE CHARGE_ARI IS NOT NULL          -- also drops LOAN_MODIFICATION (charge NULL there)
QUALIFY ROW_NUMBER() OVER (
  PARTITION BY CHARGE_ARI
  ORDER BY DECIDER_RUN_CREATED_DT DESC, DECIDER_RUN_ARI DESC
) = 1
```

This `chk` set is the base for all steps. `CHARGE_ARI` is the checkout key.

## FX rate join recipe

Rates are stored **X → USD** only; invert for USD → GBP.

```
FROM_CURRENCY_SYMBOL = 'GBP' AND TO_CURRENCY_SYMBOL = 'USD'
join:  DECIDER_RUN_DATE BETWEEN EFFECTIVE_FROM_DATE AND EFFECTIVE_TO_DATE
usd_to_gbp = 1.000 / AVERAGE_EXCHANGE_RATE
```

## Core definitions

```
bureau_monthly_gbp        = EXPERIAN_UK__CREDIT_REPORT__GROSS_INCOME_MONTHLY_IN_POUNDS__V1
self_reported_monthly_gbp = AFFIRM__USER__RAW_ANNUAL_INCOME_DOLLARS__V1 / 12.0 * usd_to_gbp
ratio                     = self_reported_monthly_gbp / NULLIF(bureau_monthly_gbp, 0)
diff_gbp                  = self_reported_monthly_gbp − bureau_monthly_gbp
```

**Direction convention (fixed — always self-reported relative to bureau):**

- Numerator / minuend = **self-reported (SRI)**; denominator / subtrahend =
  **bureau (BI)**. Bureau is the benchmark, so SRI is always measured *against*
  it. Never compute `BI/SRI` or `BI − SRI`.
- `ratio > 1` ⇒ self-reported **above** bureau = **overstating**.
- `ratio < 1` ⇒ self-reported **below** bureau = **understating**.
- `ratio = 1` ⇒ aligned. Same sign logic for `diff_gbp` (positive = overstating).
- Bands, charts, and labels all inherit this: "overstating" = SRI > BI.

Ratio is the spine of the product (robust, unit-free, segmentable, portable to
no-bureau markets). Levels/diffs are supporting context.

## Overstatement bands (5-band, symmetric around 1)

| Band                | Condition             |
| ------------------- | --------------------- |
| Strong understating | `ratio < 0.75`        |
| Mild understating   | `0.75 ≤ ratio < 0.95` |
| Aligned             | `0.95 ≤ ratio ≤ 1.05` |
| Mild overstating    | `1.05 < ratio ≤ 1.25` |
| Strong overstating  | `ratio > 1.25`        |

Boundaries are additive-symmetric around 1 (±0.05 aligned, ±0.25 strong); the
1.05 edge mirrors Affirm's existing income rule.

## Standard filters

- Always reduce to checkout grain first (dedup above).
- Usable subset: both income fields `IS NOT NULL` and `> 0`.
- DQ Step 0 is the only exception: checkout grain, but NO income filters (it
  measures coverage). Zeros are dropped because a `0` income is a missing /
  placeholder value, not a real wage, and breaks the ratio.

## Winsorization mechanics

Two jobs, opposite treatment:

- **Job A — measuring overstatement** (bands, segments, median): keep all
  outliers. Median and band shares are already robust; winsorizing would delete
  the overstaters you are studying.
- **Job B — a representative average** (trimmed-mean ratio / income levels):
  trim the top tail so a few data-entry errors don't drive the statistic.

Job B recipe: **winsorize the ratio at P99**, computed **globally** (not per
segment, so segments stay comparable):

```sql
LEAST(ratio, PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY ratio) OVER ())
```

For income-level trimmed means, cap each field at its own global P99.
Diagnostic: trimmed mean ≈ median → skew is outlier-driven; trimmed mean ≫
median → genuine right-skew (do not winsorize it away in Job A).

## Funnel enrichment join (cart value, merchant, outcomes)

Income lives ONLY in the decider table. For cart value / merchant / charge-off,
LEFT JOIN the funnel on `CHARGE_ARI`:

```sql
funnel AS (
  SELECT CHARGE_ARI, TOTAL_AMOUNT, MERCHANT_ARI, IS_CHARGED_OFF
  FROM PROD__UK.dbt_analytics.checkout_funnel_v5
  WHERE CHARGE_ARI IS NOT NULL          -- CHARGE_ARI is unique among non-null rows; no dedup needed
)
-- chk LEFT JOIN funnel USING (CHARGE_ARI)
```

- Cart value = `TOTAL_AMOUNT` (GBP). Buckets: `<£500`, `£500–1k`, `£1k+`.
- Funnel grain is one row per checkout (`CHECKOUT_ARI`); charge↔checkout is 1:1.
- Coverage ≈ 69% of decider charges; non-match is structural (separate pipeline),
  roughly even across approved/declined — **not** a decline effect. Use LEFT JOIN,
  keep all income rows, filter to matched rows for cart analysis, report coverage.
- Scope: only `classic` / `adaptive` installment loans run income underwriting;
  Affirm Go / pay-in-4 have no income and are correctly excluded by the base.

## NTA / RTA — PARKED (do not use by default)

NTA/RTA segmentation is **parked**: the definition is not yet agreed, so it is
**excluded** from the default analysis. Only include it if the user explicitly
asks and supplies the definition to use.

Why parked: there is no usable flag in the table
(`PRIOR_USER_CREDIT_APPLICATION_ARI`, `ROOT_…`, `USER_CREDIT_APPLICATION_ARI`
all NULL). A derivation from "prior approval" was considered but is unreliable —
it is left-censored (users approved before the window look like NTA) and "prior
approval" is only a proxy for the intended "prior successful payment", for which
no data exists in this table. Revisit once the business confirms the definition.

## Dataset time span

This is a **quick-glance, one-off** read. At the start of every run, report the
actual `MIN`/`MAX` `DECIDER_RUN_DATE` in the data and state that we **assume the
window is representative**. The dataset simply ends at its latest available
date — do **not** treat that boundary as an outage, gap, or reason to distrust
the result, and do not append drift/baseline-comparison commentary.

## Sanity-check ballpark (June 2026, checkout grain)

Rough expected values, for sanity-checking a run only (confirm yours land near
these; not a drift gate, not required output). Per-run results come from running
the skill.

- Usable subset n ≈ **213,630** (DQ combined completion ≈ 34.4%).
- Median ratio ≈ **0.93**; P99-trimmed mean ratio ≈ **1.02**.
- Overall bands ≈ strong-under 32% / mild-under 20% / aligned 9% /
  mild-over 14% / strong-over 25%.
