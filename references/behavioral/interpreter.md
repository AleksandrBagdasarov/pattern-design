# Interpreter

**Category:** Behavioral · **Also known as:** Little language, Expression tree, Embedded DSL · **Related:** Composite, Visitor, Specification, Interpreter Context, Strategy, Double Dispatch

## Intent
Represent the grammar of a small language as a tree of expression objects and evaluate sentences of that language by walking the tree.

## Problem
- Business users or config files need to express rules that change more often than you deploy: discount eligibility, alert conditions, feature-flag targeting, search filters.
- Smell: rules are hard-coded as nested `if`s and every change is a code release.
- Smell: rules are stored as strings and run with `eval()`, which is a security hole and impossible to validate.
- Scenario: marketing wants promo conditions like `country in ["PL","UA"] and cart_total >= 50 and not customer.is_employee`, editable in an admin UI, validated before saving, and explainable ("why didn't this promo apply?").

## Solution and structure
- **Abstract expression**: a node type with `evaluate(context)`.
- **Terminal expressions**: literals and variables (`Const(50)`, `Var("cart_total")`).
- **Non-terminal expressions**: operators that combine children (`And`, `Or`, `Not`, `Gte`, `In`).
- **Context**: the input data (and possibly functions) the expression is evaluated against.
- **Parser** (often separate and not part of the GoF pattern): turns text or JSON into the tree.

```
            And
          /     \
       In        Gte
      /  \      /    \
 Var(country) Const  Var(cart_total) Const(50)
```

The tree is a Composite; evaluation is a recursive walk (or a Visitor). For anything beyond a toy grammar, use a parser library rather than hand-rolling a tokenizer, and keep the AST as plain data so it can be stored as JSON.

## When to use
- A small, stable grammar (boolean rules, filters, simple arithmetic, templating, routing patterns).
- Rules must be stored as data, validated, versioned, and edited without a deploy.
- You need to explain, translate or optimise the rule as well as evaluate it (same tree → SQL, → human-readable text).

## When NOT to use
- The "language" is really just configuration. A typed config object (Pydantic model) is enough.
- A full programming language is required. Embed a sandboxed existing one (CEL, JSONLogic, Lua, a SQL subset) instead of growing your own.
- Performance-critical hot loops over a large grammar. Tree-walking is slow; compile to closures or bytecode, or use a real engine.
- Never use `eval`/`exec`/`new Function` on user input as a shortcut.

## Example — Python
Expression nodes as frozen dataclasses, a JSON parser, and evaluation via `match`.

```python
from dataclasses import dataclass
from typing import Any

@dataclass(frozen=True)
class Var:   name: str
@dataclass(frozen=True)
class Const: value: Any
@dataclass(frozen=True)
class Cmp:   op: str; left: "Expr"; right: "Expr"
@dataclass(frozen=True)
class Bool:  op: str; args: tuple["Expr", ...]
@dataclass(frozen=True)
class Not:   arg: "Expr"

type Expr = Var | Const | Cmp | Bool | Not
CMP = {">=": lambda a, b: a >= b, "==": lambda a, b: a == b, "in": lambda a, b: a in b}

def parse(node: dict[str, Any] | str | int | float | list) -> Expr:
    match node:
        case {"var": str(name)}:                    return Var(name)
        case {"not": inner}:                        return Not(parse(inner))
        case {"and": list(xs)}:                     return Bool("and", tuple(map(parse, xs)))
        case {"or": list(xs)}:                      return Bool("or", tuple(map(parse, xs)))
        case {"op": str(op), "left": l, "right": r} if op in CMP:
            return Cmp(op, parse(l), parse(r))
        case dict():                                raise ValueError(f"unknown node {node}")
        case _:                                     return Const(node)

def evaluate(e: Expr, ctx: dict[str, Any]) -> Any:
    match e:
        case Const(v):         return v
        case Var(n):           return ctx[n]                  # KeyError = unknown variable
        case Not(a):           return not evaluate(a, ctx)
        case Cmp(op, l, r):    return CMP[op](evaluate(l, ctx), evaluate(r, ctx))
        case Bool("and", xs):  return all(evaluate(x, ctx) for x in xs)
        case Bool("or", xs):   return any(evaluate(x, ctx) for x in xs)
    raise ValueError(f"cannot evaluate {e!r}")

rule = parse({"and": [
    {"op": "in", "left": {"var": "country"}, "right": ["PL", "UA"]},
    {"op": ">=", "left": {"var": "cart_total"}, "right": 50},
    {"not": {"var": "is_employee"}},
]})
assert evaluate(rule, {"country": "PL", "cart_total": 60, "is_employee": False})
```

