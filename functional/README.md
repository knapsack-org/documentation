# Functional specification

Language-independent description of the benchmark. Both implementations must produce **the same numbers** from these rules (exceptions: the [`ilp`](algorithms/ilp.md) item set when several optima exist, and timings/memory).

| Page | Content |
|---|---|
| [problem.md](problem.md) | The 0-1 knapsack problem, notation, conventions |
| [instances.md](instances.md) | Portable random generator and the three instance families |
| [algorithms/dp.md](algorithms/dp.md) | Dynamic programming — exact, reference optimum |
| [algorithms/bnb.md](algorithms/bnb.md) | Depth-first branch and bound with the Dantzig bound — exact |
| [algorithms/ilp.md](algorithms/ilp.md) | Integer linear program handed to a MILP solver — exact |
| [algorithms/qubo.md](algorithms/qubo.md) | QUBO variant A: penalty model + multi-start tabu search — heuristic |
| [algorithms/qubo_b.md](algorithms/qubo_b.md) | QUBO variant B: same as A today; later the MathWorks model in MATLAB — heuristic |
| [algorithms/qc.md](algorithms/qc.md) | QC: QUBO B path, with simulated QAOA and greedy modules — heuristic |
| [algorithms/greedy.md](algorithms/greedy.md) | Ratio greedy + best single item — heuristic, ½-approximation |
| [algorithms/sa.md](algorithms/sa.md) | Simulated annealing on the item bits — heuristic |
| [algorithms/ga.md](algorithms/ga.md) | Genetic algorithm with greedy repair — heuristic |
| [algorithms/ising.md](algorithms/ising.md) | Ising Hamiltonian of the QUBO + spin annealing — heuristic |
| [algorithms/fptas.md](algorithms/fptas.md) | Profit-scaling FPTAS, $(1-\varepsilon)$-approximation |
| [parameters.md](parameters.md) | All parameters, defaults and effects |
| [metrics.md](metrics.md) | Quality and execution-cost measures, summary statistics |
| [output-format.md](output-format.md) | JSON output contract (`schema_version` 4), raw results, instance set, run request |
| [acceptance-tests.md](acceptance-tests.md) | Checks every implementation must pass |

## Benchmark run in one paragraph

Find the algorithms (one folder each, alphabetical; those that cannot run in this environment are skipped) and generate `instances.count` instances ([instances](instances.md)). For each instance $k$: run the reference algorithm `dp` first (its value is the optimum $V^*$), then every other algorithm; each runs `repeats` times, keeps its last result, and is measured on quality and cost ([metrics](metrics.md)). Without a [run request](output-format.md#run-request) every run uses every algorithm on every instance ([parameters](parameters.md)). The benchmark writes the raw results; knapsack-server's output service then computes the gaps and ratios, aggregates per algorithm and per family, and writes one JSON file ([output format](output-format.md)).

## Algorithm summary

| Id | Method | Exact | Complexity (time) | Memory | `operations` unit |
|---|---|---|---|---|---|
| `dp` | Dynamic programming table | yes, always proven | $O(nC)$ | $O(nC)$ | table cells |
| `bnb` | Depth-first B&B, Dantzig bound | yes, unless node limit hit | $O(2^n)$ worst case | $O(n)$ | nodes |
| `ilp` | MILP solver | yes, by solver status | solver-dependent | solver-dependent | solver B&B nodes |
| `qubo` | QUBO variant A + tabu search | no | $O(\text{restarts}\cdot\text{iterations}\cdot n)$ | $O((n+K)^2)$ | move evaluations |
| `qubo_b` | QUBO variant B + tabu search | no | as `qubo` | as `qubo` | move evaluations |
| `qc` | QUBO + tabu search (QAOA, greedy modules) | no | as `qubo` | as `qubo` | move evaluations |
| `greedy` | ratio greedy + best single item | no (½-approximation) | $O(n \log n)$ | $O(n)$ | items scanned |
| `sa` | simulated annealing on item bits | no | $O(\text{restarts}\cdot\text{steps})$ | $O(n)$ | move evaluations |
| `ga` | genetic algorithm, greedy repair | no | $O(\text{generations}\cdot\text{pop}\cdot n)$ | $O(\text{pop}\cdot n)$ | fitness evaluations |
| `ising` | Ising Hamiltonian + spin annealing | no | $O(\text{restarts}\cdot\text{steps}\cdot N)$ worst case | $O((n+K)^2)$ | move evaluations |
| `fptas` | profit-scaling DP, $(1-\varepsilon)$-approximation | no | $O(n^2/\varepsilon)$ | $O(n^2/\varepsilon)$ | table cells |
