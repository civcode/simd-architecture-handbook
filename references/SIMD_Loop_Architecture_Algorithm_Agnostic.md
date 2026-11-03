# Designing Efficient SIMD Loop Architectures
## An algorithm-agnostic, instruction-set-independent handbook using pseudocode

**Audience:** algorithm designers, compiler engineers, performance engineers, and developers optimizing data-parallel computations.  
**Scope:** iteration-space design, loop-nest architecture, SIMD lane assignment, tiling, register residency, data locality, dependency management, and experimental optimization.  
**Not in scope:** particular applications, processor instruction sets, intrinsic functions, programming-language syntax, or claims that one loop order is universally fastest.

> **Core principle:** Do not begin SIMD optimization by replacing scalar operations with vector operations. Begin by deciding **what one lane means**, **which partial results remain live**, **which input data is reused before eviction**, and **which tile is owned by the outermost loop**. The correct loop architecture follows from those choices.

This handbook is a standalone, abstract companion to the more implementation-specific SIMD guide. It does not assume image processing, matching, filtering, any named processor instruction set, or even a particular mathematical operator. All algorithms below are described with deliberately abstract pseudocode.

---

## 1. A vocabulary that separates algorithms from hardware

A generic computation can usually be described using some combination of these logical axes:

| Symbol | Abstract role | Typical responsibility |
|---|---|---|
| `P` | Independent problem or task | One independent item, sample, region, query, or work unit |
| `O` | Output coordinate | One destination result or one member of an output collection |
| `R` | Reduction coordinate | One contribution to an output result |
| `S` | Spatial or traversal coordinate | One position in an ordered input/output domain |
| `C` | Candidate coordinate | One alternative, option, hypothesis, or comparison |
| `K` | Auxiliary channel or component | One logically distinct value attached to a coordinate |

These are **roles**, not mandatory dimensions. A single algorithm might have several output axes, several reduction axes, no reduction at all, or no meaningful independent-problem axis. The labels are introduced so that we can discuss loop transformations without naming the operation performed inside them.

**Hardware-independent quantities:**

| Symbol | Meaning |
|---|---|
| `V` | Number of simultaneously processed lanes for the active value type |
| `B_O` | Number of output coordinates in one output tile |
| `B_R` | Number of reduction coordinates in one reduction tile |
| `B_P` | Number of independent problems in one problem tile |
| `U` | Additional independent work exposed by unrolling or multiple accumulators |
| `A` | Number of live accumulators required per logical output |
| `L` | Available fast storage, abstracted as registers or cache capacity |
| `F` | Operation that evaluates one contribution |
| `⊕` | Operation that combines a contribution with a partial result |
| `e` | Identity value for `⊕`, when one exists |

**Important distinctions:**

- **Lane width** `V` is a hardware or execution-model property. **Tile width** `B_O` is an algorithm-design decision. They are not the same quantity.
- An **output tile** is the set of results being produced together. An **input tile** is the region of source data loaded or reused together. Their shapes need not match.
- A **logical reduction** is an algorithmic dependency. A **horizontal vector reduction** is one possible physical implementation of that dependency.
- **SIMD execution** processes multiple lanes with one operation. **Task parallelism** executes independent problems on separate workers. They can be composed but are not interchangeable.
- **Innermost loop** does not automatically mean **vectorized dimension**. A lane can represent an output while the inner loop iterates reduction contributions.

### 1.1 A reference computation

The generic reduction-to-output problem is:

```text
FOR each problem p in P:
    FOR each output coordinate o in OUTPUTS(p):
        result ← IDENTITY
        FOR each reduction coordinate r in REDUCTIONS(p, o):
            contribution ← EVALUATE(p, o, r)
            result ← COMBINE(result, contribution)
        OUTPUT(p, o) ← FINALIZE(result)
```

The operation `EVALUATE` can access any allowed input data. `COMBINE` can be addition, minimum, maximum, a structured state update, or another valid combiner. Nothing in the following architectural discussion requires a particular formula.

The order of operations above is a **semantic reference**, not necessarily an efficient execution order.

### 1.2 The five fundamental architecture decisions

Every successful SIMD design answers five questions:

1. **Lane ownership:** Does each lane represent a different output, reduction contribution, independent problem, candidate, or channel?
2. **Accumulator ownership:** Which loop scope owns each partial result, and how long does it remain in fast storage?
3. **Reuse ownership:** Which loop scope should keep an input tile, output tile, or both hot?
4. **Traversal order:** Which dimensions move fastest in memory and which are outermost?
5. **Boundary and semantics:** What happens to incomplete tiles, dependent updates, numerical order, and exceptional elements?

A poor answer to any one of these can overwhelm the speedup gained from vector arithmetic.

---

## 2. Begin with dependencies, not with loops

Before interchanging loops, distinguish three types of relationships.

### 2.1 Independent outputs

If each output is updated only by its own contributions, independent output coordinates may usually be computed in any order:

```text
FOR each output o:
    state[o] ← IDENTITY
    FOR each contribution r:
        state[o] ← COMBINE(state[o], EVALUATE(o, r))
```

Different `o` values can be kept in different SIMD lanes without needing cross-lane communication. This is often the simplest and most efficient mapping for a multi-output reduction.

### 2.2 Reduction dependencies

Repeated updates to the **same** output are logically ordered unless the combiner supports a different evaluation order:

```text
state ← IDENTITY
FOR each r:
    state ← COMBINE(state, EVALUATE(r))
```

To split this reduction among lanes, the algorithm must define how lane-local partial results are merged. That requires correct associativity or an explicitly accepted change in numerical semantics.

### 2.3 Cross-output dependencies

Some computations update one output using another output's newly computed value:

```text
FOR each o:
    OUTPUT[o] ← TRANSFORM(OUTPUT[o - 1], INPUT[o])
```

The loop is not embarrassingly data-parallel. A simple lane-per-output transformation changes the dependency structure. It may need a scan, a transformed recurrence, wavefront scheduling, explicit synchronization, or remain partly sequential.

**Legality rule:** A loop transformation is valid only if it preserves all required reads, writes, dependencies, boundary behavior, and numerical semantics. A faster loop nest that produces a different result is not an optimization unless the new result is explicitly acceptable.

### 2.4 Build a dependence map

Before choosing tiles, record:

| Question | Why it matters |
|---|---|
| Can two output coordinates be computed independently? | Determines whether output lanes are safe |
| Is `COMBINE` associative? | Determines whether reductions can be regrouped |
| Is `COMBINE` commutative? | Determines whether contributions may be reordered |
| Can intermediate values overflow, saturate, or round? | May forbid changing the accumulation schedule |
| Do tasks read results that another task writes? | May forbid task reordering or fusion |
| Can multiple lanes write the same destination? | May require conflict detection or another ownership model |
| Are input addresses data-dependent? | Determines whether contiguous loads, gathers, or staging are needed |

---

## 3. SIMD is a mapping from mathematical work to lanes

The most consequential choice is **what adjacent lanes represent**.

### 3.1 Output-lane mapping

Each lane owns a different output, and each lane retains its own partial accumulator.

```text
FOR output_tile o0, size B_O:
    active ← VALID_OUTPUT_LANES(o0, B_O)
    accumulators ← VECTOR_OF_IDENTITIES(active)

    FOR each reduction r:
        values ← EVALUATE_FOR_OUTPUT_LANES(o0, r, active)
        accumulators ← COMBINE_LANEWISE(accumulators, values)

    STORE_OUTPUT_LANES(o0, FINALIZE_LANEWISE(accumulators), active)
```

**Advantages:** independent per-lane accumulators; no final horizontal reduction; output values remain in registers; multiple outputs share reduction traversal and sometimes input loads.

**Potential costs:** widely separated or irregular input addresses; many live accumulators for large output tiles; inactive lanes when outputs have unequal work.

**Good starting point when:** outputs are independent and either their source addresses are contiguous or some common input can be broadcast to many outputs.

### 3.2 Reduction-lane mapping

Lanes each process a different contribution to the **same** output. Each lane holds a partial reduction, and lane results are combined at the end.

```text
FOR each output o:
    partials ← VECTOR_OF_IDENTITIES(V)

    FOR reduction_tile r0, size V:
        active ← VALID_REDUCTION_LANES(r0, V)
        values ← EVALUATE_REDUCTION_LANES(o, r0, active)
        partials ← COMBINE_LANEWISE(partials, values, active)

    result ← REDUCE_ACROSS_LANES(partials)
    OUTPUT[o] ← FINALIZE(result)
```

