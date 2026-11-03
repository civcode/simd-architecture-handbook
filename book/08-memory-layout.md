# 8. Memory access and data-layout co-design

[Previous](07-hierarchical-tiling.md) · [Contents](README.md) · [Next](09-lane-communication.md)

A vector schedule and its data layout form one design. An excellent lane mapping can become slow when adjacent lanes read distant addresses.

## 8.1 Five important access patterns

| Pattern | Abstract address sequence for lanes | Typical concern |
| --- | --- | --- |
| Contiguous | base, base+1, base+2, ... | Usually efficient transfers |
| Constant stride | base, base+s, base+2s, ... | Wasted transferred bytes when s is large |
| Broadcast | base, base, base, ... | One value reused by many lanes |
| Gather | index[0], index[1], ... | Scattered requests, index overhead |
| Scatter | destination[index[0..]] | Conflicts and locality of writes |

These describe *addresses*, not specific instructions. Hardware may lower a gather to several loads. Even contiguous accesses can cross cache lines, pages, or memory boundaries.

## 8.2 Derive addresses for every lane

~~~text
FOR lane IN ACTIVE_LANES
    logical = MAP_LANE_TO_ITERATION(tile, lane)
    read_addresses[lane]  = READ_ADDRESSES(logical)
    write_addresses[lane] = WRITE_ADDRESSES(logical)

ESTIMATE_UNIQUE_CACHE_LINES(read_addresses)
ESTIMATE_UNIQUE_CACHE_LINES(write_addresses)
DETECT_POSSIBLE_WRITE_CONFLICTS(write_addresses)
~~~

Do this separately for the hot loop and for any setup or finalization pass. A schedule with cheap hot-loop loads may require an expensive one-time transpose.

## 8.3 Structure of arrays versus array of structures

Suppose every task owns attributes **a**, **b**, and **c**.

**Record layout** stores all attributes per task together. It favors processing one complete task at a time.

**Column layout** stores each attribute for all tasks contiguously. It often favors processing the same attribute across many tasks in lanes.

~~~text
# Record-oriented access
FOR task
    a,b,c = LOAD_RECORD(task)
    EXECUTE_TASK(a,b,c)

# Column-oriented access
FOR tile IN TILES(tasks, V)
    a = LOAD_COLUMN(attribute_a, tile)
    b = LOAD_COLUMN(attribute_b, tile)
    c = LOAD_COLUMN(attribute_c, tile)
    EXECUTE_TASKS(a,b,c)
~~~

Neither is universally faster. A task may need just one field or all fields, and an alternate layout can increase scalar overhead elsewhere.

## 8.4 Blocked or hybrid layouts

An array-of-short-vectors layout organizes a limited number of tasks into packs:

~~~text
FOR pack IN TASK_PACKS
    data[pack].field_a = CONTIGUOUS_VALUES_FOR_PACK(a)
    data[pack].field_b = CONTIGUOUS_VALUES_FOR_PACK(b)
    data[pack].field_c = CONTIGUOUS_VALUES_FOR_PACK(c)
~~~

Such a layout can preserve some record locality while making per-field vector access regular. Pack width should follow an architectural cost model, not necessarily a single hardware vector width.

## 8.5 Repacking has a break-even point

Let **C_pack** be transformation cost, **C_original** the cost of one pass using the original layout, and **C_packed** one pass on the new layout. Over **K** repeated passes, packing helps only if:

\[
C_{\text{pack}} + K C_{\text{packed}} < K C_{\text{original}}
\]

This ignores secondary effects such as extra memory footprint and cache eviction. A layout beneficial for one hot kernel may harm the pipeline if data must be converted back immediately.

## 8.6 Spatial versus temporal locality

- **Spatial locality:** one access brings neighboring bytes that the program soon uses.
- **Temporal locality:** a recently accessed value is used again.
- **Register reuse:** data stays in the most local storage between uses.
- **Cache reuse:** data is reloaded from a cache instead of lower memory.

Interchanging loops can trade temporal locality of accumulators against spatial locality of inputs. Look at **the entire live tile**, not only the innermost instruction.

## 8.7 Prefetching and address predictability

Regular forward streams are often easier to prefetch than unpredictable pointer chains. A schedule that reduces arithmetic but introduces dependent indirect loads can become latency-bound.

~~~text
FOR tile
    addresses = COMPUTE_NEXT_ADDRESSES(tile)
    values = LOAD_BY_ADDRESSES(addresses)
    COMPUTE(values)
~~~

Software scheduling may overlap address generation and other independent work, but speculative reads must not access invalid memory.

## 8.8 False sharing and task ownership

Two parallel tasks can write distinct logical outputs that occupy the same cache line. This does not create a data race, but can generate coherence traffic. Organize task output ranges to reduce shared lines and avoid excessive interleaving.

## 8.9 Layout-and-loop search

For each candidate, write the tuple:

~~~text
candidate = (
    output_tile_shape,
    input_tile_shape,
    lane_mapping,
    physical_data_layout,
    access_pattern,
    conversion_cost
)
~~~

Measure whole-pipeline time when conversion is involved. **Takeaway:** vectorization should minimize *moved bytes per useful result*, not merely maximize the fraction of work expressed as vector operations.
