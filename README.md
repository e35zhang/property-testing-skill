---
name: property-first-testing
description: Design tests (and the architecture they protect) by formal decomposition instead of by feature. Use when asked to write tests, design a test strategy, model a component's correctness, decide what to test, or review a test suite. Turns "write tests for X" into a workflow — decompose X into orthogonal properties, write one basis test per property proving it in isolation, then write TLA+-style span tests or property testing libraries proving every reachable composition preserves the invariants. Counters the default failure mode where an LLM emits happy-path, data-flow, feature-shaped tests that assert outputs instead of properties.

---

# Property-First Testing

Design tests the way you design architecture: by **decomposing** a problem into
orthogonal properties, **composing** those properties into behavior, and then
**testing both tiers** — one test per basis property (proving the vector), and
TLA+-style tests over the reachable state space (proving the span).

This skill exists to defeat one specific failure mode. Asked to "write tests for
X," a language model defaults to *feature-shaped, data-flow, happy-path* tests:
call the function, assert the return value on a couple of hand-picked inputs.
Those tests encode **examples**, not **properties**. They pass while the system
is deeply broken, and they lock in whatever behavior the code happened to have
on the day they were written. This skill replaces that reflex with a disciplined
thought-flow.

> The core claim: **software engineering is a decomposition-and-composition
> problem.** Correctness is not a list of examples; it is a set of properties
> that hold over a state space. Tests are the proof, and a proof has structure.

---

## Mental Model: Architecture Is a Vector Space

Every sufficiently complex domain decomposes into a small number of **independent
axes of variation** — the basis vectors of the domain. Each axis carries an
**intrinsic property** that the others cannot synthesize. This orthogonality is
not a design choice; it is a fact about the domain that you *discover*.

The math analogy makes it concrete. Consider three functions as "basis vectors":

| Basis vector | Intrinsic property | Can the others synthesize it? |
|--------------|--------------------|-------------------------------|
| `sin(x)`     | periodic (bounded, oscillating) | No |
| `x`          | monotone (unbounded, linear growth) | No |
| `e^x`        | convex growth (accelerating) | No |

No linear combination `a·x + b·e^x` — for *any* parameters — is periodic. The
property "periodic" lives on a basis vector that `x` and `e^x` simply do not
span. That is what orthogonality *means*: a property you cannot compose out of
the other components.

Software works the same way. A subsystem's correctness properties
("labels are never reused while live," "count is conserved across churn,"
"selection is independent of arrival order") live on specific axes. If you test
by feature, you sample points in the space and hope. If you test by property,
you prove the basis vectors and then prove their span.

**Two tiers of test follow directly from this model:**

| Tier | Proves | Analogy | Scope |
|------|--------|---------|-------|
| **Basis test** | Each orthogonal component upholds its intrinsic property *in isolation* | "`e^x` is monotone-increasing" | One axis, one invariant |
| **Span test (TLA+-style)** | Every reachable composition of the axes preserves the composed invariants | "no combination of `x`, `e^x` is periodic" | Whole reachable state space |

A suite that has only the first tier proves the parts but not the whole. A suite
that has only the second proves the whole but localizes nothing when it breaks.
You want both. They are not redundant — they cover orthogonal risks.

---

## When to Use This Skill

Use it whenever the task is any of:

- "Write tests for this module / function / subsystem."
- "Design a test strategy" or "improve test coverage."
- "Model the correctness of X" / "what are the invariants of X?"
- "Review these tests" — audit whether they encode properties or examples.
- Deciding *what* to test when the surface is large and examples feel arbitrary.

If you catch yourself about to write `assert f(2, 3) == 5`, stop and run the
workflow. That assertion is a sampled point; find the property it is a sample of.

---

## The Workflow

```
   DECOMPOSE            COMPOSE              TEST THE BASIS         TEST THE SPAN
  (identify axes)  →  (state the algebra) →  (prove each vector) → (prove the span)
      Phase A            Phase B                  Phase C               Phase D
```

### Phase A — Decompose (System Identification)