**Advantages:** naturally contiguous input when the reduction axis is stored sequentially; useful when output count is small or reduction length is long.

**Potential costs:** horizontal reduction; insufficient independent accumulators; reassociation or reproducibility concerns; repeated setup per output.

**Good starting point when:** reducing one long, contiguous sequence into relatively few outputs.

### 3.3 Independent-problem mapping

Each lane owns a different problem instance.

```text
FOR problem_tile p0, size V:
    active ← VALID_PROBLEMS(p0, V)
    states ← VECTOR_OF_INITIAL_STATES(active)

    FOR each corresponding logical step t:
        values ← EVALUATE_ONE_STEP_PER_PROBLEM(p0, t, active)
        states ← UPDATE_LANEWISE(states, values, active)

    STORE_PROBLEM_RESULTS(p0, states, active)
```

**Advantages:** no cross-lane reduction; naturally independent tasks; can work well for many short, uniform problems.

**Potential costs:** adjacent problems may not occupy adjacent memory locations; data-dependent stopping conditions cause divergent lanes; working sets from many problems may compete in cache.

**Good starting point when:** many small problems have similar work and their corresponding fields can be loaded efficiently.

### 3.4 Candidate or alternative mapping

Lanes represent alternatives for a shared input item. This is structurally an output-lane mapping but important enough to recognize separately: shared input may be loaded once and reused across alternatives.

```text
FOR each input_item p:
    FOR candidate_tile c0, size B_C:
        candidate_states ← VECTOR_OF_IDENTITIES(B_C)

        FOR each common contribution r:
            common_data ← LOAD_SHARED_INPUT(p, r)
            candidate_data ← LOAD_CANDIDATE_INPUTS(p, c0, r)
            candidate_states ← COMBINE_LANEWISE(
                candidate_states,
                EVALUATE_ALL_CANDIDATES(common_data, candidate_data))

        STORE_CANDIDATE_RESULTS(p, c0, candidate_states)
```

**Trade-off:** larger candidate tiles improve reuse of `common_data` but require more registers and may increase candidate-input traffic.

### 3.5 Mixed mappings

Sometimes neither pure output nor pure reduction vectorization fills the machine. For example, a vector might represent several outputs, each with several reduction lanes. Such layouts can raise utilization but require lane partitioning and intra-group reductions.

```text
LANE_LAYOUT ← GROUPS(output_groups, reduction_lanes_per_group)

FOR output_tile:
    partials ← IDENTITIES_MATCHING(LANE_LAYOUT)
    FOR reduction_tile:
        partials ← UPDATE_MATCHING_LANES(partials, input_values)
    group_results ← REDUCE_WITHIN_EACH_OUTPUT_GROUP(partials)
    STORE(group_results)
```

Use mixed mappings only when the simpler mappings leave substantial capacity idle or produce poor locality.

### 3.6 A lane-mapping decision table

| Situation | First mapping to test | Main hazard |
|---|---|---|
| Many independent outputs, short or medium reductions | Output lanes | Register pressure or scattered loads |
| Few outputs, long contiguous reductions | Reduction lanes | Horizontal reduction and changed reduction order |
| Many short uniform independent tasks | Problem lanes | Memory layout across tasks |
| Many alternatives share a common input | Candidate lanes | Replicated per-candidate state |
| Outputs are irregular or reduction lengths vary | Grouped or compacted problem lanes | Mask cost and divergence |
| Dependence across outputs | Specialized schedule or partial scalarization | Incorrect cross-lane dependencies |

**No mapping wins by definition.** The mapping determines the required data motion; data motion often determines performance.

---

## 4. Why the scalar loop order is not the SIMD loop order

A reference algorithm may write its loops in the order most convenient for reasoning, often placing the output loop outside and the reduction loop inside:

```text
FOR o:
    acc ← IDENTITY
    FOR r:
        acc ← COMBINE(acc, EVALUATE(o, r))
    OUTPUT[o] ← acc
```

A vector implementation might instead process a **tile of outputs** at once:

```text
FOR o0 BY B_O:
    acc[0 .. B_O - 1] ← IDENTITY

    FOR r:
        FOR all active output lanes in o0:
            acc[lane] ← COMBINE(acc[lane], EVALUATE(o0 + lane, r))

    STORE(acc)
```

The mathematical dimensions are unchanged. What changed is the **scope of live intermediate results** and the **reuse opportunity for input data**.

A naive loop swap can be harmful:

```text
FOR r:
    FOR o:
        OUTPUT[o] ← COMBINE(OUTPUT[o], EVALUATE(o, r))
```

This may reread and rewrite the entire output collection for every reduction step. Its memory traffic can be far larger than the register-resident tiled version. Therefore:

> **Loop interchange, loop tiling, and accumulator placement must be designed together.** Interchanging two loops without changing where partial results live is not a complete SIMD architecture.

---

## 5. Three-loop permutations: all six are meaningful

For discussion, assume three abstract axes:

- `P`: independent problem instances;
- `O`: outputs within one problem;
- `R`: contributions per output.

For mathematical convenience, suppose every problem has the same output and reduction extents. The following table describes *raw loop permutations*. Real optimized kernels often add tiles and retain state across multiple loops, so the best version of a permutation is not necessarily the literal simple nest.

| Order (outer → inner) | Natural reuse / ownership | Typical opportunity | Main architectural risk |
|---|---|---|---|
| `P → O → R` | One problem, then one output | Simple output-local reduction | No sharing among nearby outputs; narrow SIMD if vectorizing `R` |
| `P → R → O` | One problem, one contribution, many outputs | Broadcast a common contribution; contiguous output lanes | Output state spills unless `O` is tiled |
| `O → P → R` | Same output coordinate across many problems | Batch analogous outputs across problems | Problems may be far apart in memory |
| `O → R → P` | Same output and reduction across a batch | Strong batch-lane regularity | Scattered per-problem loads; divergent bounds |
| `R → P → O` | One contribution across problems and outputs | Reuse reduction-stage setup | Large live output state or repeated memory updates |
| `R → O → P` | One contribution, output, then problem batch | Lane-per-problem at fixed coordinates | Per-problem output state retention; poor locality |

**This table does not rank the permutations by speed.** A favorable order depends on memory layout, input sharing, the number of outputs, the length of reductions, and how much state can be retained in registers or cache.

### 5.1 Why `P → R → O` can beat `P → O → R`

Assume many outputs reuse one piece of data associated with a reduction step.

```text
FOR p:
    FOR output_tile o0 BY B_O:
        accumulators ← IDENTITIES(B_O)

        FOR r:
            shared ← LOAD_SHARED(p, r)
            per_output ← LOAD_FOR_OUTPUT_TILE(p, o0, r)
            contributions ← EVALUATE(shared, per_output)
            accumulators ← COMBINE_LANEWISE(accumulators, contributions)

        STORE_OUTPUT_TILE(p, o0, accumulators)
```

`R` is inside the output tile, but it is **outside the lane-wise update**. A shared value can be loaded once per output tile and reused across lanes. The output tile still owns the accumulators and prevents full-output-array traffic.

### 5.2 Why `P → O → R` can still win

Suppose the reduction axis is contiguous, outputs share very little input, and each output has a long reduction. Then keeping `R` as the lane dimension and finishing each `O` before starting the next can be efficient:

```text
FOR p:
    FOR o:
        partials ← IDENTITIES(V)
        FOR reduction_tile r0 BY V:
            values ← LOAD_CONTIGUOUS_REDUCTION_SEGMENT(p, o, r0)
            partials ← COMBINE_LANEWISE(partials, EVALUATE(values))
        OUTPUT(p, o) ← REDUCE_ACROSS_LANES(partials)
```

Here changing to output-lane SIMD may create strided or gathered memory accesses that cost more than the horizontal reduction it avoids.

### 5.3 Why `O → P → R` may be ideal for batched workloads

If corresponding output fields of many problems are contiguous, processing the same `O` across a tile of `P` can fill lanes naturally:

```text
FOR o:
    FOR problem_tile p0 BY B_P:
        states ← IDENTITIES(B_P)
        FOR r:
            values ← LOAD_CORRESPONDING_VALUES(p0, o, r)
            states ← COMBINE_LANEWISE(states, EVALUATE(values))
        STORE_CORRESPONDING_OUTPUTS(p0, o, states)
```

