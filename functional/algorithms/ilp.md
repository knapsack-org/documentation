# `ilp` — integer linear programming (exact)

| | |
|---|---|
| Type | exact; optimality as reported by the solver |
| Method | generic mixed-integer linear programming (MILP) solver: LP-based branch and bound with cuts and heuristics |
| Parameters | none of its own (`ilp.repeats` only, see [parameters](../parameters.md#ilp)) |
| Solver | chosen by the implementation — see [Python](../../implementation/python/README.md#ilp-solver) and [MATLAB](../../implementation/matlab/README.md#ilp-solver) |

## Model

$$
\min_{x} \; -\sum_{i=1}^{n} v_i x_i
\quad\text{s.t.}\quad
\sum_{i=1}^{n} w_i x_i \le C,
\qquad 0 \le x_i \le 1,\ x_i \in \mathbb{Z}.
$$

(Minimising $-v^\top x$ is the solver form of maximising $v^\top x$.) The required relative optimality gap is **0**: the solver must prove optimality, not stop within a tolerance.

## Solution extraction

The solver returns real numbers; item $i$ is taken when $\operatorname{round}(x_i) = 1$ (i.e. $x_i > 0.5$). If the solver returns no point, the packing is empty.

## Reported values

| Field | Value |
|---|---|
| `proven_optimal` | `true` only when the solver status means "optimal solution found" |
| `operations` | the solver's branch-and-bound node count, or `null` when the solver does not expose it |
| `details.solver` | solver name |
| `details.status` | the solver's raw status code (meaning is solver-specific) |
| `details.mip_gap` | final relative MIP gap, when the solver exposes it (0 when proven) |
| `working_memory_bytes` | the problem data only ($v$, $w$, $x$); the solver's internal memory is not visible |

## Non-uniqueness

When several packings are optimal, different solvers may return different item sets with the same value. The [acceptance tests](../acceptance-tests.md) therefore compare `ilp` on **values**, not on items.

## Why it is in the benchmark

It is the least specialised method: an extra side constraint (a second resource, a conflict between two items…) costs one more row in the model, whereas `dp` and `bnb` would have to be rewritten.
