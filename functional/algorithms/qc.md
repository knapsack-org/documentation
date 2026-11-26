# `qc` — QC: QUBO + tabu search, with QAOA and greedy modules (heuristic)

| | |
|---|---|
| Type | heuristic; `proven_optimal` is always `false` |
| Time / memory | active path: as [`qubo`](qubo.md); QAOA module: $O(\text{evaluations}\cdot p \cdot N\,2^N)$ time, $O(N\,2^N)$ memory |
| Requires | integer capacity (slack encoding); QAOA: $N = n + K \le$ `qc.qaoa.max_qubits` |
| Parameters | block `qc`: `qc.penalty`, `qc.tabu.*` (as [`qubo`](../parameters.md#qubo)), `qc.qaoa.*`, `qc.repeats` — see [parameters](../parameters.md#qc) |

## 1. Method

The algorithm meant for quantum-computing experiments. Its **active path is the code of [QUBO variant B](qubo_b.md)** (so, today, of [`qubo`](qubo.md)): build the QUBO, minimise it with the multi-start tabu search, decode and repair. Its modules also hold two other solvers, whose calls are written **commented out** next to the tabu call, so switching is one line (plus the output description of their statistics, which differ):

| Solver | Input | Output |
|---|---|---|
| QUBO tabu search (active) | the QUBO $(Q, c, d)$ | best vector, its energy, search statistics |
| QAOA (§1.1) | the QUBO $(Q, c, d)$ | best sampled vector, its energy, angles |
| Greedy | the knapsack $(v, w, C)$, no QUBO | the packing of [`greedy`](greedy.md) |

### 1.1 QAOA (simulated)

The Quantum Approximate Optimisation Algorithm, simulated exactly on a state vector of $2^N$ complex amplitudes, $N = n + K$ qubits (bit $j$ of basis state $z$ is $x_j$):

1. energies $E(z) = z^\top Q z + c^\top z + d$ of every basis state, scaled to $\tilde E = (E - E_{\min}) / (E_{\max} - E_{\min}) \in [0, 1]$ (span 1 if all equal);
2. start state $|+\rangle^{\otimes N}$ (all amplitudes $2^{-N/2}$);
3. layer $\ell = 1 \dots p$: cost phase $\psi_z \leftarrow e^{-i\gamma_\ell \tilde E(z)}\psi_z$, then the mixer $e^{-i\beta_\ell X}$ on every qubit: for each pair $(a_0, a_1)$ of amplitudes that differ in that qubit, $(a_0, a_1) \leftarrow (\cos\beta\,a_0 - i\sin\beta\,a_1,\ -i\sin\beta\,a_0 + \cos\beta\,a_1)$;
4. objective $F(\gamma, \beta) = \sum_z |\psi_z|^2\,\tilde E(z)$, minimised by a **deterministic** search (both implementations take the same steps):
   - layer by layer, with the earlier layers fixed, every point of a $G \times G$ grid $\gamma = 2\pi(i - \tfrac12)/G$, $\beta = \pi(j - \tfrac12)/G$, $i, j = 1 \dots G$, in that order;
   - then `refine` rounds: for every layer, $\gamma_\ell + h_\gamma$, $\gamma_\ell - h_\gamma$, $\beta_\ell + h_\beta$, $\beta_\ell - h_\beta$ in that order, starting from $h = (2\pi/G, \pi/G)$, halved after each round;
   - a point replaces the best one only if $F$ is lower by more than $10^{-12}$;
5. `shots` samples $z$ from $|\psi_z|^2$ with the [benchmark generator](../instances.md#portable-random-generator) (seeded with the per-instance seed): sample $s$ is the first $z$ whose cumulative probability is $\ge u_s \cdot$ total; the result is the sample of lowest $E$ (first one on ties). It is then decoded and repaired as in [`qubo` §4](qubo.md#4-decoding-and-repair).

More qubits than `qc.qaoa.max_qubits` is an error (the default 20 needs ≈ 16 MB of amplitudes). Most benchmark instances ($n$ = 10–48, $K \approx 8$–11) are above it: QAOA is for small instances.

## 2. Stopping rule and limits

Active path: as [`qubo`](qubo.md). QAOA: fixed number of evaluations, $p\,G^2 + 4\,p\cdot\text{refine}$.

## 3. Randomness

The per-instance seed $\text{seed} + 1000\,k$: the tabu starts (as `qubo`), or the QAOA samples. With the same settings the active path returns **the same packing, energy and move counts as `qubo`**.

## 4. Settings

| Key | Default | Meaning |
|---|---|---|
| `qc.penalty`, `qc.tabu.*` | as `qubo` | [`qubo` settings](../parameters.md#qubo) |
| `qc.qaoa.layers` | `2` | depth $p$ |
| `qc.qaoa.shots` | `200` | samples of the final state |
| `qc.qaoa.grid` | `12` | $G$, grid points per angle |
| `qc.qaoa.refine` | `4` | coordinate-search rounds |
| `qc.qaoa.max_qubits` | `20` | largest $N$ accepted |
| `qc.repeats` | `1` | deterministic and slow |

The `qaoa` block is only used once the QAOA call is active.

## 5. Reported values

Active path: as [`qubo`](qubo.md#5-reported-values); `algorithm` = `"QC (QUBO + Tabu Search) (<mode>)"`. QAOA statistics (once active): evaluations (`iterations` = `move_evaluations`), the chosen $\gamma$, $\beta$ and $F$.

## 6. Acceptance checks

| # | Check |
|---|---|
| D3 | `qc` returns the same items, energy and move evaluations as `qubo` on P01 and instances 1–12 |
| D4 | the greedy module returns the same packing and info as [`greedy`](greedy.md) |
| D5 | QAOA on the MathWorks example ($v = (5, 2, 8, 4, 3)$, $w = (9, 8, 4, 5, 2)$, $M = 17$, $P = 9$, seed 1, defaults): $x = (0,0,1,1,1,0,1,1,0,0)$, energy $-15$, $\gamma \approx (3.468842, 7.003134)$, $\beta \approx (2.814343, 3.010693)$, $F \approx 0.02329397$, 320 evaluations; decoded items $\{3, 4, 5\}$, value 15 |
| D6 | QAOA with `max_qubits` 17 on P01 (18 qubits) is refused |