If storage groups complete problems instead, the same loop order may require expensive scattered loads. The data layout—not the symbols—decides.

### 5.4 Important condition for every permutation

If two iterations write the same partial result, the architecture needs one of the following:

- Keep that result live until all required contributions are processed.
- Maintain separate partial results, then merge them using a legal reduction.
- Read and write the result between chunks, accepting the extra traffic.
- Partition ownership to avoid multiple writers.

No loop permutation can make this state-management problem disappear.

---

## 6. Tiling makes loop architecture fundamentally different

Once axes are tiled, **tile order** and **within-tile order** become separate decisions.

### 6.1 Output tile outermost: register-resident results

```text
FOR problem p:
    FOR output_tile o0 BY B_O:
        acc ← IDENTITIES(B_O)

        FOR reduction_tile r0 BY B_R:
            FOR r IN TILE(r0, B_R):
                values ← EVALUATE_ALL_OUTPUT_LANES(p, o0, r)
                acc ← COMBINE_LANEWISE(acc, values)

        STORE_OUTPUT_TILE(p, o0, FINALIZE(acc))
```

**Primary benefit:** each output tile is read or written only at the boundary; partial results stay live inside the tile.  
**Primary cost:** the reduction input may be revisited separately for each output tile.

### 6.2 Reduction tile outermost: input-stage reuse

```text
FOR problem p:
    INITIALIZE_ALL_OUTPUT_STATES_IN_MEMORY(p)

    FOR reduction_tile r0 BY B_R:
        staged_input ← PREPARE_REDUCTION_INPUT(p, r0)

        FOR output_tile o0 BY B_O:
            acc ← LOAD_OUTPUT_STATE_TILE(p, o0)
            acc ← PROCESS(staged_input, o0, acc)
            STORE_OUTPUT_STATE_TILE(p, o0, acc)

    FINALIZE_ALL_OUTPUT_STATES(p)
```

**Primary benefit:** expensive reduction-stage data can be prepared once and used by many output tiles.  
**Primary cost:** output-state tiles may be repeatedly loaded and stored; output working set may dominate bandwidth.

This architecture is sometimes better, but it should be chosen for a **measured input-reuse advantage**, not because fewer source loads look attractive in pseudocode.

### 6.3 Two-dimensional blocked ownership

A compromise keeps a bounded set of output states while processing a bounded reduction region:

```text
FOR problem_tile p0 BY B_P:
    FOR output_tile o0 BY B_O:
        acc ← IDENTITIES(B_P, B_O)

        FOR reduction_tile r0 BY B_R:
            input_tile ← PREPARE_INPUTS(p0, o0, r0)
            acc ← UPDATE_TILE(acc, input_tile)

        STORE_AND_FINALIZE(acc)
```

This creates a three-dimensional logical work tile, even though the inner update may consist of many smaller vector operations. The tile can be tuned for register capacity, cache capacity, lane utilization, and data reuse independently.

### 6.4 Tiling is not the same as changing the mathematical algorithm

A valid tile-based implementation must cover exactly the same logical iteration pairs (or triples) as the reference:

```text
REFERENCE_WORK ← SET_OF_VALID_ITERATIONS()
TILED_WORK ← EMPTY_SET

FOR each outer tile:
    FOR each nested tile:
        FOR each valid lane:
            ADD_LOGICAL_ITERATION_TO(TILED_WORK)

ASSERT TILED_WORK == REFERENCE_WORK
ASSERT EACH_ITERATION_HAS_CORRECT_MULTIPLICITY
```

This check is especially useful for tiled reductions with halos, overlapping input windows, and incomplete boundary tiles.

---

## 7. How to choose the outermost tile

There are three competing resource goals:

1. **Keep partial outputs in registers** to avoid repeated state traffic.
2. **Keep useful inputs in cache** to avoid reloading them unnecessarily.
3. **Keep SIMD lanes busy** by grouping enough independent work.

The outermost tile should represent the resource whose reuse is most valuable and most constrained by eviction or spills.

### 7.1 Output-stationary architecture

The outermost tile selects a bounded set of outputs. Accumulators stay live until those outputs are complete.

```text
FOR output_tile:
    keep output accumulators live
    traverse all required inputs
    write completed outputs once
```

**Prefer when:** output state is expensive to reload; each output has many contributions; per-output state fits comfortably in registers; input reuse across *different* output tiles is moderate.

### 7.2 Input-stationary architecture

The outermost tile selects an input region or reduction stage. Inputs are reused across many outputs before replacement.

```text
FOR input_tile:
    load or prepare input tile
    FOR each affected output tile:
        update output state
```

**Prefer when:** expensive input preparation is shared across many outputs; input bandwidth dominates; output states are small enough to remain in a sufficiently fast storage level or can be partitioned effectively.

**Warning:** if partial output state is repeatedly moved to main memory, the saved input loads may be outweighed many times over.

### 7.3 Problem-stationary architecture

The outermost tile selects independent problems and completes each group locally.

```text
FOR problem_tile:
    prepare problem-local state
    complete the required outputs
    release problem-local state
```

**Prefer when:** each problem has reusable setup data; tasks are independent; per-problem working sets fit in cache; per-problem output write ownership is easy to maintain.

### 7.4 An outermost-loop selection checklist

Ask these in order:

1. Which values are reused most frequently?
2. Which reused values are the most expensive to reload or regenerate?
3. Which partial results must survive the longest dependency chain?
4. How many such results fit in fast storage without spilling?
5. Which axis is contiguous in physical storage?
6. Which candidate tile permits full or nearly full lane utilization?
7. Which choice creates the smallest unavoidable data traffic, not just the fewest loop iterations?
8. Does changing the outer loop create duplicate work, expensive gathers, or a second reduction step?

There is no universal answer such as “always put the largest dimension outermost” or “always put the contiguous dimension innermost.” Both can be wrong.

---
## 8. Register pressure: the invisible limit on output tile size

A large logical tile is not automatically a good tile. Its accumulator state competes with loaded inputs, temporary expressions, address calculations, control masks, and other live vectors.

### 8.1 A simple register-budget model

For an output-lane kernel, estimate:

```text
accumulator_vectors  ← CEIL(B_O / V) × A
input_vectors        ← PEAK_SIMULTANEOUS_INPUT_VECTORS()
temporary_vectors    ← PEAK_SIMULTANEOUS_TEMPORARIES()
mask_vectors         ← PEAK_SIMULTANEOUS_MASKS()
other_live_vectors   ← OTHER_REQUIRED_STATE()

peak_live_vectors ← accumulator_vectors
                  + input_vectors
                  + temporary_vectors
                  + mask_vectors
                  + other_live_vectors

IF peak_live_vectors exceeds practical fast-storage budget:
    expect spilling or forced recomputation
```

This is a *planning model*, not a precise register allocator. Actual lifetime overlap matters: values whose lifetimes do not overlap can share storage. The compiler or execution environment may also reserve some registers.

### 8.2 Why a tile twice as large may run slower

Increasing `B_O` can reduce:

- The number of times common input data is loaded.
- Outer-loop iterations and tile setup.
- The number of times reduction coordinates are traversed.

But it can increase:

- The number of simultaneously live accumulators.
- The register lifetime of every accumulator.
- Spills and reloads of partial output state.
- Instruction scheduling difficulty.
- Tail waste when the problem size is not divisible by the tile width.

A tile that just fits in the register budget may outperform both smaller and larger tiles. A tile that barely fits in theory may still be too large when the kernel has many temporary vectors.

### 8.3 Use multiple accumulators deliberately

A single dependency chain can limit throughput even when memory is fast. Several independent partial accumulators can permit more work to be in flight:

```text
acc0 ← IDENTITY
acc1 ← IDENTITY
acc2 ← IDENTITY
acc3 ← IDENTITY

FOR reduction groups:
    acc0 ← COMBINE(acc0, CONTRIBUTION_FOR_GROUP_0())
    acc1 ← COMBINE(acc1, CONTRIBUTION_FOR_GROUP_1())
    acc2 ← COMBINE(acc2, CONTRIBUTION_FOR_GROUP_2())
    acc3 ← COMBINE(acc3, CONTRIBUTION_FOR_GROUP_3())

result ← MERGE_PARTIALS_IN_ALLOWED_ORDER(acc0, acc1, acc2, acc3)
```

