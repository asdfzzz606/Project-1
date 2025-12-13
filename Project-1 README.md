
## Experimental Results

### 1. Which program is fastest? Is it always the fastest?

`alloca` is fastest at small node sizes (0.200s avg at 1M blocks with default settings). However, at large node sizes (4096 bytes), all programs converge to approximately 15 seconds, with `malloc` slightly edging out at 14.996s average.

**Not always fastest.** `alloca` dominates at small allocations due to stack efficiency, but loses its advantage when data processing (hashing) becomes the bottleneck at large node sizes.

### 2. Which program is slowest? Is it always the slowest?

`list` is slowest at small node sizes (0.298s avg at 1M blocks), likely due to `std::list` container overhead. However, at large node sizes (4096 bytes), `alloca` becomes slowest at 15.853s average.

**Answer: Not always slowest.** The slowest program changes based on workload characteristics.

### 3. Was there a trend in program execution time based on the size of data in each Node? If so, what, and why?

**Clear positive correlation.** As node size increases from ~10 bytes to 4096 bytes, runtime increases approximately 75x (from 0.2s to 15s at 1M blocks).

**Why:** Larger nodes mean:

- More data to allocate and initialize
- Significantly more data to hash (the dominant factor)
- More cache misses and memory bandwidth consumption

At large node sizes, the hashing workload completely dominates execution time, overwhelming any allocation method differences. This is why all four programs converge to similar performance (~15s) at 4096-byte nodes.

### 4. Was there a trend in program execution time based on the length of the block chain?

**Linear scaling** A 10x increase in NUM_BLOCKS yields approximately 10x runtime increase:

- 10K → 100K blocks: 0s → 0.02s (too small to measure precisely)
- 100K → 1M blocks: 0.02s → 0.2s (10x)
- 1M → 10M blocks: 0.2s → 2s (10x)

This is expected behavior for O(n) algorithms where each node requires constant-time processing (allocate, initialize, hash, traverse).

### 5. Consider heap breaks, what's noticeable? Does increasing the stack size affect the heap? Speculate on any similarities and differences in programs?

**Key Finding: `alloca` has constant 69 breaks regardless of NUM_BLOCKS**

```
NUM_BLOCKS:    100,000    1,000,000    10,000,000
alloca:             69           69            69
list:              156          933         8,711
malloc:            137          751         6,888
new:               156          933         8,711
```

**Why `alloca` is constant:** It allocates entirely on the stack, never requesting heap memory. The 69 breaks are likely from initial program setup and other overhead, not node allocation.

**Heap-based programs scale linearly:** Other programs show ~10x increase in breaks as NUM_BLOCKS increases 10x, as expected when heap needs to grow.

**Similarities:** `list` and `new` have identical break counts (both use C++ `operator new` internally).

**Differences:** `malloc` has ~20% fewer breaks than `new`/`list`, suggesting more efficient heap usage or larger allocation chunks.

**Stack size does NOT affect heap:** Increasing stack limit with `ulimit -s unlimited` only enables `alloca` to handle larger recursion depth; it doesn't change heap break behavior for other programs.

### 6. Considering either the `malloc.cpp` or `alloca.cpp` versions, generate a diagram showing two Nodes

```
Head → [Node 1] → [Node 2] → Tail (nullptr)
         ↓           ↓
    [data array]  [data array]

Node structure with 6 bytes of data:

+------------------+
| Node* next       | 8 bytes (pointer to next Node)
+------------------+
| uint8_t* bytes   | 8 bytes (pointer to data) ──→ [byte0|byte1|byte2|byte3|byte4|byte5]
+------------------+
| size_t size      | 8 bytes (number of data bytes = 6)
+------------------+
| ... (padding)    | alignment padding
+------------------+
Total Node overhead: ~24-32 bytes

The bytes pointer points to the first byte of the dynamically allocated 6-byte data array.
```

### 7. There's an overhead to allocating memory, initializing it, and eventually processing (in our case, hashing it). For each program, were any of these tasks the same? Which one(s) were different?

**Same across all programs:**

- **Data initialization:** All programs generate random data using the same `getNumBytesForBlock()` function
- **Hashing algorithm:** All use identical hashing logic to compute the final result though data size of each node highly influences this so .. same ish?
- **Traversal:** All iterate through the linked list in the same manner

**Different:**

- **Memory allocation method:**
    - `alloca`: Stack allocation via compiler intrinsic
    - `malloc`: C-style heap allocation with `malloc()`
    - `new`: C++ heap allocation with `operator new`
    - `list`: Managed allocation through `std::list` container

This difference in allocation is the primary variable being tested in the experiment.

### 8. As the size of data in a Node increases, does the significance of allocating the node increase or decrease?

**The significance of allocation DECREASES dramatically.**

**Evidence:**

- Small nodes (~10 bytes): `alloca` is 3x faster than `list` (0.196s vs 0.300s) - allocation differences are significant
- Large nodes (4096 bytes): All programs converge to ~15s - allocation differences become negligible

 At large node sizes and counts, the computational cost of hashing dominates total runtime as shown by shrinking degrees of variance. Allocation overhead becomes a smaller fraction of total work:

```
Small nodes: allocation_time ≈ hashing_time → allocation matters
Large nodes: allocation_time << hashing_time → allocation irrelevant
```

The overhead of allocation is amortized as data processing (hashing) becomes the bottleneck. This demonstrates that optimization efforts should focus on the dominant workload component.

---

## Compiler Optimization Impact

Testing with `-g` (unoptimized) vs `-O2` (optimized) at 1M blocks:

|Program|-g (unopt)|-O2 (opt)|Speedup|
|---|---|---|---|
|alloca|0.585s|0.200s|2.9x|
|list|1.926s|0.298s|6.5x|
|malloc|0.477s|0.224s|2.1x|
|new|1.850s|0.293s|6.3x|

**Result:** Compiler optimization provides 3-6x speedup. C++ container-based programs (`list`, `new`) benefit most from optimization, likely due to better inlining of template code. On second thought I likely missed some of the nuance using compiler optimization consistently in all but the optimization tests. 