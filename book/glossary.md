# Glossary and notation

[Handbook contents](README.md)

| Term | Meaning |
| --- | --- |
| **Iteration space** | Set of all logical computation points, independent of execution order |
| **Schedule** | Mapping from logical iterations to time, tiles, vector groups, lanes, and tasks |
| **P** | Independent problem/task dimension |
| **O** | Independent output dimension |
| **R** | Reduction/contribution dimension |
| **S** | Ordered state or step dimension |
| **V** | Logical number of simultaneous vector lanes |
| **T_X** | Tile extent along dimension X |
| **U** | Number of independent accumulator chains or unrolled work instances |
| **Lane mapping** | Meaning assigned to each vector lane (output, task, reduction, candidate, etc.) |
| **Stationary resource** | Data intentionally retained in registers or a cache while surrounding loops execute |
| **Output-stationary** | Keep output partials local through many contributions |
| **Input-stationary** | Retain input data while using it across many outputs |
| **Problem-stationary** | Retain context/state for a group of independent tasks |
| **Strip mining** | Split one axis into an outer tile index and an inner offset |
| **Loop interchange** | Change order of independent/permitted axes |
| **Fusion/fission** | Combine passes or separate statements into passes |
| **Unroll-and-jam** | Unroll outer work and combine the resulting inner bodies |
| **Wavefront** | Set of iterations executable when their predecessors are complete |
| **Halo** | Extra input region needed to calculate a spatial output tile |
| **Reduction** | Combination of many contributions into one result |
| **Scan** | Ordered prefix reduction producing an intermediate result for each position |
| **Segmented reduction** | Distinct reductions over variable-length segments |
| **Divergence** | Different lanes execute different work or require different step counts |
| **Predication** | Represent conditional execution via active-lane masks |
| **Compaction** | Pack active/selected lanes or work items together |
| **Gather/scatter** | Noncontiguous memory read/write indexed by lane values |
| **Horizontal reduction** | Combine multiple vector lanes into one result |
| **Lane communication** | Shift, shuffle, permute, broadcast, or reduce across lanes |
| **Register pressure** | Amount of simultaneously live vector and scalar state |
| **Spill** | Temporary state stored outside registers because capacity or allocation is insufficient |
| **Arithmetic intensity** | Useful computation per transferred byte at a specified memory level |
| **Workload regime** | Input sizes/shapes/distributions in which one schedule has characteristic performance |

## Abstract pseudocode vocabulary

| Construct | Semantic meaning |
| --- | --- |
| **TILES(domain, size)** | Partition a logical domain into nonoverlapping ordered or indexed tiles |
| **VECTOR_EVALUATE** | Evaluate several logically independent contributions or outputs |
| **VECTOR_COMBINE** | Apply a combination operation independently to corresponding lanes |
| **HORIZONTAL_REDUCE** | Merge values across lanes under specified algebraic semantics |
| **VECTOR_PREFIX_SCAN** | Compute all within-block prefixes |
| **MASKED_COMBINE** | Update only active lanes while preserving inactive states |
| **SAFE_LOAD / SAFE_GATHER** | Read only valid memory locations, including masked tails |
| **SAFE_STORE** | Commit only valid output lanes |
| **COMPRESS** | Pack selected values while specifying order |
| **GROUP_BY_KEY** | Reorganize items by a shared property; stability must be stated when required |
| **FINALIZE** | Convert an accumulator to its observable output result |
| **IDENTITY** | Neutral value under COMBINE (when one exists) |
| **DEPENDENCY_CLOSURE** | Set of inputs or states needed by a tile |
| **COMMIT** | Make a completed output observable according to the specified contract |
| **VALID_LANES** | Mask for logical elements actually represented by a partial tile |

## Important distinctions

- **Logical V** is not necessarily one physical vector register.
- **Associative math** may not imply identical floating-point execution results.
- **Logical repeated loads** are not automatically DRAM transfers; caches change physical traffic.
- **Distinct indices** do not guarantee independent writes if indirect addresses collide.
- **Masking a result** does not make an earlier invalid memory access safe.
- **More loops** can mean *less data movement* after tiling.
- **More arithmetic** can be faster when it avoids expensive data movement.
- **A faster microbenchmark** is not necessarily a faster pipeline.
- **A plausible architecture** is not a measured performance result.

[Begin reading](01-contract.md).
