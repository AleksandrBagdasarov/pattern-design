# Object Recycling

**Category:** Optimization · **Also known as:** Free list, buffer reuse, object reuse · **Related:** Object Pool, Flyweight, Micro-Optimizations, Data-Oriented Design

## Intent
Reuse already-allocated objects or buffers in a hot path — reset and hand them out again — instead of allocating new ones each time, to cut allocation cost and garbage-collector pressure.

## Problem
- A profiler or GC log shows allocation churn dominating: a game/animation loop creating thousands of short-lived vectors per frame; a parser allocating a new `bytearray` per packet; a canvas chart creating point objects on every `requestAnimationFrame`.
- Symptoms: periodic frame drops or latency spikes aligned with GC pauses; DevTools Performance shows frequent "Minor GC" slices; Python `gc` callbacks or `tracemalloc` / `memray` show millions of tiny allocations.

Profile first: Chrome DevTools Performance (GC events, JS heap line) and Memory → Allocation sampling / allocation timeline; `node --trace-gc`; Python `memray run`, `tracemalloc`, `gc.callbacks` timing, `py-spy`. Recycling only helps when allocation is actually the measured cost.

Object Pool vs Object Recycling: a **pool** manages scarce or expensive-to-create resources (DB connections, threads) with checkout/return semantics and limits. **Recycling** targets cheap-but-numerous objects where the cost is allocation and GC volume, often single-threaded and in-place.

## Solution and structure
```
hot loop ──acquire()──▶ free list ── empty? ──▶ allocate new
     │                     ▲
     └── use, then release(obj) ── reset(obj) ─┘
```
Forms, from simplest:
- **Preallocated buffer reused in place**: allocate once, overwrite each iteration (`bytearray`, `memoryview`, `Float32Array`, NumPy `out=` parameter).
- **Free list**: stack of reset objects; `acquire` pops or allocates, `release` resets and pushes.
- **Double buffering / ping-pong**: two buffers swapped per frame.
- **Arena**: allocate everything for one request/frame together and drop it at once.

## When to use
- Measured GC pressure in real-time loops: rendering, games, audio, high-rate parsing, streaming.
- Large buffers (MBs) repeatedly allocated for I/O or encoding.
- Objects with expensive construction but cheap reset.

## When NOT to use
- Typical web/CRUD code: modern generational GCs (V8, JVM) make short-lived allocations very cheap, and CPython's reference counting frees most objects immediately. Recycling adds bugs for no gain.
- Objects that escape (stored in state, sent to other components, emitted on observables): reuse corrupts data held elsewhere.
- Immutable value objects and Angular signal state: reuse breaks reference-equality change detection. Use new objects there.
- Scarce external resources (connections): use an Object Pool, which has limits, health checks and timeouts.

## Example — Python
```python
import socket
from collections.abc import Iterator

HEADER = 4

def read_frames(sock: socket.socket, max_frame: int = 1 << 20) -> Iterator[memoryview]:
    """Yields length-prefixed frames, reusing ONE buffer. Callers must copy if they keep data."""
    buf = bytearray(max_frame)                    # allocated once
    view = memoryview(buf)
    while True:
        if not _recv_exact(sock, view[:HEADER]):
            return
        size = int.from_bytes(view[:HEADER], "big")
        if size > max_frame:
            raise ValueError(f"frame too large: {size}")
        if not _recv_exact(sock, view[:size]):
            return
        yield view[:size]                         # zero-copy slice, valid until next iteration

def _recv_exact(sock: socket.socket, target: memoryview) -> bool:
    got = 0
    while got < len(target):
        n = sock.recv_into(target[got:])          # writes into existing memory, no new bytes object
        if n == 0:
            return False
        got += n
    return True

# NumPy equivalent: reuse output arrays instead of allocating per batch
#   np.multiply(prices, qty, out=totals)     # totals preallocated once
```
In Python, recycling pure-Python objects rarely pays: attribute resets cost about as much as creating a small object. The real wins are large buffers (`recv_into`, `readinto`, NumPy `out=`), which avoid big allocations and copies.

