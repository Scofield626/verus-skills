---
name: verus-proof-writing
description: Write new Verus specifications and proofs, including abstraction boundaries, loop invariants, mutable-borrow contracts, refinement, helper lemmas, and ghost state. Use for new verified code or proof design rather than one localized failing obligation.
---

# Verus Proof Writing

Build the proof around the behavior callers need, then connect the implementation to that model.

## Start with the repository's model and version

Find the verification command, pinned Verus revision, features, and existing `View` or state-machine model.
Use the repository's toolchain. Current documentation can describe syntax that its pinned version does not support.

Follow existing `spec/`, `proof/`, and `exec/` boundaries where present.
Treat `_t`, `_v`, and `_u` suffixes as project conventions. Inspect declarations to identify what is actually trusted.

For current mutable references, iterator changes, and type-invariant limits, read
[current language patterns](references/current-language.md).
For parsers, storage, controllers, or layered system models, read
[project proof patterns](references/project-patterns.md).

## Design a useful contract

- Use `spec fn` for the mathematical model and `proof fn` for logical steps.
- Express executable results through views, representation invariants, or a refinement relation.
- State the allowed input domain and the behavior of success and failure cases.
- Distinguish implementation-to-model correctness from properties of the model itself. Prove both when the task requires both.
- Check empty, boundary, and error cases before building a large proof around a specification.

Spec functions are total. `recommends` describes intended use; it is not an enforced precondition or an assumed fact.
Prove index bounds, map membership, enum variants, and witness existence where their meaning matters.
An out-of-domain spec lookup does not justify a useful value.

Do not add `assume`, `admit`, `external_body`, or an axiom to complete a proof obligation.
Keep any existing trusted boundary explicit. A verified implementation establishes its specification under those assumptions.

## Build inductive invariants

For loops, describe the processed prefix, accumulator, and unprocessed region as needed.
Include bounds and stable facts that the body and exit proof use.
With loop isolation, facts from before the loop may need to appear in its invariant.

Check three obligations separately: entry, preservation by one iteration, and sufficiency at exit.
Treat `break` and early `return` paths explicitly; inspect the installed version's loop-clause rules.
Use a decreasing measure for termination when required. Partial correctness alone does not prove termination.

For datatypes, expose a stable abstract view. Use a private type invariant when its enforcement rules fit the implementation.
Use an explicit `wf` predicate when arbitrary spec values or intermediate construction states need separate treatment.

For protocols, split domain, ownership, storage, and phase constraints into named predicates.
If preservation fails, first check whether the invariant is inductive. Syntax changes cannot repair a missing semantic condition.

## Prove one useful step at a time

Search local lemmas and the pinned `vstd` before inventing helpers.
Use an explicit lemma call for a particular instance, and induction for an unbounded recursive property.
Extra reveal fuel handles a bounded unfolding; it does not replace induction.

Use `assert(goal) by { ... }` to keep intermediate facts out of the following proof context.
Move a subproof to a helper when a small contract reduces solver work, even if the helper has only one caller.

To prove a universal property, introduce an arbitrary element with `assert forall ... by` and prove the pointwise case.
For collection equality, establish lengths or domains and pointwise agreement. Use `=~=` explicitly when extensionality is the key step.
Use `=~~=` for nested collection equality when appropriate.

## Control quantifier automation

Inspect inferred triggers before overriding them. Choose terms that occur at real use sites.
Each trigger group must cover every quantified variable.
Multiple `#![trigger ...]` attributes are alternatives; comma-separated terms inside one attribute form a group that requires all terms.

Prefer direct lemma calls to synthetic trigger helpers unless the helper has a clear activation strategy.
Watch for matching loops: an instantiation that creates a larger term matching the same trigger can cause unbounded solver work.

Use a local `broadcast use` when several instances are useful. Broader imports can be appropriate when measured proof cost stays acceptable.

## Choose ghost ownership deliberately

Use `Ghost<T>` for erased mathematical data without ownership control.
Use `Tracked<T>` for permissions and tokens whose ownership must be enforced.
Preserve token identity as well as token value when moving permissions between components.

`GhostVarAuth` and `GhostVar` are an agreeing pair. Updating requires both matching tokens.
For concurrency or logical atomicity, use `verus-concurrency` if available; its ownership rules need separate treatment.

## Finish with evidence

Run the focused verification while developing, then the repository's full verification, lint, and relevant tests.
Check that the intended implementation and feature configuration were included.
Report verification scope and remaining assumptions. Ordinary Rust compilation is not proof verification.
