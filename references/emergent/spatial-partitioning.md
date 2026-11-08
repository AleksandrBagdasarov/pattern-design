# Spatial Partitioning

**Category:** Emergent / modern · **Also known as:** Spatial index, uniform grid, spatial hash, quadtree/octree, k-d tree, R-tree, geohash · **Related:** Game Loop, Sharding, Data-Oriented Design, Caching, Flyweight

## Intent
Organize objects by their position in space so that proximity queries ("what is near this point?", "what collides with this box?", "which drivers are within 2 km?") examine only nearby candidates instead of every object.

## Problem
- Naive proximity checks are O(n) per query and O(n²) for all-pairs collision detection.
- Scenario: a ride-hailing backend matches riders to the nearest available driver by scanning all 40,000 drivers in a city on every request. At 500 requests/second the matcher saturates CPU.
- Scenario: a map dashboard in Angular renders 50,000 markers and runs hover hit-testing over all of them on every mouse move; the UI stutters.
- Smells: nested loops over all objects to find neighbors; `ORDER BY distance(...) LIMIT 10` doing a full table scan; latency growing quadratically with entity count.

## Solution and structure
Map positions to buckets so that near objects share or neighbor buckets; query only the relevant buckets, then do exact checks on the candidates.

```
 Uniform grid (cell = query radius)      Quadtree (adaptive)
 ┌──┬──┬──┬──┐                           ┌─────┬──┬──┐
 │  │ •│  │  │  query point q:           │     │• │  │
 ├──┼──┼──┼──┤  check q's cell +         │  •  ├──┼──┤
 │  │•q│• │  │  8 neighbors only         │     │••│• │
 ├──┼──┼──┼──┤                           ├─────┼──┴──┤
 │  │  │  │  │                           │     │  •  │
 └──┴──┴──┴──┘                           └─────┴─────┘
```
- **Uniform grid / spatial hash**: `cell = (floor(x / s), floor(y / s))` → dict of sets. O(1) insert/move; great when density is fairly uniform and query radius is known.
- **Quadtree/octree**: recursive subdivision where dense; handles clustered data.
- **k-d tree / ball tree**: static point sets, fast k-nearest-neighbor.
- **R-tree**: rectangles/polygons; the standard database spatial index (PostGIS GiST).
- **Geohash / S2 / H3**: hierarchical cell ids on the sphere; usable as string/integer keys in any DB or cache, and for sharding.

Two-phase query: **broad phase** (index → candidates) then **narrow phase** (exact distance/geometry test).

## When to use
- Many objects (thousands+) with frequent proximity, range or collision queries.
- Geospatial features: nearest store, geofencing, delivery zones, map viewport loading.
- Games/simulations: collision detection, AI perception, network interest management.
- Distributing location data across shards or cache keys (geohash/H3 prefixes).

## When NOT to use
- A few hundred objects: a linear scan is fast, simpler and cache-friendly. Measure first.
- Data already in a database with a spatial index: use PostGIS (`ST_DWithin` with a GiST index), MongoDB `2dsphere`, Redis `GEOSEARCH`, Elasticsearch `geo_distance` — don't hand-roll.
- Non-spatial similarity (embeddings): use a vector index (HNSW, IVF — pgvector, FAISS), which is a different family.
- Highly dynamic scenes where rebuilding a tree each frame costs more than it saves; prefer a uniform grid.

## Example — Python
A uniform-grid spatial hash for live driver positions (in-memory matcher). Cell size ≈ typical search radius.

```python
from collections import defaultdict
from dataclasses import dataclass
from math import floor, hypot

type Cell = tuple[int, int]

@dataclass(slots=True)
class Driver:
    id: str
    x: float          # projected metres (e.g. local UTM), not raw lat/lon
    y: float

class SpatialHash:
    def __init__(self, cell_size: float) -> None:
        self.s = cell_size
        self.cells: defaultdict[Cell, set[str]] = defaultdict(set)
        self.drivers: dict[str, Driver] = {}

    def _cell(self, x: float, y: float) -> Cell:
        return floor(x / self.s), floor(y / self.s)

    def upsert(self, d: Driver) -> None:
        if (old := self.drivers.get(d.id)) is not None:
            self.cells[self._cell(old.x, old.y)].discard(d.id)
        self.drivers[d.id] = d
        self.cells[self._cell(d.x, d.y)].add(d.id)

    def within(self, x: float, y: float, r: float) -> list[Driver]:
        cx0, cy0 = self._cell(x - r, y - r)
        cx1, cy1 = self._cell(x + r, y + r)
        found = []
        for cx in range(cx0, cx1 + 1):                 # broad phase: nearby cells only
            for cy in range(cy0, cy1 + 1):
                for did in self.cells.get((cx, cy), ()):
                    d = self.drivers[did]
                    if hypot(d.x - x, d.y - y) <= r:   # narrow phase: exact distance
                        found.append(d)
        return sorted(found, key=lambda d: hypot(d.x - x, d.y - y))
```
In production with Postgres: `SELECT id FROM drivers WHERE ST_DWithin(geog, ST_MakePoint($1,$2)::geography, 2000) ORDER BY geog <-> ST_MakePoint($1,$2)::geography LIMIT 10;` with a GiST index.

