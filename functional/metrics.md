# Quality and cost metrics

Each algorithm result on each instance is measured on **quality** (how good the packing is) and **execution cost** (what it took to get it). The JSON field names are given in `code`; the full layout is in [output format](output-format.md).

## Quality

Let $S$ be the returned item set and $V^*$ the [`dp`](algorithms/dp.md) optimum of the instance.

| Field | Definition |
|---|---|
| `value` | $\sum_{i \in S} v_i$ (recomputed by the runner from the items, not trusted from the algorithm) |
| `weight` | $\sum_{i \in S} w_i$ |
| `feasible` | $\text{weight} \le C$ |
| `is_optimal` | $\text{value} = V^*$ — *is* the answer optimal? |
| `proven_optimal` | did the algorithm *prove* optimality? `dp`: always; `bnb`: tree exhausted; `ilp`: solver status; `qubo`, `qubo_chz`: never |
| `gap_percent` | $100\,(V^* - \text{value}) / V^*$ if $V^* > 0$, else $0$ |

`is_optimal` and `proven_optimal` are independent: a heuristic can be optimal without proving it.

## Execution cost

Each algorithm is run `repeats` times in a row on the instance ([parameters](parameters.md#timing)); the result of the last run is kept.

| Field | Definition |
|---|---|
| `repeats` | number of timed runs |
| `wall_time_s` | **median** wall-clock time of the runs |
| `wall_time_min_s` | fastest run |
| `cpu_time_s` | process CPU time over all runs ÷ `repeats` |
| `working_memory_bytes` | bytes held by the algorithm's **own arrays** (inputs, tables, vectors, QUBO matrix). Measured the same way in every language so the values are comparable; not the solver's internal memory for `ilp` |
| `process_memory_before_bytes`, `…_after_bytes`, `…_delta_bytes` | resident memory of the whole process before/after the runs; `null` when the platform cannot report it. Indicative only |
| `operations` | algorithm-specific work count, `null` if unknown |
| `operations_unit` | `table cells` (dp) · `nodes` (bnb, ilp) · `move evaluations` (qubo, qubo_chz) |

### Ratios to `dp`

`cost_ratio_vs_dp` compares each algorithm with `dp` on the same instance (> 1 = costlier):

$$
\text{time} = \frac{t}{\max(t_{dp},\ \varepsilon)},\qquad
\text{cpu} = \frac{t^{cpu}}{\max(t^{cpu}_{dp},\ \varepsilon)},\qquad
\text{memory} = \frac{m}{m_{dp}}\ (\text{null if either is 0 or missing}),
$$

with $\varepsilon$ a tiny positive constant that avoids a division by zero (value per implementation). All three are `null` when `dp` is not among the selected algorithms.

## Summary statistics

Per algorithm, over all instances (`summary.algorithms.<id>`):

| Field | Definition |
|---|---|
| `num_optimal`, `num_proven_optimal`, `num_feasible` | counts of `is_optimal`, `proven_optimal`, `feasible` |
| `mean_gap_percent`, `max_gap_percent` | of `gap_percent` |
| `total_wall_time_s`, `mean_wall_time_s`, `median_wall_time_s`, `max_wall_time_s` | of `wall_time_s` |
| `mean_cpu_time_s` | of `cpu_time_s` |
| `mean_…`, `median_…`, `max_working_memory_bytes` | of `working_memory_bytes` |
| `mean_operations`, `max_operations` | of `operations` |
| `median_time_ratio_vs_dp`, `median_memory_ratio_vs_dp` | medians of the per-instance ratios |
| `by_family[]` | per family, sorted by family name: `count`, `num_optimal`, `mean_gap_percent`, `mean_wall_time_s`, `max_wall_time_s`, `mean_working_memory_bytes` |

Statistics ignore `null` values; a statistic over only `null`s is `null`. The median of an even count is the mean of the two middle values.

Run level: `ranking_by_median_time` lists the algorithm ids by increasing `median_wall_time_s`; `total_time_seconds` is the wall time of the whole run.

## Comparing runs

Quality and `operations` are deterministic and must match across languages. Times and memory depend on the machine, the interpreter and the libraries: compare orders of magnitude, not exact values.
