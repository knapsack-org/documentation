# MATLAB / Octave function reference

Signatures and outputs. Behaviour is specified in [functional/](../../functional/README.md); each entry links to it. Instance data are row vectors of doubles holding integers. **Items are returned 1-based and sorted.**

## Entry points

| Function | Description |
|---|---|
| `result = run_benchmark(opts)` | Full [run](../../functional/README.md#benchmark-run-in-one-paragraph). `opts`: optional override struct. Writes `output/run-dd-mm-yy-hh-mm.json`; returns the written struct plus `output_file` |
| `cfg = benchmark_config(overrides)` | [Defaults](../../functional/parameters.md), with `overrides` merged recursively. Error `benchmark_config:unknownOption` on an unknown field |

## Runner helpers (`src/helpers/`)

Called by `run_benchmark`; each is its own file, so it can be called and tested alone.

| Function | Description |
|---|---|
| `name = kb_ilp_solver_name(algos)` | `'intlinprog'` (MATLAB), `'glpk'`, or `''` if none or `ilp` not in `algos` |
| `res = kb_run_algorithm(name, v, w, C, cfg, k, optimum)` | runs one algorithm with its timing repeats; builds the [result object](../../functional/output-format.md) (quality + `cost`) |
| `results = kb_add_ratios_vs_dp(results, algos)` | adds `cost_ratio_vs_dp` ([spec](../../functional/metrics.md#ratios-to-dp)) |
| `s = kb_summary(runs, algos, total_time)` | [summary statistics](../../functional/metrics.md#summary-statistics) |
| `p = kb_penalty(values, rule)` | QUBO penalty, see [QUBO](#qubo--spec) |
| `name = kb_interpreter()` | `'GNU Octave'` or `'MATLAB'` |
| `kb_print_instance(r, algos)`, `kb_print_summary(s, algos, total_time, out_file)` | console output |
| `s = kb_human_bytes(b)` | byte count as `B`/`kB`/`MB` text |

## Algorithm loader (`src/algorithms/algorithm_loader.m`)

`L = algorithm_loader()` groups everything the runner needs to load the algorithms; used by `benchmark_config`, `run_benchmark` and `kb_run_algorithm`.

| Field | Description |
|---|---|
| `L.ids` | registry of algorithm ids, default run order (cell) |
| `c = L.configs()` | struct of the default settings blocks, one field per id (`kb_config_<id>()`) |
| `algos = L.check(requested)` | validates ids (error `run_benchmark:unknownAlgorithm`); drops `ilp` with warning `run_benchmark:noIlp` when no solver exists |
| `f = L.adapter(id)` | handle to the adapter `kb_algo_<id>` |

## Algorithm adapters (`src/algorithms/<id>/kb_algo_<id>.m`)

`out = kb_algo_<id>(v, w, C, acfg, seed)` for `dp`, `bnb`, `ilp`, `qubo`, `qubo_chz`; `acfg = cfg.<id>`, `seed = cfg.seed + 1000*k`. They call the algorithm and return the common struct `label`, `exact`, `proven_optimal`, `items`, `working_memory_bytes`, `operations`, `operations_unit`, `details`. Called by `kb_run_algorithm` through `L.adapter(id)` of the [loader](#algorithm-loader-srcalgorithmsalgorithm_loaderm); contract in [adding an algorithm](../adding-an-algorithm.md#adapter-contract).

## Algorithm settings (`src/algorithms/<id>/kb_config_<id>.m`)

`c = kb_config_<id>()` returns the default block `cfg.<id>` ([parameters](../../functional/parameters.md#algorithm-settings)); `benchmark_config` gets them for every registered id through `L.configs()` of the [loader](#algorithm-loader-srcalgorithmsalgorithm_loaderm).

## Random numbers — [spec](../../functional/instances.md#portable-random-generator)

| Function | Description |
|---|---|
| `state = kb_seed(seed)` | initial state |
| `[u, state] = kb_lcg(state, count)` | row of `count` uniforms in (0, 1) (`count` default 1) |
| `[r, state] = kb_randi(state, lo, hi, count)` | row of `count` integers in `lo..hi` |

## Instances — [spec](../../functional/instances.md)

In `src/instances/`:

- `instances = kb_generate_instances(count, seed, min_items, max_items, R, capacity_ratio)` — struct array with fields `family`, `weights`, `values`, `capacity`.
- `c = kb_config_instances()` — default block `cfg.instances` (`count`, `min_items`, `max_items`, `max_coefficient`, `capacity_ratio`). `run_benchmark` writes it with `seed` as `instance_config`.

## Exact algorithms

| Function | Output | Spec |
|---|---|---|
| `[best, items, cost] = kb_knapsack_dp(v, w, C)` | `cost.operations` = $n(C+1)$, `cost.working_memory_bytes` (computed only if `cost` is requested) | [dp](../../functional/algorithms/dp.md) |
| `[best, items, info] = kb_knapsack_bnb(v, w, C, max_nodes)` | `info.nodes`, `info.proven_optimal`, `info.working_memory_bytes`; `max_nodes` default `Inf` | [bnb](../../functional/algorithms/bnb.md) |
| `[best, items, info] = kb_knapsack_ilp(v, w, C)` | `info.solver` (`'intlinprog'`/`'glpk'`), `info.status`, `info.proven_optimal`, `info.nodes` (`NaN` with glpk), `info.working_memory_bytes`. Error `kb_knapsack_ilp:noSolver` | [ilp](../../functional/algorithms/ilp.md) |

## QUBO — [spec](../../functional/algorithms/qubo.md)

| Function | Description |
|---|---|
| `[Q, c, d, K] = kb_knapsack2qubo(v, w, M, penalty)` | [model](../../functional/algorithms/qubo.md#13-matrix-form); `penalty` default 1; column vectors, dense `Q` |
| `bits = kb_encode_slack(s, M)` | $K \times 1$ slack bits |
| `[x, energy, stats] = kb_qubo_tabu_slack(v, w, C, penalty, Q, c, d, opts)` | slack-optimal search; `x` = full $(n+K) \times 1$ vector |
| `[best_x, best_f, stats] = kb_qubo_tabu(Q, c, d, opts)` | all-bits search |
| `sol = kb_decode(x, v, w, C)` | fields `items`, `value`, `weight`, `feasible`, `repaired` (raw) and `final_items`, `final_value`, `final_weight` |

`opts` = the `tabu` parameter group plus `seed` (set by the runner to `seed + 1000*k`). `stats`: `iterations`, `restarts`, `tenure`, `move_evaluations`, `working_memory_bytes`.

Penalty rule (`kb_penalty(values, rule)`, in `src/helpers/`): a char `rule` → `max(values) + 1`, else `rule`.

## `qubo_chz` — [spec](../../functional/algorithms/qubo_chz.md)

`kb_chz_knapsack2qubo`, `kb_chz_encode_slack`, `kb_chz_qubo_tabu_slack`, `kb_chz_qubo_tabu`, `kb_chz_decode`: same signatures as the functions above, separate copies of the code. The runner uses the `qubo_chz` parameter group and seed `seed + 1000*k + seed_offset`.

## Infrastructure

| Function | Description |
|---|---|
| `b = kb_bytes(varargin)` | total `whos` bytes of the given arrays |
| `bytes = kb_process_memory()` | process RAM via `memory()`, `NaN` if unavailable |
| `kb_write_json(file, data)` | JSON writer; errors `kb_write_json:noJson`, `kb_write_json:open` |
