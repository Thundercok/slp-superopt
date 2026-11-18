# Research Notes & Empirical Analysis — `slp-superopt`

## 1. Benchmark & Machine Specification (Santa 2026: The Elf Calculator)
- **Instances:** 1,000 scored puzzles (`scored_0001`..`scored_1000` in `data/puzzles.csv`) and 200 labeled practice puzzles (`practice_0001`..`practice_0200` in `data/practice_puzzles.csv`).
- **Machine Model:**
  - **Registers:** $M = 48$ registers indexed $0 \dots 47$ (`registers = 48` across all puzzles).
  - **Initial State:** If `starting_values` has $k$ elements $[v_0, v_1, \dots, v_{k-1}]$, registers $0 \dots k-1$ are initialized to those values; registers $k \dots 47$ are initially undefined.
  - **Instruction Format:** `[op, src1, src2, dst]` where `op` $\in \{\text{"+"}, \text{"-"}, \text{"*"}, \text{"/"}\}$, `src1, src2, dst` $\in \{0 \dots 47\}$.
  - **Semantics:**
    - `src1` and `src2` must hold defined values before the instruction executes.
    - Both `src1` and `src2` are read before `dst` is overwritten (`dst` may equal `src1` or `src2`; `src1` may equal `src2`).
    - Division `"/"` must be **exact** ($R[\text{src1}] \bmod R[\text{src2}] = 0$).
    - Resulting value written to `R[dst]` must be a **positive integer** ($1 \le R[\text{dst}] \le \text{value\_limit}$).
  - **Program Limits:**
    - Max instructions per puzzle: $4,096$.
    - Max UTF-8 bytes per CSV `program` cell: $131,072$ bytes (128 KiB).
    - Every target $t \in \text{targets}$ must be present in at least one register at least once (either initially or produced by an instruction).
  - **Objective:** Minimize the sum of instruction counts across all 1,000 scored puzzles. No public/private split.

---

## 2. Initial Empirical Findings (from `puzzles.csv`, `practice_puzzles.csv`, `sample_submission.csv`)
1. **64-bit Hardware Integer Fit:**
   - Across all 1,000 puzzles, `value_limit` $\le 10^{18}$ (max bit-length = 60 bits).
   - All values fit comfortably inside standard signed 64-bit integers (`int64`), allowing fast native arithmetic in Python/Numba/C++/Rust without arbitrary-precision overhead.
2. **Input & Target Cardinality (`puzzles.csv`):**
   - Number of `starting_values` ($k$): ranges from $1$ to $8$ (mostly $2, 3, 4, 5$).
   - Number of `targets` ($|T|$): ranges from $1$ to $32$ (median $= 30$).
   - Trivial information-theoretic lower bound per puzzle: $|T \setminus S|$ (1 instruction per non-initial target).
3. **Hidden Structural Families (Revealed by `practice_puzzles.csv`):**
   - `shared` (51/200): Targets share common intermediate subexpressions (e.g., two targets differing by a small offset or sharing common factors).
   - `mixed` (48/200): Combines multiplicative and additive chains.
   - `independent` (45/200): Targets generated with less shared sub-structure.
   - `multiplicative` (40/200): Targets dominated by products/powers of starting values.
   - `tiny` (16/200): Small target counts/values amenable to exact search.
4. **Why `sample_submission.csv` Scores `1,490,379` Steps:**
   - The sample baseline executes `["/", 0, 0, k]` to produce `1`, completely **ignores** all provided `starting_values`, and synthesizes each target independently from `1` via binary doubling (`+ r r r`) and increment (`+ r 1 r`), costing ~75 instructions per target ($\sim 2,300$ steps per puzzle).

---

## 3. Research & Engineering Roadmap
1. **Phase 1 — Verification & Archive Foundation:**
   - `src/evaluator.py`: Deterministic 48-register VM, legality checker, and scorer. Verify `sample_submission.csv` = `1,490,379`.
   - `src/archive.py`: Monotonic Best-Known Solution (BKS) archive tracking the shortest verified program per `scored_XXXX` puzzle and exporting valid submission CSVs.
2. **Phase 2 — Stronger Baselines (Exploiting `starting_values` & Shared Intermediates):**
   - **Base-Aware Decomposition:** Decompose targets using the actual `starting_values` (via division/remainder/factorization and multi-base representation) instead of building from `1`.
   - **Bidirectional / Meet-in-the-Middle Search:** Forward reachability set from `starting_values` (depth $d_1$) + backward reduction from `targets` (depth $d_2$).
3. **Phase 3 — DAG Synthesis & LNS Superoptimization (Core Paper Contribution):**
   - **Common Subexpression & Pairwise Distance Mining:** Identify shared intermediate nodes across the 30 targets (e.g., minimum spanning arborescence in arithmetic operation space).
   - **Destroy-and-Repair LNS under Register Pressure:** Iteratively destroy sub-chains of an existing valid SLP and resynthesize shorter sub-DAGs while maintaining 48-register liveness constraints.
