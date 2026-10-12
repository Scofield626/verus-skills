---
name: verus-proof-repair
description: Repair failing or unstable Verus proofs while preserving executable behavior and intended specifications. Use for failed assertions, contracts, invariants, decreases checks, quantifier instantiation, solver timeouts, or proof regressions after a version change.
---

# Verus Proof Repair

Find the missing obligation before changing the proof. Preserve executable behavior and the intended contract.

## Establish the failure

1. Read the repository's verification command and toolchain pin.
1. Reproduce the failure with the same features and dependencies.
1. Distinguish parsing, type or ownership errors, failed proof obligations, and resource limits.
1. Read the failing clause, then test whether it is actually true under the available assumptions.

Do not assume every quantified failure is a trigger problem.
A false statement, missing domain condition, or non-inductive invariant needs a different repair.
After a version change, check migration notes before rewriting valid proof logic.

For focused commands, profiling, solver selection, and counterexample reasoning, read
[diagnostics and stability](references/diagnostics.md).

## Select the smallest defensible repair

| Failure | First useful checks |
| --- | --- |
| Call precondition | Exact missing fact, domain bound, required token, existing lemma |
| Postcondition | Result's abstract meaning, final loop state, early return and error paths |
| Invariant entry | Initialization and facts carried from preceding code |
| Invariant preservation | Arbitrary loop-head state, modified entries, unchanged entries, missing relation |
| Collection equality | Equal lengths or domains, then pointwise equality; nested views may need `=~~=` |
| Quantifier use | Inferred trigger, available matching terms, variable coverage, matching loops |
| Recursive proof | Induction hypothesis, decreases measure, local reveal depth |
| Arithmetic | Integer versus machine arithmetic, overflow, casts, division domain, solver mode |
| Timeout or unstable result | Excess context, broad broadcasts, unfolding, quantifier profile |

Add a local assertion when it exposes the exact missing fact.
Call a lemma when the fact requires a semantic proof step.
Do not wait until every assertion trick fails before extracting a helper that reduces proof complexity.

## Common repairs

### Keep the context small

Use `assert(goal) by { ... }` to hide intermediate facts from the following code.
For excessive incoming context, isolate a helper. Specialized solver assertions can use explicit local `requires` clauses.
An ordinary `by` block still receives the outer context.

Reveal only the needed definition. Increase recursive fuel only for a known finite unfolding.
Use induction for an unbounded claim.

### Repair quantified post-state facts

Use `assert forall ... by` to fix an arbitrary key or index.
Split modified and unchanged cases, then apply the corresponding pre-state invariant.
First prove that the strengthened invariant holds initially and survives every transition.

### Prove witnesses and domains

Prove `exists|x| P(x)` before relying on `choose|x| P(x)` satisfying `P`.
Prove membership before using a map lookup's intended value.
`recommends` does not supply these facts.

### Diagnose invariant counterexamples

A failing preservation step may describe a loop-head state that is unreachable from entry but allowed by the current invariant.
Find the missing relation that excludes this state, then prove that relation at entry and after one step.
Separate this case from a reachable execution that violates the required behavior.
Do not treat a timeout, compiler error, or arbitrary solver model as a validated counterexample.

## Preserve the proof boundary

Do not finish by weakening a postcondition, strengthening a precondition to exclude intended inputs, or changing the algorithm without justification.
Do not introduce `assume`, `admit`, `external_body`, assumed specifications, or disabled verification to get a green result.
Remove temporary diagnostic assumptions before final verification.

If the specification is false, explain the counterexample and the required behavior change.
If a proof is unstable, retain a semantic repair; a favorable random seed alone is not a repair.

## Verify the result

Run the focused proof, then the repository's full verification, lint, and relevant tests.
Record the toolchain and verified scope. Compare assumptions and public contracts with the starting version.
Use additional seeds or profiles when instability is the issue, not as a mandatory step for every edit.

For token ownership, invariant masks, atomicity, or storage transitions, consult `verus-concurrency` if available.
