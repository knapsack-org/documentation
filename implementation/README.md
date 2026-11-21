# Implementation specification

How each language implements the [functional specification](../functional/README.md). These pages do **not** restate formulas or algorithm logic; they map each functional element to code and record what is language-specific.

| Implementation | Repository | Run | Pages |
|---|---|---|---|
| Python | `knapsack-benchmark-python` | `knapsack-benchmark` | [README](python/README.md) · [module reference](python/api.md) |
| MATLAB / GNU Octave | `knapsack-benchmark-matlab` | `run_benchmark` | [README](matlab/README.md) · [function reference](matlab/api.md) |

The MATLAB/Octave project is the original; the Python project is a line-by-line port that keeps the same parameter keys, the same random generator and the same JSON structure.

Each algorithm is a self-contained folder `algorithms/<id>/` with four files, the same names in both languages: `kp_algo_<id>_loader` (identity, availability), `kp_algo_<id>_main` (the maths), `kp_algo_<id>_config` (settings) and `kp_algo_<id>_output` (benchmark adapter, `algorithms[]` entry, JSON Schemas), plus optional `modules/`. The runner finds the algorithms by scanning the folder, so adding one touches only its folder: [adding an algorithm](adding-an-algorithm.md). Neither project takes run options.

## Traceability matrix

Both projects use the same layout: helpers in `helpers/`, each algorithm in `algorithms/<id>/` (MATLAB under `src/`, Python under `src/knapsack_benchmark/`). Python names below are module-relative: `instances` is in `instances/`; `rng`, `measure`, `discover`, `phase_timer`, `run_algorithm`, `printing`, `write_json` are in `helpers/`; `kp_algo_dp_main` is `algorithms/dp/kp_algo_dp_main.py`, and likewise for every algorithm; `qubo.*` is `algorithms/qubo/modules/qubo.py`.

