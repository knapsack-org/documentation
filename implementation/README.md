# Implementation specification

How each language implements the [functional specification](../functional/README.md). These pages do **not** restate formulas or algorithm logic; they map each functional element to code and record what is language-specific.

| Implementation | Repository | Run | Pages |
|---|---|---|---|
| Python | `knapsack-benchmark-python` | `knapsack-benchmark` | [README](python/README.md) · [module reference](python/api.md) |
| MATLAB / GNU Octave | `knapsack-benchmark-matlab` | `run_benchmark` | [README](matlab/README.md) · [function reference](matlab/api.md) |

The MATLAB/Octave project is the original; the Python project is a line-by-line port that keeps the same parameter keys, the same random generator and the same JSON structure.

Each algorithm is a folder `algorithms/<id>/` holding the algorithm, its settings (`kb_config_<id>.m` / `config_<id>.py`) and its benchmark adapter (`kb_algo_<id>.m` / `algo_<id>.py`); to add one, follow [adding an algorithm](adding-an-algorithm.md).

## Traceability matrix

Both projects use the same layout: helpers in `helpers/`, each algorithm in `algorithms/<id>/` (MATLAB under `src/`, Python under `src/knapsack_benchmark/`). Python names below are module-relative: `instances` is in `instances/`; `rng`, `measure`, `run_algorithm`, `add_ratios_vs_dp`, `summary`, `write_json` are in `helpers/`; `dp.knapsack_dp` is `algorithms/dp/dp.py`, and likewise for `bnb`, `ilp`, `qubo`, `qubo_chz`.

