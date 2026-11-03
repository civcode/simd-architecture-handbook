# Designing Efficient SIMD Loop Architectures
## A pseudocode-only guide grounded in `civcode/ssbm`

**Repository:** https://github.com/civcode/ssbm  
**Source revision reviewed:** [`003a4f91a4684264186b9c7fd8e08c422770df05`](https://github.com/civcode/ssbm/tree/003a4f91a4684264186b9c7fd8e08c422770df05) (`master`, inspected 2026-11-03)  
**Audience:** algorithm designers, performance engineers, and developers learning to restructure nested loops for SIMD.  
**Notation:** all implementation examples use **language-independent pseudocode**, not C/C++ intrinsics. Instruction-family names appear only as explanations of the existing implementation.

> **Central lesson:** The optimal SIMD loop nest is not necessarily the scalar loop nest with a vectorized innermost loop. Start by deciding **what each vector lane represents**, **which output partial sums must stay live**, and **which region of input data should be reused before moving on**. Only then choose the loop order and tile dimensions.

---

## 1. What this project actually does

`ssbm` is a **sparse stereo block matcher**. Given left and right grayscale images and a list of feature points, it computes disparity values by comparing a small feature-centered patch against horizontally shifted patches in the other image.

The implementation contains three substantive SIMD computation families:

1. **Disparity cost-vector computation (SAD):** compute a sum of absolute differences for many candidate horizontal disparities. This is the main case study for multi-level loop reordering and disparity tiling.
2. **Subpixel disparity refinement:** compare a small neighborhood against seven candidate interpretations (six interpolated positions and one original-pixel position), using packed pixel operations, averaging, SAD reductions, and carefully preserved rounding.
3. **Horizontal Sobel-like prefiltering:** calculate horizontally neighboring differences over a sliding three-row neighborhood, process two output rows together, and write contiguous output pixel blocks.

The surrounding matcher performs **local-minimum selection**, an optional **reverse-match consistency check**, optional subpixel refinement, and writes one result per keypoint. There is also **thread-level parallelism across keypoints**, which is distinct from SIMD parallelism *inside* one keypoint.

### 1.1 SIMD implementations present

| Variant | SAD vector: main organization | Subpixel: main organization | Prefilter: main organization |
|---|---|---|---|
| Scalar reference | `row → patch column → disparity` | `row → column → candidate` | `two output rows → pixel` |
| SSE / 128-bit | `32 disparities → row chunk → row → column`, plus 16-disparity tail | seven candidate accumulators for 32-column case; narrower column blocks otherwise | `two output rows → 8 output pixels`, then scalar remainder |
| AVX2 / 256-bit | `64 disparities → row chunk → row → four-column group`, plus 32/16-disparity tails | 32-column SIMD comparisons and grouped sums; narrower fallback | `two output rows → 16 output pixels`, then 8-pixel and scalar tails |
| AVX-512 / 512-bit | `64 disparities → row chunk → row → four-column group`, plus 16-disparity tails | 32-column comparison with seven packed candidate sums; narrower fallback | `two output rows → 32 output pixels`, then 8-pixel and scalar tails |

**Important distinction:** SIMD width (128/256/512 bits) is **not** the same as the number of independent disparities computed per tile. A 256-bit implementation can deliberately keep a **64-disparity tile** live by using multiple vector accumulators. The AVX-512 implementation also uses a 64-disparity main tile, but its instruction mapping and register grouping differ.

**Source anchors:** [scalar SAD](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM.cpp#L86-L115), [SSE SAD](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM_SSE.cpp#L170-L276), [AVX2 SAD](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM_AVX2.cpp#L374-L438), [AVX-512 SAD](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM_AVX512.cpp#L161-L345).

---

## 2. First understand the mathematical iteration space

Use these symbols consistently:

| Symbol | Meaning |
|---|---|
| `L`, `R` | Left and right grayscale images, one 8-bit intensity per pixel |
| `(x, y)` | Current feature point in the left image |
| `W` | Odd SAD window width *and* height; `h = floor(W / 2)` |
| `D` | Number of candidate disparities for this particular keypoint |
| `d` | **SAD vector index**, from `0` through `D − 1` |
| `r`, `c` | Row/column offsets inside the feature patch |
| `B` | Disparity tile width: number of candidate costs maintained together |
| `Rchunk` | Number of feature-patch rows accumulated before flushing narrow accumulators |
| `V` | Abstract vector width expressed in lanes of a stated datatype |

In the normal forward path, SAD vector index `d` corresponds to geometric disparity `D − 1 − d`. This reversal matters: the stored cost array proceeds from the leftmost searched reference position toward the feature's x-coordinate.

For one keypoint, the conceptual result is:

\[
\operatorname{cost}[d]
= \sum_{r=-h}^{h}\sum_{c=-h}^{h}
\left|L[y+r,x+c]-R[y+r,x+c-(D-1-d)]\right|.
\]

Equivalent reference-oriented pseudocode:

```text
PROCEDURE SAD_REFERENCE(L, R, x, y, W, D):
    h ← floor(W / 2)
    cost[0 .. D-1] ← 0

    FOR r FROM -h TO h:
        FOR c FROM -h TO h:
            feature ← L[y+r, x+c]
            FOR d FROM 0 TO D-1:
                reference_x ← x+c-(D-1-d)
                cost[d] ← cost[d] + ABS(feature - R[y+r, reference_x])

    RETURN cost
```

This defines correctness. The SIMD versions compute *the same cost values* using a different visitation order and representation of the partial sums.

### 2.1 Size of the work

The number of individual pixel differences is:

\[
N_{\text{differences}} = D W^2.
\]

For a `15 × 15` patch and `128` disparities, that is **28,800** absolute differences **per keypoint, per forward SAD pass**. A reverse consistency check may require a second pass.

However, **28,800 differences does not mean 28,800 different reference pixels**. Each row of the reference image needs only `D + W − 1 = 142` distinct horizontal positions. Across 15 rows there are at most **2,130 distinct reference-pixel positions**, plus **225 feature-patch pixels**. They may be loaded multiple times; those figures are a *unique-address lower bound*, not measured cache traffic.

That large gap between **logical operations** and **distinct input data** is why loop order, overlap, broadcast reuse, and cache behavior matter so much.

### 2.2 Domain constraints that influence the architecture

In the current matcher, the available disparity range is clamped to a **multiple of 16**, constrained both by the configured maximum and by the keypoint's left border. A feature near the left border may therefore have fewer disparities, or none. The implementation also uses odd window sizes and carries a 32-bit output SAD array.

This creates a useful SIMD design contract:

```text
D ← usable disparity count, rounded down to a multiple of 16
IF D = 0:
    mark keypoint invalid
ELSE:
    execute the disparity-tiled SAD kernel
```

Other applications should not copy the multiple-of-16 rule blindly; it is a useful constraint of **this** algorithm and permits simpler tails.

---

## 3. SIMD concepts as algorithmic design decisions

### 3.1 Lane semantics: what is computed in parallel?

A SIMD vector is a fixed set of lanes which undergo the same operation. In this project, the **same patch calculation** can assign lanes to very different things:

| Candidate lane meaning | Typical operation | Used in this project? | Why it matters |
|---|---|---|---|
| **Different disparities** | Compare one feature pixel or short feature sequence against many shifted reference positions | **Yes: primary SAD vector** | Produces several output costs simultaneously and allows a register-resident disparity tile |
| **Neighboring output x-pixels** | Sobel-like prefilter over contiguous image columns | **Yes** | Naturally produces contiguous stores and sliding-window loads |
| **Neighboring pixels within one patch** | Compare two horizontal subpixel windows | **Yes: subpixel SAD** | Many input pixels reduce into a few candidate scores |
| **Different interpolation candidates** | Maintain scores for seven candidate shifts | **Yes, as multiple/packed accumulators** | Facilitates sharing the same left-image pixels among several hypotheses |
| **Different keypoints** | One lane per independent feature point | **Not a production SAD strategy here** | Would introduce irregular gathers, differing disparity bounds, and lane divergence |

**Rule:** Decide the lane meaning before choosing which loop to vectorize. If lanes represent disparities, the disparity coordinate belongs in vector lanes, not necessarily in the innermost source loop. If lanes represent output image x positions, the x coordinate should be the vectorized coordinate.

### 3.2 The operation vocabulary (abstract, not machine-specific)

The following names will be used in pseudocode:

```text
VECTOR_LOAD_CONTIGUOUS(address, lane_count)
VECTOR_SPLAT(single_value, lane_count)
VECTOR_ABS_DIFFERENCE(a, b)
VECTOR_WIDEN_UNSIGNED(bytes, wider_element_type)
VECTOR_ADD(a, b)
VECTOR_SUM_GROUPS(vector, group_width)
VECTOR_ROUNDED_AVERAGE_UNSIGNED(a, b)
VECTOR_SATURATE_TO_UNSIGNED_BYTE(vector)
VECTOR_CLAMP(vector, low, high)
VECTOR_REORDER(vector, desired_lane_order)
VECTOR_MASK_INVALID(vector, valid_lane_mask, zero)
VECTOR_STORE_CONTIGUOUS(address, vector)
```

These are **semantic operations**. Their efficient implementation depends on the target instruction set and which data arrangement those instructions require.

### 3.3 The concepts actually exercised in `ssbm`

| Concept | In practical terms | Example in repository |
|---|---|---|
| Packed integer SIMD | One operation acts on many 8-, 16-, or 32-bit values | All three SIMD kernel families |
| Unaligned contiguous loads/stores | Vector memory access without requiring each address to start at a vector-aligned boundary | Sliding reference windows, Sobel x positions |
| Broadcast / splat | Reuse one feature pixel or four-pixel feature group against multiple reference positions | SAD kernels |
| Sliding-window overlap | Candidate windows start one pixel apart and share much of their input | SAD vector |
| Four-byte block SAD operations | Compute several shifted four-pixel absolute-difference sums at once | AVX2 and AVX-512 main SAD paths |
| Narrow accumulators | Keep running sums in 16-bit lanes to increase compute density | SAD kernels; subpixel slots |
| Widening and flush | Convert partial 16-bit sums to 32-bit before overflow | SAD kernels |
| Lane-local instruction behavior | Some wide operations behave separately within smaller fixed-size sublanes | AVX2/AVX-512 packing and SAD layout |
| Cross-lane permutations | Restore disparity order before storing contiguous costs | AVX-512 SAD flush; output packing |
| Reduction | Combine groups of per-pixel differences into candidate totals | Subpixel SAD |
| Masking | Make unused logical lanes contribute zero | Partial subpixel windows |
| Saturation and clamping | Convert signed Sobel-like values into an 8-bit output range | Prefilter |
| Rounded unsigned average | Preserve the integer interpolation's tie/rounding semantics | Subpixel refinement |
| Remainder kernels | Handle incomplete main tiles without corrupting neighbors | 16/32-disparity SAD tails; prefilter tails |
| Runtime backend selection | Choose supported SIMD implementation at runtime | SSE / AVX2 / AVX-512 dispatch |
| Thread-level task decomposition | Process separate keypoint ranges on different CPU threads | TBB matcher path |

**Not observed as production loop techniques:** a dense three-dimensional cost volume, gather-based SIMD over unrelated keypoints, general-purpose image-wide keypoint spatial tiling, and a demonstrated SIMD autotuner. These are possible extensions, not techniques to attribute to the repository.

---

## 4. The central problem: reorganizing a three-loop SAD computation

There are three conceptual dimensions:

- **`r` — patch row:** input addresses move by one image row stride.
- **`c` — patch column:** feature and reference input addresses move horizontally.
- **`d` — disparity:** the **output being accumulated changes**; reference positions also shift horizontally.

The scalar reference is:

```text
FOR r:                       # one patch row
    FOR c:                   # one feature pixel in that row
        FOR d:               # update every candidate disparity
            cost[d] += pixel_difference(r, c, d)
```

A naive conclusion is: “`d` is innermost, therefore make `d` a vector loop.” This is only **one** SIMD architecture, and it is often not the fastest.

### 4.0 All six permutations of the three mathematical loops

Before tiling, there are exactly **six** ways to order three loops. Every one is mathematically valid for independent SAD outputs with an associative exact-integer sum. In practice, the **accumulator representation and input-memory layout** make them very different:

| Mathematical loop order, outer → inner | Outermost data unit | Advantage worth considering | Likely constraint on row-major images |
|---|---|---|---|
| `r → c → d` | One input patch row, then one pixel | One feature pixel reused across all candidate disparities; reference accesses contiguous in `d` | Repeated cost-array updates if all disparities cannot stay live |
| `r → d → c` | One input patch row, then output-disparity group | Each disparity tile can accumulate over a row | Flush or revisit costs after every row |
| `c → r → d` | One patch column, then a vertical traversal | Reuse one column offset, then vectorize disparities | Vertical strides between consecutive rows; repeated cost updates |
| `c → d → r` | One patch column, then output-disparity group | Hold a disparity tile across vertical pixel comparisons | Strided image accesses and per-column partial-cost flushes |
| `d → r → c` | One output-disparity group | Keep a tile live over contiguous x positions in each row | The feature patch is revisited once per tile; register budget limits tile width |
| `d → c → r` | One output-disparity group, then one patch column | Tile output still stays live across a full reduction | Innermost vertical memory strides often impede cache-line use/prefetch |

Here `r`, `c` and `d` are **conceptual** loop coordinates; SIMD execution adds a **tile loop** for `d` and sometimes a row-chunk loop for `r`. The repository effectively selects **`d → r → c`** for its SIMD SAD path, with four-column groups and overflow-safe row chunks. The comparison table is a design analysis, **not** a claim that all six variants were benchmarked.

```text
# Two less common but mathematically valid orderings:
FOR c:
    FOR disparity tile d0:
        live_partial ← zero
        FOR r:
            live_partial += comparisons at (r, c) for this disparity tile
        ADD live_partial to persistent cost[d0 ..]  # once per column

FOR disparity tile d0:
    live_partial ← zero
    FOR c:
        FOR r:
            live_partial += comparisons at (r, c) for this disparity tile
    STORE live_partial to cost[d0 ..]                # once per tile
```

The second example can hold output sums in registers, like `d → r → c`, but puts a **vertical image-stride traversal in the innermost loop**. On a row-major image it can therefore lose even when the number of output writes is unchanged. This is a good illustration of why **loop order and tile choice must be evaluated together**.

### 4.1 Why vectorizing the scalar loop may be expensive

Suppose we process `B` disparity costs together but still keep `r → c` outside:

```text
FOR r:
    FOR c:
        feature ← L[y+r, x+c]
        FOR d0 FROM 0 TO D-1 STEP B:
            a ← VECTOR_LOAD(cost[d0 .. d0+B-1])
            v ← VECTOR_LOAD(reference candidates at (r, c, d0))
            a ← a + ABS(feature broadcast to B lanes - v)
            VECTOR_STORE(cost[d0 .. d0+B-1], a)
```

This is correct *in principle*. But the partial results usually cannot all stay live in registers **across the outer `r` and `c` loops**. The implementation may perform a read-modify-write of its cost array for each of the `W²` patch positions.

For `W = 15`, that means up to **225 cost-array visits per keypoint**, each potentially involving loads and stores, even though the output array is small. L1 cache helps, but instructions, load/store ports, dependencies, and address work still cost time.

The remedy is not merely a wider vector. The remedy is to **change the lifetime of the accumulators**.

### 4.2 Move a disparity tile to the outside

```text
FOR d0 FROM 0 TO D-1 STEP B:       # choose a group of output costs
    accumulator[0 .. B-1] ← 0      # keep the whole group live

    FOR r FROM -h TO h:
        FOR c FROM -h TO h:
            feature ← L[y+r, x+c]
            refs ← contiguous reference values for disparities d0 .. d0+B-1
            accumulator ← accumulator + ABS(BROADCAST(feature) - refs)

    STORE cost[d0 .. d0+B-1] ← accumulator
```

The inner computation still traverses the full feature patch, but each output cost is now **stored once per tile**, not once per patch pixel. The output sums can stay in registers, and the scalar feature pixel is broadcast over many candidate comparisons.

This is the conceptual basis of the SIMD SAD code in `ssbm`.

### 4.3 What changed in the iteration space?

The mathematical loop nest has not changed; every `(r, c, d)` triplet is still visited. What changed is:

1. The `d` dimension has been **strip-mined** into tiles of width `B`.
2. The **tile loop** moved outside the patch traversal (**loop interchange**).
3. Each tile has a **private live partial-sum vector** (**register blocking**).
4. The loads now naturally follow **contiguous disparity positions** (**data layout matched to lanes**).
5. The narrow accumulator must be **flushed at a safe point** if its datatype cannot hold the entire patch's sum.

This is the crucial distinction between **SIMD as a loop annotation** and **SIMD as a different algorithmic architecture**.

---

## 5. Six competing loop architectures

All examples compute the same full `cost[d]` result. They make different promises about register residency, input reuse, and output traffic.

### Architecture A — Pixel-major, disparity-inner (scalar-friendly)

```text
FOR r:
    FOR c:
        feature ← load feature pixel
        FOR d0 IN disparity tiles:
            costs[d0] ← costs[d0] + compare feature with B reference pixels
```

**Good when:** the cost vector is tiny, automatically vectorized partial sums remain in registers, or the architecture offers a very efficient in-cache update path. **Risk:** repeated read-modify-write of cost tiles; expensive accumulator lifetimes. **Project status:** shape of the scalar reference, not the selected SIMD SAD main path.

### Architecture B — Disparity-tile-major, then patch rows and columns

```text
FOR d0 IN disparity tiles:
    live_costs ← zero vector group
    FOR r:
        FOR c:
            update live_costs from feature and reference
    store live_costs once
```

**Good when:** `B` output accumulators can remain in registers; feature-patch rereads are cheap; no overflow before the final store. **Risk:** large `B` causes excessive register pressure, and the feature patch is revisited once per disparity tile. **Project status:** direct architectural model for the SIMD SAD kernels.

### Architecture C — Disparity-tile-major with row chunks (the project pattern)

```text
FOR d0 IN disparity tiles:
    first_chunk ← TRUE
    FOR r0 IN row chunks:
        live_costs ← zero narrow accumulators
        FOR r IN this row chunk:
            FOR c IN patch columns or column groups:
                update live_costs
        wide_costs ← widen and reorder live_costs
        IF first_chunk:
            cost[d0 ..] ← wide_costs
        ELSE:
            cost[d0 ..] ← cost[d0 ..] + wide_costs
        first_chunk ← FALSE
```

**Good when:** the cheap vector SAD operation creates **16-bit** partial sums but the whole patch may exceed 16-bit range. **Benefit:** hot narrow accumulators stay in registers; larger final results accumulate in 32-bit storage only at chunk boundaries. **Risk:** chunk boundaries increase conversion and memory traffic; a missing final flush silently drops work. **Project status:** **implemented by SSE, AVX2 and AVX-512**, with different chunk-size rules.

### Architecture D — Row-major, disparity tiles inside each row

```text
cost[0 .. D-1] ← 0
FOR r:
    FOR d0 IN disparity tiles:
        row_accumulator ← zero
        FOR c:
            row_accumulator += comparisons for B disparities
        cost[d0 ..] += row_accumulator
```

**Good when:** a complete row of the feature patch is easier to reuse than the entire patch, or when `W` is very large. **Risk:** every disparity tile is read and written **once per row**, rather than once per window (or narrow overflow chunk). **Project status:** analytical alternative, **not** the selected main SAD architecture.

### Architecture E — Column-major with a complete vector of output costs

```text
cost[0 .. D-1] ← 0
FOR c:
    FOR r:
        feature ← load feature pixel
        FOR d0 IN disparity tiles:
            cost[d0 ..] += comparisons for B disparities
```

**Good when:** a separate transform has already made vertical columns contiguous, or an application shares column work between nearby windows. **Risk:** vertically strided memory accesses, many output updates, and often poor hardware prefetch behavior on row-major images. **Project status:** an alternative to test, **not** a production SAD loop in this repository.

### Architecture F — Spatial-feature-tile-major, then disparity tiles

```text
FOR each small spatial region of the image:
    nearby_features ← keypoints located in this region
    keep relevant image rows hot if possible

    FOR each keypoint in nearby_features:
        FOR d0 IN disparity tiles:
            compute this keypoint's SAD tile
```

**Good when:** many keypoints lie near one another so their patches overlap and repeated image fetches dominate. **Risk:** sorting/scattering the outputs, load imbalance, and irregular boundaries. **Project status:** a **proposed extension**, not the current keypoint processing order. The repository instead iterates keypoints in input order (sequentially or as thread-partitioned ranges).

### 5.1 Side-by-side decision table

| Architecture | What outermost loop chooses | What can stay in registers? | Repeated output traffic | Main tradeoff |
|---|---|---|---|---|
| A | A patch pixel | Usually a transient disparity group | Potentially every patch pixel | Feature reuse versus cost-array traffic |
| B | A disparity tile | Full tile across all patch pixels | About one final write per tile | Register lifetime versus repeated feature-patch visits |
| C | A disparity tile | Full tile for one safe row chunk | One write/add per chunk | Narrow fast arithmetic versus flush overhead |
| D | A patch row | One disparity tile across one row | Once per row | Row locality versus accumulator updates |
| E | A patch column | Transient disparity group | Potentially every patch pixel | Vertical strides and cost traffic |
| F | A spatial keypoint region | The inner algorithm's chosen tile | Depends on inner architecture | Cross-keypoint input reuse versus scheduling/scatter cost |

**There is no universal best order.** B/C are strong starting points for this repository's SAD computation because the **output space is small** (`D` costs), but the **reduction space is large** (`W²` patch positions). Another kernel with a different output/reduction balance can prefer a different outer loop.

---

## 6. Reconstructing the project's actual SAD loop nests

### 6.1 SSE: 32 disparities per main tile

The SSE kernel maintains two groups of 16 disparity candidates, expressed as four widened 16-bit accumulator vectors. It compares a **broadcast feature byte** against two contiguous 16-byte reference spans per patch position, then widens the absolute differences. The full 32-disparity tile is processed over a row chunk before widening and flushing into 32-bit output costs.

```text
PROCEDURE SAD_SSE_ARCHITECTURE(feature_patch, reference_region, W, D):
    B ← 32
    Rchunk ← floor(257 / W)

    FOR d0 FROM 0 TO largest_complete_32_tile STEP 32:
        first_chunk ← TRUE
        FOR rows IN groups of at most Rchunk patch rows:
            acc[0 .. 31] ← zero 16-bit sums
            FOR row IN rows:
                FOR column FROM 0 TO W-1:
                    feature_byte ← feature_patch[row, column]
                    refs0 ← reference values for disparities d0 .. d0+15
                    refs1 ← reference values for disparities d0+16 .. d0+31
                    acc[0..15]  += WIDEN(ABS_DIFF(BROADCAST(feature_byte), refs0))
                    acc[16..31] += WIDEN(ABS_DIFF(BROADCAST(feature_byte), refs1))
            FLUSH_AS_32_BIT_COSTS(acc, first_chunk)
            first_chunk ← FALSE

    IF 16 disparities remain:
        PROCESS_ONE_16_DISPARITY_TILE_WITH_SAME_ROW_CHUNK_RULE
```

Because each pixel difference is at most `255`, a 16-bit sum is safe for **at most 257** such differences: `257 × 255 = 65,535`. Entire rows are the flush unit, so `Rchunk × W` must not exceed 257.

**Source:** [SSE SAD and final-flush paths](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM_SSE.cpp#L170-L276).

### 6.2 AVX2: 64 disparities and four-pixel source groups

The AVX2 kernel goes further than simply doubling the bytes loaded. Its main path exploits a special sliding **four-byte SAD operation**. One feature group of four adjacent bytes contributes partial costs to many disparities. Hardware produces its results in a **nonlinear lane order**, so several accumulators are used and their outputs are reordered on flush.

The code chooses **64 disparities** as the main tile and explicitly comments that this balances register pressure with reuse of the feature broadcast across multiple groups. It has a 32-disparity secondary path and a 16-disparity tail.

```text
PROCEDURE SAD_AVX2_ARCHITECTURE(feature_patch, reference_region, W, D):
    MAIN_TILE ← 64
    SAFE_ROWS ← floor(65535 / (255 × W))
    COMPLETE_COLUMNS ← 4 × floor(W / 4)

    FOR d0 IN complete 64-disparity tiles:
        first_chunk ← TRUE
        FOR rows IN groups of at most SAFE_ROWS:
            create four 16-bit vector accumulator groups for 64 disparities
            FOR row IN rows:
                FOR c FROM 0 TO COMPLETE_COLUMNS-1 STEP 4:
                    feature4 ← load four adjacent feature bytes
                    partials ← FOUR_BYTE_SLIDING_SAD(feature4, reference bytes for tile)
                    accumulate partials in hardware's native lane order
                HANDLE_LAST_ONE_OR_THREE_COLUMNS_WITH_EXACT_CORRECTION
            ordered_costs ← REORDER_AND_WIDEN_TO_32_BIT(accumulators)
            FLUSH_OR_ADD(ordered_costs, first_chunk)
            first_chunk ← FALSE

    PROCESS_OPTIONAL_32_DISPARITY_TILE
    PROCESS_OPTIONAL_16_DISPARITY_TILE
```

The final **one or three columns** require care because `W` is odd. When there are three residual columns, the implementation computes an overlapping four-byte group and subtracts the duplicated column's contribution. This avoids processing three columns with a fully separate scalar-like path; the arithmetic correction must exactly match the pixel subset.

**Source:** [AVX2 four-column tile core](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM_AVX2.cpp#L132-L276) and [tile selection / overflow rule](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM_AVX2.cpp#L374-L438).

### 6.3 AVX-512: still 64 disparities, but a different internal lane map

The AVX-512 code again uses a **64-disparity main tile**, with two accumulators for four-byte SAD partials and four additional wide accumulators for residual columns. The four-byte instruction produces disparity groups in a different physical order; the flush operation permutes and widens them before writing four consecutive 16-cost spans.

A particularly instructive difference: for `W = 15`, the four-byte core processes **12 columns**, and the remaining three columns accumulate **directly in 32-bit sums**. Only the **12-column core** affects the 16-bit overflow bound. The AVX2 implementation instead includes its corrected remainder in the narrow sum.

```text
PROCEDURE SAD_AVX512_ARCHITECTURE(feature_patch, reference_region, W, D):
    B ← 64
    core_columns ← 4 × floor(W / 4)
    residual_columns ← W - core_columns

    IF core_columns > 0:
        Rchunk ← floor(65535 / (255 × core_columns))
    ELSE:
        Rchunk ← W

    FOR d0 IN complete 64-disparity tiles:
        first_chunk ← TRUE
        FOR rows IN groups of at most Rchunk:
            narrow_groups ← zero 16-bit SAD accumulators
            residual_groups ← zero 32-bit SAD accumulators

            FOR row IN rows:
                FOR c IN four-column groups:
                    feature4 ← load adjacent feature bytes
                    partials ← WIDE_FOUR_BYTE_SLIDING_SAD(feature4, reference tile)
                    narrow_groups += partials

                FOR c IN remaining one to three columns:
                    residual_groups += WIDENED_BYTE_ABSOLUTE_DIFFERENCES

            ordered_core ← REORDER_AND_WIDEN(narrow_groups)
            ordered_costs ← ordered_core + residual_groups
            FLUSH_OR_ADD(ordered_costs, first_chunk)
            first_chunk ← FALSE

    FOR d0 IN remaining 16-disparity tiles:
        compute direct 32-bit SAD across the full patch
        store the resulting tile
```

**Why not just use 128 disparities?** More width or more simultaneous output costs can increase register demand, data-permutation work, instruction pressure, and spill risk. This implementation **chooses** 64; the repository does **not** establish that 64 is universally optimal on all processors or for all window sizes. The correct tile size is a measured design parameter.

**Source:** [AVX-512 main tile and residual paths](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM_AVX512.cpp#L161-L345), [lane reorder and flush](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM_AVX512.cpp#L113-L159).

### 6.4 The important nested-loop order, abstracted

```text
FOR each keypoint:                         # cross-keypoint architecture
    FOR each disparity tile:               # output patch/tile chosen first
        FOR each safe row chunk:            # overflow/control tile
            FOR each patch row:            # input y direction
                FOR each patch column group:# input x direction
                    update register accumulators for the disparity tile
            flush partial sums to full-width output
```

What looks like a five-level loop is actually a **three-dimensional mathematical problem** (`row`, `column`, `disparity`) with two additional tiling dimensions (`disparity tile`, `row chunk`). The apparent depth of the code is not itself a performance problem. What matters is **the order and ownership of the live data**.

---

## 7. Choosing the outermost tile: the most important design exercise

The phrase “outermost patch” can mean three different things. Keeping them distinct prevents many bad optimization decisions.

1. **Output tile:** a set of disparity scores `cost[d0 .. d0+B-1]` to keep live. This is what the SAD SIMD kernels choose first.
2. **Reduction tile:** a subset of patch rows or columns whose contributions fit a narrow accumulator. This is the `Rchunk` row chunk.
3. **Spatial input tile:** a region of the *image* whose pixels will be reused across multiple nearby keypoints. The production SAD kernel does **not** currently make this the outermost tile.

The best ordering depends on which reuse opportunity is more valuable: keeping **outputs in registers**, keeping **inputs in cache**, or amortizing **setup across neighboring problems**.

### 7.1 A practical default hierarchy

For a sparse stereo SAD kernel like this one, start with:

```text
OUTERMOST:
    independent keypoints, optionally grouped for threads

NEXT:
    choose one disparity tile whose accumulators fit in registers

NEXT:
    choose a row chunk safe for the accumulator datatype

INNERMOST:
    traverse adjacent feature pixels in short groups
    compare each group against contiguous shifted reference pixels

ONCE PER CHUNK:
    reorder and widen the tile's results, then update final costs
```

This hierarchy follows **ownership**: one keypoint owns one cost array; one disparity tile owns one set of live scores; one row chunk owns one safe narrow reduction. It is a useful **starting point**, not a proof of optimality.

### 7.2 Register budget before cache budget

A tile only delivers its intended advantage when the accumulators really remain in registers. Increasing `B` can reduce the number of tile passes, but increases:

- the number of simultaneously live accumulators;
- the number of reference vectors loaded per source group;
- intermediate values needed for widening, masks and permutations;
- register spills and stack traffic if the live set exceeds the target's budget;
- the length of dependency chains, unless independent accumulators are used.

A **conceptual register budget** is:

\[
R_{\text{accumulators}}(B)+R_{\text{loads}}(B)+R_{\text{temporaries}}(B)+R_{\text{control}}
\;\le\;R_{\text{usable registers}}.
\]

Do not treat `B = lane count` as a requirement. A good tile often occupies **several** vector registers. Conversely, a tile twice as wide as the hardware vector is not automatically bad if its live state still fits.

**Practical test:** sweep plausible tile widths such as `16`, `32`, `64` and, where the operation supports it, `128`; inspect performance and spill evidence. The repository uses `32` as its SSE main tile and `64` as its AVX2/AVX-512 main tiles, but does not publish a controlled multi-size tile sweep establishing a global optimum.

### 7.3 Choose reduction chunk size from arithmetic, not from intuition

For unsigned byte pixels:

\[
0 \le |a-b| \le 255.
\]

A 16-bit unsigned accumulator holds at most `65,535`, so a safe chunk must obey:

\[
\text{pixel differences per lane per chunk} \le
\left\lfloor\frac{65535}{255}\right\rfloor=257.
\]

For an entire W-column row contributing to one narrow sum:

\[
Rchunk \le \left\lfloor \frac{257}{W}\right\rfloor.
\]

For the AVX-512 **four-column core**, use the number of columns actually accumulated in 16-bit lanes:

\[
Rchunk \le \left\lfloor \frac{65535}{255\,\cdot\,(4\lfloor W/4\rfloor)}\right\rfloor,
\]

when the denominator is nonzero. Residual columns are accumulated separately in 32-bit lanes.

| `W` | SSE / AVX2 safe rows per 16-bit chunk | AVX-512 four-column core: safe rows per chunk | Whole `W×W` patch fits a 16-bit result? |
|---:|---:|---:|---|
| 15 | 17 | 21 (`core_columns = 12`) | Yes, at most 225 differences |
| 17 | 15 | 16 (`core_columns = 16`) | No, up to 289 differences |
| 31 | 8 | 9 (`core_columns = 28`) | No, up to 961 differences |

These are **safety maxima**, not necessarily performance-optimal chunk sizes. Smaller chunks can improve register scheduling or latency but cost more flushes; larger chunks are illegal if they overflow. Always flush the **final partial chunk**, even when it is shorter than the earlier chunks.

### 7.4 Choose input traversal according to memory layout

These grayscale images are logically **row-major**, with a possibly padded stride between rows. Therefore:

- Adjacent **columns** are contiguous and suitable for vector loads.
- Adjacent **rows** are separated by the image stride; simply swapping row and column loops can create strided accesses.
- Adjacent **disparities** also access contiguous reference-image x positions for fixed `(row, column)`.
- Neighboring disparities have highly overlapping reference windows, so overlapping loads are usually served from nearby cache lines even if the algorithm issues several load operations.
- Feature pixels are repeatedly reused across disparity tiles; their **logical load count** is not their **actual memory-system traffic**.

That last point matters: do not predict performance solely by counting array expressions. Cache-line fills, load-port throughput, alignment crossings, store traffic, and register spills are distinct costs.

### 7.5 A numerical tile example: `W=15`, `D=128`

For one row of the reference input, the **minimum distinct** input interval is `128 + 15 − 1 = 142` pixels. A disparity tile of width `B=64` touches an interval of `64 + 15 − 1 = 78` pixels per row. Two tiles nominally visit `2 × 78 = 156` pixel positions, but **14 positions overlap** between those intervals; the union is still only 142.

A `B=16` strategy makes eight tile passes, increasing repeated feature traversals and interval overlap; a `B=128` strategy makes one pass, but must hold or spill more accumulator state. This illustrates the competing effects:

```text
smaller B:
    fewer live accumulators
    more tile setup and more revisits to the feature patch

larger B:
    fewer tile passes and potentially more feature reuse
    more register pressure, wider permutations and possible spills

optimal B:
    balances both effects on the target CPU and problem distribution
```

**Do not infer actual bytes transferred from the intervals above.** The implementations also use overlapping vector loads, cache-line granularity, and sometimes hardware-specific overfetch; the simple formulas explain locality, not actual hardware traffic.

---

## 8. A defensible performance model for loop-order experiments

A useful model has separate terms for calculation, data movement, and overhead:

\[
T \approx \max(T_{\text{compute}},T_{\text{memory}})
+T_{\text{loop/setup}}+T_{\text{permute/reduce}}+T_{\text{spill/flush}}.
\]

This is a **heuristic**, not a calibrated prediction. The terms overlap in real processors. Use it to generate hypotheses, then benchmark them.

### 8.1 Work-count estimates

For each keypoint:

- **Mathematical comparisons:** `D × W²`.
- **Number of disparity tiles:** `ceil(D / B)`.
- **Number of row chunks per tile:** `ceil(W / Rchunk)`.
- **Number of 16-bit-to-32-bit flushes:** roughly `ceil(D/B) × ceil(W/Rchunk)`, times the number of accumulator groups in a tile.
- **Feature-patch visits in a disparity-major design:** roughly `ceil(D/B) × W²` pixel positions, before accounting for any packed four-byte processing.
- **Reference-region span per tile, per row:** approximately `B + W − 1` pixels, before hardware load overlaps and boundary overfetch.

These estimates expose why neither “smallest tile” nor “largest tile” is always correct.

### 8.2 Register-resident versus memory-resident partial sums

Consider two extremes:

```text
SCHEME 1: update output memory for every source pixel
    output traffic scales with W² × D

SCHEME 2: keep B output costs in registers across the reduction
    output traffic scales with D × number_of_safe_row_chunks
```

The second often wins even if it rereads some feature pixels. The cache can supply a small feature patch cheaply, while repeated output loads/stores and long update chains can be expensive. But if `B` is too large and spills to the stack, the advantage disappears.

### 8.3 A key distinction: memory locality is not just loop adjacency

When evaluating a loop order, ask five separate questions:

1. **Spatial locality:** are consecutive input addresses adjacent on a cache line?
2. **Temporal locality:** how long until the same data is needed again?
3. **Output residency:** does a partial sum remain in a register across many updates?
4. **Instruction locality:** can one broadcast or address calculation serve several vector operations?
5. **Instruction semantics:** does the chosen CPU SAD operation naturally process the proposed tile geometry, or will rearrangement dominate?

The fastest design may purposely **repeat an L1 hit** to avoid a **register spill** or multiple **read-modify-write output updates**. Counting redundant source accesses without accounting for this distinction can lead to the wrong loop architecture.

### 8.4 A measurement matrix

Rather than tune only `B`, compare these architectural dimensions independently:

| Question | Values to try | What to measure |
|---|---|---|
| Which output dimension is register-tiled? | Disparity, row partial sum, column partial sum | Total time, load/store traffic, spills |
| Tile width `B`? | 16, 32, 64, 128 where legal | Throughput versus register pressure |
| Row chunk? | Arithmetic-safe maximum and smaller multiples | Flush count, correctness, dependency latency |
| Column processing? | One byte at a time versus groups of four | Source broadcasts, instruction count, tail cost |
| Keypoint ordering? | Input order versus spatial bins | Cache misses, sorting/scatter overhead |
| Thread task granularity? | Smaller versus larger keypoint ranges | Thread overhead, load balance, scratch reuse |
| Window and disparity shape? | Small/large `W`, short/long `D`, edge points | Robustness across real workloads |

A performance decision is only trustworthy when the **same correct numerical result** is checked for every variant.

---

## 9. Choosing the best outermost patch: three realistic scenarios

### Scenario 1 — One sparse keypoint with a large disparity range

**Choose the disparity output tile first.** One keypoint produces a small 1D cost vector and performs a large reduction into it. Holding a subset of those costs live gives a large amortization opportunity.

```text
FOR disparity tile:
    keep partial SAD outputs live
    traverse the feature patch in row-major order
    flush only when arithmetic or register constraints require it
```

This is the case most closely reflected in the current project.

### Scenario 2 — Thousands of nearby keypoints with overlapping patches

Now the outermost **image region** may become interesting. Nearby keypoints could share prefiltered image data or reference cache lines. However, each keypoint may have a different left-border-limited disparity range, so blindly assigning one SIMD lane to each keypoint creates irregular work.

```text
GROUP keypoints by coarse image region
FOR each region:
    FOR each keypoint in the region:
        compute disparity-tiled SAD as before
    place results back into original keypoint order
```

**When to test:** many features cluster spatially and image fetches dominate. **When to avoid:** points are already sparse, sorting is costly, task balancing degrades, or the entire image is already well cached. This is an **extension to evaluate**, not a claimed repo optimization.

### Scenario 3 — Dense stereo matching over most image pixels

If almost every pixel requires a disparity estimate, a different dataflow may win: **a spatial image tile** can own an entire block of output pixels and a disparity tile. A sliding sum or integral/box-filter representation may further reduce redundant window work.

```text
FOR spatial image tile:
    FOR disparity tile:
        construct or reuse the local matching-cost data
        aggregate cost across the window
        update output disparities for the spatial tile
```

This is a **different algorithmic problem** from the repository's sparse keypoint matcher. It must not be presented as if the existing `ssbm` code implements a dense cost volume.

**Decision principle:** output-tile-first is a strong default for **independent sparse reductions**; input-spatial-tile-first becomes increasingly attractive when **many neighboring outputs** reuse the same input region.

---

## 10. Other SIMD concepts used by the project

### 10.1 Absolute difference for unsigned bytes

There are multiple correct packed ways to compute `ABS(a − b)` for byte-valued inputs. The SSE SAD code uses the equivalent of **two unsigned saturating subtractions added together**: one direction is nonzero and the other is zero. The AVX2 path also uses an unsigned **maximum-minus-minimum** formulation for its residual-byte corrections. Both avoid treating pixel bytes as signed values.

```text
FUNCTION UNSIGNED_BYTE_ABS_DIFFERENCE(a, b):
    RETURN MAX(a, b) - MIN(a, b)
```

**Lesson:** the exact arithmetic operation matters for correctness, but the best representation can differ with available vector instructions.

### 10.2 Specialized sliding four-pixel SAD

The AVX2 and AVX-512 SAD cores use processor instructions that evaluate four-byte comparisons against several shifted reference positions. Conceptually:

```text
FUNCTION FOUR_PIXEL_SAD(feature4, reference_start):
    FOR local_shift IN hardware_supported_shifts:
        partial[local_shift] ← 0
        FOR t FROM 0 TO 3:
            partial[local_shift] += ABS(feature4[t] - reference[reference_start + local_shift + t])
    RETURN partial
```

The **logical output** is a set of sliding SAD partials; the **physical result lanes** are not necessarily sorted by disparity. In AVX2, some groups fall in separate 128-bit sublanes. In AVX-512, two wide accumulator groups interleave disparity ranges, requiring an explicit final reorder.

**Lesson:** instruction-native layout can dictate the best tile shape and may require a layout-conversion step. Fewer instructions for the core calculation can outweigh a permutation at the end.

### 10.3 Width changes and accumulator precision

The SAD kernels take 8-bit unsigned input differences, keep efficient **16-bit partial sums**, and finally produce **32-bit full-window costs**. The conversion must happen **before** 16-bit overflow, not after it. A wrapped 16-bit value cannot be repaired by later widening.

```text
partial16 ← zero
FOR row IN safe row chunk:
    partial16 += per-pixel differences

partial32 ← UNSIGNED_WIDEN(partial16)
full_cost32 += partial32
```

For `W = 31`, a full cost can reach `31 × 31 × 255 = 245,055`, well beyond 65,535. The repository includes regression tests specifically guarding correct flush/final-copy behavior for wider windows.

**Source:** [README flush regression explanation](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/README.md), [SSE flush](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM_SSE.cpp#L40-L62).

### 10.4 Lane-local grouping and reordering

Wide vector registers do not imply arbitrary free rearrangement of their elements. Many operations have **fixed sublane boundaries** and produce groups in an instruction-defined order. The repository handles this by accumulating in native order and reassembling **only at the output boundary**, rather than repeatedly shuffling on every patch pixel.

```text
FOR many inner-loop updates:
    accumulate in native hardware order

AT final flush:
    ordered ← REORDER(native_accumulators, disparity_index_order)
    STORE ordered to cost array
```

**Lesson:** move expensive reordering **outside** high-trip-count loops whenever possible.

### 10.5 Subpixel refinement: vector lanes represent patch pixels

Subpixel refinement is a different SIMD problem from the main SAD vector. Here there are only **seven output candidates**, but up to **32 horizontal input pixels** to compare on each row. The code effectively vectorizes over **image columns**, then reduces the columns into seven candidate scores.

It uses two rounds of **rounded unsigned byte averages** to approximate intermediate intensities. This is not identical to arbitrary-precision linear interpolation, because rounding happens at each averaging step.

```text
FUNCTION AVG_BYTE_ROUNDED(a, b):
    RETURN floor((a + b + 1) / 2)

PROCEDURE SUBPIXEL_CANDIDATE_SCORES(left_patch, right_patch, width):
    scores[0 .. 6] ← zero 16-bit values

    FOR each patch row:
        left_pixels ← up to width contiguous pixels
        right_minus_one ← corresponding right pixels shifted -1
        right_center ← corresponding right pixels shifted 0
        right_plus_one ← corresponding right pixels shifted +1

        left_half ← AVG_BYTE_ROUNDED(right_minus_one, right_center)
        right_half ← AVG_BYTE_ROUNDED(right_center, right_plus_one)

        candidate[0] ← AVG_BYTE_ROUNDED(right_minus_one, left_half)
        candidate[1] ← left_half
        candidate[2] ← AVG_BYTE_ROUNDED(right_center, left_half)
        candidate[3] ← AVG_BYTE_ROUNDED(right_center, right_half)
        candidate[4] ← right_half
        candidate[5] ← AVG_BYTE_ROUNDED(right_plus_one, right_half)
        candidate[6] ← right_center

        FOR k FROM 0 TO 6:
            scores[k] ← (scores[k] + SUM_ABS_DIFF(left_pixels, candidate[k])) MOD 65536

    best ← first smallest score, following the project's comparison order
    RETURN disparity adjustment associated with best
```

**Key architectural difference:** the SAD-vector kernel has **many outputs and a patch-pixel reduction**; the subpixel kernel has **few outputs and a wide pixel reduction**. Therefore the natural SIMD dimension flips from **disparity** to **patch columns**.

For a full 32-column subpixel window, the SSE implementation uses seven running candidate accumulators, while AVX2 compares 32-byte windows and folds their subgroup sums. AVX-512's wide subpixel path stores seven candidate sums in packed 16-bit lanes. Narrower windows use 128-bit fallback logic.

**Exactness note:** the repository preserves **16-bit modular accumulation** for subpixel candidate scores. A 32 × 32 window could mathematically sum beyond 65,535, so the code's `mod 65536` behavior is a **behavior-preserving detail**, not an assertion that it computes unrestricted-precision SAD. A redesign may improve the numeric model, but that would be a semantic change requiring new reference outputs.

**Source:** [scalar subpixel reference](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM.cpp#L117-L187), [SSE subpixel](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM_SSE.cpp#L278-L435), [AVX2 subpixel](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM_AVX2.cpp#L459-L630), [AVX-512 subpixel](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM_AVX512.cpp#L372-L567).

### 10.6 Horizontal Sobel-like prefilter: SIMD across output x

The prefilter illustrates yet another loop choice. It computes **two output rows at once** and applies a clipped horizontal derivative calculation across a contiguous run of output columns. The SIMD versions use blocks of **8 pixels (SSE), 16 (AVX2), and 32 (AVX-512)**, followed by smaller and/or scalar tails.

```text
PROCEDURE VECTOR_PREFILTER(image, output, block_width):
    FOR y FROM 0 TO image_height-2 STEP 2:
        select neighboring input rows for output rows y and y+1
        set the left and right border pixels to the border value

        FOR x IN complete block_width-sized interior pixel groups:
            left_neighbors  ← contiguous pixels at x-1
            right_neighbors ← contiguous pixels at x+1
            horizontal_differences ← right_neighbors - left_neighbors
            filtered_y   ← weighted differences for first output row
            filtered_y1  ← weighted differences for second output row
            pack/clamp each result to allowed unsigned-byte output range
            store the two contiguous output pixel blocks

        FOR each remaining interior pixel:
            calculate the scalar prefilter rule

    initialize the remaining bottom row, if any
```

The two output rows share source rows, which is why the **row-pair loop is outermost**. The contiguous x block is the SIMD dimension; disparity is irrelevant to this kernel.

The SIMD implementations perform **unsigned-byte saturation**, **clamping to the `0 .. 2×ftzero` range**, and **packing/reordering** to get two contiguous output rows. An early AVX-512 de-interleave bug corrupted outputs even though its arithmetic was correct; the repository's benchmark report documents its correction.

**Important preservation quirk:** the SIMD vector body uses the third sampled source row again in the fourth-row term for the second output row, whereas the scalar tail/reference uses a distinct fourth source row where available. The repository deliberately preserves this historical SIMD behavior; it is **not** safe to assume scalar and SIMD prefilters are mathematically identical everywhere. Distinguish **SIMD-to-SIMD bit-exactness** from **scalar-to-SIMD equivalence** when building an oracle.

**Source:** [SSE prefilter](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM_SSE.cpp#L436-L515), [AVX2 prefilter](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM_AVX2.cpp#L634-L766), [AVX-512 prefilter](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM_AVX512.cpp#L569-L719), [benchmark discussion](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/benchmarks/report_avx512.md).

### 10.7 Masked tails versus safe memory accesses

Subpixel refinement can use a logical width of 8, 16, 24, or 32 columns. The SIMD variants mask out unwanted lanes so those lanes do not contribute to SAD.

```text
loaded ← VECTOR_LOAD_CONTIGUOUS(start, full_vector_width)
valid_mask ← lanes with index < logical_width
values ← VECTOR_MASK_INVALID(loaded, valid_mask, zero)
```

**Critical distinction:** masking values **after** a full-width load makes their *arithmetic contribution* zero, but **does not make an out-of-bounds full-width memory access safe**. Either prove that the entire vector load is inside accessible memory (including row/ROI boundaries), use a genuinely predicated load, or process the final part with a narrower load. The repository's logical lane masking should not be read as a general memory-safety proof.

### 10.8 SIMD width is a backend decision, not an algorithm contract

The repository has runtime checks for SSE2, AVX2, and AVX-512 feature availability and the operating-system register state required by wider vectors. The public matcher chooses a suitable backend; the standalone AVX2/AVX-512 entry points have fallback behavior when their target is unavailable. The algorithm-level interface remains the same: **given a keypoint and a disparity range, produce ordered costs**.

```text
IF target supports this program's required 512-bit operations:
    backend ← 512-bit implementation
ELSE IF target supports this program's required 256-bit operations:
    backend ← 256-bit implementation
ELSE IF target supports the 128-bit implementation:
    backend ← 128-bit implementation
ELSE:
    backend ← scalar reference

compute all keypoints through the selected backend
```

**Design lesson:** specify loop architecture using **semantic tiles**, not hard-coded programming-language vector types. ISA-specific kernels can choose different tile shapes beneath the same numerical contract.

---

## 11. Thread parallelism versus SIMD: choose the two levels separately

The TBB variant distributes **ranges of keypoints** across CPU tasks. Each blocked-range invocation allocates reusable scratch arrays, processes its own keypoints, and writes to distinct output indices. It does not need a hot-path mutex for those writes.

```text
PARALLEL_FOR each disjoint range of keypoint indices:
    allocate scratch buffers once for this range
    select available SIMD backend

    FOR keypoint index IN assigned range:
        determine this point's valid disparity count
        compute forward SAD costs using disparity-tiled SIMD
        select local minima
        optionally compute reverse SAD costs
        optionally compute subpixel adjustment
        write only output[index]
```

Do not confuse these dimensions:

- **SIMD:** multiple lanes cooperate on *one keypoint's* candidate-cost or pixel computations.
- **Threads:** different worker tasks process *different keypoints*.

A larger SIMD tile can increase single-core register pressure; more threads can increase shared-cache pressure. The optimum may differ between **single-thread kernel benchmarks** and **full multithreaded pipeline benchmarks**. Likewise, sorting keypoints spatially may improve cache reuse but reduce the balance of a naive fixed-size task partition.

**Source:** [TBB keypoint processing](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM_SSE.cpp#L80-L169), [sequential matcher](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM.cpp#L233-L334).

---

## 12. A systematic procedure for designing a new SIMD loop nest

This is the recommended workflow to teach and reuse. It starts from the mathematics, not from a preferred vector instruction.

### Step 1 — Write the scalar contract and name each dimension

Separate **independent outputs** from **reduction coordinates**. For sparse SAD, `d` identifies outputs, and `(r, c)` are reduction coordinates.

```text
FOR each output d:
    cost[d] ← REDUCE over all (r, c) of pixel_difference(r, c, d)
```

This is not necessarily the scalar execution order; it is a better description of the **data dependencies**.

### Step 2 — Choose lane semantics using contiguous memory access

Ask whether neighboring lanes should represent **outputs**, **source pixels**, or **independent problems**. For sparse SAD, consecutive disparities correspond to consecutive right-image x positions. This is an excellent starting point for SIMD lanes.

### Step 3 — Choose which outputs must remain live

Identify the quantity most costly to repeatedly load and store: the partial cost array. Design a tile so **those outputs** remain live in registers while many source positions contribute.

### Step 4 — Select a legal tile width

Estimate the live vector register count and instruction-specific lane layout. Set an initial `B` that leaves space for input loads, broadcasts, correction values, masks, and permutations. For this project, the observed starting points are 32 disparity scores for SSE and 64 for AVX2/AVX-512.

### Step 5 — Derive accumulator bounds mathematically

Calculate the maximum per-lane contribution and total reduction length. If a narrow accumulator can overflow, split the reduction into **safe chunks** and widen or flush at a well-defined point.

### Step 6 — Order input loops for contiguous access

Keep the x direction inside the row loop for row-major images. Where the ISA supports it, group multiple neighboring source columns into one specialized operation; keep any correction semantics exact.

### Step 7 — Move setup and rearrangement out of the hot loops
Do broadcasts, constant setup, accumulator conversion, and lane reordering **as rarely as correctness allows**. Reorder the SAD lanes at the end of a row chunk, not after every four-pixel comparison.

### Step 8 — Design every tail explicitly

Handle leftover **disparities**, **patch columns**, **prefilter x-pixels**, **rows**, and **subpixel valid lanes**. Do not let a full-width load silently cross a memory boundary. Explicitly describe whether the tail uses a smaller vector, a predicated load, or scalar processing.

### Step 9 — Build independent correctness checks

Use an independent full-precision scalar SAD oracle for the main cost vector. For behavior-preserving SIMD-to-SIMD refactors, also compare against the established backend where historical rounding or wrapping matters.

### Step 10 — Benchmark an architecture matrix, not only an instruction

Measure multiple `B` and row-chunk configurations across a spread of `W`, `D`, keypoint locations, and data patterns. Test single-keypoint kernel time, batches of random keypoints, and the entire matcher separately. Reject designs that only win for one benchmark row or one CPU frequency state.

### Step 11 — Re-evaluate the outermost tile if the workload changes

If the application changes from sparse independent keypoints to many spatially clustered keypoints or dense stereo, the optimal outermost unit may change from a **disparity tile** to an **image tile**, even though the pixel-difference equation is similar.

### Step 12 — Document what was measured versus inferred

A selected tile width is a repository fact; its universal optimality is a hypothesis until measured. Cache locality is a reasoning aid; performance counters and repeated timing are evidence. Keep those categories separate.

---

## 13. A reusable SIMD SAD design template

This pseudocode describes a **configurable algorithmic architecture**, not a new claim about exactly which instructions every backend uses.

```text
PROCEDURE SIMD_SAD_FOR_KEYPOINT(L, R, x, y, W, D, B, core_column_group, max_safe_rows):
    REQUIRE W is odd and image accesses are inside valid boundaries
    REQUIRE D is nonnegative
    REQUIRE max_safe_rows is derived from the actual accumulator format

    h ← floor(W / 2)
    cost[0 .. D-1] ← 0

    FOR d0 FROM 0 TO D-1 STEP B:
        active_disparities ← MIN(B, D - d0)

        FOR r0 FROM -h TO h STEP max_safe_rows:
            active_rows ← MIN(max_safe_rows, h - r0 + 1)
            narrow_accumulators ← ZERO_FOR_ACTIVE_DISPARITIES
            wide_tail_accumulators ← ZERO_FOR_ACTIVE_DISPARITIES

            FOR rr FROM 0 TO active_rows-1:
                row ← y + r0 + rr

                FOR c0 IN complete core_column_group-sized patch groups:
                    feature_group ← L[row, x-h+c0 .. x-h+c0+group_width-1]
                    reference_region ← matching R[row] positions for the tile
                    partial_scores ← VECTOR_SLIDING_GROUP_SAD(feature_group, reference_region)
                    narrow_accumulators ← ADD_IN_NATIVE_LANE_ORDER(
                        narrow_accumulators, partial_scores)

                FOR c IN residual patch columns:
                    feature_value ← L[row, x-h+c]
                    references ← matching R[row] positions for the active disparities
                    wide_tail_accumulators += VECTOR_WIDE_ABS_DIFF(feature_value, references)

            core_scores ← UNSIGNED_WIDEN_AND_REORDER(narrow_accumulators)
            chunk_scores ← core_scores + wide_tail_accumulators
            cost[d0 .. d0+active_disparities-1] += chunk_scores

    RETURN cost
```

**The parameter `max_safe_rows` must match the accumulator's actual contributions.** For example, if the narrow accumulators cover **all W columns** it uses the W-column bound; if they cover only a four-column-multiple core, it uses the core-only bound. Do not apply a chunking formula intended for one arithmetic layout to another.

**Unoptimized conceptual details:** a real implementation may skip zero-initializing the final cost array if the first chunk stores rather than adds; it may have dedicated 16- and 32-disparity tails; or it may represent the last columns with overlap/correction instead of separate wide sums. Those changes do not alter the high-level ownership and tiling logic.

---

## 14. What the repository's benchmarks can and cannot prove

The repository has reproducible benchmark reports, but **they are not a systematic comparison of every possible loop interchange**. They mainly compare implemented SIMD backends, sequential versus keypoint-parallel matching, and kernel-level correctness.

### 14.1 Reported observations

Tests in the repository's reports used an **AMD Ryzen 9 9950X**, a **640 × 480** synthetic stereo pair, **1,000 keypoints**, `W=15`, and a disparity sweep from **64 to 240**.

| Measurement in repository | Reported result | Safe conclusion |
|---|---|---|
| Forward SAD kernel, SSE vs AVX2, `ndisp=64` | `0.621 ms` vs `0.568 ms` | Only a modest gain for a short range |
| Forward SAD kernel, SSE vs AVX2, `ndisp=240` | `1.998 ms` vs `1.227 ms` (about `1.63×`) | Wider/specialized processing benefits this larger range on this system |
| Subpixel, SSE vs AVX-512 | `0.227 ms` vs `0.120 ms` (about `1.90×`) | Pixel-lane processing and compact candidate reduction help this kernel |
| Prefilter, SSE vs AVX-512 | `0.022 ms` vs `0.011 ms` (about `1.99×`) | Contiguous x-block processing is effective here |
| Full matcher, sequential vs TBB at `ndisp=240` | `2.174 ms` vs `0.141 ms` medians (about `15.4×`) | Cross-keypoint threading is a separate, major performance axis |

**Sources:** [SSE/AVX2 + TBB report](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/benchmarks/report.md) and [AVX-512 report](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/benchmarks/report_avx512.md).

These comparisons **do not isolate** the effect of “disparity tile first” versus “row first.” The SIMD variants also differ in vector instruction choice, lane mapping, and tail implementation. A controlled loop-order experiment must implement and benchmark those alternatives explicitly.

### 14.2 The AVX-512 `ndisp=128` timing spike: an important warning

An older benchmark table shows a dramatic AVX-512 slowdown at `ndisp=128`. The repository's later investigation reports **different stable timing regimes for identical code and data**; it demonstrates a post-idle frequency effect and leaves the exact causes of other stable regimes unresolved. It concludes that the single spike is **not evidence of an intrinsic 128-disparity algorithmic penalty**.

**Do not use that spike to justify a tile architecture change.** It demonstrates that CPU frequency, warm-up, process state, memory layout, and timing methodology can distort comparisons. The follow-up report contains hypotheses about unexplained regimes; they should **not** be presented as proven cache-set or allocation effects.

**Source:** [AVX-512 spike investigation](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/REPORT_128spike.md).

### 14.3 Benchmark protocol for loop architectures

```text
FOR each architecture and tile configuration:
    build identical algorithm semantics
    verify every output against the chosen oracle

    FOR each window size and disparity-range class:
        use the same input images and keypoints
        warm up the exact tested kernel
        repeat measurements in balanced or interleaved order
        report median and a spread measure, not a single timing
        record CPU, ISA, compiler, build flags, and threading mode

    separately benchmark:
        isolated kernel
        full keypoint batch
        complete matcher

    investigate large outliers and nonmonotonic jumps before ranking variants
```

Where possible, also inspect **executed instructions, cache misses, load/store behavior, register spills, and CPU frequency**. If those counters are unavailable, distinguish hypotheses from measurements and repeat runs with careful controls.

---

## 15. Correctness traps that can masquerade as speedups

| Trap | Why it happens | Required check |
|---|---|---|
| Missing final row-chunk flush | The last chunk is shorter than the others | Compare full costs for windows 17, 31, and larger |
| Narrow accumulator wrap | Sum exceeds 65,535 before widening | Prove the bound for **every** narrow accumulator path |
| Disparity-lane permutation error | Specialized SAD operations produce noncontiguous lane order | Compare `cost[d]` at every exact index, not only the selected minimum |
| Off-by-one disparity reversal | Vector index is not geometric disparity | Verify `geometric_disparity = D - 1 - d` |
| Duplicate/missing patch column | Four-byte grouping with 1/3-column tails | Verify all window widths and each remainder case |
| Incorrect rounded interpolation | Chained byte averages are replaced by real-number interpolation | Check every subpixel candidate, including tie cases |
| Changed overflow semantics | Subpixel scores use 16-bit modular arithmetic | Match the established backend or deliberately version the numerical contract |
| Masked arithmetic but unsafe load | Inactive lanes are zeroed only after a full load | Prove addresses or use memory-safe predication/tails |
| Packed output scrambled across sublanes | Wide packing preserves instruction-defined sublane order | Verify every output pixel and row after packing |
| SIMD/scalar prefilter mismatch | Historical vector body and scalar body use a different fourth-row term | Define the exact reference whose semantics must be preserved |
| Mixed input paths | Sequential pipeline ignores computed prefilter; parallel branch uses it | Test comparable modes and fix only as a separate semantic change |
| Incorrect parallel scratch reuse | Multiple tasks write the same mutable temporary | Keep task-local scratch and disjoint result indices |

The repository specifically preserves a **sequential-path prefilter inconsistency**: the sequential public matcher passes the original images to matching even when prefiltered images were constructed, whereas the TBB path uses the prefiltered images. Avoid attributing any comparison of these pipelines solely to SIMD or threading if the input semantics differ.

**Sources:** [matcher dispatch/prefilter behavior](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM.cpp#L346-L389), [regression tests described in README](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/README.md).

---

## 16. Exercises for learning the architecture instead of memorizing it

These deliberately require **loop-design reasoning**, not knowledge of C/C++ intrinsics.

**Exercise 1 — Identify the output tile.** For SAD with `W=15`, `D=128`, assume at most 32 disparity costs can stay live. Write pseudocode for the smallest-output-traffic loop order. Count tile visits and output stores. Repeat with a 64-disparity register budget and explain the tradeoff.

**Exercise 2 — Derive safe chunks.** For a 16-bit accumulator, compute the maximum number of complete rows for windows `W=9`, `15`, `17`, and `31`. Then redo the arithmetic if only `4 × floor(W/4)` columns use 16-bit sums and all other columns use 32-bit sums.

**Exercise 3 — Compare architectures A, C, and D.** For `W=31`, `D=128`, estimate how many times the cost array must be updated from memory in each design. Do not count source expressions as cache misses. Explain why the estimates do not directly predict runtime.

**Exercise 4 — Change the SIMD dimension.** Explain why the best lanes for forward SAD are **disparities**, for the prefilter **x positions**, and for the subpixel stage **horizontal patch pixels**. Which results are reduced, and which remain independent?

**Exercise 5 — Consider spatial tiling.** Suppose 10,000 keypoints occur in tight clusters rather than uniformly across the image. Propose a spatial-bin outer loop. State which costs you expect to reduce and which new costs you introduce. Describe how to test whether the gain is real.

**Exercise 6 — Correctness under a wider instruction.** Imagine doubling vector width but keeping the same four-byte SAD operation. Specify how you would validate lane order, narrow accumulator bounds, output packing, and the memory safety of every tail.

**Exercise 7 — Explain a benchmark anomaly.** A tile size suddenly appears 2× slower for one disparity count, while adjacent counts are smooth. Outline a testing sequence that distinguishes a genuine tile-specific instruction problem from frequency, warm-up, measurement-order, and process-state artifacts.

---

## 17. The compact architecture checklist

Before selecting an outer loop for a new SIMD kernel, answer:

1. **What are the mathematically independent outputs?**
2. **What dimensions reduce into each output?**
3. **Which adjacent elements are contiguous in memory?**
4. **What exactly does one vector lane represent?**
5. **What output tile stays live without spilling?**
6. **Does the ISA offer a specialized operation that changes the useful tile shape?**
7. **How many contributions can the narrow accumulator safely hold?**
8. **Which loop order gives the best balance of input reuse and output residency?**
9. **Can lane reordering, masking, and widening be moved outside the deepest loops?**
10. **Are all tails, borders, rounding rules, and output indices exact and memory-safe?**
11. **Does the selected design win on representative window sizes, disparities, and keypoint distributions?**
12. **Have SIMD and thread-level speedups been measured separately?**

**Takeaway for `ssbm`:** its most important SIMD architectural move is to make a **disparity output tile** the dominant work unit, keep its SAD partial sums live, traverse the **patch's row-major pixels** inside it, and split the reduction into **safe row chunks** when necessary. Its other SIMD kernels demonstrate that a good lane mapping is **kernel-dependent**: source columns for subpixel refinement and output x-pixels for the prefilter. Efficient SIMD is primarily a **dataflow and loop-ownership problem**, not a matter of substituting one wide instruction for a scalar operation.

---

## 18. Source map and evidence boundaries

The following links point to the **specific repository revision** reviewed, so examples will not silently drift as `master` evolves:

| Area | Source |
|---|---|
| Mathematical scalar SAD, scalar subpixel, scalar prefilter | [`src/SparseStereoBM.cpp`](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM.cpp) |
| SSE SAD tiling, subpixel lanes, prefilter, and TBB keypoint task | [`src/SparseStereoBM_SSE.cpp`](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM_SSE.cpp) |
| AVX2 specialized four-column SAD, 64-disparity tiles, subpixel, prefilter | [`src/SparseStereoBM_AVX2.cpp`](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM_AVX2.cpp) |
| AVX-512 wide SAD, lane reorder, 16-disparity tail, subpixel and prefilter | [`src/SparseStereoBM_AVX512.cpp`](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/src/SparseStereoBM_AVX512.cpp) |
| Public matcher API and numeric constants | [`include/SparseStereoBM.h`](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/include/SparseStereoBM.h) |
| Correctness tests and known behavior-preservation issues | [`README.md`](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/README.md) |
| SSE/AVX2 and threading results | [`benchmarks/report.md`](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/benchmarks/report.md) |
| AVX-512 results and de-interleave fix | [`benchmarks/report_avx512.md`](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/benchmarks/report_avx512.md) |
| Investigation of anomalous AVX-512 `ndisp=128` timing | [`REPORT_128spike.md`](https://github.com/civcode/ssbm/blob/003a4f91a4684264186b9c7fd8e08c422770df05/REPORT_128spike.md) |

**Classification used throughout this guide:**

- **Observed in the repository:** a specific loop structure, instruction-family behavior, numeric representation, fallback, correctness test, or published benchmark result.
- **Derived from the algorithm:** a safe overflow bound, operation count, legal loop interchange, or a minimum distinct-address estimate.
- **Proposed for future measurement:** any unimplemented loop permutation, larger tile size, spatial keypoint ordering, dense stereo tiling, or claims about the best configuration on a different CPU.

This distinction is part of sound SIMD engineering: **the fastest-looking loop nest is a hypothesis until its exact outputs and measured execution time agree.**