# `ga` — genetic algorithm with greedy repair (heuristic)

| | |
|---|---|
| Type | heuristic; `proven_optimal` is always `false` |
| Time / memory | $O(\text{generations} \cdot \text{pop\_size} \cdot n)$ / $O(\text{pop\_size} \cdot n)$ |
| Requires | positive weights (repair uses $v_i / w_i$) |
| Parameters | block `ga`: `pop_size`, `generations`, `tournament`, `crossover_rate`, `mutation_factor`, `repeats` — see [parameters](../parameters.md#ga) |

## 1. Method

**Chromosome**: a bit vector $x \in \{0,1\}^n$. **Fitness**: the packing value $V(x)$ (every chromosome is repaired, hence feasible).

**Repair** of a chromosome, with items ordered by decreasing $v_i / w_i$ (ties keep the index order):
1. while $W(x) > C$: remove the taken item with the **lowest** ratio (scan in reverse order);
2. then, for every item in decreasing ratio order, add it if it is not taken and still fits.

**Generation 0**: `pop_size` chromosomes, $x_i = [u_i < 0.5]$, repaired.

**Next generation** (`generations` times): keep the best individual (the first one on ties) as elite, then produce `pop_size − 1` children:
1. select two parents, each by a **tournament**: draw `tournament` individuals uniformly, the one with the highest fitness wins (first drawn on ties);
2. with probability `crossover_rate` apply uniform crossover (bit $i$ from parent 1 if its mask uniform is $< 0.5$, else from parent 2); otherwise the child copies parent 1;
3. flip every bit with probability $\texttt{mutation\_factor} / n$;
4. repair.

The result is the best individual seen in any generation.

## 2. Stopping rule and limits

Fixed number of generations.

## 3. Randomness

One [portable LCG](../instances.md#portable-random-generator) stream seeded with $\text{seed} + 1000\,k$. Generation 0 draws $\text{pop\_size} \cdot n$ uniforms in one block. Each child then draws **one block of $2 \cdot \texttt{tournament} + 1 + 2n$ uniforms, always in full**: the tournament candidates of parent 1, those of parent 2 (an index is $\min(\lfloor u \cdot \text{pop\_size} \rfloor, \text{pop\_size} - 1)$), one crossover decision, $n$ crossover masks, $n$ mutation draws.

## 4. Settings

| Key | Default | Meaning |
|---|---|---|
| `ga.pop_size` | `30` | individuals per generation |
| `ga.generations` | `50` | generations after the random start |
| `ga.tournament` | `2` | individuals drawn per parent selection |
| `ga.crossover_rate` | `0.9` | probability of uniform crossover |
| `ga.mutation_factor` | `1.0` | per-bit flip probability = factor / $n$ |
| `ga.repeats` | `1` | timed runs per instance |

## 5. Reported values

| Field | Value |
|---|---|
| `algorithm` | `"Genetic Algorithm (Greedy Repair)"` |
| `exact` / `proven_optimal` | `false` / always `false` |
| `cost.operations` | fitness evaluations: `pop_size` + `generations` $\cdot$ (`pop_size` − 1); `cost.operations_unit` = `"fitness evaluations"` |
| `details` | `evaluations`, `generations` |

## 6. Acceptance checks

| # | Check |
|---|---|
| ga-1 | feasible packing on P01; same seed → same items |
| ga-2 | MATLAB and Python return the same items (integer-valued fitness, same random draws) |
