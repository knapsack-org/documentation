# MATLAB / Octave function reference

Signatures and outputs. Behaviour is specified in [functional/](../../functional/README.md); each entry links to it. Instance data are row vectors of doubles holding integers. **Items are returned 1-based and sorted.**

## Entry points

| Function | Description |
|---|---|
| `raw = run_benchmark(request_file, instances_file, out_dir)` | Full [run](../../functional/README.md#benchmark-run-in-one-paragraph). Every argument is optional (`''` = absent): a [run request](../../functional/output-format.md#run-request) file, an [instance set](../../functional/output-format.md#instance-set-file) used instead of the generator, and the output folder (default `output/run-dd-mm-yy-hh-mm/`). Writes [`raw.json`](../../functional/output-format.md#raw-results-knapsack-raw-v1) and, without an instance set, `instances.json`; returns the raw struct. `run_benchmark('--describe')` prints the algorithms, default settings and JSON Schemas (the knapsack-server catalog) |
| `cfg = benchmark_config(request)` | The [parameters](../../functional/parameters.md): `seed`, `instances` (from `kb_config_instances`), `algorithms` and one block per id (from `kp_algo_<id>_config`). The optional request struct is merged in with `kb_merge_settings`; an unknown key is an error |

## Runner helpers (`src/helpers/`)

Called by `run_benchmark`; each is its own file, so it can be called and tested alone.

| Function | Description |
|---|---|
| `[algos, skipped] = kb_discover_algorithms()` | scans `src/algorithms/`; `algos`: struct array (alphabetical) with `id`, `label`, `short_label`, `exact`, `reference`, `environment` (struct), `output` (the `kp_algo_<id>_output` struct); `skipped`: cell of `{id, reason}`. Errors `kb_discover_algorithms:badId`, `:missingFile`, `:idMismatch`, `:duplicateName`, `:reference` |
| `res = kb_run_algorithm(algo, v, w, C, cfg, k)` | runs one discovered algorithm with its timing repeats (`algo.output.run`) on instance `k` (global id, seed `cfg.seed + 1000 k`) and builds the raw [result object](../../functional/output-format.md#raw-results-knapsack-raw-v1) (quality + `cost`). knapsack-server's output service computes `is_optimal`, `gap_percent` and the ratios |
| `out = kb_merge_settings(base, override, where)` | merges a request block into a settings struct; errors on an unknown key |
| `[mark, times] = kb_phase_timer()` | phase timer for `kp_algo_<id>_main`: each `mark('name')` stores in the `containers.Map` `times('name')` the seconds since the previous mark |
| `name = kb_interpreter()` | `'GNU Octave'` or `'MATLAB'` |
| `kb_print_instance(...)`, `kb_print_progress(instance, algorithm, event)` | console output, including the progress lines `@@progress {"instance":…,"algorithm":…,"event":…}` |
| `txt = kb_write_json(file, data, algos)` | JSON writer; the schema files are inserted verbatim |
| `kb_lcg_states` | the next generator states as one vector operation (used by the randomised algorithms) |

## Algorithm folders (`src/algorithms/<id>/`)

Contract and skeletons: [adding an algorithm](../adding-an-algorithm.md). For every `<id>` (`bnb`, `dp`, `ilp`, `qubo`, …):

| Function | Description |
|---|---|
| `L = kp_algo_<id>_loader()` | `id`, `label`, `description`, `short_label`, `exact`, `reference`, `available` (handle → logical), `environment` (handle → struct) |
| `[items, info] = kp_algo_<id>_main(v, w, C, acfg, seed, mark)` | the maths; trailing arguments optional, see below |
| `c = kp_algo_<id>_config()` | the block `cfg.<id>` ([parameters](../../functional/parameters.md#algorithm-settings)), always with `repeats` |
| `O = kp_algo_<id>_output()` | `O.run(v, w, C, acfg, seed)` → common result (`label`, `exact`, `proven_optimal`, `items`, `working_memory_bytes`, `operations`, `operations_unit`, `details`); `O.entry(acfg, env)` → `algorithms[]` entry. Its JSON Schema is the file `kp_algo_<id>_output.schema.json` next to it (`entry`, `config`, `details`), byte-identical to the Python file |

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

## Infrastructure

| Function | Description |
|---|---|
| `b = kb_bytes(varargin)` | total `whos` bytes of the given arrays |
| `bytes = kb_process_memory()` | process RAM via `memory()`, `NaN` if unavailable |
| `kb_write_json(file, data)` | JSON writer; errors `kb_write_json:noJson`, `kb_write_json:open` |
