# `<id>` — <full name> (<exact | heuristic>)

> Template for the functional page of a new algorithm. Copy to `functional/algorithms/<id>.md`, replace every `<…>`, delete this note. Steps for adding the algorithm to both implementations: [adding an algorithm](../../implementation/adding-an-algorithm.md).

| | |
|---|---|
| Type | <exact; optimality proven when … / heuristic; no optimality proof> |
| Time / memory | <$O(\dots)$ / $O(\dots)$> |
| Requires | <assumptions on the data, e.g. integer weights> |
| Parameters | block `<id>`: `<id>.<key>`, …, `<id>.repeats` — see [parameters](../parameters.md#<id>) |

## 1. Method

<What is computed, with formulas. No file names, function names or library names: those belong in `implementation/`.>

## 2. Stopping rule and limits

<When it stops; what happens when a limit is reached (e.g. `proven_optimal = false`).>

## 3. Randomness

<None, or: which streams it draws from. The runner passes the per-instance seed $\text{seed} + 1000\,k$; an algorithm may add its own offset.>

## 4. Settings

| Key | Default | Meaning |
|---|---|---|
| `<id>.<key>` | `<value>` | <effect> |
| `<id>.repeats` | `<n>` | timed runs per instance (all repeats give the same result) |

## 5. Reported values

| Field | Value |
|---|---|
| `algorithm` | `"<label>"` (also `algorithms[].label` in the [output](../output-format.md#algorithms)) |
| `exact` | `<true | false>` |
| `proven_optimal` | <when true> |
| `cost.operations` | <what is counted>; `cost.operations_unit` = `"<unit>"` |
| `details` | `<field>`: <meaning>, … |

## 6. Acceptance checks

| # | Check |
|---|---|
| <id>-1 | <e.g. finds 309 on P01> |
