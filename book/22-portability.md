# 22. Portable vectors and parallel hierarchy

[Previous](21-architecture-search.md) · [Contents](README.md) · [Next](23-case-studies.md)

A hardware-agnostic handbook should still teach where hardware constraints enter the design. Express those constraints as parameters, not named intrinsics or instruction-set assumptions.

## 22.1 Logical width versus physical width

Let **V** be the number of logical elements processed as one vector group. On a particular target, the implementation may use multiple physical vector operations, a scalable vector size, or even scalar operations.

~~~text
V = QUERY_PREFERRED_VECTOR_GROUP_SIZE(element_type, workload)

FOR tile IN TILES(items, V)
    active = VALID_LANES(tile)
    VECTOR_PROCESS(tile, active)
~~~

The algorithm remains correct for any V ≥ 1. Performance still depends on element width, packing, register capacity, and how the target implements masks and rearrangements.

## 22.2 Width-agnostic loops

Avoid hardcoding that every input length is a multiple of some width:

~~~text
base = 0
WHILE base < N
    active = MIN(V, N - base)
    values = SAFE_LOAD_ACTIVE(input, base, active)
    result = VECTOR_EVALUATE(values, active)
    SAFE_STORE_ACTIVE(output, base, result, active)
    base += active
~~~

A width-agnostic schedule needs correct semantics for partial groups. It may still use target-specific specializations for performance behind the abstract interface.

## 22.3 Wider is not automatically faster

A wider vector may:

- process more independent items per operation;
- increase accumulator register demand;
- increase masked-lane waste for short tasks;
- require more costly lane rearrangements;
- change attainable core frequency or throughput on a given machine;
- expose different memory-alignment and cache-line behavior.

Measure the full architecture at different effective widths when feasible.

## 22.4 SIMD versus threads versus instruction-level parallelism

These are distinct axes:

- **SIMD:** several logical data lanes in one vector operation.
- **Instruction-level parallelism:** several independent operations in flight.
- **Task/thread parallelism:** separate workers execute independent work chunks.

A design can exploit all three:

~~~text
PARALLEL_FOR task_chunk IN TASK_CHUNKS
    LOCAL_STATE = INITIALIZE_WORKER_STATE()

    FOR output_tile IN OUTPUT_TILES(task_chunk)
        acc[INDEPENDENT_CHAINS] = VECTOR_IDENTITIES()

        FOR reduction_tile IN REDUCTION_TILES(output_tile)
            UPDATE_INDEPENDENT_VECTOR_CHAINS(
                acc, reduction_tile, LOCAL_STATE)

        STORE_FINALIZED(output_tile, MERGE_CHAINS(acc))
~~~

This does not imply the outer parallel task is always the right place. In some problems parallelizing reduction tiles is necessary because there are too few independent outputs.

## 22.5 Thread ownership and scratch storage

Disjoint logical outputs are the simplest way to avoid races. Allocate scratch per task or worker. When multiple tasks share a cache line, coherence traffic may reduce speed even without races; when they update the same output, a defined merge protocol is required.

## 22.6 Adapt architecture to workload regimes

A single dispatch policy may select among architecture families:

~~~text
IF NUMBER_OF_OUTPUTS is large AND REDUCTION_LENGTH is moderate
    architecture = OUTPUT_LANES
ELSE IF NUMBER_OF_OUTPUTS is small AND REDUCTION_LENGTH is large
    architecture = REDUCTION_LANES
ELSE IF TASK_LENGTHS_ARE_HIGHLY_VARIABLE
    architecture = BUCKET_OR_COMPACT
ELSE
    architecture = SIMPLE_BASELINE

EXECUTE_VALIDATED_ARCHITECTURE(architecture)
~~~

These are *illustrative heuristics*, not performance guarantees. The threshold choices and feature estimates must be grounded in measurements.

## 22.7 Portable cost-model vocabulary

Describe each architecture using:

- active logical lanes;
- number of independent accumulator chains;
- unique input/output bytes per tile;
- estimated cache-line requests;
- cross-lane reductions and permutations;
- sequential dependency depth;
- grouping/packing setup;
- thread ownership and scratch footprint.

These metrics can be compared conceptually across targets without relying on a specific machine model.

## 22.8 Portability is semantic first, performance second

An algorithm can be **functionally portable** but optimized differently for different targets. Preserve one semantic contract and several validated schedules. Avoid baking assumptions about preferred widths, subgroup boundaries, or instruction costs into the algorithm itself.

**Takeaway:** portable SIMD architecture separates **what is computed**, **how the work is scheduled**, and **how a target realizes each abstract vector operation**.
