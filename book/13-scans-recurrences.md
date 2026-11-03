# 13. Scans, recurrences, and dependency chains

[Previous](12-maps-reductions.md) · [Contents](README.md) · [Next](14-neighborhoods-wavefronts.md)

A reduction produces one summary. A **scan** produces an intermediate summary at *every* position. A **recurrence** evolves a state. Both initially appear serial, yet certain algebraic and scheduling transformations expose vector parallelism.

## 13.1 Inclusive scan

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

## 13.2 Proof sketch for blocked scan

Let the elements in the current block be b0, b1, ..., bk. Its j-th local prefix is b0 ⊕ ... ⊕ bj. Let carry be the prefix of every element before this block. By associativity:

carry ⊕ (b0 ⊕ ... ⊕ bj)

equals the full prefix through bj. Update carry using the last valid local prefix. If the reference defines exact floating-point addition order, the transformed expression may differ because of rounding; then exact equivalence is **not** guaranteed.

## 13.3 Exclusive scan

An exclusive scan writes the prefix **before** the current input:

~~~text
state = IDENTITY
FOR i
    output[i] = state
    state = COMBINE(state, input[i])
~~~

A vector scan can compute an inclusive scan and shift its results by one position while inserting the carry from the previous block. Ensure the first lane receives the correct identity/carry and that the final block uses only active elements.

## 13.4 A general recurrence

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

## 13.5 Composable transitions

If every step can be represented by a transformation **T_i**, and transformation composition is associative, the state after k steps is:

\[
s_k = (T_k \circ T_{k-1} \circ \dots \circ T_0)(s_{\text{initial}})
\]

This enables a scan of transformations—*not* necessarily a scan of scalar input values.

~~~text
FOR step
    op[step] = ENCODE_TRANSFORMATION(input[step])

composed_prefix = ASSOCIATIVE_SCAN(op, COMPOSE)

FOR step
    state[step] = APPLY(composed_prefix[step], initial_state)
~~~

The transformed representation may be larger than the original state. The parallel scan helps only when transition composition and application are cheaper than the avoided sequential dependency.

## 13.6 Parallelism across independent recurrences

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

## 13.7 Recurrence distance and pipelining

A recurrence may depend on a state several steps earlier instead of only the previous step. If the dependency graph splits into independent residue classes, process those classes as separate lanes:

~~~text
FOR group_of_independent_chains
    states = LOAD_CHAIN_STARTS(group_of_independent_chains)
    FOR chain_step IN ORDER
        states = VECTOR_TRANSITION(states, chain_step)
    STORE_CHAIN_ENDS(states)
~~~

Never assume independence solely because indices differ; prove no cross-chain edges.

## 13.8 When scans lose

A scan transformation can be slower than a serial fold because of lane shuffle cost, operator representation overhead, tiny input size, or memory bandwidth saturation. Cross-block carries remain sequential in the simple blocked algorithm. A hierarchical scan can parallelize block totals at the cost of extra passes.

**Takeaway:** ask whether a dependency is **a required order** or **a structure that can be represented by associative composition**.