This exposes instruction-level parallelism, but it may change the reduction order and it consumes more registers. Multiple accumulators are therefore most useful when the combiner permits a legal merge and the register budget still has headroom.

### 8.4 Two different meanings of a large tile

A `B_O = 64` output tile might consist of:

- One logical vector with 64 lanes; or
- Four 16-lane vectors; or
- Eight 8-lane vectors; or
- Another grouped or mixed layout.

The **logical tile width** describes how many outputs are finished together. The **physical vector configuration** describes how that work is represented. Code architecture should not conflate the two.

---

## 9. Cache locality and data movement determine loop order

An operation can have excellent vector arithmetic and still run slowly if each lane forces a different cache line or if input data is revisited after eviction.

### 9.1 Separate temporal and spatial locality

- **Spatial locality:** neighboring accesses touch nearby addresses, so the same transferred memory unit supplies several useful values.
- **Temporal locality:** a value is reused before it leaves fast storage.
- **Output locality:** intermediate result state remains near the computation instead of being repeatedly materialized in memory.
- **Cross-output reuse:** multiple outputs consume some of the same inputs.
- **Cross-problem reuse:** independent problems share setup, lookup data, or other common resources.

For SIMD, these interact: a lane mapping with excellent output locality can have poor input locality, and the reverse can also occur.

### 9.2 Data layout can reverse which loop order wins

Consider the same logical two-dimensional data with two possible physical arrangements:

```text
LAYOUT_A: values grouped by output

    output 0: r0 r1 r2 r3 ...
    output 1: r0 r1 r2 r3 ...
    output 2: r0 r1 r2 r3 ...

LAYOUT_B: values grouped by reduction step

    step r0: o0 o1 o2 o3 ...
    step r1: o0 o1 o2 o3 ...
    step r2: o0 o1 o2 o3 ...
```

Under `LAYOUT_A`, vectorizing reduction coordinates can read contiguous elements for one output. Under `LAYOUT_B`, vectorizing output coordinates can read contiguous elements at one reduction step.

The arithmetic is identical; the memory access pattern is not.

### 9.3 Array-of-records versus record-of-arrays

When independent problems are stored as full records, corresponding fields from neighboring problems are separated in memory:

```text
RECORD_LAYOUT:
    [p0.fieldA, p0.fieldB, p0.fieldC]
    [p1.fieldA, p1.fieldB, p1.fieldC]
    [p2.fieldA, p2.fieldB, p2.fieldC]
```

If SIMD lanes represent independent problems and all lanes need `fieldA`, a grouped-by-field layout can improve contiguity:

```text
FIELD_LAYOUT:
    [p0.fieldA, p1.fieldA, p2.fieldA]
    [p0.fieldB, p1.fieldB, p2.fieldB]
    [p0.fieldC, p1.fieldC, p2.fieldC]
```

Neither layout is universally better: record layout can be superior for scalar task-local processing; field layout can be superior for batch-lane processing. Conversion also has a cost. A third option groups a small block of problems and stores fields contiguously **within each block**.

### 9.4 Repacking only pays when amortized

```text
FOR problem_block:
    packed_input ← REORGANIZE_FOR_LANES(original_input)
    FOR many repeated uses:
        results ← COMPUTE_FROM_PACKED(packed_input)
    WRITE_RESULTS(results)
```

Repacking pays when saved gather, shuffle, and cache costs across repeated uses exceed the cost of copying or reorganizing data. For a one-time short operation, direct strided access can be cheaper.

### 9.5 Estimate bytes, not only operations

For each candidate architecture, estimate:

```text
bytes_loaded_from_inputs
bytes_loaded_from_partial_outputs
bytes_written_to_partial_outputs
bytes_written_to_final_outputs
bytes_moved_by_repacking
bytes_loaded_because_of_repeated_tiles
```

Distinguish **logical bytes requested** from **physical bytes transferred**. A small number of scattered loads can transfer many cache lines; overlapping loads may be served from cache. Cache-line size, associativity, replacement, translation behavior, and competing workloads also matter.

### 9.6 A practical locality thought experiment

Suppose one reduction value `shared[r]` is used by every output. If there are `N_O` outputs and output tile size `B_O`, then a simple output-stationary traversal may load each `shared[r]` approximately `CEIL(N_O / B_O)` times. If the shared data remains cached, those loads need not all reach slower memory.

An input-stationary traversal may load `shared[r]` once, but might repeatedly reload and store the partial outputs. The correct comparison is:

```text
saved cost of shared-input access
VERSUS
added cost of partial-output traffic and larger live state
```

Counting only source-array loads would miss the deciding cost.

---

## 10. Reduction tiles, accumulator width, and numerical behavior

A reduction architecture must choose not only *where* partial results live, but also *how long* they can safely be accumulated.

### 10.1 Narrow accumulators can be faster—but only within valid bounds

Suppose individual nonnegative contributions are bounded by `C_MAX`, and the accumulator can represent values up to `M_MAX`. Starting from zero, a safe chunk length is at most:

```text
MAX_SAFE_CONTRIBUTIONS ← FLOOR(M_MAX / C_MAX)
```

A reduction tile with no more than that many contributions can safely use the narrow accumulator, assuming ordinary exact integer addition and the stated bounds.

```text
FOR output_tile:
    wide_result ← ZERO_IN_WIDE_REPRESENTATION()

    FOR reduction_chunk with safe contribution count:
        narrow_acc ← ZERO_IN_NARROW_REPRESENTATION()

        FOR contribution IN reduction_chunk:
            narrow_acc ← ADD_EXACT_WITHIN_NARROW_RANGE(
                narrow_acc, EVALUATE(contribution))

        wide_result ← ADD_WIDENED(wide_result, narrow_acc)

    STORE(wide_result)
```

**Crucial:** every chunk, including the final incomplete chunk, must be flushed into the wide result exactly once.

### 10.2 Signed and mixed-sign contributions require a different bound

When contributions can be negative, a single nonnegative maximum is insufficient. Bound the worst possible positive and negative partial sums, or prove the relevant arithmetic is safe for the actual ordering. Saturating and wrapping arithmetic also require their own explicit semantics.

### 10.3 Floating-point reassociation is not automatically equivalent

For floating-point data:

```text
COMBINE(COMBINE(a, b), c)
```

may differ from:

```text
COMBINE(a, COMBINE(b, c))
```

because of rounding, exceptional values, and special-value rules. Reduction-lane parallelism and multi-accumulator unrolling may therefore change exact results.

Choose and document a numerical contract:

1. **Bitwise-identical:** preserve the specified sequence and exact intermediate behavior.
2. **Tolerance-based:** permit bounded numerical differences with explicit thresholds.
3. **Reproducible alternative:** use a defined parallel reduction tree across all variants.

Do not silently compare an approximate SIMD result against an exact reference using only a loose average-error metric. Extreme and cancellation-heavy inputs matter.

### 10.4 Selection and tie-breaking are part of the contract

A reduction that returns the location of an optimum needs a tie-breaking policy. For example:

```text
IF candidate_value < current_best_value:
    replace current best
ELSE:
    preserve current best
```

is different from replacing on equality. If a vector reduction merges partial minima, it must reproduce the required tie order, often by comparing `(value, logical_index)` according to an explicitly defined total ordering.

---

## 11. Handling tails and incomplete tiles safely

Real problem sizes are rarely exact multiples of vector and tile widths. A complete architecture specifies every tail at every level.

### 11.1 Masked execution

```text
FOR output_tile o0 BY B_O:
    active_output_lanes ← lane_index + o0 < output_count

    acc ← IDENTITIES(B_O)
    FOR reduction_tile r0 BY B_R:
        active_reduction_lanes ← lane_index + r0 < reduction_count
        UPDATE_ONLY_VALID_COMBINATIONS(acc,
                                       active_output_lanes,
                                       active_reduction_lanes)

    STORE_ONLY_VALID_OUTPUTS(acc, active_output_lanes)
```

An inactive lane must not contribute to a result. Its input access also must be legal for the execution model; simply discarding the computed value afterward does not make an out-of-bounds read safe.

### 11.2 Smaller-vector fallback

```text
PROCESS_ALL_FULL_TILES_WITH_MAIN_VECTOR_WIDTH()
PROCESS_REMAINDER_WITH_SMALLER_VECTOR_GROUPS()
PROCESS_FINAL_INCOMPLETE_GROUP_WITH_SCALAR_OR_MASKED_LOGIC()
```

