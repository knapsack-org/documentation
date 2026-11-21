# MATLAB / GNU Octave implementation — `knapsack-benchmark-matlab`

The original implementation, written in code that runs unchanged in **GNU Octave** and **MATLAB**. Behaviour: [functional specification](../../functional/README.md). Function-level reference: [api.md](api.md).

## Requirements

| Environment | Version | Notes |
|---|---|---|
| GNU Octave | ≥ 7 | needs `jsonencode`; `glpk` is built in. Tested with Octave 11.3 on Windows |
| MATLAB | ≥ R2021a | `jsonencode` with pretty printing. `ilp` needs the Optimization Toolbox (`intlinprog`) or a `glpk` function on the path, otherwise it is skipped with a warning. **Not yet run in MATLAB** — only Octave runs exist |

No toolbox is needed for `dp`, `bnb`, `qubo`, `qubo_chz`. MATLAB's own `knapsack2qubo` / `qubo.solve` (Quantum Computing support package) are **not** used: they do not exist in Octave; `qubo/modules/kb_knapsack2qubo` rebuilds the same model.

## Get the code

```bash
git clone https://github.com/knapsack-org/knapsack-benchmark-matlab.git
cd knapsack-benchmark-matlab
```

## Run

From a terminal in the project folder:

```bash
octave-cli --eval "run_benchmark"
octave-cli --eval "run_benchmark('req.json', 'instances.json', 'out');"   % each argument optional ('' = absent)
octave-cli --eval "run_benchmark('--describe');"                         % algorithms, default settings, JSON Schemas
```

or, in the Octave/MATLAB desktop, `cd` to the folder and call `run_benchmark`. A full default run of the 10 algorithms is ≈ 1,050 s of algorithm time in Octave (one process, after the 2026-11-21 speed-ups); knapsack-server splits the instances over several processes.

