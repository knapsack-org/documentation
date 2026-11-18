# `sa` — simulated annealing on the item bits (heuristic)

| | |
|---|---|
| Type | heuristic; `proven_optimal` is always `false` |
| Time / memory | $O(\text{restarts} \cdot \text{steps})$ / $O(n)$, one move costs $O(1)$ |
| Requires | nothing beyond the instance |
| Parameters | block `sa`: `penalty`, `restarts`, `steps_factor`, `t0_factor`, `t_end_factor`, `repeats` — see [parameters](../parameters.md#sa) |

## 1. Method

State $x \in \{0,1\}^n$ with weight $W(x)$ and value $V(x)$; energy (the same penalty form as [`qubo`](qubo.md) with the slack minimised out)

$$
E(x) = -V(x) + P\,\max\big(0,\ W(x) - C\big)^2,
\qquad P = \max_i v_i + 1 \ (\text{`"auto"`}).
$$

Each of `restarts` runs: start from $x_i = [u_i < 0.5]$ (one uniform per item), then do $T_{\text{steps}} = \lfloor \texttt{steps\_factor} \cdot n \rfloor$ proposals. A proposal flips item $j = \lfloor u\,n \rfloor + 1$ and is accepted if $\Delta E \le 0$, or else if a new uniform $u' < \exp(-\Delta E / T)$. $\Delta E$ is computed in $O(1)$ from $W$ and $V$.

The temperature falls geometrically from $T_0 = \texttt{t0\_factor} \cdot \max_i v_i$ to $T_1 = \texttt{t\_end\_factor} \cdot \max_i v_i$:
$T \leftarrow \alpha T$ after every proposal, $\alpha = (T_1 / T_0)^{1/(T_{\text{steps}} - 1)}$.

The result is the **best feasible** packing visited ($W \le C$, highest value; the empty packing at least). No repair is needed.

## 2. Stopping rule and limits

Fixed number of proposals: $\texttt{restarts} \cdot T_{\text{steps}}$.

## 3. Randomness

One [portable LCG](../instances.md#portable-random-generator) stream seeded with the per-instance seed $\text{seed} + 1000\,k$. Draw order: $n$ uniforms for the start of a restart, then per proposal one uniform for $j$ and, only when $\Delta E > 0$, one for the acceptance test.

## 4. Settings

| Key | Default | Meaning |
|---|---|---|
| `sa.penalty` | `"auto"` | $P$: `"auto"` = $\max v + 1$, or a number |
| `sa.restarts` | `10` | independent annealing runs |
| `sa.steps_factor` | `500` | proposals per restart = factor $\cdot\, n$ |
| `sa.t0_factor` | `1.0` | start temperature = factor $\cdot \max v$ |
| `sa.t_end_factor` | `0.01` | end temperature = factor $\cdot \max v$ |
| `sa.repeats` | `1` | timed runs per instance |

## 5. Reported values

| Field | Value |
|---|---|
| `algorithm` | `"Simulated Annealing"` |
| `exact` / `proven_optimal` | `false` / always `false` |
| `cost.operations` | proposals; `cost.operations_unit` = `"move evaluations"` |
| `details` | `penalty`, `proposals`, `accepted` (accepted moves), `t_start`, `t_end` |

## 6. Acceptance checks

| # | Check |
|---|---|
| sa-1 | feasible packing on P01; same seed → same items |
| sa-2 | MATLAB and Python return the same items (the generator and the loop are identical; the only floating-point operations are `exp` and `^`) |