This avoids paying mask overhead in the main loop at the cost of multiple code paths.

### 11.3 Scalar cleanup

```text
full_end ← FLOOR(output_count / V) × V

FOR o0 FROM 0 TO full_end - 1 BY V:
    PROCESS_FULL_VECTOR(o0)

FOR o FROM full_end TO output_count - 1:
    PROCESS_ONE_OUTPUT(o)
```

It can be a good baseline, especially when the tail is small. But if each scalar remainder triggers expensive setup, a masked final vector may be preferable.

### 11.4 Tail choices interact with tile width

An apparently good `B_O` can waste substantial work when common output counts are often slightly larger than multiples of the tile width. Measure lane utilization over the **actual distribution of problem sizes**, not only a conveniently divisible synthetic size.

---

## 12. Loop unrolling, software pipelining, and instruction-level parallelism

Changing loop order is only one dimension of SIMD architecture. Two loops with the same order can have very different performance because of unrolling, dependency depth, and state lifetime.

### 12.1 Sequential dependency chain

```text
acc ← IDENTITY
FOR r:
    acc ← COMBINE(acc, EVALUATE(r))
```

If `COMBINE` has latency, the next update must wait on the previous result.

### 12.2 Multiple independent update chains

```text
acc0, acc1, acc2, acc3 ← IDENTITIES()

FOR reduction_groups:
    acc0 ← COMBINE(acc0, EVALUATE(group, lane_group_0))
    acc1 ← COMBINE(acc1, EVALUATE(group, lane_group_1))
    acc2 ← COMBINE(acc2, EVALUATE(group, lane_group_2))
    acc3 ← COMBINE(acc3, EVALUATE(group, lane_group_3))

result ← LEGAL_MERGE(acc0, acc1, acc2, acc3)
```

This helps only if the reduction can be legally partitioned and merged. For output-lane SIMD, similar independence may already exist because different output tiles use separate accumulators.

### 12.3 Overlap loading with computation

```text
current ← LOAD_INPUT_TILE(first_tile)

FOR each remaining tile:
    next ← PREPARE_OR_PREFETCH(next_tile)
    COMPUTE(current)
    current ← next

COMPUTE(current)
```

This conceptual schedule can shorten exposed memory latency, but it can also increase simultaneous live data and cache demand. It is usually valuable after the fundamental tile and lane mapping are already sensible.

### 12.4 Unrolling can harm instruction-cache locality

Expanding every loop into many specialized copies may improve short inner-loop control overhead but increase code size, register usage, and instruction-fetch pressure. Evaluate unrolling independently of tile width; do not assume more unrolling always helps.

---

## 13. Divergence, irregular addresses, and work imbalance

SIMD is easiest when all lanes perform comparable work. Real algorithms often contain data-dependent conditions and variable-length reductions.

### 13.1 Mask inactive lanes

```text
active ← INITIAL_VALID_LANES()
state ← INITIAL_STATES()

WHILE ANY(active):
    contributions, next_active ← STEP_ACTIVE_LANES(state, active)
    state ← UPDATE_ACTIVE_LANES(state, contributions, active)
    active ← next_active
```

When most lanes become inactive, the vector keeps executing for a small amount of useful work.

### 13.2 Group similar workloads

```text
problems ← GROUP_BY_ESTIMATED_WORK_LENGTH(problems)

FOR each group:
    PROCESS_PROBLEMS_WITH_SIMILAR_EXTENTS(group)
```

Sorting or bucketing has overhead and may disrupt original output order. It is worthwhile only if the repeated benefit exceeds that overhead, and results must be restored to their correct destinations.

### 13.3 Compact active lanes

```text
active_items ← LOAD_PROBLEM_BATCH()

WHILE active_items is not empty:
    PROCESS_ONE_STAGE(active_items)
    FINISHED ← DETECT_COMPLETED_ITEMS(active_items)
    WRITE_FINISHED_RESULTS(FINISHED)
    active_items ← COMPACT_UNFINISHED_ITEMS(active_items)
    REFILL_FREE_LANES_IF_AVAILABLE(active_items)
```

Compaction improves utilization in long-running irregular tasks, but it requires lane-to-output identity tracking, data movement, and possibly extra gathers. It is not automatically better for short loops.

### 13.4 Masked arithmetic versus actual memory safety

A masked computation is only safe if inactive addresses are not accessed in a forbidden way. Conceptually separate address formation, legal loading, arithmetic masking, and storage masking. This distinction is necessary for portable pseudocode and for architectures with different masking semantics.

---

## 14. Multi-dimensional iteration spaces: the outermost *patch* choice

Many algorithms have two or more output coordinates and one or more reduction coordinates. The same SIMD principles apply even when there are four, five, or six original loops.

Assume a logical computation produces `OUTPUT[a, b]` from contributions indexed by `r`:

```text
FOR a:
    FOR b:
        acc ← IDENTITY
        FOR r:
            acc ← COMBINE(acc, EVALUATE(a, b, r))
        OUTPUT[a, b] ← FINALIZE(acc)
```

You now have at least four independent design choices:

1. Tile `a` first or `b` first.
2. Use lanes across `a`, across `b`, across `r`, or across combinations.
3. Keep an output rectangle or an input/reduction slice stationary.
4. Choose tile shapes that fit both the register budget and the memory layout.

### 14.1 Output-rectangle-stationary architecture

```text
FOR a_tile BY B_A:
    FOR b_tile BY B_B:
        acc ← IDENTITIES(B_A, B_B)

        FOR reduction_tile BY B_R:
            inputs ← LOAD_REQUIRED_INPUTS(a_tile, b_tile, reduction_tile)
            acc ← UPDATE_OUTPUT_RECTANGLE(acc, inputs)

        STORE_OUTPUT_RECTANGLE(acc)
```

This can reuse overlapping source data across several neighboring outputs, but the number of live accumulators grows approximately with the rectangle area. A wide but shallow tile and a narrow but tall tile can have radically different memory behavior.

### 14.2 Reduction-slice-stationary architecture

```text
INITIALIZE_OUTPUT_STATE()

FOR reduction_tile BY B_R:
    input_slice ← PREPARE_SHARED_REDUCTION_SLICE()

    FOR a_tile BY B_A:
        FOR b_tile BY B_B:
            partials ← LOAD_OUTPUT_RECTANGLE_STATE(a_tile, b_tile)
            partials ← UPDATE_FROM_SLICE(partials, input_slice)            STORE_OUTPUT_RECTANGLE_STATE(a_tile, b_tile, partials)

FINALIZE_OUTPUT_STATE()
```

This may improve input reuse but turns the outputs into repeatedly updated memory state. It is attractive only when shared input value is sufficiently high and the output-state traffic remains manageable.

### 14.3 Shape matters more than tile area alone

Two tiles with the same number of elements can have different:

- Contiguity along the fastest-varying storage axis.
- Overlap or halo size of source data.
- Number of input streams.
- Register permutations needed to map lanes to outputs.
- Cache-line and page behavior.
- Number of edge or masked elements.

Therefore, optimize `(B_A, B_B, B_R)` jointly where necessary; do not reduce a multi-dimensional tile to a single “block size” number.

### 14.4 Spatial overlap and halo overhead

If neighboring output tiles require overlapping input neighborhoods, the useful-input-to-total-input ratio depends on tile shape.

```text
FOR spatial_output_tile:
    required_input_region ← EXPAND_BY_DEPENDENCY_HALO(spatial_output_tile)
    LOAD_REGION(required_input_region)
    PRODUCE_OUTPUTS_IN_TILE(spatial_output_tile)
```

Increasing spatial tile dimensions can amortize halo overhead, but beyond the cache budget the larger input region may become counterproductive. A narrow tile in the contiguous direction may also underutilize cache lines.

---

## 15. Fusing stages and deciding where results live

Loop restructuring is not limited to a single computation. If several stages consume each other's results, fusion can eliminate intermediate memory traffic.

### 15.1 Separate stages

```text
FOR each item:
    INTERMEDIATE[item] ← STAGE_ONE(item)

FOR each item:
    OUTPUT[item] ← STAGE_TWO(INTERMEDIATE[item])
```

### 15.2 Fused stages

```text
FOR item_tile:
    temporary ← STAGE_ONE_FOR_TILE(item_tile)
    result ← STAGE_TWO_FOR_TILE(temporary)
    STORE_RESULT(result)
```

