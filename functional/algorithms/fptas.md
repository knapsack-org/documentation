# `fptas` — profit-scaling FPTAS (approximation scheme)

| | |
|---|---|
| Type | approximation scheme: value $\ge (1 - \varepsilon)\,V^*$; `proven_optimal` is always `false` |
| Time / memory | $O(n^2 / \varepsilon)$ / $O(n^2 / \varepsilon)$ (a $n' \times (S{+}1)$ table is kept for backtracking) |
| Requires | nothing beyond the instance |
| Parameters | block `fptas`: `epsilon`, `repeats` — see [parameters](../parameters.md#fptas) |

## 1. Method (Ibarra–Kim)

Let $F = \{i : w_i \le C\}$, $n' = |F|$ and $v_{\max} = \max_{i \in F} v_i$. Scale the values:

$$
s_i = \Big\lfloor \frac{v_i\, n'}{\varepsilon\, v_{\max}} \Big\rfloor \quad (i \in F),
\qquad S = \sum_{i \in F} s_i,
\qquad \text{scale } K = \frac{\varepsilon\, v_{\max}}{n'} .
$$

$\text{minw}(p)$ = least weight of a subset of $F$ with scaled profit exactly $p$, for $p = 0 \dots S$. Start with $\text{minw}(0) = 0$ and $+\infty$ elsewhere; for each item $i \in F$ in index order with $s_i > 0$, for all $p \ge s_i$: if $\text{minw}(p - s_i) + w_i < \text{minw}(p)$ (**strictly**) then set $\text{minw}(p)$ to it and remember that item $i$ improved entry $p$. Items with $s_i = 0$ are never taken.

The answer is the largest $p$ with $\text{minw}(p) \le C$; the items are recovered by backtracking through the remembered improvements ($i = n' \dots 1$: if item $i$ improved entry $p$, take it and $p \leftarrow p - s_i$).

**Guarantee.** Rounding down loses less than $K$ per item, so the packing found has value at least $V^* - n' K = V^* - \varepsilon v_{\max} \ge (1 - \varepsilon)\,V^*$.

## 2. Stopping rule and limits

None; the table size $n'(S+1)$ grows as $\varepsilon$ shrinks ($S \le n'^2 / \varepsilon$). If no item fits alone the result is the empty packing.

## 3. Randomness

None.

## 4. Settings

| Key | Default | Meaning |
|---|---|---|
| `fptas.epsilon` | `0.1` | accuracy: value $\ge (1 - \varepsilon) V^*$; smaller = slower and more accurate |
| `fptas.repeats` | `5` | timed runs per instance (all repeats give the same result) |

## 5. Reported values

| Field | Value |
|---|---|
| `algorithm` | `"FPTAS (Profit Scaling)"` |
| `exact` / `proven_optimal` | `false` / always `false` |
| `cost.operations` | table cells $n'(S+1)$; `cost.operations_unit` = `"table cells"` |
| `details` | `epsilon`; `scale` ($K$); `scaled_profit_total` ($S$); `table_cells` |

## 6. Acceptance checks

| # | Check |
|---|---|
| fptas-1 | on P01, for $\varepsilon \in \{0.05, 0.2, 0.5, 1\}$ the packing is feasible with value $\ge (1 - \varepsilon) \cdot 309$ |
| fptas-2 | on P01 with $\varepsilon = 0.01$ the packing is items 1 2 3 4 6 (value 309) |
| fptas-3 | MATLAB and Python return the same items |
