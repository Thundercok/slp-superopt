# `slp-superopt`: Straight-Line Program Superoptimization under Register Constraints

Research repository for **Straight-Line Program (SLP) Synthesis and Superoptimization** under register pressure, evaluated on multi-target arithmetic synthesis instances (Santa 2026 Elf Calculator benchmark).

## 1. Problem Formulation
Given an initial register state $R_0$ across $M = 48$ bounded registers and a target set $T = \{t_1, t_2, \dots, t_k\} \subset \mathbb{Z}^+$, synthesize a minimum-length sequence of straight-line instructions:
$$P = \langle (\text{op}_i, s_{i,1}, s_{i,2}, d_i) \rangle_{i=1}^L, \quad \text{op}_i \in \{+, -, \times, \div\}$$
such that:
1. Every division is exact ($R[s_{i,1}] \bmod R[s_{i,2}] = 0$).
2. Every intermediate register value satisfies $1 \le R[d_i] \le V_{\max}$ (`value_limit`).
3. Every target $t \in T$ appears in at least one register during execution.
4. Total instruction count $\sum L$ across all 1,000 benchmark instances is minimized.

## 2. Repository Structure
```text
slp-superopt/
├── data/               # Benchmark datasets (puzzles.csv, sample_submission.csv)
├── submissions/        # Best solution archive & valid submission exports
├── src/
│   ├── evaluator.py    # Fast deterministic 48-register VM & legality verifier
│   ├── archive.py      # Per-instance best-known solution (BKS) tracker
│   └── solvers/        # Search, DAG synthesis, and LNS superoptimization solvers
└── tests/              # Unit tests for VM semantics & solver invariants
```
