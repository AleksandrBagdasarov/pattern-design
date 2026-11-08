# Blackboard

**Category:** Architectural · **Also known as:** shared-repository architecture, tuple space (related) · **Related:** Microkernel, Mediator, Observer, Pipes and Filters, Chain of Responsibility, Event-Driven

## Intent
Solve a problem with no deterministic algorithm by letting several independent **knowledge sources** (specialists) read a shared **blackboard** of partial solutions and contribute to it incrementally, while a **controller** decides which specialist acts next, until an acceptable solution emerges.

## Problem
- The solution path is not known in advance; it depends on what partial results appear.
- Several heterogeneous techniques each solve part of the problem (rules, ML models, lookups), and their order varies per input.
- A fixed pipeline either runs useless steps or misses opportunities when a later result could improve an earlier one.
- Scenario: a fraud/KYC review system. A document parser extracts fields, an OCR model reads IDs, a sanctions checker needs a name, an address verifier needs a postcode, a risk scorer needs all of them. Which runs next depends on what is already known and how confident it is.

## Solution and structure
```
           ┌───────────── Blackboard (shared state: hypotheses + confidence) ─────────────┐
           │  name=?  dob=1990-01-02 (0.9)  address=?  sanctions=?  risk=?               │
           └───────▲────────────▲──────────────▲───────────────▲───────────────────────────┘
         reads/    │            │              │               │
         writes  OCR KS    Parser KS     Sanctions KS     Risk KS        (knowledge sources)
                   ▲
     Controller: picks the next KS whose preconditions hold, by priority, until goal or no progress
```
- **Blackboard:** structured shared data (facts, hypotheses with confidence, provenance).
- **Knowledge source (KS):** independent module with a *precondition* ("can I contribute now?") and an *action* (write new facts). KSs never call each other.
- **Controller:** loop that selects and runs an eligible KS (by priority, cost, expected gain), stops at the goal or when no KS can contribute.

## When to use
- Problems solved by combining heuristics or specialists in data-dependent order: speech/image recognition, planning, diagnostics, document understanding, multi-agent LLM workflows.
- Adding a new specialist must not require rewiring others.
- Partial, uncertain results must be refined over several passes.

## When NOT to use
- The order of steps is fixed: use Pipes and Filters or a plain function sequence.
- Deterministic workflows with clear business rules: use a workflow/state machine.
- High-throughput transactional systems: shared mutable state becomes a contention and consistency problem.
- Smell: a blackboard used as a global dict that any code mutates, without a controller or KS contracts.

## Example — Python
```python
from dataclasses import dataclass, field
from typing import Protocol

@dataclass
class Board:
    facts: dict[str, object] = field(default_factory=dict)
    log: list[str] = field(default_factory=list)
    def add(self, key: str, value: object, by: str) -> None:
        self.facts[key] = value
        self.log.append(f"{by} set {key}")

class KnowledgeSource(Protocol):
    name: str
    def can_contribute(self, b: Board) -> bool: ...
    def contribute(self, b: Board) -> None: ...

class Ocr:
    name = "ocr"
    def can_contribute(self, b: Board) -> bool: return "image" in b.facts and "full_name" not in b.facts
    def contribute(self, b: Board) -> None: b.add("full_name", "JOHN DOE", self.name)   # model call here

class Sanctions:
    name = "sanctions"
    def can_contribute(self, b: Board) -> bool: return "full_name" in b.facts and "sanctions_hit" not in b.facts
    def contribute(self, b: Board) -> None: b.add("sanctions_hit", False, self.name)

class Risk:
    name = "risk"
    def can_contribute(self, b: Board) -> bool: return "sanctions_hit" in b.facts and "risk" not in b.facts
    def contribute(self, b: Board) -> None:
        b.add("risk", "high" if b.facts["sanctions_hit"] else "low", self.name)

def solve(board: Board, sources: list[KnowledgeSource], goal: str, max_steps: int = 50) -> Board:
    for _ in range(max_steps):                      # controller
        if goal in board.facts:
            return board
        ready = [ks for ks in sources if ks.can_contribute(board)]
        if not ready:
            raise RuntimeError(f"stuck without {goal}: {board.log}")
        ready[0].contribute(board)                  # simplest strategy: first eligible by priority
    raise RuntimeError("step budget exhausted")

result = solve(Board({"image": b"..."}), [Risk(), Sanctions(), Ocr()], goal="risk")
```
The source list order does not matter: the controller derives the order from preconditions.

