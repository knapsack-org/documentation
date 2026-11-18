# Python implementation — `knapsack-benchmark-python`

Python port of the MATLAB/Octave benchmark. Behaviour: [functional specification](../../functional/README.md). Function-level reference: [api.md](api.md).

## Requirements

| | |
|---|---|
| Python | ≥ 3.10 |
| Required | `numpy >= 1.24`, `scipy >= 1.9` (`scipy.optimize.milp`) |
| Optional extras | `memory` → `psutil >= 5.9` (process memory, else `null`); `test` → `pytest >= 7` |
| Packaging | `pyproject.toml`, build backend `hatchling`, package `knapsack-benchmark` 0.1.0, import name `knapsack_benchmark`, licence MIT |

## Install

```bash
git clone https://github.com/knapsack-org/knapsack-benchmark-python.git
cd knapsack-benchmark-python
python -m venv .venv
.venv\Scripts\activate          # Windows; Linux/macOS: source .venv/bin/activate
pip install -e ".[memory,test]"
```

## Run

```bash
knapsack-benchmark                          # or: python -m knapsack_benchmark
```

| Option | Effect |
|---|---|
| `-n`, `--num-instances N` | `instances.count` |
| `--seed S` | `seed` |
| `-a`, `--algorithms dp,bnb,…` | `algorithms` (comma-separated) |
| `-o`, `--output-dir DIR` | output folder (default `<project>/output`) |
| `--config FILE.json` | overrides from a JSON file (any [parameter](../../functional/parameters.md)) |
| `--set KEY=VALUE` | one override, dotted keys for nested groups, value parsed as JSON (else kept as a string); repeatable |
| `--trace-memory` | `timing.trace_memory = true` (≈ 6× slower) |
| `-q`, `--quiet` | no console output |

Precedence: `--config` file, then each `--set`, then `-n`, `--seed`, `-a`, `--trace-memory`. An unknown parameter or algorithm prints `error: …` and exits with code **2**.

```bash
knapsack-benchmark -n 10 -a dp,bnb,ilp
knapsack-benchmark --set qubo_chz.tabu.restarts=50 --set bnb.max_nodes=100000
```

From Python:

```python
from knapsack_benchmark import run_benchmark, benchmark_config
res = run_benchmark({"instances": {"count": 10}, "qubo_chz": {"tabu": {"restarts": 50}}},
                    output_dir="results", verbose=False)
res["summary"]["algorithms"]["qubo"]["mean_gap_percent"], res["output_file"]
```

`run_benchmark` returns the same dict that is written to JSON, plus `output_file`.

## Project layout

```
knapsack-benchmark-python/
├── pyproject.toml
├── src/knapsack_benchmark/
│   ├── __init__.py     exports run_benchmark, benchmark_config; __version__
│   ├── __main__.py     python -m knapsack_benchmark
│   ├── cli.py          command line
│   ├── benchmark.py    runner: generate → run each algorithm → summary → JSON
│   ├── config.py       defaults and override merging
│   ├── helpers/
│   │   ├── run_algorithm.py, add_ratios_vs_dp.py,
│   │   │   summary.py, write_json.py, printing.py      runner helpers
│   │   ├── rng.py          portable Park–Miller generator
│   │   └── measure.py      time / CPU / memory measurement
│   ├── instances/      instances.py (generator), config_instances.py (settings cfg["instances"])
│   └── algorithms/
│       ├── algorithm_loader.py   registry, default settings, id check, adapter lookup
│       ├── each folder also holds its benchmark adapter algo_<id>.py
│       ├── dp/dp.py · bnb/bnb.py · ilp/ilp.py
│       ├── qubo/qubo.py    QUBO model, slack encoding, both tabu searches, decode, penalty
│       └── qubo_chz/qubo_chz.py   independent copy of qubo.py
├── tests/test_p01.py, tests/test_parity.py
└── output/             run-*.json (git-ignored)
```

## Language-specific choices

