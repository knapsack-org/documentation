# Instance families

The benchmark generates its instances; no data file is read. Same parameters ⇒ bit-identical instances in Octave, MATLAB and Python.

## Portable random generator

All randomness (instances **and** tabu-search starts) comes from the Park–Miller "minimal standard" linear congruential generator, not from the language's `rand`:

$$
m = 2^{31} - 1 = 2147483647,\qquad
s_{t+1} = 16807 \cdot s_t \bmod m,\qquad
u_{t+1} = \frac{s_{t+1}}{m} \in (0, 1).
$$

All products stay below $2^{53}$, so double-precision arithmetic is exact in every language.

| Function | Definition |
|---|---|
| seed → state | $s_0 = (\lvert \lfloor \text{seed} \rfloor \rvert \bmod 2147483646) + 1$, so $s_0 \in [1, 2^{31}-2]$ |
| uniform | next $u$ as above; drawing $k$ numbers advances the state $k$ times |
| integer in $[lo, hi]$ | $r = \min\big(lo + \lfloor u\,(hi - lo + 1) \rfloor,\ hi\big)$ |

## Generation procedure

Inputs: `seed` and the `instances` block: `count` ($N$), `min_items`, `max_items`, `max_coefficient` ($R$), `capacity_ratio` ($\rho$). See [parameters](parameters.md).

One generator state is started from `seed` and carried through all instances, in order. For instance $k = 1 \dots N$:

1. **Size** — grows linearly:
   $$n_k = \operatorname{round}\!\Big(n_{\min} + (n_{\max} - n_{\min})\,\frac{k-1}{N-1}\Big)\quad (N > 1),\qquad n_k = n_{\min}\ (N = 1).$$
2. **Family** — rotates: $k \bmod 3 = 1 \to$ `uncorrelated`, $2 \to$ `weakly_correlated`, $0 \to$ `strongly_correlated`.
3. **Weights** — $n_k$ draws: $w_i \sim U\{1, \dots, R\}$.
4. **Values** — depend on the family (draws, if any, come right after the weights):

| Family | Value law | Extra draws |
|---|---|---|
| `uncorrelated` | $v_i \sim U\{1, \dots, R\}$ | $n_k$ |
| `weakly_correlated` | $v_i = \max\big(w_i + \varepsilon_i,\ 1\big)$, $\varepsilon_i \sim U\{-\lfloor R/10 \rfloor, \dots, \lfloor R/10 \rfloor\}$ | $n_k$ |
| `strongly_correlated` | $v_i = w_i + \lfloor R/10 \rfloor$ | none |

5. **Capacity** — $C = \lfloor \rho \sum_i w_i \rfloor$.

## Defaults and resulting set

`seed` 2026, $N = 100$, $n$ from 10 to 48, $R = 100$, $\rho = 0.5$ → 34 uncorrelated, 33 weakly and 33 strongly correlated instances. Instance 1: uncorrelated, $n = 10$, $C = 285$, optimum 345.

## Why the families differ in hardness

| Family | Ratios $v_i / w_i$ | Typical behaviour |
|---|---|---|
| uncorrelated | spread out | greedy is close to optimal; B&B prunes fast |
| weakly correlated | moderately spread | medium |
| strongly correlated | nearly equal | many packings of near-equal value; the Dantzig bound is weak and B&B node counts explode (up to 284 157 nodes at default settings) |

[`dp`](algorithms/dp.md) is insensitive to the family: its cost depends only on $n \cdot C$.

Reference: D. Pisinger, *Where are the hard knapsack problems?*, Computers & Operations Research 32 (2005) — families as defined there; the benchmark uses three of them with small sizes.
