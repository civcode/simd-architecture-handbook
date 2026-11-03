# Contributing to the handbook

This repository is a guide to **architecture reasoning**, not a catalog of processor-specific intrinsics.

For a new chapter or pattern, include:

1. **Semantic contract:** inputs, outputs, reference computation, permitted reassociation, aliasing, error policy, and boundary conditions.
2. **Dependence graph:** identify independent, reduction, ordered, and potentially conflicting axes.
3. **Candidate schedules:** at least two architectures or a clear reason only one is legal.
4. **Resource reasoning:** active lanes, register state, memory movement, locality, lane communication, branch divergence, and setup cost.
5. **Pseudocode:** abstract and consistent with the glossary; no language-specific vector intrinsics.
6. **Verification:** reference comparison, corner cases, and workload dimensions under which the proposal may fail.
7. **Measurement:** if providing speedups, give the platform, inputs, benchmark methodology, and variability; distinguish observed data from theoretical predictions.

Keep examples function-agnostic. It is fine to mention application domains as motivation, but algorithm explanations should stand on generic roles such as output, reduction, state, and index.

## Suggested editing checklist

- Every pseudocode loop covers the correct set of logical iterations exactly once, or explains its legal reordering.
- No masked operation is implicitly assumed to make an invalid address safe.
- Reduction reassociation is guarded by explicit algebraic or tolerance assumptions.
- Scratch state is initialized, private where needed, and finalized even on a short final tile.
- Each architectural recommendation states its trade-off and what could make it lose.
- Relative links resolve within the repository.
- Do not add automatic GitHub Actions triggers; local validation is preferred.

Licensing has not yet been decided; avoid adding incompatible third-party material.