## Example — TypeScript
```ts
interface Particle { x: number; y: number; vx: number; vy: number; ttl: number; alive: boolean }

export class ParticleRecycler {
  private readonly free: Particle[] = [];
  readonly active: Particle[] = [];

  spawn(x: number, y: number, vx: number, vy: number, ttl: number): void {
    const p = this.free.pop() ?? { x: 0, y: 0, vx: 0, vy: 0, ttl: 0, alive: false };  // same shape always
    p.x = x; p.y = y; p.vx = vx; p.vy = vy; p.ttl = ttl; p.alive = true;
    this.active.push(p);
  }

  step(dt: number): void {
    let w = 0;
    for (let i = 0; i < this.active.length; i++) {
      const p = this.active[i]!;
      p.x += p.vx * dt; p.y += p.vy * dt; p.ttl -= dt;
      if (p.ttl > 0) this.active[w++] = p;              // compact in place, no new array
      else { p.alive = false; this.free.push(p); }      // recycle
    }
    this.active.length = w;
  }
}

// Reusable scratch buffer for a canvas chart: no Float32Array allocation per frame
const scratch = new Float32Array(10_000 * 2);
export function projectPoints(values: ArrayLike<number>, width: number, height: number, max: number): Float32Array {
  const n = Math.min(values.length, scratch.length / 2);
  for (let i = 0; i < n; i++) { scratch[2 * i] = (i / n) * width; scratch[2 * i + 1] = height - (values[i]! / max) * height; }
  return scratch.subarray(0, n * 2);                    // view, valid until next call
}
```
Run the loop outside Angular change detection (or in a zoneless app with no signal writes per frame) and push only summary state into signals.

## In the wild
- Python: `socket.recv_into`, `io.BufferedReader.readinto`, `memoryview`, NumPy `out=` ufunc parameter, `asyncio.BufferedProtocol` (`get_buffer`); CPython's internal free lists for floats, tuples, frames.
- JS: Phaser/Three.js object pools for vectors and particles, `ArrayBuffer`/`TypedArray` reuse, Node `Buffer.allocUnsafe` slab pooling, `Buffer.poolSize`.
- Netty `ByteBuf` pooling, Go `sync.Pool`, .NET `ArrayPool<T>`, Android `RecyclerView` (view recycling) and Angular CDK virtual scroll (reuses DOM views).

## Trade-offs
| Pros | Cons |
|---|---|
| Fewer allocations and GC pauses in hot loops | Use-after-release bugs: stale data visible elsewhere |
| Predictable memory footprint | Manual lifecycle in a GC language |
| Big wins for large buffers (no copy, no alloc) | Little or no gain for small objects on modern GCs/CPython |
| Stable object shapes help JIT optimization | Pools retain peak memory forever unless trimmed |

## Pitfalls
- **Leaking a recycled object** (storing a yielded `memoryview` or a returned `subarray` beyond the next call). Document "valid until next call"; copy (`bytes(view)`, `slice()`) when retaining.
- **Incomplete reset:** old fields leak into the next use. Reset every field in one place.
- **Double release:** the same object handed out twice. Track an `alive` flag in debug builds.
- **Unbounded free lists** after a spike: cap the free list size.
- **Reuse with immutability-based change detection** (signals, OnPush, React): mutated-in-place objects do not trigger updates.

## Refactoring path
- **Introduce it:** profile shows GC/allocation as top cost → reuse the single biggest buffer first (`recv_into`, `out=`, preallocated typed array) → measure → only then add free lists for numerous small objects.
- **Remove it / simplify:** if profiling no longer shows GC pressure (runtime upgrade, smaller workload) or reuse bugs appear, return to plain allocation; keep buffer reuse only where it measurably helps.

## Related patterns
- [Object Pool](../creational/object-pool.md) — managed reuse of scarce, expensive resources with limits.
- [Flyweight](../structural/flyweight.md) — share immutable intrinsic state instead of reusing mutable objects.
- [Micro-Optimizations](../optimization/micro-optimizations.md) — recycling is one of the allocation-reduction techniques.
- [Data-Oriented Design](../optimization/data-oriented-design.md) — struct-of-arrays removes per-object allocations entirely.
