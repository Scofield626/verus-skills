______________________________________________________________________

## name: verus-style description: Apply maintainable Verus code style. Use when reviewing or editing Verus code for module layout, naming, spec/proof/exec separation, trigger discipline, broadcast facts, ghost ownership style, or common vstd import patterns.

# Verus Style

Use this skill when the goal is not just to verify code, but to keep the proof development readable, repairable, and stable under solver changes.

## Module layout

Prefer an explicit split between:

- abstract models and predicates
- reusable proof lemmas
- executable wrappers

Repository patterns to follow:

- `spec/`, `proof/`, `exec/` directories
- `_t.rs` for trusted or interface-facing code
- `_v.rs` for verified implementation and proof-heavy code

If one file must contain all three layers, separate them by sections or trait blocks. Do not interleave spec, proof, and exec line by line.

## Naming

Use names that reveal proof role.

- `spec_*` for mathematical helpers
- `lemma_*` for reusable proof steps
- `theorem_*` for roundtrip or inverse-direction facts
- `wf` / `well_formed` for representation invariants
- `*_trigger` only for synthetic trigger helpers

Name proof steps by the obligation they discharge, not by the syntax they use.

## Spec / proof / exec separation

Keep each layer doing one job.

- `spec fn`: define models and relations
- `proof fn`: prove reusable facts
- `exec fn`: implement behavior and call proof code at the boundary

Preferred pattern:

```rust
spec fn abs_model(...) -> ... { ... }
proof fn lemma_model_step(...) ensures ... { ... }
fn exec_impl(...) ensures result@ == abs_model(...) { ... }
```

Do not let a `proof fn` become a second implementation of the algorithm.

## Invariant style

Loop invariants should contain both bounds and semantics.

Type invariants should expose a stable abstract view.

Protocol invariants should be decomposed into small named predicates, usually by:

- domain completeness
- ordering or range facts
- ownership agreement
- storage agreement
- phase-specific facts for enum variants

If an exec method depends on a type invariant, bring it back with `use_type_invariant`.

## Broadcast facts

Use `broadcast proof fn` for reusable algebraic or extensional facts, then `broadcast use` them locally.

Prefer this:

```rust
broadcast proof fn encode_decode(...) ensures ...
{
    ...
}

proof fn theorem(...)
{
    broadcast use encode_decode;
    ...
}
```

Do not leave heavy broadcast imports wider than necessary.

Prefer modern `broadcast proof fn` / `broadcast use` over old `broadcast_forall` style unless you are matching existing local code.

## Trigger discipline

Write explicit triggers in nontrivial quantified formulas.

Good trigger shapes:

- exact lookup terms like `s[i]`
- `map.contains_key(k)`
- paired domain-and-lookup triggers for maps
- tiny synthetic helpers when the natural body has no stable trigger

Common patterns:

```rust
forall |i: int| 0 <= i < s.len() ==> #[trigger] P(s[i])

forall |k|
    #![trigger m.dom().contains(k)]
    #![trigger m.index(k)]
    m.dom().contains(k) ==> ...
```

Avoid:

- arithmetic-only triggers
- triggers that omit quantified variables
- giant trigger terms that almost never appear in proof code

## Ghost ownership style

Use `Ghost<T>` for erased non-linear facts.

Use `Tracked<T>` for:

- permissions
- tokens
- invariant payloads
- linear ghost witnesses

Keep ownership transfers inside the narrowest proof scope that performs them.

When opening an invariant:

- reconstruct the payload completely before exit
- do not leak partially moved state across the boundary

## Common `vstd` import patterns

`vstd::prelude::*` is the default.

Add narrow imports based on proof shape:

- `seq`, `map`, `set`, `multiset` for abstract state proofs
- `invariant`, `atomic_ghost`, `modes`, `cell` for concurrency or ghost ownership
- `calc_macro` for structured rewrites
- solver modes like `compute_only`, `bit_vector`, and `nonlinear_arith` deliberately and locally

Use existing `vstd` lemmas and macros before writing custom low-level boilerplate.

## Review checklist

When editing or reviewing Verus code, ask:

1. Is the abstraction split clear?
1. Are names revealing proof role?
1. Is the invariant decomposed into reusable parts?
1. Are triggers explicit and syntactically useful?
1. Is ghost ownership local and linear where it should be?
1. Are heavy `vstd` imports and broadcast facts kept local?

## Escalation rule

If the task is specifically about concurrent protocols, `AtomicInvariant`, `atomic_with_ghost!`, tokenized state machines, or thread sharing patterns, switch to `verus-concurrency`.
