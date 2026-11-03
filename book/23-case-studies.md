# 23. Fifteen abstract before/after case studies

[Previous](22-portability.md) · [Contents](README.md) · [Next](24-practice.md)

This chapter deliberately uses **generic operations** rather than algorithms named after a domain. The goal is to train an architectural reflex: recognize the dependency structure, choose the owners of live data, transform the schedule, and identify the condition under which the transformation would *lose*.

Every case has: (1) original computation, (2) vector-oriented candidate, (3) why it may win, (4) what can make it worse, and (5) a legality assumption. A candidate is a hypothesis, not a claim that it is the fastest implementation.

---

## Case 1 — Independent element map: one logical loop

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

## Case 2 — Two-dimensional independent map: collapse versus nested lanes

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

## Case 3 — One long reduction: split the dependency chain

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

## Case 4 — Many outputs × one reduction: keep output lanes stationary

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

## Case 5 — Three axes P × O × R: outermost-tile decision

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

## Case 6 — Four axes with a shared inner dimension: strip mine, then jam

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

## Case 7 — Many outputs read overlapping input neighborhoods

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

## Case 8 — Prefix scan: transform one serial chain into local parallel work

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

## Case 9 — Independent recurrences: vectorize across streams

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

## Case 10 — Two-dimensional dependency: compute wavefronts

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

## Case 11 — Conditional transformation: branch to mask

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

## Case 12 — Filter/compact: convert control-dependent writes into a prefix problem

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

## Case 13 — Indirect scatter reduction: reorganize by destination

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
    FOR edge IN group IN ORIGINAL_TARGET_ORDER
        acc = COMBINE(acc,VALUE(edge))
    STORE_FINAL_OUTPUT(output,target,acc)
~~~

**Why it may win:** fewer random output writes and a regular reduction loop per destination. **Why it may lose:** grouping/sorting cost and uneven groups. **Legal if:** cross-target order is unobservable, groups preserve per-target contribution order, and each group begins with that target's original initialized output. This retains the per-target sequence even for a nonassociative COMBINE; starting from IDENTITY and merging the group into a preexisting output once would instead require stronger algebraic assumptions.

**Alternative:** vectorize value preparation but apply writes sequentially when updates are nonassociative.

## Case 14 — Multi-stage pipeline: recompute instead of materialize

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

## Case 15 — Six-dimensional work: compose hierarchical tiles, not 720 permutations

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

## Cross-case design lessons

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

## Reusable case-study worksheet

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
