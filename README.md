# SIMD Architecture Handbook

**Designing vectorized algorithms through dependency analysis, loop transformation, data movement, and empirical architecture search.**

This is a **function-agnostic, programming-language-independent, instruction-set-independent** handbook. All executable-style examples use **abstract pseudocode**, not intrinsics or C/C++ SIMD implementations.

> Vectorization is not primarily a matter of replacing scalar operations with wider instructions. It is a matter of choosing **which work is simultaneous, which data is stationary, what can be reordered, and where partial results live**.

## Start here

- [Read the handbook](book/README.md) — 24 linked chapters, from first principles to worked transformations.
- [Start with the design procedure](book/21-architecture-search.md) — how to compare loop schedules and tiles.
- [Browse abstract case studies](book/23-case-studies.md) — transformations across algorithm classes.
- [Use the design worksheet](book/24-practice.md) — architecture checklist, exercises, and answer sketches.

## What you will learn

1. Model a computation using **independent problems, outputs, reductions, state, and dependencies**.
2. Choose a lane mapping and evaluate loop structures with **zero to arbitrary depth**.
3. Transform legal loop nests through **interchange, tiling, fusion/fission, skewing, unroll-and-jam, and scheduling**.
4. Balance **register lifetime, reuse, cache footprint, access regularity, lane communication, and instruction-level parallelism**.
5. Reorganize **maps, reductions, scans, neighborhoods, irregular updates, searches, and stateful computations**.
6. Test correctness, build a performance model, and **measure competing architectures** rather than assume a universal optimum.

## Organization

The handbook is divided into five parts:

| Part | Subject |
| --- | --- |
| I (1–5) | Computational model, legality, lanes, loop depths, transformations |
| II (6–11) | Stationary data, tiling, layouts, lane movement, control flow, irregular accesses |
| III (12–18) | Algorithm-family transformations |
| IV (19–22) | Correctness, performance modeling, architecture search, portability |
| V (23–24) | Worked abstract case studies, exercises, design worksheet |

## Earlier documents

The original material is preserved, not overwritten:

- [SIMD Loop Architecture — grounded in **ssbm**](references/SIMD_Loop_Architecture_SSBM.md)
- [SIMD Loop Architecture — algorithm-agnostic first edition](references/SIMD_Loop_Architecture_Algorithm_Agnostic.md)

The **ssbm** guide is a source-based worked example. The new handbook does **not** depend on that project and does not use its functions as examples.

## Conventions

- **P** — independent problem or task
- **O** — output item
- **R** — reduction or contribution
- **S** — dependent state or step
- **V** — number of active vector lanes
- **T** — tile extent (with a subscript for a dimension)
- **EVALUATE, COMBINE, FINALIZE** — abstract operations whose algebraic properties must be specified

Pseudocode operators such as VECTOR, LOAD, MASK, SHUFFLE, and REDUCE describe **semantic work**, not instructions a particular processor must possess.

**Performance statements are hypotheses unless benchmarked on a specified workload and platform.** Numeric examples illustrate reasoning, not guaranteed speedups.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). The most useful additions are architecture comparisons that give the reference semantics, dependency proof, at least two candidate schedules, predicted resource costs, and correctness tests.

## License

A license has not yet been selected. Do not infer a license from public GitHub visibility.
