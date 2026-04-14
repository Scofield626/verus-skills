---
name: verus-proof-repair
description: Repair failing Verus proofs with minimal edits. Use when verification is already written but one or more obligations fail, such as assertions, loop invariants, preconditions, postconditions, trigger instantiation, fuel, or arithmetic proof failures.
---

# Verus Proof Repair

Use this skill when the proof mostly exists and the task is to make it verify with the smallest defensible change.

## Repair priorities

1. Preserve exec behavior.
1. Prefer local proof edits over refactors.
1. Change proof artifacts before changing specs.
1. Add reusable helper lemmas only when local assertions are not enough.
1. Treat `assume` as temporary proof debt, not a finished solution.

## First classify the failure

### Precondition failure

Make the call-site fact explicit just before the call.

Typical repairs:

- add a local `assert(...)`
- pull the relevant loop invariant into scope
- prove a witness before `choose`
- call an existing helper lemma

### Postcondition failure

Strengthen the final loop invariant or add a proof step just before return.

Typical repairs:

- restate the abstract meaning of the final state
- add a local `assert` for the exact ensures clause
- move a missing semantic fact into the loop invariant

### Loop invariant entry or maintenance failure

For entry:

- prove the invariant before the loop
- propagate needed length or bound facts from earlier loops

For maintenance:

- assert the failing invariant at the end of the loop body
- break one large invariant into smaller clauses
- materialize the missing semantic step with a helper lemma

### Assertion failure

Insert the narrowest missing fact first.

Typical repairs:

- assert the trigger term explicitly
- split one opaque jump into multiple assertions
- reveal one recursive spec at the needed depth
- convert a large goal into `calc!` or a helper lemma

### Arithmetic or bitvector failure

Dispatch to the right solver mode.

Patterns:

```rust
assert(expr) by (compute_only);
assert(bit_expr) by (bit_vector);
assert(arith_expr) by (nonlinear_arith);
```

### Trigger or quantifier failure

Assume the issue is trigger mismatch until proven otherwise.

Repairs:

- assert the exact instantiation term
- rewrite the quantifier trigger
- add a synthetic trigger helper
- re-prove the post-state with `assert forall ... by { ... }`

## High-value repair moves

### 1. Narrow with a local assertion

```rust
assert(P(s[k]));
assert(goal_about(s[k]));
```

This is often enough to make an existing quantified hypothesis fire.

### 2. Reveal only what you need

Use `reveal`, `hide`, and `reveal_with_fuel` on the exact function and scope needed.

```rust
proof {
    reveal_with_fuel(triangle, 3);
}
```

Do not globally unfold a recursive or opaque definition unless the whole file depends on it.

### 3. Materialize witnesses explicitly

```rust
assert(exists|i: int| P(i));
let i = choose|i: int| P(i);
```

If no witness is in scope, prove one concretely first.

### 4. Split quantified repairs out of transition proofs

When a state transition breaks a protocol invariant:

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

This is the standard repair for non-inductive post-state invariants.

### 5. Add a decreases witness helper

If Verus cannot see why recursion decreases, isolate that proof.

Prefer:

- explicit `decreases`
- a dedicated decreases helper
- a separate monotonicity lemma

## Diagnostic workflow

1. Read the failing obligation, not just the error summary.
1. Identify whether the missing fact is semantic, arithmetic, or trigger-related.
1. Try the smallest local assertion first.
1. If that fails, add or strengthen the loop invariant.
1. If recursion is involved, inspect reveal/fuel and decreases next.
1. If quantified post-state facts fail, switch to `assert forall ... by { ... }`.
1. Only after those fail should you add a new helper lemma.

## Edits that are usually too large

- changing the algorithm when only the proof is broken
- weakening a spec to avoid proving it
- replacing a proof with broad `assume`
- refactoring many unrelated proof blocks at once

## Concurrency escalation

If the failure is about:

- `AtomicInvariant`
- `LocalInvariant`
- `atomic_with_ghost!`
- `tokenized_state_machine!`
- namespace masks
- storage `deposit` / `withdraw` / `guard`

switch to `verus-concurrency`, because those failures are protocol- and ownership-specific rather than ordinary sequential proof repairs.
