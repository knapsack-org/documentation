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
knapsack-benchmark [--request req.json] [--instances inst.json] [--out DIR]   # or: python -m knapsack_benchmark ...
knapsack-benchmark --describe        # algorithms, default settings and JSON Schemas as JSON
```

`--request` is a [run request](../../functional/output-format.md#run-request), `--instances` an [instance set](../../functional/output-format.md#instance-set-file) used instead of the generator, and `--out` the output folder (default `output/run-dd-mm-yy-hh-mm/`). The run writes [`raw.json`](../../functional/output-format.md#raw-results-knapsack-raw-v1) and, without `--instances`, `instances.json`; the run file (`schema_version` 4) is built by `pnpm -C <knapsack-server> knapsack-output <out dir>`. Without a request every algorithm found in `algorithms/` runs on every generated instance. To change a default [parameter](../../functional/parameters.md), edit the file that defines it:

| Parameter | File (under `src/knapsack_benchmark/`) |
|---|---|
| `seed`, `timing.trace_memory` | `config.py` |
| `instances.*` | `instances/config_instances.py` |
| `<id>.*` (e.g. `bnb.max_nodes`, `qubo.tabu.mode`) | `algorithms/<id>/kp_algo_<id>_config.py` |

From Python:

```python
from knapsack_benchmark import run_benchmark
raw = run_benchmark({"instances": {"count": 5}}, out_dir="output/try")   # raw results
raw["instances"][0]["results"]["qubo"]["value"]
```

Errors: `helpers.discover.DiscoveryError` (bad folder name, missing contract file, loader id ≠ folder name, not exactly one available exact reference), `ValueError` (unknown key in a request). An unavailable algorithm is skipped with a warning and listed in `skipped_algorithms`.

`run_benchmark` returns the dict written to `raw.json`.

## Project layout

```
knapsack-benchmark-python/
├── pyproject.toml
├── src/knapsack_benchmark/
│   ├── __init__.py     exports run_benchmark, benchmark_config; __version__
│   ├── __main__.py     python -m knapsack_benchmark
│   ├── cli.py          command line (--request, --instances, --out, --describe)
│   ├── benchmark.py    runner: discover → generate or read instances → run each algorithm (measured) → raw.json
│   ├── config.py       master seed, timing option, settings collected from instances/ and algorithms/<id>
│   ├── helpers/        generic code, no algorithm names
│   │   ├── discover.py     scan algorithms/, check the folder contract, reference, availability
│   │   ├── run_algorithm.py, phase_timer.py, write_json.py, printing.py      runner helpers
│   │   ├── rng.py          portable Park–Miller generator
│   │   └── measure.py      time / CPU / memory measurement
│   ├── instances/      instances.py (generator), config_instances.py (settings cfg["instances"])
│   └── algorithms/     one self-contained package per algorithm
│       ├── <id>/kp_algo_<id>_loader.py, _main.py, _config.py, _output.py, _output.schema.json (+ optional modules/)
│       ├── bnb/ · dp/ · ilp/     the four files only
│       ├── qubo/modules/qubo.py  QUBO model, slack encoding, both tabu searches, decode, penalty
│       ├── qubo_b/modules/qubo.py  copy (QUBO variant B)
│       └── qc/modules/       qubo.py (copy), qaoa.py (simulated QAOA), greedy.py (copy of greedy)
├── tests/test_p01.py, test_parity.py, test_heuristics.py, test_qc.py
└── output/             run-*/raw.json, instances.json (git-ignored)
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
| working memory | `_main` returns `working_memory_bytes` = sum of `ndarray.nbytes` of its arrays. QUBO search vectors are **estimated** (slack-optimal: $10n \times 8$ bytes; all-bits: $6N \times 8$ bytes), then the maximum with the model size $Q + c + d$ is kept |
| process memory | `psutil.Process().memory_info().rss` before/after; `null` without psutil |
| peak allocation (Python only) | with `timing.trace_memory = True` in `config.py`: one extra **untimed** run under `tracemalloc`, peak written to `cost.traced_peak_memory_bytes` (key always present, `null` when off) |

$\varepsilon = 10^{-12}$ in the [ratios to the reference](../../functional/metrics.md#ratios-to-the-reference). Statistics skip `None` values (`benchmark._stat`).

### JSON writing
`helpers/write_json._clean` converts NumPy scalars/arrays to Python types, `NaN`/`±∞` to `null`, and integral floats below $2^{53}$ to `int`; `write_json` writes UTF-8 with 2-space indentation and creates the folder.

Python-only output keys: `environment.numpy`, `environment.scipy`, `details.mip_gap` (ilp), `cost.traced_peak_memory_bytes`.

## Tests

```bash
pytest                   # everything, including one full 100-instance run (≈ 1–3 min)
pytest -m "not slow"     # without the full run
```

| Test | [Acceptance check](../../functional/acceptance-tests.md) |
|---|---|
| `test_p01.py::test_every_algorithm_on_p01` (every discovered algorithm through its adapter, with its schemas) | A1, A2, A4 (exact values), C5 (schemas) |
| `test_p01.py::test_schema_validator_rejects` | — |
| `test_p01.py::test_exact_solvers_p01` (dp, bnb, ilp) | A1, A2, A4 |
| `test_p01.py::test_bnb_node_limit` | A3 |
| `test_p01.py::test_mathworks_knapsack2qubo_example` | B1, B2 |
| `test_p01.py::test_qubo_tabu_p01` | A5, A6 |
| `test_p01.py::test_slack_encoding` (qubo) | B3 |
| `test_p01.py::test_instances_match_matlab` | C1 (first instance: n = 10, C = 285, optimum 345) |
| `test_qc.py` (correlation settings, `qubo_b` / `qc` = `qubo`, QC greedy and QAOA modules) | D1–D6 |
| `test_parity.py::test_bnb_nodes_and_optimum` | C2 |
| `test_parity.py::test_full_run_matches_octave` (`slow`; writes `output/run-*.json` and deletes it) | C3 (QUBO values of all 100 instances, for `qubo`, `qubo_b` and `qc`), D3, C4, C5 (version, order, reference), expected results |

The expected values in `test_parity.py` come from the Octave runs; they are unchanged in the v2 runs `run-16-11-26-16-38.json` / `run-16-11-26-16-39.json` and the v3 reference runs `run-18-11-26-19-30.json` (Octave) / `run-18-11-26-19-32.json` (Python). Run files are not in git (`output/*.json` is ignored).

## Measured cost

Default run, 100 instances, five core algorithms: ≈ 45 s in a Linux container, 57–71 s on the Windows test machine (68.0 s for `run-18-11-26-19-32.json`). Most of the time goes to the two QUBO tabu searches. Per-algorithm figures: [implementation overview](../README.md#differences-between-the-implementations).
