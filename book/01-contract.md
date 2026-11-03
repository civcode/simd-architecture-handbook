# 1. The semantic contract and iteration-space model

[Contents](README.md) · [Next: Dependencies](02-dependencies.md)

## 1.1 Begin with a specification, not a vector

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

## 1.2 Distinguish dimensions from loops

A **logical axis** describes varying work; a **loop** specifies when it executes.

An iteration space of depth three can be implemented as two loops when one axis is expressed by lanes, or as six loops after tiling. Similarly, a one-dimensional iteration space may use several nested loops to traverse cache tiles and vector tails.

The proposed architecture is a schedule:

~~~text
SCHEDULE(logical_iteration) ->
    (task, tile, sub_tile, vector_group, lane, execution_order)
~~~

A schedule is valid when it includes all required iterations, preserves required dependency order, and commits outputs without races or unintended duplication.

## 1.3 Separate four kinds of work

**Independent work:** can proceed in any order and be distributed across lanes or tasks.

**Reduction work:** several contributions produce one output; arbitrary reassociation is legal only if the combine operation permits it.

**Ordered work:** step k reads a result from an earlier step; a schedule must respect that edge or use a proven algorithmic transformation.

**Conflicting work:** two iterations may update the same location. Independence cannot be assumed from distinct loop indices alone.

These categories—not the names of application functions—determine the available SIMD architectures.

## 1.4 Resource vocabulary

- **Lanes:** conceptual parallel positions for simultaneous values.
- **Accumulators:** live partial outputs, possibly one per lane.
- **Tile:** a bounded set of logical iterations processed as a unit.
- **Stationary resource:** the data deliberately retained through an inner loop.
- **Lane communication:** movement or reduction among lane positions.
- **Register pressure:** number and width of simultaneously live values.
- **Working set:** data referenced while a tile is processed, distinguishing unique bytes from repeated accesses.
- **Tail:** logical work not filling a complete vector or tile.

## 1.5 Example: identical semantics, different schedules

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

## 1.6 First exercise

Given a computation with two outputs that read overlapping input regions but different contribution counts, identify (a) legal output parallelism, (b) reduction order constraints, (c) shared inputs, and (d) whether output-stationary or input-stationary work deserves testing.

**Takeaway:** optimize a *semantic iteration space*, not the incidental spelling of its original loops.
