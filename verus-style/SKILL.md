---
name: verus-style
description: Review or edit Verus code for maintainable specifications, proof boundaries, module layout, naming, trigger and broadcast discipline, ghost ownership, and version-aware vstd usage.
---

# Verus Style

Keep proof interfaces clear and solver dependencies deliberate. Follow the repository's coding rules and pinned Verus version.

## Organize around proof boundaries

Separate mathematical models, reusable lemmas, and executable behavior where the repository does so.
Small implementations can keep related spec, proof, and exec items together.
Local proof blocks beside the operation they justify often improve readability.

Use `View` or `DeepView` consistently with the existing model. Explain representation invariants independently of storage details.
Do not assume `_t`, `_v`, or `_u` suffixes establish trust by themselves.
For example, verified-storage and verified-nrkernel use different verified-file suffixes.

Use descriptive lemma names. Match existing `spec_*`, `lemma_*`, `theorem_*`, and `wf` conventions instead of renaming unrelated code.
Comments should explain the missing mathematical step, framing condition, or trust assumption.

## Keep contracts readable and complete

Group bounds, representation facts, and behavioral facts into separate clauses.
Use named predicates for invariant clauses that appear across several transitions.
State success and failure behavior to the extent required by callers.
Do not describe a success-only implication as total equivalence.

For mutable references, distinguish entry, final, and outstanding-borrow values using the syntax supported by the pinned toolchain.
Do not mix old postcondition conventions with current `final` semantics.

Type invariants encapsulate validity of exec and tracked objects; arbitrary spec values do not inherit that validity.
Use `use_type_invariant` when its fact is needed and permitted.

## Expose only useful automation

Start with an existing lemma or explicit proof step. Make a lemma `broadcast` when repeated automatic instantiation helps callers.
Group related broadcast lemmas behind a meaningful proof interface.
Measure broad imports before narrowing them; local imports and explicit calls are useful controls, not universal style requirements.

Keep `open` versus `closed` separate from opacity:

- Visibility controls what clients can depend on.
- `#[verifier::opaque]`, `reveal`, and fuel control unfolding available to the solver.
- `assert ... by` and helper contracts control how much proof context flows forward.

Inspect inferred triggers before adding annotations. Each alternative trigger group must cover all quantified variables.
Distinguish alternative groups from a multi-term group; adding alternatives can increase instantiations.
Avoid matching loops and synthetic triggers with no natural caller use.

Use `compute_only`, `bit_vector`, and `nonlinear_arith` as local proof modes, not imports.
Search the pinned `vstd` for lemma names and module paths. Resource APIs have moved between versions.

## Make ownership and trust visible

Use `Ghost<T>` for mathematical data and `Tracked<T>` for permissions or protocol resources.
Show where tokens are acquired, transferred, and returned. Preserve their identities across updates.
Keep invariant scopes small and reconstruct their complete payload before exit.

Review `assume`, `admit`, `external_body`, `external`, and assumed specifications as proof boundaries.
An existing trusted wrapper can be necessary; do not expand it to hide an unresolved obligation.
Name the modeled hardware, external library, or environment behavior on which the contract relies.

## Review and validation

Check these points before finishing:

1. The public contract captures the intended behavior and domain.
1. The proof connects the implementation to that contract.
1. Automation choices have clear use sites and acceptable verification cost.
1. Ownership and trust boundaries are explicit.
1. The repository's formatting, lint, tests, and full verification pass for the intended configuration.

Read [maintenance evidence](references/maintenance.md) when choosing broadcast scope, reviewing trusted interfaces, or updating old syntax.
