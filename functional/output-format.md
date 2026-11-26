# Output format (JSON, `schema_version` 4)

The run file is one UTF-8 JSON file. It is the contract between the two benchmark implementations, [knapsack-server](https://github.com/knapsack-org/knapsack-server) and the [Knapview](https://github.com/knapsack-org/knapview) viewer.

Since version 4 the benchmarks no longer write it themselves. A run writes [raw results](#raw-results-knapsack-raw-v1) (`raw.json`) and the [instance set](#instance-set-file) (`instances.json`). knapsack-server's output service builds the run file from them and adds the optimum, the gaps, the ratios and the summary:

- **Standalone run:** `pnpm -C <knapsack-server> knapsack-output <out dir>` writes `run-dd-mm-yy-hh-mm.json` (day-month-year-hour-minute of the end of the run).
- **Server run:** one file per environment, `runs/<runId>/<env>.json.gz`, plus `instances.json.gz` and `manifest.json`.

## Versioning

- `schema_version` (integer) identifies the structure; current value **4**, the same for both environments.
- It must be increased in **both** implementations on any breaking change (renamed/removed field, changed type or meaning). Adding an optional field is not breaking.
- Readers reject a version they do not support. Knapview reads only version 4 (older files must be re-run).
- History: **1** (2026-11-14) first version. **2** (2026-11-16) `algorithms` is a list of objects with each algorithm's settings; per-algorithm parameter blocks; `instance_config` and the `instances` parameter block ([renamed keys](parameters.md#keys-renamed-in-schema_version-2)). **3** (2026-11-18) algorithms found by scanning their folders and listed in alphabetical order; `algorithms[]` gains `description`, `short_label` and `reference`, labels are capitalised; new `skipped_algorithms`; `environment.ilp_solver` → `environment.algorithms.ilp.solver`; ratios renamed `*_vs_dp` → `*_vs_reference`; generated `description`. **4** (2026-11-20) built by knapsack-server's output service from raw results (the benchmarks no longer compute `is_optimal`, `gap_percent`, ratios or the summary); optional [`server`](#server) and [`instance_set`](#instance_set) blocks; `summary.total_time_seconds` is the execution wall clock. JSON Schema (2020-12): `knapsack-server/libs/contracts/schemas/run-v4.schema.json`.

Each algorithm describes its own blocks with a JSON Schema file, `algorithms/<id>/kp_algo_<id>_output.schema.json`: its `algorithms[]` entry (`entry`), its settings (`config`) and its `results.<id>.details` (`details`). The file is byte-identical in both implementations (knapsack-server CI checks this). The output service validates every run file against these schemas before writing it ([adding an algorithm](../implementation/adding-an-algorithm.md#output-contract--kp_algo_id_output)).

## Encoding rules

- Missing or undefined numbers (`NaN`, `±∞`) are written as `null`.
- Integral numbers are written without a decimal part.
- Item lists are arrays of 1-based indices (possibly empty `[]`).
- Booleans are JSON `true` / `false`.

## Top level

| Field | Type | Meaning |
|---|---|---|
| `project` | string | `knapsack-benchmark-matlab` or `knapsack-benchmark-python` |
| `schema_version` | int | `4` |
| `algorithms` | object[] | algorithms actually run, in alphabetical order of their ids: [below](#algorithms) (version 1: string[] of ids) |
| `skipped_algorithms` | object[] | `{id, reason}` of each algorithm that could not run in this environment (e.g. `ilp` without a solver); `[]` when all ran |
| `instance_config` | object | settings of the instance generator: [below](#instance_config) (not in version 1) |
| `reference_algorithm` | string | id of the reference algorithm, whose value is the optimum $V^*$ (`"dp"`) |
| `description` | string | human-readable summary, built from the labels and the reference id |
| `created` | string | local date-time `YYYY-MM-DDTHH:MM:SS` |
| `environment` | object | [below](#environment) |
| `parameters` | object | the full resolved [parameters](parameters.md) |
| `summary` | object | [below](#summary) |
| `instances` | object[] | [below](#instances) |
| `server` | object | optional: present in server runs only, [below](#server) |
| `instance_set` | object | optional: [below](#instance_set) |

### `algorithms`

One object per algorithm run, in alphabetical order of the ids:

| Field | Type | Meaning |
|---|---|---|
| `id` | string | algorithm id, key of `summary.algorithms` and `instances[].results` |
| `label` | string | human-readable name, main words capitalised (same as `results.<id>.algorithm`), e.g. `Branch and Bound (Dantzig Bound, Depth-First)` |
| `description` | string | one or two sentences on the method |
| `short_label` | string | short name for compact displays, e.g. `BnB` |
| `exact` | bool | the method can prove optimality |
| `reference` | bool | `true` for the reference algorithm only |
| `config` | object | the resolved settings block of the algorithm (same as `parameters.<id>`, see [parameters](parameters.md#algorithm-settings)) |

### `instance_config`

The inputs of the [instance generator](instances.md#generation-procedure), so the instances can be regenerated:

| Field | Type | Meaning |
|---|---|---|
| `seed` | int | master seed (same as `parameters.seed`) |
| `count`, `min_items`, `max_items`, `max_coefficient`, `capacity_ratio` | number | the `instances` block of the [parameters](parameters.md#instances) (same as `parameters.instances`) |
| `correlation` | object | `families` (string[]), `weak_noise_ratio`, `strong_offset_ratio` ([parameters](parameters.md#instances)); written since 2026-11-26, optional in `schema_version` 4: a file without it was generated with the defaults |

### `environment`

| Field | Type | Meaning |
|---|---|---|
| `interpreter` | string | `GNU Octave`, `MATLAB` or `Python` |
| `version` | string | interpreter version |
| `computer` | string | platform string |
| `process_memory_available` | bool | whether process memory could be read |
| `algorithms.<id>` | object | facts an algorithm adds about the environment, e.g. `algorithms.ilp.solver` (`glpk`, `intlinprog`, `scipy.optimize.milp (HiGHS)`); only algorithms with something to add; absent when none has |
| *implementation-specific* | | extra keys allowed (e.g. library versions) |

### `summary`

| Field | Type | Meaning |
|---|---|---|
| `num_instances` | int | |
| `total_time_seconds` | number | wall time of the execution (all algorithms on all instances; for a server run, all processes of the environment) |
| `ranking_by_median_time` | string[] | ids, fastest first |
| `algorithms.<id>` | object | per-algorithm statistics, fields in [metrics § Summary](metrics.md#summary-statistics) |

### `instances[]`

| Field | Type | Meaning |
|---|---|---|
| `id` | int | 1-based instance number $k$ |
| `family` | string | `uncorrelated` · `weakly_correlated` · `strongly_correlated` |
| `n` | int | number of items |
| `capacity` | int | $C$ |
| `weights`, `values` | int[] | $w$, $v$; left out when `instance_set.file` is set |
| `optimum` | number | $V^*$: value found by the reference algorithm |
| `optimal_items` | int[] | items of the reference algorithm's packing |
| `results.<id>` | object | one per algorithm, [below](#instancesresultsid) |

### `instances[].results.<id>`

| Field | Type | Meaning |
|---|---|---|
| `algorithm` | string | display label |
| `exact` | bool | exact method (`dp`, `bnb`, `ilp`) or heuristic |
| `proven_optimal`, `is_optimal`, `feasible` | bool | [metrics § Quality](metrics.md#quality) |
| `value`, `weight` | number | of the returned packing |
| `items` | int[] | returned packing |
| `gap_percent` | number | |
| `cost` | object | `repeats`, `wall_time_s`, `wall_time_min_s`, `cpu_time_s`, `working_memory_bytes`, `process_memory_before_bytes`, `process_memory_after_bytes`, `process_memory_delta_bytes`, `operations`, `operations_unit` ([metrics § Cost](metrics.md#execution-cost)); implementations may add optional keys |
| `cost_ratio_vs_reference` | object | `time`, `cpu`, `memory` ([metrics](metrics.md#ratios-to-the-reference)) |
| `details` | object | algorithm-specific, below |

### `details` by algorithm

| Algorithm | Fields | Spec |
|---|---|---|
| `dp` | `table_cells` | [dp](algorithms/dp.md#reported-values) |
| `bnb` | `nodes`, `node_limit`, `node_limit_reached` | [bnb](algorithms/bnb.md#5-output) |
| `ilp` | `solver`, `status`, optional `mip_gap` | [ilp](algorithms/ilp.md#reported-values) |
| `qubo`, `qubo_b`, `qc` | `penalty`, `variables`, `slack_bits`, `model_memory_bytes`, `energy`, `iterations`, `raw_items`, `raw_value`, `raw_weight`, `feasible_before_repair`, `repaired`, `build_time_s`, `solve_time_s`, `decode_time_s` | [qubo](algorithms/qubo.md#5-reported-values) |
| `greedy` | `greedy_value`, `single_value`, `chose_single` | [greedy](algorithms/greedy.md#5-reported-values) |
| `sa` | `penalty`, `proposals`, `accepted`, `t_start`, `t_end` | [sa](algorithms/sa.md#5-reported-values) |
| `ga` | `evaluations`, `generations` | [ga](algorithms/ga.md#5-reported-values) |
| `ising` | `penalty`, `variables`, `slack_bits`, `model_memory_bytes`, `energy`, `proposals`, `accepted`, `raw_items`, `raw_value`, `raw_weight`, `feasible_before_repair`, `repaired`, `build_time_s`, `solve_time_s`, `decode_time_s` | [ising](algorithms/ising.md#6-reported-values) |
| `fptas` | `epsilon`, `scale`, `scaled_profit_total`, `table_cells` | [fptas](algorithms/fptas.md#5-reported-values) |

### `server`

Written by knapsack-server for a run it executed:

| Field | Type | Meaning |
|---|---|---|
| `run_id` | string | ULID of the run |
| `request` | object | the [run request](#run-request) after defaults |
| `code` | object | `benchmark_sha` (commit of the benchmark, may be `null`), `server_version`, `image_digest` (may be `null`) |
| `vm` | object or null | `type`, `market` (`spot` · `on-demand`), `vcpus`, `region`; `null` for a local container |
| `processes` | int | benchmark processes run in parallel; the instances are cut into consecutive chunks (about 4 per process) that the processes take in turn. Each chunk is saved when it ends, so a worker restarted after a Spot interruption runs only the instances not saved yet |
| `phases_s` | object | seconds per phase: `queued`, `instances`, `boot`, `execution`, `output` (`null` while running; `output` is still running when the file is written) |

### `instance_set`

| Field | Type | Meaning |
|---|---|---|
| `hash` | string | SHA-256 of the canonical JSON `{seed, instance_config}`, the same for every run with the same instance settings |
| `file` | string | optional: name of the [instance set file](#instance-set-file) next to the run file (`instances.json.gz`). When it is set, `instances[]` leave out `weights` and `values`, and readers join them from that file. Uploaded and CLI files embed them and have no `file`. |

## Raw results (`knapsack-raw` v1)

The benchmarks write `raw.json` (one file per process in a server run). The fields are the run file's own fields, minus everything computed from the reference:

`{format: "knapsack-raw", format_version: 1, project, created, environment, algorithms, algorithm_schemas: {<id>: <content of kp_algo_<id>_output.schema.json>}, skipped_algorithms, reference_algorithm, parameters, instance_config, instances: [{id, results: {<id>: result}}], timing: {total_time_seconds}}`

A result is [`instances[].results.<id>`](#instancesresultsid) without `is_optimal`, `gap_percent` and `cost_ratio_vs_reference`. The output service computes those, `optimum`, `optimal_items` and the [`summary`](#summary) ([metrics](metrics.md)).

## Instance set file

`instances.json` of a run, and the input of the benchmarks' `--instances` option: `{format: "knapsack-instances", format_version: 1, hash (optional), instance_config, instances: [{id, family, n, capacity, weights, values}]}`. The ids are global: a process that gets part of the set still seeds instance $k$ with `seed + 1000 k` ([instances](instances.md)).

## Run request

The input of a server run (`POST /api/v1/runs`) and of the benchmarks' `--request` option. Every field is optional; absent fields come from the [settings files](parameters.md).

| Field | Type | Meaning |
|---|---|---|
| `use_case` | string | optional [use case](parameters.md#use-cases) (`cargo_loading`, `capital_budgeting`, `cloud_packing`, `portfolio`, `ad_slots`); the server rejects instance settings outside its rules; absent = none. Server runs keep it in `server.request`; the benchmarks ignore it |
| `environments` | string[] | `matlab`, `python` (default both; the benchmarks ignore it) |
| `seed` | int | master seed (default 2026) |
| `instances` | object | part of the [`instances` block](parameters.md#instances): `count` (≤ 100 on the server), `min_items`, `max_items`, `max_coefficient`, `capacity_ratio` |
| `algorithms` | string[] | ids to run; the reference `dp` is always added (default: all) |
| `algorithm_config` | object | `{<id>: partial settings}`, merged into the defaults and checked against that algorithm's `config` schema; an unknown key is an error (server: `repeats` ≤ 5) |

## Abridged example

```json
{
  "project": "knapsack-benchmark-python",
  "schema_version": 4,
  "algorithms": [
    { "id": "bnb", "label": "Branch and Bound (Dantzig Bound, Depth-First)",
      "description": "Exact depth-first search over take/skip decisions, ...", "short_label": "BnB",
      "exact": true, "reference": false, "config": { "max_nodes": 5000000, "repeats": 1 } },
    { "id": "dp", "label": "Dynamic Programming 0-1",
      "description": "Exact table of the best value for the first i items ...", "short_label": "DP",
      "exact": true, "reference": true, "config": { "repeats": 5 } },
    "..."
  ],
  "skipped_algorithms": [],
  "instance_config": { "seed": 2026, "count": 100, "min_items": 10, "max_items": 48,
                       "max_coefficient": 100, "capacity_ratio": 0.5 },
  "reference_algorithm": "dp",
  "created": "2026-11-18T19:32:00",
  "environment": { "interpreter": "Python", "version": "3.14.5", "computer": "...",
                   "process_memory_available": true,
                   "algorithms": { "ilp": { "solver": "scipy.optimize.milp (HiGHS)" } } },
  "parameters": { "seed": 2026, "instances": { "count": 100, "...": "..." }, "dp": { "repeats": 5 }, "...": "..." },
  "summary": { "num_instances": 100, "total_time_seconds": 68.0,
               "ranking_by_median_time": ["...", "..."],
               "algorithms": { "qubo": { "num_optimal": 44, "mean_gap_percent": 1.02, "...": "..." } } },
  "instances": [
    { "id": 1, "family": "uncorrelated", "n": 10, "capacity": 285,
      "weights": ["..."], "values": ["..."], "optimum": 345, "optimal_items": ["..."],
      "results": {
        "dp": { "algorithm": "Dynamic Programming 0-1", "exact": true, "proven_optimal": true,
                "value": 345, "items": ["..."], "weight": "...", "feasible": true, "is_optimal": true,
                "gap_percent": 0,
                "cost": { "repeats": 5, "wall_time_s": 0.0002, "...": "...",
                          "operations": 2860, "operations_unit": "table cells" },
                "details": { "table_cells": 2860 },
                "cost_ratio_vs_reference": { "time": 1, "cpu": 1, "memory": 1 } }
      } }
  ]
}
```
