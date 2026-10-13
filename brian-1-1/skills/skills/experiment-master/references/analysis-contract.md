# Analysis Contract — Master ↔ analyses

Analyses are files the Master reads in the same conversation, not subagents.

## Request

```
{ capability, depth, purpose: ANALYSIS | PAB_CHECK, focus, metrics?, allowed_cuts?, target?,
  scope?,                                           # every mode when a scope was requested (below)
  prior_findings[], query_budget,
  retention_days_basis?: DEFAULT | USER_REQUESTED,   # classic retention only
  horizon_basis?: DEFAULT | USER_REQUESTED,          # LTV & LTDays only
  context: { exp_id, version, windows, design_status, explicit_control, comparison_type,
             comparisons: [{pair_id, reference, comparator}],
             observed_active_span?: {first_active_day, last_active_day},   # install-cohort capabilities
                                         # (classic retention, LTV & LTDays) only: the unclipped
                                         # reporting span (analyses/classic-retention.md, ltv-ltdays.md)
             stats_config: {tails = 2, confidence_level = 0.95, alpha = 0.05,
                            power_effect_size_basis = CONFIG_REQUIRED},
             pab_config: {pab_confidence_floor = CONFIG_REQUIRED} } }
```

`metrics?` names the metrics to compute; engagement needs none: it runs its Clear-first metrics
(`analyses/engagement.md`), with `clear` as the implicit primary action. Classic retention receives one
`classic_retention_dX` per requested retention day. LTV & LTDays receives one metric per requested
family and horizon (`ltv_dX`, `ad_ltv_dX`, `iap_ltv_dX`, `ltdays_dX`). Rolling LTV receives
`rolling_ltv_d7`, `iap_rolling_ltv_d7`, `ad_rolling_ltv_d7`. Economy receives its six metrics for each
requested `cash_type` (`analyses/economy.md`). `query_budget` is configurable.
The `stats_config` defaults are the initial BRIAN statistical configuration (two-sided paired t-test) and
are overridable per request. Power has no default basis. No further statistical policy is inferred from
these defaults. `target?` (PAB_CHECK only) names the finding, pair and cut to check.
`pab_config.pab_confidence_floor` has no default: it is CONFIG_REQUIRED until configured, and the
analysis file defines how it is used.

