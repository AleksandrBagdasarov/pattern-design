# Visitor

**Category:** Behavioral · **Also known as:** — · **Related:** Double Dispatch, Composite, Interpreter, Iterator, Strategy

## Intent
Add new operations over a stable object structure (a tree of node types) without modifying the node classes, by putting each operation in its own visitor object that has one method per node type.

## Problem
- You have a fixed set of node types (an AST, a document model, a query tree, a pricing rule tree) and keep adding operations over them: render, validate, serialise, estimate cost, collect dependencies.
- Smell: every node class grows `to_html()`, `to_markdown()`, `validate()`, `word_count()`, so a node file mixes rendering, persistence and analytics concerns.
- Smell: one big function with `isinstance` checks on each node type, duplicated per operation, and a new node type is silently missed in some of them.
- Scenario: a CMS stores pages as a tree of `Heading`, `Paragraph`, `Image`, `Table` blocks. Product wants HTML export, plain-text search indexing, accessibility checks (missing alt text) and a reading-time estimate.

## Solution and structure
- **Element** nodes expose `accept(visitor)`, which calls the visitor method for their own type (`visitor.visit_image(self)`). That is double dispatch.
- **Visitor** declares `visit_<node>` for every node type. Each concrete visitor is one operation.
- An **object structure** (often a Composite) is traversed either by the visitor or by the nodes.

```
          accept(v)                    visit_heading / visit_paragraph / visit_image
Node ─────────────────▶ Visitor ◀──── HtmlExporter, AltTextChecker, ReadingTime
```

The trade-off is the **expression problem**: Visitor makes adding operations cheap and adding node types expensive (every visitor must change). Plain methods on nodes are the opposite.

In Python and TypeScript you rarely need `accept()`. Use `match` on node classes, `functools.singledispatchmethod`, name-based lookup (`getattr(self, f"visit_{type(node).__name__}")`, as `ast.NodeVisitor` does), or a `switch` on a discriminated union.

## When to use
- Node types are stable; operations are many and grow.
- Operations need to accumulate state while walking (symbol tables, totals, error lists).
- Compilers, linters, code generators, query builders, document converters, schema transformers.
- You want each concern (render, validate, export) in its own module, not spread over node classes.

## When NOT to use
- Node types change often (plugins add new blocks). Every addition breaks every visitor; prefer methods on nodes or a registry of handlers with a default.
- One or two operations. A method on each node or a single `match` is simpler.
- A flat list, not a structure. A loop is enough.
- Do not add `accept()` boilerplate in Python/TS just to match the textbook.

## Example — Python
Name-based dispatch, like `ast.NodeVisitor`, with a fallback for unknown nodes.

```python
from dataclasses import dataclass, field
from html import escape

@dataclass(frozen=True)
class Heading:   text: str; level: int = 2
@dataclass(frozen=True)
class Paragraph: text: str
@dataclass(frozen=True)
class Image:     src: str; alt: str = ""
@dataclass(frozen=True)
class Section:   children: tuple["Block", ...]

type Block = Heading | Paragraph | Image | Section

class BlockVisitor[R]:
    def visit(self, node: Block) -> R:
        method = getattr(self, f"visit_{type(node).__name__.lower()}", self.generic_visit)
        return method(node)

    def generic_visit(self, node: Block) -> R:
        raise NotImplementedError(f"{type(self).__name__} cannot handle {type(node).__name__}")

class HtmlExporter(BlockVisitor[str]):
    def visit_heading(self, n: Heading) -> str:     return f"<h{n.level}>{escape(n.text)}</h{n.level}>"
    def visit_paragraph(self, n: Paragraph) -> str: return f"<p>{escape(n.text)}</p>"
    def visit_image(self, n: Image) -> str:         return f'<img src="{escape(n.src)}" alt="{escape(n.alt)}">'
    def visit_section(self, n: Section) -> str:     return "<section>" + "".join(map(self.visit, n.children)) + "</section>"

@dataclass
class AltTextChecker(BlockVisitor[None]):
    problems: list[str] = field(default_factory=list)
    def visit_image(self, n: Image) -> None:
        if not n.alt.strip():
            self.problems.append(f"missing alt on {n.src}")
    def visit_section(self, n: Section) -> None:
        for child in n.children:
            self.visit(child)
    def generic_visit(self, node: Block) -> None:   # headings/paragraphs: nothing to check
        return None

page = Section((Heading("Pricing"), Image("/plans.png"), Paragraph("From 9 EUR")))
print(HtmlExporter().visit(page))
checker = AltTextChecker(); checker.visit(page); print(checker.problems)
```

