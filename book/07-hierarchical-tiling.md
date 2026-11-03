# 7. Register, cache, and multi-level tiling

[Previous](06-stationary-architectures.md) · [Contents](README.md) · [Next](08-memory-layout.md)

## 7.1 Why one tile size is insufficient

A fast architecture typically coordinates several capacities: registers for active operands and partial results; nearby caches for reused input/output tiles; larger caches for task and working-set staging; memory for the complete data set.

A tile that fits the largest cache may still spill registers. A register-perfect microtile may repeatedly reload data that a larger cache tile could share.

~~~text
FOR task_tile IN TILES(TASKS, T_TASK)
    FOR cache_output_tile IN TILES(OUTPUTS, T_OC)
        FOR cache_reduction_tile IN TILES(REDUCTIONS, T_RC)
            STAGE_SHARED_INPUTS(cache_reduction_tile)
            FOR reg_output_tile IN TILES(cache_output_tile, T_OR)
                acc = LOAD_OR_INITIALIZE_PARTIAL(reg_output_tile)
                FOR reg_reduction_tile IN TILES(cache_reduction_tile, T_RR)
                    acc = VECTOR_FOLD(acc, reg_reduction_tile)
                SAVE_PARTIAL(reg_output_tile, acc)
        FINALIZE(cache_output_tile)
~~~

This hierarchy may incur partial stores for every cache reduction tile. A better schedule for small outputs could keep a register output tile alive across *all* reduction tiles; choose based on register and traffic constraints.

## 7.2 A register-budget estimate

Let **R_total** be the available architectural register capacity measured in *vector register equivalents*. Estimate:

~~~text
R_required =
    R_output_accumulators
  + R_input_operands
  + R_temporary_values
  + R_masks_and_indices
  + R_pipeline_overlap
  + R_safety_margin
~~~

Feasibility requires **R_required <= usable register budget**. This is approximate: compilers or implementations may require additional registers, and register allocation granularity is architecture dependent.

For output-lane mapping:

~~~text
accumulator_vectors =
    CEIL(T_OUTPUT / EFFECTIVE_LANES_PER_ACCUMULATOR)
    × INDEPENDENT_ACCUMULATOR_CHAINS
~~~

Increasing tile area can increase live vectors even if the total number of arithmetic operations is unchanged.

## 7.3 How to select a tile shape

Suppose a two-dimensional output tile is **T_A × T_B**. A wider tile may improve contiguous access along A, while a taller tile may increase reuse along B. Equal area does not imply equal access cost.

Evaluate the footprint:

~~~text
footprint(tile) =
    UNIQUE_INPUT_BYTES(tile + required_halo)
  + LIVE_PARTIAL_OUTPUT_BYTES(tile)
  + INDEX_AND_MASK_BYTES(tile)
  + SCRATCH_BYTES(tile)
~~~

Choose candidate shapes under cache and register budgets, then measure. Unique bytes and **actual transferred bytes** differ because of alignment, caching, read-for-ownership, and cache-line size.

## 7.4 Reduction chunk size from arithmetic bounds

If a narrow accumulator has maximum representable positive value **M**, each contribution lies in **[0, C]**, and initial value is zero, at most

\[
K_{\max} = \left\lfloor\frac{M}{C}\right\rfloor
\]

contributions can be added without overflow. Choose a chunk size no larger than this bound **provided the partial accumulation truly obeys those assumptions**.

~~~text
wide_total = 0
FOR chunk IN TILES(CONTRIBUTIONS, K_safe)
    narrow_acc = 0
    FOR r IN chunk
        narrow_acc += NONNEGATIVE_BOUNDED_CONTRIBUTION(r)
    wide_total += WIDEN(narrow_acc)
~~~

Signed contributions, saturating operators, nonzero initial states, and other semantics require different reasoning. Always flush the final incomplete chunk.

## 7.5 Trade-off: independent chains versus spills

~~~text
FOR output_tile
    acc[0..U-1] = IDENTITIES
    FOR reduction_tile
        FOR chain IN 0..U-1
            acc[chain] = COMBINE(
                acc[chain], CONTRIBUTION(chain, reduction_tile))
    COMMIT(MERGE(acc))
~~~

More chains can hide dependency latency; too many may spill. The optimal U depends on actual operations, compiler/code generator decisions, lane width, and available execution units.

## 7.6 Halo and boundary overhead

For a neighborhood of radius h around a tile with shape **T_A × T_B**, the required input region may have size **(T_A+2h) × (T_B+2h)**. The ratio of staged input area to useful output area is:

\[
\frac{(T_A+2h)(T_B+2h)}{T_A T_B}
\]

Small tiles waste more boundary overlap; huge tiles may no longer fit in fast storage. This is an example of selecting the **outermost patch** from data reuse, not from loop naming.

## 7.7 Tiling for parallel tasks

Tile ownership also determines scheduling overhead and imbalance. A large task tile amortizes setup but creates fewer parallel tasks. A small task tile balances better but increases task-management cost and often duplicates context.

Use a two-level strategy:

~~~text
PARALLEL_FOR task_chunk IN TASK_CHUNKS
    LOCAL_SCRATCH = ALLOCATE_ONCE(task_chunk)
    FOR vector_tile IN VECTOR_TILES(task_chunk)
        EXECUTE_WITH_LOCAL_SCRATCH(vector_tile, LOCAL_SCRATCH)
~~~

Do not share mutable scratch across simultaneous tasks unless explicitly synchronized.

## 7.8 Practical tile search

Try a bounded candidate set for **T_OUTPUT**, **T_REDUCTION**, **T_TASK**, and accumulator chain count. Reject candidates that violate arithmetic, bounds, or register constraints. Compare median runtime *and* variation across representative short, medium, and long reduction lengths.

**Takeaway:** every tile size expresses a compromise between work reuse, storage capacity, and scheduling overhead.
