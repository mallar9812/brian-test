# Charts

Charts are requested through `visual_requirements` and built as `VisualEvidence`
(`references/visual-evidence.md`). `game-viz-design` renders them. Every rule in
`visual-evidence.md` § Rendering rules and in `game-viz-design` applies unchanged. This file only says
which visuals the generator requests.

## Which visuals

The defaults are those of `visual-evidence.md` § Visuals by mode and depth, at least to the SR-22
floor:
- **Per metric:** one focused chart for each metric with a result. Never one overloaded chart.
- **Experiment period:** a `SERIES` from the returned `observations`. Each group is drawn as its own
  line on the metric's native date axis.
  - With several variants, the reference group and every variant of the same metric share one chart
    only when every comparison returns the same reference series. Otherwise each comparison gets its
    own chart.
  - Groups are never pooled.
- **Daily PAB:** when the series carries `pab_observations` (`visual-evidence.md` § Building it,
  step 7), the same chart also shows each group's pre-experiment values, with the experiment start
  marked (rendering rule 10). Otherwise no pre-experiment trend is drawn, and a returned `pab_value`
  is stated as a labelled pooled PAB movement.
- **No series returned, or count mismatch:** when `observations` is not returned or its count differs
  from `n_pairs`, the visual is a `SUMMARY` with the header the existing rule gives ("daily series not
  returned", "returned series does not match the test").
- **Breakdowns:** a `BREAKDOWN` from the cut findings as returned (bars). No daily series is drawn for
  a pooled cell.
- **New and old users:** separate bars or panels, never a combined line.
- **PERIOD:** the trigger series with lookback, comparison and impact windows on the trigger's own
  axis; cross-capability evidence with correlation labels, one panel per date axis; game-level
  movement as a card; a lifecycle timeline.
- **Single-arm (`NO_COMPARISON`):** a descriptive `SUMMARY` or card per variant.

## Kept from the existing rules

- **Date axes:**
  - Install-date and activity-date results are never on one chart.
  - Daily PAB observations are activity dates, so an install-date metric never gets a pre-experiment
    trend.
- **Intervals:** drawn only when returned. None is computed, and none is drawn per day.
- **Colours:**
  - Delta colours are neutral unless the direction is established (`visual-evidence.md` § Direction).
  - Economy `spend_per_dau` is always neutral.
  - KPI and healthcheck colours are unchanged.
- **Gaps and partial days:** excluded or missing days are gaps, never zero. Days with only a numerator
  or only a denominator are not plotted.
- **No confidence conversion:** charts show no "Confidence (1−p)" or converted confidence value.
