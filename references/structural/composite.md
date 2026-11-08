# Composite

**Category:** Structural · **Also known as:** Part-whole hierarchy, Object tree · **Related:** Decorator, Visitor, Iterator, Chain of Responsibility, Interpreter, Flyweight, Specification

## Intent
Compose objects into tree structures and let clients treat individual objects (leaves) and groups (composites) through the same interface.

## Problem
- Client code is full of `if isinstance(node, Folder): for child in node.children: ... else: ...`, repeated for every operation.
- The structure is naturally recursive: product bundles containing products and other bundles, org units containing people and units, UI panels containing widgets and panels, filter expressions containing filters.
- Adding a new node type means editing every traversal.

Scenario: a shop sells products and bundles ("Office starter kit" = desk + chair + "Cable pack" bundle). Price, weight and stock checks must work on either, recursively, with bundle discounts.

## Solution and structure
- **Component**: the common interface (`price()`, `weight()`).
- **Leaf**: a node without children; implements the operations directly.
- **Composite**: holds children (Components); implements operations by delegating to children and combining results.
- **Client**: works with Component only.

```
          Component
          ▲       ▲
        Leaf    Composite ◆── children: list[Component]
```
Design choice: put `add()/remove()` only on Composite (type-safe, the common modern choice) or on Component (uniform but leaves must reject them).

## When to use
- The domain is a genuine part-whole tree and clients should not care about depth.
- The same operations apply recursively (sum, render, validate, evaluate, search).
- You build trees from configuration or user input (rule builders, menus, page layouts).

## When NOT to use
- The structure is flat or fixed-depth (an order has lines; lines have no children): use a list.
- Leaves and composites need very different operations; a forced common interface becomes meaningless.
- Graphs with sharing or cycles: Composite assumes a tree; use an explicit graph model.
- Deep trees in hot paths with recursion limits (Python default ~1000): consider iterative traversal.

## Example — Python
Product bundles with recursive pricing; dataclasses and a Protocol, no base class needed.

```python
from dataclasses import dataclass, field
from decimal import Decimal
from typing import Protocol


class Item(Protocol):                       # Component
    name: str
    def price(self) -> Decimal: ...
    def weight_kg(self) -> Decimal: ...


@dataclass
class Product:                              # Leaf
    name: str
    unit_price: Decimal
    weight: Decimal

    def price(self) -> Decimal: return self.unit_price
    def weight_kg(self) -> Decimal: return self.weight


@dataclass
class Bundle:                               # Composite
    name: str
    items: list[Item] = field(default_factory=list)
    discount: Decimal = Decimal("0")        # 0.10 = 10% off

    def add(self, item: Item) -> "Bundle":
        self.items.append(item)
        return self

    def price(self) -> Decimal:
        subtotal = sum((i.price() for i in self.items), Decimal("0"))
        return (subtotal * (1 - self.discount)).quantize(Decimal("0.01"))

    def weight_kg(self) -> Decimal:
        return sum((i.weight_kg() for i in self.items), Decimal("0"))


cables = Bundle("Cable pack").add(Product("HDMI", Decimal("9.99"), Decimal("0.2"))) \
                             .add(Product("USB-C", Decimal("14.99"), Decimal("0.1")))
kit = Bundle("Office starter", discount=Decimal("0.10")) \
    .add(Product("Desk", Decimal("249"), Decimal("30"))) \
    .add(Product("Chair", Decimal("179"), Decimal("12"))) \
    .add(cables)

print(kit.price(), kit.weight_kg())        # client doesn't care about depth
```

## Example — TypeScript
A recursive Angular component rendering a category tree; the template is the Composite's operation.

```ts
import { Component, input } from '@angular/core';

export type Node =
  | { kind: 'leaf'; id: string; label: string; count: number }
  | { kind: 'group'; id: string; label: string; children: Node[] };

export const total = (n: Node): number =>
  n.kind === 'leaf' ? n.count : n.children.reduce((s, c) => s + total(c), 0);

@Component({
  selector: 'app-tree-node',
  // A standalone component can use its own selector recursively without importing itself.
  template: `
    <span>{{ node().label }} ({{ count() }})</span>
    @if (node().kind === 'group') {
      <ul>
        @for (child of children(); track child.id) {
          <li><app-tree-node [node]="child" /></li>
        }
      </ul>
    }
  `,
})
export class TreeNode {
  readonly node = input.required<Node>();
  protected count = () => total(this.node());
  protected children = () => { const n = this.node(); return n.kind === 'group' ? n.children : []; };
}
```
In TS, a discriminated union is often a better Composite than a class hierarchy: the compiler checks every `kind` is handled. Use `computed()` instead of plain methods if `total()` is expensive.

## In the wild
- DOM: `Node`/`Element` trees; Angular/React/Vue component trees.
- Python `ast` nodes, `pathlib`-based file trees, `xml.etree.ElementTree`, `tkinter`/Qt widget hierarchies.
- SQLAlchemy `and_()`/`or_()` expressions; Django `Q` objects; Pydantic nested models.
- Angular `FormGroup`/`FormArray`/`FormControl`: one `AbstractControl` interface, `valid`/`value` computed recursively — a textbook Composite.
- Infrastructure: AWS CDK construct tree, Terraform modules, Kubernetes owner references.

## Trade-offs
| Pros | Cons |
|---|---|
| Clients treat leaves and trees uniformly | Overly general interface; hard to restrict which children are allowed |
| New node types added without changing clients | Recursive operations can be slow or hit recursion limits |
| Natural fit for recursive domains and UIs | Uniform `add()` on leaves is unsafe; restricted `add()` loses uniformity |
| Operations compose (sum, render, validate) | Cross-cutting operations over many node types may need Visitor |

## Pitfalls
- **Cycles** (a bundle containing itself) cause infinite recursion. Fix: validate on `add()` or model as a DAG with checks.
- **Recomputing totals on every access** in deep trees. Fix: cache with invalidation on change, or `computed()` signals.
- **Children list exposed and mutated externally.** Fix: expose a read-only view, mutate via `add()/remove()`.
- **Parent references kept in sync manually** go stale. Fix: set parent inside `add()`/`remove()` only, or avoid parent pointers.
- **Mutating nested form/state trees in place** under OnPush/signals: no change detected. Fix: immutable updates.

## Refactoring path
- **Introduce it:** find repeated `isinstance` branches over a recursive structure; extract a common interface; move each branch's logic into leaf/composite methods; delete the switches.
- **Remove it / simplify:** if the tree is always depth 1, replace with a list of leaves; if operations multiply, keep the data tree and move operations into functions or a Visitor.

## Related patterns
- [Decorator](../structural/decorator.md) — wraps one component; Composite aggregates many.
- [Visitor](../behavioral/visitor.md) — adds operations over a composite tree without changing node classes.
- [Iterator](../behavioral/iterator.md) — traverses the tree (depth-first/breadth-first generators).
- [Chain of Responsibility](../behavioral/chain-of-responsibility.md) — requests often bubble up the parent chain of a composite (DOM events).
- [Interpreter](../behavioral/interpreter.md) — an AST is a Composite with an `evaluate` operation.
- [Specification](../behavioral/specification.md) — composite rules (`and`, `or`, `not`).
- [Flyweight](../structural/flyweight.md) — share identical leaves in large trees.
