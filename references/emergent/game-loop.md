# Game Loop

**Category:** Emergent / modern · **Also known as:** Main loop, tick loop, fixed-timestep loop, simulation loop · **Related:** Event Loop, Scheduler, Entity Component System, Spatial Partitioning, Data-Oriented Design

## Intent
Advance a simulation in discrete, regularly spaced steps (process input → update state → render/emit), decoupling the progression of simulated time from how fast the hardware or the network can go.

## Problem
- Behavior depends on elapsed time (physics, animations, cooldowns, rate-based polling), and the speed of updates must not vary with CPU load or frame rate.
- Scenario: a fleet-tracking dashboard moves vehicles by `speed * 1` per frame. On a 144 Hz monitor vehicles move 2.4x faster than on 60 Hz; on a slow laptop they crawl.
- Scenario: a trading bot polls prices "as fast as it can" in a `while True`, burns a full core, and its rate limiter breaks whenever an API call is slow.
- Smells: `sleep(0.016)` hardcoded inside logic; state updated inside input handlers; logic that is non-deterministic and cannot be replayed in tests.

## Solution and structure
A loop owns time. Each iteration measures the real time elapsed, feeds it into an accumulator, and runs `update(fixed_dt)` as many times as the accumulator allows. It then renders once, optionally interpolating between the last two states (`alpha = accumulator / dt`).

```
 ┌────────────── loop ───────────────┐
 │ now = clock()                      │
 │ acc += now - prev                  │
 │ process_input()                    │
 │ while acc >= DT: update(DT); acc-=DT│   fixed step: deterministic
 │ render(alpha = acc / DT)           │   variable rate: as fast as allowed
 └────────────────────────────────────┘
```
- **Clock**: monotonic time source (`time.monotonic`, `performance.now()`), never wall clock.
- **Update**: pure-ish step function `state, dt → state`. Deterministic given the inputs.
- **Render/emit**: reads state, never mutates it.
- **Spiral-of-death guard**: cap the number of updates per frame (or clamp `acc`) so a long pause does not trigger thousands of catch-up steps.

Variants: *variable timestep* (`update(elapsed)`, simpler, non-deterministic); *fixed timestep + interpolation* (the standard for physics); *fixed rate with sleep* (servers, bots, ticking game servers at 20–64 Hz).

## When to use
- Real-time simulations, games, physics, animations driven by custom logic.
- Multiplayer game servers that broadcast state at a fixed tick rate.
- Control loops and agents that must act at a steady cadence (robotics, polling agents, rate-paced crawlers).
- You need deterministic replays: same inputs + fixed dt → same result.

## When NOT to use
- Event-driven apps (web servers, CRUD UIs): they react to events; an Event Loop and callbacks are the right model. A tick loop wastes CPU when nothing happens.
- Periodic jobs (every 5 minutes): use a Scheduler (cron, `asyncio` task with `sleep`, Celery beat).
- UI animations in the browser: CSS transitions / Web Animations API, or Angular animations. Hand-rolled loops are only needed for canvas/WebGL.
- Don't write your own engine loop inside an engine (Unity, Godot, Phaser, pygame's `Clock`) — hook into its update callbacks.

## Example — Python
asyncio fixed-timestep loop for a tick-based game server (or any paced agent); `update` is a pure function, easy to test.

```python
import asyncio, time
from dataclasses import dataclass, replace

DT = 1 / 20          # 20 ticks per second
MAX_STEPS = 5        # spiral-of-death guard

@dataclass(frozen=True, slots=True)
class Vehicle:
    id: str
    x: float
    vx: float

def update(world: tuple[Vehicle, ...], dt: float) -> tuple[Vehicle, ...]:
    return tuple(replace(v, x=v.x + v.vx * dt) for v in world)

async def run(world: tuple[Vehicle, ...], broadcast, stop: asyncio.Event) -> None:
    prev, acc = time.monotonic(), 0.0
    while not stop.is_set():
        now = time.monotonic()
        acc += now - prev
        prev = now
        steps = 0
        while acc >= DT and steps < MAX_STEPS:
            world = update(world, DT)
            acc -= DT
            steps += 1
        if steps == MAX_STEPS:
            acc = 0.0                       # drop backlog instead of spiralling
        await broadcast(world)              # render/emit: read-only
        await asyncio.sleep(max(0.0, DT - (time.monotonic() - now)))
```

