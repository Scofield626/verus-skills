______________________________________________________________________

## name: verus-proof-writing description: Write new Verus proofs from scratch. Use when the task is to design specs, invariants, helper lemmas, quantifiers, triggers, or ghost state for new verified code rather than repairing one localized failing proof.

# Verus Proof Writing

Use this skill when you are building a proof, not just patching one verifier error.

## Operating model

Start by locating the abstraction boundary the repo already uses.

- Prefer `spec/`, `proof/`, and `exec/` splits when they exist.
- In repos using `_t.rs` and `_v.rs`, keep trusted interfaces and verified implementation facts on the right side of that split.
- Do not prove exec behavior directly over raw mutable fields if the codebase already has a view function, protocol state machine, or abstract model.

## Choose `spec fn` vs `proof fn` vs `exec fn`

Use `spec fn` for the mathematical model.

- Recursive sequence or map properties
- Abstract state predicates
- View functions and representation invariants

Use `proof fn` for reusable logical steps.

- Monotonicity lemmas
- Induction over a recursive spec
- Extensionality, range, or algebraic helper lemmas

Use `exec fn` only for the concrete implementation.

- Keep the implementation tied to the abstract model through postconditions
- Pull in proof facts only where the exec step needs them

Preferred shape:

```rust
spec fn abs_model(...) -> ... { ... }
proof fn lemma_step(...) ensures ... { ... }
fn exec_impl(...) ensures result@ == abs_model(...) { ... }
```

## Build invariants in layers

### Loop invariants

Mirror the abstract meaning of the processed prefix or current counter state.

Always include:

- index bounds
- semantic meaning of the accumulator
- any arithmetic or length bounds needed in the loop body
- any postcondition fragment that must survive loop exit

Pattern:

```rust
while i < n
    invariant
        0 <= i <= n,
        acc == spec_prefix(xs@, i as int),
        forall|j: int| 0 <= j <= i ==> prefix_ok(xs@.take(j)),
    decreases n - i
{
    ...
}
```

### Datatype invariants

Expose a stable abstract view, not every implementation detail.

- Define `View` when proofs should talk about a `Map`, `Seq`, or abstract protocol state
- Use `#[verifier::type_invariant]` on exec-facing structures
- Reintroduce the invariant explicitly with `use_type_invariant` inside methods

### Protocol invariants

Split them into small named predicates.

Common layers:

- domain completeness
- ordering or range facts
- ownership agreement
- storage agreement
- phase-specific constraints for each enum variant

For complex transitions, re-prove quantified post-state facts one clause at a time:

```rust
assert forall |k|
    #[trigger] post.map.contains_key(k)
    implies post.inv_for(k)
by {
    match post.map[k] {
        ...
    }
}
```

## Standard lemma patterns

Use pointwise lemmas to establish universal facts:

```rust
assert forall |i| Cond(i) implies Prop(i) by {
    lemma_for_one(i);
}
```

Use explicit decreases metrics for relational induction:

```rust
proof fn lemma(i: nat, j: nat)
    requires i <= j,
    decreases j - i
{
    ...
}
```

When recursion is structurally obvious to you but not to Verus, add a dedicated decreases witness helper instead of complicating the main lemma.

## Quantifier and trigger discipline

Trigger on the exact term you plan to mention later.

Good trigger shapes:

- `#[trigger] s[i]`
- `#[trigger] map.contains_key(k)`
- paired map triggers: membership plus lookup
- a tiny synthetic trigger helper like `idx_trigger(i)`

Patterns:

```rust
forall |i: int| 0 <= i < s.len() ==> #[trigger] P(s[i])

forall |k|
    #![trigger m.dom().contains(k)]
    #![trigger m.index(k)]
    m.dom().contains(k) ==> ...
```

Do not:

- use arithmetic-only triggers
- use a trigger that omits quantified variables
- expect the solver to instantiate a useful forall unless you assert the trigger term first

## Ghost state choices

Use `Ghost<T>` for erased facts with no linear ownership.

Use `Tracked<T>` for:

- permissions
- protocol tokens
- linear ghost witnesses
- invariant payloads

Keep ownership transfers local to the proof scope where they occur.

For shared mutable proofs:

- use `AtomicInvariant` or `LocalInvariant` to store ghost state at rest
- rebuild the full payload before leaving the open block

For agreement patterns:

- use state-machine tokens if agreement is intrinsic to the protocol
- use `GhostVar` / `GhostVarAuth` when you need one writer and many readers

## Default workflow

1. Find the abstraction boundary already used by the repo.
1. Write the smallest `spec fn` that captures the intended behavior.
1. Add one or two helper predicates for the invariant shape.
1. Write a `proof fn` per reusable proof step.
1. Add loop or protocol invariants that mirror the abstract model.
1. Add explicit triggers and decreases clauses before trying heavier repairs.
1. Only then connect the proof back to exec code.

## Escalation rule

If the task is primarily about `AtomicInvariant`, `atomic_with_ghost!`, `GhostVarAuth`, tokenized state machines, or thread protocols, switch to `verus-concurrency`.
