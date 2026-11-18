# Python module reference — `knapsack_benchmark`

Signatures and return values. Behaviour is specified in [functional/](../../functional/README.md); each entry links to it. Inputs `v`, `w` are array-likes (converted to `float64`), `C` an integer capacity. **Items are returned 1-based and sorted.**

## `knapsack_benchmark` (package)

| Name | |
|---|---|
| `run_benchmark` | re-export of `benchmark.run_benchmark` |
| `benchmark_config` | re-export of `config.benchmark_config` |
| `__version__` | `"0.1.0"` |

## `benchmark`

| Function | Description |
|---|---|
| `run_benchmark() -> dict` | Full [run](../../functional/README.md#benchmark-run-in-one-paragraph) of every discovered algorithm on every instance; no parameter; writes to `<project>/output`. Raises `DiscoveryError` / `SchemaError` (see [README](README.md#run)). Returns the result dict + `output_file` |
| `SCHEMA_VERSION` | `3` |

## Runner helpers (`helpers/`, one module per MATLAB `kb_*.m` helper)

| Module · function | Description |
|---|---|
| `discover.discover_algorithms() -> (algos, skipped)` | scans `algorithms/`; `algos`: list of dicts (alphabetical) with `id`, `label`, `short_label`, `exact`, `reference`, `environment` (dict), `output` (the `kp_algo_<id>_output` module); `skipped`: `[{id, reason}]`. Raises `DiscoveryError` |
| `discover.algorithm_ids() -> list[str]`, `discover.contract_module(id, part)` | every folder id (alphabetical, available or not); the module `algorithms/<id>/kp_algo_<id>_<part>.py` |
| `run_algorithm.run_algorithm(algo, v, w, C, cfg, k) -> dict` | Runs one discovered algorithm with its repeats (`algo["output"].run`), validates `details`, builds the [result object](../../functional/output-format.md#instancesresultsid); `is_optimal` / `gap_percent` are filled by `run_benchmark` once the reference value is known |
| `add_ratios.add_ratios(results, ref) -> None` | Adds `cost_ratio_vs_reference` in place ([spec](../../functional/metrics.md#ratios-to-the-reference)) |
| `schema.validate(x, schema, where="$") -> None` | Raises `schema.SchemaError` when `x` does not match the JSON Schema subset `schema` |
| `phase_timer.phase_timer() -> (mark, times)` | phase timer for `kp_algo_<id>_main`: each `mark(name)` stores in `times[name]` the seconds since the previous mark |
| `summary.summarize(runs, algos, total_time) -> dict` | [Summary statistics](../../functional/metrics.md#summary-statistics) |
| `write_json.write_json(path, data) -> None` | JSON writer (see [README](README.md#json-writing)) |
| `printing.print_instance(r, algos, log=print)`, `printing.print_summary(s, algos, total_time, out_file, log=print)`, `printing.human_bytes(b)` | console output |

## Algorithm packages (`algorithms/<id>/`)

Contract and skeletons: [adding an algorithm](../adding-an-algorithm.md). For every `<id>` (`bnb`, `dp`, `ilp`, `qubo`, `qubo_chz`, …):

| Module · name | Description |
|---|---|
| `kp_algo_<id>_loader.kp_algo_<id>_loader() -> dict` | `id`, `label`, `description`, `short_label`, `exact`, `reference`, `available` (callable → bool), `environment` (callable → dict) |
| `kp_algo_<id>_main.kp_algo_<id>_main(v, w, C, acfg=None, seed=None, mark=None) -> (items, info)` | the maths |
| `kp_algo_<id>_config.kp_algo_<id>_config() -> dict` | the block `cfg["<id>"]` ([parameters](../../functional/parameters.md#algorithm-settings)), always with `repeats` |
| `kp_algo_<id>_output.run(v, w, C, acfg, seed) -> dict` | common result: `label`, `exact`, `proven_optimal`, `items`, `working_memory_bytes`, `operations`, `operations_unit`, `details` |
| `kp_algo_<id>_output.entry(acfg, env) -> dict` | `algorithms[]` entry |
| `kp_algo_<id>_output.SCHEMA` | `{"entry": …, "details": …}` JSON Schemas |

## `config`

| Name | Description |
|---|---|
| `benchmark_config() -> dict` | The [parameters](../../functional/parameters.md): `seed`, `instances`, `algorithms` (every folder id, alphabetical), one block per id (from `kp_algo_<id>_config`) and `timing`. No argument, no merging |

## `rng` — [portable generator](../../functional/instances.md#portable-random-generator)

| Function | Returns |
|---|---|
| `seed_state(seed) -> int` | initial state |
| `lcg(state, count=1) -> (ndarray, int)` | `count` uniforms in (0, 1), new state |
| `randi(state, lo, hi, count=1) -> (ndarray, int)` | `count` integers in `lo..hi` (as floats), new state |
| `mround(x) -> int` | MATLAB-style round, $\lfloor x + 0.5 \rfloor$ |
| `M31` | $2^{31} - 1$ |

## `instances/` — [instance families](../../functional/instances.md)

Module `instances.instances` (generator) and `instances.config_instances` (settings):

| Name | Description |
|---|---|
| `FAMILIES` | `("uncorrelated", "weakly_correlated", "strongly_correlated")` |
| `Instance` | dataclass: `family: str`, `weights: ndarray`, `values: ndarray`, `capacity: int` |
| `generate_instances(count, seed, min_items, max_items, R, capacity_ratio) -> list[Instance]` | |
| `config_instances() -> dict` | default block `cfg["instances"]` (`count`, `min_items`, `max_items`, `max_coefficient`, `capacity_ratio`); `run_benchmark` writes it with `seed` as `instance_config` |

## `dp` — [spec](../../functional/algorithms/dp.md)

`kp_algo_dp_main(v, w, C, acfg=None, seed=None, mark=None) -> (items, info)` — `info = {"value", "operations": n*(C+1), "working_memory_bytes"}`.

## `bnb` — [spec](../../functional/algorithms/bnb.md)

`kp_algo_bnb_main(v, w, C, acfg=None, seed=None, mark=None) -> (items, info)` — node limit `acfg["max_nodes"]` (`inf` when `acfg` is `None`); `info = {"value", "nodes", "proven_optimal", "working_memory_bytes"}`.

## `ilp` — [spec](../../functional/algorithms/ilp.md)

`kp_algo_ilp_main(v, w, C, acfg=None, seed=None, mark=None) -> (items, info)` — `info = {"value", "solver": SOLVER, "status", "proven_optimal", "nodes", "mip_gap", "working_memory_bytes"}` with `SOLVER = "scipy.optimize.milp (HiGHS)"`; `nodes` / `mip_gap` are `None` if SciPy does not report them.

## `qubo` — [spec](../../functional/algorithms/qubo.md)

`kp_algo_qubo_main(v, w, C, acfg, seed, mark=None) -> (items, info)`: penalty → model → tabu search → decode, calling `mark("build")`, `mark("solve")`, `mark("decode")`. The functions below are in `algorithms/qubo/modules/qubo.py`.

| Function | Description |
|---|---|
| `slack_coefficients(M) -> list[int]` | slack weights $b_1 \dots b_K$ |
| `knapsack2qubo(v, w, M, penalty=1.0) -> (Q, c, d, K)` | [model](../../functional/algorithms/qubo.md#13-matrix-form); `Q` is a dense $(n+K)^2$ array |
| `encode_slack(s, M) -> ndarray` | $K$ slack bits |
| `energy(Q, c, d, x) -> float` | $x^\top Q x + c^\top x + d$ |
| `qubo_tabu_slack(v, w, C, penalty, Q, c, d, opts) -> (x, energy, TabuStats)` | slack-optimal search; `x` is the full $(n+K)$ vector |
| `qubo_tabu(Q, c, d, opts) -> (x, energy, TabuStats)` | all-bits search |
| `decode(x, v, w, C) -> dict` | keys `items`, `value`, `weight`, `feasible`, `repaired` (raw) and `final_items`, `final_value`, `final_weight` |
| `penalty_value(values, rule) -> float` | any string → $\max v + 1$, else `float(rule)` |
| `TabuStats` | dataclass: `iterations`, `restarts`, `tenure`, `move_evaluations`, `array_bytes` (returned by `_main` as `working_memory_bytes`) |

`opts` = the `tabu` parameter group plus `seed` (set by the runner to `seed + 1000·k`).

## `qubo_chz` — [spec](../../functional/algorithms/qubo_chz.md)

`kp_algo_qubo_chz_main` and `algorithms/qubo_chz/modules/qubo_chz.py`: same functions and signatures as `qubo`; a separate copy of the code. It uses the `qubo_chz` parameter group and seed `seed + 1000·k + seed_offset`.

## `measure` — [cost metrics](../../functional/metrics.md#execution-cost)

| Function | Description |
|---|---|
| `measure(fn, repeats, trace_memory=False) -> (result, cost: dict)` | runs `fn` (optional `tracemalloc` run, then `repeats` timed runs); `cost` keys: `repeats`, `wall_time_s`, `wall_time_min_s`, `cpu_time_s`, `traced_peak_memory_bytes`, `process_memory_before_bytes`, `process_memory_after_bytes`, `process_memory_delta_bytes` |
| `process_memory() -> int or None` | RSS in bytes, `None` without psutil |

## `cli`

`main(argv=None) -> int` — [command line](README.md#run): no options; returns 0, or 2 when any argument is given (0 for `-h` / `--help`).