## Example — TypeScript
Static points (map markers) indexed with a k-d tree via `kdbush` for hover hit-testing; built once when data changes.

```ts
import KDBush from 'kdbush';               // v4 API: new KDBush(n), add(x, y), finish(); check docs
import { Injectable, computed, signal } from '@angular/core';

export interface Marker { id: string; x: number; y: number; label: string }

@Injectable({ providedIn: 'root' })
export class MarkerIndex {
  readonly markers = signal<Marker[]>([]);

  private readonly index = computed(() => {    // rebuilt only when markers change
    const ms = this.markers();
    const idx = new KDBush(Math.max(ms.length, 1));
    for (const m of ms) idx.add(m.x, m.y);
    idx.finish();
    return idx;
  });

  /** Markers within r screen pixels of the cursor, nearest first. */
  hitTest(x: number, y: number, r = 8): Marker[] {
    const ms = this.markers();
    return this.index()
      .within(x, y, r)
      .map(i => ms[i])
      .sort((a, b) => Math.hypot(a.x - x, a.y - y) - Math.hypot(b.x - x, b.y - y));
  }
}
```

## In the wild
- Databases: PostGIS GiST/SP-GiST (R-tree-like), MySQL spatial indexes, MongoDB `2dsphere`, Redis `GEOADD`/`GEOSEARCH` (geohash in a sorted set), Elasticsearch/OpenSearch geo queries, DynamoDB + geohash keys.
- Libraries: SciPy `cKDTree`, scikit-learn `BallTree`, Shapely `STRtree`, `rtree` (libspatialindex), Uber H3, Google S2; JS `rbush`, `kdbush`, `flatbush`, Turf.js, Leaflet/MapLibre marker clustering (supercluster).
- Games: Box2D/Bullet broad phase (dynamic AABB trees, sweep-and-prune), Unity/Godot physics, MMO interest management grids.

## Trade-offs
| Pros | Cons |
|---|---|
| Proximity queries drop from O(n) to ~O(k) candidates | Extra memory and update cost on every move |
| Scales to large entity counts | Cell size/tree parameters need tuning |
| Geohash/H3 keys enable sharding and caching | Uneven density hurts uniform grids |
| Broad/narrow phase separation keeps exact logic simple | Edge cases at cell boundaries and the antimeridian/poles |

## Pitfalls
- Using raw latitude/longitude as Euclidean coordinates: a degree of longitude shrinks toward the poles. Project to metres or use haversine/geography types.
- Checking only the query point's own cell and missing neighbors across a boundary; include all cells overlapping the query radius.
- Forgetting to remove an object from its old cell on move, leaving ghosts.
- Cell size far from the query radius: too small → many cells per query; too large → many candidates per cell.
- Rebuilding a tree per query or per frame for dynamic data; update incrementally or use a grid.
- Geohash prefix ≠ proximity: two adjacent points can have totally different prefixes across a cell edge; query neighbor cells too.

## Refactoring path
- **Introduce it:** measure that proximity scans dominate; isolate the scan behind `within(point, r)` / `nearest(point, k)`; swap the implementation for a grid or library index; keep the exact-check step; property-test against the brute-force version.
- **Remove it / simplify:** if entity counts stay small, or the data moves into PostGIS/Redis GEO, delete the custom index and use the scan or the database query.

## Related patterns
- [Game Loop](../emergent/game-loop.md) — per-tick collision checks are the classic consumer.
- [Sharding](../cloud/sharding.md) — geohash/H3 cells are natural shard keys.
- [Data-Oriented Design](../optimization/data-oriented-design.md) — flat arrays of coordinates make indexes fast.
- [Caching](../optimization/caching.md) — cell ids make good cache keys for map tiles and area queries.
- [Flyweight](../structural/flyweight.md) — share heavy per-type data among many positioned objects.