| Functional element | Python (`src/knapsack_benchmark/`) | MATLAB / Octave |
|---|---|---|
| [Runner](../functional/README.md#benchmark-run-in-one-paragraph) | `benchmark.run_benchmark`, CLI `cli.main` | `run_benchmark.m` |
| [Algorithm discovery](../functional/parameters.md#algorithms) | `discover.discover_algorithms`, loaders `kp_algo_<id>_loader` | `kb_discover_algorithms`, loaders `kp_algo_<id>_loader` |
| [Output schemas](../functional/output-format.md) | `algorithms/<id>/kp_algo_<id>_output.schema.json` (byte-identical in both; checked by knapsack-server's output service) | same file |
| [Parameters](../functional/parameters.md) | `config.benchmark_config`, `algorithms/<id>/kp_algo_<id>_config.py` | `benchmark_config.m`, `algorithms/<id>/kp_algo_<id>_config.m` |
| [Portable generator](../functional/instances.md#portable-random-generator) | `rng.seed_state`, `rng.lcg`, `rng.randi`, `rng.mround` | `kb_seed`, `kb_lcg`, `kb_randi`, built-in `round` |
| [Instance families](../functional/instances.md#generation-procedure) | `instances/instances.generate_instances`, `instances/config_instances.config_instances` | `src/instances/`: `kb_generate_instances`, `kb_config_instances` |
| [`dp`](../functional/algorithms/dp.md) | `kp_algo_dp_main` | `kp_algo_dp_main` |
| [`bnb`](../functional/algorithms/bnb.md) | `kp_algo_bnb_main` | `kp_algo_bnb_main` |
| [`ilp`](../functional/algorithms/ilp.md) | `kp_algo_ilp_main` (SciPy `milp`, HiGHS) | `kp_algo_ilp_main` (`intlinprog` or `glpk`; `modules/kb_ilp_solver_name`) |
| [`qubo` pipeline](../functional/algorithms/qubo.md) | `kp_algo_qubo_main` (phases marked for `details.*_time_s`) | `kp_algo_qubo_main` |
| [QUBO model](../functional/algorithms/qubo.md#1-model) | `qubo.knapsack2qubo`, `qubo.slack_coefficients` | `modules/kb_knapsack2qubo` |
| [Slack encoding](../functional/algorithms/qubo.md#12-slack-encoding) | `qubo.encode_slack` | `modules/kb_encode_slack` |
| [Penalty](../functional/algorithms/qubo.md#14-penalty) | `qubo.penalty_value` | `modules/kb_penalty` |
| [Tabu, slack-optimal](../functional/algorithms/qubo.md#2-solver--tabu-search-mode-slack-optimal-default) | `qubo.qubo_tabu_slack` | `modules/kb_qubo_tabu_slack` |
| [Tabu, all-bits](../functional/algorithms/qubo.md#3-solver--tabu-search-mode-all-bits-comparison) | `qubo.qubo_tabu` | `modules/kb_qubo_tabu` |
| [Decode / repair](../functional/algorithms/qubo.md#4-decoding-and-repair) | `qubo.decode` | `modules/kb_decode` |
| [`greedy`](../functional/algorithms/greedy.md) | `algorithms/greedy/kp_algo_greedy_main.py` | `kp_algo_greedy_main` |
| [`sa`](../functional/algorithms/sa.md) | `algorithms/sa/kp_algo_sa_main.py` | `kp_algo_sa_main` |
| [`ga`](../functional/algorithms/ga.md) | `algorithms/ga/kp_algo_ga_main.py` | `kp_algo_ga_main` |
| [`ising`](../functional/algorithms/ising.md) — [model](../functional/algorithms/ising.md#1-model), [solver](../functional/algorithms/ising.md#2-solver--single-spin-flip-annealing), [decode](../functional/algorithms/ising.md#3-decoding-and-repair) | `algorithms/ising/kp_algo_ising_main.py`; `modules/ising.py`: `knapsack2ising`, `ising_anneal`, `decode` | `kp_algo_ising_main`; `modules/`: `kb_ising_model`, `kb_ising_anneal`, `kb_ising_decode` |
| [`fptas`](../functional/algorithms/fptas.md) | `algorithms/fptas/kp_algo_fptas_main.py` | `kp_algo_fptas_main` |
| [Quality metrics](../functional/metrics.md#quality) | `run_algorithm.run_algorithm` | `kb_run_algorithm` |
| [Cost metrics](../functional/metrics.md#execution-cost) | `measure.measure`, `measure.process_memory` | `kb_run_algorithm`, `kb_process_memory`, `kb_bytes`, `whos` |
| [Ratios to the reference](../functional/metrics.md#ratios-to-the-reference), [summary](../functional/metrics.md#summary-statistics), `is_optimal`, `gap_percent` | none: knapsack-server `libs/output`, once for both | none |
| [Raw results](../functional/output-format.md#raw-results-knapsack-raw-v1) | `benchmark.run_benchmark`, `write_json.write_json` | `run_benchmark`, `kb_write_json` |
| [Run request](../functional/output-format.md#run-request), [instance set](../functional/output-format.md#instance-set-file) | `cli.main`, `config.benchmark_config` | `run_benchmark`, `benchmark_config`, `kb_merge_settings` |
| [Run file](../functional/output-format.md) (`schema_version` 4) | knapsack-server `knapsack-output` | knapsack-server `knapsack-output` |
| [Acceptance tests](../functional/acceptance-tests.md) | `tests/test_p01.py`, `tests/test_parity.py` | `tests/test_p01.m` |

## Differences between the implementations

All are allowed by the functional spec unless marked **deviation**.

| Topic | Python | MATLAB / Octave |
|---|---|---|
| ILP solver | SciPy `milp` (HiGHS), `mip_rel_gap = 0`; always available | `intlinprog` (MATLAB + Optimization Toolbox) or `glpk` (Octave); the loader reports `ilp` unavailable if neither exists (skipped with a warning, listed in `skipped_algorithms`) |
| ILP `operations` | HiGHS node count | `intlinprog` node count; `null` with `glpk` |
| ILP gap tolerance | `mip_rel_gap = 0` | `glpk`: exact; `intlinprog`: `RelativeGapTolerance = 0` |
| `details.mip_gap` | yes | no |
| Working memory | sum of NumPy `nbytes` of the algorithm's arrays (QUBO search vectors estimated) | `whos` inside the function at its end |
| Process memory | `psutil` RSS (optional dependency) | `memory()`: Octave on Linux/Windows, MATLAB on Windows only |
| Peak allocation | `cost.traced_peak_memory_bytes` with `timing.trace_memory = True` in `config.py` (`tracemalloc`) | not available |
| Timers | `time.perf_counter`, `time.process_time` | `tic`/`toc`, `cputime` |
| $\varepsilon$ in ratios to the reference | $10^{-12}$ | `eps` ($2.2\cdot10^{-16}$) |
| Memory ratio when $m_{ref} = 0$ | `null` | `Inf`/`NaN` → written as `null` |
| `null` values in statistics | ignored | `NaN` propagates (a single missing value makes the statistic `null`) — **deviation**, no effect today because a given algorithm reports either all or no `operations` |
| `environment` extras | `numpy`, `scipy` versions | none |
| Output folder | always `<project>/output` | always `<project>/output` |
| Settings | edited in the `.py` files (`config.py`, `instances/config_instances.py`, `kp_algo_<id>_config.py`); the CLI rejects any argument (exit 2) | edited in the `.m` files (`benchmark_config.m`, `src/instances/kb_config_instances.m`, `kp_algo_<id>_config.m`); `run_benchmark()` takes no argument |
| Duplicate module names | impossible (packages) | an error at discovery (one shared path) |

Measured cost on the default run (orders of magnitude only; different machines):

| Algorithm | Python median time | Octave median time | Working memory (both) |
|---|---|---|---|
| `dp` | 0.56 ms | 3.9 ms | 171 kB |
| `bnb` | 0.88 ms | 7.9 ms | ≈ 2 kB |
| `ilp` | 18 ms (HiGHS) | 2.5 ms (GLPK) | 0.9 kB (problem data) |
| `qubo` | 0.30 s | 0.81–0.86 s | 15 kB |
| whole run | 68 s | 214 s | |

Since `schema_version` 4 (2026-11-20), knapsack-server's tests rebuild these runs' run files from their raw part and match every computed field (parity (b)); the Knapview fixtures are those rebuilt v4 files. Reference runs of 2026-11-18 (`schema_version` 3): Octave 11.3 `run-18-11-26-19-30.json`, Python 3.14.5 `run-18-11-26-19-32.json`, Windows test machine, five core algorithms.
