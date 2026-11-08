# Flyweight

**Category:** Structural · **Also known as:** Interning, Hash-consing · **Related:** Factory, Multiton, Object Pool, Value Object, Data-Oriented Design, Caching

## Intent
Support very large numbers of fine-grained objects efficiently by sharing the parts of their state that are identical and immutable (intrinsic state), while passing the varying parts (extrinsic state) in from outside.

## Problem
- Memory grows linearly with object count, and most of each object is duplicated data: 2 million order lines each holding a full copy of the product's name, category, tax rules and currency.
- Allocation and GC churn dominate profiles in rendering, parsing or simulation.
- The duplicated data never changes after creation.

Scenario: a map view renders 100,000 markers. Each marker has a position (unique) and a style (icon, color, size, label font) from a set of 12 styles. Storing the style per marker wastes memory; sharing 12 style objects does not.

## Solution and structure
- **Flyweight**: an immutable object holding intrinsic (shared) state.
- **Flyweight factory**: returns an existing flyweight for a key or creates and caches one. Clients must use the factory so sharing actually happens.
- **Context / client**: holds extrinsic state (position, quantity, owner) plus a reference to a flyweight; passes extrinsic state to flyweight operations when needed.

```
Marker(x, y, style=●)──┐
Marker(x, y, style=●)──┼──> MarkerStyle("hospital", red, 24px)   (shared, frozen)
Marker(x, y, style=●)──┘
```
Precondition: intrinsic state must be immutable. A mutable shared object is a global-state bug.

## When to use
- The application uses a very large number of objects (rule of thumb: 10^5 and more) *and* memory or allocation is a measured problem.
- Most object state can be made extrinsic or is shared among many objects.
- Object identity of the shared part does not matter to the application.

## When NOT to use
- Without a profile showing memory pressure: it complicates code for no gain.
- When objects are few (hundreds or thousands): Python object overhead is irrelevant there.
- When the "shared" state actually varies or is mutated per instance.
- When a columnar/array layout (NumPy, pandas categoricals, typed arrays) solves it better: that is Data-Oriented Design and usually beats object-level flyweights by an order of magnitude.

## Example — Python
`functools.cache` is a ready-made flyweight factory; frozen slotted dataclasses make the flyweights immutable and small.

```python
from dataclasses import dataclass
from decimal import Decimal
from functools import cache


@dataclass(frozen=True, slots=True)
class ProductInfo:                       # flyweight: intrinsic, immutable
    sku: str
    name: str
    category: str
    vat_rate: Decimal
    currency: str


@cache                                   # flyweight factory
def product_info(sku: str) -> ProductInfo:
    row = CATALOG[sku]                   # one DB/catalog lookup per SKU
    return ProductInfo(sku, row["name"], row["category"], Decimal(row["vat"]), "EUR")


@dataclass(slots=True)
class OrderLine:                         # context: extrinsic state + reference
    order_id: int
    qty: int
    unit_price: Decimal
    product: ProductInfo

    def gross(self) -> Decimal:
        return self.qty * self.unit_price * (1 + self.product.vat_rate)


CATALOG = {"SKU-1": {"name": "Coffee beans 1kg", "category": "grocery", "vat": "0.07"}}

lines = [OrderLine(i, 2, Decimal("12.50"), product_info("SKU-1")) for i in range(1_000_000)]
assert lines[0].product is lines[-1].product      # one shared object, not a million
```
Python itself does this for small ints, interned strings (`sys.intern`) and `None`/`True`/`False`. For pure tabular data, `pandas` `category` dtype or Polars `Categorical` apply the same idea column-wise.

## Example — TypeScript
Sharing icon/style objects across thousands of map markers.

```ts
interface MarkerStyle {
  readonly icon: string;
  readonly color: string;
  readonly size: number;
}

class MarkerStyleFactory {
  private readonly cache = new Map<string, MarkerStyle>();

  get(kind: 'hospital' | 'school' | 'shop', highlighted = false): MarkerStyle {
    const key = `${kind}:${highlighted}`;
    let style = this.cache.get(key);
    if (!style) {
      style = Object.freeze({
        icon: `/icons/${kind}.svg`,
        color: highlighted ? '#e33' : '#36c',
        size: highlighted ? 32 : 24,
      });
      this.cache.set(key, style);
    }
    return style;
  }
}

interface Marker { readonly lat: number; readonly lng: number; readonly style: MarkerStyle }

const styles = new MarkerStyleFactory();
const markers: Marker[] = places.map((p) => ({ lat: p.lat, lng: p.lng, style: styles.get(p.kind) }));

declare const places: { lat: number; lng: number; kind: 'hospital' | 'school' | 'shop' }[];
```
For 10^6 points, go further: store `lat`/`lng` in `Float64Array`s and a `Uint8Array` of style indices (data-oriented layout).

## In the wild
- Python: small-int cache, `sys.intern`, `enum` members, `re` compiled-pattern cache, `functools.cache`.
- Java `Integer.valueOf` cache and `String.intern`; .NET string interning.
- Browsers share glyph and font data across text nodes; game engines share meshes/textures across instances (GPU instancing).
- Angular/React virtual-scroll lists reuse a small pool of row components for thousands of rows (closer to Object Pool, same motivation).
- pandas/Arrow dictionary-encoded (categorical) columns.

## Trade-offs
| Pros | Cons |
|---|---|
| Large memory savings when duplication is high | More complex: state split into intrinsic and extrinsic |
| Fewer allocations, less GC pressure | Shared objects must be strictly immutable |
| Identity comparison (`is`/`===`) becomes cheap equality | Factory cache can itself leak memory if keys are unbounded |
| Central place to build expensive shared state | CPU cost of passing/recomputing extrinsic state |

## Pitfalls
- **Mutable flyweight**: one caller changes `style.color` and every marker changes. Fix: `frozen=True`, `Object.freeze`, `readonly`.
- **Bypassing the factory** (`ProductInfo(...)` constructed directly), so nothing is shared. Fix: make the factory the only public constructor path.
- **Unbounded cache** keyed by user input. Fix: bounded cache (`lru_cache(maxsize=...)`) or `weakref.WeakValueDictionary`.
- **Thread safety of the factory**: two threads create two "shared" objects. Usually harmless if immutable; if identity matters, lock creation.
- **Premature use**: applying it to 500 objects. Fix: measure with `tracemalloc` / Chrome heap snapshot first.

## Refactoring path
- **Introduce it:** profile memory; identify fields duplicated across many instances; move them into a frozen class; add a cached factory; replace the fields with a reference; verify with a heap measurement.
- **Remove it / simplify:** if counts are small or the shared part needs per-instance mutation, inline the fields back; if data is tabular, replace objects with columnar arrays instead.

## Related patterns
- [Factory](../creational/factory.md) — the flyweight factory is a caching factory.
- [Multiton](../creational/multiton.md) — one instance per key; Flyweight adds the intrinsic/extrinsic split for memory.
- [Object Pool](../creational/object-pool.md) — reuses mutable objects over time; Flyweight shares immutable objects simultaneously.
- [Value Object](../emergent/value-object.md) — immutable values are natural flyweight candidates.
- [Data-Oriented Design](../optimization/data-oriented-design.md) — the array-based alternative for very large counts.
- [Caching](../optimization/caching.md) — the factory's cache follows normal cache-sizing rules.
