# 10. Predication, divergence, masks, and tails

[Previous](09-lane-communication.md) · [Contents](README.md) · [Next](11-irregular-work.md)

SIMD performs best when multiple lanes follow a common control path. Divergent tasks can still be vectorized by turning control flow into data flow, grouping similar tasks, or compacting active work.

## 10.1 Predication and masked select

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

## 10.2 Masked arithmetic is not automatically a safe memory access

A masked result is not proof that invalid pointers are never dereferenced. Keep address validation and memory operation semantics distinct:

~~~text
valid = IN_BOUNDS(indices)
safe_indices = SELECT(valid, indices, SAFE_FALLBACK_INDICES)
values = SAFE_GATHER(data, safe_indices, valid)
~~~

The abstract **SAFE_GATHER** contract must guarantee no illegal access for inactive lanes. Simply loading from invalid addresses and zeroing afterward is incorrect. A fallback address must itself be valid; an explicit predicated load can be another implementation.

## 10.3 Three architectures for variable work

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

## 10.4 Tail handling

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

## 10.5 Expensive branches and side effects

Predication is an attractive replacement for branch-heavy arithmetic, but it may be wrong to evaluate both branches if either has side effects, traps, observable exceptions, or invalid memory accesses.

~~~text
IF ANY(mask_A)
    EXECUTE_A_FOR_ACTIVE_LANES(mask_A)
IF ANY(mask_B)
    EXECUTE_B_FOR_ACTIVE_LANES(mask_B)
~~~

This pattern executes only the needed branch kernels and may be preferable when one branch is rare and expensive.

## 10.6 Workload grouping break-even

Grouping helps when:

\[
\text{saved inactive-lane and branch cost}
>
\text{classification}+\text{movement}+\text{restore-order cost}
\]

A grouping strategy should be compared on *total end-to-end work*, including preprocessing and output restoration.

**Takeaway:** the fastest divergent architecture often changes **which tasks share a vector**, not merely how branch instructions are expressed.
