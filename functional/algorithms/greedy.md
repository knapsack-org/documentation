# `greedy` — ratio greedy with a best-single-item fix (heuristic)

| | |
|---|---|
| Type | heuristic with a guarantee: value $\ge V^*/2$; `proven_optimal` is always `false` |
| Time / memory | $O(n \log n)$ / $O(n)$ |
| Requires | positive weights (the ratio $v_i / w_i$ is formed) |
| Parameters | none (only `greedy.repeats`, see [parameters](../parameters.md#greedy)) |

## 1. Method

1. Sort the items by decreasing ratio $v_i / w_i$; **ties keep the index order** (stable sort).
2. Scan the items in that order and take every item that still fits in the remaining capacity (an item that does not fit is skipped and the scan goes on). Let $G$ be this packing and $V(G)$ its value.
3. Let $s$ be the item with the largest value among those that fit alone ($w_i \le C$), the lowest index on ties, and $V(s) = v_s$.
4. Return $s$ alone if $v_s > V(G)$, otherwise $G$.

Step 3–4 repairs the classical weakness of the greedy rule (one heavy valuable item left out); it makes $\max(V(G), v_s) \ge V^*/2$.

## 2. Stopping rule and limits

One pass; no limit.

## 3. Randomness

None.

## 4. Settings

| Key | Default | Meaning |
|---|---|---|
| `greedy.repeats` | `20` | timed runs per instance (all repeats give the same result) |

## 5. Reported values

| Field | Value |
|---|---|
| `algorithm` | `"Greedy by Value/Weight Ratio + Best Single Item"` |
| `exact` | `false` |
| `proven_optimal` | always `false` |
| `cost.operations` | $n$ items scanned; `cost.operations_unit` = `"items scanned"` |
| `details` | `greedy_value`: $V(G)$; `single_value`: $v_s$ ($0$ if no item fits); `chose_single`: $v_s > V(G)$ |

## 6. Acceptance checks

| # | Check |
|---|---|
| greedy-1 | on P01 the packing is feasible and its value is at least $309 / 2$; `operations` = 10 |
| greedy-2 | MATLAB and Python return the same items on every instance |
