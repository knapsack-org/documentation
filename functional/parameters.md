# Parameters

Every run is fully defined by these parameters (plus the code version). Both implementations use the **same keys and defaults**; a run writes the complete resolved set to `parameters` in the [output](output-format.md).

Without a [run request](output-format.md#run-request) the benchmark runs every algorithm on every generated instance with the values of the settings files. A request (`--request` / the first argument of `run_benchmark`, or a knapsack-server run) changes `seed`, `instances.*`, the selection of algorithms and `<id>.*` for one run; absent fields keep the file values. To change a default, edit the file that defines it: `seed` in the benchmark configuration, `instances.*` in the instance generator's settings, `<id>.*` in the algorithm's own `kp_algo_<id>_config` (file names per implementation in [MATLAB](../implementation/matlab/README.md#run) and [Python](../implementation/python/README.md#run)).

## Instances

| Key | Default | Meaning | Used in |
|---|---|---|---|
| `seed` | `2026` | master seed: instances, and the tabu streams of `qubo` | [instances](instances.md), [qubo](algorithms/qubo.md#2-solver--tabu-search-mode-slack-optimal-default) |
| `instances.count` | `100` | number of generated instances $N$ | [instances](instances.md) |
| `instances.min_items` | `10` | size of the first instance | [instances](instances.md) |
| `instances.max_items` | `48` | size of the last instance | [instances](instances.md) |
| `instances.max_coefficient` | `100` | $R$: weights in $1 \dots R$; correlation offset $\lfloor R/10 \rfloor$ | [instances](instances.md) |
| `instances.capacity_ratio` | `0.5` | $\rho$: $C = \lfloor \rho \sum w_i \rfloor$ | [instances](instances.md) |

The `instances` block is the instance generator's settings. A run writes it, with `seed`, as `instance_config` in the [output](output-format.md#instance_config).

## Algorithms

| Key | Value | Meaning |
|---|---|---|
| `algorithms` | ids of every algorithm folder, alphabetical | written by the run, not set: every algorithm found in the algorithms folder runs, in alphabetical order of its id. An algorithm whose loader says it cannot run in this environment (e.g. `ilp` without a solver) is skipped with a warning and listed in `skipped_algorithms` of the [output](output-format.md#top-level) |

Exactly one algorithm declares itself the **reference** (`dp`); its value on an instance is the optimum $V^*$ that gaps and ratios use. A run stops with an error when no available exact algorithm, or more than one, is the reference.

## Algorithm settings

Each algorithm has its own settings block, keyed by its id, defined in the algorithm's own config file (`kp_algo_<id>_config`). Every block has `repeats`. A run writes each resolved block twice: in `parameters.<id>` and in `algorithms[].config` of the [output](output-format.md#algorithms).

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
| `ilp.time_limit_s` | `60` | seconds one solve may take (0 < t ≤ 300); when reached the solve stops, not proven optimal ([ilp](algorithms/ilp.md)) |
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

### `greedy`

| Key | Default | Meaning |
|---|---|---|
| `greedy.repeats` | `20` | timed runs per instance ([greedy](algorithms/greedy.md)) |

### `sa`

| Key | Default | Meaning |
|---|---|---|
| `sa.penalty` | `"auto"` | $P$ of the energy: `"auto"` = $\max v + 1$, or a number |
| `sa.restarts` | `10` | independent annealing runs |
| `sa.steps_factor` | `500` | proposals per restart = factor $\cdot\, n$ |
| `sa.t0_factor` | `1.0` | start temperature = factor $\cdot \max v$ |
| `sa.t_end_factor` | `0.01` | end temperature = factor $\cdot \max v$ |
| `sa.repeats` | `1` | ([sa](algorithms/sa.md)) |

### `ga`

| Key | Default | Meaning |
|---|---|---|
| `ga.pop_size` | `30` | individuals per generation (one elite kept) |
| `ga.generations` | `50` | generations after the random start |
| `ga.tournament` | `2` | individuals drawn per parent selection |
| `ga.crossover_rate` | `0.9` | probability of uniform crossover |
| `ga.mutation_factor` | `1.0` | per-bit flip probability = factor / $n$ |
| `ga.repeats` | `1` | ([ga](algorithms/ga.md)) |

### `ising`

| Key | Default | Meaning |
|---|---|---|
| `ising.penalty` | `"auto"` | $P$ of the QUBO / Ising model: `"auto"` = $\max v + 1$, or a number |
| `ising.restarts` | `5` | independent annealing runs |
| `ising.steps_factor` | `2000` | proposals per restart = factor $\cdot\, N$ ($N = n + K$ spins) |
| `ising.t0_factor` | `100` | start temperature = factor $\cdot\, P \max w$ |
| `ising.t_end_factor` | `1` | end temperature = factor $\cdot\, P \max w$ |
| `ising.repeats` | `1` | ([ising](algorithms/ising.md)) |

### `fptas`

| Key | Default | Meaning |
|---|---|---|
| `fptas.epsilon` | `0.1` | accuracy: value $\ge (1 - \varepsilon) V^*$ |
| `fptas.repeats` | `5` | ([fptas](algorithms/fptas.md)) |

## `timing`

| Key | Default | Meaning |
|---|---|---|
| `timing.trace_memory` | `false` | Python only, set in `config.py`: extra untimed run to measure peak allocations ([Python](../implementation/python/README.md#cost-measurement)) |

## Keys renamed in `schema_version` 2

`schema_version` 3 renamed no parameter; it removed the overrides and the selection of algorithms. `schema_version` 4 renamed none either; it brought back per-run values through the [run request](output-format.md#run-request).

| Version 1 | Version 2 |
|---|---|
| `bnb_max_nodes` | `bnb.max_nodes` |
| `penalty`, `tabu.*` | `qubo.penalty`, `qubo.tabu.*` |
| `timing.repeats_<id>` | `<id>.repeats` |
| `num_instances`, `min_items`, `max_items`, `max_coefficient`, `capacity_ratio` | `instances.count`, `instances.min_items`, `instances.max_items`, `instances.max_coefficient`, `instances.capacity_ratio` |

All repeats return the same result (every algorithm is deterministic); repeats only stabilise the time measurement ([metrics](metrics.md#execution-cost)).
