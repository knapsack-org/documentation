# MATLAB / GNU Octave implementation — `knapsack-benchmark-matlab`

The original implementation, written in code that runs unchanged in **GNU Octave** and **MATLAB**. Behaviour: [functional specification](../../functional/README.md). Function-level reference: [api.md](api.md).

## Requirements

| Environment | Version | Notes |
|---|---|---|
| GNU Octave | ≥ 7 | needs `jsonencode`; `glpk` is built in. Tested with Octave 11.3 on Windows |
| MATLAB | ≥ R2021a | `jsonencode` with pretty printing. `ilp` needs the Optimization Toolbox (`intlinprog`) or a `glpk` function on the path, otherwise it is skipped with a warning. **Not yet run in MATLAB** — only Octave runs exist |

No toolbox is needed for `dp`, `bnb`, `qubo`, `qubo_chz`. MATLAB's own `knapsack2qubo` / `qubo.solve` (Quantum Computing support package) are **not** used: they do not exist in Octave; `kb_knapsack2qubo` rebuilds the same model.

## Get the code

```bash
git clone https://github.com/knapsack-org/knapsack-benchmark-matlab.git
cd knapsack-benchmark-matlab
```

## Run

From a terminal in the project folder:

```bash
octave-cli --eval "run_benchmark"
```

or, in the Octave/MATLAB desktop, `cd` to the folder and call `run_benchmark`. A full default run takes ≈ 3 minutes in Octave (153 s, 200.5 s and 185.1 s measured).

Overrides are a `struct` with any [parameter](../../functional/parameters.md); nested groups are nested structs and are merged field by field:

```matlab
run_benchmark(struct('instances', struct('count', 10)))
run_benchmark(struct('algorithms', {{'dp', 'bnb', 'ilp'}}))     % double braces: a cell inside struct()
run_benchmark(struct('bnb', struct('max_nodes', 1e5)))
run_benchmark(struct('qubo', struct('tabu', struct('mode', 'all-bits'))))
run_benchmark(struct('qubo_chz', struct('tabu', struct('restarts', 50))))
result = run_benchmark();  result.output_file
```

Errors: `benchmark_config:unknownOption` (unknown parameter), `run_benchmark:unknownAlgorithm` (unknown id). The output folder is always `<project>/output`.

## Project layout

```
knapsack-benchmark-matlab/
├── run_benchmark.m       entry point: generate → run each algorithm → summary → JSON (no local functions)
├── benchmark_config.m    defaults and override merging
├── src/
│   ├── helpers/
│   │   ├── kb_ilp_solver_name.m, kb_run_algorithm.m,
│   │   │   kb_add_ratios_vs_dp.m, kb_summary.m, kb_penalty.m, kb_interpreter.m,
│   │   │   kb_print_instance.m, kb_print_summary.m, kb_human_bytes.m   runner helpers
│   │   ├── kb_seed.m, kb_lcg.m, kb_randi.m      portable generator
│   │   ├── kb_bytes.m, kb_process_memory.m       memory measurement
│   │   └── kb_write_json.m
│   ├── instances/   kb_generate_instances.m (generator), kb_config_instances.m (settings cfg.instances)
│   └── algorithms/
│       ├── algorithm_loader.m   registry, default settings, id check, adapter lookup
│       ├── each folder also holds its benchmark adapter kb_algo_<id>.m
│       ├── dp/kb_knapsack_dp.m · bnb/kb_knapsack_bnb.m · ilp/kb_knapsack_ilp.m
│       ├── qubo/        kb_knapsack2qubo.m, kb_encode_slack.m, kb_qubo_tabu_slack.m, kb_qubo_tabu.m, kb_decode.m
│       └── qubo_chz/    kb_chz_*.m: copies of the five QUBO files
├── tests/test_p01.m
└── output/               run-*.json (git-ignored)
```

`run_benchmark` adds `src/` and all its subfolders to the path itself (`genpath`).

## Language-specific choices

### Numbers and indexing
- Native 1-based indexing; row vectors for instance data.
- `sort` is stable in MATLAB and Octave, as `bnb` requires; `min` returns the first index on ties.
- Item lists, weights and values are passed through `num2cell` before encoding so that a 1-element or empty list is still written as a JSON array.

### ILP solver
Chosen at run time by `kb_knapsack_ilp`:

| Condition | Solver | `proven_optimal` | `operations` |
|---|---|---|---|
| MATLAB and `intlinprog` exists | `intlinprog(-v, 1:n, w', C, [], [], 0, 1, opts)` with `Display = 'off'`, `RelativeGapTolerance = 0` | `exitflag == 1` | `output.numnodes` |
| otherwise, `glpk` exists (always in Octave) | `glpk(v, w', C, 0, 1, 'U', 'I…I', -1, struct('msglev', 0))` (maximise) | `errnum == 0` and `status == 5` | `null` (not exposed) |
| neither | `ilp` removed from `algorithms` with warning `run_benchmark:noIlp` | | |

`intlinprog` uses HiGHS from R2024a; earlier releases use MathWorks' legacy algorithm.

`RelativeGapTolerance` is set to 0 (its default is $10^{-4}$) so that `exitflag = 1` means a full proof, as the functional spec requires (fixed 2026-11-16; not yet run in MATLAB).

### Cost measurement
Done in `kb_run_algorithm` (`src/helpers/`):

| Metric | How |
|---|---|
| wall time | `tic`/`toc` around each repeat; `median` and `min` |
| CPU time | `cputime` over all repeats ÷ repeats |
| working memory | `whos` at the **end** of each algorithm function: sum of the bytes of every local variable (inputs included). For QUBO: max of the tabu function's `whos` and `kb_bytes(Q, c, d)` |
| process memory | `kb_process_memory` → `memory()`: `ram_used_octave` (Octave, Linux and Windows) or `MemUsedMATLAB` (MATLAB, Windows only); `NaN` → `null` otherwise |
| peak allocation | not available |

$\varepsilon$ = `eps` in the [ratios to dp](../../functional/metrics.md#ratios-to-dp). Statistics use plain `mean`/`median`/`max`, so a `NaN` propagates (see [deviations](../README.md#differences-between-the-implementations)).

### JSON writing
`kb_write_json` uses `jsonencode(data, 'PrettyPrint', true)` (falls back to compact output if pretty printing is unsupported); `NaN`/`Inf` become `null`. `created` and the file name come from `datestr(now, …)`.

`environment.interpreter` is `GNU Octave` or `MATLAB`; `environment.computer` is `computer()`.

## Tests

```bash
octave-cli --eval "addpath('tests'); test_p01"
```

Prints `test_p01: all checks passed`. The ILP check is skipped (with a message) when no solver is found.

| Assertion block in `test_p01.m` | [Acceptance check](../../functional/acceptance-tests.md) |
|---|---|
| DP on P01 | A1 |
| B&B on P01, B&B with `max_nodes = 3` | A2, A3 |
| ILP on P01 | A4 |
| MathWorks example | B1, B2 |
| slack-optimal and all-bits tabu on P01 | A5, A6 |
| `qubo_chz` vs `qubo` | A7 |
| slack round trip 0…17 (both encoders) | B3 |

There is no MATLAB-side parity test: checks C1–C5 are run by the Python suite against the values of an Octave run.

## Measured cost (Octave 11.3, Windows)

ILP (GLPK) median 1.7 ms; DP 2.7 ms; B&B 5.6 ms median but 10.9 s on instance 96; QUBO 0.62 s. Ranking by median time: ilp < dp < bnb < qubo. Side-by-side with Python: [implementation overview](../README.md#differences-between-the-implementations).
