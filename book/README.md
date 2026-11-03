# Handbook: Designing Vectorized Algorithms

An abstract, practical guide to inventing efficient vector execution architectures.

**Prefer one file?** Read the [complete, self-contained handbook](../HANDBOOK.md), containing all 24 chapters and the glossary.

**Read in order** if SIMD architecture is new to you; jump to Parts III–V if you already understand dependency graphs, tiling, and lane mappings.

## Part I — Model and transform the computation

1. [The semantic contract and iteration-space model](01-contract.md)
2. [Dependence analysis and transformation legality](02-dependencies.md)
3. [Mapping work to vector lanes](03-lane-mapping.md)
4. [Loop architectures from zero to N levels](04-loop-depths.md)
5. [The algebra of loop transformations](05-loop-transformations.md)

## Part II — Data movement and execution architecture

6. [Output-, input-, and problem-stationary schedules](06-stationary-architectures.md)
7. [Register, cache, and multi-level tiling](07-hierarchical-tiling.md)
8. [Memory access and data-layout co-design](08-memory-layout.md)
9. [Lane communication, reductions, and rearrangement](09-lane-communication.md)
10. [Predication, divergence, masks, and tails](10-control-flow.md)
11. [Irregular indexing, conflicts, and sparse work](11-irregular-work.md)

## Part III — Transform algorithm families

12. [Maps, independent outputs, and reductions](12-maps-reductions.md)
13. [Scans, recurrences, and dependency chains](13-scans-recurrences.md)
14. [Neighborhoods, wavefronts, and space-time schedules](14-neighborhoods-wavefronts.md)
15. [Search, selection, ordering, and comparison networks](15-search-selection.md)
16. [Segmented, sparse, and indirect algorithms](16-sparse-segmented.md)
17. [Stateful, variable-length, and convergent algorithms](17-stateful-work.md)
18. [Multi-stage pipelines, fusion, and recomputation](18-pipelines.md)

## Part IV — Prove and optimize

19. [Numerical correctness and semantic equivalence](19-correctness.md)
20. [Performance and communication models](20-performance-models.md)
21. [Systematic architecture search and measurement](21-architecture-search.md)
22. [Portable vectors and parallel hierarchy](22-portability.md)

## Part V — Practice

23. [15 abstract before/after case studies](23-case-studies.md)
24. [Design worksheet, exercises, and solutions](24-practice.md)

[Terminology and symbol glossary](glossary.md) · [Repository introduction](../README.md) · [First-edition references](../references/)

### How to read the examples

A line such as **VECTOR_EVALUATE(output_ids, reduction_id)** means “evaluate independent contributions for the listed outputs,” **not** a promise that a particular hardware instruction exists. A line such as **VECTOR_REDUCE** explicitly implies cross-lane communication. Each example needs the surrounding assumptions to be legal.

Where an implementation could change floating-point order, selection ties, aliasing behavior, or integer overflow semantics, this is called out explicitly. There is **no single fastest loop order** independent of data layout, data sizes, machine, and task.