| Functional element | Python (`src/knapsack_benchmark/`) | MATLAB / Octave |
|---|---|---|
| [Runner](../functional/README.md#benchmark-run-in-one-paragraph) | `benchmark.run_benchmark`, CLI `cli.main` | `run_benchmark.m` |
| [Parameters](../functional/parameters.md) | `config.benchmark_config`, `algorithms/<id>/config_<id>.py` | `benchmark_config.m`, `algorithms/<id>/kb_config_<id>.m` |
| [Portable generator](../functional/instances.md#portable-random-generator) | `rng.seed_state`, `rng.lcg`, `rng.randi`, `rng.mround` | `kb_seed`, `kb_lcg`, `kb_randi`, built-in `round` |
| [Instance families](../functional/instances.md#generation-procedure) | `instances/instances.generate_instances`, `instances/config_instances.config_instances` | `src/instances/`: `kb_generate_instances`, `kb_config_instances` |
| [`dp`](../functional/algorithms/dp.md) | `dp.knapsack_dp` | `kb_knapsack_dp` |
| [`bnb`](../functional/algorithms/bnb.md) | `bnb.knapsack_bnb` | `kb_knapsack_bnb` |
| [`ilp`](../functional/algorithms/ilp.md) | `ilp.knapsack_ilp` (SciPy `milp`, HiGHS) | `kb_knapsack_ilp` (`intlinprog` or `glpk`) |
| [QUBO model](../functional/algorithms/qubo.md#1-model) | `qubo.knapsack2qubo`, `qubo.slack_coefficients` | `kb_knapsack2qubo` |
| [Slack encoding](../functional/algorithms/qubo.md#12-slack-encoding) | `qubo.encode_slack` | `kb_encode_slack` |
| [Penalty](../functional/algorithms/qubo.md#14-penalty) | `qubo.penalty_value` | `kb_penalty` |
| [Tabu, slack-optimal](../functional/algorithms/qubo.md#2-solver--tabu-search-mode-slack-optimal-default) | `qubo.qubo_tabu_slack` | `kb_qubo_tabu_slack` |
| [Tabu, all-bits](../functional/algorithms/qubo.md#3-solver--tabu-search-mode-all-bits-comparison) | `qubo.qubo_tabu` | `kb_qubo_tabu` |
| [Decode / repair](../functional/algorithms/qubo.md#4-decoding-and-repair) | `qubo.decode` | `kb_decode` |
| [`qubo_chz`](../functional/algorithms/qubo_chz.md) | module `qubo_chz` (same function names) | `kb_chz_*` (same names with the `chz_` prefix) |
| [`greedy`](../functional/algorithms/greedy.md) | `algorithms/greedy/kp_algo_greedy_main.py` | `kp_algo_greedy_main` |
| [`sa`](../functional/algorithms/sa.md) | `algorithms/sa/kp_algo_sa_main.py` | `kp_algo_sa_main` |
| [`ga`](../functional/algorithms/ga.md) | `algorithms/ga/kp_algo_ga_main.py` | `kp_algo_ga_main` |
| [`ising`](../functional/algorithms/ising.md) — [model](../functional/algorithms/ising.md#1-model), [solver](../functional/algorithms/ising.md#2-solver--single-spin-flip-annealing), [decode](../functional/algorithms/ising.md#3-decoding-and-repair) | `algorithms/ising/kp_algo_ising_main.py`; `modules/ising.py`: `knapsack2ising`, `ising_anneal`, `decode` | `kp_algo_ising_main`; `modules/`: `kb_ising_model`, `kb_ising_anneal`, `kb_ising_decode` |
| [`fptas`](../functional/algorithms/fptas.md) | `algorithms/fptas/kp_algo_fptas_main.py` | `kp_algo_fptas_main` |
| [Quality metrics](../functional/metrics.md#quality) | `run_algorithm.run_algorithm` | `kb_run_algorithm` |
| [Cost metrics](../functional/metrics.md#execution-cost) | `measure.measure`, `measure.process_memory` | `kb_run_algorithm`, `kb_process_memory`, `kb_bytes`, `whos` |
| [Ratios to dp](../functional/metrics.md#ratios-to-dp) | `add_ratios_vs_dp.add_ratios_vs_dp` | `kb_add_ratios_vs_dp` |
| [Summary](../functional/metrics.md#summary-statistics) | `summary.summarize` | `kb_summary` |
| [Output file](../functional/output-format.md) | `write_json.write_json` | `kb_write_json` |
| [Acceptance tests](../functional/acceptance-tests.md) | `tests/test_p01.py`, `tests/test_parity.py` | `tests/test_p01.m` |

## Differences between the implementations

All are allowed by the functional spec unless marked **deviation**.

| Topic | Python | MATLAB / Octave |
|---|---|---|
| ILP solver | SciPy `milp` (HiGHS), `mip_rel_gap = 0` | `intlinprog` (MATLAB + Optimization Toolbox) or `glpk` (Octave); `ilp` skipped with a warning if neither exists |
| ILP `operations` | HiGHS node count | `intlinprog` node count; `null` with `glpk` |
| ILP gap tolerance | `mip_rel_gap = 0` | `glpk`: exact; `intlinprog`: `RelativeGapTolerance = 0` |
| `details.mip_gap` | yes | no |
| Working memory | sum of NumPy `nbytes` of the algorithm's arrays (QUBO search vectors estimated) | `whos` inside the function at its end |
| Process memory | `psutil` RSS (optional dependency) | `memory()`: Octave on Linux/Windows, MATLAB on Windows only |
| Peak allocation | `cost.traced_peak_memory_bytes` with `--trace-memory` (`tracemalloc`) | not available |
| Timers | `time.perf_counter`, `time.process_time` | `tic`/`toc`, `cputime` |
| $\varepsilon$ in ratios to dp | $10^{-12}$ | `eps` ($2.2\cdot10^{-16}$) |
| Memory ratio when $m_{dp} = 0$ | `null` | `Inf`/`NaN` → written as `null` |
| `null` values in statistics | ignored | `NaN` propagates (a single missing value makes the statistic `null`) — **deviation**, no effect today because a given algorithm reports either all or no `operations` |
| `environment` extras | `numpy`, `scipy` versions | none |
| Output folder | `--output-dir` / `output_dir` argument | always `<project>/output` |
| Configuration input | Python `dict`, JSON file, `--set key=value` | MATLAB `struct` |

Measured cost on the default run (orders of magnitude only; different machines):

| Algorithm | Python median time | Octave median time | Working memory (both) |
|---|---|---|---|
| `dp` | 0.21 ms | 2.7 ms | 171 kB |
| `bnb` | 0.50 ms | 6.0 ms | ≈ 2 kB |
| `ilp` | 12.6 ms (HiGHS) | 1.6 ms (GLPK) | 0.9 kB (problem data) |
| `qubo`, `qubo_chz` | 0.19 s | 0.64 s | 15 kB |
| whole run | ≈ 45–71 s | ≈ 153–200 s | |
