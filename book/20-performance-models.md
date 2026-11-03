# 20. Performance and communication models

[Previous](19-correctness.md) · [Contents](README.md) · [Next](21-architecture-search.md)

A performance model is not a magic formula for the fastest loop order. Its purpose is to **rule out implausible architectures**, identify bottlenecks, and decide what to measure.

## 20.1 Decompose elapsed time

A useful accounting is:

\[
T_{\text{total}}
\approx T_{\text{setup}} + T_{\text{packing}} + T_{\text{kernel}}
+ T_{\text{finalization}} + T_{\text{unpacking}} + T_{\text{scheduling}}
\]

These are *measured phase costs* when separated; execution inside a kernel often overlaps computation, memory, and control, so their microarchitectural costs cannot generally be added without double-counting.

## 20.2 Effective vector utilization

Define:

\[
U_{\text{lane}} =
\frac{\text{useful logical lane-operations}}
     {\text{total issued logical lane-operations}}
\]

A vector architecture with V lanes and only half its lanes active often provides less than half the nominal arithmetic throughput, even before masking and branching overhead. More formally, compare **useful results per unit time**, not vector instructions per second.

## 20.3 Arithmetic intensity and the roofline idea

Let **F** represent useful compute operations and **B** the transferred bytes at a specified memory-hierarchy level. Arithmetic intensity is:

\[
I = F/B
\]

An upper bound on achievable compute rate at that level is conceptually:

\[
P \leq \min(P_{\text{compute peak}}, I \cdot BW_{\text{level}})
\]

This is a bound under stated assumptions, not a runtime prediction. Cache hit rates change B at each level, and irregular accesses may be latency-bound without saturating bandwidth.

## 20.4 Output-stationary versus input-stationary traffic

Suppose each of N_O output tiles consumes N_R input tiles.

- Output-stationary may read shared inputs repeatedly for each output tile but write each completed output once.
- Input-stationary may read a shared input tile once but repeatedly load/store output partials.

Build a **reuse table**:

| Resource | Unique bytes | Uses per tile | Repeated transfers |
| --- | ---: | ---: | ---: |
| Input tile | B_in | N_O | Depends on retention |
| Output partials | B_out | N_R | Depends on accumulator residency |
| Indices/masks | B_idx | Per logical vector | Depends on precomputation |
| Conversion buffers | B_conv | Once or every pass | Depends on pipeline structure |

Measure or estimate *actual* cache-line traffic rather than equating repeated logical loads with repeated DRAM transfers.

## 20.5 Register spills

If a microtile requires more live state than the allocator can keep in registers, spills create hidden stores/loads and lengthen dependency chains. A larger vector width can increase register demand even when the number of logical operations decreases.

A rough model:

~~~text
required_live_state =
    accumulator_vectors
  + operand_vectors
  + intermediate_vectors
  + masks_and_indices
  + latency_hiding_vectors
~~~

Compare this estimate to the target's usable register resources, leaving margin for setup and code generation.

## 20.6 Dependency latency versus throughput

Consider an accumulation:

~~~text
acc = IDENTITY
FOR r
    acc = COMBINE(acc, EVALUATE(r))
~~~

If each update consumes the previous one, critical-path latency may limit throughput. Multiple independent chains can hide that latency:

~~~text
acc0, acc1, ... = IDENTITIES
FOR groups_of_contributions
    UPDATE_INDEPENDENT_ACCUMULATORS(acc0, acc1, ...)
COMBINE_FINAL_PARTIALS()
~~~

Too many chains increase register pressure and merge cost. The fastest number is not dictated by vector width alone.

## 20.7 Address-generation cost

Even when memory data is cached, complex address calculations or long pointer chains can be slow. For each candidate, count:

- dependent address computations;
- contiguous versus indirect accesses;
- number of independent outstanding requests;
- index conversion and permutation work;
- cache-line and page-crossing frequency.

## 20.8 A multidimensional cost hypothesis

An architecture candidate **A** can be evaluated using:

\[
C(A) = (M,\ L,\ R,\ X,\ D,\ G,\ S)
\]

where:

- **M**: estimated memory traffic;
- **L**: lane utilization;
- **R**: register demand;
- **X**: cross-lane exchange;
- **D**: dependency critical path;
- **G**: gather/scatter and address-generation cost;
- **S**: setup, packing, and scheduling overhead.

These are not naturally comparable units. Use the vector to characterize tradeoffs and decide experiments, not to manufacture a single score without calibration.

## 20.9 Bottleneck-driven optimization

| Evidence | Candidate changes |
| --- | --- |
| High transferred bytes, low arithmetic utilization | Improve locality/tiling, fuse stages |
| Long dependency chain | Increase independent accumulators/tasks |
| Low useful-lane fraction | Bucket, compact, change lane mapping |
| Many lane permutations | Change layout/lane mapping, recompute |
| Register spills | Smaller output tile, fewer live chains, fission |
| Scatter conflicts | Group by destination, privatize |
| Setup dominates small inputs | Use simpler path, reduce packing |
| Poor scaling with threads | Adjust task granularity, reduce false sharing |

A change should target a measured bottleneck; indiscriminate tiling often adds cost.

**Takeaway:** model *the costs introduced by the schedule*, then validate the important hypotheses empirically.
