# 12. Maps, independent outputs, and reductions

[Previous](11-irregular-work.md) · [Contents](README.md) · [Next](13-scans-recurrences.md)

**Map** and **reduce** are fundamental algorithm families, but they imply different best-case SIMD architectures. A map produces independent results; a reduction combines many contributions into fewer results.

## 12.1 Map: parallelism without cross-item communication

~~~text
FOR item IN ITEMS
    output[item] = EVALUATE(input[item], constants)
~~~

A natural schedule maps adjacent items to lanes:

~~~text
FOR tile IN TILES(ITEMS, V)
    valid = VALID_LANES(tile)
    input_values = SAFE_VECTOR_LOAD(input, tile, valid)
    output_values = VECTOR_EVALUATE(input_values, constants)
    SAFE_VECTOR_STORE(output, tile, output_values, valid)
~~~

The principal design choices are memory layout, alignment, partial tails, evaluation cost, and whether interleaving independent instructions hides latency. If EVALUATE is tiny, packing and boundary handling can dominate.

## 12.2 Reduction: one output, many contributors

~~~text
acc = IDENTITY
FOR r IN REDUCTION_DOMAIN
    acc = COMBINE(acc, EVALUATE(r))
result = FINALIZE(acc)
~~~

Three candidate schedules illustrate different resource costs.

**A. Ordered scalar fold** (reference; useful when COMBINE is nonassociative):

~~~text
acc = IDENTITY
FOR r IN ORIGINAL_ORDER
    acc = COMBINE(acc, EVALUATE(r))
result = FINALIZE(acc)
~~~

**B. Reduction lanes** (requires legal reassociation):

~~~text
partials = VECTOR_IDENTITY(V)
FOR rtile IN TILES(REDUCTION_DOMAIN, V)
    x = VECTOR_EVALUATE(rtile)
    partials = VECTOR_COMBINE(partials, x)
result = FINALIZE(HORIZONTAL_REDUCE(partials, COMBINE))
~~~

**C. Several independent outputs** (output lanes, usually no horizontal fold):

~~~text
FOR otile IN TILES(OUTPUTS, V)
    accum = VECTOR_IDENTITY(otile)
    FOR r IN REDUCTIONS_FOR_TILE(otile)
        accum = VECTOR_COMBINE(
            accum, VECTOR_EVALUATE(otile, r))
    VECTOR_STORE_FINALIZED(otile, accum)
~~~

A longer reduction is not automatically a reason to choose reduction lanes. When many independent outputs exist, output lanes can keep each accumulator independent and avoid horizontal communication.

## 12.3 Batched multi-output reduction

Suppose every contribution **r** is shared by several outputs **o**:

~~~text
FOR o
    acc[o] = IDENTITY
    FOR r
        acc[o] = COMBINE(acc[o], EVALUATE(o, r))
~~~

Reordering to **r → o** may reuse inputs:

~~~text
acc[ALL_OUTPUTS] = IDENTITIES

FOR r
    common = LOAD_SHARED_INPUT(r)
    FOR otile IN OUTPUT_TILES
        acc[otile] = VECTOR_COMBINE(
            acc[otile],
            VECTOR_EVALUATE(common, otile))
~~~

This becomes inefficient if all partial outputs must be updated in memory for every r. A hybrid output tile keeps a bounded group of partials live while traversing all r; it sacrifices some shared-input reuse but avoids repeated output traffic.

## 12.4 Segmented reductions

A different reduction length per output creates ragged loops:

~~~text
FOR o
    acc = IDENTITY
    FOR r IN SEGMENT(o)
        acc = COMBINE(acc, EVALUATE(o,r))
    result[o] = FINALIZE(acc)
~~~

Choose among:

- **Several outputs per vector** when segments are similarly short.
- **Several contributions per vector** for long segments.
- **Bucketed outputs** when segment sizes vary greatly.
- **Flattened contribution list** with segment markers when work balance dominates.

The correct mapping is determined by the *distribution of segment lengths*, not just the average.

## 12.5 Multiple independent accumulator chains

An accumulator dependency can limit throughput even if its update is vector-wide. If algebra permits independent partials:

~~~text
partial[0..U-1] = IDENTITIES

FOR r IN REDUCTION_DOMAIN
    chain = PARTITION_ASSIGNMENT(r, U)
    partial[chain] = COMBINE(partial[chain], EVALUATE(r))

result = FINALIZE(MERGE(partial, DEFINED_ORDER))
~~~

Increasing U can expose parallel instruction chains. It also raises live-state demand and can change numerical order. The optimal U is a measured property.

## 12.6 Identity, empty domains, and finalization

The identity must be correct for COMBINE. For a minimum it may be +infinity, for a maximum −infinity, for a logical AND true. Empty reductions can require special output semantics. FINALIZE may include normalization, tie breaking, quantization, or clamping and should generally happen after the same set of contributions as in the reference.

## 12.7 A decision table

| Work shape | First architecture to test | Alternative |
| --- | --- | --- |
| Millions of independent outputs | Adjacent output lanes | Pack by attribute/layout |
| One output, long associative reduction | Reduction lanes | Multiple partial chains |
| Many outputs, common reduction inputs | Output tile stationary | Input tile stationary |
| Many short ragged segments | One segment per lane | Bucket by length |
| Few very long segments | Contribution lanes | Parallel partial reductions |
| Strict ordered fold | Independent problems in lanes | Proven scan-specific transformation |

**Takeaway:** distinguish the number of **independent results** from the number of **contributions per result** before assigning lanes.
