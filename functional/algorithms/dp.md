# `dp` — dynamic programming (exact, reference)

| | |
|---|---|
| Type | exact; optimality always proven |
| Role | **reference algorithm**: its optimum $V^*$ is the yardstick for every other algorithm |
| Time / memory | $O(nC)$ / $O(nC)$ (the full table is kept for backtracking) |
| Requires | integer weights and capacity ($C$ is floored) |
| Parameters | none (only `dp.repeats`, see [parameters](../parameters.md#dp)) |

## Recurrence

$m(i, j)$ = best value using the first $i$ items with capacity $j$, for $i = 0 \dots n$, $j = 0 \dots C$:

$$
m(0, j) = 0,
\qquad
m(i, j) =
\begin{cases}
m(i-1, j) & \text{if } w_i > j,\\[2pt]
\max\big(m(i-1, j),\ m(i-1, j - w_i) + v_i\big) & \text{if } w_i \le j.
\end{cases}
$$

The optimum is $V^* = m(n, C)$. Rows are computed one at a time; within a row all capacities $j \ge w_i$ are updated at once (vectorised), which is valid because row $i$ reads only row $i-1$.

## Item recovery (backtracking)

Start with $j = C$; for $i = n, n-1, \dots, 1$: if $m(i, j) \ne m(i-1, j)$, item $i$ is taken and $j \leftarrow j - w_i$.

When several packings are optimal, this rule fixes **which** one is reported: it prefers not taking the highest-indexed items. The result is the instance's `optimal_items` in the [output](../output-format.md#instances).

## Reported values

| Field | Value |
|---|---|
| `proven_optimal` | always `true` |
| `operations` | $n\,(C + 1)$ table cells (`operations_unit` = `table cells`) |
| `details.table_cells` | same as `operations` |
| `working_memory_bytes` | dominated by the table: $8\,(n+1)(C+1)$ bytes in double precision, plus the input vectors |

## Example (P01)

$n = 10$, $C = 165$ → 1 660 table cells; $V^* = 309$, items $\{1, 2, 3, 4, 6\}$.
