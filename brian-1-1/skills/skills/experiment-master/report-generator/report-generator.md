# Report generator — presentation only

Assembles the experiment answer from what the Master already holds. It is a presentation step that
runs after Synthesis (`SKILL.md` § Synthesis, § Visualization hand-off). It is not a registered
analysis capability and has no registry entry.

## What it never does

- Issue a query, run or recompute a statistic, PAB value, interval, power or adjustment, classify
  meaningfulness or correlation, or change routing, mode, depth, budget or state.
- Take a number, a direction or a verdict from prose. Every figure comes from a structured field
  (`AnalysisResponse`, `result_ledger`, PERIOD records) and keeps its source reference.
- Include analyst commentary. Metric-analyst and investigator prose is not part of the generated report.
- Recommend shipping or killing a variant.
- Show "Confidence (1−p)" or any converted confidence value, except the Detailed Read Confidence
  column (`summary-table.md` § Detailed Read table): `(1 − p) × 100`, a display transformation of the
  returned p-value, not a recomputed statistic. Results follow `output-format.md`
  § Results and meaningfulness unchanged: point estimate and supplied 95% CI first, significance as a
  separate statement, meaningfulness beside it.
- Fill a missing value, unit, PAB value, window, weight, confidence or classification. Unavailable
  values keep their status and reason and are never shown as zero.
- Claim that a chart was rendered or that daily PAB data was retrieved live. It describes only what
  the structured inputs contain.

## Inputs (all already held by the Master)

| Input | Source |
|---|---|
| Recorded mode, depth, `depth_basis`, selected capabilities, `segment_ref` | `MasterState` (`references/master-state.md`) |
| Resolved context: experiment, versions, windows, comparisons, lifecycle, annotation status, scope | `MasterState.context` (`references/context-schema.md`) |
| Results and their states, meaningfulness, PERIOD correlation | `result_ledger` (`master-state.md` § Results and meaningfulness) |
| Statistics, observations, data conditions, PAB results, findings and cuts | stored `analysis_responses` (`references/analysis-contract.md`) |
| PERIOD trigger, game-level movement, evidence records | `PeriodEvidenceRecord`s (`references/period-evidence.md`) |
| Visual requirements | `visual_requirements` (`master-state.md` § Visual requirements) |

## Steps

1. **Template.** Select the structure with `references/report-templates.md` § Selection, from recorded
   state only. Special states and `NO_COMPARISON` keep their existing formats. The selection never
   changes mode or depth.
1a. **Detailed Read (experiment-read wording).** When the selected structure is A and the user's own
   request (`MasterState.request.raw`) asks for an experiment read without a high-level qualifier, lay
   the answer out as `template-population.md` § Detailed Read and build its table with
   `summary-table.md` § Detailed Read table. The wording is decided in this order:
   - **High-level qualifier** ("high-level read", "high level read", "high-level summary"): the
     Standard Experiment Read, unchanged.
   - **Experiment-read wording** ("exp read", "exp reads", "read experiment X", "experiment read",
     "detailed read"): the Detailed Read.
   - **Anything else** (for example "how did the test do?", "A/B results"): the Standard Experiment
     Read, unchanged.

   This is a per-response presentation choice taken from the user's wording (brian-master SR-2). It
   never changes the selected template, mode, depth, capabilities or budget, and a stream-origin
   request never takes it. Every other answer keeps its existing order and structure.
2. **Adapt.** Build one report record per `result_ledger` entry with `adapter.md`. Cut findings and
   PERIOD evidence are read from their own sources, unchanged.
3. **Populate.** Fill each section of the selected template with `template-population.md`. A section
   without evidence is left out (SR-26), but every non-computed state is still named.
4. **Tables.** Build the results tables with `summary-table.md`.
5. **Charts.** Build `visual_requirements` and `VisualEvidence` with `charts.md`;
   `game-viz-design` renders them.
6. **Assemble** in SR-15 order as `report-templates.md` § Shared rules places each section: Experiment
   Card, period and segment line, visuals, bottom line, supporting data, next question.

## Statuses

Every status is carried with its reason, in the existing vocabulary:
- ledger states `NOT_IN_TABLE`, `NO_SCOPED_EXPOSURE`, `INSUFFICIENT_DATA`, `UNSUPPORTED_FOR_SCOPE`,
  `NOT_TESTED`, `NOT_RUN` and `NO_COMPARISON`;
- meaningfulness classifications;
- `pab[]` statuses `COMPUTED`, `NOT_APPLICABLE`, `NOT_IN_TABLE`, `CONFIG_REQUIRED` and `TBD`;
- `CapabilityResult` availability statuses (`experiment-contracts/references/capability-result.md`).

`output-format.md` wording rules apply: not run is never worded as missing data or as no movement.
