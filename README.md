# dbl-uncertain-reachability

An independent reimplementation exploring source-to-target (s-t) reachability
estimation on large-scale, **dynamic, uncertain directed graphs** — extending
a state-of-the-art deterministic reachability index to the probabilistic
setting.

> This project is inspired by academic research I conducted at IIT Jammu on
> uncertain graph reachability. The original codebase, datasets, and specific
> experimental results are confidential to the institution. This repository
> is a standalone, independently written reimplementation for demonstration
> purposes only — it does not contain any original source code, data, or
> design documents from that work.

---

## Problem Statement

Designing an efficient algorithm to answer s-t reachability queries on
**large, dynamic, uncertain directed graphs** — graphs where edges can be
inserted/deleted over time, and each edge carries a probability of existing.

In such graphs, the answer to an s-t reachability query is itself a
**probability**: the likelihood that a path exists from `s` to `t` across
all possible worlds implied by the edge probabilities. Computing this
exactly requires enumerating an exponential number of possible graphs,
making it intractable at scale — so approximate, sampling-based methods
are used instead.

---

## Approach

### 1. Base Index: DBL (Dynamic reachability via Bidirectional Leaf-DL labeling)
Built on top of **DBL**, a published state-of-the-art dynamic reachability
index for large directed graphs (Lyu et al., *"DBL: Efficient Reachability
Queries on Dynamic Graphs"*, arXiv:2101.09441). DBL avoids maintaining a full
DAG/topological structure and instead uses **Dynamic Landmark (DL)** labels
combined with **Bidirectional Leaf (BL)** labels to answer reachability
queries efficiently, while supporting fast edge insertions/deletions.

### 2. Extension to Uncertain Graphs
The deterministic DBL index answers "is `t` reachable from `s`?" with a
yes/no. To extend this to **uncertain graphs**, each edge `(u, v)` is
assigned an existence probability `p(u, v)`, and reachability is estimated
via **Monte Carlo sampling**:

- Sample the graph `N` times — each edge is included independently with
  probability `p(u, v)`.
- Run the DBL reachability check on each sampled ("possible world") graph.
- Estimate `P(s ⤳ t) ≈ (# samples where t is reachable) / N`.

### 3. Sample Size via Hoeffding's Inequality
Rather than choosing `N` arbitrarily, the required number of samples is
derived from **Hoeffding's Inequality**, based on a target error tolerance
`ε` and confidence level `1 - δ`:

```
N ≥ (1 / 2ε²) · ln(2 / δ)
```

This ensures the sampling-based probability estimate stays within `ε` of
the true value with probability at least `1 - δ`, avoiding both
under-sampling (poor accuracy) and over-sampling (wasted computation).

### 4. Edge Probability Assignment Schemes
Real-world datasets (e.g. from [SNAP](https://snap.stanford.edu/data/) or
[SocioPatterns](http://www.sociopatterns.org/datasets)) don't come with
edge probabilities, so probabilities are synthetically assigned using one
of three standard schemes from influence/uncertain-graph literature:

| Scheme | Rule |
|---|---|
| **Uniform** | All edges get the same fixed probability (e.g. 0.1 or 0.2) |
| **Trivalency** | Each edge's probability is drawn uniformly at random from `{0.1, 0.01, 0.001}` |
| **Weighted Cascade** | For edge `(u, v)`, probability = `1 / (deg(u) + deg(v))` |

### 5. Dynamic Query Workload
Queries are issued at runtime against the current graph snapshot; after a
batch of queries, edges are inserted/deleted, and further queries are run
— testing the index's ability to handle reachability estimation under
graph evolution without full recomputation.

---

## Project Structure

```
├── src/
│   ├── graph.hpp / .cpp             # Dynamic uncertain graph representation
│   ├── dbl_index.hpp / .cpp         # DBL reachability index (DL + BL labels)
│   ├── sampler.hpp / .cpp           # Monte Carlo sampling engine
│   ├── prob_assignment.hpp / .cpp   # Uniform / Trivalency / Weighted Cascade
│   └── sample_size.hpp              # Hoeffding's-inequality-based N calculation
├── benchmark/
│   ├── load_dataset.cpp             # SNAP / SocioPatterns dataset loader
│   └── benchmark.cpp                # Query throughput & accuracy measurement
├── results/
│   └── plots, benchmark logs
└── README.md
```

---

## Benchmarks

Measured on the real [SNAP Wiki-Vote dataset](https://snap.stanford.edu/data/wiki-Vote.html)
(7,115 nodes, 103,689 edges, directed) using C++ `<chrono>`, single-threaded,
in this sandboxed environment.

### Deterministic DBL index

| Metric | Value |
|---|---|
| Index build time (k=64 landmarks, k'=64 leaf buckets) | 525.9 ms |
| Query throughput | ~95.5M queries/sec (1M queries in 10.5 ms) |
| Avg. query latency | 0.0105 μs |
| Avg. edge-insertion update latency | 0.43 μs (2,000 random inserts) |

### Uncertain-graph Monte Carlo estimation (ε=0.02, δ=0.05 → N≈4,611 samples/query)

| Probability Scheme | Avg. Estimated P(s⤳t) | Avg. Time per Query |
|---|---|---|
| Uniform (p=0.1) | 0.0208 | 7,703.7 ms |
| Trivalency | 0.0003 | 5,336.9 ms |
| Weighted Cascade | 0.0000 | 3,711.9 ms |

*(Averaged over 5 random s-t query pairs. The Monte Carlo step re-samples
all ~103K edges per world, which dominates runtime — this is the
straightforward baseline implementation described above; not yet optimized
with techniques like reduced-variance sampling or incremental world
updates.)*

---

## Key Learnings

- How a deterministic reachability index (DBL) can be extended to a
  probabilistic setting via sampling, without redesigning the core index
- Using Hoeffding's Inequality to rigorously bound sample size against
  accuracy/confidence requirements, instead of guessing `N`
- Trade-offs across the three edge-probability assignment schemes and how
  they affect reachability estimates on real-world graphs
- Handling reachability queries under graph evolution (inserts/deletes)
  without full index recomputation

---

## References

- Lyu, B. et al. *"DBL: Efficient Reachability Queries on Dynamic Graphs."*
  [arXiv:2101.09441](https://arxiv.org/abs/2101.09441)
- Hoeffding, W. (1963). *"Probability Inequalities for Sums of Bounded
  Random Variables."*
- SNAP Datasets — https://snap.stanford.edu/data/
- SocioPatterns Datasets — http://www.sociopatterns.org/datasets

---

## License

MIT License — see [LICENSE](./LICENSE).
