# 24. Design worksheet, exercises, and answer sketches

[Previous](23-case-studies.md) · [Contents](README.md) · [Glossary](glossary.md)

This chapter turns the handbook into a repeatable design practice. First specify the computation, then derive legal architectures, then narrow candidates using resource reasoning, and finally measure.

## 24.1 Copyable architecture-specification template

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

## 24.2 The 12-step design method

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

## 24.3 Exercises

### Exercise 1 — One loop, small N

A pure independent map operates on N=5 elements with a logical vector width of 16. Should a vector kernel always win?

**Hint:** count active lanes and include setup and tail handling.

### Exercise 2 — Two axes, short reduction

There are 100,000 outputs, each with 3 contributions. Compare output-lane and reduction-lane mapping.

**Hint:** how many horizontal reductions are needed if each output uses its own reduction vector?

### Exercise 3 — Two axes, long reduction

There are only two outputs, each with 100,000 contributions. What changes?

**Hint:** output lanes may underfill a wide vector; independent contribution chains could help, subject to arithmetic laws.

### Exercise 4 — Three axes, shared inputs

Problems P contain O outputs, and all O outputs reuse each input for reduction index r. Compare P–O–R versus P–R–O. What extra state does the second order require?

### Exercise 5 — Four axes, short inner dimensions

A computation has independent axes A and B and output axes C and D, each of length 2. Explain how to fill a width-16 vector without pretending either C or D is a reduction axis.

### Exercise 6 — A dependent scan

An inclusive scan uses a mathematically associative operation. Show how to process blocks in parallel internally and carry results between blocks. What changes if the operation is floating-point addition with bit-exact output requirements?

### Exercise 7 — Colliding scatter

Several lanes update the same destination using COMBINE. State two legal strategies when COMBINE is associative, and one conservative strategy when it is not.

### Exercise 8 — Ragged state machines

Task lengths are [2, 2, 2, 40]. Compare lockstep execution, sorting by length, and lane refill. Include preprocessing cost.

### Exercise 9 — Neighborhood patch

A two-dimensional neighborhood has radius 2. Compare halo overhead for output tile shapes (8,8) and (32,8). Why isn't the larger tile automatically best?

### Exercise 10 — Register pressure

An output-stationary microtile doubles its output count and its number of independent accumulator chains. What happens to live accumulator demand? Which other resource costs may improve or worsen?

### Exercise 11 — Fusing stages

Two stages use incompatible lane layouts. Give one reason fusion helps and two reasons fission could win.

### Exercise 12 — Recompute or materialize

A reusable intermediate is inexpensive to compute but expensive to load and rearrange. What measurements establish whether recomputing it inside each consumer tile is profitable?

### Exercise 13 — Ordered dependence graph

Each point (i,j) depends on (i−1,j) and (i,j−1). Derive a wavefront index and specify which points in a wavefront can execute together.

### Exercise 14 — Full multidimensional architecture

Given two problem axes, two output axes, and two reduction axes, propose: (a) a register tile, (b) a cache tile, and (c) a parallel task tile. Explain which axis is stationary at each level.

### Exercise 15 — Misleading benchmark

A candidate appears 2× faster in one run but equal to baseline in another. List environmental variables and describe a paired experimental design that can distinguish code-structure effects from machine state.

---

## 24.4 Answer sketches

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

## 24.5 Final design review checklist

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

[Return to handbook contents](README.md).
