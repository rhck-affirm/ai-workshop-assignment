---
name: self-reported-income-monitor
description: >-
  Monitors UK self-reported vs Experian bureau income on POS underwriting
  decisions in Snowflake. Produces a DQ completion chart, headline income-level
  and overstatement-band charts, a monthly overstatement trend, and optional
  segment splits — using outlier-robust ratio metrics. Use when the user says
  "run the income monitor", "compare self-reported vs bureau income",
  "overstatement analysis", or "check income drift".
---

# Self-Reported vs Bureau Income Monitor

Compares self-reported income against Experian bureau income on UK POS
underwriting decisions, to quantify over/under-statement at a glance.

**All tables, fields, definitions, FX/dedup/join recipes, band cut-offs,
winsorization mechanics, NTA/RTA derivation, and baseline anchors live in
[`reference.md`](reference.md). Read it first.** This file is *how to work*.

## Core principles

- **Ratio is the spine.** `self_reported / bureau`, per checkout. It is robust,
  unit-free, segmentable, and portable to no-bureau markets. Levels and diffs
  are supporting context only.
- **Robust metrics by default.** Headline = median ratio + 5-band shares (both
  outlier-resistant). The mean is a *cross-check*, never the headline.
- **Winsorize the population average, never the overstatement measurement.**
  Two jobs, opposite treatment (full mechanics in `reference.md`):
  - Job A (bands, medians, segments): keep outliers — they *are* the signal.
  - Job B (any reported mean / a haircut factor): trim the top tail (P99).
  - Diagnostic: mean ≈ median ⇒ skew is outlier-driven; mean ≫ median ⇒ genuine
    right-skew (do not trim it away in Job A).
- **State winsorization once, up front, then stop repeating it.** Declare the
  P99 trim (and every other assumption) in the Assumptions block at the very
  top of the output; everywhere after that, just say "mean" — do not keep
  re-labelling it "winsorized mean" in prose, tables, or chart text.

## Coding standards

- **Checkout grain first, always.** Dedup to one row per `CHARGE_ARI` (latest
  decider run) before anything else — see the dedup recipe in `reference.md`.
  Build it once as a base CTE (`chk`) and reuse it for every step.
- **Compose canonical recipes, don't hand-roll full queries.** Use the
  fragments in `reference.md` (FX join, grain dedup `QUALIFY`, funnel `LEFT JOIN`,
  NTA/RTA window) and stitch them together; keep definitions in one place.
- **Usable subset = both incomes `IS NOT NULL` and `> 0`** (a `0` is a
  placeholder, not a wage, and breaks the ratio). DQ Step 0 is the only step
  that runs without income filters.
- **Reconcile counts.** Segment splits should sum back to the usable n; cart
  splits should sum to the funnel-matched subset. Report coverage; never silently
  drop rows.
- **Compute winsorization caps globally**, not per segment, so segments stay
  comparable.
- **Verify, don't assume, the grain.** This table is one row per *decider run*,
  not per checkout; non-matches against the funnel are structural, not a decline
  effect (see `reference.md`). State assumptions and check them in data.

## Charting conventions

- Render with matplotlib, save a PNG, and embed it in the response.
- **Never mix scales on one axis.** Levels (£) and shares (%) go on separate
  charts. The trend chart uses exactly two axes (% left, ratio right).
- Label values on bars; always show the relevant `n` (per chart / per segment)
  so small, noisy segments are visible.
- Segment splits = one **100%-stacked bar per dimension** with consistent band
  colours, so composition is comparable across segment values.
- **Always define the bands by their ratio ranges on any band chart** — in the
  axis labels, legend, or a caption — and state `income ratio = self-reported ÷
  bureau`. Ranges: strong under `<0.75`, mild under `0.75–0.95`, aligned
  `0.95–1.05`, mild over `1.05–1.25`, strong over `>1.25`.
- **Legends must stay readable and out of the data.** Place them outside the
  plotting area (e.g. below the chart) or in genuinely empty space; never let a
  legend overlap bars, points, or lines. Use multiple columns for many entries.

### Canonical colour scheme (always use these exact colours)

Mnemonic: **orange = self-reported / over-statement, blue = bureau /
under-statement, grey = neutral (aligned / combined).** Keep this identical
across every chart so colour always means the same thing.

| Use | Hex |
| --- | --- |
| Self-reported series | `#F58518` (orange) |
| Bureau series | `#4C78A8` (blue) |
| Combined / neutral | `#9AA0A6` (grey) |
| Band — strong under (`<0.75`) | `#1F4E79` (dark blue) |
| Band — mild under (`0.75–0.95`) | `#7FB2E5` (light blue) |
| Band — aligned (`0.95–1.05`) | `#BFBFBF` (grey) |
| Band — mild over (`1.05–1.25`) | `#F9B25A` (light orange) |
| Band — strong over (`>1.25`) | `#D4691A` (dark orange) |

Trend lines: `% overstating` → `#F58518`, `% understating` → `#4C78A8`,
`median ratio` (right axis) → `#5A5A6E` (dashed). Median-ratio dots → `#4C78A8`.

