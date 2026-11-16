# Parameters

Every run is fully defined by these parameters (plus the code version). Both implementations use the **same keys and defaults**; a run writes the complete resolved set to `parameters` in the [output](output-format.md).

Overrides are merged **key by key** into the defaults (nested groups are merged, not replaced). An **unknown key is an error**.

## Instances

| Key | Default | Meaning | Used in |
|---|---|---|---|
| `seed` | `2026` | master seed: instances, and the tabu streams of `qubo` / `qubo_chz` | [instances](instances.md), [qubo](algorithms/qubo.md#2-solver--tabu-search-mode-slack-optimal-default) |
| `instances.count` | `100` | number of generated instances $N$ | [instances](instances.md) |
| `instances.min_items` | `10` | size of the first instance | [instances](instances.md) |
| `instances.max_items` | `48` | size of the last instance | [instances](instances.md) |
| `instances.max_coefficient` | `100` | $R$: weights in $1 \dots R$; correlation offset $\lfloor R/10 \rfloor$ | [instances](instances.md) |
| `instances.capacity_ratio` | `0.5` | $\rho$: $C = \lfloor \rho \sum w_i \rfloor$ | [instances](instances.md) |

The `instances` block is the instance generator's settings. A run writes it, with `seed`, as `instance_config` in the [output](output-format.md#instance_config).

## Algorithm selection

| Key | Default | Meaning |
|---|---|---|
| `algorithms` | `["dp", "bnb", "ilp", "qubo", "qubo_chz"]` | algorithms to run, in this order. Unknown ids are an error. The reference optimum is always computed with `dp`, even if `dp` is not listed |

## Algorithm settings

Each algorithm has its own settings block, keyed by its id. Every block has `repeats`. A run writes each resolved block twice: in `parameters.<id>` and in `algorithms[].config` of the [output](output-format.md#algorithms).

### `dp`

| Key | Default | Meaning |
|---|---|---|
| `dp.repeats` | `5` | timed runs per instance |

### `bnb`

| Key | Default | Meaning |
|---|---|---|
| `bnb.max_nodes` | `5000000` | node limit; when exceeded the search stops and the result is not proven optimal ([bnb](algorithms/bnb.md)) |
| `bnb.repeats` | `1` | can be slow on strongly correlated instances |

### `ilp`

| Key | Default | Meaning |
|---|---|---|
| `ilp.repeats` | `3` | |

### `qubo`

| Key | Default | Meaning |
|---|---|---|
| `qubo.penalty` | `"auto"` | $P$: `"auto"` → $\max v_i + 1$, or a number ([qubo §1.4](algorithms/qubo.md#14-penalty)) |
| `qubo.tabu.mode` | `"slack-optimal"` | `"slack-optimal"` (flip item bits, slack set optimally) or `"all-bits"` (flip any of the $n+K$ bits) |
| `qubo.tabu.restarts` | `20` | independent random starts |
| `qubo.tabu.max_iter_factor` | `100` | iterations per restart $= \text{factor} \times N$ |
| `qubo.tabu.stall_factor` | `20` | a restart ends after more than $\text{factor} \times N$ iterations without improving its own best |
| `qubo.tabu.tenure_min` | `10` | tabu tenure $T = \text{tenure\_min} + \operatorname{round}(N/10)$ |
| `qubo.repeats` | `1` | deterministic and slow |

$N$ = number of searched bits: $n$ in `slack-optimal` mode, $n + K$ in `all-bits` mode.

### `qubo_chz`

Own copy of the QUBO settings, so `qubo_chz` can be tuned without touching `qubo` ([qubo_chz](algorithms/qubo_chz.md)).

| Key | Default | Meaning |
|---|---|---|
| `qubo_chz.penalty` | `"auto"` | as `qubo.penalty` |
| `qubo_chz.seed_offset` | `0` | added to the tabu seed; `0` → same random starts as `qubo` |
| `qubo_chz.tabu.*` | same as `qubo.tabu.*` | as `qubo.tabu.*` |
| `qubo_chz.repeats` | `1` | |

## `timing`

| Key | Default | Meaning |
|---|---|---|
| `timing.trace_memory` | `false` | Python only: extra untimed run to measure peak allocations ([Python](../implementation/python/README.md#cost-measurement)) |

## Keys renamed in `schema_version` 2

| Version 1 | Version 2 |
|---|---|
| `bnb_max_nodes` | `bnb.max_nodes` |
| `penalty`, `tabu.*` | `qubo.penalty`, `qubo.tabu.*` |
| `timing.repeats_<id>` | `<id>.repeats` |
| `num_instances`, `min_items`, `max_items`, `max_coefficient`, `capacity_ratio` | `instances.count`, `instances.min_items`, `instances.max_items`, `instances.max_coefficient`, `instances.capacity_ratio` |

All repeats return the same result (every algorithm is deterministic); repeats only stabilise the time measurement ([metrics](metrics.md#execution-cost)).
