# 19. Numerical correctness and semantic equivalence

[Previous](18-pipelines.md) · [Contents](README.md) · [Next](20-performance-models.md)

Performance claims matter only after the transformed algorithm satisfies its **semantic contract**. Some transformations preserve exact results; others are valid only under an explicit tolerance policy.

## 19.1 Define the equivalence level

| Equivalence class | Required property |
| --- | --- |
| Bit-exact | Every observable output bit matches |
| Value-exact | Mathematically equivalent under specified arithmetic |
| Deterministic | Same results across repeated executions |
| Tolerance-bounded | Error stays within stated absolute/relative bound |
| Decision-equivalent | Same ordering, classifications, or branch decisions |
| Statistical | Distributional properties match a specified test |

These categories are not interchangeable. An algorithm may be mathematically exact yet not bit-exact under floating-point reordering.

## 19.2 Reassociation and floating-point behavior

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

## 19.3 Integer overflow and saturation

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

## 19.4 Tie breaking and stable selection

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

## 19.5 Masks, bounds, and tails

A masked-off lane must not access invalid memory unless the memory operation guarantees nonaccess for inactive lanes. A masked reduction lane must contribute the appropriate **identity**. Test smallest sizes, zero-length segments, sizes just below/above vector width, and every remainder class.

## 19.6 Aliasing and in-place updates

A legal out-of-place vector schedule can be wrong in place:

~~~text
FOR i IN ORIGINAL_ORDER
    data[i] = F(data[i], data[i-1])
~~~

Reordering iterations may overwrite an input needed by a later iteration. Options include snapshot buffers, wavefront/ordered scheduling, or proving a safe direction. The existence of apparently independent output positions is insufficient.

## 19.7 Parallel races and update conflicts

Two threads may update the same destination even when their loop indices differ. Scratch storage must be private or synchronized. Deterministic results may require fixed merge order, and nonassociative reductions cannot be arbitrarily merged.

## 19.8 Differential testing protocol

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

## 19.9 Metamorphic properties

When an independent oracle is unavailable, validate properties that must hold under legal transformations: partitioning the input into tiles should not change a mathematically associative result; output permutation followed by inverse permutation should recover the same set of independent results; changing task group size should preserve per-task results.

These tests supplement but do not replace comparison to a trusted reference.

## 19.10 Proof obligations for every optimization

1. Define logical work set and every required read/write.
2. Show all work is included (no duplicates or omissions).
3. Show all true dependencies are respected.
4. Establish arithmetic equivalence or explicit tolerated differences.
5. Prove tail and boundary memory safety.
6. Preserve output placement and tie/termination rules.
7. Test against an independent oracle under edge workloads.

**Takeaway:** correctness is a property of the **schedule plus arithmetic semantics**, not simply its final-looking formula.
