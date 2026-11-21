# Adding an algorithm

Every algorithm is a **self-contained folder**. The benchmark finds it by scanning the algorithms folder, so adding, removing or changing an algorithm touches **only its folder** (plus its functional page in this repository). No runner, registry, helper, test or viewer file is edited.

Use `<id>` for the new id: lower case, matching `[a-z][a-z0-9_]*` (e.g. `greedy`). The folder name is the id.

## Checklist

| # | Step | MATLAB / Octave (`knapsack-benchmark-matlab`) | Python (`knapsack-benchmark-python`) |
|---|---|---|---|
| 1 | Functional page | `documentation/functional/algorithms/<id>.md` from [the template](../functional/algorithms/_template.md); add its settings to [parameters](../functional/parameters.md) and its `details` to [output format](../functional/output-format.md#details-by-algorithm) | same page (one spec for both) |
| 2 | Folder | `src/algorithms/<id>/` | `src/knapsack_benchmark/algorithms/<id>/` with an empty `__init__.py` |
| 3 | Loader | `kp_algo_<id>_loader.m` | `kp_algo_<id>_loader.py` |
| 4 | Maths | `kp_algo_<id>_main.m` | `kp_algo_<id>_main.py` |
| 5 | Settings | `kp_algo_<id>_config.m` | `kp_algo_<id>_config.py` |
| 6 | Output | `kp_algo_<id>_output.m` + `kp_algo_<id>_output.schema.json` | `kp_algo_<id>_output.py` + the same `.schema.json` (byte-identical) |
| 7 | Helpers (optional) | `modules/*.m` | `modules/*.py` with `__init__.py` |
| 8 | Check | `octave-cli --eval "addpath('tests'); test_p01"`, then a full run | `pytest`, then a full run |

The P01 test runs **every** discovered algorithm through its adapter (exact: value 309; heuristic: feasible) and checks its keys against its schema file, so a new algorithm needs no test edit. knapsack-server's CI checks that the schema files are byte-identical in both repositories (`scripts/check-schemas.mjs`), and its output service validates every run file against them. Algorithm-specific unit tests may still be added to `tests/`. Then check parity: both projects must give the same values (see [acceptance tests](../functional/acceptance-tests.md)).

Adding an algorithm adds an `algorithms[]` entry, a `parameters.<id>` block and a `results.<id>` per instance: this is **not** a breaking change of the output (no `schema_version` bump). Knapview names it with its `short_label` and shows it with a neutral colour and the generic widgets.

## Folder rules

- The four `kp_algo_<id>_*` files are required; discovery stops with an error when one is missing or when the loader's `id` differs from the folder name.
- Algorithm code may call the generic helpers (`kb_bytes`, `kb_lcg`, `kb_randi`, `kb_seed` in MATLAB; `helpers.rng`, `helpers.phase_timer` in Python) but **never another algorithm's folder**. Code shared with another algorithm is copied into this folder's `modules/` (e.g. `qubo/modules/kb_penalty.m` and `qubo_chz/modules/kb_chz_penalty.m`).
- MATLAB only: every `.m` file name under `src/algorithms/` must be unique across folders, because all folders share one path (`genpath`) and a duplicate would silently shadow the other. Discovery stops with `kb_discover_algorithms:duplicateName`. Prefix module files with the id when in doubt (`kb_chz_*`).
- Only the four contract files use the `kp_` prefix; module files keep `kb_` (MATLAB) or any module name (Python).

## Loader contract — `kp_algo_<id>_loader`

Returns a struct (MATLAB) / dict (Python), read by discovery:

| Field | Type | Meaning |
|---|---|---|
| `id` | string | the folder name |
| `label` | string | human-readable name, main words capitalised (`Branch and Bound (Dantzig Bound, Depth-First)`); written to `algorithms[].label` and `results.<id>.algorithm` (the output may add a variant, e.g. the tabu mode) |
| `description` | string | one or two sentences on the method, written to `algorithms[].description` |
| `short_label` | string | short name for chips and legends (`BnB`) |
| `exact` | bool | the method can prove optimality |
| `reference` | bool | `true` only for `dp`: its value is the optimum the others are compared with. Exactly one available exact algorithm must be the reference |
| `available` | function → bool | `false` when the algorithm cannot run here (e.g. `ilp` without a solver): it is skipped with a warning and listed in `skipped_algorithms` |
| `environment` | function → struct/dict | facts written to `environment.algorithms.<id>` (e.g. `{solver}` for `ilp`); empty when there is nothing to add |

## Main contract — `kp_algo_<id>_main`

The maths only, no benchmark wrapping:

`[items, info] = kp_algo_<id>_main(v, w, C, acfg, seed, mark)` (Python: `kp_algo_<id>_main(v, w, C, acfg=None, seed=None, mark=None) -> (items, info)`).

| Input | Meaning |
|---|---|
| `v`, `w`, `C` | values and weights (1 × n), capacity |
| `acfg` | the settings block `cfg.<id>` |
| `seed` | `cfg.seed + 1000 * k` for instance `k`; use it (plus an own offset if needed) for any randomness, ignore it otherwise |
| `mark` | optional phase hook: call `mark('build')`, `mark('solve')`, … at the end of each phase you want timed; the adapter turns the marks into `details.<phase>_time_s`. Default: none |

`items` are the selected items, **1-based**, sorted. `info` holds what the output needs: the value, counters only the algorithm can see (nodes, move evaluations, its own array bytes) and any by-product (energy, raw solution).

MATLAB: accept unused trailing arguments with `varargin`, not `~`: Octave then counts an extra `ans` variable in `whos`, which changes `working_memory_bytes`. Pass `_main` only the arguments it uses.

## Settings contract — `kp_algo_<id>_config`

Returns the default block `cfg.<id>` (struct / dict), which **must contain `repeats`**. It is the only place a default of this algorithm is changed; a [run request](../functional/output-format.md#run-request) may override settings for one run (`algorithm_config.<id>`, checked against the `config` schema).

## Output contract — `kp_algo_<id>_output`

MATLAB: `O = kp_algo_<id>_output()` returns a struct with `run` and `entry`; Python: a module with `run` and `entry`. The JSON Schema is the file `kp_algo_<id>_output.schema.json` next to them.

| Part | Meaning |
|---|---|
| `run(v, w, C, acfg, seed)` | the benchmark adapter: calls `_main` (with a `kb_phase_timer` / `phase_timer` as `mark` if it has phases), returns the common result below |
| `entry(acfg, env)` | the algorithm's `algorithms[]` entry: `{id, label, description, short_label, exact, reference, config}` (`env` = the loader's environment, e.g. for a solver name in the label) |
| `kp_algo_<id>_output.schema.json` | JSON Schema 2020-12 with three parts: `entry` (its `config` property is only `{"type": "object"}`), `config` and `details`. The benchmarks insert it verbatim into `raw.json` (`algorithm_schemas.<id>`); knapsack-server validates the run file against it |

Common result returned by `run` — exactly these fields:

| Field | Meaning |
|---|---|
| `label` | written as `results.<id>.algorithm` |
| `exact` | `true` if the method can prove optimality |
| `proven_optimal` | `true` if optimality was proven for this instance |
| `items` | selected items, **1-based**, sorted |
| `working_memory_bytes` | bytes of the algorithm's own arrays |
| `operations`, `operations_unit` | main work counter and its unit (`null` if not available) |
| `details` | algorithm-specific values, documented in the functional page and described by the `details` schema |

The runner (`kb_run_algorithm` / `run_algorithm`) times `run` `acfg.repeats` times and derives value, weight, feasibility and cost from `items`; knapsack-server adds the gap and the ratios.

## MATLAB / Octave skeleton

`src/algorithms/<id>/kp_algo_<id>_loader.m`

```matlab
function L = kp_algo_<id>_loader()
%KP_ALGO_<ID>_LOADER  Identity of '<id>' for KB_DISCOVER_ALGORITHMS.
  L = struct();
  L.id = '<id>';                        % must equal the folder name
  L.label = '<Full Name>';
  L.description = '<One or two sentences on the method.>';
  L.short_label = '<SHORT>';            % algorithm chip in Knapview
  L.exact = false;
  L.reference = false;
  L.available = @() true;
  L.environment = @() struct();         % nothing to add to environment.algorithms
end
```

`src/algorithms/<id>/kp_algo_<id>_main.m`

```matlab
function [items, info] = kp_algo_<id>_main(v, w, C, acfg, varargin)
%KP_ALGO_<ID>_MAIN  <What it computes>.
  limit = acfg.my_limit;
  % ... the method ...
  items = find(take);
  info = struct('value', sum(v(items)), 'steps', steps);
  vars = whos();
  info.working_memory_bytes = sum([vars.bytes]);
end
```

`src/algorithms/<id>/kp_algo_<id>_config.m`

```matlab
function c = kp_algo_<id>_config()
%KP_ALGO_<ID>_CONFIG  Settings of '<id>' (block cfg.<id> of BENCHMARK_CONFIG).
  c = struct();
  c.my_limit = 1000;            % <meaning>
  c.repeats = 3;                % timed runs per instance
end
```

`src/algorithms/<id>/kp_algo_<id>_output.m`

```matlab
function O = kp_algo_<id>_output()
%KP_ALGO_<ID>_OUTPUT  Benchmark adapter and JSON description of '<id>'.
  O = struct('run', @run_algo, 'entry', @build_entry);   % JSON Schema: kp_algo_<id>_output.schema.json
end

function out = run_algo(v, w, C, acfg, seed) %#ok<INUSD>
  [items, info] = kp_algo_<id>_main(v, w, C, acfg);
  L = kp_algo_<id>_loader();
  out = struct('label', L.label, 'exact', false, 'proven_optimal', false, 'items', items, ...
               'working_memory_bytes', info.working_memory_bytes, ...
               'operations', info.steps, 'operations_unit', '<unit>');
  out.details = struct('steps', info.steps);
end

function e = build_entry(acfg, env) %#ok<INUSD>
  L = kp_algo_<id>_loader();
  e = struct('id', L.id, 'label', L.label, 'description', L.description, ...
             'short_label', L.short_label, 'exact', L.exact, 'reference', L.reference, ...
             'config', acfg);
end
```

MATLAB arrays in `details` must be cells (`num2cell`) so that a one-element list is still written as a JSON array.

`kp_algo_<id>_output.schema.json` (the same bytes in both repositories: copy the file)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "entry": {
    "type": "object", "additionalProperties": false,
    "required": ["id", "label", "description", "short_label", "exact", "reference", "config"],
    "properties": {
      "id": { "enum": ["<id>"] }, "label": { "type": "string" },
      "description": { "type": "string" }, "short_label": { "type": "string" },
      "exact": { "type": "boolean" }, "reference": { "type": "boolean" },
      "config": { "type": "object" }
    }
  },
  "config": {
    "type": "object", "additionalProperties": false, "required": ["my_limit", "repeats"],
    "properties": { "my_limit": { "type": "integer", "minimum": 1 }, "repeats": { "type": "integer", "minimum": 1 } }
  },
  "details": {
    "type": "object", "additionalProperties": false, "required": ["steps"],
    "properties": { "steps": { "type": "integer", "minimum": 0 } }
  }
}
```

`entry` describes the `algorithms[]` entry, `config` the settings block (a run request's `algorithm_config.<id>` is checked against it after merging) and `details` the `results.<id>.details` block.

## Python skeleton

`src/knapsack_benchmark/algorithms/<id>/kp_algo_<id>_loader.py`

```python
"""Identity of '<id>' for discover_algorithms (port of kp_algo_<id>_loader.m)."""
from __future__ import annotations


