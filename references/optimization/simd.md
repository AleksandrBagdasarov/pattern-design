# SIMD (Single Instruction, Multiple Data)

**Category:** Optimization · **Also known as:** Vectorization, array programming · **Related:** Data-Oriented Design, Micro-Optimizations, Pipes and Filters, Thread Pool

## Intent
Apply one operation to many data elements at once — at the hardware level with vector instructions (SSE/AVX/NEON), and at the code level by expressing work as whole-array operations that a native library or compiler can vectorize.

## Problem
- A numeric hot loop processes millions of elements one at a time: price conversions, image pixels, sensor readings, scoring, distance calculations.
- In Python, the per-element interpreter overhead (~50–100 ns per operation) dwarfs the actual arithmetic (~1 ns); in JS, scalar loops leave the CPU's vector units idle.

Scenario: computing cosine similarity between a query embedding and 500k product embeddings (dimension 384) takes 40 s in pure Python loops; one NumPy matrix-vector product takes ~50 ms.

Profile first: `py-spy`/`scalene` (scalene shows the Python-vs-native time split — a high Python share in a numeric loop is the SIMD signal); `perf stat` for vector-instruction counts; Chrome DevTools Performance for long JS tasks in numeric code.

## Solution and structure
```
scalar:     for i in range(n): c[i] = a[i] * b[i]       → n interpreted iterations
vectorized: c = a * b                                    → one call into C; loop runs 4–16 lanes/instruction
```
Levels, from most to least practical for application developers:
1. **Use a vectorized library**: NumPy, Polars, PyArrow compute, SciPy, scikit-learn, PyTorch; in JS, typed-array libraries, WebGPU/WebGL, or WebAssembly modules.
2. **Rewrite loops as array expressions**: masks instead of `if` (`np.where`), reductions (`sum`, `max`), broadcasting instead of nested loops.
3. **Compile with auto-vectorization**: Numba `@njit`, Cython, Rust (PyO3) — the compiler emits SIMD for simple loops over contiguous arrays.
4. **Explicit intrinsics / SIMD types**: Rust `std::simd` (nightly), C intrinsics, WebAssembly SIMD128 (`v128`) — only for library authors.

Prerequisite: contiguous, homogeneous, aligned data (see Data-Oriented Design). SIMD cannot help with arrays of Python objects or JS object graphs.

## When to use
- Numeric work over large homogeneous arrays: finance calculations on columns, embeddings/similarity, image/audio processing, statistics, simulations.
- Data already in (or cheaply convertible to) NumPy/Arrow/typed arrays.
- The same operation applies to all elements, with simple branching expressible as masks.

## When NOT to use
- Small arrays (dozens of elements): call overhead into native code exceeds the gain.
- Branch-heavy, per-element business logic (each order has different rules).
- Data that must first be converted from objects on every call: conversion cost can dominate; restructure the data model instead.
- Exact decimal money arithmetic where `float64` rounding is unacceptable: use integer cents arrays (`int64`) — still vectorizable — not floats.
- I/O-bound workloads.

## Example — Python
```python
import numpy as np
from numpy.typing import NDArray

def top_k_similar(query: NDArray[np.float32], catalog: NDArray[np.float32], k: int = 10) -> NDArray[np.intp]:
    """catalog: (n, d) L2-normalized embeddings; query: (d,) normalized."""
    scores = catalog @ query                         # BLAS matvec: SIMD + multithreaded
    idx = np.argpartition(scores, -k)[-k:]           # O(n) selection, not a full sort
    return idx[np.argsort(scores[idx])[::-1]]

def apply_discounts(cents: NDArray[np.int64], tier: NDArray[np.int8]) -> NDArray[np.int64]:
    # branch → mask; integer arithmetic keeps money exact
    pct = np.select([tier == 2, tier == 1], [20, 10], default=0).astype(np.int64)
    return cents - (cents * pct + 50) // 100        # round half up on non-negative cents

# Scalar baseline for comparison (do not ship):
def apply_discounts_slow(cents: list[int], tier: list[int]) -> list[int]:
    return [c - (c * {2: 20, 1: 10}.get(t, 0) + 50) // 100 for c, t in zip(cents, tier)]

# When the loop can't be expressed as array ops, compile it:
from numba import njit

@njit(cache=True, fastmath=False)
def running_drawdown(prices: NDArray[np.float64]) -> NDArray[np.float64]:
    out = np.empty_like(prices); peak = prices[0]
    for i in range(prices.size):
        peak = max(peak, prices[i]); out[i] = (prices[i] - peak) / peak
    return out
```
Verify vectorized and scalar versions agree (Hypothesis with `hypothesis.extra.numpy.arrays`) before deleting the baseline. Running max has a dependency chain, so Numba gains come mostly from removing interpreter overhead; `np.maximum.accumulate(prices)` is the array-level alternative.

