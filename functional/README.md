# Functional specification

Language-independent description of the benchmark. Both implementations must produce **the same numbers** from these rules (exceptions: the [`ilp`](algorithms/ilp.md) item set when several optima exist, and timings/memory).

| Page | Content |
|---|---|
| [problem.md](problem.md) | The 0-1 knapsack problem, notation, conventions |
| [instances.md](instances.md) | Portable random generator and the three instance families |
| [algorithms/dp.md](algorithms/dp.md) | Dynamic programming — exact, reference optimum |
| [algorithms/bnb.md](algorithms/bnb.md) | Depth-first branch and bound with the Dantzig bound — exact |
| [algorithms/ilp.md](algorithms/ilp.md) | Integer linear program handed to a MILP solver — exact |
| [algorithms/qubo.md](algorithms/qubo.md) | QUBO penalty model + multi-start tabu search — heuristic |
| [algorithms/qubo_chz.md](algorithms/qubo_chz.md) | Independent copy of `qubo` for experiments — heuristic |
| [algorithms/greedy.md](algorithms/greedy.md) | Ratio greedy + best single item — heuristic, ½-approximation |
| [algorithms/sa.md](algorithms/sa.md) | Simulated annealing on the item bits — heuristic |
| [algorithms/ga.md](algorithms/ga.md) | Genetic algorithm with greedy repair — heuristic |
| [algorithms/ising.md](algorithms/ising.md) | Ising Hamiltonian of the QUBO + spin annealing — heuristic |
| [algorithms/fptas.md](algorithms/fptas.md) | Profit-scaling FPTAS, $(1-\varepsilon)$-approximation |
| [parameters.md](parameters.md) | All parameters, defaults and effects |
| [metrics.md](metrics.md) | Quality and execution-cost measures, summary statistics |
| [output-format.md](output-format.md) | JSON output contract (`schema_version` 2) |
| [acceptance-tests.md](acceptance-tests.md) | Checks every implementation must pass |

## Benchmark run in one paragraph

Generate `instances.count` instances ([instances](instances.md)). For each instance $k$: compute the reference optimum with `dp` (untimed), then run every selected algorithm `repeats` times, keep its last result, and measure quality and cost ([metrics](metrics.md)). Then aggregate per algorithm and per family, and write one JSON file ([output format](output-format.md)).

## Algorithm summary

| Id | Method | Exact | Complexity (time) | Memory | `operations` unit |
|---|---|---|---|---|---|
| `dp` | Dynamic programming table | yes, always proven | $O(nC)$ | $O(nC)$ | table cells |
| `bnb` | Depth-first B&B, Dantzig bound | yes, unless node limit hit | $O(2^n)$ worst case | $O(n)$ | nodes |
| `ilp` | MILP solver | yes, by solver status | solver-dependent | solver-dependent | solver B&B nodes |
| `qubo` | QUBO + tabu search | no | $O(\text{restarts}\cdot\text{iterations}\cdot n)$ | $O((n+K)^2)$ | move evaluations |
| `qubo_chz` | copy of `qubo` | no | same as `qubo` | same | move evaluations |
| `greedy` | ratio greedy + best single item | no (½-approximation) | $O(n \log n)$ | $O(n)$ | items scanned |
| `sa` | simulated annealing on item bits | no | $O(\text{restarts}\cdot\text{steps})$ | $O(n)$ | move evaluations |
| `ga` | genetic algorithm, greedy repair | no | $O(\text{generations}\cdot\text{pop}\cdot n)$ | $O(\text{pop}\cdot n)$ | fitness evaluations |
| `ising` | Ising Hamiltonian + spin annealing | no | $O(\text{restarts}\cdot\text{steps}\cdot N)$ worst case | $O((n+K)^2)$ | move evaluations |
| `fptas` | profit-scaling DP, $(1-\varepsilon)$-approximation | no | $O(n^2/\varepsilon)$ | $O(n^2/\varepsilon)$ | table cells |
