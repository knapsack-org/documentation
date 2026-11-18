# `ising` — Ising Hamiltonian + simulated annealing (heuristic)

| | |
|---|---|
| Type | heuristic; `proven_optimal` is always `false` |
| Pipeline | **build** the Ising model → **solve** it by spin annealing → **decode** (and repair) the packing |
| Variables | $N = n + K$ spins ($n$ items, $K$ slack bits) |
| Time / memory | $O(\text{restarts} \cdot \text{steps} \cdot (1 + \text{acceptance} \cdot N))$ / $O(N^2)$ |
| Parameters | block `ising`: `penalty`, `restarts`, `steps_factor`, `t0_factor`, `t_end_factor`, `repeats` — see [parameters](../parameters.md#ising) |

The same optimisation problem as [`qubo`](qubo.md), written in spin variables and solved with the annealer used by Ising machines. Compared with [`sa`](sa.md) (items only, slack minimised out) it isolates the effect of the **formulation**: here the slack bits are ordinary spins.

## 1. Model

Take the QUBO of [`qubo` §1](qubo.md#1-model) with the same $P$ and the same slack encoding: $z = [x;\,y]$, $f(z) = z^\top Q z + c^\top z + d$, with $Q = P(aa^\top - \operatorname{diag}(a \circ a))$, $c = [-v;\,0] + P(a \circ a - 2Ca)$, $d = PC^2$, $a = [w;\,b]$.

Substitute $z_i = (1 + \sigma_i)/2$ with $\sigma_i \in \{-1, +1\}$:

$$
H(\sigma) = \tfrac12\,\sigma^\top J \sigma + h^\top \sigma + E_0,
\qquad
J = \tfrac12 Q,\quad
h = \tfrac12 Q\mathbf 1 + \tfrac12 c,\quad
E_0 = d + \tfrac14\,\mathbf 1^\top Q \mathbf 1 + \tfrac12\,\mathbf 1^\top c .
$$

$J$ is symmetric with a zero diagonal. $H(\sigma)$ **equals** the QUBO energy of $z = (1+\sigma)/2$ for every $\sigma$ (checked in the tests), so the minimum is still the knapsack optimum when $P > \max v$.

## 2. Solver — single-spin-flip annealing

Local field $g = h + J\sigma$. Flipping spin $j$ changes the energy by

$$
\Delta H_j = -2\,\sigma_j\,g_j ,
$$

and after an accepted flip $g \leftarrow g - 2\sigma_j J_{:,j}$ (using $\sigma_j$ before the flip), $\sigma_j \leftarrow -\sigma_j$.

Each of `restarts` runs: start from $\sigma_i = +1$ if $u_i < 0.5$ else $-1$; do $\lfloor \texttt{steps\_factor} \cdot N \rfloor$ proposals; a proposal picks spin $j = \lfloor u\,N \rfloor + 1$ and is accepted if $\Delta H \le 0$, or else if a new uniform $u' < \exp(-\Delta H / T)$. Temperature: geometric from $\texttt{t0\_factor} \cdot \Theta$ to $\texttt{t\_end\_factor} \cdot \Theta$ with $\Theta = P \max_i w_i$ (the energy scale of flipping one heavy item). The search must be very hot: the slack and item bits only trade off through the penalty term. The lowest $H$ visited is kept, whatever its feasibility.

## 3. Decoding and repair

Item $i$ is taken when $\sigma_i = +1$ (slack spins are ignored). If the packing is overweight, items are removed one at a time, lowest $v_i / w_i$ first (lowest index on ties), until $W \le C$. Raw and final packings are both reported, as for [`qubo`](qubo.md#4-decoding-and-repair).

## 4. Randomness

One [portable LCG](../instances.md#portable-random-generator) stream seeded with $\text{seed} + 1000\,k$: $N$ uniforms per restart start, then per proposal one uniform for $j$ and, only when $\Delta H > 0$, one for the acceptance test. All matrix entries are multiples of $1/4$ well below $2^{53}$, so the sums are exact in both languages.

## 5. Settings

| Key | Default | Meaning |
|---|---|---|
| `ising.penalty` | `"auto"` | $P$: `"auto"` = $\max v + 1$, or a number |
| `ising.restarts` | `5` | independent annealing runs |
| `ising.steps_factor` | `2000` | proposals per restart = factor $\cdot\, N$ |
| `ising.t0_factor` | `100` | start temperature = factor $\cdot\, P \max w$ |
| `ising.t_end_factor` | `1` | end temperature = factor $\cdot\, P \max w$ |
| `ising.repeats` | `1` | timed runs per instance |

## 6. Reported values

| Field | Value |
|---|---|
| `algorithm` | `"Ising Hamiltonian + Simulated Annealing"` |
| `exact` / `proven_optimal` | `false` / always `false` |
| `cost.operations` | proposals; `cost.operations_unit` = `"move evaluations"` |
| `details` | `penalty`, `variables` ($N$), `slack_bits` ($K$), `model_memory_bytes` ($J$, $h$, $E_0$), `energy` (lowest $H$ found), `proposals`, `accepted`, `raw_items`, `raw_value`, `raw_weight`, `feasible_before_repair`, `repaired`, `build_time_s`, `solve_time_s`, `decode_time_s` |

## 7. Known weakness

Single spin flips cannot move a slack value and the items together, so the search must pass through states with a large penalty: it needs a very hot start (`t0_factor` 100 × $P \max w$) and many steps, and is still the least accurate heuristic of the set (mean gap ≈ 6 % against under 1 % for `sa`). It is a property of the formulation, not a defect ([`qubo`'s `all-bits` mode](qubo.md#3-solver--tabu-search-mode-all-bits-comparison) has the same limit).

## 8. Acceptance checks

| # | Check |
|---|---|
| ising-1 | $H(\sigma)$ equals the QUBO energy of $(1+\sigma)/2$ on random vectors (MathWorks `knapsack2qubo` example 1); $J$ symmetric with zero diagonal; $K = 5$ |
| ising-2 | feasible packing on P01; same seed → same items |
| ising-3 | MATLAB and Python return the same items |
