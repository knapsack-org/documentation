# `bnb` — branch and bound (exact)

| | |
|---|---|
| Type | exact; optimality proven unless the node limit is reached |
| Search | depth-first, iterative backtracking (no recursion, no stored tree) |
| Bound | Dantzig bound (LP relaxation) |
| Time / memory | $O(2^n)$ nodes worst case / $O(n)$ |
| Parameters | `bnb.max_nodes` (default $5\cdot10^6$), `bnb.repeats` — see [parameters](../parameters.md#bnb) |

## 1. Pre-processing

Sort the items by **decreasing ratio** $v_i / w_i$ with a **stable** sort (equal ratios keep the index order). Below, items are numbered $1 \dots n$ in that order. Prefix sums $\bar W_k = \sum_{i \le k} w_i$, $\bar V_k = \sum_{i \le k} v_i$ ($\bar W_0 = \bar V_0 = 0$) make each bound $O(n)$ at most.

## 2. Initial lower bound (greedy)

Scan the sorted items once and take every item that still fits (the scan **continues** after an item that does not fit). The value of this packing is the initial incumbent `best`.

## 3. Upper bound at a node

A node is a partial packing: items $1 \dots k-1$ decided, current load $L$, value $V$, residual capacity $r = C - L$. Let the **critical item** $j$ be the first item $j \ge k$ such that $\bar W_j - \bar W_{k-1} > r$ (items $k \dots j-1$ fit together, item $j$ does not).

$$
ub =
\begin{cases}
V + \bar V_n - \bar V_{k-1} & \text{if no critical item (all remaining items fit)},\\[2pt]
V + (\bar V_{j-1} - \bar V_{k-1}) + v_j\,\dfrac{r - (\bar W_{j-1} - \bar W_{k-1})}{w_j} & \text{otherwise.}
\end{cases}
$$

This is the optimum of the continuous (fractional) relaxation of the remaining sub-problem.

**Pruning rule:** values are integers, so the node is pruned when $\lfloor ub + 10^{-9} \rfloor \le \text{best}$ (the $10^{-9}$ absorbs floating-point error).

## 4. Search loop

```
k ← 1, x ← 0, L ← 0, V ← 0, nodes ← 0
loop
    nodes ← nodes + 1
    if nodes > max_nodes: stop, proven ← false
    if V > best: best ← V, record x
    if k > n                      → backtrack
    else if node pruned (§3)      → backtrack
    else if L + w_k ≤ C           → take item k   (x_k ← 1, update L, V), k ← k + 1
    else                          → skip item k   (x_k ← 0), k ← k + 1
    backtrack: let i = last taken item among 1..k-1
               if none: stop (tree exhausted, proven ← true)
               x_i ← 0, L ← L − w_i, V ← V − v_i, k ← i + 1   (explore the "skip i" branch)
```

"Take" is always tried before "skip". An item that does not fit is skipped without creating a branch.

## 5. Output

Items are mapped back to their original indices and sorted ascending.

| Field | Value |
|---|---|
| `proven_optimal` | `true` if the tree was exhausted, `false` if the node limit stopped the search (the best packing found so far is still returned) |
| `operations` | number of loop iterations (`nodes`) |
| `details` | `nodes`, `node_limit` (= `bnb.max_nodes`), `node_limit_reached` (= not proven) |
| `working_memory_bytes` | the $O(n)$ vectors: inputs, order, sorted copies, prefix sums, $x$, incumbent |

## 6. Behaviour by family (default run, 100 instances)

| Family | Mean nodes | Max nodes |
|---|---|---|
| uncorrelated | 73 | 152 |
| weakly correlated | 140 | 377 |
| strongly correlated | 14 608 | 284 157 (instance 96) |

Total over the 100 instances: 489 151 nodes; median 114. Near-equal ratios make the Dantzig bound weak, so strongly correlated instances dominate the cost.
