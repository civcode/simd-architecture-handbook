# 11. Irregular indexing, conflicts, and sparse work

[Previous](10-control-flow.md) · [Contents](README.md) · [Next](12-maps-reductions.md)

Irregular algorithms are not intrinsically non-vectorizable. Their limits are usually dependency uncertainty, address-generation latency, write conflicts, and low lane occupancy.

## 11.1 Reads and writes are different problems

An indirect read can be independent even when addresses are scattered:

~~~text
FOR item
    result[item] = EVALUATE(data[read_index[item]])
~~~

An indirect update may have conflicts:

~~~text
FOR item
    key = write_index[item]
    output[key] = COMBINE(output[key], VALUE(item))
~~~

Two items can map to the same key. Treating them as independent vector lanes is not correct without a conflict policy.

## 11.2 Vectorizing independent gathers

~~~text
FOR tile IN TILES(items, V)
    indices = LOAD(read_index, tile)
    valid = VALIDATE(indices)
    values = SAFE_GATHER(data, indices, valid)
    results = VECTOR_EVALUATE(values)
    STORE(output, tile, results)
~~~

Useful when index lookups are the bottleneck? Not necessarily: a gather can be limited by several independent cache misses. Creating multiple in-flight groups can sometimes hide latency.

## 11.3 Conflict resolution architectures

**Sequential within tile:** vectorize address/value preparation, then apply updates in a defined order.

~~~text
FOR tile
    indices, values = VECTOR_PREPARE(tile)
    FOR lane IN ORIGINAL_ORDER(tile)
        UPDATE_ONE(output, indices[lane], values[lane])
~~~

**Privatized partials:** aggregate updates into a private table and merge with the required semantics.

~~~text
FOR tile
    private = EMPTY_MAP()
    FOR item IN tile
        AGGREGATE_BY_KEY(private, INDEX(item), VALUE(item))
    MERGE_KEYS(output, private)
~~~

**Group-by-destination:** sort or bucket items by target index, producing conflict-free segments.

~~~text
groups = GROUP_ITEMS_BY_TARGET(items)
FOR group IN groups
    combined = REDUCE_VALUES(group)
    APPLY_ONE_UPDATE(output, TARGET(group), combined)
~~~

The latter two require a legal associative grouping rule or explicitly ordered merge. Atomics may also be appropriate for parallel execution, but their availability and semantics are not presumed here.

## 11.4 Sparse work representation

Sparse computations store only active relationships. A common representation has a list of destination segments, each with a variable number of contributions:

~~~text
FOR destination IN DESTINATIONS
    acc = IDENTITY
    FOR item IN SEGMENT(destination)
        acc = COMBINE(acc, EVALUATE(item))
    result[destination] = FINALIZE(acc)
~~~

The common SIMD choices are **many destinations per vector** (good for similarly sized segments), **many contributions per vector** (good for long segments), and **flattened nonzero list** (good for balancing work across segments).

## 11.5 Work scheduling from metadata

Precompute cheap metadata:

~~~text
FOR destination
    estimate[destination] = SEGMENT_LENGTH(destination)

small, medium, large = PARTITION_BY_ESTIMATED_WORK(estimate)

VECTORIZE_MANY_DESTINATIONS(small)
VECTORIZE_ONE_OR_FEW_DESTINATIONS(medium)
PARALLELIZE_AND_REDUCE_LONG_SEGMENTS(large)
~~~

Thresholds are tuning parameters, not algorithmic truths. Reordering destinations is legal only when result order can be restored and there are no cross-destination dependencies.

## 11.6 Address locality versus load balance

Sorting by address improves locality but can unbalance work. Sorting by length improves occupancy but can scatter memory access. A useful search varies both **grouping key** and **tile size**, not only lane width.

## 11.7 Safety and determinism

Verify:

- Every valid element contributes exactly once.
- Empty segments produce the specified identity/finalization behavior.
- Index bounds are checked before any memory access.
- Collisions resolve in the required order or under a proven reduction law.
- Scratch tables are initialized and thread-private where necessary.
- Reordered outputs return to their original logical positions.

**Takeaway:** irregular SIMD design is often a **data organization problem** solved before the inner arithmetic begins.
