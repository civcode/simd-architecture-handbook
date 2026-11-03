# 14. Neighborhoods, wavefronts, and space-time schedules

[Previous](13-scans-recurrences.md) · [Contents](README.md) · [Next](15-search-selection.md)

Neighborhood computations use nearby input elements to produce outputs. Many outputs overlap in their required inputs, which makes them ideal for **spatial tiling**. Ordered neighborhoods also introduce **wavefront** scheduling.

## 14.1 Read-only neighborhoods

Let output at coordinate x depend on inputs at offsets from an abstract set H:

~~~text
FOR x IN OUTPUT_COORDINATES
    acc = IDENTITY
    FOR offset IN H
        acc = COMBINE(acc, EVALUATE(input[x + offset]))
    output[x] = FINALIZE(acc)
~~~

If all reads come from an immutable input snapshot and outputs are distinct, x iterations are independent. Two natural lane mappings are **adjacent x positions** (reuse overlapping input windows) and **offsets in H** (one output per group, followed by a reduction).

## 14.2 Spatial output tiling

~~~text
FOR output_patch IN PARTITION_OUTPUT_DOMAIN(T_X, T_Y)
    required_inputs = COMPUTE_HALO(output_patch, neighborhood)
    staged = STAGE_INPUT_PATCH(required_inputs)

    FOR vector_output_tile IN TILES(output_patch, V)
        acc = VECTOR_IDENTITY(vector_output_tile)
        FOR offset IN neighborhood
            acc = VECTOR_COMBINE(
                acc, VECTOR_EVALUATE(staged,
                                     vector_output_tile, offset))
        STORE_FINALIZED(vector_output_tile, acc)
~~~

One staged patch may be reused by many outputs. But staged data costs storage and boundary duplication. Favor a patch large enough to amortize halo traffic, small enough to fit available cache, and shaped to make inner addresses predictable.

## 14.3 Halo overhead and tile shape

For radius h and rectangular output tile A × B, the staged input region may be (A+2h) × (B+2h), giving overhead ratio:

\[
\text{halo ratio} = \frac{(A+2h)(B+2h)}{AB}
\]

This favors larger tiles, but larger tiles can spill registers, exceed cache, or reduce parallel-task count. On a row-major layout, increasing A and B changes memory behavior differently despite equal area.

## 14.4 Sliding reuse within a vector

Adjacent outputs often require shifted views of nearby input values:

~~~text
input_strip = LOAD_NEIGHBORHOOD_STRIP(output_tile)

FOR offset IN H
    shifted = SHIFT_OR_REINDEX(input_strip, offset)
    acc = COMBINE(acc, VECTOR_EVALUATE(shifted))
~~~

This may replace repeated memory loads with lane rearrangements. Whether it helps depends on the relative cost of register communication versus cached loads. Cross-vector boundaries complicate sliding windows.

## 14.5 Ordered neighborhood dependencies

A different pattern reads **previously computed outputs**:

~~~text
FOR i IN ROW_ORDER
    FOR j IN COLUMN_ORDER
        output[i,j] = F(output[i-1,j], output[i,j-1], input[i,j])
~~~

The output at (i,j) has incoming edges from above and left. The row cannot be vectorized across j without violating the left dependence.

However, define wavefront **w = i+j**. Every direct predecessor belongs to wavefront w−1, so points on the same wavefront can execute concurrently when there are no further conflicting edges:

~~~text
FOR w IN INCREASING_WAVEFRONT_ORDER
    ready = ALL_VALID_POINTS_WITH_SUM(w)
    FOR tile IN TILES(ready, V)
        predecessor_a = READ_ABOVE(tile)
        predecessor_b = READ_LEFT(tile)
        WRITE(tile, VECTOR_F(predecessor_a, predecessor_b, input[tile]))
~~~

This trades a regular inner row traversal for diagonal access and limited parallelism at the first/last wavefronts.

## 14.6 Tile wavefronts

Scheduling one point at a time across diagonals can be too granular. Partition the iteration space into tiles, then order tiles by their dependence edges:

~~~text
tiles = PARTITION_DEPENDENT_DOMAIN(T_X, T_Y)
dependency_graph = EDGES_BETWEEN_TILES(tiles)

WHILE UNFINISHED_TILES_EXIST
    ready = TILES_WITH_COMPLETED_PREDECESSORS(dependency_graph)
    EXECUTE_READY_TILES_IN_PARALLEL(ready)
    MARK_COMPLETED_AND_RELEASE_SUCCESSORS(ready)
~~~

Within each tile, use a legal serial, diagonal, or independently vectorized schedule. This separates cross-tile concurrency from within-tile SIMD.

## 14.7 Space-time tiling

For iterative computations that update a spatial domain over time, retaining a patch through several time steps can reduce memory traffic. But the required halo grows as information propagates between steps.

~~~text
FOR space_time_tile IN LEGAL_TILED_SCHEDULE
    STAGE_REQUIRED_INITIAL_STATE_AND_HALO(space_time_tile)
    FOR local_step IN CAUSAL_ORDER(space_time_tile)
        UPDATE_ONLY_POINTS_WITH_READY_PREDECESSORS(local_step)
    COMMIT_BOUNDARY_AND_FINAL_STATE(space_time_tile)
~~~

A space-time tile is valid only if it contains, or receives, all predecessors. It can reduce repeated loads but increase redundant halo work and scratch footprint.

## 14.8 Choosing the outermost patch

Compare:

- **Output patch first:** reuse local output accumulators and neighboring inputs.
- **Input patch first:** reuse a staged source region across output groups.
- **Time tile first:** reuse evolving local state across steps.
- **Task patch first:** avoid cross-task writes and schedule parallel work.

These are distinct architectures even if the innermost arithmetic is identical.

**Takeaway:** the correct outermost tile is a **region of reusable data and legal dependencies**, not necessarily an original loop axis.