**Scope (every mode) and focus (PERIOD_IMPACT_ANALYSIS only).** `scope?` is the request's canonical
scope, `[{dimension, values[]}]` — values within a dimension are alternatives, dimensions combine —
resolved by the Master (`protocols/context-resolution.md` step 2b) for the scope lens being evaluated.
It is set in FULL_EXPERIMENT_READ, TARGETED_DIAGNOSIS and PERIOD_IMPACT_ANALYSIS whenever the request
has a scope, and is absent (overall) when it has none. A platform selects units and is never a `scope`
entry. Every capability evaluated for one unit and lens receives the identical canonical scope. The
capability evaluates its metrics inside the scope as a fixed cut cell under its own existing cut rules
(grain arm × day × cell, the cell's own denominator per its analysis file, `UNOBSERVED` for an absent
cell); its Stage 2 cuts run inside that cell. A requested dimension that its analysis file does not
support as a cut for a metric (for a DAU-normalized metric, a cut without a DAU split) makes that metric
`UNSUPPORTED_FOR_SCOPE`; scoped values with no rows for the unit make the scope `UNAVAILABLE`. A
dimension is never dropped and a metric is never computed at a broader scope instead. In PERIOD mode
`focus` is `TRIGGER` (the capability of the user's trigger metric) or `CROSS_CAPABILITY`; the
analysis is identical either way, the capability never receives another capability's results, and it
performs no cross-capability correlation (the Master's, `period-evidence.md`).

## Response

```
{ status: OK | PARTIAL | BLOCKED | NO_COMPARISON,
  reason?,                                          # SINGLE_ARM when NO_COMPARISON
  exp_id, version,                                  # echoed from the request
  scope?: {canonical, status: SUPPORTED | UNSUPPORTED | UNAVAILABLE,          # echoed when sent
           dimension_status: [{dimension, status, reason?}],
           metric_status: [{metric, status: SUPPORTED | UNSUPPORTED_FOR_SCOPE}]},
  definition_lock: [{metric, numerator, denominator, denominator_table, grain, table,
                     definition_status?}],        # one per metric; table = the numerator's source
  primary_metric?, supporting_diagnostics?: [metric],   # engagement only: Clear / DAU; the diagnostics
                                                        # returned (unavailable ones appear in data_conditions)
  explicit_control, comparison_type,                # echoed from the request
  comparison_results: [{
    exp_id, version, metric, pair_id, reference, comparator, comparison_label,
    scope?,                                         # the canonical scope it was computed in
    cut?: [{dimension, value}],                     # a breakdown cell inside the scope (below);
                                                    # absent = the whole canonical scope
    retention_day?,                                 # classic retention only: X of classic_retention_dX
    horizon_day?,                                   # LTV & LTDays only: X of the metric
    reference_value, comparator_value,
    exposure: {reference, comparator},              # the denominator and its source table, per arm
    pairing_unit?: CALENDAR_DAY | INSTALL_COHORT,   # absent = CALENDAR_DAY (Paired test, below)
    n_paired_days,                                  # CALENDAR_DAY
    n_paired_cohorts?,                              # INSTALL_COHORT, in place of n_paired_days
    delta_abs,                                      # comparator − reference
    delta_pct_of_reference,                         # null + reason when reference = 0 (Economy net:
                                                    # also when ≤ 0, analyses/economy.md E1)
    stat: {
      method: "paired_t_daily" | "paired_t_install_cohort", tails: 2,
      n_pairs, mean_paired_diff, sd_paired_diff, t, df, p_value,
      confidence: {level, ci_low, ci_high},
      power: {value, alpha, effect_size_basis},     # value null while basis = CONFIG_REQUIRED
      status: COMPUTED | NOT_COMPUTABLE_DF_LT_1 | NOT_COMPUTABLE_ZERO_VARIANCE | CONFIG_REQUIRED },
    tbd: {min_sample: "TBD", srm: "TBD", multiple_testing: "TBD", peeking: "TBD"},
    n_comparisons_in_experiment,
    observations?: [{x, reference_value, comparator_value, diff}] }],   # the paired units the test
                                                    # used; visualization only (§ Observations)
  descriptive?: [{arm, metric, value, exposure}],   # populated when status = NO_COMPARISON
  cuts, blocked, freshness,
  excluded_cohorts?: [{variant, install_dates, installs,           # install-cohort capabilities only
                        reason: OUTSIDE_REQUESTED_PERIOD | OUTSIDE_STABLE_WINDOW}],
  d0_installs_check?: [{variant, cut?, d0_users, installs, difference}],  # classic retention only;
                                                    # descriptive, never used in a result
  windows: {analysis_window: {start, end},          # echoed from the request context
            mature_analysis_window: [{measure, start, end,
                                      basis: CUTOFF_APPLIED | NO_CUTOFF_APPLIED}]},
  data_conditions: [{exp_id, version,
                     scope: VARIANT_DAY | METRIC_DAY | CUT_CELL | METRIC
                            | VARIANT_COHORT | VARIANT_COHORT_DAY,         # the last two: cohort grain
                     condition: MISSING | UNOBSERVED | NOT_IN_TABLE | ABSENT_ROW_LATER_DIFF,
                     metric?, variant?, day?, install_date?, retention_day?, horizon_day?,
                     cut?: [{dimension, value}],
                     effect: EXCLUDED_FROM_PAIR | NOT_READ_AS_ZERO | NOT_RUN
                             | READ_AS_ZERO_LATER_DIFF, probe?}],
  pab?: [{exp_id, version, metric, pair_id, cut?: [{dimension, value}],      # PAB_CHECK only
          status: COMPUTED | NOT_APPLICABLE | NOT_IN_TABLE | CONFIG_REQUIRED | TBD,
          reason?, pab_value?, observed_movement?, adjusted_movement?,
          observations?: [{arm, series: [{x, numerator?, denominator?, value?}]}],   # daily PAB
                                                    # rows by group (§ PAB observations)
          confidence?: {value?, floor?,
                        outcome: BELOW_FLOOR | ALWAYS_USE | UNSPECIFIED_BAND | NOT_DETERMINED}}],
  queries_executed,                                 # self-reported; see Obligations
  not_tested?: [{step, reason: OUT_OF_BUDGET}],
  followups: [{capability, finding_ref, reason, expected_evidence,
               scope: {pair_ids[], cuts[], stage: STAGE_2 | STAGE_3_PLUS},
               purpose: ANALYSIS | PAB_CHECK}],   # suggestions only
  findings: [Finding] }

Finding = { ...protocol Finding fields,
            exp_id, version,
            scope?,                                 # the canonical scope, when the request had one
            comparison: {metric, pair_id, comparison_type, reference, comparator, explicit_control},
            causal_scope: RELATIVE_TO_CONTROL | RELATIVE_TO_REFERENCE_VARIANT }
```

## Paired test

The pairing unit is the calendar day unless the analysis file declares another; the only other unit
is the install-date cohort (`pairing_unit = INSTALL_COHORT`, `analyses/classic-retention.md` and
`analyses/ltv-ltdays.md`), where
each paired unit is one install cohort and `n_paired_cohorts` replaces `n_paired_days`. Observations
are per arm and unit; for a ratio metric the observation is that arm's ratio of the numerator to its
denominator on that unit (the denominator and its source table are named in `definition_lock`).
`d_i = comparator_i − reference_i` on each paired unit (day or cohort); the arithmetic below is the
same for both units: `t = mean(d) / (sd(d) / √n)`, `df = n − 1`;
two-sided p-value from the t distribution with `df`; the confidence interval uses
`confidence_level`. `alpha` is carried in the configuration and used where a computation requires
it (power, once its basis is set); this contract defines no significance verdict and no
meaningfulness verdict. The Master interprets the returned statistics for meaningfulness
(`master-state.md` § Results and meaningfulness); analyses return them unchanged and never classify.
`n < 2` → `NOT_COMPUTABLE_DF_LT_1`; `sd = 0` → `NOT_COMPUTABLE_ZERO_VARIANCE`. These are
arithmetic computability conditions, not statistical policy or sample-size rules. The `tbd` fields
are literal markers: analyses never apply or fill them, and no multiple-testing adjustment is
applied (`n_comparisons_in_experiment` — the number of comparisons carried for this experiment-version,
each metric, for example each Engagement metric (Clear / DAU, Clears / User, Clear Users / DAU and each
supporting diagnostic), counted separately — is carried only).

## Breakdown cells

A `comparison_results[]` entry with `cut` is a breakdown cell: one value per dimension inside the
canonical scope, in the same form as `data_conditions.cut`, with the same fields as a whole-scope entry.
Cell-level statistical entries may be returned only when the capability's own analysis file explicitly
states that the capability produces cell-level statistics; that file then applies the paired test above
to the cell's observations on its pairing unit unchanged, and no other method is introduced. This
contract grants no capability that permission by itself. The existing Revenue, Sessions, Engagement and
Rolling Retention analysis files state no such behaviour (and `analyses/rolling-ltv.md` and
`analyses/economy.md` state that they produce none), so they gain no cell-statistic behaviour from
this section and continue to return breakdown cells descriptively only. `analyses/classic-retention.md`
and `analyses/ltv-ltdays.md` state it, each for its own supported cut dimensions only. Descriptive cell
movement (movement, share, weight) stays in `findings` and `cuts` as the analysis file defines. A cell
entry never replaces or alters the whole-scope entry for the same metric and pair.

## Observations

`observations?` exposes, for a `comparison_results[]` entry whose paired test ran, the paired units that
test already used: one entry per paired day (`pairing_unit = CALENDAR_DAY`, `x` = the activity date) or
per paired install cohort (`INSTALL_COHORT`, `x` = the install date), with that unit's
`reference_value`, `comparator_value` and `diff` (`comparator_value − reference_value`, the `d_i` of
§ Paired test), in `x` order. It adds no query, statistic, rule or classification:

- It lists exactly the units the statistic was computed on, with the same values; nothing is
  recomputed, re-read or filled in.
- A unit excluded from the pair (`MISSING`, `UNOBSERVED`, immature, outside a window) is not listed and
  is never given a value; its reason stays in `data_conditions[]`, `excluded_cohorts[]` and
  `windows.mature_analysis_window`.
- Only an entry with a paired test carries it: a whole-scope entry, and a breakdown cell only where the
  capability's analysis file states cell-level statistics. A descriptive breakdown cell, a
  `descriptive[]` entry (`NO_COMPARISON`) and a PAB value carry none.
- Every entry whose paired test ran returns it: `stat.status` is `COMPUTED` or a `NOT_COMPUTABLE_*`
  status with at least one paired unit, and the field lists exactly `stat.n_pairs` units
  (`n_paired_days` or `n_paired_cohorts`). An entry whose test did not run (`CONFIG_REQUIRED`, or no
  paired unit) returns none.
- The field stays optional in the schema: an absent field means the units were not returned, never
  zero and never "no movement". It changes no other field of the entry. The Master uses it for visualization only
  (`references/visual-evidence.md`), never for meaningfulness, correlation or maturity.

## PAB observations

`pab[].observations?` exposes the daily rows a PAB check already reads, one entry per compared arm: the
pair's reference arm and its comparator arm. With several variants, each pair's `pab[]` entry carries
its own two arms, so the reference arm's series repeats unchanged in each pair. It adds no query,
statistic, rule, threshold or classification:

- **Statement grain.** A PAB check's statements are grouped by `variant` and `date_field`, plus the
  metric's own selectors and the targeted cut. The selectors are, for example, `rev_type`, `action_type`
  and `puzzle_type`, or `cash_type`, `earn_type` and `category`. The tables, filters and number of
  statements are those of the analysis file (`capability-registry.md` PAB follow-up statements). No
  statement is added.
- **Series items.** Each series item is one `date_field` day that has rows for that arm:
  - `numerator`: the metric's PAB numerator summed over the arm's rows of that day;
  - `denominator`: the metric's PAB denominator summed over the arm's rows of that day, from the
    denominator table the analysis file names;
  - `value`: `numerator` ÷ `denominator`, present only when both are defined and the denominator is
    > 0.
  A day with rows in only one of the two tables keeps the side it has, with no `value`. A day with no
  rows is not listed and is never filled in. Missing-row handling stays TBD, so no reason is attached
  to an absent day.
- **Pooled value unchanged.** The pooled PAB value is computed exactly as before. Per arm, it is the sum
  of all listed numerators divided by the sum of all listed denominators, which equals the existing sum
  over the window's rows. It is never an average of daily values. `pab_value`, `observed_movement`,
  `adjusted_movement` and `confidence` are computed from the pooled values, exactly as the analysis
  file defines.
- **Groups and availability.** Arms are never merged into one series. Observations are returned for
  every `pab[]` entry whose PAB ratios were computed. An absent field means the daily rows were not
  returned (pooled-only), never zero.
- **No user type.** The PAB tables carry no `user_type`, so these observations do not make a PAB
  result specific to new or old users.

## Data conditions and PAB

- `data_conditions[]` carries the conditions an analysis's approved methodology defines, at the scope
  where they occur. `MISSING`: a variant-day with no row where the methodology requires one — a
  descriptive condition, excluded from the pairing and reported. `UNOBSERVED`: a cut-level cell with no
  row on a variant-day that has other rows — never read as zero; any further handling is the analysis
  file's. `NOT_IN_TABLE`: only an absent pinned row (`METRIC_DAY`) or an absent named metric
  (`METRIC`), with the probe quoted. `NOT_IN_TABLE` for an absent `(exp_id, version)` stays the
  Master's coverage result, and an unobserved cut-level cell is never `NOT_IN_TABLE`. The analysis
  never fills, estimates or zeroes any of them.
- Cohort grain (`analyses/classic-retention.md` and `analyses/ltv-ltdays.md` only): `VARIANT_COHORT` is a
  variant's install cohort, `VARIANT_COHORT_DAY` that cohort on one retention day or horizon.
  **Capability-level exception:** `ABSENT_ROW_LATER_DIFF` / `READ_AS_ZERO_LATER_DIFF` is the one case
  where an absent row is read as zero — a mature cohort's day-X row is absent while a later-day row of
  the same cohort, variant and cell exists. It applies only where that analysis file defines it and
  nowhere else (not in LTV & LTDays); every other absent row stays `MISSING` or `UNOBSERVED`.
- Rolling LTV (`analyses/rolling-ltv.md` only) has two capability-level exceptions, defined there and
  applying nowhere else: a stored NULL `d7_rev` in a row whose forward users are 0 is read as zero
  revenue, and a stored 0 Independent `d7_rev` with NULL forward users (a known source defect) is
  `MISSING`, never an observed zero.
- Economy (`analyses/economy.md` only) has three capability-level exceptions, defined there and applying
  nowhere else: `spend_per_dau` keeps the source's sign (negative for most currencies) and its relative
  change uses the signed reference (E3); `net_per_dau` with a reference ≤ 0 returns
  `delta_pct_of_reference` null with reason `REFERENCE_NON_POSITIVE` (E1); and PAB net is derived as PAB
  earn + PAB spend (E2).
- `pab[]` appears only in the response to a `PAB_CHECK` request and carries the PAB result or status
  for the targeted finding. `adjusted_movement` is returned only where the analysis file defines the
  adjustment; this contract defines no generic PAB adjustment. Confidence information is carried as
  returned: its calculation is TBD and its floor is configurable, as the analysis file states, and the
  contract defines no threshold.

## Obligations

- Compute exactly the `comparisons[]` received; **never re-derive the reference arm** and never
  add pairs.
- Every request, response, comparison result and finding carries both `exp_id` and `version`, and every
  query and result is bounded by both; versions are never pooled, including on transition days. A
  request without both does not run; the Master reports the condition.
- A response may carry several metrics (retention D1 and D3, classic retention days, LTV & LTDays
  families and horizons, Rolling LTV metrics, Economy metrics per currency, or the Clear and
  supporting-diagnostic engagement metrics). Each has its own `definition_lock` entry and its own
  `comparison_results`; metrics are never combined.
- `design_status = NO_COMPARISON` → return `status: NO_COMPARISON`, `comparison_results: []`,
  `reason: SINGLE_ARM` and `descriptive[]`. This is not BLOCKED.
- `design_status = UNRESOLVED` → the analysis does not run; the Master reports the condition.
- `query_budget` is a hard stop: on reaching it, stop querying, return `status: PARTIAL` and list each
  unexecuted step in `not_tested` as `OUT_OF_BUDGET`.
- `observations` repeats values already computed for the paired test; returning them issues no query
  and spends no `query_budget`. Every entry whose paired test ran returns them (§ Observations).
- `pab[].observations` repeats the daily rows the PAB statements already read; returning them adds no
  statement and spends no extra `query_budget` (§ PAB observations).
- `queries_executed` is the analysis's own count. The Master's spend accounting uses the count of query
  statements observed being executed where available, and records any discrepancy
  (`references/master-state.md`).
- `followups` are suggestions: an analysis never executes one. Each carries `capability`, `finding_ref`
  (the triggering finding), `reason`, `expected_evidence` and a `scope` from which the Master can compute
  its budget cost; the Master decides whether to run it. Stages are defined in `master-state.md`.
- Always return `windows`. `mature_analysis_window` has one entry per measure the analysis reports: the
  span of `analysis_window` on which that measure is matured. `basis` is `CUTOFF_APPLIED` when the
  analysis dropped unmatured days (for example retention) and `NO_CUTOFF_APPLIED` when it applied none.
  For classic retention and LTV & LTDays the window and its matured span are install dates (cohorts),
  not activity days.
  `freshness` is not used for maturity. `requested_period` and `experiment_active_window` are not
  analysis outputs; they come from the request and discovery (`references/period-evidence.md`).
- A `PAB_CHECK` request runs the analysis file's PAB methodology for the targeted finding only: it does
  not add or recompute comparisons, and it returns `pab[]`.
- Use the supplied `comparison_label`; wording per `references/output-format.md`.
- Every figure comes from a bounded query run this conversation; a BLOCKED note follows the
  protocol grammar and quotes its probe.
