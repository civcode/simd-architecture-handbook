# 15. Search, selection, ordering, and comparison networks

[Previous](14-neighborhoods-wavefronts.md) · [Contents](README.md) · [Next](16-sparse-segmented.md)

Search and selection often contain branches and data-dependent results. They can still be reorganized by evaluating candidates in parallel and reducing the decision afterward.

## 15.1 Independent predicate evaluation

~~~text
FOR item IN ITEMS
    is_match[item] = PREDICATE(item)
~~~

A SIMD map evaluates the predicate for several items and stores a Boolean mask. Whether to materialize that mask depends on what follows.

## 15.2 Find the first satisfying item

Reference:

~~~text
answer = NONE
FOR i IN ASCENDING_ORDER
    IF PREDICATE(i)
        answer = i
        BREAK
~~~

A vectorized schedule evaluates blocks in order, identifies the first matching lane in each block, then stops at the first matching block:

~~~text
answer = NONE
FOR block IN ASCENDING_TILES(ITEMS, V)
    mask = VECTOR_PREDICATE(block)
    IF ANY(mask)
        answer = FIRST_ACTIVE_INDEX(block, mask)
        BREAK
~~~

This preserves the **first-match index** if PREDICATE is pure and speculative evaluation of other lanes in a block is safe. It can do more work than the scalar version when matches occur extremely early.

## 15.3 Best candidate among many alternatives

~~~text
best_value = WORST_POSSIBLE_VALUE
best_index = INVALID

FOR candidate IN ORIGINAL_ORDER
    value = SCORE(candidate)
    IF BETTER(value, best_value, candidate, best_index)
        best_value = value
        best_index = candidate
~~~

Evaluate candidate tiles in lanes and reduce (value,index) pairs:

~~~text
FOR tile IN CANDIDATE_TILES
    scores = VECTOR_SCORE(tile)
    tile_best = REDUCE_ARG_BEST(scores, candidate_ids, TIE_RULE)
    global_best = MERGE_ARG_BEST(global_best, tile_best, TIE_RULE)
~~~

A well-defined **total ordering** including tie rules is important for determinism. NaN treatment and signed-zero comparisons need explicit contracts when floating-point scores are involved.

## 15.4 Filter and compact

Reference:

~~~text
count = 0
FOR i
    IF KEEP(i)
        output[count] = input[i]
        count += 1
~~~

Vector architecture:

~~~text
count = 0

FOR tile IN TILES(input, V)
    mask = VECTOR_KEEP(tile)
    selected = COMPRESS_ACTIVE_VALUES(tile, mask)
    STORE_CONTIGUOUS(output, count, selected)
    count += NUMBER_OF_ACTIVE_LANES(mask)
~~~

This preserves stable input order if compaction preserves lane order and tiles are processed in original order. Some targets implement compaction directly; others require tables, prefix counts, or scalar scatter.

## 15.5 Comparison networks

A comparison network replaces data-dependent branches with a predetermined sequence of compare-and-exchange stages:

~~~text
FOR stage IN NETWORK_STAGES
    pairs = PREDEFINED_DISJOINT_PAIRS(stage)
    (lower, upper) = COMPARE_EXCHANGE(values, pairs, ORDER_RULE)
    values = ROUTE_TO_DESTINATIONS(lower, upper, stage)
~~~

Fixed networks can be effective for small sets where predictable control and vector lanes matter more than minimizing comparisons. Their costs include lane permutations, unused comparisons, and data-movement stages.

## 15.6 Partition-and-select versus full ordering

If the output only needs a minimum, maximum, top-k set, or threshold membership, fully sorting everything may be unnecessary. Prefer the weakest operation satisfying the contract:

| Requested output | Candidate organization |
| --- | --- |
| Single best | Vector local extrema + global reduction |
| First match | Ordered tiles + first active lane |
| All passing items | Predicate + stable compaction |
| Small top-k | Tile-local candidate structures + merge |
| Full ordering | Sorting/merging networks or partition strategy |

## 15.7 Divergence-aware search

For multiple independent searches with different lengths, map one search to each lane. For one search across many candidates, map candidates to lanes. The first wins when searches are plentiful and work lengths similar; the second when each search is expensive and associative selection is cheap.

**Takeaway:** the SIMD-friendly architecture often evaluates **more candidates at once** but postpones the scalar decision until a small result summary is available.
