# Template population — sections and their sources

`references/report-templates.md` defines the templates, their selection and their sections. This file
only says which structured field fills each section and what is shown when it is empty. It changes no
template, section, requirement or selection rule.

Empty handling: an optional section with no evidence is left out (SR-26). A required section that
lacks its input is still shown, with the status of the missing input (for example
"Stage 2 not run", "PAB not run", "daily series not returned").

## Shared

| Section | Source |
|---|---|
| Experiment Card | `MasterState.context` and annotations, per `report-templates.md` § Experiment Card. Missing fields as defined there ("not available", "unknown", "allocation not documented in annotations"). |
| Period & segment line | `MasterState.context` windows, scope, versions (`output-format.md` § Period & segment line) |
| Charts | `charts.md` |
| Next question | the Master's recorded follow-ups and unrun work; never generated from prose |

## A — Standard Experiment Read

| Section | Source |
|---|---|
| Setup and windows | `MasterState.context`: design, comparisons, analysis and matured windows, scope |
| Outcome summary | `result_ledger` meaningfulness per unit and pair, and the annotation status; never a ship or kill recommendation |
| Primary metric outcomes | only metrics the user named as primary in this conversation; absent otherwise |
| Supporting metric outcomes | `summary-table.md`, every other result once |
| Key positive and negative movements | results whose `direction` (`visual-evidence.md` § Direction) is established; absent otherwise |
| Evidence quality and limitations | `data_conditions`, maturity, `UNCONFIRMED` definitions, `NOT_TESTED` work |

## B — Targeted Diagnosis

| Section | Source |
|---|---|
| Requested metric and exact scope | `MasterState` request inputs and canonical scope with value-group expansions |
| Movement of the requested metric | the requested metric's ledger entries, per unit and pair (`summary-table.md` rows) |
| Where it moved | the capability's Stage 2 `findings[]` and `cuts`, as returned |
| Cohort and segment findings | the same sources; New and Old user cells shown as separate descriptive findings |
| Established vs not established | ledger `direction` and states |
| Depth-3 additions | mix-shift, evidence timeline and hypothesis table, only when produced |

It carries no full-experiment results table.

## C — PERIOD Impact Read

| Section | Source |
|---|---|
| Trigger metric, scope, lookback, windows, onset | `PeriodEvidenceRecord` and `MasterState.period` |
| Trigger movement and evidence | game-level movement (labelled game level) and trigger evidence per unit, as recorded |
| Supporting capability evidence | ledger entries with their movement labels |
| Same-axis correlation | ledger `correlation`; `NOT_COMPARABLE` listed by name |
| Limitations | `NOT_TESTED`, `OUT_OF_BUDGET` units, excluded units, data conditions |

Association is never worded as cause.

## D — Metric Investigation

| Section | Source |
|---|---|
| Metric definition and reference value | `definition_lock` and the reference value as returned |
| Overall movement | the metric's ledger entries per unit and pair |
| Driver breakdown | Stage 2 contributors and Stage 3 mix-shift, as returned |
| Supported cuts and cohorts | cell results under their parent, as returned |
| Final diagnosis and evidence strength | established findings and association-only hypotheses with their status; no ship or kill |

## Detailed Read (Template A, on experiment-read wording)

Used only when `report-generator.md` step 1a selects it for a `FULL_EXPERIMENT_READ`: the user's
request asks for an experiment read without a high-level qualifier (brian-master SR-2, for that
response). Template selection, special states and Templates B, C and D are unchanged. In those, an
explicitly requested detailed read is answered in the existing format, with a note that the Detailed
Read layout does not apply and why. Sections, exactly in this order:

| # | Section | Source | When unavailable |
|---|---|---|---|
| 1 | Verdict | Outcome summary: meaningfulness per unit and pair, and annotation status (`SHIPPED` or `KILLED` only as annotated) | never a ship or kill recommendation |
| 2 | Objective | Experiment Card user-provided objective | "not provided" |
| 3 | Feature description | Experiment Card user-provided changes per arm | "not provided" |
| 4 | Experiment details | Experiment Card and setup and windows | as the Experiment Card defines |
| 5 | Summary | `summary-table.md` § Detailed Read table, priority rows first | row statuses |
| 6 | Recommendations | recorded follow-ups, offered checks and unresolved data gaps | "no further step identified"; never ship or kill |
| 7 | General metrics | results of the registered capabilities, by capability, with their charts | statuses carried |
| 8 | Feature metrics | always present | "Feature-metric data is unavailable: no capability or source returns feature-specific metrics for this experiment." |
| 9 | Detailed insights | Stage 2 and Stage 3 findings, cuts and grey-zone evidence, as returned | "No additional detailed insights are available: no supported findings were returned." |
| 10 | Closing summary | what the data does not establish, and the next question | — |

**Layout rules for a Detailed Read:**
- **Verdict first.** The answer begins with section 1, Verdict. No title, heading, preamble, context
  paragraph or Experiment Card comes before it. The first line of the answer is the section 1 heading
  itself; no sentence precedes it, including one saying what is being read or which tools are unavailable.
- **All ten sections.** All 10 sections appear, each under its own numbered heading, in the order
  above. Section 9 is never left out: when no supported findings exist, it states the sentence above.
- **Experiment Card in section 4 only.** The Experiment Card's content appears only inside section 4,
  Experiment details. It is not shown before section 1 and not repeated later. For a Detailed Read this
  replaces the shared rule that the Experiment Card opens the answer.
- **Where context goes.** Context that would otherwise be a preamble (what ran, what did not, data
  provenance, tool or chart availability) goes in section 4 or section 10.

Charts still meet the SR-22 floor and sit inside General metrics and Detailed insights. Numbers come
only from structured results, never from analyst prose, and no new statistic is derived.