Parse and validate when the rule is saved (unknown variables, wrong types), not when it is evaluated in checkout.

## Example — TypeScript
The same AST as a discriminated union; it serialises to JSON for an Angular rule-builder UI.

```typescript
export type Expr =
  | { t: 'var'; name: string }
  | { t: 'const'; value: string | number | boolean | Array<string | number> }
  | { t: 'cmp'; op: '>=' | '==' | 'in'; left: Expr; right: Expr }
  | { t: 'and' | 'or'; args: Expr[] }
  | { t: 'not'; arg: Expr };

type Ctx = Readonly<Record<string, unknown>>;

export function evaluate(e: Expr, ctx: Ctx): unknown {
  switch (e.t) {
    case 'const': return e.value;
    case 'var':
      if (!(e.name in ctx)) throw new Error(`unknown variable ${e.name}`);
      return ctx[e.name];
    case 'not': return !evaluate(e.arg, ctx);
    case 'and': return e.args.every(a => evaluate(a, ctx));
    case 'or': return e.args.some(a => evaluate(a, ctx));
    case 'cmp': {
      const l = evaluate(e.left, ctx), r = evaluate(e.right, ctx);
      if (e.op === 'in') return Array.isArray(r) && r.includes(l as never);
      if (e.op === '==') return l === r;
      return typeof l === 'number' && typeof r === 'number' && l >= r;
    }
  }
}

/** Second interpretation of the same tree: human-readable explanation. */
export function explain(e: Expr): string {
  switch (e.t) {
    case 'const': return JSON.stringify(e.value);
    case 'var': return e.name;
    case 'not': return `not (${explain(e.arg)})`;
    case 'and': case 'or': return e.args.map(explain).join(` ${e.t} `);
    case 'cmp': return `${explain(e.left)} ${e.op} ${explain(e.right)}`;
  }
}
```

## In the wild
- Python `ast` + `compile`, `re` (regex is an interpreted language), `string.Template`, Jinja2, `fnmatch`.
- SQLAlchemy expression language (`and_(User.age >= 18, ...)` builds a tree compiled to SQL), Django `Q` objects.
- JSONLogic, Google CEL (used by Kubernetes validation and Firebase rules), AWS IAM policy conditions, EventBridge event patterns, OPA/Rego.
- Angular template expressions and the Angular router path matcher; MongoDB query documents.
- Spreadsheet formula engines, Elasticsearch query DSL.

## Trade-offs
| Pros | Cons |
|---|---|
| Rules become data: stored, versioned, edited without deploys | You now own a language: docs, errors, tooling, versioning |
| One AST supports many interpretations (evaluate, explain, to-SQL) | Tree-walking is slow for large or hot rules |
| Safe alternative to `eval` | Grammars grow; a "small" DSL creeps toward a bad general language |
| Easy to add a new operator node | Debugging user-written rules needs good error messages |

## Pitfalls
- **Using `eval` on rule strings.** Fix: parse into an AST with a whitelist of nodes and variables.
- **Validating late.** Unknown variables or type errors show up at checkout. Fix: validate the tree on save (type-check variables against a schema).
- **Unbounded evaluation.** Deeply nested or huge rules from users can exhaust the stack or CPU. Fix: limits on depth, size and evaluation time.
- **Grammar creep** (loops, functions, variables). Fix: switch to an existing engine (CEL, JSONLogic) before adding control flow.
- **Hidden truthiness differences** between languages (`0`, `""`, `[]`). Fix: define the semantics explicitly and share tests across backend and frontend evaluators.

## Refactoring path
- **Introduce it:** from hard-coded, frequently changing rule `if`s. (1) Write the current rules as AST data and a tiny evaluator, with tests asserting it matches the old code on real inputs. (2) Store rules as JSON and load them. (3) Add validation on save and an `explain` interpretation. (4) Delete the hard-coded branches.
- **Remove it / simplify:** if the rules have not changed in a year and only developers edit them, replace the DSL with plain code or a Specification object built in Python/TS. If the DSL keeps growing, migrate to CEL/JSONLogic.

## Related patterns
- [Composite](../structural/composite.md) — the expression tree is a Composite.
- [Visitor](visitor.md) — add interpretations (evaluate, explain, to-SQL) without touching node types.
- [Specification](specification.md) — composable rules built in code; Interpreter adds a stored, parseable language.
- [Interpreter Context](../emergent/interpreter-context.md) — how variables and environment are passed through evaluation.
- [Double Dispatch](double-dispatch.md) — the mechanism behind per-node evaluation.
- [Strategy](strategy.md) — when the "rule" is a small fixed set of code options, not a language.