### Numbers and indexing
- All arrays are `float64` NumPy arrays (integers stored as floats, as in MATLAB) so the arithmetic matches Octave bit for bit.
- Algorithms work 0-based internally and convert to **1-based** item lists at their boundary.
- `rng.mround` reproduces MATLAB `round` (half away from zero) for the non-negative values used; Python's built-in `round` (half to even) is never used for spec values.
- Stable sort for `bnb`: `np.argsort(..., kind="stable")`; lowest-index tie-breaks come from `np.argmin`.

### ILP solver
`scipy.optimize.milp(c=-v, integrality=1, bounds=[0,1], constraints=w·x ≤ C, options={"mip_rel_gap": 0})` — HiGHS, the engine MATLAB `intlinprog` uses since R2024a.
- `proven_optimal` ⇔ `res.status == 0`; `details.status` is SciPy's status code.
- `operations` = `res.mip_node_count`; `details.mip_gap` = `res.mip_gap`.
- Some HiGHS builds print debug lines from C++; the call is wrapped in `_silence_native_output()`, which redirects file descriptors 1 and 2 to `os.devnull` (falls back to no redirection when the process has no real descriptors, e.g. in some IDEs).

### Cost measurement
Implemented in `measure.measure(fn, repeats, trace_memory)`:

| Metric | How |
|---|---|
| wall time | `time.perf_counter` around each repeat; median and min |
| CPU time | `time.process_time` over all repeats ÷ repeats |
| working memory | the algorithm returns `array_bytes` = sum of `ndarray.nbytes` of its arrays. QUBO search vectors are **estimated** (slack-optimal: $10n \times 8$ bytes; all-bits: $6N \times 8$ bytes), then the maximum with the model size $Q + c + d$ is kept |
| process memory | `psutil.Process().memory_info().rss` before/after; `null` without psutil |
| peak allocation (Python only) | with `timing.trace_memory`: one extra **untimed** run under `tracemalloc`, peak written to `cost.traced_peak_memory_bytes` (key always present, `null` when off) |

$\varepsilon = 10^{-12}$ in the [ratios to dp](../../functional/metrics.md#ratios-to-dp). Statistics skip `None` values (`benchmark._stat`).

### JSON writing
`helpers/write_json._clean` converts NumPy scalars/arrays to Python types, `NaN`/`±∞` to `null`, and integral floats below $2^{53}$ to `int`; `write_json` writes UTF-8 with 2-space indentation and creates the folder.

Python-only output keys: `environment.numpy`, `environment.scipy`, `details.mip_gap` (ilp), `cost.traced_peak_memory_bytes`.

## Tests

```bash
pytest                   # everything, ≈ 50 s
pytest -m "not slow"     # without the full 100-instance run, ≈ 3 s
```

| Test | [Acceptance check](../../functional/acceptance-tests.md) |
|---|---|
| `test_p01.py::test_exact_solvers_p01` (dp, bnb, ilp) | A1, A2, A4 |
| `test_p01.py::test_bnb_node_limit` | A3 |
| `test_p01.py::test_mathworks_knapsack2qubo_example` | B1, B2 |
| `test_p01.py::test_qubo_tabu_p01` | A5, A6 |
| `test_p01.py::test_qubo_chz_matches_qubo` | A7 |
| `test_p01.py::test_slack_encoding` (qubo, qubo_chz) | B3 |
| `test_p01.py::test_instances_match_matlab` | C1 (first instance: n = 10, C = 285, optimum 345) |
| `test_parity.py::test_bnb_nodes_and_optimum` | C2 |
| `test_parity.py::test_full_run_matches_octave` (`slow`) | C3 (QUBO values of all 100 instances), C4, C5, expected results |

Known issue: `test_parity.py` and the project README name the Octave reference file `run-14-11-26-14-00.json`; the current reference runs are `run-16-11-26-16-38.json` (Octave) and `run-16-11-26-16-39.json` (Python), `schema_version` 2, same values. The test embeds the expected values, so it still passes. Run files are not in git (`output/*.json` is ignored).

## Measured cost

Default run, 100 instances: ≈ 45 s in a Linux container, 62.8 s, 70.7 s and 57.3 s on the Windows test machine. Most of the time goes to the two QUBO tabu searches. Per-algorithm figures: [implementation overview](../README.md#differences-between-the-implementations).
