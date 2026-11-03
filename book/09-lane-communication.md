# 9. Lane communication, reductions, and rearrangement

[Previous](08-memory-layout.md) · [Contents](README.md) · [Next](10-control-flow.md)

Values already in vector registers are not necessarily free to move. Some operations act independently on each lane; others move information between lanes and may create additional latency, throughput demand, or cross-group restrictions.

## 9.1 Classify operations by communication

| Pattern | Data movement | Typical architectural consequence |
| --- | --- | --- |
| Lane-wise | Lane i uses lane i | Excellent for output-lane mapping |
| Broadcast | One value reaches many lanes | Cost amortized across a tile |
| Neighbor shift | Lane i reads lane i±k | Useful for scans and neighborhoods |
| Permutation | Arbitrary lane reorder | May be expensive or decomposed |
| Horizontal reduction | Many lanes contribute to fewer results | Requires reduction tree or equivalent |
| Transposition | Lanes exchange multiple fields | Useful when layouts differ |
| Compaction | Keep only active lanes | Improves utilization at bookkeeping cost |

Instruction availability and cost vary. The abstract model should assign **communication cost** rather than presume every permutation is one cheap operation.

## 9.2 Why output lanes avoid horizontal reduction

~~~text
# One independent accumulator per output lane
acc = ZERO_VECTOR
FOR r
    acc = COMBINE(acc, EVALUATE_FOR_OUTPUT_LANES(r))
STORE(acc)
~~~

Each output result stays in one lane. No cross-lane transfer is necessary until optional packing or storage.

In a reduction-lane architecture, many lanes jointly calculate one result:

~~~text
partials = ZERO_VECTOR
FOR rtile
    partials = COMBINE(partials, EVALUATE_FOR_REDUCTION_LANES(rtile))
result = HORIZONTAL_REDUCE(partials)
~~~

This may be a win if it dramatically improves contiguous input access. Its reduction tree can alter floating-point rounding order.

## 9.3 Reduction tree cost

With V partials, a balanced associative tree takes approximately **ceil(log2 V)** combination stages, though actual instruction count depends on available rearrangements.

~~~text
values = VECTOR_PARTIALS
WHILE COUNT(values) > 1
    values = PAIRWISE_COMBINE(values)
result = values[0]
~~~

The tree is a logical illustration. Some implementations use dedicated reduction support; others use multiple moves and combinations.

## 9.4 Partial reductions and multiple independent chains

Instead of horizontally reducing every vector group:

~~~text
partial[lanes] = IDENTITY
FOR tile IN REDUCTION_TILES
    partial = VECTOR_COMBINE(partial, EVALUATE(tile))
result = HORIZONTAL_REDUCE(partial)
~~~

This reduces horizontal-operation frequency. A larger number of independent partial vectors can help latency hiding but increases register pressure.

## 9.5 Communication in scans

A prefix operation needs information from earlier lanes:

~~~text
prefix = values
distance = 1
WHILE distance < V
    shifted = SHIFT_RIGHT_WITH_IDENTITY(prefix, distance)
    prefix = MASKED_COMBINE(prefix, shifted, lanes >= distance)
    distance = distance * 2
~~~

This form is legal for associative COMBINE with a valid identity; it performs a logarithmic number of cross-lane stages. When COMBINE is not associative, the reference sequential order may be unavoidable.

## 9.6 Transposition between lane maps

Sometimes one stage wants lanes as **problems**, while the next wants lanes as **fields** or **outputs**.

~~~text
tile = LOAD_RECORD_ORIENTED_GROUP()
transposed = REORGANIZE_LANES(tile, target_layout)
COMPUTE_BY_FIELD(transposed)
STORE_OR_REPACK(transposed)
~~~

This can pay off when many operations reuse the transposed representation. When only one cheap operation is performed, the transpose overhead may dominate.

## 9.7 Lane width and subgroups

Logical vector width V is not always one fully connected communication domain. A target may have subgroups with cheaper within-group exchange than cross-group exchange.

Design two reduction levels if beneficial:

~~~text
FOR subgroup
    local_result[subgroup] = REDUCE_WITHIN_SUBGROUP(values[subgroup])
result = MERGE_SUBGROUP_RESULTS(local_result)
~~~

The algorithm remains hardware-agnostic, while the schedule exposes a parameter for subgroup size.

## 9.8 Choosing among competing maps

A useful thought experiment: if mapping A performs **twice as many lane-wise calculations** but no cross-lane movement, while B halves arithmetic but needs many permutations and a horizontal fold, the cheaper architecture is not obvious. Estimate useful operations, communication, registers, and data movement together.

**Takeaway:** keep lanes independent whenever it does not sacrifice substantially more valuable input or output reuse.
