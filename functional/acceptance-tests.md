# Acceptance tests

Checks every implementation must pass. Where they are coded today: [Python tests](../implementation/python/README.md#tests), [MATLAB tests](../implementation/matlab/README.md#tests).

## A. Reference instance P01

[P01](problem.md#reference-instance-p01): $C = 165$, optimum 309, items $\{1, 2, 3, 4, 6\}$.

| # | Check |
|---|---|
| A1 | `dp` returns 309 and items $\{1, 2, 3, 4, 6\}$ |
| A2 | `bnb` returns 309, items $\{1, 2, 3, 4, 6\}$, `proven_optimal` = true |
| A3 | `bnb` with a node limit of 3 returns a value $\le 309$ |
| A4 | `ilp` returns 309, items $\{1, 2, 3, 4, 6\}$, proven optimal (skipped if no solver is available) |
| A5 | `qubo` slack-optimal tabu search with $P = \max v + 1 = 93$, default tabu settings, seed 1: decoded packing feasible with value 309, returned energy $-309$ |
| A6 | `qubo` all-bits tabu search, same settings: decoded weight $\le C$, and the returned energy equals $z^\top Q z + c^\top z + d$ of the returned vector (tolerance $10^{-9}$) |
| A7 | `qubo_chz` builds the same $Q, c, d$ as `qubo`, its slack-optimal search returns the same vector and energy ($-309$), and decodes to 309 |

## B. QUBO model (MathWorks `knapsack2qubo` example)

$v = (5, 2, 8, 4, 3)$, $w = (9, 8, 4, 5, 2)$, $C = 17$, $P = 1$.

| # | Check |
|---|---|
| B1 | $n + K = 10$ variables ($K = 5$, slack weights $(1, 2, 4, 8, 2)$) and $d = 289$ |
| B2 | for $z = (1,0,1,0,1,\ 0,1,0,0,0)$ — items 1, 3, 5 (weight 15) and slack 2 — $z^\top Q z + c^\top z + d = -16$ |
| B3 | slack round trip: for $s = 0 \dots 17$, the encoded bits weighted by $(1, 2, 4, 8, 2)$ sum back to $s$ (also for `qubo_chz`'s encoder) |

## C. Generator and full-run parity (default parameters, 100 instances)

| # | Check |
|---|---|
| C1 | Same instances in every implementation (weights, values, capacities) |
| C2 | `bnb` proves the `dp` optimum on all 100 instances; total nodes 489 151; max 284 157, reached on instance 96 |
| C3 | `dp`, `bnb`, `qubo`, `qubo_chz`: identical values, items, node counts, tabu iteration counts and energies across implementations |
| C4 | `ilp`: identical **values**; item sets may differ when several optima exist (seen on 30 of 100 instances between HiGHS and GLPK) |
| C5 | `schema_version` = 3; algorithms in alphabetical order; reference `dp`; every `algorithms[]` entry and every `details` block valid against its algorithm's JSON Schema |

## Expected benchmark results

Default parameters, seed 2026, 100 instances:

| Algorithm | Optimal | Proven | Mean gap | Max gap | Notes |
|---|---|---|---|---|---|
| `dp` | 100 | 100 | 0 % | 0 % | reference |
| `bnb` | 100 | 100 | 0 % | 0 % | median 114 nodes, mean ≈ 4 900 |
| `ilp` | 100 | 100 | 0 % | 0 % | |
| `qubo` | 44 | 0 | 1.02 % | 4.80 % | all feasible before repair |
| `qubo_chz` | 44 | 0 | 1.02 % | 4.80 % | identical to `qubo` until modified |

Time and memory are machine-dependent; measured orders of magnitude are listed on the implementation pages.

## Edge cases (not yet covered by tests)

Capacity 0; an item heavier than the capacity; all items fit; equal ratios; $N = 1$ instance. These are listed for future work; the current implementations are not verified on them.
