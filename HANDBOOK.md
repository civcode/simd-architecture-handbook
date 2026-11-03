# SIMD Architecture Handbook

*Designing Vectorized Algorithms — Complete Single-File Edition*

This standalone document combines **all 24 chapters and the glossary** from the modular [SIMD Architecture Handbook](https://github.com/civcode/simd-architecture-handbook/tree/main/book). It uses function-agnostic, instruction-set-independent pseudocode. The two earlier guides remain separately preserved in the repository under `references/`.

**Principle:** choose a legal computational schedule, vector lane meaning, stationary data, and tile hierarchy first; individual vector instructions are only one way to realize that architecture.

## Contents

### Part I — Model and transform the computation
- [1. The semantic contract and iteration-space model](#1-the-semantic-contract-and-iterationspace-model)
- [2. Dependence analysis and transformation legality](#2-dependence-analysis-and-transformation-legality)
- [3. Mapping work to vector lanes](#3-mapping-work-to-vector-lanes)
- [4. Loop architectures from zero to N levels](#4-loop-architectures-from-zero-to-n-levels)
- [5. The algebra of loop transformations](#5-the-algebra-of-loop-transformations)

### Part II — Data movement and execution architecture
- [6. Output-, input-, and problem-stationary schedules](#6-output-input-and-problemstationary-schedules)
- [7. Register, cache, and multi-level tiling](#7-register-cache-and-multilevel-tiling)
- [8. Memory access and data-layout co-design](#8-memory-access-and-datalayout-codesign)
- [9. Lane communication, reductions, and rearrangement](#9-lane-communication-reductions-and-rearrangement)
- [10. Predication, divergence, masks, and tails](#10-predication-divergence-masks-and-tails)
- [11. Irregular indexing, conflicts, and sparse work](#11-irregular-indexing-conflicts-and-sparse-work)

### Part III — Transform algorithm families
- [12. Maps, independent outputs, and reductions](#12-maps-independent-outputs-and-reductions)
- [13. Scans, recurrences, and dependency chains](#13-scans-recurrences-and-dependency-chains)
- [14. Neighborhoods, wavefronts, and space-time schedules](#14-neighborhoods-wavefronts-and-spacetime-schedules)
- [15. Search, selection, ordering, and comparison networks](#15-search-selection-ordering-and-comparison-networks)
- [16. Segmented, sparse, and indirect algorithms](#16-segmented-sparse-and-indirect-algorithms)
- [17. Stateful, variable-length, and convergent algorithms](#17-stateful-variablelength-and-convergent-algorithms)
- [18. Multi-stage pipelines, fusion, and recomputation](#18-multistage-pipelines-fusion-and-recomputation)

### Part IV — Prove and optimize
- [19. Numerical correctness and semantic equivalence](#19-numerical-correctness-and-semantic-equivalence)
- [20. Performance and communication models](#20-performance-and-communication-models)
- [21. Systematic architecture search and measurement](#21-systematic-architecture-search-and-measurement)
- [22. Portable vectors and parallel hierarchy](#22-portable-vectors-and-parallel-hierarchy)

### Part V — Practice
- [23. Fifteen abstract before/after case studies](#23-fifteen-abstract-beforeafter-case-studies)
- [24. Design worksheet, exercises, and answer sketches](#24-design-worksheet-exercises-and-answer-sketches)

- [Glossary and notation](#glossary-and-notation)

---

## 1. The semantic contract and iteration-space model


### 1.1 Begin with a specification, not a vector

Before changing a loop, define what the computation **means**. The reference computation is an executable mathematical contract, while loop order and lane width are implementation decisions.

A useful generic form is:

~~~text
FOR each independent problem p IN P
    FOR each output o IN OUTPUTS(p)
        a = IDENTITY(p, o)
        FOR each contribution r IN CONTRIBUTIONS(p, o)
            value = EVALUATE(p, o, r, inputs)
            a = COMBINE(a, value)
        result[p, o] = FINALIZE(a)
~~~

The iteration space is the set of triples **(p, o, r)**, not a particular nesting. Real algorithms may also have ordered state **s**, conditional edges, variable bounds, or multiple reductions.

Specify:

| Contract item | Design question |
| --- | --- |
| Inputs | Mutable? Aliased? Dense or indirect? |
| Outputs | Independent? Ordered? Can two iterations target one location? |
| Contribution | What data is read for each (p,o,r)? |
| Combine | Associative? Commutative? Exact? Saturating? |
| Finalize | Applied after all contributions or at each step? |
| Boundaries | What happens for zero-length, partial, invalid, or masked work? |
| Observable behavior | Exact equality, error bound, tie rule, deterministic order? |

### 1.2 Distinguish dimensions from loops

A **logical axis** describes varying work; a **loop** specifies when it executes.

An iteration space of depth three can be implemented as two loops when one axis is expressed by lanes, or as six loops after tiling. Similarly, a one-dimensional iteration space may use several nested loops to traverse cache tiles and vector tails.

The proposed architecture is a schedule:

~~~text
SCHEDULE(logical_iteration) ->
    (task, tile, sub_tile, vector_group, lane, execution_order)
~~~

A schedule is valid when it includes all required iterations, preserves required dependency order, and commits outputs without races or unintended duplication.

### 1.3 Separate four kinds of work

**Independent work:** can proceed in any order and be distributed across lanes or tasks.

**Reduction work:** several contributions produce one output; arbitrary reassociation is legal only if the combine operation permits it.

**Ordered work:** step k reads a result from an earlier step; a schedule must respect that edge or use a proven algorithmic transformation.

**Conflicting work:** two iterations may update the same location. Independence cannot be assumed from distinct loop indices alone.

These categories—not the names of application functions—determine the available SIMD architectures.

### 1.4 Resource vocabulary

- **Lanes:** conceptual parallel positions for simultaneous values.
- **Accumulators:** live partial outputs, possibly one per lane.
- **Tile:** a bounded set of logical iterations processed as a unit.
- **Stationary resource:** the data deliberately retained through an inner loop.
- **Lane communication:** movement or reduction among lane positions.
- **Register pressure:** number and width of simultaneously live values.
- **Working set:** data referenced while a tile is processed, distinguishing unique bytes from repeated accesses.
- **Tail:** logical work not filling a complete vector or tile.

### 1.5 Example: identical semantics, different schedules

~~~text
# Schedule A: contributions inside each output
FOR p
    FOR o
        a = IDENTITY
        FOR r
            a = COMBINE(a, EVALUATE(p, o, r))
        result[p,o] = FINALIZE(a)

# Schedule B: a tile of outputs shares traversal of r
FOR p
    FOR otile IN TILES(OUTPUTS(p), T_O)
        a[otile] = IDENTITY_VECTOR(otile)
        FOR r
            a[otile] = VECTOR_COMBINE(
                a[otile], VECTOR_EVALUATE(p, otile, r))
        STORE_FINALIZED(result, otile, a[otile])
~~~

B retains multiple output accumulators, while A has a shorter live range and may need a horizontal reduction if its innermost dimension becomes the vector axis. Neither is inherently faster; each implies different memory traffic and register demands.

### 1.6 First exercise

Given a computation with two outputs that read overlapping input regions but different contribution counts, identify (a) legal output parallelism, (b) reduction order constraints, (c) shared inputs, and (d) whether output-stationary or input-stationary work deserves testing.

**Takeaway:** optimize a *semantic iteration space*, not the incidental spelling of its original loops.


---

## 2. Dependence analysis and transformation legality


### 2.1 The four dependence questions

For every pair of iterations, ask whether the later iteration reads data written by the earlier one (**true dependence**), overwrites data the earlier one will read (**anti-dependence**), writes the same location (**output dependence**), or accesses disjoint state.

Loop interchange preserves results only when the new ordering respects all required dependences, or when a proven transformation removes them.

A compact dependency table:

| Relation | Effect on vectorization | Common response |
| --- | --- | --- |
| Independent | Broad freedom | Put iterations in lanes |
| Reduction | Reassociation may be restricted | Separate partial sums, preserve order, or tolerate differences |
| Loop-carried true dependence | Must respect producer before consumer | Scan, wavefront, independent streams |
| Anti/output dependence | Ordering or renaming required | Privatize, rename buffers, double buffer |
| Possible alias | Independence unknown | Prove nonaliasing or use conservative path |
| Conflict on indexed write | Several lanes may collide | Group, privatize, combine, or resolve conflicts |

### 2.2 Read/write-set method

Let **READ(i)** and **WRITE(i)** be locations read and written by iteration i. If for two iterations i and j all relevant intersections involving writes are empty, they are memory-independent. Otherwise an edge may exist.

This is a reasoning tool, not an assumption that array subscripts imply distinct addresses. Two different indices into an indirection array may still target the same destination.

~~~text
FOR i IN iteration_space
    READ_SET[i]  = ADDRESSES_READ_BY(i)
    WRITE_SET[i] = ADDRESSES_WRITTEN_BY(i)

FOR each ordered pair (i, j)
    IF DEPENDENCE_EXISTS(READ_SET, WRITE_SET, i, j)
        ADD_EDGE(i, j)
~~~

A legal parallel schedule is a topological ordering of the required edges. Side effects outside arrays—including observable I/O, exceptions, and mutable global state—also matter.

### 2.3 Reductions are not automatically associative

~~~text
FOR r
    sum = COMBINE(sum, EVALUATE(r))
~~~

If **COMBINE** is associative under the specified arithmetic semantics, lane-local accumulators may be merged in another order. If it is not, a regrouping changes results.

**Integer addition** may be associative only under a defined overflow regime; saturating addition generally is not. **Floating-point addition** is not exactly associative. An approximate result contract may permit a different tree, but then specify error bounds and reproducibility expectations.

### 2.4 Privatization removes write conflicts

~~~text
# Conflicting direct update
FOR i
    destination[INDEX(i)] =
        COMBINE(destination[INDEX(i)], VALUE(i))

# Conditional transformation
FOR tile IN TILES(iterations, T_I)
    PRIVATE = EMPTY_PARTIAL_UPDATES()
    FOR i IN tile
        LOCAL_ACCUMULATE(PRIVATE, INDEX(i), VALUE(i))
    MERGE_WITH_CONFLICT_POLICY(destination, PRIVATE)
~~~

This is legal only when the merge implements the reference contract. A local map can increase memory usage and may be slower than a direct update if conflicts are rare.

### 2.5 Dependency direction can reveal a wavefront

~~~text
FOR time
    FOR position
        state[time, position] = F(
            state[time-1, position],
            state[time-1, position-1])
~~~

Every point depends only on the previous time step. Positions at a fixed time are independent if all reads address the completed earlier layer. A schedule can process **time** sequentially and **position tiles** in parallel. Different dependence graphs may instead admit diagonal wavefronts; never assume diagonals are safe without checking the actual edges.

### 2.6 Legality checklist

Before applying a transformation, answer:

1. Does every required logical iteration execute exactly once?
2. Is every read preceded by its producer?
3. Are all simultaneous writes independent or conflict-resolved?
4. Is accumulator combination order equivalent under the contract?
5. Are intermediate and final conversions applied at the same semantic points?
6. Does the transformed code preserve boundary conditions and side effects?

**Takeaway:** the fastest illegal schedule is not an optimization.


---

## 3. Mapping work to vector lanes


### 3.1 The fundamental lane choices

A logical vector of width **V** can represent:

| Mapping | Lane i represents | Typical benefit | Typical cost |
| --- | --- | --- | --- |
| Output lanes | A distinct output o | Independent accumulator lanes, no horizontal reduction | Many live outputs |
| Reduction lanes | A distinct contribution r | Contiguous input traversal | Horizontal reduction and altered order |
| Problem lanes | A distinct independent task p | Batched reuse of control and vector utilization | Divergence, irregular interleaved inputs |
| Candidate lanes | A different alternative for the same output | Concurrent comparisons or hypotheses | Cross-lane selection |
| Mixed map | Flattened pair such as (p,o) | Flexibility and high occupancy | Index conversion and layout complexity |

The logical choice is independent of any named instruction set.

### 3.2 Output mapping

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

### 3.3 Reduction mapping

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

### 3.4 Problem mapping

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

### 3.5 Selecting lane meaning

Estimate for each mapping:

1. Contiguous bytes read and written per vector operation.
2. Number of useful lanes across the actual workload distribution.
3. Number of live accumulator and temporary vectors.
4. Cross-lane reductions, shuffles, and conversions.
5. Setup cost paid per tile.
6. Branch divergence and data-dependent memory access.
7. Correctness implications of changing reduction order.

Compare **cost per useful result**, not merely operations per vector.

### 3.6 Mixed mapping and dimensional flattening

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


---

## 4. Loop architectures from zero to N levels


### 4.1 Count logical axes, not syntactic loops

A **zero-loop** computation is a single logical operation; it can still use a vector internally. A one-axis computation can gain tile loops. A many-axis computation can be flattened into a vector index. Thus neither source loop count nor loop nesting depth predicts SIMD suitability.

| Logical dimensions | Natural starting point | Main architecture decision |
| --- | --- | --- |
| 0 | One vector operation | Is packing/setup worthwhile? |
| 1 | Vectorized independent items | Tail, alignment, throughput, latency |
| 2 | Outputs × reductions, or two independent axes | Which axis occupies lanes? |
| 3 | Problems × outputs × reductions | Outer owner, lane mapping, reuse |
| 4–N | Repeated independent, output, reduction, state axes | Scheduling and hierarchical tiles |
| Ragged/dynamic | Variable-length nested work | Grouping, masking, queues, segmented processing |

### 4.2 Zero and one logical loop

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

### 4.3 Two-loop choices

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

### 4.4 Three logical axes and six orders

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

### 4.5 Four or more axes: classify before permuting

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

### 4.6 When a flattened index is better

If multiple small independent axes leave many lanes empty, flatten their product:

~~~text
items = ENUMERATE_VALID_COMBINATIONS(P0, P1, O0)

FOR vector_tile IN TILES(items, V)
    ids = DECODE_COORDINATES(vector_tile)
    VECTOR_EXECUTE(ids)
~~~

The mapping costs division/modulo, decoding, irregular stores, or a prebuilt index table. Prefer a simple affine decoding rule or precomputed compact work list when the original dimensions are tiny.

### 4.7 Variable-depth and ragged iteration

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

### 4.8 A decision procedure for arbitrary depth

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


---

## 5. The algebra of loop transformations


Transformations create schedules with different communication and parallelism. Every one must be checked against the dependence graph and numerical contract.

### 5.1 Interchange

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

### 5.2 Strip mining and vector grouping

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

### 5.3 Multi-dimensional tiling

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

### 5.4 Fusion versus fission

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

### 5.5 Collapse and flattening

~~~text
FOR i IN I
    FOR j IN J
        PROCESS(i,j)

FOR k IN 0..(SIZE(I)*SIZE(J))-1
    i,j = DECODE(k)
    PROCESS(i,j)
~~~

Useful when the original inner axis is shorter than a vector, or when tasks must be distributed evenly. Beware index-decoding overhead and loss of predictable data strides.

### 5.6 Unrolling and unroll-and-jam

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

### 5.7 Loop skewing and wavefront scheduling

If an iteration at (i,j) depends on (i-1,j) and (i,j-1), processing a row in parallel violates an edge. Let **wavefront = i + j**. All direct predecessors lie on the previous wavefront, so equal-wavefront points are independent *provided no additional dependencies exist*.

~~~text
FOR w IN LEGAL_WAVEFRONT_RANGE
    ready = POINTS_WITH_SUM_OF_COORDINATES(w)
    FOR tile IN TILES(ready, V)
        predecessors = READ_PRIOR_WAVEFRONTS(tile)
        WRITE_RESULTS(tile, VECTOR_EVALUATE(predecessors))
~~~

The transformation creates parallelism at the cost of noncontiguous access and small wavefronts near boundaries. Tile the wavefronts when the state space is large.

### 5.8 Loop reversal and distribution

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

### 5.9 Reduction splitting and tree re-association

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

### 5.10 A transformation ledger

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


---

## 6. Output-, input-, and problem-stationary schedules


A **stationary** schedule keeps one expensive-to-move resource near the compute units while repeatedly using it. The outermost loop usually chooses the owner of that resource.

### 6.1 Output-stationary: retain partial results

~~~text
FOR problem IN PROBLEMS
    FOR otile IN TILES(OUTPUTS(problem), T_O)
        accum = VECTOR_IDENTITY(otile)
        FOR r IN CONTRIBUTIONS(problem, otile)
            accum = VECTOR_COMBINE(
                accum, VECTOR_EVALUATE(problem, otile, r))
        STORE_FINALIZED(problem, otile, accum)
~~~

The output tile is initialized once, updated repeatedly in fast storage, and committed once. **Best candidate when** output updates are expensive, reduction length is substantial, and input reuse across *different* output tiles is moderate.

**Potential failure:** each tile rereads common inputs; too many live outputs trigger spills or reduce independent load/computation overlap.

### 6.2 Input-stationary: retain shared source values

~~~text
FOR problem IN PROBLEMS
    CLEAR(output_partials[problem])
    FOR rtile IN TILES(CONTRIBUTIONS(problem), T_R)
        source_tile = LOAD_SHARED_INPUTS(problem, rtile)
        FOR otile IN TILES(OUTPUTS(problem), T_O)
            updates = VECTOR_EVALUATE(source_tile, otile)
            ACCUMULATE_TO_PARTIALS(output_partials, otile, updates)
    FINALIZE_ALL(output_partials)
~~~

**Best candidate when** one source tile contributes to many outputs and loading those sources is expensive.

**Potential failure:** partial-output updates repeatedly traverse memory; if the output set is small enough to remain in registers, an output-stationary schedule may save more traffic.

### 6.3 Problem-stationary: retain task context

~~~text
FOR ptile IN TILES(PROBLEMS, T_P)
    task_context = INITIALIZE_TASK_STATES(ptile)
    FOR stage IN STAGES
        EXECUTE_STAGE_ON_TASK_TILE(task_context, stage)
    COMMIT_TASK_RESULTS(task_context)
~~~

This is valuable when each task has private state or when multiple stages can reuse setup for the same task. It can be weak when inputs across adjacent tasks overlap but the task tile prevents shared-input reuse.

### 6.4 Hybrid ownership is often better

Rather than choose only one stationary resource, tile two axes:

~~~text
FOR ptile IN TILES(PROBLEMS, T_P)
    FOR otile IN TILES(OUTPUTS(ptile), T_O)
        output_acc = KEEP_LIVE(otile)
        FOR rtile IN TILES(REDUCTIONS(ptile,otile), T_R)
            shared_inputs = STAGE_INPUT_TILE(ptile, rtile)
            output_acc = FOLD(output_acc, shared_inputs, otile)
        COMMIT(output_acc)
~~~

Here tasks are outermost, output partials stay in registers, and each reduction tile can stage shared input values. The tile sizes determine which reuse is worth retaining.

### 6.5 Simple communication model

Let:

- **B_in** = bytes of shared input per reduction tile.
- **B_out** = bytes of live outputs for an output tile.
- **N_O** = number of output tiles.
- **N_R** = number of reduction tiles.

Ignoring caching and internal temporal overlap:

- Output-stationary may stage shared input **N_O × N_R** times while committing output once.
- Input-stationary may stage shared input **N_R** times while reading/writing output partials as often as **N_O × N_R**.

This is a **traffic upper-level comparison**, not a prediction of DRAM bytes. Frequently reused data may be served by caches, hardware prefetchers, or register-resident sub-tiles.

### 6.6 Why the outermost tile matters

Consider two candidate schedules:

~~~text
# Choose a patch of outputs first
FOR output_patch
    FOR contributing_patch
        UPDATE_OUTPUT_PATCH()

# Choose contributing inputs first
FOR contributing_patch
    FOR output_patch
        UPDATE_OUTPUT_PATCH()
~~~

The first retains outputs but reloads inputs; the second reuses inputs but transports output partials. The fastest choice changes as output count, overlap, register budget, and cache residency change.

### 6.7 Stationary selection questions

1. What is expensive to reload: inputs, outputs, or task metadata?
2. How many uses occur before a stationary value is no longer needed?
3. Does retaining it consume scarce registers or scratch storage?
4. Can neighboring output or problem tiles share the value?
5. What is the address stride of the innermost work?
6. Does a larger outer tile increase irregular masking or divergence?
7. Which resource is actually bottlenecking the current schedule?

**Takeaway:** stationary architecture is a choice of **data lifetime**, not merely a loop permutation.


---

## 7. Register, cache, and multi-level tiling


### 7.1 Why one tile size is insufficient

A fast architecture typically coordinates several capacities: registers for active operands and partial results; nearby caches for reused input/output tiles; larger caches for task and working-set staging; memory for the complete data set.

A tile that fits the largest cache may still spill registers. A register-perfect microtile may repeatedly reload data that a larger cache tile could share.

~~~text
FOR task_tile IN TILES(TASKS, T_TASK)
    FOR cache_output_tile IN TILES(OUTPUTS, T_OC)
        FOR cache_reduction_tile IN TILES(REDUCTIONS, T_RC)
            STAGE_SHARED_INPUTS(cache_reduction_tile)
            FOR reg_output_tile IN TILES(cache_output_tile, T_OR)
                acc = LOAD_OR_INITIALIZE_PARTIAL(reg_output_tile)
                FOR reg_reduction_tile IN TILES(cache_reduction_tile, T_RR)
                    acc = VECTOR_FOLD(acc, reg_reduction_tile)
                SAVE_PARTIAL(reg_output_tile, acc)
        FINALIZE(cache_output_tile)
~~~

This hierarchy may incur partial stores for every cache reduction tile. A better schedule for small outputs could keep a register output tile alive across *all* reduction tiles; choose based on register and traffic constraints.

### 7.2 A register-budget estimate

Let **R_total** be the available architectural register capacity measured in *vector register equivalents*. Estimate:

~~~text
R_required =
    R_output_accumulators
  + R_input_operands
  + R_temporary_values
  + R_masks_and_indices
  + R_pipeline_overlap
  + R_safety_margin
~~~

Feasibility requires **R_required <= usable register budget**. This is approximate: compilers or implementations may require additional registers, and register allocation granularity is architecture dependent.

For output-lane mapping:

~~~text
accumulator_vectors =
    CEIL(T_OUTPUT / EFFECTIVE_LANES_PER_ACCUMULATOR)
    × INDEPENDENT_ACCUMULATOR_CHAINS
~~~

Increasing tile area can increase live vectors even if the total number of arithmetic operations is unchanged.

### 7.3 How to select a tile shape

Suppose a two-dimensional output tile is **T_A × T_B**. A wider tile may improve contiguous access along A, while a taller tile may increase reuse along B. Equal area does not imply equal access cost.

Evaluate the footprint:

~~~text
footprint(tile) =
    UNIQUE_INPUT_BYTES(tile + required_halo)
  + LIVE_PARTIAL_OUTPUT_BYTES(tile)
  + INDEX_AND_MASK_BYTES(tile)
  + SCRATCH_BYTES(tile)
~~~

Choose candidate shapes under cache and register budgets, then measure. Unique bytes and **actual transferred bytes** differ because of alignment, caching, read-for-ownership, and cache-line size.

### 7.4 Reduction chunk size from arithmetic bounds

If a narrow accumulator has maximum representable positive value **M**, each contribution lies in **[0, C]**, and initial value is zero, at most

\[
K_{\max} = \left\lfloor\frac{M}{C}\right\rfloor
\]

contributions can be added without overflow. Choose a chunk size no larger than this bound **provided the partial accumulation truly obeys those assumptions**.

~~~text
wide_total = 0
FOR chunk IN TILES(CONTRIBUTIONS, K_safe)
    narrow_acc = 0
    FOR r IN chunk
        narrow_acc += NONNEGATIVE_BOUNDED_CONTRIBUTION(r)
    wide_total += WIDEN(narrow_acc)
~~~

Signed contributions, saturating operators, nonzero initial states, and other semantics require different reasoning. Always flush the final incomplete chunk.

### 7.5 Trade-off: independent chains versus spills

~~~text
FOR output_tile
    acc[0..U-1] = IDENTITIES
    FOR reduction_tile
        FOR chain IN 0..U-1
            acc[chain] = COMBINE(
                acc[chain], CONTRIBUTION(chain, reduction_tile))
    COMMIT(MERGE(acc))
~~~

More chains can hide dependency latency; too many may spill. The optimal U depends on actual operations, compiler/code generator decisions, lane width, and available execution units.

### 7.6 Halo and boundary overhead

For a neighborhood of radius h around a tile with shape **T_A × T_B**, the required input region may have size **(T_A+2h) × (T_B+2h)**. The ratio of staged input area to useful output area is:

\[
\frac{(T_A+2h)(T_B+2h)}{T_A T_B}
\]

Small tiles waste more boundary overlap; huge tiles may no longer fit in fast storage. This is an example of selecting the **outermost patch** from data reuse, not from loop naming.

### 7.7 Tiling for parallel tasks

Tile ownership also determines scheduling overhead and imbalance. A large task tile amortizes setup but creates fewer parallel tasks. A small task tile balances better but increases task-management cost and often duplicates context.

Use a two-level strategy:

~~~text
PARALLEL_FOR task_chunk IN TASK_CHUNKS
    LOCAL_SCRATCH = ALLOCATE_ONCE(task_chunk)
    FOR vector_tile IN VECTOR_TILES(task_chunk)
        EXECUTE_WITH_LOCAL_SCRATCH(vector_tile, LOCAL_SCRATCH)
~~~

Do not share mutable scratch across simultaneous tasks unless explicitly synchronized.

### 7.8 Practical tile search

Try a bounded candidate set for **T_OUTPUT**, **T_REDUCTION**, **T_TASK**, and accumulator chain count. Reject candidates that violate arithmetic, bounds, or register constraints. Compare median runtime *and* variation across representative short, medium, and long reduction lengths.

**Takeaway:** every tile size expresses a compromise between work reuse, storage capacity, and scheduling overhead.


---

## 8. Memory access and data-layout co-design


A vector schedule and its data layout form one design. An excellent lane mapping can become slow when adjacent lanes read distant addresses.

### 8.1 Five important access patterns

| Pattern | Abstract address sequence for lanes | Typical concern |
| --- | --- | --- |
| Contiguous | base, base+1, base+2, ... | Usually efficient transfers |
| Constant stride | base, base+s, base+2s, ... | Wasted transferred bytes when s is large |
| Broadcast | base, base, base, ... | One value reused by many lanes |
| Gather | index[0], index[1], ... | Scattered requests, index overhead |
| Scatter | destination[index[0..]] | Conflicts and locality of writes |

These describe *addresses*, not specific instructions. Hardware may lower a gather to several loads. Even contiguous accesses can cross cache lines, pages, or memory boundaries.

### 8.2 Derive addresses for every lane

~~~text
FOR lane IN ACTIVE_LANES
    logical = MAP_LANE_TO_ITERATION(tile, lane)
    read_addresses[lane]  = READ_ADDRESSES(logical)
    write_addresses[lane] = WRITE_ADDRESSES(logical)

ESTIMATE_UNIQUE_CACHE_LINES(read_addresses)
ESTIMATE_UNIQUE_CACHE_LINES(write_addresses)
DETECT_POSSIBLE_WRITE_CONFLICTS(write_addresses)
~~~

Do this separately for the hot loop and for any setup or finalization pass. A schedule with cheap hot-loop loads may require an expensive one-time transpose.

### 8.3 Structure of arrays versus array of structures

Suppose every task owns attributes **a**, **b**, and **c**.

**Record layout** stores all attributes per task together. It favors processing one complete task at a time.

**Column layout** stores each attribute for all tasks contiguously. It often favors processing the same attribute across many tasks in lanes.

~~~text
# Record-oriented access
FOR task
    a,b,c = LOAD_RECORD(task)
    EXECUTE_TASK(a,b,c)

# Column-oriented access
FOR tile IN TILES(tasks, V)
    a = LOAD_COLUMN(attribute_a, tile)
    b = LOAD_COLUMN(attribute_b, tile)
    c = LOAD_COLUMN(attribute_c, tile)
    EXECUTE_TASKS(a,b,c)
~~~

Neither is universally faster. A task may need just one field or all fields, and an alternate layout can increase scalar overhead elsewhere.

### 8.4 Blocked or hybrid layouts

An array-of-short-vectors layout organizes a limited number of tasks into packs:

~~~text
FOR pack IN TASK_PACKS
    data[pack].field_a = CONTIGUOUS_VALUES_FOR_PACK(a)
    data[pack].field_b = CONTIGUOUS_VALUES_FOR_PACK(b)
    data[pack].field_c = CONTIGUOUS_VALUES_FOR_PACK(c)
~~~

Such a layout can preserve some record locality while making per-field vector access regular. Pack width should follow an architectural cost model, not necessarily a single hardware vector width.

### 8.5 Repacking has a break-even point

Let **C_pack** be transformation cost, **C_original** the cost of one pass using the original layout, and **C_packed** one pass on the new layout. Over **K** repeated passes, packing helps only if:

\[
C_{\text{pack}} + K C_{\text{packed}} < K C_{\text{original}}
\]

This ignores secondary effects such as extra memory footprint and cache eviction. A layout beneficial for one hot kernel may harm the pipeline if data must be converted back immediately.

### 8.6 Spatial versus temporal locality

- **Spatial locality:** one access brings neighboring bytes that the program soon uses.
- **Temporal locality:** a recently accessed value is used again.
- **Register reuse:** data stays in the most local storage between uses.
- **Cache reuse:** data is reloaded from a cache instead of lower memory.

Interchanging loops can trade temporal locality of accumulators against spatial locality of inputs. Look at **the entire live tile**, not only the innermost instruction.

### 8.7 Prefetching and address predictability

Regular forward streams are often easier to prefetch than unpredictable pointer chains. A schedule that reduces arithmetic but introduces dependent indirect loads can become latency-bound.

~~~text
FOR tile
    addresses = COMPUTE_NEXT_ADDRESSES(tile)
    values = LOAD_BY_ADDRESSES(addresses)
    COMPUTE(values)
~~~

Software scheduling may overlap address generation and other independent work, but speculative reads must not access invalid memory.

### 8.8 False sharing and task ownership

Two parallel tasks can write distinct logical outputs that occupy the same cache line. This does not create a data race, but can generate coherence traffic. Organize task output ranges to reduce shared lines and avoid excessive interleaving.

### 8.9 Layout-and-loop search

For each candidate, write the tuple:

~~~text
candidate = (
    output_tile_shape,
    input_tile_shape,
    lane_mapping,
    physical_data_layout,
    access_pattern,
    conversion_cost
)
~~~

Measure whole-pipeline time when conversion is involved. **Takeaway:** vectorization should minimize *moved bytes per useful result*, not merely maximize the fraction of work expressed as vector operations.


---

## 9. Lane communication, reductions, and rearrangement


Values already in vector registers are not necessarily free to move. Some operations act independently on each lane; others move information between lanes and may create additional latency, throughput demand, or cross-group restrictions.

### 9.1 Classify operations by communication

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

### 9.2 Why output lanes avoid horizontal reduction

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

### 9.3 Reduction tree cost

With V partials, a balanced associative tree takes approximately **ceil(log2 V)** combination stages, though actual instruction count depends on available rearrangements.

~~~text
values = VECTOR_PARTIALS
WHILE COUNT(values) > 1
    values = PAIRWISE_COMBINE(values)
result = values[0]
~~~

The tree is a logical illustration. Some implementations use dedicated reduction support; others use multiple moves and combinations.

### 9.4 Partial reductions and multiple independent chains

Instead of horizontally reducing every vector group:

~~~text
partial[lanes] = IDENTITY
FOR tile IN REDUCTION_TILES
    partial = VECTOR_COMBINE(partial, EVALUATE(tile))
result = HORIZONTAL_REDUCE(partial)
~~~

This reduces horizontal-operation frequency. A larger number of independent partial vectors can help latency hiding but increases register pressure.

### 9.5 Communication in scans

A prefix operation needs information from earlier lanes:

~~~text
prefix = values
distance = 1
WHILE distance < V
    shifted = SHIFT_RIGHT_WITH_IDENTITY(prefix, distance)
    # Earlier elements must appear on the LEFT for an ordered prefix.
    prefix = MASKED_COMBINE(shifted, prefix, lanes >= distance)
    distance = distance * 2
~~~

Here MASKED_COMBINE(left, right, active) computes COMBINE(left, right) for active lanes and preserves the old prefix for inactive lanes. The operand order matters: an associative operation need **not** be commutative. For a left-to-right prefix, the shifted *earlier* partial must be the left operand. This form is legal for associative COMBINE with a valid identity; it performs logarithmic cross-lane stages. When COMBINE is not associative, the reference sequential order may be unavoidable.

### 9.6 Transposition between lane maps

Sometimes one stage wants lanes as **problems**, while the next wants lanes as **fields** or **outputs**.

~~~text
tile = LOAD_RECORD_ORIENTED_GROUP()
transposed = REORGANIZE_LANES(tile, target_layout)
COMPUTE_BY_FIELD(transposed)
STORE_OR_REPACK(transposed)
~~~

This can pay off when many operations reuse the transposed representation. When only one cheap operation is performed, the transpose overhead may dominate.

### 9.7 Lane width and subgroups

Logical vector width V is not always one fully connected communication domain. A target may have subgroups with cheaper within-group exchange than cross-group exchange.

Design two reduction levels if beneficial:

~~~text
FOR subgroup
    local_result[subgroup] = REDUCE_WITHIN_SUBGROUP(values[subgroup])
result = MERGE_SUBGROUP_RESULTS(local_result)
~~~

The algorithm remains hardware-agnostic, while the schedule exposes a parameter for subgroup size.

### 9.8 Choosing among competing maps

A useful thought experiment: if mapping A performs **twice as many lane-wise calculations** but no cross-lane movement, while B halves arithmetic but needs many permutations and a horizontal fold, the cheaper architecture is not obvious. Estimate useful operations, communication, registers, and data movement together.

**Takeaway:** keep lanes independent whenever it does not sacrifice substantially more valuable input or output reuse.


---

## 10. Predication, divergence, masks, and tails


SIMD performs best when multiple lanes follow a common control path. Divergent tasks can still be vectorized by turning control flow into data flow, grouping similar tasks, or compacting active work.

### 10.1 Predication and masked select

~~~text
# Independent branches per item
FOR i
    IF CONDITION(i)
        result[i] = BRANCH_A(i)
    ELSE
        result[i] = BRANCH_B(i)

# Predicated organization
FOR tile IN TILES(items, V)
    mask = VECTOR_CONDITION(tile)
    a = VECTOR_BRANCH_A(tile, mask)
    b = VECTOR_BRANCH_B(tile, NOT mask)
    result[tile] = SELECT(mask, a, b)
~~~

Depending on the target and operations, the implementation may compute both paths or guard each path with masks. Some effects cannot be safely evaluated speculatively.

### 10.2 Masked arithmetic is not automatically a safe memory access

A masked result is not proof that invalid pointers are never dereferenced. Keep address validation and memory operation semantics distinct:

~~~text
valid = IN_BOUNDS(indices)
safe_indices = SELECT(valid, indices, SAFE_FALLBACK_INDICES)
values = SAFE_GATHER(data, safe_indices, valid)
~~~

The abstract **SAFE_GATHER** contract must guarantee no illegal access for inactive lanes. Simply loading from invalid addresses and zeroing afterward is incorrect. A fallback address must itself be valid; an explicit predicated load can be another implementation.

### 10.3 Three architectures for variable work

Consider tasks with different step counts:

~~~text
FOR task
    FOR step IN 0..length[task]-1
        UPDATE(task, step)
~~~

**A. Lockstep masking**

~~~text
FOR group IN TILES(tasks, V)
    active = VALID_GROUP_LANES(group)
    FOR step IN 0..MAX_LENGTH(group)-1
        step_mask = active AND (step < length[group])
        MASKED_UPDATE(group, step, step_mask)
~~~

Good when lengths are similar; poor when one long task keeps many lanes idle.

**B. Bucketing by length**

~~~text
buckets = GROUP_BY_WORK_ESTIMATE(tasks)
FOR bucket
    FOR group IN TILES(bucket, V)
        PROCESS_LOCKSTEP(group)
~~~

Amortizes grouping when workload is large, but changes traversal order and may scatter output stores.

**C. Active-lane compaction**

~~~text
queue = INITIAL_TASKS()
WHILE NOT EMPTY(queue)
    group = TAKE_UP_TO_V_READY_TASKS(queue)
    STEP_ALL_ACTIVE(group)
    FINISHED_TO_OUTPUTS(group)
    RETURN_UNFINISHED_TO_QUEUE(group)
~~~

Keeps lanes busy but pays for queue operations, state movement, and possibly poor locality.

### 10.4 Tail handling

Common policies:

- **Mask the final vector:** one general kernel with active-lane mask.
- **Use a smaller vector group:** can reduce masked overhead if supported by implementation.
- **Use a scalar cleanup:** simple when the remaining work is tiny.
- **Pad to a safe size:** useful for repeatedly reused data, but requires explicit initialization and safe bounds.

~~~text
FOR base IN 0..N-1 STEP V
    active = MIN(V, N-base)
    mask = LANE_ID < active
    x = SAFE_MASKED_LOAD(input, base, mask)
    y = VECTOR_EVALUATE(x, mask)
    SAFE_MASKED_STORE(output, base, y, mask)
~~~

For a reduction, masked-off lanes must contribute the identity; for a minimum operation, zero is often the wrong identity.

### 10.5 Expensive branches and side effects

Predication is an attractive replacement for branch-heavy arithmetic, but it may be wrong to evaluate both branches if either has side effects, traps, observable exceptions, or invalid memory accesses.

~~~text
IF ANY(mask_A)
    EXECUTE_A_FOR_ACTIVE_LANES(mask_A)
IF ANY(mask_B)
    EXECUTE_B_FOR_ACTIVE_LANES(mask_B)
~~~

This pattern executes only the needed branch kernels and may be preferable when one branch is rare and expensive.

### 10.6 Workload grouping break-even

Grouping helps when:

\[
\text{saved inactive-lane and branch cost}
>
\text{classification}+\text{movement}+\text{restore-order cost}
\]

A grouping strategy should be compared on *total end-to-end work*, including preprocessing and output restoration.

**Takeaway:** the fastest divergent architecture often changes **which tasks share a vector**, not merely how branch instructions are expressed.


---

## 11. Irregular indexing, conflicts, and sparse work


Irregular algorithms are not intrinsically non-vectorizable. Their limits are usually dependency uncertainty, address-generation latency, write conflicts, and low lane occupancy.

### 11.1 Reads and writes are different problems

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

### 11.2 Vectorizing independent gathers

~~~text
FOR tile IN TILES(items, V)
    indices = LOAD(read_index, tile)
    valid = VALIDATE(indices)
    values = SAFE_GATHER(data, indices, valid)
    results = VECTOR_EVALUATE(values)
    STORE(output, tile, results)
~~~

Useful when index lookups are the bottleneck? Not necessarily: a gather can be limited by several independent cache misses. Creating multiple in-flight groups can sometimes hide latency.

### 11.3 Conflict resolution architectures

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

### 11.4 Sparse work representation

Sparse computations store only active relationships. A common representation has a list of destination segments, each with a variable number of contributions:

~~~text
FOR destination IN DESTINATIONS
    acc = IDENTITY
    FOR item IN SEGMENT(destination)
        acc = COMBINE(acc, EVALUATE(item))
    result[destination] = FINALIZE(acc)
~~~

The common SIMD choices are **many destinations per vector** (good for similarly sized segments), **many contributions per vector** (good for long segments), and **flattened nonzero list** (good for balancing work across segments).

### 11.5 Work scheduling from metadata

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

### 11.6 Address locality versus load balance

Sorting by address improves locality but can unbalance work. Sorting by length improves occupancy but can scatter memory access. A useful search varies both **grouping key** and **tile size**, not only lane width.

### 11.7 Safety and determinism

Verify:

- Every valid element contributes exactly once.
- Empty segments produce the specified identity/finalization behavior.
- Index bounds are checked before any memory access.
- Collisions resolve in the required order or under a proven reduction law.
- Scratch tables are initialized and thread-private where necessary.
- Reordered outputs return to their original logical positions.

**Takeaway:** irregular SIMD design is often a **data organization problem** solved before the inner arithmetic begins.


---

## 12. Maps, independent outputs, and reductions


**Map** and **reduce** are fundamental algorithm families, but they imply different best-case SIMD architectures. A map produces independent results; a reduction combines many contributions into fewer results.

### 12.1 Map: parallelism without cross-item communication

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

### 12.2 Reduction: one output, many contributors

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

### 12.3 Batched multi-output reduction

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

### 12.4 Segmented reductions

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

### 12.5 Multiple independent accumulator chains

An accumulator dependency can limit throughput even if its update is vector-wide. If algebra permits independent partials:

~~~text
partial[0..U-1] = IDENTITIES

FOR r IN REDUCTION_DOMAIN
    chain = PARTITION_ASSIGNMENT(r, U)
    partial[chain] = COMBINE(partial[chain], EVALUATE(r))

result = FINALIZE(MERGE(partial, DEFINED_ORDER))
~~~

Increasing U can expose parallel instruction chains. It also raises live-state demand and can change numerical order. The optimal U is a measured property.

### 12.6 Identity, empty domains, and finalization

The identity must be correct for COMBINE. For a minimum it may be +infinity, for a maximum −infinity, for a logical AND true. Empty reductions can require special output semantics. FINALIZE may include normalization, tie breaking, quantization, or clamping and should generally happen after the same set of contributions as in the reference.

### 12.7 A decision table

| Work shape | First architecture to test | Alternative |
| --- | --- | --- |
| Millions of independent outputs | Adjacent output lanes | Pack by attribute/layout |
| One output, long associative reduction | Reduction lanes | Multiple partial chains |
| Many outputs, common reduction inputs | Output tile stationary | Input tile stationary |
| Many short ragged segments | One segment per lane | Bucket by length |
| Few very long segments | Contribution lanes | Parallel partial reductions |
| Strict ordered fold | Independent problems in lanes | Proven scan-specific transformation |

**Takeaway:** distinguish the number of **independent results** from the number of **contributions per result** before assigning lanes.

---

## 13. Scans, recurrences, and dependency chains


A reduction produces one summary. A **scan** produces an intermediate summary at *every* position. A **recurrence** evolves a state. Both initially appear serial, yet certain algebraic and scheduling transformations expose vector parallelism.

### 13.1 Inclusive scan

Reference:

~~~text
state = IDENTITY
FOR i IN ORIGINAL_ORDER
    state = COMBINE(state, input[i])
    prefix[i] = state
~~~

Assume COMBINE is associative with IDENTITY, and scan order is defined by the sequence of inputs. A blocked scan computes prefixes within a block and combines them with the carry from prior blocks:

~~~text
carry = IDENTITY

FOR block IN TILES(input, V)
    local = VECTOR_PREFIX_SCAN(block, COMBINE, IDENTITY)
    global_prefix = COMBINE_EACH(carry, local)
    SAFE_STORE_PREFIX(block, global_prefix)
    carry = COMBINE(carry, LAST_VALID(local))
~~~

The important dependency is **between blocks**, not necessarily between lanes within a block. The vector prefix uses lane communication; a parallel scan inside the block may require logarithmic exchange stages.

### 13.2 Proof sketch for blocked scan

Let the elements in the current block be b0, b1, ..., bk. Its j-th local prefix is b0 ⊕ ... ⊕ bj. Let carry be the prefix of every element before this block. By associativity:

carry ⊕ (b0 ⊕ ... ⊕ bj)

equals the full prefix through bj. Update carry using the last valid local prefix. If the reference defines exact floating-point addition order, the transformed expression may differ because of rounding; then exact equivalence is **not** guaranteed.

### 13.3 Exclusive scan

An exclusive scan writes the prefix **before** the current input:

~~~text
state = IDENTITY
FOR i
    output[i] = state
    state = COMBINE(state, input[i])
~~~

A vector scan can compute an inclusive scan and shift its results by one position while inserting the carry from the previous block. Ensure the first lane receives the correct identity/carry and that the final block uses only active elements.

### 13.4 A general recurrence

~~~text
state = INITIAL_STATE
FOR step IN STEPS
    state = TRANSITION(state, input[step])
    output[step] = OBSERVE(state)
~~~

This is **not** automatically a scan. Parallelizing steps requires additional structure.

Three general strategies:

1. **Independent streams:** assign one state machine per lane, preserving order *within* each stream.
2. **Composable transitions:** represent each TRANSITION as a composable operator, then scan the operators.
3. **Speculation with validation:** evaluate candidate transitions ahead, then resolve dependencies; correctness and payoff are workload-specific.

### 13.5 Composable transitions

If every step can be represented by a transformation **T_i**, and transformation composition is associative, the state after k steps is:

\[
s_k = (T_k \circ T_{k-1} \circ \dots \circ T_0)(s_{\text{initial}})
\]

This enables a scan of transformations—*not* necessarily a scan of scalar input values. **Composition order is essential:** define the scan's chronological operator by COMPOSE(earlier, later) = later ∘ earlier (apply earlier first, then later). Do not use the reverse ordering simply because composition is associative; composition is not generally commutative.

~~~text
FUNCTION COMPOSE(earlier, later):
    RETURN TRANSFORMATION_THAT_APPLIES(earlier, THEN= later)

FOR step
    op[step] = ENCODE_TRANSFORMATION(input[step])

composed_prefix = ASSOCIATIVE_SCAN(op, COMPOSE)

FOR step
    state[step] = APPLY(composed_prefix[step], initial_state)
~~~

The identity for this scan is the identity transformation. Each prefix must represent the original left-to-right sequence of state updates.

The transformed representation may be larger than the original state. The parallel scan helps only when transition composition and application are cheaper than the avoided sequential dependency.

### 13.6 Parallelism across independent recurrences

When each task owns its own state:

~~~text
FOR task
    state[task] = INITIAL(task)
    FOR step IN STEPS(task)
        state[task] = TRANSITION(state[task], input[task,step])
~~~

Vectorize over **task**, not **step**:

~~~text
FOR group IN TILES(tasks, V)
    state = LOAD_INITIAL_STATES(group)
    FOR step IN COMMON_STEP_RANGE(group)
        active = STEP_EXISTS(group, step)
        state = MASKED_TRANSITION(state, group, step, active)
    STORE_FINAL_STATES(group, state)
~~~

This preserves each task's sequential semantics. It may require bucketing to avoid idle lanes when step counts differ.

### 13.7 Recurrence distance and pipelining

A recurrence may depend on a state several steps earlier instead of only the previous step. If the dependency graph splits into independent residue classes, process those classes as separate lanes:

~~~text
FOR group_of_independent_chains
    states = LOAD_CHAIN_STARTS(group_of_independent_chains)
    FOR chain_step IN ORDER
        states = VECTOR_TRANSITION(states, chain_step)
    STORE_CHAIN_ENDS(states)
~~~

Never assume independence solely because indices differ; prove no cross-chain edges.

### 13.8 When scans lose

A scan transformation can be slower than a serial fold because of lane shuffle cost, operator representation overhead, tiny input size, or memory bandwidth saturation. Cross-block carries remain sequential in the simple blocked algorithm. A hierarchical scan can parallelize block totals at the cost of extra passes.

**Takeaway:** ask whether a dependency is **a required order** or **a structure that can be represented by associative composition**.

---

## 14. Neighborhoods, wavefronts, and space-time schedules


Neighborhood computations use nearby input elements to produce outputs. Many outputs overlap in their required inputs, which makes them ideal for **spatial tiling**. Ordered neighborhoods also introduce **wavefront** scheduling.

### 14.1 Read-only neighborhoods

Let output at coordinate x depend on inputs at offsets from an abstract set H:

~~~text
FOR x IN OUTPUT_COORDINATES
    acc = IDENTITY
    FOR offset IN H
        acc = COMBINE(acc, EVALUATE(input[x + offset]))
    output[x] = FINALIZE(acc)
~~~

If all reads come from an immutable input snapshot and outputs are distinct, x iterations are independent. Two natural lane mappings are **adjacent x positions** (reuse overlapping input windows) and **offsets in H** (one output per group, followed by a reduction).

### 14.2 Spatial output tiling

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

### 14.3 Halo overhead and tile shape

For radius h and rectangular output tile A × B, the staged input region may be (A+2h) × (B+2h), giving overhead ratio:

\[
\text{halo ratio} = \frac{(A+2h)(B+2h)}{AB}
\]

This favors larger tiles, but larger tiles can spill registers, exceed cache, or reduce parallel-task count. On a row-major layout, increasing A and B changes memory behavior differently despite equal area.

### 14.4 Sliding reuse within a vector

Adjacent outputs often require shifted views of nearby input values:

~~~text
input_strip = LOAD_NEIGHBORHOOD_STRIP(output_tile)

FOR offset IN H
    shifted = SHIFT_OR_REINDEX(input_strip, offset)
    acc = COMBINE(acc, VECTOR_EVALUATE(shifted))
~~~

This may replace repeated memory loads with lane rearrangements. Whether it helps depends on the relative cost of register communication versus cached loads. Cross-vector boundaries complicate sliding windows.

### 14.5 Ordered neighborhood dependencies

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

### 14.6 Tile wavefronts

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

### 14.7 Space-time tiling

For iterative computations that update a spatial domain over time, retaining a patch through several time steps can reduce memory traffic. But the required halo grows as information propagates between steps.

~~~text
FOR space_time_tile IN LEGAL_TILED_SCHEDULE
    STAGE_REQUIRED_INITIAL_STATE_AND_HALO(space_time_tile)
    FOR local_step IN CAUSAL_ORDER(space_time_tile)
        UPDATE_ONLY_POINTS_WITH_READY_PREDECESSORS(local_step)
    COMMIT_BOUNDARY_AND_FINAL_STATE(space_time_tile)
~~~

A space-time tile is valid only if it contains, or receives, all predecessors. It can reduce repeated loads but increase redundant halo work and scratch footprint.

### 14.8 Choosing the outermost patch

Compare:

- **Output patch first:** reuse local output accumulators and neighboring inputs.
- **Input patch first:** reuse a staged source region across output groups.
- **Time tile first:** reuse evolving local state across steps.
- **Task patch first:** avoid cross-task writes and schedule parallel work.

These are distinct architectures even if the innermost arithmetic is identical.

**Takeaway:** the correct outermost tile is a **region of reusable data and legal dependencies**, not necessarily an original loop axis.

---

## 15. Search, selection, ordering, and comparison networks


Search and selection often contain branches and data-dependent results. They can still be reorganized by evaluating candidates in parallel and reducing the decision afterward.

### 15.1 Independent predicate evaluation

~~~text
FOR item IN ITEMS
    is_match[item] = PREDICATE(item)
~~~

A SIMD map evaluates the predicate for several items and stores a Boolean mask. Whether to materialize that mask depends on what follows.

### 15.2 Find the first satisfying item

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

### 15.3 Best candidate among many alternatives

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

### 15.4 Filter and compact

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

### 15.5 Comparison networks

A comparison network replaces data-dependent branches with a predetermined sequence of compare-and-exchange stages:

~~~text
FOR stage IN NETWORK_STAGES
    pairs = PREDEFINED_DISJOINT_PAIRS(stage)
    (lower, upper) = COMPARE_EXCHANGE(values, pairs, ORDER_RULE)
    values = ROUTE_TO_DESTINATIONS(lower, upper, stage)
~~~

Fixed networks can be effective for small sets where predictable control and vector lanes matter more than minimizing comparisons. Their costs include lane permutations, unused comparisons, and data-movement stages.

### 15.6 Partition-and-select versus full ordering

If the output only needs a minimum, maximum, top-k set, or threshold membership, fully sorting everything may be unnecessary. Prefer the weakest operation satisfying the contract:

| Requested output | Candidate organization |
| --- | --- |
| Single best | Vector local extrema + global reduction |
| First match | Ordered tiles + first active lane |
| All passing items | Predicate + stable compaction |
| Small top-k | Tile-local candidate structures + merge |
| Full ordering | Sorting/merging networks or partition strategy |

### 15.7 Divergence-aware search

For multiple independent searches with different lengths, map one search to each lane. For one search across many candidates, map candidates to lanes. The first wins when searches are plentiful and work lengths similar; the second when each search is expensive and associative selection is cheap.

**Takeaway:** the SIMD-friendly architecture often evaluates **more candidates at once** but postpones the scalar decision until a small result summary is available.

---

## 16. Segmented, sparse, and indirect algorithms


Sparse and segmented algorithms combine variable-length work, indirect input addresses, and potentially colliding updates. Their best architecture is usually chosen from **metadata about work shape** before vector arithmetic begins.

### 16.1 Three general representations

1. **Destination-centric:** each output owns a segment of contributions.
2. **Contribution-centric:** a flat list of (destination, input) relationships.
3. **Bucketed or blocked:** relationships reorganized by length, destination, or input locality.

These representations expose different loop nests even when they encode the same mathematical relation.

### 16.2 Destination-centric: reduce each segment

~~~text
FOR destination IN DESTINATIONS
    acc = IDENTITY
    FOR edge IN SEGMENT(destination)
        acc = COMBINE(acc, EVALUATE(edge))
    result[destination] = FINALIZE(acc)
~~~

This is naturally output-stationary. Map several destinations to lanes when segments are short and similar; map contributions to lanes and horizontally reduce for very long segments.

### 16.3 Contribution-centric: traverse every relationship

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

FOR target IN GROUPED_TARGETS
    partial = IDENTITY
    FOR edge IN grouped[target]
        partial = COMBINE(partial, EVALUATE(edge))
    COMMIT(target, partial)
~~~

If COMBINE is nonassociative, preserve the contribution order within each target. Grouping has preprocessing and memory costs.

### 16.4 Segmented vector reduction

A flattened relationship list can carry flags marking the starts of segments:

~~~text
values = EVALUATE_RELATIONSHIP_TILES()
segment_start = LOAD_SEGMENT_START_FLAGS()

partial_segment_sums =
    SEGMENTED_SCAN(values, segment_start, COMBINE)

WRITE_FINAL_VALUE_AT_SEGMENT_ENDS(partial_segment_sums)
~~~

Segmented scans require carefully defined handling for segments crossing vector and block boundaries. They may incur many lane operations when segments are tiny.

### 16.5 Three-way dispatch by work length

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

### 16.6 Input-centric grouping

If many relationships read the same source, grouping by source rather than destination can reduce gather traffic:

~~~text
FOR source_group IN GROUP_RELATIONSHIPS_BY_SOURCE
    shared_value = LOAD_SOURCE(source_group.source)
    FOR relationship_tile IN source_group.relationships
        contributions = VECTOR_EVALUATE(shared_value, relationship_tile)
        RESOLVE_AND_APPLY_OUTPUT_UPDATES(contributions)
~~~

This is input-stationary, but output collisions may now dominate. The transformation is useful only if saved source traffic exceeds extra output-update cost and grouping overhead.

### 16.7 Indirection chains

A lookup may depend on the result of the previous lookup:

~~~text
position = START
WHILE NOT DONE(position)
    position = NEXT(position)
~~~

A single chain has a true dependency, so a vector spanning successive chain steps is generally not useful. When many independent chains exist, assign a chain to each lane:

~~~text
states = LOAD_STARTS(chain_group)
active = VALID_LANES(chain_group) AND NOT DONE(states)

WHILE ANY(active)
    next_states = SAFE_VECTOR_FOLLOW(states, active)
    states = SELECT(active, next_states, states)
    active = UPDATE_COMPLETION_MASK(states, active)
~~~

Long-tail chains can leave lanes idle; compaction or work queues can reassign completed lanes.

### 16.8 Operational cautions

- Indirect addresses must be validated before loads.
- Concurrent updates require explicit conflict resolution.
- Grouping may reorder operations and change results when order matters.
- Metadata preprocessing should be included in performance measurements.
- Sparse workloads can be limited by address latency rather than vector throughput.

**Takeaway:** for sparse SIMD, optimize the **shape and order of relationships** as aggressively as the computations on them.

---

## 17. Stateful, variable-length, and convergent algorithms


A task that evolves its own state is often sequential *within the task* but independent of other tasks. The important transformation is to vectorize the **population of state machines**, not to parallelize illegal state transitions.

### 17.1 Reference semantics: one independent state per task

~~~text
FOR task IN TASKS
    state = INITIAL_STATE(task)
    step = 0
    WHILE NOT TERMINAL(state, task, step)
        state = TRANSITION(state, INPUT(task, step))
        step += 1
    output[task] = FINALIZE(state)
~~~

The step dependency is real: step k+1 consumes state from k. But different tasks may be independent if they do not modify shared state.

### 17.2 Task lanes and lockstep execution

~~~text
FOR task_group IN TILES(TASKS, V)
    state = VECTOR_INITIAL_STATE(task_group)
    step  = ZERO_VECTOR
    # Some tasks may already be terminal at their initial state.
    valid = VALID_LANES(task_group)
    active = valid AND NOT MASKED_VECTOR_TERMINAL(
        state, task_group, step, valid)

    WHILE ANY(active)
        next_input = SAFE_LOAD_NEXT_INPUT(task_group, step, active)
        proposed = VECTOR_TRANSITION(state, next_input, active)
        state = SELECT(active, proposed, state)
        step = MASKED_INCREMENT(step, active)
        active = active AND NOT VECTOR_TERMINAL(state, task_group, step)

    STORE_FINALIZED_STATES(task_group, state)
~~~

Every lane retains the state of one task. The inner iterations remain dependent, so peak vector arithmetic is limited by latency of TRANSITION and the number of independent task lanes.

### 17.3 The long-tail problem

Suppose task lengths are [2, 2, 2, 40]. A vector with four lanes performs 40 lockstep iterations even though only one lane is active after iteration 2. The useful-lane fraction becomes very low.

Options:

- **Group by expected length:** sort or bucket tasks so durations are similar.
- **Dynamic compaction:** move active states together after lanes finish.
- **Work stealing at lane granularity:** refill completed lanes from a task queue.
- **Multiple vectors in flight:** improve latency hiding while waiting for memory-dependent transitions.

All four increase bookkeeping. The best strategy depends on state size, transition cost, and whether work length can be predicted.

### 17.4 Dynamic refill

~~~text
queue = UNSTARTED_TASKS
group = FILL_LANES_FROM_QUEUE(queue)
state = INITIALIZE_LANES(group)
active = VALID_LANES(group)

WHILE ANY(active) OR NOT EMPTY(queue)
    # A newly initialized task may be terminal without taking a step.
    finished = active AND TERMINAL(state)
    WRITE_FINISHED_RESULTS(group, state, finished)
    active = active AND NOT finished

    REPLACE_EMPTY_LANES_FROM_QUEUE(
        group, state, active, queue)

    # Repeat initial-termination handling for any just-refilled lanes.
    finished = active AND TERMINAL(state)
    WRITE_FINISHED_RESULTS(group, state, finished)
    active = active AND NOT finished

    IF ANY(active)
        STEP_ACTIVE_LANES(group, state, active)

# A concrete implementation must repeatedly retire zero-step tasks until
# a lane holds nonterminal work or the queue is empty, including before
# any first transition. No terminal or inactive lane may advance.
~~~

This resembles vectorized discrete-event or stream processing. The logical order of task completion changes, so preserve task IDs and commit to the original output positions. Do not use this when interactions among tasks require a global chronological order.

### 17.5 Convergence and fixed points

A task may terminate when an error/residual falls below a threshold:

~~~text
WHILE NOT CONVERGED(state)
    state = ITERATE(state)
~~~

Lockstep vector execution is straightforward with masks, but numerical convergence may differ if the vectorized ITERATE changes operation order. Per-lane stopping preserves each task's semantics only if masked lanes stop receiving updates.

### 17.6 Batched state with common setup

Tasks may share immutable parameters:

~~~text
FOR batch IN GROUP_BY_SHARED_PARAMETER(TASKS)
    common = LOAD_SHARED_PARAMETER(batch)
    states = INITIALIZE_TASK_STATES(batch)

    WHILE ANY_ACTIVE(states)
        states = VECTOR_STEP(states, common)
~~~

The batch can reduce parameter and lookup overhead. If groups are tiny, it may reduce occupancy; prefer a hybrid grouping that balances common setup against active lanes.

### 17.7 State representation matters

A state with many fields can be stored as records, field-wise arrays, or blocks of records. Field-wise storage often suits many task lanes, but repeated per-task control decisions may favor packed task records. Consider the full set of fields touched by each transition, not just the most frequently updated one.

### 17.8 When can time steps be vectorized?

Only after proving stronger properties, such as:

- Transitions compose associatively into a compact representation.
- Dependencies split into independent chains.
- A wavefront exposes ready steps.
- A speculative schedule can validate and repair violations within the required semantics.

Without such a proof, vectorizing across ordered steps changes the algorithm.

### 17.9 Correctness and performance tests

Test zero-step tasks, immediate termination, maximum-length tasks, mixed lengths, compaction/refill boundaries, task identity restoration, and deterministic stopping rules. Report the distribution of active lanes over time, not just total runtime.

**Takeaway:** evolving-state algorithms become SIMD-friendly by **interleaving independent histories** while preserving each history's dependencies.

---

## 18. Multi-stage pipelines, fusion, and recomputation


Many algorithms are pipelines: data passes through several transformations before yielding a result. SIMD architecture must choose whether to **fuse stages**, **materialize intermediates**, **recompute values**, or **change lane mapping between stages**.

### 18.1 An abstract two-stage pipeline

~~~text
FOR item
    intermediate[item] = A(input[item])

FOR item
    output[item] = B(intermediate[item])
~~~

A fused schedule is:

~~~text
FOR tile IN TILES(ITEMS, V)
    values = VECTOR_LOAD(input, tile)
    temp = VECTOR_A(values)
    results = VECTOR_B(temp)
    VECTOR_STORE(output, tile, results)
~~~

Fusion removes intermediate writes and reads, but the combined stage has more live temporaries. If A and B prefer incompatible layouts or lane maps, materializing may still be faster.

### 18.2 Loop fusion can change the optimal tile

Suppose A has substantial local input reuse and B operates independently on completed outputs. A may prefer a large cache tile while B prefers full-width contiguous output vectors.

Three architectures:

**Separate passes:** optimize each stage independently, paying intermediate memory traffic.

**Full fusion:** keep each small intermediate vector in registers, possibly sacrificing A's larger input tile.

**Tile-level fusion:** compute a cache tile for A, then immediately run B on that tile before evicting it.

~~~text
FOR cache_tile IN TILES(DOMAIN, T_C)
    temp_tile = EXECUTE_A_FOR_TILE(cache_tile)
    EXECUTE_B_FOR_TILE(temp_tile)
~~~

The third balances fast-cache reuse and per-stage vectorization.

### 18.3 Recompute versus materialize

Sometimes an intermediate is inexpensive but costly to store, reload, or rearrange.

~~~text
# Materialize once
FOR item
    temp[item] = SMALL_TRANSFORM(input[item])
FOR consumer
    FOR item
        USE(consumer, temp[item])

# Recompute per consumer tile
FOR consumer_tile
    FOR item_tile
        temp_local = SMALL_TRANSFORM(input[item_tile])
        USE(consumer_tile, temp_local)
~~~

Recomputation is beneficial only if extra arithmetic is cheaper than saved data movement and it does not disrupt other reuse. It can also be required to avoid expensive cross-lane communication.

### 18.4 A simple break-even inequality

Let **C_recompute** be added arithmetic cost, **C_saved_memory** avoided loads/stores, **C_saved_shuffle** avoided rearrangement, and **C_extra_pressure** added register/spill cost. Recompute is favorable when approximately:

\[
C_{\text{recompute}} + C_{\text{extra pressure}}
<
C_{\text{saved memory}} + C_{\text{saved shuffle}}
\]

These terms are workload- and machine-dependent. The inequality is a guide for experiments, not an exact cost model.

### 18.5 Producer-consumer tiling

Two stages may traverse different iteration spaces:

~~~text
FOR intermediate_index
    temp[intermediate_index] = PRODUCE(inputs)

FOR output_index
    result[output_index] =
        CONSUME(SUBSET_OF(temp, output_index))
~~~

A tile of outputs needs some subset of intermediate values. Compute only those intermediates and retain them for the consumer tile:

~~~text
FOR output_tile
    required_intermediates = DEPENDENCY_CLOSURE(output_tile)
    temp_local = PRODUCE(required_intermediates)
    CONSUME_AND_COMMIT(output_tile, temp_local)
~~~

Overlapping output tiles may recompute some intermediates. Whether this wins depends on tile overlap, producer cost, and cache footprint.

### 18.6 Pipeline stages with different lane mappings

Suppose stage A benefits from lanes representing **independent tasks**, while stage B benefits from lanes representing **output coordinates**.

~~~text
stage_A_tile = VECTOR_PRODUCE(task_lanes)
repacked = CHANGE_LANE_LAYOUT(stage_A_tile, output_lane_layout)
VECTOR_CONSUME(repacked)
~~~

Alternatives are repacking in registers, writing and reloading a transformed intermediate layout, or running each stage with its own mapping. Evaluate total cost including conversion.

### 18.7 Fusion legality

Fusion is legal when it preserves observable ordering and every consumed intermediate is produced before use. If A produces data that multiple later B iterations share, naïve fusion may duplicate work or change side-effect order. In-place processing also requires alias analysis to avoid overwriting inputs needed later.

### 18.8 When fission is better

Separate stages if fusion creates too many live values, excessive branching, incompatible lane maps, or instruction-cache pressure. A pipeline split can also enable specialized kernels and clearer parallel scheduling.

### 18.9 Full-pipeline benchmarking

Time **packing + producer + intermediate handling + consumer + unpacking**. A kernel-level win can be outweighed by layout conversions. Benchmark realistic reuse counts and input sizes, not only a single hot loop.

**Takeaway:** the optimal SIMD architecture can require **more arithmetic but less communication**, or **more passes but less register pressure**.

---

## 19. Numerical correctness and semantic equivalence


Performance claims matter only after the transformed algorithm satisfies its **semantic contract**. Some transformations preserve exact results; others are valid only under an explicit tolerance policy.

### 19.1 Define the equivalence level

| Equivalence class | Required property |
| --- | --- |
| Bit-exact | Every observable output bit matches |
| Value-exact | Mathematically equivalent under specified arithmetic |
| Deterministic | Same results across repeated executions |
| Tolerance-bounded | Error stays within stated absolute/relative bound |
| Decision-equivalent | Same ordering, classifications, or branch decisions |
| Statistical | Distributional properties match a specified test |

These categories are not interchangeable. An algorithm may be mathematically exact yet not bit-exact under floating-point reordering.

### 19.2 Reassociation and floating-point behavior

~~~text
# Sequential reference
acc = 0
FOR i IN ORIGINAL_ORDER
    acc = FLOAT_ADD(acc, value[i])

# Parallel regrouping
parts = PARTITION(value, V)
partial = FLOAT_REDUCE_EACH(parts)
acc = FLOAT_REDUCE(partial)
~~~

Floating-point addition generally is not associative due to rounding, signed zeros, infinities, and NaNs. If an application requires bit-exact equality, preserve the reference order or use a separately specified reproducible reduction mechanism. If approximate equality is allowed, define and test error bounds.

### 19.3 Integer overflow and saturation

A narrower accumulator can improve throughput but only if its range is safe or if wrapping/saturation semantics are intentionally preserved.

For nonnegative contributions bounded by C and a narrow accumulator with largest value M, a safe chunk size is at most floor(M/C). This formula assumes zero initial accumulator and no negative contributions.

~~~text
wide_total = 0
FOR chunk IN SAFE_CHUNKS(contributions)
    narrow = ZERO
    FOR value IN chunk
        narrow = EXACT_NARROW_ADD_WITHOUT_OVERFLOW(narrow, value)
    wide_total = WIDE_ADD(wide_total, WIDEN(narrow))
~~~

For saturating addition, clipping each step is not generally equivalent to combining large partials then clipping once. Verify the original saturation points.

### 19.4 Tie breaking and stable selection

Parallel minimum/maximum selection must define which index wins when values are equal. A stable tie rule might select the smallest original index.

~~~text
FUNCTION BETTER(candidate, current)
    IF candidate.value < current.value
        RETURN TRUE
    IF candidate.value > current.value
        RETURN FALSE
    RETURN candidate.original_index < current.original_index
~~~

Floating-point NaNs require an explicit comparison policy. Do not presume a naive pairwise minimum implements the reference.

### 19.5 Masks, bounds, and tails

A masked-off lane must not access invalid memory unless the memory operation guarantees nonaccess for inactive lanes. A masked reduction lane must contribute the appropriate **identity**. Test smallest sizes, zero-length segments, sizes just below/above vector width, and every remainder class.

### 19.6 Aliasing and in-place updates

A legal out-of-place vector schedule can be wrong in place:

~~~text
FOR i IN ORIGINAL_ORDER
    data[i] = F(data[i], data[i-1])
~~~

Reordering iterations may overwrite an input needed by a later iteration. Options include snapshot buffers, wavefront/ordered scheduling, or proving a safe direction. The existence of apparently independent output positions is insufficient.

### 19.7 Parallel races and update conflicts

Two threads may update the same destination even when their loop indices differ. Scratch storage must be private or synchronized. Deterministic results may require fixed merge order, and nonassociative reductions cannot be arbitrarily merged.

### 19.8 Differential testing protocol

~~~text
FOR seed IN DIVERSE_SEEDS
    FOR shape IN TEST_SHAPES
        input = GENERATE_INPUT(seed, shape)

        reference = EXECUTE_REFERENCE(input)
        FOR architecture IN VALID_ARCHITECTURES
            actual = EXECUTE_ARCHITECTURE(architecture, input)
            ASSERT_CONTRACT(reference, actual, EQUIVALENCE_POLICY)
~~~

Include adversarial cases: zeros, maxima/minima, alternating extremes, duplicate indices, overlapping writes, invalid and boundary indices, empty tasks, mixed task lengths, and degenerate tiles.

### 19.9 Metamorphic properties

When an independent oracle is unavailable, validate properties that must hold under legal transformations: partitioning the input into tiles should not change a mathematically associative result; output permutation followed by inverse permutation should recover the same set of independent results; changing task group size should preserve per-task results.

These tests supplement but do not replace comparison to a trusted reference.

### 19.10 Proof obligations for every optimization

1. Define logical work set and every required read/write.
2. Show all work is included (no duplicates or omissions).
3. Show all true dependencies are respected.
4. Establish arithmetic equivalence or explicit tolerated differences.
5. Prove tail and boundary memory safety.
6. Preserve output placement and tie/termination rules.
7. Test against an independent oracle under edge workloads.

**Takeaway:** correctness is a property of the **schedule plus arithmetic semantics**, not simply its final-looking formula.

---

## 20. Performance and communication models


A performance model is not a magic formula for the fastest loop order. Its purpose is to **rule out implausible architectures**, identify bottlenecks, and decide what to measure.

### 20.1 Decompose elapsed time

A useful accounting is:

\[
T_{\text{total}}
\approx T_{\text{setup}} + T_{\text{packing}} + T_{\text{kernel}}
+ T_{\text{finalization}} + T_{\text{unpacking}} + T_{\text{scheduling}}
\]

These are *measured phase costs* when separated; execution inside a kernel often overlaps computation, memory, and control, so their microarchitectural costs cannot generally be added without double-counting.

### 20.2 Effective vector utilization

Define:

\[
U_{\text{lane}} =
\frac{\text{useful logical lane-operations}}
     {\text{total issued logical lane-operations}}
\]

A vector architecture with V lanes and only half its lanes active often provides less than half the nominal arithmetic throughput, even before masking and branching overhead. More formally, compare **useful results per unit time**, not vector instructions per second.

### 20.3 Arithmetic intensity and the roofline idea

Let **F** represent useful compute operations and **B** the transferred bytes at a specified memory-hierarchy level. Arithmetic intensity is:

\[
I = F/B
\]

An upper bound on achievable compute rate at that level is conceptually:

\[
P \leq \min(P_{\text{compute peak}}, I \cdot BW_{\text{level}})
\]

This is a bound under stated assumptions, not a runtime prediction. Cache hit rates change B at each level, and irregular accesses may be latency-bound without saturating bandwidth.

### 20.4 Output-stationary versus input-stationary traffic

Suppose each of N_O output tiles consumes N_R input tiles.

- Output-stationary may read shared inputs repeatedly for each output tile but write each completed output once.
- Input-stationary may read a shared input tile once but repeatedly load/store output partials.

Build a **reuse table**:

| Resource | Unique bytes | Uses per tile | Repeated transfers |
| --- | ---: | ---: | ---: |
| Input tile | B_in | N_O | Depends on retention |
| Output partials | B_out | N_R | Depends on accumulator residency |
| Indices/masks | B_idx | Per logical vector | Depends on precomputation |
| Conversion buffers | B_conv | Once or every pass | Depends on pipeline structure |

Measure or estimate *actual* cache-line traffic rather than equating repeated logical loads with repeated DRAM transfers.

### 20.5 Register spills

If a microtile requires more live state than the allocator can keep in registers, spills create hidden stores/loads and lengthen dependency chains. A larger vector width can increase register demand even when the number of logical operations decreases.

A rough model:

~~~text
required_live_state =
    accumulator_vectors
  + operand_vectors
  + intermediate_vectors
  + masks_and_indices
  + latency_hiding_vectors
~~~

Compare this estimate to the target's usable register resources, leaving margin for setup and code generation.

### 20.6 Dependency latency versus throughput

Consider an accumulation:

~~~text
acc = IDENTITY
FOR r
    acc = COMBINE(acc, EVALUATE(r))
~~~

If each update consumes the previous one, critical-path latency may limit throughput. Multiple independent chains can hide that latency:

~~~text
acc0, acc1, ... = IDENTITIES
FOR groups_of_contributions
    UPDATE_INDEPENDENT_ACCUMULATORS(acc0, acc1, ...)
COMBINE_FINAL_PARTIALS()
~~~

Too many chains increase register pressure and merge cost. The fastest number is not dictated by vector width alone.

### 20.7 Address-generation cost

Even when memory data is cached, complex address calculations or long pointer chains can be slow. For each candidate, count:

- dependent address computations;
- contiguous versus indirect accesses;
- number of independent outstanding requests;
- index conversion and permutation work;
- cache-line and page-crossing frequency.

### 20.8 A multidimensional cost hypothesis

An architecture candidate **A** can be evaluated using:

\[
C(A) = (M,\ L,\ R,\ X,\ D,\ G,\ S)
\]

where:

- **M**: estimated memory traffic;
- **L**: lane utilization;
- **R**: register demand;
- **X**: cross-lane exchange;
- **D**: dependency critical path;
- **G**: gather/scatter and address-generation cost;
- **S**: setup, packing, and scheduling overhead.

These are not naturally comparable units. Use the vector to characterize tradeoffs and decide experiments, not to manufacture a single score without calibration.

### 20.9 Bottleneck-driven optimization

| Evidence | Candidate changes |
| --- | --- |
| High transferred bytes, low arithmetic utilization | Improve locality/tiling, fuse stages |
| Long dependency chain | Increase independent accumulators/tasks |
| Low useful-lane fraction | Bucket, compact, change lane mapping |
| Many lane permutations | Change layout/lane mapping, recompute |
| Register spills | Smaller output tile, fewer live chains, fission |
| Scatter conflicts | Group by destination, privatize |
| Setup dominates small inputs | Use simpler path, reduce packing |
| Poor scaling with threads | Adjust task granularity, reduce false sharing |

A change should target a measured bottleneck; indiscriminate tiling often adds cost.

**Takeaway:** model *the costs introduced by the schedule*, then validate the important hypotheses empirically.

---

## 21. Systematic architecture search and measurement


Efficient SIMD architecture is a constrained search problem. The constraints are correctness, dependencies, memory safety, and machine resources. Performance is an empirical objective.

### 21.1 Define the architecture tuple

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

### 21.2 Stage 1: semantic and dependency model

Write the reference pseudocode and classify every axis: problem, output, reduction, ordered state, or conditional/irregular dimension. Identify which axes commute and what output equality means.

Do not change source layout or execution order until the reference behavior is precisely defined.

### 21.3 Stage 2: generate *families*, not blind permutations

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

### 21.4 Stage 3: resource feasibility

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

### 21.5 Stage 4: correctness-first implementation

For each candidate, compare to a trusted reference on diverse randomized and adversarial cases. Include incomplete tiles, extreme values, repeated indices, boundary data, empty segments, and minimum/maximum task lengths.

Never accept a performance win that changes the contract unintentionally.

### 21.6 Stage 5: benchmark protocol

1. Use the same data for competing architectures.
2. Warm up the relevant code and inputs before the measured section.
3. Randomize or rotate candidate execution order to avoid systematic drift.
4. Record repetitions across fresh runs, not only many iterations in one process.
5. Record machine, compiler/code-generator configuration, workload, input distribution, and parallel worker count.
6. Report **median**, distribution/spread, and any outliers or distinct timing regimes.
7. Compare the end-to-end pipeline when packing, layout conversion, or scheduling changes.
8. Profile likely bottlenecks when one architecture surprises the model.

Large speedups should be treated as measurements under a specific workload, not as portable laws.

### 21.7 Control hidden variables

Potential confounders: CPU frequency state, thermal behavior, thread migration, background activity, cache residency, allocation layout, dynamic dispatch, and benchmark instrumentation overhead.

Use equivalent input sets and avoid interpreting a single anomalous timing point as an algorithmic property. Re-running with interleaved candidate order helps, but cannot eliminate every environmental effect.

### 21.8 Change one dimension at a time, then explore interactions

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

### 21.9 Architecture comparison worksheet

| Candidate | Lane axis | Outermost tile | Live partials | Input access | Cross-lane work | Expected risk |
| --- | --- | --- | --- | --- | --- | --- |
| A | outputs | output tile | high/medium | repeated | low | spills |
| B | contributions | reduction tile | few | contiguous | high | horizontal fold |
| C | tasks | task tile | per task | potentially gathered | low | divergence |
| D | outputs | input tile | repeatedly updated | high reuse | low | output traffic |

Fill this with workload-specific values rather than generic labels before benchmarking.

### 21.10 When to stop

Stop tuning when gains are smaller than measurement uncertainty, a more complex schedule harms maintainability for negligible benefit, or remaining runtime is dominated by an unaffected component. Document why the chosen architecture wins, and the workload shapes on which it may lose.

**Takeaway:** architecture selection is an experimental science with correctness constraints—not a rule that the widest vectors or deepest tiles always win.

---

## 22. Portable vectors and parallel hierarchy


A hardware-agnostic handbook should still teach where hardware constraints enter the design. Express those constraints as parameters, not named intrinsics or instruction-set assumptions.

### 22.1 Logical width versus physical width

Let **V** be the number of logical elements processed as one vector group. On a particular target, the implementation may use multiple physical vector operations, a scalable vector size, or even scalar operations.

~~~text
V = QUERY_PREFERRED_VECTOR_GROUP_SIZE(element_type, workload)

FOR tile IN TILES(items, V)
    active = VALID_LANES(tile)
    VECTOR_PROCESS(tile, active)
~~~

The algorithm remains correct for any V ≥ 1. Performance still depends on element width, packing, register capacity, and how the target implements masks and rearrangements.

### 22.2 Width-agnostic loops

Avoid hardcoding that every input length is a multiple of some width:

~~~text
base = 0
WHILE base < N
    active = MIN(V, N - base)
    values = SAFE_LOAD_ACTIVE(input, base, active)
    result = VECTOR_EVALUATE(values, active)
    SAFE_STORE_ACTIVE(output, base, result, active)
    base += active
~~~

A width-agnostic schedule needs correct semantics for partial groups. It may still use target-specific specializations for performance behind the abstract interface.

### 22.3 Wider is not automatically faster

A wider vector may:

- process more independent items per operation;
- increase accumulator register demand;
- increase masked-lane waste for short tasks;
- require more costly lane rearrangements;
- change attainable core frequency or throughput on a given machine;
- expose different memory-alignment and cache-line behavior.

Measure the full architecture at different effective widths when feasible.

### 22.4 SIMD versus threads versus instruction-level parallelism

These are distinct axes:

- **SIMD:** several logical data lanes in one vector operation.
- **Instruction-level parallelism:** several independent operations in flight.
- **Task/thread parallelism:** separate workers execute independent work chunks.

A design can exploit all three:

~~~text
PARALLEL_FOR task_chunk IN TASK_CHUNKS
    LOCAL_STATE = INITIALIZE_WORKER_STATE()

    FOR output_tile IN OUTPUT_TILES(task_chunk)
        acc[INDEPENDENT_CHAINS] = VECTOR_IDENTITIES()

        FOR reduction_tile IN REDUCTION_TILES(output_tile)
            UPDATE_INDEPENDENT_VECTOR_CHAINS(
                acc, reduction_tile, LOCAL_STATE)

        STORE_FINALIZED(output_tile, MERGE_CHAINS(acc))
~~~

This does not imply the outer parallel task is always the right place. In some problems parallelizing reduction tiles is necessary because there are too few independent outputs.

### 22.5 Thread ownership and scratch storage

Disjoint logical outputs are the simplest way to avoid races. Allocate scratch per task or worker. When multiple tasks share a cache line, coherence traffic may reduce speed even without races; when they update the same output, a defined merge protocol is required.

### 22.6 Adapt architecture to workload regimes

A single dispatch policy may select among architecture families:

~~~text
IF NUMBER_OF_OUTPUTS is large AND REDUCTION_LENGTH is moderate
    architecture = OUTPUT_LANES
ELSE IF NUMBER_OF_OUTPUTS is small AND REDUCTION_LENGTH is large
    architecture = REDUCTION_LANES
ELSE IF TASK_LENGTHS_ARE_HIGHLY_VARIABLE
    architecture = BUCKET_OR_COMPACT
ELSE
    architecture = SIMPLE_BASELINE

EXECUTE_VALIDATED_ARCHITECTURE(architecture)
~~~

These are *illustrative heuristics*, not performance guarantees. The threshold choices and feature estimates must be grounded in measurements.

### 22.7 Portable cost-model vocabulary

Describe each architecture using:

- active logical lanes;
- number of independent accumulator chains;
- unique input/output bytes per tile;
- estimated cache-line requests;
- cross-lane reductions and permutations;
- sequential dependency depth;
- grouping/packing setup;
- thread ownership and scratch footprint.

These metrics can be compared conceptually across targets without relying on a specific machine model.

### 22.8 Portability is semantic first, performance second

An algorithm can be **functionally portable** but optimized differently for different targets. Preserve one semantic contract and several validated schedules. Avoid baking assumptions about preferred widths, subgroup boundaries, or instruction costs into the algorithm itself.

**Takeaway:** portable SIMD architecture separates **what is computed**, **how the work is scheduled**, and **how a target realizes each abstract vector operation**.

---

## 23. Fifteen abstract before/after case studies


This chapter deliberately uses **generic operations** rather than algorithms named after a domain. The goal is to train an architectural reflex: recognize the dependency structure, choose the owners of live data, transform the schedule, and identify the condition under which the transformation would *lose*.

Every case has: (1) original computation, (2) vector-oriented candidate, (3) why it may win, (4) what can make it worse, and (5) a legality assumption. A candidate is a hypothesis, not a claim that it is the fastest implementation.

---

### Case 1 — Independent element map: one logical loop

**Problem:** each input element produces exactly one independent output.

~~~text
# Reference
FOR i IN 0..N-1
    output[i] = F(input[i])

# Vector candidate: consecutive items become lanes
FOR tile IN TILES(0..N-1, V)
    active = VALID_LANES(tile)
    x = SAFE_LOAD(input, tile, active)
    y = VECTOR_F(x)
    SAFE_STORE(output, tile, y, active)
~~~

**Why it may win:** adjacent memory accesses and lane-wise arithmetic. **Why it may lose:** tiny N, expensive setup, or a scalar fast path with lower latency. **Legal if:** F has independent side-effect-free element semantics and output does not overwrite an input needed by another item.

**Design question:** Is a second outer cache-tile loop useful, or does it merely add overhead?

### Case 2 — Two-dimensional independent map: collapse versus nested lanes

**Problem:** all (a,b) pairs are independent, but the b dimension is short.

~~~text
# Reference
FOR a
    FOR b
        output[a,b] = F(input[a,b])

# Candidate: flatten the valid pair set
pairs = ENUMERATE_PAIRS_IN_STORAGE_ORDER(A, B)

FOR tile IN TILES(pairs, V)
    (a,b) = DECODE_PAIRS(tile)
    results = VECTOR_F(LOAD_PAIRS(input, tile))
    WRITE_PAIRS(output, tile, results)
~~~

**Why it may win:** fills vectors when B < V. **Why it may lose:** index decoding and noncontiguous pair access, especially if each row has padding. **Legal if:** all pair evaluations are independent and the pair enumeration is bijective.

**Alternative:** keep b inner and use several a rows to fill a vector without building a separate pair list.

### Case 3 — One long reduction: split the dependency chain

**Problem:** a single result depends on many contributions.

~~~text
# Reference
acc = IDENTITY
FOR r
    acc = COMBINE(acc, EVALUATE(r))
result = FINALIZE(acc)

# Candidate: lane-partial reduction
partials = VECTOR_IDENTITY(V)
FOR rtile IN TILES(REDUCTIONS, V)
    values = VECTOR_EVALUATE(rtile)
    partials = VECTOR_COMBINE(partials, values)
result = FINALIZE(HORIZONTAL_REDUCE(partials))
~~~

**Why it may win:** input lanes are contiguous and partial chains can execute independently. **Why it may lose:** final cross-lane reduction and different numerical order. **Legal if:** COMBINE is associative under the required result contract, or the changed order is explicitly tolerated.

**Alternative:** use multiple independent scalar/vector accumulators while keeping a predetermined merge order.

### Case 4 — Many outputs × one reduction: keep output lanes stationary

**Problem:** many output cells repeatedly accumulate contributions.

~~~text
# Reference
FOR o
    acc = IDENTITY
    FOR r
        acc = COMBINE(acc, EVALUATE(o,r))
    output[o] = FINALIZE(acc)

# Candidate: output tile owns live accumulators
FOR otile IN TILES(OUTPUTS, T_O)
    acc = VECTOR_IDENTITY(otile)
    FOR r
        acc = VECTOR_COMBINE(acc, VECTOR_EVALUATE(otile,r))
    STORE_FINALIZED(otile, acc)
~~~

**Why it may win:** one initialization and one store per output, no horizontal fold. **Why it may lose:** rereads shared inputs for each otile, or spills if T_O is too large. **Legal if:** different output accumulators are independent and each per-output reduction retains the required arithmetic semantics.

**Alternative:** input-stationary traversal when the same input is reused across many otile values.

### Case 5 — Three axes P × O × R: outermost-tile decision

**Problem:** independent problems contain many outputs; each output reduces contributions.

~~~text
# Reference: P → O → R
FOR p
    FOR o
        a = IDENTITY
        FOR r
            a = COMBINE(a, EVALUATE(p,o,r))
        output[p,o] = FINALIZE(a)

# Candidate: P tile → O tile → R tile
FOR ptile IN TILES(PROBLEMS, T_P)
    FOR otile IN TILES(OUTPUTS(ptile), T_O)
        a = VECTOR_IDENTITY(ptile, otile)
        FOR rtile IN TILES(REDUCTIONS(ptile,otile), T_R)
            a = FOLD_TILE(a, ptile, otile, rtile)
        COMMIT_FINAL(ptile, otile, a)
~~~

**Why it may win:** amortizes task setup and keeps output partials local. **Why it may lose:** too many simultaneous (p,o) accumulators or divergent reduction bounds. **Legal if:** output independence and per-output reduction semantics are preserved.

**Compare another candidate:** R tile → P tile → O tile, staging shared input once but repeatedly updating partial outputs. Which is faster depends on the input-to-output reuse ratio and storage hierarchy.

### Case 6 — Four axes with a shared inner dimension: strip mine, then jam

**Problem:** two independent outer axes (a,b), one output axis o, and one reduction r.

~~~text
# Reference
FOR a
    FOR b
        FOR o
            acc = IDENTITY
            FOR r
                acc = COMBINE(acc, EVALUATE(a,b,o,r))
            output[a,b,o] = FINALIZE(acc)

# Candidate: flatten independent pairs, tile output, jam the reduction
work = VALID_TRIPLES_OF(a,b,o)
FOR work_tile IN TILES(work, T_O)
    acc = VECTOR_IDENTITY(work_tile)
    FOR r IN COMMON_REDUCTION_STEPS(work_tile)
        mask = CONTRIBUTION_EXISTS(work_tile,r)
        acc = MASKED_VECTOR_COMBINE(
            acc, VECTOR_EVALUATE(work_tile,r,mask), mask)
    COMMIT_FINAL(work_tile, acc)
~~~

**Why it may win:** packs small output ranges from multiple (a,b) pairs into full vectors. **Why it may lose:** masks and gathers when different tasks have different r bounds. **Legal if:** no cross-pair dependency exists and every valid (a,b,o,r) contribution occurs exactly once.

**Alternative:** retain a,b as outer task ownership and reduce indexing overhead by keeping each pair contiguous.

### Case 7 — Many outputs read overlapping input neighborhoods

**Problem:** adjacent results depend on overlapping, immutable input sets.

~~~text
# Reference
FOR x IN OUTPUTS
    a = IDENTITY
    FOR offset IN NEIGHBORHOOD
        a = COMBINE(a, EVALUATE(input[x+offset]))
    output[x] = FINALIZE(a)

# Candidate: process an output patch with a shared input halo
FOR patch IN OUTPUT_PATCHES(T)
    input_region = STAGE_INPUTS(REQUIRED_HALO(patch))
    FOR xtile IN TILES(patch, V)
        a = VECTOR_IDENTITY(xtile)
        FOR offset IN NEIGHBORHOOD
            a = VECTOR_COMBINE(
                a, VECTOR_EVALUATE(input_region, xtile, offset))
        COMMIT(xtile,a)
~~~

**Why it may win:** reuse of overlapping source values and regular output stores. **Why it may lose:** staging overhead, duplicated halo, or cross-lane shifts. **Legal if:** all reads observe an immutable input snapshot and boundaries are handled as specified.

**Alternative:** no staging; rely on cache hits and reduce code complexity.

### Case 8 — Prefix scan: transform one serial chain into local parallel work

**Problem:** all prefixes are output, not just the final reduction.

~~~text
# Reference
carry = IDENTITY
FOR i
    carry = COMBINE(carry, input[i])
    output[i] = carry

# Candidate: local vector scan plus carry
carry = IDENTITY
FOR block IN TILES(input, V)
    local = VECTOR_PREFIX_SCAN(block, COMBINE)
    output_block = COMBINE_EACH(carry, local)
    STORE_PREFIX_BLOCK(output_block)
    carry = COMBINE(carry, LAST_VALID(local))
~~~

**Why it may win:** O(log V) lane stages replace O(V) sequential stages within blocks; inputs are contiguous. **Why it may lose:** costly lane communication and sequential block carry. **Legal if:** associative combination with a valid identity; exact floating-point order may not be preserved.

**Alternative:** hierarchical multi-pass scan for very large data and ample parallelism.

### Case 9 — Independent recurrences: vectorize across streams

**Problem:** each task has an ordered state history.

~~~text
# Reference
FOR task
    state = INITIAL(task)
    FOR step IN STEPS(task)
        state = TRANSITION(state, input[task,step])
    output[task] = state

# Candidate: each lane owns a state
FOR group IN TILES(tasks, V)
    states = VECTOR_INITIAL(group)
    valid = VALID_LANES(group)
    active = valid AND NOT MASKED_VECTOR_TERMINAL(states, group, 0, valid)
    FOR step IN COMMON_STEP_DOMAIN(group)
        step_active = active AND STEP_EXISTS(group,step)
        states = MASKED_VECTOR_TRANSITION(states, group, step, step_active)
        active = step_active AND NOT MASKED_VECTOR_TERMINAL(
            states, group, step+1, step_active)
    STORE_STATES(group,states)
~~~

**Why it may win:** preserves dependency within each stream while executing independent streams together. **Why it may lose:** many inactive lanes when durations differ. **Legal if:** states do not interact and masked-off streams do not advance.

**Alternative:** bucket by step count or dynamically refill completed lanes.

### Case 10 — Two-dimensional dependency: compute wavefronts

**Problem:** each point uses completed results above and to the left.

~~~text
# Reference
FOR i IN ASCENDING_ORDER
    FOR j IN ASCENDING_ORDER
        output[i,j] = F(output[i-1,j], output[i,j-1], input[i,j])

# Candidate: diagonals of ready points
FOR wave IN INCREASING_SUMS(i,j)
    ready = VALID_POINTS_WHERE(i+j == wave)
    FOR tile IN TILES(ready,V)
        above = READ_ALREADY_COMPUTED_ABOVE(tile)
        left = READ_ALREADY_COMPUTED_LEFT(tile)
        VECTOR_WRITE(tile, VECTOR_F(above,left,INPUT_AT(tile)))
~~~

**Why it may win:** equal-diagonal points can be independent. **Why it may lose:** diagonal gathers, small boundary wavefronts, synchronization between waves. **Legal if:** dependencies are exactly the specified predecessors and all required boundary states are available.

**Alternative:** block wavefronts and vectorize independent work within a ready tile.

### Case 11 — Conditional transformation: branch to mask

**Problem:** per-item branching disrupts lockstep execution.

~~~text
# Reference
FOR i
    IF TEST(i)
        output[i] = A(i)
    ELSE
        output[i] = B(i)

# Candidate: predicated vector control
FOR tile IN TILES(items,V)
    mask = VECTOR_TEST(tile)
    a = SAFE_MASKED_A(tile,mask)
    b = SAFE_MASKED_B(tile,NOT mask)
    VECTOR_STORE(tile,SELECT(mask,a,b))
~~~

**Why it may win:** predictable vectorized work without independent control transfers per lane. **Why it may lose:** evaluating both expensive paths or costly masked operations. **Legal if:** both branches have compatible side-effect semantics and no masked-off path performs invalid memory access.

**Alternative:** split items by predicate into two compact groups, then execute each branch densely.

### Case 12 — Filter/compact: convert control-dependent writes into a prefix problem

**Problem:** keep only passing items, preserving their original order.

~~~text
# Reference
count = 0
FOR i IN ORIGINAL_ORDER
    IF KEEP(i)
        output[count] = input[i]
        count += 1

# Candidate: vector predicates plus stable compaction
count = 0
FOR tile IN ASCENDING_TILES(input,V)
    mask = VECTOR_KEEP(tile)
    selected = COMPRESS_IN_ORIGINAL_LANE_ORDER(tile,mask)
    STORE_CONTIGUOUS(output,count,selected)
    count += NUMBER_ACTIVE(mask)
~~~

**Why it may win:** contiguous output stores replace branch-dependent scalar writes. **Why it may lose:** compaction overhead when almost everything is kept or dropped; short inputs. **Legal if:** KEEP is pure and compaction is stable.

**Alternative:** one pass computes flags and counts, a second pass writes selected values at prefix-derived positions.

### Case 13 — Indirect scatter reduction: reorganize by destination

**Problem:** many contributions may update the same output.

~~~text
# Reference
FOR edge IN ORIGINAL_ORDER
    target = TARGET(edge)
    output[target] = COMBINE(output[target],VALUE(edge))

# Candidate: stable grouping by output destination
groups = STABLE_GROUP_BY_TARGET(edges)
FOR group IN groups
    target = TARGET(group)
    acc = LOAD_INITIAL_OUTPUT(output,target)
    FOR edge IN ORIGINAL_ORDER_WITHIN(group)
        acc = COMBINE(acc,VALUE(edge))
    STORE_FINAL_OUTPUT(output,target,acc)
~~~

**Why it may win:** fewer random output writes and a regular reduction loop per destination. **Why it may lose:** grouping/sorting cost and uneven groups. **Legal if:** cross-target order is unobservable, groups preserve per-target contribution order, and each group begins with that target's original initialized output. This retains the per-target sequence even for a nonassociative COMBINE; starting from IDENTITY and merging the group into a preexisting output once would instead require stronger algebraic assumptions.

**Alternative:** vectorize value preparation but apply writes sequentially when updates are nonassociative.

### Case 14 — Multi-stage pipeline: recompute instead of materialize

**Problem:** a small intermediate is shared by multiple consumers.

~~~text
# Reference
FOR i
    temp[i] = PRODUCE(input[i])
FOR consumer
    FOR i
        output[consumer,i] = USE(consumer,temp[i])

# Candidate: keep an intermediate tile local
FOR consumer_tile IN TILES(CONSUMERS,T_C)
    FOR input_tile IN TILES(INPUTS,T_I)
        local_temp = VECTOR_PRODUCE(input_tile)
        FOR consumer IN consumer_tile
            UPDATE_OUTPUT_WITH(local_temp, consumer)
~~~

**Why it may win:** avoids full intermediate storage or repeated cache misses; local_temp reused across consumer tile. **Why it may lose:** repeats PRODUCE across consumer tiles and creates more live output state. **Legal if:** PRODUCE is pure, outputs do not depend on materialization timing, and each consumer receives the same logical intermediate.

**Alternative:** materialize temp once, then use separate optimized consumer tiles.

### Case 15 — Six-dimensional work: compose hierarchical tiles, not 720 permutations

**Problem:** two task axes, two output axes, two reduction axes.

~~~text
# Reference
FOR p0
    FOR p1
        FOR o0
            FOR o1
                a = IDENTITY
                FOR r0
                    FOR r1
                        a = COMBINE(a, EVALUATE(p0,p1,o0,o1,r0,r1))
                output[p0,p1,o0,o1] = FINALIZE(a)

# Candidate: task, output, and reduction tiles
FOR task_tile IN TILES(P0 × P1,T_P)
    FOR output_tile IN TILES(O0 × O1,T_O)
        a = VECTOR_IDENTITY(task_tile,output_tile)
        FOR reduction_tile IN TILES(R0 × R1,T_R)
            a = FOLD_VALID_CONTRIBUTIONS(
                a,task_tile,output_tile,reduction_tile)
        COMMIT_FINAL(task_tile,output_tile,a)
~~~

**Why it may win:** avoids exploring arbitrary order permutations, makes register and cache ownership explicit, and permits flattening small axes. **Why it may lose:** overlarge live state, expensive index decoding, or poor input reuse between output tiles. **Legal if:** all required contribution pairs are covered exactly once, task/output independence is real, and reduction reordering satisfies arithmetic semantics.

**Alternate schedule:** make a reduction tile outermost and update many output tiles using one staged input region; this exchanges input reuse for output-partial traffic.

---

### Cross-case design lessons

| Repeated observation | Architectural response |
| --- | --- |
| Too few outputs to fill lanes | Split reduction across lanes or batch independent tasks |
| Too few contributors per output | Batch outputs rather than vectorize reduction |
| Long true dependency within task | Map tasks to lanes |
| Large overlap between neighboring outputs | Stage a shared input patch |
| Large common input per reduction step | Consider input-stationary traversal |
| Many colliding indirect writes | Group/privatize by destination |
| Expensive cross-lane reductions | Map outputs to lanes or accumulate longer before folding |
| Too many live accumulators | Reduce tile size, add flush boundary, or fission |
| Variable task lengths | Bucket or refill lanes |
| Incompatible pipeline lane mappings | Explicit repack, separate stages, or tile-level fusion |

### Reusable case-study worksheet

For any new algorithm, record:

1. **Reference equation/pseudocode** and exact output contract.
2. **Logical dimensions** and dependency edges.
3. **Three viable lane mappings** or reasons fewer are legal.
4. **Two tile-first architectures** with explicit stationary resources.
5. **Memory access pattern** for the innermost vector group.
6. **Live accumulator count**, scratch bytes, and horizontal communication.
7. **Overflow, aliasing, and boundary safety**.
8. **Predicted failure regime** (small sizes, irregular inputs, low reuse, etc.).
9. **Correctness oracle** and benchmark configurations.
10. **Measured result**, including variation and whole-pipeline cost.

A case study is complete only when its tradeoff is falsifiable. “Vector version is faster” is not an architectural explanation.

---

## 24. Design worksheet, exercises, and answer sketches


This chapter turns the handbook into a repeatable design practice. First specify the computation, then derive legal architectures, then narrow candidates using resource reasoning, and finally measure.

### 24.1 Copyable architecture-specification template

~~~text
ALGORITHM_SPEC
    logical_problem_axes   = [...]
    logical_output_axes    = [...]
    logical_reduction_axes = [...]
    ordered_state_axes     = [...]

    reference              = REFERENCE_PSEUDOCODE
    inputs_and_aliasing     = [...]
    output_contract         = [...]
    combine_laws            = [...]
    boundaries              = [...]
    observable_side_effects = [...]

DEPENDENCE_SPEC
    independent_axes       = [...]
    reduction_edges        = [...]
    ordered_edges          = [...]
    possible_write_conflict = [...]

ARCHITECTURE_CANDIDATE
    loop_schedule           = [...]
    outermost_data_patch    = [...]
    stationary_resource     = INPUT | OUTPUT | PROBLEM | HYBRID
    lane_semantics          = [...]
    logical_vector_width    = V
    register_output_tile    = [...]
    cache_input_tile        = [...]
    task_tile               = [...]
    accumulator_chains      = U
    data_layout             = [...]
    mask_and_tail_strategy  = [...]
    grouping_cost           = [...]
    expected_bottleneck     = [...]
    expected_failure_case   = [...]

VERIFICATION
    exact_or_tolerant       = [...]
    reference_oracle        = [...]
    edge_cases              = [...]
    random_test_inputs      = [...]
    dependency_proof        = [...]
    memory_safety_proof     = [...]

BENCHMARK
    workload_shapes         = [...]
    input_distributions     = [...]
    setup_included          = TRUE | FALSE
    pipeline_conversion     = INCLUDED | NONE
    warmup_policy           = [...]
    candidate_order_policy  = [...]
    repetition_policy       = [...]
    variability_report      = [...]
~~~

### 24.2 The 12-step design method

1. **State the semantic contract** and reference algorithm.
2. **Identify logical axes** independently of source loop syntax.
3. **Construct a dependence and conflict model**.
4. **List legal lane mappings** (output, reduction, task, candidate, mixed).
5. **Choose a stationary resource** (outputs, input tiles, task state).
6. **Build two or more complete loop schedules**.
7. **Quantify live registers and accumulator overflow risk**.
8. **Quantify data movement** at relevant storage levels.
9. **Estimate lane occupancy and communication**.
10. **Prove correctness and test boundary cases**.
11. **Benchmark representative workloads** with drift control.
12. **Document why the winner wins and when it loses**.

### 24.3 Exercises

#### Exercise 1 — One loop, small N

A pure independent map operates on N=5 elements with a logical vector width of 16. Should a vector kernel always win?

**Hint:** count active lanes and include setup and tail handling.

#### Exercise 2 — Two axes, short reduction

There are 100,000 outputs, each with 3 contributions. Compare output-lane and reduction-lane mapping.

**Hint:** how many horizontal reductions are needed if each output uses its own reduction vector?

#### Exercise 3 — Two axes, long reduction

There are only two outputs, each with 100,000 contributions. What changes?

**Hint:** output lanes may underfill a wide vector; independent contribution chains could help, subject to arithmetic laws.

#### Exercise 4 — Three axes, shared inputs

Problems P contain O outputs, and all O outputs reuse each input for reduction index r. Compare P–O–R versus P–R–O. What extra state does the second order require?

#### Exercise 5 — Four axes, short inner dimensions

A computation has independent axes A and B and output axes C and D, each of length 2. Explain how to fill a width-16 vector without pretending either C or D is a reduction axis.

#### Exercise 6 — A dependent scan

An inclusive scan uses a mathematically associative operation. Show how to process blocks in parallel internally and carry results between blocks. What changes if the operation is floating-point addition with bit-exact output requirements?

#### Exercise 7 — Colliding scatter

Several lanes update the same destination using COMBINE. State two legal strategies when COMBINE is associative, and one conservative strategy when it is not.

#### Exercise 8 — Ragged state machines

Task lengths are [2, 2, 2, 40]. Compare lockstep execution, sorting by length, and lane refill. Include preprocessing cost.

#### Exercise 9 — Neighborhood patch

A two-dimensional neighborhood has radius 2. Compare halo overhead for output tile shapes (8,8) and (32,8). Why isn't the larger tile automatically best?

#### Exercise 10 — Register pressure

An output-stationary microtile doubles its output count and its number of independent accumulator chains. What happens to live accumulator demand? Which other resource costs may improve or worsen?

#### Exercise 11 — Fusing stages

Two stages use incompatible lane layouts. Give one reason fusion helps and two reasons fission could win.

#### Exercise 12 — Recompute or materialize

A reusable intermediate is inexpensive to compute but expensive to load and rearrange. What measurements establish whether recomputing it inside each consumer tile is profitable?

#### Exercise 13 — Ordered dependence graph

Each point (i,j) depends on (i−1,j) and (i,j−1). Derive a wavefront index and specify which points in a wavefront can execute together.

#### Exercise 14 — Full multidimensional architecture

Given two problem axes, two output axes, and two reduction axes, propose: (a) a register tile, (b) a cache tile, and (c) a parallel task tile. Explain which axis is stationary at each level.

#### Exercise 15 — Misleading benchmark

A candidate appears 2× faster in one run but equal to baseline in another. List environmental variables and describe a paired experimental design that can distinguish code-structure effects from machine state.

---

### 24.4 Answer sketches

**1.** No. Only five useful lanes can operate; vector setup and masked or scalar tail handling may cost more than a short scalar loop. Benchmark both.

**2.** Output lanes naturally process many independent results without horizontal reductions; reduction lanes may force a fold for each three-contribution result and waste vector slots.

**3.** With only two outputs, reduction lanes or multiple independent partial chains become attractive. Exact floating-point order could prohibit reassociation.

**4.** P–R–O can load an input once and share it across outputs, but must keep O partial accumulators live or repeatedly read/write them. P–O–R uses fewer live partials but can reload shared inputs.

**5.** Flatten the valid (A,B,C,D) output coordinates into a list or combine several neighboring (A,B) groups. Ensure output indexing and memory locality remain efficient.

**6.** Build each block's local prefixes, merge with a carry from prior blocks, and update that carry. Bit-exact floating-point addition usually prevents an arbitrary parallel prefix tree.

**7.** Group contributions by destination or accumulate into private tables before merging. If nonassociative, prepare in parallel but apply updates in the reference order.

**8.** Lockstep wasting 38 iterations for three lanes is poor. Sorting/grouping by expected length improves similarity but requires extra passes. Refill can maintain occupancy but must move state safely and restore output identity.

**9.** For radius 2, (8,8) stages (12×12)/(8×8)=2.25 input-area units per output; (32,8) stages (36×12)/(32×8)=1.6875. Larger tiles lower halo ratio but may increase register/cache footprint and reduce parallelism.

**10.** Live accumulator state can increase by approximately 4× if both output count and chain count double. Reuse and latency hiding may improve; spilling or instruction-cache pressure may worsen.

**11.** Fusion avoids intermediate memory traffic. Fission may reduce live registers and allow each stage its own contiguous lane layout, with no costly register transpose.

**12.** Compare end-to-end producer, storage, reload, conversion, and consumer time across realistic reuse counts. Quantify added arithmetic versus avoided memory/communication, including spills.

**13.** w=i+j. Every required predecessor has sum w−1, so all valid points sharing w are ready after wavefront w−1 completes, assuming no additional edges.

**14.** Use a small output-coordinate product as register tile, a larger shared-input/reduction tile for cache, and disjoint problem-coordinate groups as thread tasks. Change the arrangement if task count is too small or cross-problem input reuse dominates.

**15.** Consider frequency, temperature, thread migration, memory layout, caches, background load, instrumentation, and dispatch. Alternate candidate order, warm up, repeat across fresh processes, record distributions, and examine whether timing regimes correlate with machine state.

### 24.5 Final design review checklist

- [ ] All logical contributions are covered exactly once.
- [ ] Dependencies and output conflicts are respected.
- [ ] Reassociation and overflow semantics are explicit.
- [ ] No inactive lane performs an invalid access.
- [ ] Final tails and empty segments are correct.
- [ ] The lane mapping and outermost stationary tile are documented.
- [ ] Register/live-state budgets are plausible.
- [ ] Memory-access patterns are identified for the innermost operation.
- [ ] Packing, conversion, task scheduling, and finalization costs are measured.
- [ ] The chosen schedule has a documented workload regime in which it might lose.
- [ ] A trusted reference verifies correctness under adversarial inputs.

**Final principle:** the SIMD programmer's primary output is an **execution architecture**—a legal schedule that places independent work, live results, and reused data in the right relationship. Vector instructions are a realization of that architecture, not its definition.

---

## Glossary and notation


| Term | Meaning |
| --- | --- |
| **Iteration space** | Set of all logical computation points, independent of execution order |
| **Schedule** | Mapping from logical iterations to time, tiles, vector groups, lanes, and tasks |
| **P** | Independent problem/task dimension |
| **O** | Independent output dimension |
| **R** | Reduction/contribution dimension |
| **S** | Ordered state or step dimension |
| **V** | Logical number of simultaneous vector lanes |
| **T_X** | Tile extent along dimension X |
| **U** | Number of independent accumulator chains or unrolled work instances |
| **Lane mapping** | Meaning assigned to each vector lane (output, task, reduction, candidate, etc.) |
| **Stationary resource** | Data intentionally retained in registers or a cache while surrounding loops execute |
| **Output-stationary** | Keep output partials local through many contributions |
| **Input-stationary** | Retain input data while using it across many outputs |
| **Problem-stationary** | Retain context/state for a group of independent tasks |
| **Strip mining** | Split one axis into an outer tile index and an inner offset |
| **Loop interchange** | Change order of independent/permitted axes |
| **Fusion/fission** | Combine passes or separate statements into passes |
| **Unroll-and-jam** | Unroll outer work and combine the resulting inner bodies |
| **Wavefront** | Set of iterations executable when their predecessors are complete |
| **Halo** | Extra input region needed to calculate a spatial output tile |
| **Reduction** | Combination of many contributions into one result |
| **Scan** | Ordered prefix reduction producing an intermediate result for each position |
| **Segmented reduction** | Distinct reductions over variable-length segments |
| **Divergence** | Different lanes execute different work or require different step counts |
| **Predication** | Represent conditional execution via active-lane masks |
| **Compaction** | Pack active/selected lanes or work items together |
| **Gather/scatter** | Noncontiguous memory read/write indexed by lane values |
| **Horizontal reduction** | Combine multiple vector lanes into one result |
| **Lane communication** | Shift, shuffle, permute, broadcast, or reduce across lanes |
| **Register pressure** | Amount of simultaneously live vector and scalar state |
| **Spill** | Temporary state stored outside registers because capacity or allocation is insufficient |
| **Arithmetic intensity** | Useful computation per transferred byte at a specified memory level |
| **Workload regime** | Input sizes/shapes/distributions in which one schedule has characteristic performance |

### Abstract pseudocode vocabulary

| Construct | Semantic meaning |
| --- | --- |
| **TILES(domain, size)** | Partition a logical domain into nonoverlapping ordered or indexed tiles |
| **VECTOR_EVALUATE** | Evaluate several logically independent contributions or outputs |
| **VECTOR_COMBINE** | Apply a combination operation independently to corresponding lanes |
| **HORIZONTAL_REDUCE** | Merge values across lanes under specified algebraic semantics |
| **VECTOR_PREFIX_SCAN** | Compute all within-block prefixes |
| **MASKED_COMBINE** | Update only active lanes while preserving inactive states |
| **SAFE_LOAD / SAFE_GATHER** | Read only valid memory locations, including masked tails |
| **SAFE_STORE** | Commit only valid output lanes |
| **COMPRESS** | Pack selected values while specifying order |
| **GROUP_BY_KEY** | Reorganize items by a shared property; stability must be stated when required |
| **FINALIZE** | Convert an accumulator to its observable output result |
| **IDENTITY** | Neutral value under COMBINE (when one exists) |
| **DEPENDENCY_CLOSURE** | Set of inputs or states needed by a tile |
| **COMMIT** | Make a completed output observable according to the specified contract |
| **VALID_LANES** | Mask for logical elements actually represented by a partial tile |

### Important distinctions

- **Logical V** is not necessarily one physical vector register.
- **Associative math** may not imply identical floating-point execution results.
- **Logical repeated loads** are not automatically DRAM transfers; caches change physical traffic.
- **Distinct indices** do not guarantee independent writes if indirect addresses collide.
- **Masking a result** does not make an earlier invalid memory access safe.
- **More loops** can mean *less data movement* after tiling.
- **More arithmetic** can be faster when it avoids expensive data movement.
- **A faster microbenchmark** is not necessarily a faster pipeline.
- **A plausible architecture** is not a measured performance result.
