# 5. The algebra of loop transformations

[Previous](04-loop-depths.md) · [Contents](README.md) · [Next](06-stationary-architectures.md)

Transformations create schedules with different communication and parallelism. Every one must be checked against the dependence graph and numerical contract.

## 5.1 Interchange

Swapping axes changes the frequency with which data is loaded and state is retained.

~~~text
# Original
FOR i
    FOR j
        PROCESS(i, j)

# Interchanged
FOR j
    FOR i
        PROCESS(i, j)
~~~

Legal when iterations do not require the original relative order. Interchange can turn strided accesses contiguous, but can simultaneously turn output accumulators into high-traffic memory variables.

## 5.2 Strip mining and vector grouping

Convert one axis into tiles plus local positions:

~~~text
FOR i IN 0..N-1
    PROCESS(i)

# Strip mined
FOR base IN 0..N-1 STEP T
    FOR offset IN 0..T-1
        IF base + offset < N
            PROCESS(base + offset)
~~~

The local axis may map onto lanes, unrolled scalar work, or a smaller tile. Strip mining preserves iteration order if the inner offset is traversed in order.

## 5.3 Multi-dimensional tiling

~~~text
FOR i
    FOR j
        PROCESS(i,j)

# Blocked
FOR ib IN TILES(I, T_I)
    FOR jb IN TILES(J, T_J)
        FOR i IN ib
            FOR j IN jb
                PROCESS(i,j)
~~~

Blocking can change cross-tile order. For dependent computations, pick a tile traversal that respects edges. Tile shapes should reflect reuse, not symmetry for its own sake.

## 5.4 Fusion versus fission

Fusion combines successive passes to keep intermediate values local.

~~~text
# Two full passes
FOR i
    temporary[i] = STAGE_A(input[i])
FOR i
    output[i] = STAGE_B(temporary[i])

# Fused
FOR i
    tmp = STAGE_A(input[i])
    output[i] = STAGE_B(tmp)
~~~

Fusion eliminates materialized intermediate traffic but may increase register lifetime and inhibit vectorization if the stages prefer different lane maps. Fission can improve scheduling, instruction-cache behavior, or permit a prepacked layout between stages.

## 5.5 Collapse and flattening

~~~text
FOR i IN I
    FOR j IN J
        PROCESS(i,j)

FOR k IN 0..(SIZE(I)*SIZE(J))-1
    i,j = DECODE(k)
    PROCESS(i,j)
~~~

Useful when the original inner axis is shorter than a vector, or when tasks must be distributed evenly. Beware index-decoding overhead and loss of predictable data strides.

## 5.6 Unrolling and unroll-and-jam

Unrolling duplicates independent iterations or reduction operations; it can expose parallel instruction chains and reduce control overhead.

~~~text
FOR o IN OUTPUTS STEP U
    a[0..U-1] = IDENTITY
    FOR r IN REDUCTIONS
        FOR u IN 0..U-1
            a[u] = COMBINE(a[u], EVALUATE(o+u, r))
    COMMIT(a)
~~~

Here the inner output iterations are **jammed** together under a shared reduction traversal. The win is reused inputs and independent accumulators; the losses are register pressure and code size. A larger U can run slower.

## 5.7 Loop skewing and wavefront scheduling

If an iteration at (i,j) depends on (i-1,j) and (i,j-1), processing a row in parallel violates an edge. Let **wavefront = i + j**. All direct predecessors lie on the previous wavefront, so equal-wavefront points are independent *provided no additional dependencies exist*.

~~~text
FOR w IN LEGAL_WAVEFRONT_RANGE
    ready = POINTS_WITH_SUM_OF_COORDINATES(w)
    FOR tile IN TILES(ready, V)
        predecessors = READ_PRIOR_WAVEFRONTS(tile)
        WRITE_RESULTS(tile, VECTOR_EVALUATE(predecessors))
~~~

The transformation creates parallelism at the cost of noncontiguous access and small wavefronts near boundaries. Tile the wavefronts when the state space is large.

## 5.8 Loop reversal and distribution

Reversing traversal can turn a descending memory access into ascending; it is legal only if no directed dependency forbids it. Distributing one loop with multiple statements into distinct loops may expose vectorization but add intermediate traffic.

~~~text
FOR i
    A(i)
    B(i)

# Only if dependencies permit:
FOR i
    A(i)
FOR i
    B(i)
~~~

## 5.9 Reduction splitting and tree re-association

~~~text
# Reference: serial fold
a = IDENTITY
FOR r
    a = COMBINE(a, EVALUATE(r))

# Transformed: partial folds and merge
partials = ARRAY_OF_IDENTITIES(K)
FOR r
    lane = ASSIGN_PARTIAL(r, K)
    partials[lane] = COMBINE(partials[lane], EVALUATE(r))
a = MERGE_IN_DEFINED_ORDER(partials)
~~~

Exact equivalence requires compatible algebra or a specified approximate contract. Independent chains may shorten critical paths even without changing vector width.

## 5.10 A transformation ledger

Record for each proposed schedule:

| Transformation | Why attempted | Legality proof | Resource change |
| --- | --- | --- | --- |
| Interchange | Change inner stride/reuse | Dependence direction | Accumulator and input lifetime |
| Tile | Improve locality | Inter-tile edges | Extra loops, tile storage |
| Fuse | Eliminate temporary | Producer/consumer alias analysis | More live values |
| Fission | Separate bottlenecks | Dependence between statements | More memory passes |
| Unroll-and-jam | More independent work | Independent output updates | Register demand/code size |
| Skew | Expose parallel front | Dependency vector constraints | New address/order complexity |
| Flatten | Fill lanes across axes | Valid index mapping | Index decoding and scatter |

**Takeaway:** a transformation is an exchange between computation, storage, and scheduling; none is unconditionally beneficial.