## Tone & framing

Be **neutral and descriptive**, not judgemental. Report what the data shows;
let the reader draw conclusions. Specifically:

- **Lead with the aggregate finding.** At baseline the two measures are *well
  aligned* in aggregate and across every segment (medians ~0.92–0.95, means
  ~1.00–1.06, income levels within a few %). State that first. The row-level
  dispersion is secondary detail, not the headline.
- **Present both tails symmetrically.** Always pair over- and under-statement
  (e.g. "52% understate, 39% overstate"). Never spotlight overstatement alone or
  call it "the risk" — understatement is, if anything, slightly more common.
- **Distinguish aggregate alignment from row-level agreement.** High dispersion
  with near-parity medians/means is a *coexisting* fact, not a contradiction or a
  failure. Say "agree well in aggregate; disperse at the individual level."
- **Describe trends factually and directionally correct.** A median ratio moving
  0.87 → 0.96 is *converging toward parity*, not "more over-reporting."
- **Avoid loaded / prescriptive words**: "unreliable", "weak", "don't trust",
  "real risk", "worrying", "worth watching", "should tighten". No
  recommendations unless the user explicitly asks for them.

## Output format

This is a **quick-glance, one-off read** — not a drift-detection report. So:

- **Start with an `Assumptions` block** (before any chart). List, concisely:
  - Dataset time span — the actual min/max `DECIDER_RUN_DATE` in the data — and
    the explicit statement that we **assume this window is representative**. Do
    **not** flag the latest month/quarter boundary as a data problem or "outage";
    it is just where the dataset ends.
  - Winsorization: means use a one-sided P99 trim on the ratio (Job B). After
    stating it here, refer to it simply as "mean" thereafter.
  - Usable subset = both incomes present and `> 0`; checkout grain.
  - Bureau income is a modeled Experian estimate (capped ~£10k/mo) — the
    operational benchmark, not ground truth.
- **End with a `TLDR` header** + a couple of plain bullet points summarising the
  headline read. The **first bullet must be the aggregate-alignment finding**
  (overall and per segment); later bullets can add the row-level dispersion
  (stated symmetrically) and the coverage caveat. Keep it neutral per Tone &
  framing. **Do not** append a drift table, baseline comparison, recommendations,
  or "flag before this monitor is trusted" language.

## Workflow

0. **Confirm assumptions first (gate — do not skip).** Before running the
   analysis, query just the dataset span (`MIN`/`MAX DECIDER_RUN_DATE`) and
   present the full `Assumptions` block to the user. Then **ask the user to
   confirm they agree** (or want changes) and **wait for their reply** before
   proceeding. Only run the charts once they've agreed.
1. **DQ.** On the deduped `chk` set with **no income filters**, compute
   completion (`IS NOT NULL`) rates for bureau, self-reported, and combined.
   Render a 3-bar chart (0–100%). The combined rate **is** the usable subset —
   state it; the rest cannot be compared.
2. **Headline — two charts.**
   - 2a Income levels (£/mo): median SRI, median BI, mean SRI, mean BI
     (means P99-trimmed per the assumptions). Title `Self-reported vs bureau income levels — UK POS`.
   - 2b Overstatement bands (%): the 5 ratio bands, summing to 100%.
     Title `Income reporting bands — UK POS`.
3. **Trend (monthly).** Group by `DATE_TRUNC('month', DECIDER_RUN_DATE)`. One
   chart: `% overstating` + `% understating` (left), `median ratio` (right).
4. **Segments (if requested).** Re-run the 2b band distribution per dimension:
   approved vs declined, cart-value buckets (funnel join), plus any
   user-supplied dimension (see Extensible segmentation).
5. **TLDR.** Close with the TLDR bullets (see Output format).

### Extensible segmentation

Segmentation is field-agnostic. If the user names a segment dimension and tells
you where the field lives (table + join key — usually `CHARGE_ARI` or
`USER_ARI`), join it onto the checkout base and re-run the **same** 2b band
split (and optionally median ratio) for it — no other change needed. Always
report per-segment `n` and join coverage.

**NTA vs RTA is parked for now** — the definition is not yet agreed, so do not
include it as a segment unless the user explicitly asks and supplies the
definition to use.

## Interpretation guide

Neutral readings only — describe, don't prescribe.

| Observation                          | Neutral reading                                                       |
| ------------------------------------ | -------------------------------------------------------------------- |
| medians/means within a few % of parity | Self-reported and bureau agree well in aggregate                   |
| median ≈ 1 but wide band spread      | Aggregate alignment with high row-level dispersion (the two coexist) |
| under-share ≈ over-share             | Dispersion is roughly symmetric; it nets out near parity            |
| median ratio < 1, mean ratio slightly > 1 | More rows understate; a thinner overstater tail lifts the mean |
| median ratio rising toward 1 over time | Converging toward parity                                          |
| segment band shares look alike       | Reporting behaviour is broad-based, not segment-specific           |
| mean ≈ median                        | Skew is outlier/data-entry driven, not systematic over-reporting   |
