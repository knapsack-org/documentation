# `qubo_b` — QUBO variant B: QUBO model + tabu search (heuristic)

| | |
|---|---|
| Type | heuristic; `proven_optimal` is always `false` |
| Time / memory | as [`qubo`](qubo.md) |
| Requires | integer capacity (slack encoding) |
| Parameters | block `qubo_b`: same keys and defaults as [`qubo`](../parameters.md#qubo) — see [parameters](../parameters.md#qubo_b) |

## 1. Method

Variant B is today **the same method as variant A**, [`qubo`](qubo.md): the same model (§1), tabu searches (§2, §3), decoding and repair (§4) and reported values (§5), with the id `qubo_b` and the label `QUBO B + Tabu Search (<mode>)`.

It exists to host, in a later iteration, the vendor model of the MATLAB *Quantum Computing* support package in the MATLAB implementation (steps: [MATLAB implementation](../../implementation/matlab/README.md#qubo-variant-b-and-mathworks-knapsack2qubo)). Until then both implementations give variant A's results.

## 2. Stopping rule and limits

As [`qubo`](qubo.md#2-solver--tabu-search-mode-slack-optimal-default).

## 3. Randomness

The per-instance seed $\text{seed} + 1000\,k$, as `qubo`: with the same settings `qubo_b` returns **the same packing, energy and move counts as `qubo`** on every instance.

## 4. Settings

As [`qubo`](qubo.md), under `qubo_b.*` ([parameters](../parameters.md#qubo_b)).

## 5. Reported values

As [`qubo`](qubo.md#5-reported-values); `algorithm` = `"QUBO B + Tabu Search (<mode>)"`.

## 6. Acceptance checks

| # | Check |
|---|---|
| D3 | `qubo_b` returns the same items, energy and move evaluations as `qubo` on P01 and instances 1–12 ([acceptance tests](../acceptance-tests.md#d-correlation-settings-qubo-variant-b-qc)) |
