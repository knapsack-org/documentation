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
| `run_benchmark(overrides=None, output_dir=None, verbose=True) -> dict` | Full [run](../../functional/README.md#benchmark-run-in-one-paragraph). `overrides`: dict merged by `benchmark_config`. `output_dir`: default `<project>/output`. Raises `ValueError` on an unknown algorithm, `KeyError` on an unknown parameter. Returns the result dict + `output_file` |
| `SCHEMA_VERSION` | `1` |

## Runner helpers (`helpers/`, one module per MATLAB `kb_*.m` helper)

| Module · function | Description |
|---|---|
| `check_algorithms.check_algorithms(requested) -> list` | `ValueError` on an unknown id |
| `run_algorithm.run_algorithm(name, v, w, C, cfg, k, optimum) -> dict` | Runs one algorithm with its repeats and builds the [result object](../../functional/output-format.md#instancesresultsid) (without `cost_ratio_vs_dp`) |
| `run_algorithm.adapter(name)` | the adapter `algo_<name>` of `algorithms/<name>/algo_<name>.py` (imported by name) |
| `add_ratios_vs_dp.add_ratios_vs_dp(results) -> None` | Adds `cost_ratio_vs_dp` in place ([spec](../../functional/metrics.md#ratios-to-dp)) |
| `summary.summarize(runs, algos, total_time) -> dict` | [Summary statistics](../../functional/metrics.md#summary-statistics) |
| `write_json.write_json(path, data) -> None` | JSON writer (see [README](README.md#json-writing)) |
| `printing.print_instance(r, algos, log=print)`, `printing.print_summary(s, algos, total_time, out_file, log=print)`, `printing.human_bytes(b)` | console output |

## Algorithm adapters (`algorithms/<id>/algo_<id>.py`)

`algo_<id>(v, w, C, acfg, seed) -> dict` for `dp`, `bnb`, `ilp`, `qubo`, `qubo_chz`; `acfg = cfg["<id>"]`, `seed = cfg["seed"] + 1000*k`. They call the algorithm and return `label`, `exact`, `proven_optimal`, `items`, `array_bytes`, `operations`, `operations_unit`, `details`. `algo_qubo_chz` is an independent copy of `algo_qubo` (uses only `qubo_chz.py`). Contract: [adding an algorithm](../adding-an-algorithm.md#adapter-contract).

## Algorithm settings (`algorithms/<id>/config_<id>.py`)

`config_<id>() -> dict` returns the default block `cfg["<id>"]` ([parameters](../../functional/parameters.md#algorithm-settings)); `benchmark_config` imports it by name for every id of `config.ALGORITHMS` (the registry).

## `config`

| Name | Description |
|---|---|
| `ALGORITHMS` | `("dp", "bnb", "ilp", "qubo", "qubo_chz")` |
| `benchmark_config(overrides=None) -> dict` | Deep copy of the [defaults](../../functional/parameters.md) with `overrides` merged recursively; `KeyError('Unknown option "a.b"')` on an unknown key |

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

`knapsack_dp(v, w, C) -> (best: float, items: list[int], info: dict)` — `info = {"operations": n*(C+1), "array_bytes": …}`.

## `bnb` — [spec](../../functional/algorithms/bnb.md)

`knapsack_bnb(v, w, C, max_nodes=math.inf) -> (best, items, info)` — `info = {"nodes", "proven_optimal", "array_bytes"}`.

## `ilp` — [spec](../../functional/algorithms/ilp.md)

`knapsack_ilp(v, w, C) -> (best, items, info)` — `info = {"solver": "scipy.optimize.milp (HiGHS)", "status", "proven_optimal", "nodes", "mip_gap", "array_bytes"}`; `nodes` / `mip_gap` are `None` if SciPy does not report them.

## `qubo` — [spec](../../functional/algorithms/qubo.md)

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
| `TabuStats` | dataclass: `iterations`, `restarts`, `tenure`, `move_evaluations`, `array_bytes` |

`opts` = the `tabu` parameter group plus `seed` (set by the runner to `seed + 1000·k`).

## `qubo_chz` — [spec](../../functional/algorithms/qubo_chz.md)

Same functions and signatures as `qubo`; a separate copy of the code. The runner calls it with the `qubo_chz` parameter group and seed `seed + 1000·k + seed_offset`.

## `measure` — [cost metrics](../../functional/metrics.md#execution-cost)

| Function | Description |
|---|---|
| `measure(fn, repeats, trace_memory=False) -> (result, cost: dict)` | runs `fn` (optional `tracemalloc` run, then `repeats` timed runs); `cost` keys: `repeats`, `wall_time_s`, `wall_time_min_s`, `cpu_time_s`, `traced_peak_memory_bytes`, `process_memory_before_bytes`, `process_memory_after_bytes`, `process_memory_delta_bytes` |
| `process_memory() -> int or None` | RSS in bytes, `None` without psutil |

## `cli`

`main(argv=None) -> int` — [command line](README.md#run); returns 0, or 2 on a configuration error.
