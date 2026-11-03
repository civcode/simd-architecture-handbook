# 3. Mapping work to vector lanes

[Previous](02-dependencies.md) · [Contents](README.md) · [Next](04-loop-depths.md)

## 3.1 The fundamental lane choices

A logical vector of width **V** can represent:

| Mapping | Lane i represents | Typical benefit | Typical cost |
| --- | --- | --- | --- |
| Output lanes | A distinct output o | Independent accumulator lanes, no horizontal reduction | Many live outputs |
| Reduction lanes | A distinct contribution r | Contiguous input traversal | Horizontal reduction and altered order |
| Problem lanes | A distinct independent task p | Batched reuse of control and vector utilization | Divergence, irregular interleaved inputs |
| Candidate lanes | A different alternative for the same output | Concurrent comparisons or hypotheses | Cross-lane selection |
| Mixed map | Flattened pair such as (p,o) | Flexibility and high occupancy | Index conversion and layout complexity |

The logical choice is independent of any named instruction set.

## 3.2 Output mapping

~~~text
FOR p
    FOR otile IN TILES(OUTPUTS(p), V)
        acc = VECTOR_IDENTITY(otile)
        FOR r
            values = VECTOR_EVALUATE(p, otile, r)
            acc = VECTOR_COMBINE(acc, values)
        STORE(result[p, otile], VECTOR_FINALIZE(acc))
~~~

Each lane's accumulator is independent. The cost is carrying **V** partial results throughout the reduction loop; an implementation may need multiple vectors to expose instruction-level parallelism.

## 3.3 Reduction mapping

~~~text
FOR p
    FOR o
        partial = IDENTITY
        FOR rtile IN TILES(CONTRIBUTIONS(p,o), V)
            values = VECTOR_EVALUATE(p, o, rtile)
            partial = COMBINE(partial,
                              HORIZONTAL_REDUCE(values, COMBINE))
        result[p,o] = FINALIZE(partial)
~~~

Alternatively accumulate a lane-local vector across multiple reduction tiles and reduce it only once. This can be faster, but changes grouping and possibly floating-point results.

## 3.4 Problem mapping

~~~text
FOR ptile IN TILES(PROBLEMS, V)
    active = VALID_LANES(ptile)
    output_state = VECTOR_IDENTITY(ptile)
    FOR step IN SHARED_STEPS(ptile)
        active_step = active AND STEP_VALID(ptile, step)
        values = VECTOR_EVALUATE(ptile, step, active_step)
        output_state = MASKED_COMBINE(
            output_state, values, active_step)
    STORE_FINALIZED(ptile, output_state, active)
~~~

When tasks have different lengths, some lanes become idle. Sorting/grouping by expected work may improve occupancy, but reordering must preserve any observable task ordering.

## 3.5 Selecting lane meaning

Estimate for each mapping:

1. Contiguous bytes read and written per vector operation.
2. Number of useful lanes across the actual workload distribution.
3. Number of live accumulator and temporary vectors.
4. Cross-lane reductions, shuffles, and conversions.
5. Setup cost paid per tile.
6. Branch divergence and data-dependent memory access.
7. Correctness implications of changing reduction order.

Compare **cost per useful result**, not merely operations per vector.

## 3.6 Mixed mapping and dimensional flattening

An output tile can contain pairs **(p, o)** rather than only consecutive o. If tasks have fewer outputs than available lanes, packing output pairs from several tasks may improve utilization:

~~~text
work_items = FLATTEN_VALID_PAIRS(PROBLEMS, OUTPUTS)

FOR tile IN TILES(work_items, V)
    (p, o) = UNPACK_PAIRS(tile)
    acc = VECTOR_IDENTITY(tile)
    FOR r IN COMMON_REDUCTION_DOMAIN(tile)
        mask = CONTRIBUTION_EXISTS(p, o, r)
        acc = MASKED_VECTOR_COMBINE(
            acc, VECTOR_EVALUATE(p,o,r,mask), mask)
    SCATTER_FINALIZED_BY_PAIR(result, tile, acc)
~~~

The flattened representation may require gathers/scatters, and a common reduction domain can waste masked lanes.

**Takeaway:** define lane semantics before deciding loop order or vector width.
