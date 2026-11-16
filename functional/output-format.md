# Output format (JSON, `schema_version` 2)

One run writes one UTF-8 JSON file named `run-dd-mm-yy-hh-mm.json` (day-month-year-hour-minute of the end of the run). This is the contract between the two benchmark implementations and the Knapview viewer (repository `knapview`).

## Versioning

- `schema_version` (integer) identifies the structure; current value **2**, identical in both implementations.
- It must be increased in **both** implementations on any breaking change (renamed/removed field, changed type or meaning). Adding an optional field is not breaking.
- Readers reject a version they do not support. Knapview reads only version 2 (a version 1 file must be re-run).
- History: **1** (2026-11-14) first version. **2** (2026-11-16) `algorithms` is a list of objects with each algorithm's settings; per-algorithm parameter blocks; `instance_config` and the `instances` parameter block ([renamed keys](parameters.md#keys-renamed-in-schema_version-2)).

## Encoding rules

- Missing or undefined numbers (`NaN`, `±∞`) are written as `null`.
- Integral numbers are written without a decimal part.
- Item lists are arrays of 1-based indices (possibly empty `[]`).
- Booleans are JSON `true` / `false`.

## Top level

| Field | Type | Meaning |
|---|---|---|
| `project` | string | `knapsack-benchmark-matlab` or `knapsack-benchmark-python` |
| `schema_version` | int | `2` |
| `algorithms` | object[] | algorithms actually run, in run order: [below](#algorithms) (version 1: string[] of ids) |
| `instance_config` | object | settings of the instance generator: [below](#instance_config) (not in version 1) |
| `reference_algorithm` | string | `"dp"` |
| `description` | string | human-readable summary |
| `created` | string | local date-time `YYYY-MM-DDTHH:MM:SS` |
| `environment` | object | [below](#environment) |
| `parameters` | object | the full resolved [parameters](parameters.md) |
| `summary` | object | [below](#summary) |
| `instances` | object[] | [below](#instances) |

### `algorithms`

One object per algorithm run, in run order:

| Field | Type | Meaning |
|---|---|---|
| `id` | string | algorithm id, key of `summary.algorithms` and `instances[].results` |
| `label` | string | human-readable name (same as `results.<id>.algorithm`) |
| `exact` | bool | the method can prove optimality |
| `config` | object | the resolved settings block of the algorithm (same as `parameters.<id>`, see [parameters](parameters.md#algorithm-settings)) |

### `instance_config`

The inputs of the [instance generator](instances.md#generation-procedure), so the instances can be regenerated:

| Field | Type | Meaning |
|---|---|---|
| `seed` | int | master seed (same as `parameters.seed`) |
| `count`, `min_items`, `max_items`, `max_coefficient`, `capacity_ratio` | number | the `instances` block of the [parameters](parameters.md#instances) (same as `parameters.instances`) |

### `environment`

| Field | Type | Meaning |
|---|---|---|
| `interpreter` | string | `GNU Octave`, `MATLAB` or `Python` |
| `version` | string | interpreter version |
| `computer` | string | platform string |
| `ilp_solver` | string | ILP engine used, `""` if `ilp` was not run |
| `process_memory_available` | bool | whether process memory could be read |
| *implementation-specific* | | extra keys allowed (e.g. library versions) |

### `summary`

| Field | Type | Meaning |
|---|---|---|
| `num_instances` | int | |
| `total_time_seconds` | number | wall time of the run |
| `ranking_by_median_time` | string[] | ids, fastest first |
| `algorithms.<id>` | object | per-algorithm statistics, fields in [metrics § Summary](metrics.md#summary-statistics) |

### `instances[]`

| Field | Type | Meaning |
|---|---|---|
| `id` | int | 1-based instance number $k$ |
| `family` | string | `uncorrelated` · `weakly_correlated` · `strongly_correlated` |
| `n` | int | number of items |
| `capacity` | int | $C$ |
| `weights`, `values` | int[] | $w$, $v$ |
| `optimum` | number | $V^*$ from `dp` |
| `optimal_items` | int[] | items of the `dp` optimal packing |
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
| `cost_ratio_vs_dp` | object | `time`, `cpu`, `memory` |
| `details` | object | algorithm-specific, below |

### `details` by algorithm

| Algorithm | Fields | Spec |
|---|---|---|
| `dp` | `table_cells` | [dp](algorithms/dp.md#reported-values) |
| `bnb` | `nodes`, `node_limit`, `node_limit_reached` | [bnb](algorithms/bnb.md#5-output) |
| `ilp` | `solver`, `status`, optional `mip_gap` | [ilp](algorithms/ilp.md#reported-values) |
| `qubo`, `qubo_chz` | `penalty`, `variables`, `slack_bits`, `model_memory_bytes`, `energy`, `iterations`, `raw_items`, `raw_value`, `raw_weight`, `feasible_before_repair`, `repaired`, `build_time_s`, `solve_time_s`, `decode_time_s` | [qubo](algorithms/qubo.md#5-reported-values) |

## Abridged example

```json
{
  "project": "knapsack-benchmark-python",
  "schema_version": 2,
  "algorithms": [
    { "id": "dp", "label": "dynamic programming 0-1", "exact": true, "config": { "repeats": 5 } },
    { "id": "bnb", "label": "branch and bound (Dantzig bound, depth-first)", "exact": true,
      "config": { "max_nodes": 5000000, "repeats": 1 } },
    "..."
  ],
  "instance_config": { "seed": 2026, "count": 100, "min_items": 10, "max_items": 48,
                       "max_coefficient": 100, "capacity_ratio": 0.5 },
  "reference_algorithm": "dp",
  "created": "2026-11-14T15:08:00",
  "environment": { "interpreter": "Python", "version": "3.12.0", "computer": "...",
                   "process_memory_available": true, "ilp_solver": "scipy.optimize.milp (HiGHS)" },
  "parameters": { "seed": 2026, "instances": { "count": 100, "...": "..." }, "dp": { "repeats": 5 }, "...": "..." },
  "summary": { "num_instances": 100, "total_time_seconds": 57.3,
               "ranking_by_median_time": ["...", "..."],
               "algorithms": { "qubo": { "num_optimal": 44, "mean_gap_percent": 1.02, "...": "..." } } },
  "instances": [
    { "id": 1, "family": "uncorrelated", "n": 10, "capacity": 285,
      "weights": ["..."], "values": ["..."], "optimum": 345, "optimal_items": ["..."],
      "results": {
        "dp": { "algorithm": "dynamic programming 0-1", "exact": true, "proven_optimal": true,
                "value": 345, "items": ["..."], "weight": "...", "feasible": true, "is_optimal": true,
                "gap_percent": 0,
                "cost": { "repeats": 5, "wall_time_s": 0.0002, "...": "...",
                          "operations": 2860, "operations_unit": "table cells" },
                "details": { "table_cells": 2860 },
                "cost_ratio_vs_dp": { "time": 1, "cpu": 1, "memory": 1 } }
      } }
  ]
}
```
