# Adapter — AnalysisResponse + ResultEntry → report record

`AnalysisResponse` (`references/analysis-contract.md`) and `ResultEntry` (`references/master-state.md`
§ Results and meaningfulness) are the producer structures. The adapter reads them, field by field,
into a report record that uses the field names and availability statuses of
`experiment-contracts/references/capability-result.md`. A field is filled only when its source exists.
Nothing is computed.

## Conformance

The report record is a **partial** CapabilityResult. Two required contract fields have no producer
source, so a report record is never presented or validated as a conforming CapabilityResult:

| Gap | CapabilityResult field | Why it cannot be filled | Handling |
|---|---|---|---|
| M1 | `metric.unit` (required string) | `definition_lock` returns numerator, denominator and tables, not a unit | left empty and marked `NOT_RETURNED`. Display units come only from existing rules: `GENERAL` relative %, `RETENTION` bps, Economy "in-game units" (`output-format.md`). |
| M2 | `pab.window`, `pab.pooled`, `pab.movement_unit` | `pab[]` returns no window, no per-group pooled values and no unit | `NOT_RETURNED` (window, pooled) and `UNRESOLVED` (unit). Economy's PAB window and per-group values stated in prose are not parsed. |

## Field mapping

| Report field | Source | Rule |
|---|---|---|
| `result_id` | `ResultEntry.result_id` | copied |
| `result_kind` | `ResultEntry.state` | `COMPUTED`, `INSUFFICIENT_DATA` → `COMPARISON`; `NO_COMPARISON` → `DESCRIPTIVE`; `NOT_TESTED`, `NOT_RUN`, `NOT_IN_TABLE`, `UNSUPPORTED_FOR_SCOPE`, `NO_SCOPED_EXPOSURE` → `NOT_RUN` |
| `state` | `ResultEntry.state` | copied |
| `not_run.status`, `not_run.reason` | `ResultEntry.state`, `ResultEntry.state_ref` (or the `not_tested[].reason`) | copied; `NOT_RUN` kind only |
| `producer.capability` | `ResultEntry.key.capability` | copied |
| `producer.component` | — | `experiment-master` |
| `producer.method_ref` | registry "Analysis file" column for the capability | copied |
| `producer.adapter` | — | `experiment-master/report-generator/adapter.md` |
| `experiment.exp_id`, `experiment.version` | `ResultEntry.key` | copied; versions never pooled |
| `experiment.name` | `MasterState.context` experiment name | copied when present |
| `comparison.pair_id` | `ResultEntry.key.pair_id` | copied |
| `comparison.reference.arm`, `comparison.comparator.arm` | `comparison_results[].reference`, `.comparator` | copied; the label is the arm name as returned, and a reference variant is never labelled control |
| `comparison.comparison_type` | `AnalysisResponse.comparison_type` | copied |
| `comparison.causal_scope` | request context `causal_scope` | copied |
| `descriptive[]` | `AnalysisResponse.descriptive[]` (`arm`, `value`, `exposure`) | copied as `AVAILABLE` values; `DESCRIPTIVE` kind only |
| `metric.id` | `ResultEntry.key.metric` | copied |
| `metric.label` | `definition_lock[].metric` | copied; no friendlier name is invented |
| `metric.definition_ref` | `definition_lock` entry of the metric, with the `request_id` | copied |
| `metric.definition_status` | `definition_lock[].definition_status` | copied when present |
| `metric.unit` | — | gap M1 |
| `metric.metric_class` | `ResultEntry.meaningfulness.metric_class` | copied |
| `metric.sign_convention` | capability | `STORED_SIGN` for every Economy metric (spend and net can be negative); `NON_NEGATIVE` for the others |
| `metric.intended_direction` | — | `UNRESOLVED` (no stated or confirmed intent in this phase) |
| `population.scope`, `population.cut` | `ResultEntry.key.scope`, `.cut` | copied |
| `population.user_type` | scope or cut | `NEW` or `OLD` when a single `user_type` value is in the scope or cut; `ALL` when the scope has no `user_type` filter; otherwise `UNSPECIFIED` |
| `population.source` | — | `CANONICAL` |
| `window.analysis`, `window.mature` | `AnalysisResponse.windows` | copied |
| `window.date_basis` | registry PERIOD column date axis | copied (`INSTALL_DATE` for Classic Retention and LTV & LTDays; `ACTIVITY_DATE` otherwise) |
| `window.observation_unit` | `comparison_results[].pairing_unit` | copied; absent means `CALENDAR_DAY` (contract default) |
| `values.reference_value`, `.comparator_value`, `.delta_abs` | `comparison_results[]` | `AVAILABLE` as returned, sign kept |
| `values.delta_pct` | `delta_pct_of_reference` | `AVAILABLE` when returned; `NOT_COMPUTABLE` with the returned reason when null (e.g. `REFERENCE_ZERO`, `REFERENCE_NON_POSITIVE`) |
| `statistics.method`, `.status`, `.n_pairs` | `stat.method`, `stat.status`, `stat.n_pairs` | copied |
| `statistics.mean_paired_diff`, `.p_value` | `stat` | `AVAILABLE` when returned; `NOT_COMPUTABLE` when `stat.status` is a `NOT_COMPUTABLE_*` status |
| `statistics.interval` | `stat.confidence` (`level`, `ci_low`, `ci_high`) | `AVAILABLE` only when returned; never computed |
| `statistics.power` | `stat.power.value` | `AVAILABLE` when not null; `CONFIG_REQUIRED` while `effect_size_basis` is `CONFIG_REQUIRED` |
| `observations` | `comparison_results[].observations` | `AVAILABLE` as returned; `NOT_RETURNED` when absent |
| `pab.run` | `pab[]` entry for the metric, pair and cut | `RUN` when present; `NOT_SUPPORTED` for Classic Retention, LTV & LTDays and Rolling LTV; `NOT_RUN` otherwise |
| `pab.result_ref`, `pab.status` | the `PAB_CHECK` `request_id`; `pab[].status` | copied when `RUN` |
| `pab.pab_value`, `.observed_movement`, `.adjusted_movement` | `pab[]` | `AVAILABLE` when returned; `NOT_RETURNED` when absent; `NOT_RUN` when no PAB check ran |
| `pab.confidence` | `pab[].confidence` | `AVAILABLE` (value, floor, outcome) when returned; otherwise the `pab[]` status (`CONFIG_REQUIRED`, `TBD`); never converted |
| `pab.observations` | `pab[].observations` | each arm's series copied with `{x, value}` for days that have a `value`; numerator-only or denominator-only days are not plotted and keep their source entry; `NOT_RETURNED` when absent |
| `pab.daily_trend_available` | `pab.observations` | true only when both compared arms have at least one plotted day |
| `pab.excluded_days` | — | `NOT_RETURNED` (PAB missing-row handling is TBD) |
| `pab.window`, `pab.pooled`, `pab.movement_unit` | — | gap M2 |
| `meaningfulness` | `ResultEntry.meaningfulness` | copied |
| `limitations` | `data_conditions[]`, `excluded_cohorts[]`, `definition_status`, scope status | copied, each with its source |
| `provenance.queries_executed` | `AnalysisResponse.queries_executed` | `NOT_RETURNED` per result, because the count is per response; the response-level count is reported once per response |
| `provenance.tables` | `definition_lock[].table`, `denominator_table` | copied |
| `provenance.evidence_refs` | `ResultEntry.evidence_ref`, `result_id` | copied |

## Kept in their own sources (not forced into CapabilityResult)

- **Cut findings:** movement, share, weight and contribution of breakdown cells (`findings[]`, `cuts`).
  Descriptive cells keep "movement shown descriptively; no cell-level statistic". New-user and
  old-user cells are cut findings.
- **Cell statistics:** Classic Retention and LTV & LTDays cell results are ledger entries with a
  `parent_result_id`, adapted like any other result and reported under their parent.
- **PERIOD evidence:** trigger evidence, `correlation` labels (including `NOT_COMPARABLE`),
  game-level movement and lifecycle (`PeriodEvidenceRecord`, `master-state.md`).
- **Experiment Card fields:** `MasterState.context` and annotations (`report-templates.md` § Experiment
  Card).
