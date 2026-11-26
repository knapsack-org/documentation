# `qubo` — QUBO variant A: QUBO model + tabu search (heuristic)

Variant A of the QUBO approach (label `QUBO A + Tabu Search (<mode>)`, short label `QUBO A`). [Variant B](qubo_b.md) and [QC](qc.md) run the same method today.

| | |
|---|---|
| Type | heuristic; `proven_optimal` is always `false` |
| Pipeline | **build** the QUBO → **solve** it with a multi-start tabu search → **decode** (and repair) the packing |
| Variables | $n + K$ binary variables ($n$ items, $K$ slack bits) |
| Parameters | `qubo.penalty`, `qubo.tabu.*`, `qubo.repeats` — see [parameters](../parameters.md#qubo) |

A QUBO (*Quadratic Unconstrained Binary Optimisation*) problem is $\min_{x \in \{0,1\}^N} x^\top Q x + c^\top x + d$. Constraints are not allowed, so the capacity constraint becomes a penalty.

## 1. Model

### 1.1 Penalty form

With a slack variable $s \in \{0, \dots, C\}$ turning $W(x) \le C$ into the equality $W(x) + s = C$:

$$
f(x, s) = -\sum_{i=1}^{n} v_i x_i \;+\; P\Big(\sum_{i=1}^{n} w_i x_i + s - C\Big)^2 .
$$

### 1.2 Slack encoding

$s$ is written on

$$
K = \max\big(1,\ \lceil \log_2 (C+1) \rceil\big)
$$

bits $y_1 \dots y_K$ with weights

$$
b_j = 2^{j-1}\ \ (j = 1 \dots K-1),\qquad b_K = C - (2^{K-1} - 1),\qquad s = \sum_{j=1}^{K} b_j\,y_j .
$$

The last weight is truncated so that the largest representable value is exactly $C$; every $s \in \{0, \dots, C\}$ is representable. This is the encoding of MATLAB's `knapsack2qubo`.

**Encoding a value** $s$ (used to write the final slack bits): if $s > 2^{K-1} - 1$, set $y_K = 1$ and $s \leftarrow s - b_K$; then $y_j = \lfloor s / 2^{j-1} \rfloor \bmod 2$ for $j = 1 \dots K-1$.

Example: $C = 17$ → $K = 5$, weights $(1, 2, 4, 8, 2)$.

### 1.3 Matrix form

Let $z = [x;\ y] \in \{0,1\}^{n+K}$ and $a = [w;\ b]$ (item weights, then slack weights). Expanding the square and using $z_i^2 = z_i$:

$$
f(z) = z^\top Q z + c^\top z + d,
$$

$$
Q = P\,\big(a a^\top - \operatorname{diag}(a \circ a)\big)\ \ \text{(symmetric, zero diagonal)},
\qquad
c = \begin{bmatrix} -v \\ 0_K \end{bmatrix} + P\,\big(a \circ a - 2C\,a\big),
\qquad
d = P\,C^2 ,
$$

where $\circ$ is the element-wise product. $Q$ stores both triangles, so each pair $i \ne j$ contributes $2 Q_{ij} z_i z_j$.

### 1.4 Penalty

| `penalty` | $P$ |
|---|---|
| `"auto"` (default) | $\max_i v_i + 1$ |
| a number | that number |

With $P > \max_i v_i$, any infeasible packing costs more than it can gain, so the **QUBO minimum is the knapsack optimum** (energy $= -V^*$). A smaller $P$ can make an infeasible packing the minimum.

## 2. Solver — tabu search, mode `slack-optimal` (default)

**Key idea.** For fixed item bits $x$ with weight $W$ and value $V$, the slack that minimises $f$ is $s^* = C - W$ if $W \le C$, else $s^* = 0$. Minimising over the slack exactly leaves an energy on the item bits only:

$$
E(x) = -V(x) + P\cdot\max\big(0,\ W(x) - C\big)^2 .
$$

The search therefore flips **item bits only**; the slack bits are always at their optimal value, so every visited point is a valid QUBO assignment. (Flipping slack bits one at a time, as in `all-bits` mode, gets stuck in deep local minima.)

**Settings.** $N = n$ searched bits; per restart at most $\text{max\_iter} = \texttt{max\_iter\_factor}\cdot N$ iterations; stall limit $\texttt{stall\_factor}\cdot N$; tabu tenure $T = \texttt{tenure\_min} + \operatorname{round}(N/10)$; `restarts` independent starts.

**Random stream.** A dedicated generator ([portable LCG](../instances.md#portable-random-generator)) is seeded per instance with $\texttt{seed} + 1000\,k$ ($k$ = 1-based instance id). Each restart draws $N$ uniforms $u_i$ from the same stream (the state carries over between restarts) and starts from $x_i = [u_i < 0.5]$.

**Algorithm.**

```
best ← empty packing, E_best ← 0
repeat `restarts` times
    x ← random start;  E ← E(x);  E_run ← E;  if E < E_best: record x
    tabu_until ← 0 for every item;  stall ← 0
    for it = 1 .. max_iter
        for every item i: Δ_i ← E(x with bit i flipped) − E          (O(1) each, using W and V)
        allowed_i ← (tabu_until_i < it) or (E + Δ_i < E_best)        (aspiration)
        if no item allowed: allow all
        j ← argmin over allowed of Δ_i  (lowest index on ties)
        flip x_j;  E ← E + Δ_j;  tabu_until_j ← it + T
        if E < E_best: E_best ← E, record x
        if E < E_run: E_run ← E, stall ← 0
        else: stall ← stall + 1; if stall > stall limit: end this restart
    iterations ← iterations + it
return z = [best; encode(s*(best))] and its energy recomputed as zᵀQz + cᵀz + d
```

The energy is recomputed from $Q, c, d$ so the result can be checked against the model; for a feasible best packing it equals $-V$.

**Cost.** `operations` = move evaluations = $\text{iterations} \times N$ (one $\Delta_i$ per item per iteration).

## 3. Solver — tabu search, mode `all-bits` (comparison)

The textbook single-bit-flip tabu search on all $N = n + K$ bits of $f(z)$, kept for comparison:

- start: $z_i = [u_i < 0.5]$ for all $N$ bits; incumbent $z = 0$ with $f = d$;
- local field $g = c + 2Qz$; flipping bit $i$ changes $f$ by $\Delta_i = (1 - 2z_i)\,g_i$;
- after flipping bit $j$ (with $\sigma = 1 - 2z_j$ taken before the flip): $g \leftarrow g + 2\sigma\,Q_{:,j}$;
- same tabu, aspiration, stall and restart rules as above, with $N = n + K$;
- `operations` = iterations × $(n + K)$.

The returned $z$ may be infeasible; decoding repairs it.

## 4. Decoding and repair

1. Take the item part: item $i$ is taken when $\operatorname{round}(z_i) = 1$. This gives the **raw** packing (`raw_items`, `raw_value`, `raw_weight`, `feasible_before_repair`).
2. If the raw packing is overweight, **repair**: repeatedly remove the taken item with the lowest ratio $v_i / w_i$ (lowest index on ties) until $W \le C$; `repaired` = `true`.
3. The **final** packing (repaired or not) is the algorithm's result (`items`, `value`, `weight`).

At the default settings the slack-optimal search returns a feasible packing on all 100 instances (no repair).

## 5. Reported values

| Field | Value |
|---|---|
| `proven_optimal` | always `false` (a heuristic can still be optimal: see `is_optimal` in [metrics](../metrics.md)) |
| `operations` | move evaluations |
| `details.penalty` | $P$ |
| `details.variables`, `details.slack_bits` | $n + K$, $K$ |
| `details.model_memory_bytes` | size of $Q$, $c$, $d$ |
| `details.energy` | QUBO energy of the returned $z$ |
| `details.iterations` | total tabu iterations over all restarts |
| `details.raw_items`, `raw_value`, `raw_weight`, `feasible_before_repair`, `repaired` | decoding, before repair (§4) |
| `details.build_time_s`, `solve_time_s`, `decode_time_s` | time of each pipeline stage (last repeat) |
| `working_memory_bytes` | $Q$, $c$ and the search vectors: $\approx 8(n+K)^2$ bytes for $Q$ |

## 6. Default results (100 instances)

44 / 100 optimal, mean gap 1.02 %, max gap 4.80 %, all feasible. See [acceptance tests](../acceptance-tests.md#expected-benchmark-results).

## References

- A. Lucas, *Ising formulations of many NP problems*, Frontiers in Physics 2 (2014) — penalty/slack QUBO formulations.
- F. Glover, *Tabu search*, ORSA Journal on Computing (1989–1990).
- MathWorks, [`knapsack2qubo`](https://www.mathworks.com/help/matlab/ref/knapsack2qubo.html) — same model (Quantum Computing support package, R2025b).