**Potential benefit:** the intermediate may never be written to main memory.  
**Potential cost:** live temporary state and register pressure increase; stage-specific optimal tile shapes may conflict; a stage may benefit from a global pass or from sharing data across tiles.

### 15.3 Fusion is not always legal

Fusion must preserve dependencies. If stage two needs data that stage one has not yet produced—for example, neighboring outputs or a global statistic—the fused tile may require a halo, a look-ahead region, a multi-pass schedule, or a separate intermediate.

### 15.4 A useful rule

Fuse adjacent stages when intermediate values are short-lived and local **and** the combined register/cache footprint still fits. Keep stages separate when they have incompatible reuse scales, require global synchronization, or create excessive spills when fused.

---
## 16. A cost model for comparing candidate loop architectures

Do not optimize loop order by intuition alone. Start with a deliberately simple model that highlights major trade-offs, then refine it with measurements.

### 16.1 Separate arithmetic, memory, and overhead

For each candidate, estimate:

```text
work_count           ← NUMBER_OF_LOGICAL_EVALUATIONS()
vector_operation_count ← WORK_MAPPED_TO_VECTORS()
slow_memory_bytes    ← ESTIMATED_BYTES_FROM_LOWER_MEMORY_LEVELS()
state_traffic_bytes  ← ESTIMATED_PARTIAL_OUTPUT_LOADS_AND_STORES()
repacking_bytes      ← ESTIMATED_LAYOUT_CONVERSION_TRAFFIC()
shuffle_work         ← ESTIMATED_LANE_REORGANIZATION()
setup_work           ← TILE_SETUP_AND_LOOP_CONTROL()
spill_risk           ← ESTIMATE_PEAK_LIVE_STATE()
active_lane_fraction ← USEFUL_LANES / EXECUTED_LANES
```

A useful first-order lower bound is:

```text
compute_time_bound ← required_operations / sustainable_compute_rate
memory_time_bound  ← required_bytes / sustainable_memory_bandwidth

idealized_time_lower_bound ← MAX(compute_time_bound, memory_time_bound)
```

Real execution also has latency, dependency chains, instruction issue limits, cache conflicts, branch effects, page translation, synchronization, and setup overhead. The bound helps identify the likely bottleneck; it does not predict exact runtime.

### 16.2 Three levels of locality

A practical hierarchy is:

1. **Registers:** best for live accumulator state and repeated small operands.
2. **Near-core cache:** best for input regions or intermediate tiles that are too large for registers.
3. **Larger caches and memory:** unavoidable for large datasets, but expensive when revisited needlessly.

Which exact storage level dominates depends on the hardware and workload. A tile can fit by capacity yet perform poorly because of cache associativity, multiple input streams, or conflicts. Treat capacity estimates as upper bounds, not guarantees.

### 16.3 Compare *marginal* tile improvements

Suppose increasing `B_O` from `B1` to `B2` reduces repeated shared-input loads. The larger tile is beneficial only if:

```text
cost_saved_by_input_reuse
+
cost_saved_by_fewer_outer_iterations
>
cost_added_by_register_pressure
+
cost_added_by_spills_or_permutations
+
cost_added_by_worse_tail_behavior
+
cost_added_by_other_resource_pressure
```

This comparison is more useful than a rule such as “choose the largest tile that fits the vector width.”

### 16.4 Do not overvalue one metric

A candidate with more arithmetic can be faster if it avoids irregular memory accesses. A candidate that loads fewer logical bytes can be slower if its loads touch more cache lines. A wider vector can be slower if it requires more rearrangement, reduces available parallel instruction streams, or triggers a different operating regime. Measure full-stage throughput, not just the number of vector instructions.

---

## 17. A complete architecture-design procedure

This procedure is intended to be repeated for any compute-intensive nested loop.

### Step 1 — Write the exact scalar semantics

```text
FOR each legal logical coordinate:
    perform the specified reads
    compute the specified contribution
    update the specified destination
    apply the specified finalization and boundary rules
```

State the legal input domain, output domain, dependencies, type widths, overflow behavior, rounding, ordering, and tie-breaking.

### Step 2 — Identify the independent axes

Separate output coordinates, reduction coordinates, independent problems, and any auxiliary dimensions. Mark which axes are contiguous in storage and which involve data-dependent indexing.

### Step 3 — Enumerate legal lane mappings

At minimum, consider:

```text
MAPPING_A ← different output per lane
MAPPING_B ← different reduction contribution per lane
MAPPING_C ← different independent problem per lane
MAPPING_D ← mixed lane grouping, only if useful
```

Eliminate mappings that violate dependencies or require unacceptable changes in numerical behavior.

### Step 4 — Choose a stationary resource

Create at least three architectural candidates:

```text
CANDIDATE_1 ← output-stationary
CANDIDATE_2 ← input-stationary
CANDIDATE_3 ← problem-stationary
```

For each one, explicitly mark which state is kept in registers, which data is intended to stay cached, and which values are repeatedly moved through memory.

### Step 5 — Design the tile hierarchy

Choose preliminary sizes for output, reduction, problem, and spatial tiles. Respect:

```text
register_state_fits
cache_working_set_is_reasonable
accumulators_do_not_overflow
valid_dependencies_are_preserved
lane_utilization_is_acceptable
```

A usable tile hierarchy may contain more than three loops. A four- or five-level tiled implementation is not inherently worse than a three-level nest: the important question is the cost of work inside the loops.

### Step 6 — Choose the within-tile traversal

Ask whether input loads can be reused across lanes, whether addresses are contiguous, and whether loads or computation can be overlapped. Keep the innermost update as regular as possible without losing valuable input reuse.

### Step 7 — Specify tails and unusual cases

Handle incomplete output tiles, incomplete reduction chunks, short or empty problems, irregular bounds, masked lanes, data-dependent branches, and all required boundary behavior.

### Step 8 — Measure a small architecture matrix

Vary one major architectural dimension at a time, then test selected combinations. If an output-stationary design wins, explore its register footprint and reduction chunking rather than testing arbitrary permutations forever.

### Step 9 — Validate exact semantics

Compare results with a trustworthy reference using the chosen numerical contract. Include edge cases designed to exercise partial tiles, overflow boundaries, and equality cases.

### Step 10 — Optimize the whole computation

A locally fastest inner kernel might create expensive setup, repacking, intermediate arrays, or loss of parallelism elsewhere. Check end-to-end performance after the kernel-level experiments.

---

## 18. A purely abstract worked architecture comparison

The following example is intentionally **not** a specific algorithm. Assume:

- `P_COUNT` independent problems.
- Each problem has `O_COUNT = 96` independent outputs.
- Each output combines `R_COUNT = 64` contributions.
- Each contribution uses one item `SHARED[p, r]` and one output-specific item `SPECIFIC[p, o, r]`.
- The hypothetical execution model processes `V = 16` outputs per vector.
- The practical budget for live accumulator vectors is approximately four after reserving space for other values.

The reference semantics are:

```text
FOR p:
    FOR o FROM 0 TO 95:
        state ← IDENTITY
        FOR r FROM 0 TO 63:
            state ← COMBINE(state,
                            EVALUATE(SHARED[p, r], SPECIFIC[p, o, r]))
        OUTPUT[p, o] ← state
```

### 18.1 Candidate A: one output at a time

```text
FOR p:
    FOR o:
        state ← IDENTITY
        FOR r:
            state ← COMBINE(state, EVALUATE(SHARED[p, r], SPECIFIC[p, o, r]))
        OUTPUT[p, o] ← state
```

Benefits: very small accumulator state; straightforward semantics.  
Costs: `SHARED[p, r]` may be revisited for every `o`, though a small shared array may stay hot in cache. Vectorization across reduction contributions would require a different mapping and possibly a final lane reduction.

### 18.2 Candidate B: output tile of 32

```text
FOR p:
    FOR o0 IN {0, 32, 64}:
        state[32] ← IDENTITIES()

        FOR r:
            shared ← SHARED[p, r]
            specific[32] ← LOAD_SPECIFIC_OUTPUT_TILE(p, o0, r)
            state ← COMBINE_LANEWISE(state, EVALUATE(shared, specific))

        STORE_OUTPUT_TILE(p, o0, state)
```

There are two logical 16-lane accumulator vectors. Each shared item is used by 32 outputs before advancing to the next reduction step, but the shared input is traversed three times per problem.

### 18.3 Candidate C: output tile of 64

