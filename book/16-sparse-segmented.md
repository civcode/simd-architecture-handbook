# 16. Segmented, sparse, and indirect algorithms

[Previous](15-search-selection.md) · [Contents](README.md) · [Next](17-stateful-work.md)

Sparse and segmented algorithms combine variable-length work, indirect input addresses, and potentially colliding updates. Their best architecture is usually chosen from **metadata about work shape** before vector arithmetic begins.

## 16.1 Three general representations

1. **Destination-centric:** each output owns a segment of contributions.
2. **Contribution-centric:** a flat list of (destination, input) relationships.
3. **Bucketed or blocked:** relationships reorganized by length, destination, or input locality.

These representations expose different loop nests even when they encode the same mathematical relation.

## 16.2 Destination-centric: reduce each segment

~~~text
FOR destination IN DESTINATIONS
    acc = IDENTITY
    FOR edge IN SEGMENT(destination)
        acc = COMBINE(acc, EVALUATE(edge))
    result[destination] = FINALIZE(acc)
~~~

This is naturally output-stationary. Map several destinations to lanes when segments are short and similar; map contributions to lanes and horizontally reduce for very long segments.

## 16.3 Contribution-centric: traverse every relationship

~~~text
partials[DESTINATIONS] = IDENTITIES

FOR edge IN RELATIONSHIPS
    destination = TARGET(edge)
    partials[destination] = COMBINE(
        partials[destination], EVALUATE(edge))
~~~

This makes input traversal contiguous if the relationships are stored in a flat list, but creates indirect writes and possible conflicts.

A possible transformation groups edges by target:

~~~text
grouped = STABLE_GROUP_BY_TARGET(RELATIONSHIPS)

FOR target IN DESTINATIONS
    partial = IDENTITY
    FOR edge IN grouped[target] IN ORIGINAL_ORDER
        partial = COMBINE(partial, EVALUATE(edge))
    result[target] = FINALIZE(partial)
~~~

If COMBINE is nonassociative, preserve the contribution order within each target. Grouping has preprocessing and memory costs.

## 16.4 Segmented vector reduction

A flattened relationship list can carry flags marking the starts of segments:

~~~text
values = EVALUATE_RELATIONSHIP_TILES()
segment_start = LOAD_SEGMENT_START_FLAGS()

partial_segment_sums =
    SEGMENTED_SCAN(values, segment_start, COMBINE)

WRITE_FINAL_VALUE_AT_SEGMENT_ENDS(partial_segment_sums)

# Empty segments have no relationship and therefore no segment-end value.
FOR destination IN EMPTY_SEGMENTS
    result[destination] = FINALIZE(IDENTITY)
~~~

Segmented scans require carefully defined handling for segments crossing vector and block boundaries. Empty segments must still receive their contractually required result. They may incur many lane operations when segments are tiny.

## 16.5 Three-way dispatch by work length

A useful high-level schedule:

~~~text
metadata = ANALYZE_SEGMENT_LENGTHS(segments)

small  = SELECT(metadata, length <= SMALL_LIMIT)
medium = SELECT(metadata, SMALL_LIMIT < length <= LARGE_LIMIT)
large  = SELECT(metadata, length > LARGE_LIMIT)

PROCESS_MANY_SEGMENTS_PER_VECTOR(small)
PROCESS_FEW_SEGMENTS_WITH_BLOCKED_REDUCTIONS(medium)
PROCESS_LONG_SEGMENTS_WITH_PARALLEL_PARTIALS(large)
~~~

Thresholds should be tuned using representative workload distributions, and output association must be preserved.

## 16.6 Input-centric grouping

If many relationships read the same source, grouping by source rather than destination can reduce gather traffic:

~~~text
FOR source_group IN GROUP_RELATIONSHIPS_BY_SOURCE
    shared_value = LOAD_SOURCE(source_group.source)
    FOR relationship_tile IN source_group.relationships
        contributions = VECTOR_EVALUATE(shared_value, relationship_tile)
        RESOLVE_AND_APPLY_OUTPUT_UPDATES(contributions)
~~~

This is input-stationary, but output collisions may now dominate. The transformation is useful only if saved source traffic exceeds extra output-update cost and grouping overhead.

## 16.7 Indirection chains

A lookup may depend on the result of the previous lookup:

~~~text
position = START
WHILE NOT DONE(position)
    position = NEXT(position)
~~~

A single chain has a true dependency, so a vector spanning successive chain steps is generally not useful. When many independent chains exist, assign a chain to each lane:

~~~text
states = LOAD_STARTS(chain_group)
valid = VALID_LANES(chain_group)
active = valid AND NOT MASKED_DONE(states, valid)

WHILE ANY(active)
    next_states = SAFE_VECTOR_FOLLOW(states, active)
    states = SELECT(active, next_states, states)
    active = UPDATE_COMPLETION_MASK(states, active)
~~~

Long-tail chains can leave lanes idle; compaction or work queues can reassign completed lanes.

## 16.8 Operational cautions

- Indirect addresses must be validated before loads.
- Concurrent updates require explicit conflict resolution.
- Grouping may reorder operations and change results when order matters.
- Metadata preprocessing should be included in performance measurements.
- Sparse workloads can be limited by address latency rather than vector throughput.

**Takeaway:** for sparse SIMD, optimize the **shape and order of relationships** as aggressively as the computations on them.
