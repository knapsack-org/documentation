# MATLAB / Octave function reference

Signatures and outputs. Behaviour is specified in [functional/](../../functional/README.md); each entry links to it. Instance data are row vectors of doubles holding integers. **Items are returned 1-based and sorted.**

## Entry points

| Function | Description |
|---|---|
| `result = run_benchmark()` | Full [run](../../functional/README.md#benchmark-run-in-one-paragraph) of every discovered algorithm on every instance; no argument. Writes `output/run-dd-mm-yy-hh-mm.json`; returns the written struct plus `output_file` |
| `cfg = benchmark_config()` | The [parameters](../../functional/parameters.md): `seed`, `instances` (from `kb_config_instances`), `algorithms` (every folder id, alphabetical) and one block per id (from `kp_algo_<id>_config`). No argument, no merging |

## Runner helpers (`src/helpers/`)

Called by `run_benchmark`; each is its own file, so it can be called and tested alone.

| Function | Description |
|---|---|
| `[algos, skipped] = kb_discover_algorithms()` | scans `src/algorithms/`; `algos`: struct array (alphabetical) with `id`, `label`, `short_label`, `exact`, `reference`, `environment` (struct), `output` (the `kp_algo_<id>_output` struct); `skipped`: cell of `{id, reason}`. Errors `kb_discover_algorithms:badId`, `:missingFile`, `:idMismatch`, `:duplicateName`, `:reference` |
| `res = kb_run_algorithm(algo, v, w, C, cfg, k)` | runs one discovered algorithm with its timing repeats (`algo.output.run`); validates `details`; builds the [result object](../../functional/output-format.md) (quality + `cost`); `is_optimal` / `gap_percent` are filled by `run_benchmark` once the reference value is known |
| `results = kb_add_ratios(results, ids, ref)` | adds `cost_ratio_vs_reference` ([spec](../../functional/metrics.md#ratios-to-the-reference)) |
| `kb_validate_schema(x, schema, where)` | errors `kb_validate_schema:invalid` (message starts with `where`) when `x` does not match the JSON Schema subset `schema` (struct from `jsondecode`) |
| `[mark, times] = kb_phase_timer()` | phase timer for `kp_algo_<id>_main`: each `mark('name')` stores in the `containers.Map` `times('name')` the seconds since the previous mark |
| `s = kb_summary(runs, algos, total_time)` | [summary statistics](../../functional/metrics.md#summary-statistics) |
| `name = kb_interpreter()` | `'GNU Octave'` or `'MATLAB'` |
| `kb_print_instance(r, algos)`, `kb_print_summary(s, algos, total_time, out_file)` | console output |
| `s = kb_human_bytes(b)` | byte count as `B`/`kB`/`MB` text |

## Algorithm folders (`src/algorithms/<id>/`)

Contract and skeletons: [adding an algorithm](../adding-an-algorithm.md). For every `<id>` (`bnb`, `dp`, `ilp`, `qubo`, `qubo_chz`, …):

| Function | Description |
|---|---|
| `L = kp_algo_<id>_loader()` | `id`, `label`, `description`, `short_label`, `exact`, `reference`, `available` (handle → logical), `environment` (handle → struct) |
| `[items, info] = kp_algo_<id>_main(v, w, C, acfg, seed, mark)` | the maths; trailing arguments optional, see below |
| `c = kp_algo_<id>_config()` | the block `cfg.<id>` ([parameters](../../functional/parameters.md#algorithm-settings)), always with `repeats` |
| `O = kp_algo_<id>_output()` | `O.run(v, w, C, acfg, seed)` → common result (`label`, `exact`, `proven_optimal`, `items`, `working_memory_bytes`, `operations`, `operations_unit`, `details`); `O.entry(acfg, env)` → `algorithms[]` entry; `O.schema.entry`, `O.schema.details` (JSON Schema structs) |

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
| `[items, info] = kp_algo_dp_main(v, w, C)` | `info.value`, `info.operations` = $n(C+1)$, `info.working_memory_bytes` (computed only if `info` is requested) | [dp](../../functional/algorithms/dp.md) |
| `[items, info] = kp_algo_bnb_main(v, w, C, acfg)` | `acfg.max_nodes` (default `Inf` when `acfg` is omitted); `info.value`, `info.nodes`, `info.proven_optimal`, `info.working_memory_bytes` | [bnb](../../functional/algorithms/bnb.md) |
| `[items, info] = kp_algo_ilp_main(v, w, C)` | `info.value`, `info.solver` (`'intlinprog'`/`'glpk'`), `info.status`, `info.proven_optimal`, `info.nodes` (`NaN` with glpk), `info.working_memory_bytes`. Error `kp_algo_ilp_main:noSolver` | [ilp](../../functional/algorithms/ilp.md) |
| `name = kb_ilp_solver_name()` (`ilp/modules/`) | `'intlinprog'` (MATLAB), `'glpk'`, or `''` if none | |

## QUBO — [spec](../../functional/algorithms/qubo.md)

| Function | Description |
|---|---|
| `[items, info] = kp_algo_qubo_main(v, w, C, acfg, seed, mark)` | penalty → model → tabu search → decode; calls `mark('build')`, `mark('solve')`, `mark('decode')`; `info`: `penalty`, `variables`, `slack_bits`, `model_memory_bytes`, `energy`, `mode`, `iterations`, `move_evaluations`, `working_memory_bytes`, `sol` |
| `[Q, c, d, K] = kb_knapsack2qubo(v, w, M, penalty)` | [model](../../functional/algorithms/qubo.md#13-matrix-form); `penalty` default 1; column vectors, dense `Q` |
| `bits = kb_encode_slack(s, M)` | $K \times 1$ slack bits |
| `[x, energy, stats] = kb_qubo_tabu_slack(v, w, C, penalty, Q, c, d, opts)` | slack-optimal search; `x` = full $(n+K) \times 1$ vector |
| `[best_x, best_f, stats] = kb_qubo_tabu(Q, c, d, opts)` | all-bits search |
| `sol = kb_decode(x, v, w, C)` | fields `items`, `value`, `weight`, `feasible`, `repaired` (raw) and `final_items`, `final_value`, `final_weight` |

`opts` = the `tabu` parameter group plus `seed` (set by the runner to `seed + 1000*k`). `stats`: `iterations`, `restarts`, `tenure`, `move_evaluations`, `working_memory_bytes`.

The QUBO functions are in `src/algorithms/qubo/modules/`. Penalty rule (`kb_penalty(values, rule)`, same folder): a char `rule` → `max(values) + 1`, else `rule`.

## `qubo_chz` — [spec](../../functional/algorithms/qubo_chz.md)

`kp_algo_qubo_chz_main` and, in `src/algorithms/qubo_chz/modules/`, `kb_chz_penalty`, `kb_chz_knapsack2qubo`, `kb_chz_encode_slack`, `kb_chz_qubo_tabu_slack`, `kb_chz_qubo_tabu`, `kb_chz_decode`: same signatures as the functions above, separate copies of the code. It uses the `qubo_chz` parameter group and seed `seed + 1000*k + seed_offset`.

## Infrastructure

| Function | Description |
|---|---|
| `b = kb_bytes(varargin)` | total `whos` bytes of the given arrays |
| `bytes = kb_process_memory()` | process RAM via `memory()`, `NaN` if unavailable |
| `kb_write_json(file, data)` | JSON writer; errors `kb_write_json:noJson`, `kb_write_json:open` |