def kp_algo_<id>_loader() -> dict:
    return {
        "id": "<id>",                  # must equal the folder name
        "label": "<Full Name>",
        "description": "<One or two sentences on the method.>",
        "short_label": "<SHORT>",      # algorithm chip in Knapview
        "exact": False,
        "reference": False,
        "available": lambda: True,
        "environment": lambda: {},
    }
```

`src/knapsack_benchmark/algorithms/<id>/kp_algo_<id>_main.py`

```python
"""<What it computes> (port of kp_algo_<id>_main.m)."""
from __future__ import annotations


def kp_algo_<id>_main(v, w, C, acfg=None, seed=None, mark=None) -> tuple[list[int], dict]:
    limit = acfg["my_limit"]
    # ... the method ...
    return items, {"value": value, "steps": steps, "working_memory_bytes": array_bytes}
```

`src/knapsack_benchmark/algorithms/<id>/kp_algo_<id>_config.py`

```python
"""Settings of '<id>' (block cfg["<id>"]; port of kp_algo_<id>_config.m)."""


def kp_algo_<id>_config() -> dict:
    return {
        "my_limit": 1000,          # <meaning>
        "repeats": 3,              # timed runs per instance
    }
```

`src/knapsack_benchmark/algorithms/<id>/kp_algo_<id>_output.py`

```python
"""Benchmark adapter and JSON description of '<id>' (port of kp_algo_<id>_output.m)."""
from __future__ import annotations

from .kp_algo_<id>_loader import kp_algo_<id>_loader
from .kp_algo_<id>_main import kp_algo_<id>_main


def run(v, w, C, acfg, seed):
    items, info = kp_algo_<id>_main(v, w, C, acfg, seed)
    return {"label": kp_algo_<id>_loader()["label"], "exact": False, "proven_optimal": False,
            "items": items, "working_memory_bytes": info["working_memory_bytes"],
            "operations": info["steps"], "operations_unit": "<unit>",
            "details": {"steps": info["steps"]}}


def entry(acfg, env):
    L = kp_algo_<id>_loader()
    return {"id": L["id"], "label": L["label"], "description": L["description"],
            "short_label": L["short_label"], "exact": L["exact"], "reference": L["reference"],
            "config": acfg}

```

## Existing examples

| Kind | Look at |
|---|---|
| exact, no settings but `repeats`, reference | `dp` |
| exact with a limit | `bnb` (`max_nodes`) |
| external solver, `available()` and `environment()`, solver name in the label | `ilp` |
| randomised, nested settings, uses `seed`, timed phases with `mark`, `modules/` | `qubo`; `qubo_chz` adds `seed_offset` and copies the modules |
