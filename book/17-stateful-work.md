# 17. Stateful, variable-length, and convergent algorithms

[Previous](16-sparse-segmented.md) · [Contents](README.md) · [Next](18-pipelines.md)

A task that evolves its own state is often sequential *within the task* but independent of other tasks. The important transformation is to vectorize the **population of state machines**, not to parallelize illegal state transitions.

## 17.1 Reference semantics: one independent state per task

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

## 17.2 Task lanes and lockstep execution

~~~text
FOR task_group IN TILES(TASKS, V)
    state = VECTOR_INITIAL_STATE(task_group)
    step  = ZERO_VECTOR
    # Some tasks may already be terminal at their initial state.
    active = VALID_LANES(task_group)
             AND NOT VECTOR_TERMINAL(state, task_group, step)

    WHILE ANY(active)
        next_input = SAFE_LOAD_NEXT_INPUT(task_group, step, active)
        proposed = VECTOR_TRANSITION(state, next_input, active)
        state = SELECT(active, proposed, state)
        step = MASKED_INCREMENT(step, active)
        active = active AND NOT VECTOR_TERMINAL(state, task_group, step)

    STORE_FINALIZED_STATES(task_group, state)
~~~

Every lane retains the state of one task. The inner iterations remain dependent, so peak vector arithmetic is limited by latency of TRANSITION and the number of independent task lanes.

## 17.3 The long-tail problem

Suppose task lengths are [2, 2, 2, 40]. A vector with four lanes performs 40 lockstep iterations even though only one lane is active after iteration 2. The useful-lane fraction becomes very low.

Options:

- **Group by expected length:** sort or bucket tasks so durations are similar.
- **Dynamic compaction:** move active states together after lanes finish.
- **Work stealing at lane granularity:** refill completed lanes from a task queue.
- **Multiple vectors in flight:** improve latency hiding while waiting for memory-dependent transitions.

All four increase bookkeeping. The best strategy depends on state size, transition cost, and whether work length can be predicted.

## 17.4 Dynamic refill

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

## 17.5 Convergence and fixed points

A task may terminate when an error/residual falls below a threshold:

~~~text
WHILE NOT CONVERGED(state)
    state = ITERATE(state)
~~~

Lockstep vector execution is straightforward with masks, but numerical convergence may differ if the vectorized ITERATE changes operation order. Per-lane stopping preserves each task's semantics only if masked lanes stop receiving updates.

## 17.6 Batched state with common setup

Tasks may share immutable parameters:

~~~text
FOR batch IN GROUP_BY_SHARED_PARAMETER(TASKS)
    common = LOAD_SHARED_PARAMETER(batch)
    states = INITIALIZE_TASK_STATES(batch)

    WHILE ANY_ACTIVE(states)
        states = VECTOR_STEP(states, common)
~~~

The batch can reduce parameter and lookup overhead. If groups are tiny, it may reduce occupancy; prefer a hybrid grouping that balances common setup against active lanes.

## 17.7 State representation matters

A state with many fields can be stored as records, field-wise arrays, or blocks of records. Field-wise storage often suits many task lanes, but repeated per-task control decisions may favor packed task records. Consider the full set of fields touched by each transition, not just the most frequently updated one.

## 17.8 When can time steps be vectorized?

Only after proving stronger properties, such as:

- Transitions compose associatively into a compact representation.
- Dependencies split into independent chains.
- A wavefront exposes ready steps.
- A speculative schedule can validate and repair violations within the required semantics.

Without such a proof, vectorizing across ordered steps changes the algorithm.

## 17.9 Correctness and performance tests

Test zero-step tasks, immediate termination, maximum-length tasks, mixed lengths, compaction/refill boundaries, task identity restoration, and deterministic stopping rules. Report the distribution of active lanes over time, not just total runtime.

**Takeaway:** evolving-state algorithms become SIMD-friendly by **interleaving independent histories** while preserving each history's dependencies.