## Example — TypeScript
```ts
// Async knowledge sources (e.g. LLM agents or HTTP services) contributing to a shared board.
type Facts = Partial<{ text: string; language: string; summary: string; tags: string[] }>;

interface KnowledgeSource {
  name: string;
  priority: number;
  canContribute(f: Facts): boolean;
  contribute(f: Readonly<Facts>): Promise<Facts>;      // returns a patch, never mutates
}

export async function runBlackboard(initial: Facts, sources: KnowledgeSource[], isDone: (f: Facts) => boolean) {
  let facts: Facts = { ...initial };
  const trace: string[] = [];
  for (let step = 0; step < 20 && !isDone(facts); step++) {
    const ready = sources.filter(s => s.canContribute(facts)).sort((a, b) => b.priority - a.priority);
    if (ready.length === 0) throw new Error(`No progress. Trace: ${trace.join(' → ')}`);
    const patch = await ready[0].contribute(facts);
    facts = { ...facts, ...patch };                     // controller owns all writes
    trace.push(ready[0].name);
  }
  return { facts, trace };
}

// const detectLang: KnowledgeSource = { name: 'lang', priority: 2,
//   canContribute: f => !!f.text && !f.language, contribute: async f => ({ language: 'en' }) };
```

## In the wild
- Hearsay-II speech recognition (origin), many expert systems, robotics sensor fusion.
- Multi-agent LLM frameworks with a shared scratchpad/state (e.g. LangGraph's shared graph state, multi-agent "shared memory" designs).
- Tuple spaces (Linda, JavaSpaces) and some data-lake "enrichment" designs follow the same idea.

## Trade-offs
| Pros | Cons |
|---|---|
| Specialists added/removed independently | Control strategy is hard to design and to test |
| Handles non-deterministic, data-dependent solution paths | Non-deterministic order makes debugging hard |
| Supports partial, uncertain, incremental solutions | Shared state needs concurrency control if KSs run in parallel |
| Good fit for heterogeneous techniques (rules + ML) | Termination and performance are not guaranteed |

## Pitfalls
- **No termination guard:** KSs that keep rewriting facts loop forever. Set step budgets and "no progress" detection.
- **KSs calling each other:** breaks independence. All communication goes through the board.
- **Unstructured board:** a bag of strings. Define a schema with provenance and confidence per fact.
- **Concurrent writes:** parallel KSs overwrite each other. Have KSs return patches; let the controller apply them atomically.
- **No trace:** record which KS wrote what, or results are unexplainable (essential in compliance and AI systems).

## Refactoring path
- **Introduce it:** when a pipeline accumulates `if already_have_x: skip` branches and re-run loops, turn each step into a KS with an explicit precondition, move shared data into a board object, and replace the fixed order with a controller loop.
- **Remove it / simplify:** if logs show the controller always picks the same order, replace it with a fixed pipeline of the same functions.

## Related patterns
- [Pipes and Filters](../architectural/pipes-and-filters.md) — fixed order; the blackboard's order is dynamic.
- [Mediator](../behavioral/mediator.md) — the controller mediates; KSs never talk directly.
- [Observer](../behavioral/observer.md) — KSs can be triggered by board changes instead of polling.
- [Chain of Responsibility](../behavioral/chain-of-responsibility.md) — first handler that can act, but one pass only.
- [Microkernel](../architectural/microkernel.md) — knowledge sources are often loaded as plug-ins.
- [Event-Driven](../architectural/event-driven.md) — a distributed blackboard can be built on change events.
