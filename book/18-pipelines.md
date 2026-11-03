# 18. Multi-stage pipelines, fusion, and recomputation

[Previous](17-stateful-work.md) · [Contents](README.md) · [Next](19-correctness.md)

Many algorithms are pipelines: data passes through several transformations before yielding a result. SIMD architecture must choose whether to **fuse stages**, **materialize intermediates**, **recompute values**, or **change lane mapping between stages**.

## 18.1 An abstract two-stage pipeline

~~~text
FOR item
    intermediate[item] = A(input[item])

FOR item
    output[item] = B(intermediate[item])
~~~

A fused schedule is:

~~~text
FOR tile IN TILES(ITEMS, V)
    values = VECTOR_LOAD(input, tile)
    temp = VECTOR_A(values)
    results = VECTOR_B(temp)
    VECTOR_STORE(output, tile, results)
~~~

Fusion removes intermediate writes and reads, but the combined stage has more live temporaries. If A and B prefer incompatible layouts or lane maps, materializing may still be faster.

## 18.2 Loop fusion can change the optimal tile

Suppose A has substantial local input reuse and B operates independently on completed outputs. A may prefer a large cache tile while B prefers full-width contiguous output vectors.

Three architectures:

**Separate passes:** optimize each stage independently, paying intermediate memory traffic.

**Full fusion:** keep each small intermediate vector in registers, possibly sacrificing A's larger input tile.

**Tile-level fusion:** compute a cache tile for A, then immediately run B on that tile before evicting it.

~~~text
FOR cache_tile IN TILES(DOMAIN, T_C)
    temp_tile = EXECUTE_A_FOR_TILE(cache_tile)
    EXECUTE_B_FOR_TILE(temp_tile)
~~~

The third balances fast-cache reuse and per-stage vectorization.

## 18.3 Recompute versus materialize

Sometimes an intermediate is inexpensive but costly to store, reload, or rearrange.

~~~text
# Materialize once
FOR item
    temp[item] = SMALL_TRANSFORM(input[item])
FOR consumer
    FOR item
        USE(consumer, temp[item])

# Recompute per consumer tile
FOR consumer_tile
    FOR item_tile
        temp_local = SMALL_TRANSFORM(input[item_tile])
        USE(consumer_tile, temp_local)
~~~

Recomputation is beneficial only if extra arithmetic is cheaper than saved data movement and it does not disrupt other reuse. It can also be required to avoid expensive cross-lane communication.

## 18.4 A simple break-even inequality

Let **C_recompute** be added arithmetic cost, **C_saved_memory** avoided loads/stores, **C_saved_shuffle** avoided rearrangement, and **C_extra_pressure** added register/spill cost. Recompute is favorable when approximately:

\[
C_{\text{recompute}} + C_{\text{extra pressure}}
<
C_{\text{saved memory}} + C_{\text{saved shuffle}}
\]

These terms are workload- and machine-dependent. The inequality is a guide for experiments, not an exact cost model.

## 18.5 Producer-consumer tiling

Two stages may traverse different iteration spaces:

~~~text
FOR intermediate_index
    temp[intermediate_index] = PRODUCE(inputs)

FOR output_index
    result[output_index] =
        CONSUME(SUBSET_OF(temp, output_index))
~~~

A tile of outputs needs some subset of intermediate values. Compute only those intermediates and retain them for the consumer tile:

~~~text
FOR output_tile
    required_intermediates = DEPENDENCY_CLOSURE(output_tile)
    temp_local = PRODUCE(required_intermediates)
    CONSUME_AND_COMMIT(output_tile, temp_local)
~~~

Overlapping output tiles may recompute some intermediates. Whether this wins depends on tile overlap, producer cost, and cache footprint.

## 18.6 Pipeline stages with different lane mappings

Suppose stage A benefits from lanes representing **independent tasks**, while stage B benefits from lanes representing **output coordinates**.

~~~text
stage_A_tile = VECTOR_PRODUCE(task_lanes)
repacked = CHANGE_LANE_LAYOUT(stage_A_tile, output_lane_layout)
VECTOR_CONSUME(repacked)
~~~

Alternatives are repacking in registers, writing and reloading a transformed intermediate layout, or running each stage with its own mapping. Evaluate total cost including conversion.

## 18.7 Fusion legality

Fusion is legal when it preserves observable ordering and every consumed intermediate is produced before use. If A produces data that multiple later B iterations share, naïve fusion may duplicate work or change side-effect order. In-place processing also requires alias analysis to avoid overwriting inputs needed later.

## 18.8 When fission is better

Separate stages if fusion creates too many live values, excessive branching, incompatible lane maps, or instruction-cache pressure. A pipeline split can also enable specialized kernels and clearer parallel scheduling.

## 18.9 Full-pipeline benchmarking

Time **packing + producer + intermediate handling + consumer + unpacking**. A kernel-level win can be outweighed by layout conversions. Benchmark realistic reuse counts and input sizes, not only a single hot loop.

**Takeaway:** the optimal SIMD architecture can require **more arithmetic but less communication**, or **more passes but less register pressure**.
