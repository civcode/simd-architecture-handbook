# 6. Output-, input-, and problem-stationary schedules

[Previous](05-loop-transformations.md) · [Contents](README.md) · [Next](07-hierarchical-tiling.md)

A **stationary** schedule keeps one expensive-to-move resource near the compute units while repeatedly using it. The outermost loop usually chooses the owner of that resource.

## 6.1 Output-stationary: retain partial results

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

## 6.2 Input-stationary: retain shared source values

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

## 6.3 Problem-stationary: retain task context

~~~text
FOR ptile IN TILES(PROBLEMS, T_P)
    task_context = INITIALIZE_TASK_STATES(ptile)
    FOR stage IN STAGES
        EXECUTE_STAGE_ON_TASK_TILE(task_context, stage)
    COMMIT_TASK_RESULTS(task_context)
~~~

This is valuable when each task has private state or when multiple stages can reuse setup for the same task. It can be weak when inputs across adjacent tasks overlap but the task tile prevents shared-input reuse.

## 6.4 Hybrid ownership is often better

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

## 6.5 Simple communication model

Let:

- **B_in** = bytes of shared input per reduction tile.
- **B_out** = bytes of live outputs for an output tile.
- **N_O** = number of output tiles.
- **N_R** = number of reduction tiles.

Ignoring caching and internal temporal overlap:

- Output-stationary may stage shared input **N_O × N_R** times while committing output once.
- Input-stationary may stage shared input **N_R** times while reading/writing output partials as often as **N_O × N_R**.

This is a **traffic upper-level comparison**, not a prediction of DRAM bytes. Frequently reused data may be served by caches, hardware prefetchers, or register-resident sub-tiles.

## 6.6 Why the outermost tile matters

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

## 6.7 Stationary selection questions

1. What is expensive to reload: inputs, outputs, or task metadata?
2. How many uses occur before a stationary value is no longer needed?
3. Does retaining it consume scarce registers or scratch storage?
4. Can neighboring output or problem tiles share the value?
5. What is the address stride of the innermost work?
6. Does a larger outer tile increase irregular masking or divergence?
7. Which resource is actually bottlenecking the current schedule?

**Takeaway:** stationary architecture is a choice of **data lifetime**, not merely a loop permutation.
