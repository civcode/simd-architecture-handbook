# 21. Systematic architecture search and measurement

[Previous](20-performance-models.md) · [Contents](README.md) · [Next](22-portability.md)

Efficient SIMD architecture is a constrained search problem. The constraints are correctness, dependencies, memory safety, and machine resources. Performance is an empirical objective.

## 21.1 Define the architecture tuple

Let

\[
A = (L, T, V, S, M, U, B)
\]

where:

| Variable | Choice |
| --- | --- |
| L | Logical loop schedule and interchange |
| T | Tile shapes at register/cache/task levels |
| V | Lane mapping and effective vector width |
| S | Which data is kept stationary |
| M | Memory layout, address pattern, conversion |
| U | Independent accumulator chains and unrolling |
| B | Boundary, masking, grouping, and tail strategy |

The best architecture is:

\[
A^\star = \arg\min_{A\in\mathcal{A}_{\text{legal}}} T_{\text{measured}}(A)
\]

where the legal set excludes schedules violating dependencies, observable semantics, or memory safety.

## 21.2 Stage 1: semantic and dependency model

Write the reference pseudocode and classify every axis: problem, output, reduction, ordered state, or conditional/irregular dimension. Identify which axes commute and what output equality means.

Do not change source layout or execution order until the reference behavior is precisely defined.

## 21.3 Stage 2: generate *families*, not blind permutations

Generate promising architectural families:

1. Output-stationary.
2. Input-stationary.
3. Problem-stationary.
4. Reduction-lane or multi-chain.
5. Hybrid tiled schedules.
6. Bucketing/compaction for ragged work.
7. Wavefront/scan transformations for dependency chains.
8. Packed or transformed data layouts.

Prune candidates that cannot exploit meaningful independence or would demand unbounded live accumulators.

## 21.4 Stage 3: resource feasibility

~~~text
candidates = GENERATE_LEGAL_SCHEDULES(specification)

FOR candidate IN candidates
    candidate.tile_options = CHOOSE_SMALL_TILE_GRID()
    candidate.live_vectors = ESTIMATE_LIVE_VECTORS(candidate)
    candidate.footprint = ESTIMATE_WORKING_SET(candidate)
    candidate.lane_occupancy = ESTIMATE_ACTIVE_LANES(candidate)
    candidate.communication = ESTIMATE_DATA_MOVEMENT(candidate)

    IF NOT ARITHMETICALLY_SAFE(candidate)
        REJECT(candidate)
    ELSE IF LIKELY_TO_SPILL_EXCESSIVELY(candidate)
        DEPRIORITIZE(candidate)
    ELSE
        KEEP(candidate)
~~~

Feasibility estimates should narrow the list without assuming a universal performance order.

## 21.5 Stage 4: correctness-first implementation

For each candidate, compare to a trusted reference on diverse randomized and adversarial cases. Include incomplete tiles, extreme values, repeated indices, boundary data, empty segments, and minimum/maximum task lengths.

Never accept a performance win that changes the contract unintentionally.

## 21.6 Stage 5: benchmark protocol

1. Use the same data for competing architectures.
2. Warm up the relevant code and inputs before the measured section.
3. Randomize or rotate candidate execution order to avoid systematic drift.
4. Record repetitions across fresh runs, not only many iterations in one process.
5. Record machine, compiler/code-generator configuration, workload, input distribution, and parallel worker count.
6. Report **median**, distribution/spread, and any outliers or distinct timing regimes.
7. Compare the end-to-end pipeline when packing, layout conversion, or scheduling changes.
8. Profile likely bottlenecks when one architecture surprises the model.

Large speedups should be treated as measurements under a specific workload, not as portable laws.

## 21.7 Control hidden variables

Potential confounders: CPU frequency state, thermal behavior, thread migration, background activity, cache residency, allocation layout, dynamic dispatch, and benchmark instrumentation overhead.

Use equivalent input sets and avoid interpreting a single anomalous timing point as an algorithmic property. Re-running with interleaved candidate order helps, but cannot eliminate every environmental effect.

## 21.8 Change one dimension at a time, then explore interactions

A sensible first pass:

~~~text
baseline = VALIDATED_REFERENCE_ARCHITECTURE()

FOR lane_mapping IN FEASIBLE_LANE_MAPS
    MEASURE_WITH_FIXED_OTHER_CHOICES(lane_mapping)

FOR loop_schedule IN PROMISING_SCHEDULES
    MEASURE_WITH_FIXED_LAYOUT_AND_TILES(loop_schedule)

FOR tile_shape IN RESOURCE_FEASIBLE_GRID
    MEASURE(tile_shape)

FOR layout IN FEASIBLE_DATA_LAYOUTS
    MEASURE_TOTAL_PIPELINE(layout)
~~~

After narrowing the search, test interactions (for example, layout × lane mapping × tile shape). Changing only one parameter forever can miss a better combination.

## 21.9 Architecture comparison worksheet

| Candidate | Lane axis | Outermost tile | Live partials | Input access | Cross-lane work | Expected risk |
| --- | --- | --- | --- | --- | --- | --- |
| A | outputs | output tile | high/medium | repeated | low | spills |
| B | contributions | reduction tile | few | contiguous | high | horizontal fold |
| C | tasks | task tile | per task | potentially gathered | low | divergence |
| D | outputs | input tile | repeatedly updated | high reuse | low | output traffic |

Fill this with workload-specific values rather than generic labels before benchmarking.

## 21.10 When to stop

Stop tuning when gains are smaller than measurement uncertainty, a more complex schedule harms maintainability for negligible benefit, or remaining runtime is dominated by an unaffected component. Document why the chosen architecture wins, and the workload shapes on which it may lose.

**Takeaway:** architecture selection is an experimental science with correctness constraints—not a rule that the widest vectors or deepest tiles always win.