Find the basis. This is the hard 10% and it is a *human/LLM-reasoning* step, not
a coding step.

1. **Name the boundary.** State the subsystem as a map: what are its inputs, its
   outputs, and the two worlds it bridges. Write one sentence:
   *"X turns ⟨inputs⟩ into ⟨outputs⟩, maintaining ⟨state⟩."*
2. **Enumerate behaviors.** List everything the subsystem does. Don't organize
   yet — just enumerate. Each behavior is a data point.
3. **Find the axes of variation.** For each pair of behaviors ask: *"If I changed
   A, would B have to change?"* If no, A and B lie on different axes. An axis is
   an independent direction of change.
   - A change along axis 1 does not require a change along axis 2.
   - A property on axis 1 can be *stated and verified* without reference to axis 2.
4. **Reduce to the “eigen-basis”.** Find the *minimal, spanning, independent* set of
   properties that generate all behaviors:
   - **Spanning** — every behavior is a composition of basis properties.
   - **Independent** — no basis property is derivable from the others.
   - **Minimal** — remove any one and some behavior can no longer be expressed.
5. **Write the property for each axis.** For every basis vector, state the
   *invariant or law* that characterizes it in isolation. This sentence becomes a
   basis test in Phase C. Use the taxonomy below to name it precisely.

Output of Phase A: a short list of axes, each with a one-line formal property.

### Phase B — Compose (State the Algebra)

Express the system as composition, and write down the state machine you will
later check.

1. **Feature = composition.** For each externally visible behavior, write it as
   `behavior = compose(prop_axis_i, prop_axis_j, ...)`. Shared properties across
   many behaviors are shared components (this is also your architecture: orthogonal
   axes → modules with no cross-imports, layered as a DAG).
2. **Define the state machine** you will model-check in Phase D:
   - **VARIABLES** — the minimal state the invariants talk about.
   - **Init** — the clean starting state (every test behavior begins here).
   - **Next** — a *disjunction of actions*: the finite alphabet of operations
     that drive transitions (`create`, `remove`, `reconfigure`, `crash`, …).
   - **Invariants** — the properties from Phase A, now as predicates over
     VARIABLES that must hold in *every reachable state*.

Output of Phase B: the action alphabet + the invariant predicates.

### Phase C — Test the Basis (Prove Each Vector)

For each axis, write a test that proves its intrinsic property **in isolation**.

- **Exercise the real component, mock only across the boundary.** Replace only
  what lives on the *other side* of the identified boundary (persistence,
  downstream subsystem, clock). Run the real production code path. Shrinking a
  resource (a 128K table → 256 entries) to make the test fast is fine *as long as
  the same production code runs*.
