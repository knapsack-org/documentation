# Adding an algorithm

Every algorithm is a self-contained folder with the same three parts in both implementations: the **algorithm** itself, its **settings** (one config function) and a **benchmark adapter** that turns its output into the common result struct. The runner finds the config and adapter by name, so a new algorithm touches no runner code.

Use `<id>` for the new id (lower case, `[a-z0-9_]`, e.g. `greedy`).

## Checklist

| # | Step | MATLAB / Octave (`knapsack-benchmark-matlab`) | Python (`knapsack-benchmark-python`) |
|---|---|---|---|
| 1 | Functional page | `documentation/functional/algorithms/<id>.md` from [the template](../functional/algorithms/_template.md); add its settings to [parameters](../functional/parameters.md) and its `details` to [output format](../functional/output-format.md#details-by-algorithm) | same page (one spec for both) |
| 2 | Folder | `src/algorithms/<id>/` | `src/knapsack_benchmark/algorithms/<id>/` with an empty `__init__.py` |
| 3 | Algorithm | `kb_<name>.m` (one or more files) | `<id>.py` |
| 4 | Settings | `kb_config_<id>.m` → struct, must contain `repeats` | `config_<id>.py`: `config_<id>() -> dict`, must contain `repeats` |
| 5 | Adapter | `kb_algo_<id>.m`: `out = kb_algo_<id>(v, w, C, acfg, seed)` | `algo_<id>.py`: `algo_<id>(v, w, C, acfg, seed) -> dict` |
| 6 | Register | add `'<id>'` to `src/helpers/kb_algorithm_ids.m` | add `"<id>"` to `ALGORITHMS` in `config.py` |
| 7 | Test | `tests/test_p01.m`: P01 optimum 309 (exact) or a feasible value (heuristic) | `tests/test_p01.py`, same check |
| 8 | Docs | [traceability matrix](README.md#traceability-matrix), [MATLAB reference](matlab/api.md), project README | [Python reference](python/api.md), project README |

Then check parity: run both projects on the same instances and compare the values (see [acceptance tests](../functional/acceptance-tests.md)).

Registering an id changes the default `algorithms` list and adds a `parameters.<id>` block to every run: older files simply lack it, so this is **not** a breaking change of the output (no `schema_version` bump). Knapview shows an unknown id with a neutral colour and the generic widgets.

## Adapter contract

Inputs: `v`, `w` (values and weights, 1 × n), `C` (capacity), `acfg` = the algorithm's settings block `cfg.<id>` after overrides, `seed` = `cfg.seed + 1000 * k` for instance `k` (use it, plus an own offset if needed, for any randomness; ignore it otherwise).

Output — exactly these fields:

| Field | Meaning |
|---|---|
| `label` | human-readable name, written as `algorithms[].label` and `results.<id>.algorithm` |
| `exact` | `true` if the method can prove optimality |
| `proven_optimal` | `true` if optimality was proven for this instance |
| `items` | selected items, **1-based**, sorted |
| `working_memory_bytes` (MATLAB) / `array_bytes` (Python) | bytes of the algorithm's own arrays |
| `operations`, `operations_unit` | main work counter and its unit (`null` if not available) |
| `details` | algorithm-specific values (struct / dict), documented in the functional page |

The runner (`kb_run_algorithm` / `run_algorithm`) times the adapter `acfg.repeats` times and derives value, weight, feasibility, gap and cost from `items`.

## MATLAB / Octave skeleton

`src/algorithms/<id>/kb_config_<id>.m`

```matlab
function c = kb_config_<id>()
%KB_CONFIG_<ID>  Default settings of '<id>' (block cfg.<id> of BENCHMARK_CONFIG).
  c = struct();
  c.my_limit = 1000;            % <meaning>
  c.repeats = 3;                % timed runs per instance
end
```

`src/algorithms/<id>/kb_algo_<id>.m`

```matlab
function out = kb_algo_<id>(v, w, C, acfg, seed) %#ok<INUSD>
%KB_ALGO_<ID>  Benchmark adapter of '<id>': run KB_<NAME>, return the common result struct.
%   OUT = KB_ALGO_<ID>(V, W, C, ACFG, SEED): ACFG = cfg.<id>, SEED = cfg.seed + 1000 * k.
  [~, items, info] = kb_<name>(v, w, C, acfg.my_limit);
  out = struct('label', '<full name>', 'exact', false, 'proven_optimal', false, ...
               'items', items, 'working_memory_bytes', kb_bytes(v, w), ...
               'operations', info.steps, 'operations_unit', '<unit>');
  out.details = struct('steps', info.steps);
end
```

## Python skeleton

`src/knapsack_benchmark/algorithms/<id>/config_<id>.py`

```python
"""Default settings of '<id>' (block cfg["<id>"]; port of kb_config_<id>.m)."""


def config_<id>() -> dict:
    return {
        "my_limit": 1000,          # <meaning>
        "repeats": 3,              # timed runs per instance
    }
```

`src/knapsack_benchmark/algorithms/<id>/algo_<id>.py`

```python
"""Benchmark adapter of '<id>' (port of kb_algo_<id>.m)."""
from __future__ import annotations

from .<id> import <name>


def algo_<id>(v, w, C, acfg, seed):
    """algo_<id>(v, w, C, acfg, seed): acfg = cfg["<id>"], seed = cfg["seed"] + 1000 * k."""
    _, items, info = <name>(v, w, C, acfg["my_limit"])
    return {"label": "<full name>", "exact": False, "proven_optimal": False,
            "items": items, "array_bytes": int(v.nbytes + w.nbytes),
            "operations": info["steps"], "operations_unit": "<unit>",
            "details": {"steps": info["steps"]}}
```

## Existing examples

| Kind | Look at |
|---|---|
| exact, no settings but `repeats` | `dp` |
| exact with a limit | `bnb` (`max_nodes`) |
| external solver | `ilp` |
| randomised, nested settings, uses `seed` | `qubo`; `qubo_chz` adds `seed_offset` |
