# Entity-Component-System (ECS)

**Category:** Structural (component patterns) · **Also known as:** ECS, Data-oriented entity model · **Related:** Component-Based Design, Data-Oriented Design, Game Loop, Flyweight, Spatial Partitioning, Object Pool

## Intent
Model a world of many things as **entities** (plain IDs) that own **components** (plain data, no behavior), and put all behavior into **systems** that iterate over every entity having a given set of components — separating data from logic and laying data out for fast batch processing.

## Problem
- Thousands to millions of objects are updated every tick (games, simulations, IoT digital twins, particle effects, agent-based models).
- Object-per-thing designs with virtual methods scatter data across the heap; CPU caches miss on every object, and per-object method dispatch dominates.
- Deep game-object hierarchies (`Enemy` → `FlyingEnemy` → `FlyingShootingEnemy`) do not fit runtime combinations.
- Cross-cutting rules ("everything with `Position` and `Velocity` moves") are duplicated across classes.

Scenario: a fleet simulation tracks 200,000 delivery vehicles; each tick updates positions, battery levels and assignment logic. Only some vehicles have `Battery`, some have `Cargo`, all have `Position`.

## Solution and structure
- **Entity**: just an identifier (int). No data, no methods.
- **Component**: a plain data record (`Position(x, y)`, `Velocity(dx, dy)`, `Battery(level)`), stored in per-type storage (arrays, sparse sets or archetype tables).
- **System**: a function that queries "all entities with components A and B" and processes them in bulk (`movement_system`, `battery_system`).
- **World / registry**: creates entities, stores components, runs queries and schedules systems each tick.

```
Entity ids:   0    1    2    3
Position:   [p0] [p1] [p2] [p3]      <- contiguous arrays per component
Velocity:   [v0]      [v2] [v3]
Battery:         [b1] [b2]
movement_system: for e in query(Position, Velocity): pos[e] += vel[e] * dt
```
Storage strategies: **archetypes** (entities with the same component set share a table; fast iteration, slower add/remove; Unity DOTS, Bevy, flecs) vs. **sparse sets** (fast add/remove; EnTT).

## When to use
- Large numbers of homogeneous-ish entities processed every frame/tick.
- Entities gain and lose capabilities dynamically (status effects, equipment).
- Performance limited by memory access patterns, or you want to parallelise systems.
- Clear, data-driven rules that apply to "anything with X and Y".

## When NOT to use
- Business/CRUD applications (orders, users, invoices): use plain domain objects, a database and services; ECS adds indirection with no gain.
- Small counts (hundreds of entities): ordinary objects or Component-Based Design are simpler.
- Logic that is mostly about one entity's rich invariants (domain model) rather than bulk transformation.
- In Python, a hand-written ECS over dicts gains organisation, not speed; for speed, use NumPy arrays (vectorised systems) or a native engine.

## Example — Python
A minimal ECS where components are NumPy columns and systems are vectorised functions; this is where Python actually benefits.

```python
from dataclasses import dataclass, field

import numpy as np


@dataclass
class World:
    capacity: int
    alive: np.ndarray = field(init=False)
    pos: np.ndarray = field(init=False)          # Position component (x, y)
    vel: np.ndarray = field(init=False)          # Velocity component
    battery: np.ndarray = field(init=False)      # Battery component, NaN = absent
    _next: int = 0

    def __post_init__(self) -> None:
        n = self.capacity
        self.alive = np.zeros(n, dtype=bool)
        self.pos = np.zeros((n, 2))
        self.vel = np.zeros((n, 2))
        self.battery = np.full(n, np.nan)

    def spawn(self, x: float, y: float, vx: float, vy: float,
              battery: float | None = None) -> int:
        e = self._next; self._next += 1
        self.alive[e] = True
        self.pos[e], self.vel[e] = (x, y), (vx, vy)
        if battery is not None:
            self.battery[e] = battery
        return e


def movement_system(w: World, dt: float) -> None:          # query: Position + Velocity
    w.pos[w.alive] += w.vel[w.alive] * dt


def battery_system(w: World, dt: float) -> None:           # query: Battery + Velocity
    has = w.alive & ~np.isnan(w.battery)
    speed = np.linalg.norm(w.vel[has], axis=1)
    w.battery[has] = np.maximum(0.0, w.battery[has] - 0.01 * speed * dt)
    w.vel[has & (w.battery <= 0)] = 0.0                     # empty battery stops vehicle


world = World(capacity=200_000)
for i in range(200_000):
    world.spawn(0, 0, 1.0, 0.5, battery=100.0 if i % 2 else None)
for _ in range(60):                                         # game/simulation loop
    movement_system(world, 1 / 60); battery_system(world, 1 / 60)
```