Name-based lookup is not type-checked; for exhaustiveness use `match node:` with `assert_never` in the final case.

## Example — TypeScript
A discriminated union plus a typed visitor object; the mapped type makes missing handlers a compile error.

```typescript
type Block =
  | { kind: 'heading'; text: string; level: number }
  | { kind: 'paragraph'; text: string }
  | { kind: 'image'; src: string; alt: string }
  | { kind: 'section'; children: Block[] };

type Visitor<R> = { [K in Block['kind']]: (node: Extract<Block, { kind: K }>, visit: (b: Block) => R) => R };

export function walk<R>(node: Block, v: Visitor<R>): R {
  const visit = (b: Block): R => walk(b, v);
  return (v[node.kind] as (n: Block, visit: (b: Block) => R) => R)(node, visit);
}

const WORDS_PER_MINUTE = 200;
export const wordCount: Visitor<number> = {
  heading: n => n.text.split(/\s+/).length,
  paragraph: n => n.text.split(/\s+/).length,
  image: () => 0,
  section: (n, visit) => n.children.reduce((sum, c) => sum + visit(c), 0),
};

export const readingMinutes = (page: Block): number =>
  Math.max(1, Math.ceil(walk(page, wordCount) / WORDS_PER_MINUTE));
```

## In the wild
- Python `ast.NodeVisitor` / `ast.NodeTransformer` (name-based `visit_<Class>`), `libcst` visitors.
- SQLAlchemy SQL compiler (`visit_select`, `visit_binary` per dialect).
- TypeScript compiler API `ts.forEachChild` / `visitEachChild`, Babel and ESLint visitors keyed by node type.
- Angular compiler template AST visitors (`TmplAstVisitor`); internal API, may change between versions.
- `docutils` node visitors (Sphinx writers), Jinja2 `NodeVisitor`, GraphQL-js `visit()`.

## Trade-offs
| Pros | Cons |
|---|---|
| New operation = new visitor, nodes untouched | New node type = change every visitor |
| Each concern lives in one place | Visitors often need node internals, weakening encapsulation |
| Visitors can accumulate state across the walk | Classic `accept` boilerplate is noisy in dynamic languages |
| Typed visitors give exhaustiveness checks | Traversal logic can be duplicated across visitors |

## Pitfalls
- **Forgetting to recurse** into children in a container node, so nested nodes are skipped. Fix: a base `generic_visit` that walks children, as `ast.NodeVisitor` does.
- **Silent no-op for new node types.** Fix: raise in the default, or use typed exhaustive visitors.
- **Mutating the tree while visiting.** Fix: return new nodes (transformer style) and keep nodes immutable.
- **Deep recursion** on huge trees hits Python's recursion limit. Fix: iterative traversal with an explicit stack.
- **Using Visitor when node types are open-ended.** Fix: methods on nodes or a handler registry with a default.

## Refactoring path
- **Introduce it:** from node classes accumulating unrelated `to_x()` methods, or from duplicated `isinstance` walkers. (1) Pick one operation and write a visitor that reproduces it, with tests comparing outputs. (2) Switch callers to the visitor and delete the node methods. (3) Repeat per operation. (4) Extract shared traversal into a base class.
- **Remove it / simplify:** if there is only one visitor, inline it as a recursive function with `match`. If node types keep growing, move behaviour back onto nodes.

## Related patterns
- [Double Dispatch](double-dispatch.md) — the mechanism Visitor is built on.
- [Composite](../structural/composite.md) — the tree structure visitors usually walk.
- [Interpreter](interpreter.md) — evaluation is just one more visitor over the AST.
- [Iterator](iterator.md) — traverses without type-specific behaviour; combine to separate "walk" from "do".
- [Strategy](strategy.md) — a visitor is a strategy with one method per node type.
