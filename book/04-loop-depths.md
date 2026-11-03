# 4. Loop architectures from zero to N levels

[Previous](03-lane-mapping.md) · [Contents](README.md) · [Next](05-loop-transformations.md)

## 4.1 Count logical axes, not syntactic loops

A **zero-loop** computation is a single logical operation; it can still use a vector internally. A one-axis computation can gain tile loops. A many-axis computation can be flattened into a vector index. Thus neither source loop count nor loop nesting depth predicts SIMD suitability.

| Logical dimensions | Natural starting point | Main architecture decision |
| --- | --- | --- |
| 0 | One vector operation | Is packing/setup worthwhile? |
| 1 | Vectorized independent items | Tail, alignment, throughput, latency |
| 2 | Outputs × reductions, or two independent axes | Which axis occupies lanes? |
| 3 | Problems × outputs × reductions | Outer owner, lane mapping, reuse |
| 4–N | Repeated independent, output, reduction, state axes | Scheduling and hierarchical tiles |
| Ragged/dynamic | Variable-length nested work | Grouping, masking, queues, segmented processing |

## 4.2 Zero and one logical loop

A short fixed-width transformation may have no runtime loop:

~~~text
values = LOAD_FIXED_GROUP(inputs)
outputs = VECTOR_TRANSFORM(values)
STORE_FIXED_GROUP(outputs)
~~~

Using a vector may not help if packing is expensive or the operation occurs once.

A single logical loop commonly becomes **three execution levels**: outer tile, vector group, and tail.

~~~text
FOR cache_tile IN TILES(0..N, T_C)
    FOR vector_tile IN TILES(cache_tile, V)
        mask = VALID_LANES(vector_tile)
        x = MASKED_LOAD(input, vector_tile, mask)
        y = VECTOR_EVALUATE(x)
        MASKED_STORE(output, vector_tile, y, mask)
~~~

The outer tile is useful only when it enables reuse, parallel work ownership, or better scheduling; otherwise it may be redundant overhead.

## 4.3 Two-loop choices

For independent outputs and an inner reduction:

~~~text
# A: output-stationary, output lanes
FOR otile
    acc = VECTOR_IDENTITY(otile)
    FOR r
        acc = VECTOR_COMBINE(acc, EVALUATE_ALL(otile, r))
    WRITE(otile, acc)

# B: reduction-stationary, reduction lanes
FOR o
    partial = VECTOR_IDENTITY(reduction_lanes)
    FOR rtile
        partial = VECTOR_COMBINE(partial, EVALUATE_ALL(o, rtile))
    WRITE(o, HORIZONTAL_REDUCE(partial))
~~~

Other two-loop structures can represent two **independent** axes, in which case flattening or two-dimensional tiling may be preferable to treating either as a reduction.

## 4.4 Three logical axes and six orders

With problems **P**, outputs **O**, and reductions **R**, all six syntactic orders can be considered, but **only legal orders are candidates**:

| Order | Likely resource focus | Main challenge |
| --- | --- | --- |
| P–O–R | One output per task | Repeated shared-input loads |
| P–R–O | Shared input per contribution | Many partial outputs live or updated in memory |
| O–P–R | Same output across tasks | Interleaved problem data |
| O–R–P | Same output and reduction, many tasks | Irregular task addresses |
| R–P–O | Reuse one contribution globally | Broad output state traffic |
| R–O–P | Reuse one contribution across tasks/outputs | Data layout and concurrent updates |

For example, moving R outside O may require storing a live accumulator for *every* O, or an explicit output tile. The permutation alone hides this cost.

~~~text
FOR p
    FOR rtile
        FOR otile
            partial[p, otile] +=
                CONTRIBUTIONS(p, otile, rtile)
~~~

This schedule can be attractive with strong input reuse and poor when partial-output traffic dominates.

## 4.5 Four or more axes: classify before permuting

Suppose the logical space has independent task axes **P0, P1**, output axes **O0, O1**, and reductions **R0, R1**.

There are many permutations, but a more productive method is:

1. Mark axes that commute legally.
2. Group related axes by the input and output storage they touch.
3. Assign one or two axes to vector lanes.
4. Assign inner tiles to register/local storage.
5. Assign outer tiles to cache/task ownership.
6. Schedule ordered axes with explicit dependency constraints.

~~~text
FOR task_tile IN TILES(P0 × P1, T_P)
    FOR output_tile IN TILES(O0 × O1, T_O)
        local = VECTOR_IDENTITY(output_tile)
        FOR reduction_tile IN TILES(R0 × R1, T_R)
            local = FOLD_TILE(local,
                              task_tile, output_tile, reduction_tile)
        COMMIT_FINAL(output_tile, local)
~~~

The source may have six nested loops, whereas the execution has three top-level tile loops plus vectorized and unrolled inner work. Tiles can be *rectangles, strips, diagonals, or ragged partitions*, depending on dependencies.

## 4.6 When a flattened index is better

If multiple small independent axes leave many lanes empty, flatten their product:

~~~text
items = ENUMERATE_VALID_COMBINATIONS(P0, P1, O0)

FOR vector_tile IN TILES(items, V)
    ids = DECODE_COORDINATES(vector_tile)
    VECTOR_EXECUTE(ids)
~~~

The mapping costs division/modulo, decoding, irregular stores, or a prebuilt index table. Prefer a simple affine decoding rule or precomputed compact work list when the original dimensions are tiny.

## 4.7 Variable-depth and ragged iteration

Many computations contain a nested loop whose bound depends on outer indices.

~~~text
FOR p
    FOR o IN OUTPUTS(p)
        FOR r IN CONTRIBUTIONS(p,o)
            UPDATE(p,o,r)
~~~

Three alternatives:

- **Padding:** process a common maximum bound with masks.
- **Bucketing:** group work with similar bounds, reducing inactive lanes.
- **Worklist:** compact valid triples into a flat list, then vectorize the list.

Worklists trade preprocessing and storage for a more regular inner kernel; bucketing trades order and grouping overhead for occupancy.

## 4.8 A decision procedure for arbitrary depth

~~~text
axes = CLASSIFY_LOGICAL_DIMENSIONS(specification)
dependences = BUILD_DEPENDENCE_GRAPH(axes)
candidate_groups = LEGAL_GROUPINGS(axes, dependences)

FOR grouping IN candidate_groups
    FOR lane_map IN FEASIBLE_LANE_MAPS(grouping)
        FOR tile_shape IN RESOURCE_FEASIBLE_TILES(grouping)
            schedule = CONSTRUCT_SCHEDULE(grouping, lane_map, tile_shape)
            IF PROVES_EQUIVALENT(schedule)
                ESTIMATE_COST(schedule)
                KEEP_FOR_MEASUREMENT(schedule)
~~~

Do **not** enumerate factorial permutations blindly. Eliminate illegal and obviously resource-infeasible schedules early.

**Takeaway:** deep SIMD architecture is a problem of dimension classification and scheduling, not simply choosing an outermost FOR loop.
