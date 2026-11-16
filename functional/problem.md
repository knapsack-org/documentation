# Problem and conventions

## The 0-1 knapsack problem

Given $n$ items with values $v_i > 0$ and weights $w_i > 0$, and a capacity $C \ge 0$, choose a subset that maximises the total value without exceeding the capacity:

$$
\max_{x} \sum_{i=1}^{n} v_i x_i
\quad\text{subject to}\quad
\sum_{i=1}^{n} w_i x_i \le C,
\qquad x_i \in \{0, 1\}.
$$

In the benchmark, $w_i$, $v_i$ and $C$ are **positive integers** (required by [`dp`](algorithms/dp.md); [`bnb`](algorithms/bnb.md) uses integrality of values to tighten its bound).

The problem is NP-hard, but pseudo-polynomial: [`dp`](algorithms/dp.md) solves it in $O(nC)$.

## Notation

| Symbol | Meaning |
|---|---|
| $n$ | number of items |
| $v_i$, $w_i$ | value and weight of item $i$ |
| $C$ | capacity |
| $x \in \{0,1\}^n$ | packing: $x_i = 1$ if item $i$ is taken |
| $V(x) = \sum v_i x_i$, $W(x) = \sum w_i x_i$ | value and weight of a packing |
| $V^*$ | optimum value (from [`dp`](algorithms/dp.md)) |
| $R$ | `instances.max_coefficient`, upper bound of generated weights |
| $K$ | number of QUBO slack bits ([`qubo`](algorithms/qubo.md)) |
| $P$ | QUBO penalty weight |

A packing is **feasible** when $W(x) \le C$.

## Conventions

- **Item indices are 1-based** everywhere they are reported (lists of items in the output). Item lists are sorted ascending.
- **Units:** time in seconds, memory in bytes, gaps in percent.
- **Rounding:** `round` means *round half away from zero* (MATLAB `round`); `floor` is the mathematical floor.
- **Ties** in an arg-min/arg-max pick the **lowest index**; sorts are **stable** (equal keys keep the index order).
- **Determinism:** with the same parameters, every algorithm returns the same result on every run and in every language (random numbers come from the portable generator in [instances.md](instances.md#portable-random-generator)).

## Reference instance P01

Used by the [acceptance tests](acceptance-tests.md):

| Item | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|---|---|---|---|---|---|---|---|---|---|---|
| $w_i$ | 23 | 31 | 29 | 44 | 53 | 38 | 63 | 85 | 89 | 82 |
| $v_i$ | 92 | 57 | 49 | 68 | 60 | 43 | 67 | 84 | 87 | 72 |

$C = 165$. Optimum $V^* = 309$, items $\{1, 2, 3, 4, 6\}$, weight 165. Source: J. Burkardt, `knapsack_01` dataset.