- **Assert the property, not an output.** Walk the whole structure and check the
  invariant on every element, not just the return value. (e.g. "every node on the
  USED list also carries the IN-TREE flag" — checked across all nodes.)
- **Choose white-box or black-box per property.** Structural invariants
  (list/tree integrity, conservation) need white-box state inspection; behavioral
  invariants (output selection) can be black-box on the emitted messages. Pick
  what the property requires.
- **One property per test.** If a test asserts two unrelated properties, it is
  sampling two axes at once and will be hard to localize.

Output of Phase C: one focused test per basis property.

### Phase D — Test the Span (TLA+-Style)

Prove the composition. This is bounded model checking, done in whatever language or property testing libraries that fits the most. The fundamentals contains:

1. **Enumerate the reachable state space.** Generate *all interleavings* of the
   action alphabet up to a bounded length (e.g. all sequences of length 1..3, plus
   removals, reconfigurations, and re-additions). This is TLC's job, hand-rolled:
   explore reachable states by applying every action in every order.
2. **Assert the composed invariant in every reachable state** — after *each*
   action, not only at the end.
3. **Inject faults as first-class actions.** `crash`/`restart` is just another
   action in the alphabet. After it, assert the recovery invariant (state restored,
   nothing lost, nothing duplicated).
4. **Check the span is not too big.** Assert that *illegal* states are
   unreachable — the "no periodicity from monotone bases" check. If an action
   sequence reaches a state the invariant forbids, either the code or the model is
   wrong.

Output of Phase D: a permutation harness + invariant checks (+ optional baseline).

---

## Property Taxonomy

Naming the property precisely is most of the battle — the right word tells you
what test to write. Reach for these before writing any assertion:

| Property class | Informal meaning | Typical assertion shape |
|----------------|------------------|-------------------------|
| **Invariant (safety)** | "Nothing bad ever holds in any reachable state" | after every action, `P(state)` is true |
| **Conservation** | A quantity is invariant under churn | `count_before == count_after`; total is constant across alloc/free |
| **Flag / structural consistency** | Membership sets agree | on USED list ⟺ carries USED flag ∧ present in index |
| **Ordering / temporal (liveness)** | "Something good eventually, in order" | freed `a,b,c` ⇒ reclaimed `a,b,c` (FIFO) |
| **Commutativity / order-independence** | Result doesn't depend on arrival order | `apply(perm(actions))` yields the same winner for all perms |
| **Idempotence / round-trip** | `op` then `op⁻¹` is identity | `create→delete→create` reproduces the exact prior state |
| **Recovery / durability** | Survives fault injection | after `crash+restart`, state restored, UUIDs preserved |
| **Monotonicity** | A measure only moves one way | version/sequence never decreases |
| **Referential integrity** | No dangling references | every reference resolves to a live object or the null sentinel |
| **Boundedness** | Resource cannot exceed a limit | reachable states never exceed capacity |

If you can't name the property, you have not finished Phase A. Go back.

---

## Templates

### Basis test (Phase C) — prove one vector in isolation

```
test_<axis>_<property>():
    # ARRANGE: real component, mock ONLY across the boundary
    init_real_component(size=SMALL)          # shrink resource, same prod code
    mock_the_thing_on_the_other_side()       # persistence / downstream / clock

    # ACT: drive the single axis
    ... apply operations that exercise ONLY this property ...

    # ASSERT: the intrinsic invariant, over the WHOLE structure
    for element in walk(structure):
        assert invariant_holds(element)      # not just the return value
    assert conserved_quantity == expected    # e.g. used + free == TOTAL
```

### Span harness (Phase D) — prove the composition over reachable states

```
ALPHABET = [create_a, create_b, remove_a, remove_b, reconfigure, crash]

for seq in bounded_interleavings(ALPHABET, max_len=3):   # + removals/flaps
    state = Init()
    for action in seq:
        state = apply(action, state)
        assert composed_invariant(state)     # checked after EVERY action
```

### TLA+ sketch (Phase B) — the model the harness approximates

```
VARIABLES s                          \* the minimal state the invariants talk about
Init  == s = <clean initial state>
Next  == \/ CreateA(s) \/ CreateB(s)
         \/ RemoveA(s) \/ RemoveB(s)
         \/ Reconfigure(s) \/ Crash(s)      \* the action alphabet
Inv   == /\ Conserved(s)             \* count invariant
         /\ FlagsConsistent(s)       \* structural invariant
         /\ NoDangling(s)            \* referential integrity
\* Spec == Init /\ [][Next]_s ; check Inv is an invariant of Spec
```

You do not have to run TLC to benefit. Writing the four parts —
VARIABLES / Init / Next / Inv — *is* the design act. The permutation harness in
Phase D is the executable approximation of `Init /\ [][Next]_s ⇒ []Inv`.

---

## Worked Example (Abstract — a Bounded Reusable-ID Pool)

Deliberately domain-free: a pool that hands out integer IDs, lets them be freed
and reused, and must survive a crash. It shows the whole flow without any
business or protocol specifics.

**Phase A — Decompose.** Boundary: *"pool turns alloc/free requests into stable
ID assignments, maintaining a used-set, a free-list, and a durable log."* Axes:

| Axis | Intrinsic property (name from taxonomy) |
|------|------------------------------------------|
| Allocation | **Conservation** — `used + free == CAPACITY` always |
| Membership | **Structural consistency** — id in used-set ⟺ used-flag ∧ indexed |
| Reuse | **Ordering** — freed ids are reclaimed FIFO |
| Identity | **Idempotence** — re-allocating a still-known key returns the same id |
| Durability | **Recovery** — after crash+restart, every id survives, none duplicated |

**Phase B — Compose.** Alphabet: `alloc(key)`, `free(id)`, `crash`. Invariant:
`Conservation ∧ StructuralConsistency ∧ NoDangling`.

**Phase C — Basis tests.** Five tests, one per axis. `test_reuse_ordering` frees
`[b, a, c]` and asserts the next three allocs return `b, a, c`. It runs the real
alloc/free; it mocks only the durable log (the thing across the boundary) and
shrinks CAPACITY to 256 so it finishes in milliseconds.

**Phase D — Span test.** Enumerate every interleaving of `alloc`/`free`/`crash`
up to length 3 over a small key set; after each action assert
`used + free == CAPACITY` and no id appears in both sets; golden-master the final
assignment map for each sequence. If a future refactor makes
`alloc→free→crash→alloc` return a duplicate id, the baseline diff catches it and
the conservation assertion localizes it.

Notice what is *absent*: no `assert alloc() == 0` example test. Every assertion is
a property that holds for a *class* of inputs, checked across the reachable space.

---

## Anti-Patterns (the default LLM reflex to suppress)

| Anti-pattern | Why it fails | Fix |
|--------------|--------------|-----|
| **Example test** — `assert f(2,3)==5` | Samples one point; says nothing about the space | Name the property `f` is sampling; test that |
| **Feature-shaped test** — one test per user story | Bundles many axes; breaks localize nothing | Decompose into per-axis basis tests |
| **Happy-path only** | The reachable space includes churn, reorder, faults | Enumerate interleavings; inject `crash` |
| **Assert the output, not the invariant** | Output can be right while state is corrupt | Walk the structure; assert conservation/consistency |
| **Golden-master with no model** | Locks in *current* behavior, including bugs | Baseline only *after* stating the invariant it must satisfy; review diffs |
| **Mock the component under test** | Proves the mock, not the code | Mock only *across the boundary*; run real production paths |
| **Test order-dependence by accident** | Hidden coupling passes silently | Add an explicit commutativity test over permutations |

---

## Validation Checklist

Before calling a suite done, verify:

- [ ] **Phase A done:** every axis has a *named* property from the taxonomy — no
      axis is described only by an example.
- [ ] **Basis coverage:** exactly one focused basis test per axis; each asserts a
      property over the whole structure, not a single return value.
- [ ] **Boundary discipline:** mocks exist *only* across the identified boundary;
      the real production code path runs in every basis test.
- [ ] **Alphabet defined:** the finite action set (including fault actions) is
      written down explicitly.
- [ ] **Span coverage:** a harness enumerates interleavings of the alphabet and
      asserts the composed invariant after *every* action, not just at the end.
- [ ] **Fault injection:** at least one `crash`/`restart`-class action with a
      recovery invariant.
- [ ] **Illegality checked:** the suite would *fail* if an action sequence reached
      a forbidden state (the "no periodicity from monotone bases" guard).
- [ ] **Baseline reviewed:** if golden-mastering, an unintended baseline diff is
      treated as a bug, and re-baselining is deliberate and reviewed.
- [ ] **No example tests remain** that could be replaced by a property.

---

## Division of Labor (Human vs LLM)

Mirror the architecture-refactoring split: the human does **system
identification** (Phase A — discover the axes, name the properties); the LLM does
**mechanical execution** (Phases C–D — generate the per-axis basis tests, write
the permutation harness,etc.). The properties named in Phase A
are the specification the LLM executes against — and, not incidentally, naming
them in the prompt activates the model's formal-methods reasoning, steering it
away from the example-test reflex. When the human skips Phase A, the LLM has
nothing to compose and falls back to sampling points. **The human provides the
basis; the LLM proves the span.**

