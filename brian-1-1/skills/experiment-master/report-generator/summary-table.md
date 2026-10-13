# Results tables

The results tables of the templates (Template A's supporting and primary metric outcomes; the movement
rows of B and D), built from report records (`adapter.md`). One row per result; every result appears
exactly once. Rows are ordered by capability in registry order, then by metric. Breakdown cells are
shown under their parent, never as summary rows.

## Comparisons

- **One block per comparison** (`pair_id`), each with the same columns. With several variants there
  is one block per pair, e.g. "var1 vs control" and "var2 vs control".
- **Labels:** arms are named as returned. A reference variant is never called control
  (`output-format.md` § Comparison language).
- **Units:** units are never pooled, averaged or combined, across variants or across versions.

## Columns

| Column | Source | When unavailable |
|---|---|---|
| Capability / metric | `producer.capability`, `metric.label` | — |
| Reference value | `values.reference_value` | its status, e.g. "not returned" |
| Comparator value | `values.comparator_value` | its status |
| Δ (signed) | `values.delta_abs`; sign kept (Economy spend stays negative) | its status |
| Relative change | `GENERAL`: `values.delta_pct` (%). `RETENTION`: `delta_abs` in bps, per `output-format.md` | `NOT_COMPUTABLE` with the returned reason (e.g. reference 0, Economy `REFERENCE_NON_POSITIVE`) |
| 95% CI | `statistics.interval`, exactly as returned | "no interval returned" with the statistics status (e.g. zero variance) |
| p-value | `statistics.p_value`, as returned; a separate significance statement, never worded as meaningfulness | "not computable" with the status |
| Meaningfulness | `meaningfulness`, with threshold and source | its classification and reason (`NOT_CLASSIFIED`, `NOT_APPLICABLE`, …) |
| Power | `statistics.power` | "N/A — not configured" while the basis is `CONFIG_REQUIRED` |
| Reliability Score | — | "N/A — not defined" (no approved definition) |
| PAB movement | `pab.pab_value` | "PAB not run", "PAB not supported for this capability", or the `pab[]` status |
| PAB confidence | `pab.confidence`, as returned; never converted | the `pab[]` status, e.g. `CONFIG_REQUIRED` |
| PAB-adjusted movement | `pab.adjusted_movement`, only where returned | "not returned" |

The point estimate and the supplied 95% CI come before the meaningfulness classification. A
significance statement stays separate from meaningfulness (`output-format.md` § Results and
meaningfulness). This table shows no "Confidence (1−p)" column; only the Detailed Read table below has a
Confidence column.

## Non-comparison rows

- **`DESCRIPTIVE`:** each arm's value as returned. No delta, interval, p-value, direction or
  classification (`NOT_APPLICABLE`).
- **`NOT_RUN`:** the state and reason (e.g. `NOT_TESTED: OUT_OF_BUDGET`, `NOT_IN_TABLE` with its
  probe). No value cells.

## New and old users

User-type results exist today only as descriptive cut findings, and only for capabilities whose
analysis files support a `user_type` cut.
- **Separate rows:** New and Old cells appear in their own rows under the parent result, with movement
  as returned and "no cell-level statistic".
- **Combined adjusted row:** shown as "unresolved". Showing a value would need all of these:
  - an approved weight definition and unit-comparability basis
    (`experiment-contracts/references/compatibility.md` § Approved definitions; none exists);
  - producer-supplied component results;
  - an Old-user PAB-adjusted movement.

  The PAB tables carry no `user_type`, so PAB results never become user-type specific.
- **No defaults:** no weight, default or combined figure is ever computed.

## Detailed Read table

Used only in a Detailed Read (`template-population.md` § Detailed Read). Columns, exactly:

`Metrics | Control | Var | Delta | PAB Adjusted Delta | Confidence | PAB`

- **Priority rows first.**
  - LTV: each LTV & LTDays or Rolling LTV result in its own row, or its state when not run.
  - `Revenue / DAU excl. xpromo`, labelled "net/gross not established" and never called net revenue.
  - DAU: "not available — no registered capability returns a DAU comparison".
  Then every other result once, in registry order.
- **Header labels.** The `Control` header is used only when the reference arm is the explicit control;
  otherwise it reads `Reference (<arm>)`. There is one block per pair, with `Var (<arm>)`, so
  variants stay distinct.
- **Delta:** `GENERAL` → `delta_pct_of_reference` in %; `RETENTION` → `delta_abs` × 10,000 in bps, as
  returned.
- **PAB Adjusted Delta:** `adjusted_movement` only when returned, with "unit not stated". Otherwise
  the status: "PAB not run", "PAB not supported for this capability" or "not returned".
- **Confidence:** `(1 − stat.p_value) × 100`, from the returned p-value only, never from
  `stat.confidence` (that is the CI). It is shown as a percentage, never a decimal, with at most one
  decimal place and no trailing zeros (`48.4%`, `95%`, `99.9%`, `0%`). Above 99.9% it is shown as
  `>99.9%`, never `100%`. With no valid p-value (missing, not a number, or outside 0–1) no value is
  shown: "not computable" with the statistics status, as for the p-value. The confidence interval is
  never placed in this column.
- **PAB:** `pab_value` and the PAB status or confidence outcome, as returned; otherwise the status.
- **Under the table, per result:** the CI on its own labelled line, `95% CI of the mean paired
  difference (<Var> − <Control>): [low, high]`, followed by its unit when known and "unit not returned"
  otherwise, or "no interval returned" with the statistics status. Absolute CI bounds are never placed
  in the Confidence cell or beside a percentage Delta. Then the meaningfulness classification and the
  p-value, stated separately (`output-format.md` § Results and meaningfulness).
- **Caveat, once under the table:** "Confidence = 1 − p-value is a reporting convention, not the
  probability that a variant is better."