The arguments are a [run request](../../functional/output-format.md#run-request) file, an [instance set](../../functional/output-format.md#instance-set-file) used instead of the generator, and the output folder (default `output/run-dd-mm-yy-hh-mm/`). The run writes [`raw.json`](../../functional/output-format.md#raw-results-knapsack-raw-v1) and, without an instance set, `instances.json`; the run file (`schema_version` 4) is built by `pnpm -C <knapsack-server> knapsack-output <out dir>`. Without a request every algorithm found in `src/algorithms/` runs on every generated instance. To change a default [parameter](../../functional/parameters.md), edit the file that defines it:

| Parameter | File |
|---|---|
| `seed` | `benchmark_config.m` |
| `instances.*` | `src/instances/kb_config_instances.m` |
| `<id>.*` (e.g. `bnb.max_nodes`, `qubo.tabu.mode`, `qubo_chz.seed_offset`) | `src/algorithms/<id>/kp_algo_<id>_config.m` |

```matlab
raw = run_benchmark('', '', 'output/try');   % raw results; output/try/raw.json, output/try/instances.json
```

Errors at discovery (`kb_discover_algorithms:*`): missing contract file, loader id ≠ folder name, duplicate `.m` name across algorithm folders, not exactly one available exact reference. An unavailable algorithm (e.g. `ilp` without a solver) gives warning `run_benchmark:unavailable` and is listed in `skipped_algorithms`. An unknown key in a request is an error (`kb_merge_settings`).

## Project layout

```
knapsack-benchmark-matlab/
├── run_benchmark.m       entry point: discover → generate or read instances → run each algorithm (timed) → raw.json
├── benchmark_config.m    master seed + settings collected from src/instances and src/algorithms/<id>
├── src/
│   ├── helpers/          generic code, no algorithm names
│   │   ├── kb_discover_algorithms.m   scan src/algorithms/, check the folder contract, reference, availability
│   │   ├── kb_run_algorithm.m, kb_merge_settings.m, kb_phase_timer.m, kb_interpreter.m,
│   │   │   kb_print_instance.m, kb_print_progress.m   runner helpers, console and progress lines
│   │   ├── kb_seed.m, kb_lcg.m, kb_lcg_states.m, kb_randi.m      portable generator
│   │   ├── kb_bytes.m, kb_process_memory.m       memory measurement
│   │   └── kb_write_json.m
│   ├── instances/   kb_generate_instances.m (generator), kb_config_instances.m (settings cfg.instances)
│   └── algorithms/       one self-contained folder per algorithm, nothing else
│       ├── <id>/kp_algo_<id>_loader.m, _main.m, _config.m, _output.m, _output.schema.json (+ optional modules/)
│       ├── dp/ · bnb/      the four files only
│       ├── ilp/modules/    kb_ilp_solver_name.m (used by the loader's availability check)
│       ├── qubo/modules/   kb_penalty.m, kb_knapsack2qubo.m, kb_encode_slack.m, kb_qubo_tabu_slack.m, kb_qubo_tabu.m, kb_decode.m
│       └── qubo_chz/modules/   kb_chz_*.m: copies of the six QUBO modules
├── tests/test_p01.m
└── output/               run-*/raw.json, instances.json (git-ignored)
```

`run_benchmark` adds `src/` and all its subfolders to the path itself (`genpath`).

## Language-specific choices

### Numbers and indexing
- Native 1-based indexing; row vectors for instance data.
- `sort` is stable in MATLAB and Octave, as `bnb` requires; `min` returns the first index on ties.
- Item lists, weights and values are passed through `num2cell` before encoding so that a 1-element or empty list is still written as a JSON array.

### ILP solver
Chosen at run time by `kp_algo_ilp_main` (availability: `ilp/modules/kb_ilp_solver_name`, called by the loader):

| Condition | Solver | `proven_optimal` | `operations` |
|---|---|---|---|
| MATLAB and `intlinprog` exists | `intlinprog(-v, 1:n, w', C, [], [], 0, 1, opts)` with `Display = 'off'`, `RelativeGapTolerance = 0` | `exitflag == 1` | `output.numnodes` |
| otherwise, `glpk` exists (always in Octave) | `glpk(v, w', C, 0, 1, 'U', 'I…I', -1, struct('msglev', 0))` (maximise) | `errnum == 0` and `status == 5` | `null` (not exposed) |
| neither | the loader reports `ilp` unavailable: skipped with warning `run_benchmark:unavailable`, listed in `skipped_algorithms` | | |

`intlinprog` uses HiGHS from R2024a; earlier releases use MathWorks' legacy algorithm.

`RelativeGapTolerance` is set to 0 (its default is $10^{-4}$) so that `exitflag = 1` means a full proof, as the functional spec requires (fixed 2026-11-16; not yet run in MATLAB).

### Cost measurement
Done in `kb_run_algorithm` (`src/helpers/`):

| Metric | How |
|---|---|
| wall time | `tic`/`toc` around each repeat; `median` and `min` |
| CPU time | `cputime` over all repeats ÷ repeats |
| working memory | `whos` at the **end** of each `_main` function: sum of the bytes of every local variable (inputs included). `_main` takes unused arguments as `varargin`, not `~` (Octave would count an extra `ans`), and each adapter passes only the arguments its `_main` uses. For QUBO: max of the tabu function's `whos` and `kb_bytes(Q, c, d)` |
| process memory | `kb_process_memory` → `memory()`: `ram_used_octave` (Octave, Linux and Windows) or `MemUsedMATLAB` (MATLAB, Windows only); `NaN` → `null` otherwise |
| peak allocation | not available |

$\varepsilon$ = `eps` in the [ratios to the reference](../../functional/metrics.md#ratios-to-the-reference). Statistics use plain `mean`/`median`/`max`, so a `NaN` propagates (see [deviations](../README.md#differences-between-the-implementations)).

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
| every discovered algorithm on P01 through its adapter (exact → 309, heuristic → feasible), entry and details valid against its schemas | A1, A2, A4 (exact values), C5 (schemas) |
| schema validator rejects bad values | — |
| DP on P01 | A1 |
| B&B on P01, B&B with `max_nodes = 3` | A2, A3 |
| ILP on P01 | A4 |
| MathWorks example | B1, B2 |
| slack-optimal and all-bits tabu on P01 | A5, A6 |
| `qubo_chz` vs `qubo` | A7 |
| slack round trip 0…17 (both encoders) | B3 |

There is no MATLAB-side parity test: checks C1–C5 are run by the Python suite against the values of an Octave run.

## Measured cost (Octave 11.3, Windows)

Run `run-18-11-26-19-30.json` (214.1 s): ILP (GLPK) median 2.5 ms; DP 3.9 ms; B&B 7.9 ms median but 14.2 s on instance 96; QUBO 0.86 s. Ranking by median time: ilp < dp < bnb < qubo_chz < qubo. Side-by-side with Python: [implementation overview](../README.md#differences-between-the-implementations).