## Example — TypeScript
Browser canvas loop with `requestAnimationFrame`; in Angular run it outside reactive change detection and push only summaries into a signal.

```ts
import { Component, DestroyRef, ElementRef, inject, signal, viewChild } from '@angular/core';

const DT = 1000 / 60;

@Component({ selector: 'app-sim', template: `<canvas #c width="800" height="400"></canvas><p>{{ fps() }} fps</p>` })
export class Sim {
  private canvas = viewChild.required<ElementRef<HTMLCanvasElement>>('c');
  protected fps = signal(0);
  private x = 0;
  private vx = 0.1;            // px per ms

  constructor() {
    let prev = performance.now(), acc = 0, frames = 0, lastFps = prev, id = 0;
    const frame = (now: number) => {
      acc += Math.min(now - prev, 250);            // clamp after tab was hidden
      prev = now;
      while (acc >= DT) { this.x = (this.x + this.vx * DT) % 800; acc -= DT; }
      const ctx = this.canvas().nativeElement.getContext('2d')!;
      ctx.clearRect(0, 0, 800, 400);
      ctx.fillRect(this.x, 180, 40, 40);
      frames++;
      if (now - lastFps > 1000) { this.fps.set(frames); frames = 0; lastFps = now; }
      id = requestAnimationFrame(frame);
    };
    id = requestAnimationFrame(frame);
    inject(DestroyRef).onDestroy(() => cancelAnimationFrame(id));
  }
}
```
With zoneless Angular (default in recent versions) `requestAnimationFrame` does not trigger change detection; only the `fps` signal update schedules a render. Under Zone.js, wrap the loop in `NgZone.runOutsideAngular`.

## In the wild
- Browser `requestAnimationFrame`; pygame `Clock.tick(fps)`; Unity `Update`/`FixedUpdate`; Godot `_process`/`_physics_process`; Phaser, Three.js render loops.
- Game servers (Source engine tick rate, Minecraft 20 TPS), robotics control loops (ROS timers).
- The canonical write-up: Glenn Fiedler, "Fix Your Timestep!".

## Trade-offs
| Pros | Cons |
|---|---|
| Simulation speed independent of hardware | Burns CPU/battery when idle unless it sleeps |
| Fixed step gives determinism and replayability | Interpolation and accumulators add subtle complexity |
| Clear separation of input, update, render | Long frames cause catch-up bursts (needs a guard) |
| Easy to test `update` in isolation | Coarse tick rate adds latency to reactions |

## Pitfalls
- Using wall clock (`time.time`, `Date.now`) — it jumps with NTP/DST. Use monotonic clocks.
- Multiplying by frame count instead of `dt` — speed depends on FPS.
- No cap on catch-up steps → "spiral of death" after a GC pause or a backgrounded tab.
- Mutating state inside render or inside input handlers; queue input and apply it during update.
- Blocking calls (sync HTTP, disk) inside the loop stall every tick; do I/O off-loop and feed results in as input.
- In Angular with Zone.js, a rAF loop triggers change detection 60 times a second; run outside the zone or go zoneless.

## Refactoring path
- **Introduce it:** extract all time-dependent logic into `update(state, dt)`; replace ad hoc sleeps/timers with one loop owner; switch to a monotonic clock; add the accumulator and step cap; move I/O to the edges.
- **Remove it / simplify:** if updates only happen in response to events, delete the loop and react to events directly; if work is periodic but not real-time, replace the loop with a scheduler or `setInterval`/`asyncio` periodic task.

## Related patterns
- [Event Loop](../emergent/event-loop.md) — reacts to I/O readiness; a game loop is a timed driver often running *on* an event loop.
- [Scheduler](../concurrency/scheduler.md) — for coarse periodic work rather than continuous simulation.
- [Entity Component System](../structural/entity-component-system.md) — the usual way to organize what `update` iterates over.
- [Spatial Partitioning](../emergent/spatial-partitioning.md) — keeps per-tick collision/proximity queries cheap.
- [Data-Oriented Design](../optimization/data-oriented-design.md) — lays out state for fast per-tick iteration.