```text
FOR p:
    FOR o0 IN {0, 64}:
        width ← MIN(64, 96 - o0)
        state[width] ← IDENTITIES()

        FOR r:
            shared ← SHARED[p, r]
            specific[width] ← LOAD_SPECIFIC_OUTPUT_TILE(p, o0, r)
            state ← COMBINE_LANEWISE(state, EVALUATE(shared, specific))

        STORE_OUTPUT_TILE(p, o0, state)
```

The first tile has four 16-lane accumulator vectors, near the assumed register budget. The second has only 32 valid outputs. Compared with the 32-output tile, the shared input is traversed twice rather than three times. But this benefit matters only to the extent that the shared access is actually expensive.

### 18.4 Candidate D: output tile of 96

```text
FOR p:
    state[96] ← IDENTITIES()
    FOR r:
        shared ← SHARED[p, r]
        specific[96] ← LOAD_ALL_SPECIFIC_OUTPUTS(p, r)
        state ← COMBINE_LANEWISE(state, EVALUATE(shared, specific))
    STORE_ALL_OUTPUTS(p, state)
```

The shared input is traversed once, but six accumulator vectors are required before counting temporary vectors. Under the hypothetical four-vector budget, spilling is likely. Candidate D may therefore be slower than Candidate B or C despite better apparent reuse.

### 18.5 Candidate E: one reduction tile at a time

```text
FOR p:
    state[96] ← INITIALIZE_IN_MEMORY()

    FOR r0 BY 8:
        shared_tile ← LOAD_SHARED_TILE(p, r0, 8)

        FOR o0 BY 32:
            partials ← LOAD_STATE_TILE(state, o0, 32)
            partials ← PROCESS_EIGHT_CONTRIBUTIONS(
                partials, shared_tile, p, o0, r0)
            STORE_STATE_TILE(state, o0, partials)

    FINALIZE_AND_STORE(state)
```

There are eight reduction tiles. If partial states really reside in memory, each output state is loaded and stored about eight times, instead of being written once at completion in the output-stationary candidates. That extra state traffic must be justified by input reuse or other savings.

### 18.6 What this comparison teaches

| Candidate | Shared-input traversals per problem | Nominal accumulator vectors kept live | Main concern |
|---|---:|---:|---|
| A: one output | Up to 96 | 1 scalar state | Repeated shared-input access |
| B: 32-output tile | 3 | 2 | More outer tiles |
| C: 64-output tile | 2 | 4 for main tile | Near register budget |
| D: 96-output tile | 1 | 6 | Likely spilling under stated budget |
| E: 8-step reduction tiles | 1 | 2 per active output tile | Repeated partial-state memory traffic |

These are **structural estimates**, not measured timings. Cache residency, specific-data layout, issue throughput, compiler decisions, and other live values may change the ranking. The example shows why a superficially optimal outer-loop choice can fail once resource constraints are included.

---

## 19. Benchmarking loop architectures without misleading yourself

A benchmark should reveal the actual cost of architectural choices, not mostly process startup, allocation, cold cache, or random machine-state variation.

### 19.1 Control the experiment

Hold constant:

- Logical work and input values.
- Output requirements and numerical tolerance.
- Input layout, unless layout is the explicit variable being tested.
- Number of workers and ownership partition.
- Compiler optimization assumptions or execution environment.
- Warmup, timing boundaries, and result validation.

Evaluate both a **kernel-only** measurement and an **end-to-end** measurement. Kernel-only timing reveals the loop-nest effect; end-to-end timing captures data conversion, scheduling, allocations, and other overhead.

### 19.2 Use repeated, interleaved trials

```text
variants ← [baseline, output_tile_small, output_tile_large,
            reduction_tile, problem_tile]

FOR each variant:
    VERIFY_RESULT_AGAINST_REFERENCE(variant)
    WARM_UP(variant)

FOR trial IN many_trials:
    order ← CONTROLLED_PERMUTATION(variants, trial)
    FOR variant IN order:
        elapsed ← TIME_COMPLETE_FIXED_WORKLOAD(variant)
        RECORD(variant, trial, elapsed)

FOR variant:
    REPORT_MEDIAN_AND_VARIABILITY(variant)
    REPORT_WORK_NORMALIZED_THROUGHPUT(variant)
```

Interleaving reduces the chance that one variant is consistently measured during a faster or slower machine regime. For very short kernels, time sufficiently many repeated operations per sample and make sure the work cannot be eliminated or simplified by the execution environment.

### 19.3 Test more than one problem size

A good test matrix varies:

| Dimension | What to include |
|---|---|
| Output count | Small, moderate, large, not divisible by `V` |
| Reduction length | Very short, medium, long, not divisible by `B_R` |
| Number of independent problems | Fewer than `V`, around `V`, much larger than `V` |
| Access pattern | Contiguous, strided, irregular |
| Reuse | No shared input, moderate reuse, heavy reuse |
| Data footprint | Fits in registers, fits in cache, exceeds cache |
| Control flow | Uniform and irregular work lengths |
| Numerical cases | Zero, extremes, overflow boundaries, ties, special values |

Measure steady-state throughput and, where relevant, latency for a single problem. An architecture optimized for large batches may be poor for a low-latency single-task path.

### 19.4 Use an ablation matrix

When performance improves, isolate *why*:

```text
EXPERIMENT_1: change loop order only
EXPERIMENT_2: change output tile width only
EXPERIMENT_3: change reduction chunk width only
EXPERIMENT_4: change lane mapping only
EXPERIMENT_5: change data layout only
EXPERIMENT_6: change number of independent accumulators only
EXPERIMENT_7: combine the strongest individually validated changes
```

The best combined result is not guaranteed to equal the sum of individual wins. Optimizations interact, especially through register pressure and cache reuse.

### 19.5 Avoid an attractive but false conclusion

A performance spike at one problem size does not automatically prove an algorithmic boundary bug. It may come from spills, cache mapping, code-path changes, dynamic frequency, scheduling, or measurements taken in different machine states. Repeat targeted tests under controlled conditions before attributing anomalies to a specific loop structure.

---

## 20. Common failure modes and their corrections

| Failure | Why it happens | Better design question |
|---|---|---|
| Vectorize the innermost scalar loop without analysis | Scalar nesting may prioritize an unrelated locality objective | Which logical axis gives independent lanes with the least data movement? |
| Make output tile as wide as possible | Accumulator and temporary state exceed register capacity | What is the peak *live* state, not just output count? |
| Put reduction loop outermost to reuse input | Output partial sums are reread and rewritten | Where will all partial outputs reside across reduction tiles? |
| Optimize only arithmetic instructions | Loads, gathers, packing, and shuffles dominate | What is the total cost of data access and lane reorganization? |
| Use one tile shape for every size | Different regimes favor different mappings | Should small and large extents use separate strategies? |
| Ignore incomplete tiles | Inactive lanes waste work or trigger invalid memory access | What exact mask, remainder, or fallback handles each boundary? |
| Assume vector reduction preserves scalar results | Reassociation changes rounding, saturation, or tie order | What numerical contract must every variant satisfy? |
| Repack all data unconditionally | Conversion cost can exceed the benefit | How many downstream uses amortize the repacking? |
| Fuse every neighboring stage | Live data and dependency scope become too large | Does the fused working set still fit the intended storage level? |
| Use maximum unrolling | Instruction footprint and register usage explode | How many independent chains are actually needed? |
| Parallelize every available axis | More workers may increase contention, contention for bandwidth, and synchronization | What axis provides independent output ownership and sufficient work per worker? |
| Trust one benchmark run | Machine state and measurement noise distort the result | Do repeated interleaved trials agree? |

### 20.1 Task parallelism as another outer layer

Independent problems can often be assigned to workers while each worker executes its own SIMD-tiled kernel:

```text
PARTITION independent problems into disjoint worker ranges

PARALLEL FOR each worker_range:
    worker_local_scratch ← ALLOCATE_ONCE()

    FOR problem IN worker_range:
        RUN_SIMD_TILE_ARCHITECTURE(problem, worker_local_scratch)

    RELEASE(worker_local_scratch)
```

The worker layer should not force threads to update a common output tile without a deliberate synchronization strategy. Worker-local scratch avoids unnecessary sharing, and output partitioning should minimize interference between workers writing adjacent locations. More workers are not necessarily faster when bandwidth or shared caches are already saturated.

---

## 21. A reusable architecture specification template

Copy this section into a new design note before implementing a SIMD version of any algorithm.