## Example — TypeScript
Structure-of-arrays with typed arrays and a bitmask query, suitable for a canvas/WebGL visualisation in the browser.

```ts
const enum C { Position = 1 << 0, Velocity = 1 << 1, Selected = 1 << 2 }

export class World {
  readonly mask: Uint8Array;
  readonly x: Float32Array; readonly y: Float32Array;
  readonly vx: Float32Array; readonly vy: Float32Array;
  private next = 0;

  constructor(readonly capacity: number) {
    this.mask = new Uint8Array(capacity);
    this.x = new Float32Array(capacity); this.y = new Float32Array(capacity);
    this.vx = new Float32Array(capacity); this.vy = new Float32Array(capacity);
  }

  spawn(x: number, y: number, vx = 0, vy = 0): number {
    const e = this.next++;
    this.x[e] = x; this.y[e] = y; this.vx[e] = vx; this.vy[e] = vy;
    this.mask[e] = C.Position | (vx || vy ? C.Velocity : 0);
    return e;
  }

  *query(required: number): Generator<number> {
    for (let e = 0; e < this.next; e++) if ((this.mask[e] & required) === required) yield e;
  }
}

export function movementSystem(w: World, dt: number): void {
  for (const e of w.query(C.Position | C.Velocity)) { w.x[e] += w.vx[e] * dt; w.y[e] += w.vy[e] * dt; }
}

// In Angular: run systems in requestAnimationFrame outside change detection,
// draw to <canvas>, and expose only aggregates (counts, selection) as signals.
```

## In the wild
- Game engines: Unity DOTS/Entities, Bevy (Rust), EnTT (C++), flecs (C/C++), Godot (node-based, not ECS by default).
- JS: bitECS, becsy, Miniplex for browser games and visualisations.
- Python: `esper` (a small pure-Python ECS library; organisational benefits, not speed).
- The idea of columnar storage + set-based queries is the same as in pandas/Polars, Arrow and columnar databases.

## Trade-offs
| Pros | Cons |
|---|---|
| Cache-friendly, vectorisable, parallelisable systems | Unfamiliar model; logic scattered across systems |
| Runtime composition: add/remove components freely | Entity relationships (parent/child, references) are awkward |
| Rules expressed once for all matching entities | System ordering and inter-system dependencies need scheduling |
| Easy to serialise/snapshot world state | Overkill and slower to develop for typical business apps |

## Pitfalls
- **Putting methods and references in components** (turning them back into objects). Fix: components are data; behavior goes in systems.
- **Holding raw entity IDs after destruction** (ID reused, wrong entity). Fix: generational IDs (index + generation).
- **Adding/removing components while iterating** a query. Fix: defer structural changes with a command buffer applied after the system runs.
- **Implicit system order** causing one-frame lag bugs. Fix: explicit schedule/stages.
- **Python loops per entity** that are slower than plain objects. Fix: vectorise with NumPy, or accept ECS only for structure.

## Refactoring path
- **Introduce it:** profile first; pick the hottest per-object update loop; move the involved fields into arrays keyed by entity index; turn the method into a system over the arrays; migrate other behaviors one system at a time.
- **Remove it / simplify:** if entity counts are small and logic is entity-centric, convert components back into fields of a class and systems into methods (Component-Based Design).

## Related patterns
- [Component-Based Design](../structural/component-based-design.md) — objects own components with behavior; ECS separates data and logic.
- [Data-Oriented Design](../optimization/data-oriented-design.md) — the memory-layout philosophy behind ECS.
- [Game Loop](../emergent/game-loop.md) — runs the systems each tick.
- [Spatial Partitioning](../emergent/spatial-partitioning.md) — accelerates proximity queries over entities.
- [Object Pool](../creational/object-pool.md) — recycling entity slots and component storage.
- [Flyweight](../structural/flyweight.md) — share immutable data (meshes, sprites) referenced by components.
