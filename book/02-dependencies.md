# 2. Dependence analysis and transformation legality

[Previous](01-contract.md) · [Contents](README.md) · [Next](03-lane-mapping.md)

## 2.1 The four dependence questions

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

## 2.2 Read/write-set method

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

## 2.3 Reductions are not automatically associative

~~~text
FOR r
    sum = COMBINE(sum, EVALUATE(r))
~~~

If **COMBINE** is associative under the specified arithmetic semantics, lane-local accumulators may be merged in another order. If it is not, a regrouping changes results.

**Integer addition** may be associative only under a defined overflow regime; saturating addition generally is not. **Floating-point addition** is not exactly associative. An approximate result contract may permit a different tree, but then specify error bounds and reproducibility expectations.

## 2.4 Privatization removes write conflicts

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

## 2.5 Dependency direction can reveal a wavefront

~~~text
FOR time
    FOR position
        state[time, position] = F(
            state[time-1, position],
            state[time-1, position-1])
~~~

Every point depends only on the previous time step. Positions at a fixed time are independent if all reads address the completed earlier layer. A schedule can process **time** sequentially and **position tiles** in parallel. Different dependence graphs may instead admit diagonal wavefronts; never assume diagonals are safe without checking the actual edges.

## 2.6 Legality checklist

Before applying a transformation, answer:

1. Does every required logical iteration execute exactly once?
2. Is every read preceded by its producer?
3. Are all simultaneous writes independent or conflict-resolved?
4. Is accumulator combination order equivalent under the contract?
5. Are intermediate and final conversions applied at the same semantic points?
6. Does the transformed code preserve boundary conditions and side effects?

**Takeaway:** the fastest illegal schedule is not an optimization.