```text
ARCHITECTURE SPECIFICATION

1. SEMANTICS
   Logical inputs:
   Logical outputs:
   Per-output contribution rule:
   Required evaluation order:
   Numerical and tie-breaking contract:
   Boundary conditions:

2. ITERATION SPACE
   Independent-problem axes:
   Output axes:
   Reduction axes:
   Auxiliary axes:
   Variable-size dimensions:

3. DEPENDENCIES
   Independent outputs:
   Cross-output reads and writes:
   Associativity and commutativity assumptions:
   Shared writable state:

4. DATA LAYOUT
   Fastest-varying input axis:
   Fastest-varying output axis:
   Common input shared between outputs:
   Address regularity:
   Data conversion requirements:

5. SIMD LANE MAPPING
   Meaning of one lane:
   Logical lanes per operation:
   Horizontal reduction required:
   Inactive-lane behavior:
   Cross-lane rearrangement required:

6. TILE OWNERSHIP
   Outermost tile selects:
   Accumulators are owned by:
   Inputs expected to be reused:
   Maximum live state:
   Chosen tile sizes:
   Reasons for rejecting alternatives:

7. INNER LOOP
   Order of within-tile axes:
   Main update:
   Reduction flush points:
   Unrolling and independent chains:
   Tail processing:

8. CORRECTNESS
   Reference comparison:
   Extreme-value tests:
   Tail and boundary tests:
   Reproducibility or tolerance policy:

9. PERFORMANCE
   Estimated input traffic:
   Estimated partial-output traffic:
   Estimated register footprint:
   Expected bottleneck:
   Benchmark variants:
   Problem-size regimes:

10. DECISION
    Chosen architecture:
    Measured evidence:
    Known limitations:
    Fallback architecture:
```

### 21.1 A reusable architecture pseudocode skeleton

```text
PROCEDURE GENERIC_SIMD_ARCHITECTURE(inputs, outputs, configuration):
    SELECT_LANE_MAPPING(configuration)
    SELECT_TILE_HIERARCHY(configuration)

    FOR outer_tile IN PRIMARY_REUSE_DIMENSION:
        PREPARE_TILE_LOCAL_STATE(outer_tile)

        FOR middle_tile IN SECONDARY_REUSE_DIMENSION:
            STAGE_REUSED_INPUTS_IF_PROFITABLE(middle_tile)

            FOR inner_tile IN REMAINING_ITERATION_SPACE:
                valid_lanes ← COMPUTE_VALID_LANE_MASKS(inner_tile)
                values ← LOAD_ONLY_VALID_INPUTS(inner_tile, valid_lanes)
                contributions ← EVALUATE_LANEWISE(values)
                UPDATE_OWNED_PARTIAL_RESULTS(contributions, valid_lanes)

            FLUSH_NARROW_PARTIALS_WHEN_REQUIRED()

        FINALIZE_AND_STORE_OWNED_OUTPUTS()

    PROCESS_ANY_REQUIRED_FALLBACKS()
```

This is a *design skeleton*, not a prescription to literally implement every named step. Some architectures merge stages, some eliminate staging, and some require the outer tile to select reduction inputs rather than outputs. The key is that ownership and reuse are explicit.

---

## 22. Final design rules and mental models

### 22.1 Ten rules worth remembering

1. **Vector lanes are an architectural assignment, not a consequence of loop syntax.** Decide what each lane represents before choosing a loop order.
2. **The most important loop is often the outermost tile loop.** It determines which data gets reused before the computation moves elsewhere.
3. **Keep accumulators stationary when their reload cost dominates.** A tile is useful only if its live state fits the intended fast storage.
4. **Keep inputs stationary when shared-input reuse outweighs state traffic.** Counting input loads alone is insufficient.
5. **Tile dimensions are independent of vector width.** One logical tile may contain several vectors; one vector may encode several logical groups.
6. **Contiguous access can matter more than fewer arithmetic operations.** Choose lane mappings using actual physical data layout.
7. **Reduction ordering is a correctness decision.** Overflow, rounding, and tie-breaking must be part of the design specification.
8. **Incomplete tiles are not an afterthought.** Tail frequency and safety can change the best architecture.
9. **Bigger tiles, wider vectors, more unrolling, and more fusion can all be slower.** Each consumes resources and changes data movement.
10. **Benchmark controlled alternatives.** Architecture selection is a measurement-driven search over legal mappings and tile hierarchies.

### 22.2 The one-page decision sequence

```text
START WITH EXACT SEMANTICS
    ↓
IDENTIFY OUTPUT, REDUCTION, PROBLEM, AND SPATIAL AXES
    ↓
MAP DEPENDENCIES AND LEGAL REORDERINGS
    ↓
CHOOSE WHAT EACH SIMD LANE REPRESENTS
    ↓
SELECT WHAT SHOULD STAY STATIONARY:
    OUTPUTS / INPUTS / PROBLEMS
    ↓
CHOOSE OUTERMOST TILE AND ITS OWNERSHIP
    ↓
FIT ACCUMULATORS IN REGISTERS
    ↓
FIT REUSED INPUTS IN CACHE WHERE PRACTICAL
    ↓
CHOOSE INNER TRAVERSAL FOR CONTIGUITY AND REUSE
    ↓
HANDLE REDUCTION WIDTHS, NUMERICAL ORDER, AND TAILS
    ↓
BENCHMARK A CONTROLLED SET OF ALTERNATIVES
    ↓
VALIDATE FULL RESULTS AND END-TO-END PERFORMANCE
```

### 22.3 The governing idea

The purpose of efficient SIMD architecture is not merely to perform more arithmetic simultaneously. It is to arrange the work so that **useful operands arrive together, independent results stay live long enough, and expensive movement is minimized**.

The optimal nesting of loops is therefore a property of the **iteration-space dependencies, physical data layout, reuse pattern, register budget, and execution environment together**. It is not determined by the mathematical operator, the number of loops in the reference, or the width of a vector alone.

---

## Appendix A — Glossary

| Term | Meaning |
|---|---|
| Accumulator residency | Keeping partial result state in fast local storage until completion or a planned flush |
| Arithmetic intensity | Useful arithmetic work relative to data moved between specified storage levels |
| Broadcast | Making one logical value available to multiple lanes |
| Data layout | Physical ordering of values in memory |
| Gather | Loading values from noncontiguous addresses into lanes |
| Halo | Extra input region required around a tile to compute its outputs |
| Horizontal reduction | Combining values from several SIMD lanes into one or fewer results |
| Input-stationary | Prioritizing reuse of an input tile across multiple outputs |
| Lane mapping | The meaning assigned to each SIMD lane |
| Lane utilization | Fraction of processed lanes that perform useful logical work |
| Loop interchange | Changing the nesting order of two or more loop axes |
| Loop strip-mining | Splitting an iteration axis into a tile index and an index within the tile |
| Mask | Selection of valid or active lanes |
| Output-stationary | Keeping a bounded output tile's partial results live while consuming its inputs |
| Problem-stationary | Keeping one independent problem or batch local while its work is completed |
| Reduction chunk | Bounded group of contributions accumulated before merging or flushing |
| Register pressure | Demand for simultaneous live fast-storage values |
| Repacking | Rearranging input data to make lane access more efficient |
| Scatter | Writing lane results to noncontiguous destinations |
| Spill | Moving live state to slower storage because fast storage is insufficient |
| Tail | Incomplete final tile or vector group |
| Temporal locality | Reusing the same data before it is evicted from fast storage |
| Tile | Bounded part of a multi-dimensional iteration space processed as one ownership unit |
| Tile hierarchy | Nested decomposition of axes into successively smaller units of work |
| Vector width | Number of lanes available for the active type and execution model |

## Appendix B — What this document deliberately does not assume

This handbook does not assume that:

- The operation is an absolute difference, multiplication, addition, filter, match, or any other particular function.
- The data is an image, signal, matrix, geometric structure, or another particular domain.
- The target machine supports a named vector instruction family.
- SIMD vectors contain a fixed number of bytes or elements.
- All inputs are contiguous or aligned.
- The reduction is associative or may be freely reordered.
- All problems have equal dimensions.
- The available register and cache capacities are known in advance.
- The best solution is a single universal kernel.

The principles are intended to guide the **design and evaluation of loop architectures** independently of the computation placed inside their abstract `EVALUATE`, `COMBINE`, and `FINALIZE` operations.