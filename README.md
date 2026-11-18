# knapsack — documentation

Documentation of the **knapsack benchmark**: five 0-1 knapsack algorithms run on reproducible generated instances, implemented twice:

| Repository | Language | Implementation docs |
|---|---|---|
| [`knapsack-benchmark-python`](https://github.com/knapsack-org/knapsack-benchmark-python) | Python ≥ 3.10, NumPy, SciPy | [implementation/python](implementation/python/README.md) |
| [`knapsack-benchmark-matlab`](https://github.com/knapsack-org/knapsack-benchmark-matlab) | GNU Octave ≥ 7 / MATLAB ≥ R2021a | [implementation/matlab](implementation/matlab/README.md) |

The results viewer [`knapview`](https://github.com/knapsack-org/knapview) reads the JSON both write ([output format](functional/output-format.md)).

## How this documentation is split

| Part | Answers | Contains | Never contains |
|---|---|---|---|
| [**functional/**](functional/README.md) | *What* is computed | problem definition, algorithm descriptions and formulas, parameters, instance-family formulas, metrics, output contract, acceptance tests | file names, function names, language syntax, libraries |
| [**implementation/**](implementation/README.md) | *How* each language does it | install and run, modules/functions, library choices (HiGHS, GLPK, `whos`, `tracemalloc`…), known differences between the two | formulas or algorithm logic (it links to `functional/` instead) |

Rule: a behaviour is specified **once**, in `functional/`. Implementation pages map it to code and list deviations. If the code and `functional/` disagree, one of them is a bug.

## Contents

**Functional specification**
1. [Problem and conventions](functional/problem.md) — 0-1 knapsack, notation, indexing, units
2. [Instance families](functional/instances.md) — random generator, Pisinger families, sizes, capacity
3. Algorithms — [`dp`](functional/algorithms/dp.md) · [`bnb`](functional/algorithms/bnb.md) · [`ilp`](functional/algorithms/ilp.md) · [`qubo`](functional/algorithms/qubo.md) · [`qubo_chz`](functional/algorithms/qubo_chz.md)
4. [Parameters](functional/parameters.md) — every setting, default and effect
5. [Quality and cost metrics](functional/metrics.md) — gap, optimality, time, memory, operations, summary statistics
6. [Output format](functional/output-format.md) — JSON `schema_version` 3
7. [Acceptance tests](functional/acceptance-tests.md) — P01, the MathWorks QUBO example, expected benchmark results

**Implementation specification**
- [Overview and traceability matrix](implementation/README.md)
- [Adding an algorithm](implementation/adding-an-algorithm.md) — checklist, adapter contract, MATLAB and Python skeletons; [functional page template](functional/algorithms/_template.md)
- [Python](implementation/python/README.md) · [Python module reference](implementation/python/api.md)
- [MATLAB / Octave](implementation/matlab/README.md) · [MATLAB function reference](implementation/matlab/api.md)

[References](references.md)

## Conventions

- Plain GitHub Markdown; formulas in LaTeX (`$…$`, `$$…$$`), rendered by GitHub and Obsidian.
- Scope: what is **implemented today** (benchmark projects as of 2026-11-18: self-contained algorithm folders found by scanning, no run options, JSON `schema_version` 3). Planned work (the `knapsack-python` / `knapsack-matlab` libraries, other variants) is out of scope.
- Source of truth for this content: the code of both benchmark projects. Background notes (theory, library surveys, EN/FR) live in the project's Obsidian vault.
