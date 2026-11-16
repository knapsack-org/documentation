# `qubo_chz` — independent copy of `qubo` (heuristic, experimental)

| | |
|---|---|
| Status | **identical to [`qubo`](qubo.md)** since its creation (2026-11-14); meant to be modified |
| Purpose | experiment with the QUBO formulation or the solver without changing `qubo`, and compare both in the same run |
| Parameters | its own block `qubo_chz.penalty`, `qubo_chz.seed_offset`, `qubo_chz.tabu.*`, plus `qubo_chz.repeats` — see [parameters](../parameters.md#qubo_chz) |

## Specification

Everything in [`qubo`](qubo.md) applies — model, slack encoding, penalty, both tabu modes, decoding, reported fields — with these differences:

| Aspect | `qubo` | `qubo_chz` |
|---|---|---|
| Settings | `qubo.penalty`, `qubo.tabu` | `qubo_chz.penalty`, `qubo_chz.tabu` |
| Seed of the tabu stream for instance $k$ | $\texttt{seed} + 1000\,k$ | $\texttt{seed} + 1000\,k + \texttt{qubo\_chz.seed\_offset}$ |
| `algorithm` label | `QUBO + tabu search (<mode>)` | `QUBO CHZ + tabu search (<mode>)` |

With the default settings (`seed_offset` = 0 and the same tabu values), `qubo_chz` returns **exactly** the same packings, energies and iteration counts as `qubo`; only timings differ.

## Change procedure

When the method is changed:

1. Describe the new behaviour **on this page** (replace "Everything in `qubo` applies" by the actual differences, with formulas).
2. Change the `qubo_chz` code only (see the implementation pages).
3. The acceptance check "`qubo_chz` equals `qubo`" then fails by design: update or remove it in both implementations.
4. Keep Python and MATLAB in step so their results stay identical.