## Example — TypeScript
```ts
// JS has no portable SIMD API (SIMD.js was withdrawn). Options: typed-array loops the JIT
// optimizes well, WebAssembly SIMD128 modules, or GPU (WebGPU/WebGL) for very large data.

export function dot(a: Float32Array, b: Float32Array): number {
  let s0 = 0, s1 = 0, s2 = 0, s3 = 0;              // 4 independent accumulators: more ILP
  const n = a.length - (a.length % 4);
  for (let i = 0; i < n; i += 4) {
    s0 += a[i]! * b[i]!; s1 += a[i + 1]! * b[i + 1]!;
    s2 += a[i + 2]! * b[i + 2]!; s3 += a[i + 3]! * b[i + 3]!;
  }
  let s = s0 + s1 + s2 + s3;
  for (let i = n; i < a.length; i++) s += a[i]! * b[i]!;
  return s;
}

// WebAssembly SIMD: compile Rust/C with SIMD enabled, load in a Web Worker
export async function loadSimdKernel(url: string) {
  const { instance } = await WebAssembly.instantiateStreaming(fetch(url), {});
  const exp = instance.exports as { memory: WebAssembly.Memory; dot_f32: (pa: number, pb: number, n: number) => number };
  return (a: Float32Array, b: Float32Array): number => {
    const heap = new Float32Array(exp.memory.buffer);  // assumes a and b fit at offsets 0 and n
    heap.set(a, 0); heap.set(b, a.length);
    return exp.dot_f32(0, a.length * 4, a.length);
  };
}
```
Feature-detect Wasm SIMD (e.g. the `wasm-feature-detect` package) and keep a scalar fallback. Run heavy kernels in a Web Worker so the Angular UI thread stays responsive; in Node, native addons or `onnxruntime-node`/`@tensorflow/tfjs-node` provide SIMD-backed kernels.

## In the wild
- NumPy ufuncs (with runtime CPU dispatch for SSE/AVX/AVX-512/NEON), BLAS/LAPACK (OpenBLAS, MKL, Accelerate), Polars/Arrow compute kernels, `simdjson`/`orjson`, Numba, PyTorch.
- Databases: DuckDB, ClickHouse, Velox use vectorized execution engines.
- Web: WebAssembly SIMD128 (supported in all major browsers), WebGPU compute shaders, TensorFlow.js/ONNX Runtime Web backends.
- Rust `std::simd` (portable SIMD; still nightly at last check — verify), C++ `std::experimental::simd`, Java Vector API (incubator).

## Trade-offs
| Pros | Cons |
|---|---|
| 10–1000× speedups over interpreted scalar loops | Requires contiguous homogeneous data layout |
| Libraries give it for free with array syntax | Branchy logic is awkward (masks compute both sides) |
| Lower energy per element | Float results may differ slightly from scalar order of operations |
| Scales with hardware (wider vectors, more cores) | Debugging/vector shape errors; Wasm/native toolchains add build complexity |

## Pitfalls
- **Hidden Python loops:** `np.vectorize`, `df.apply(lambda …)` and `list(map(...))` are *not* vectorized. Use ufuncs and expressions.
- **Accidental copies and temporaries** in chained expressions on huge arrays: use `out=`, in-place ops, or `numexpr`/Polars lazy expressions.
- **dtype surprises:** int overflow wraps silently in NumPy fixed-width ints; `float32` precision for money. Choose dtypes deliberately.
- **Non-contiguous views** (strided slices, Fortran order) slow kernels; `np.ascontiguousarray` when needed.
- **Benchmarking the first Numba call** includes compile time; warm up or use `cache=True`.

## Refactoring path
- **Introduce it:** profile → confirm interpreter-bound numeric loop → move data into arrays once (DOD) → replace loop with array operations; if impossible, Numba/Cython → equivalence test vs scalar baseline → measure.
- **Remove it / simplify:** if arrays are small or conversions dominate, revert to plain loops; if a native dependency (Numba, Wasm toolchain) causes deployment pain for little gain, fall back to NumPy-level vectorization.

## Related patterns
- [Data-Oriented Design](../optimization/data-oriented-design.md) — the data layout SIMD requires.
- [Micro-Optimizations](../optimization/micro-optimizations.md) — scalar tuning; SIMD replaces the loop entirely.
- [Pipes and Filters](../architectural/pipes-and-filters.md) — vectorized engines process batches through filter stages.
- [Thread Pool](../concurrency/thread-pool.md) — combine with SIMD for multi-core parallelism (many NumPy/Polars ops release the GIL).
