# Diagnostics and proof stability

Reviewed 2026-10-12. Use the target repository's pinned verifier and build command.
The options below are supported by the inspected current Verus sources; check `verus --help` for older pins.

For regressions after an upgrade, check the
[mutable-reference migration](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/migration-mut-ref.md) and
[iterator migration](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/migration-iterators.md)
before treating an obsolete contract or ghost field as a failed mathematical proof.

## Focus the failing proof

For a standalone file with a root-level function:

```sh
verus file.rs --verify-root --verify-function lemma_name --expand-errors
verus file.rs --verify-root --verify-function lemma_name --triggers
```

For a nested module, use `--verify-only-module module_path` with `--verify-function`.
Preserve crate dependencies and feature flags; a multi-crate project usually needs its existing wrapper or Cargo Verus command.
Focused verification is a development check. Run the complete configured target before declaring the change verified.
The [CLI source](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/rust_verify/src/config.rs) defines selector requirements and diagnostic flags.

## Probe what is known and what is sufficient

The Verus-team [ProofPlumber paper (CAV 2024)](https://www.andrew.cmu.edu/user/bparno/papers/proof-plumber.pdf)
explains a useful role for intermediate assertions: they test both whether available facts imply a candidate fact and whether that fact closes the goal.
Use this to isolate the missing step before trying unrelated edits. The paper describes research tooling; check current editor support before prescribing a proof action.

## Reduce context before raising limits

An SMT solver searches among the facts made available to it. More facts can make a proof slower or less stable.

1. Use `--expand-errors` to expose the failing part of a compound goal.
1. Check a local assertion or helper with the exact domain and semantic facts.
1. Inspect broad `broadcast use` groups and recursive unfolding.
1. Profile the affected function when solver cost remains unclear.

`assert(goal) by { ... }` exposes only the proved goal to subsequent code, while retaining incoming facts inside the block.
A helper with a small contract can also reduce incoming context.
See [assert-by scoping](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/reference-assert-by.md) and
[proof decomposition](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/breaking_proofs_into_pieces.md).

For a timeout:

```sh
verus file.rs --verify-root --verify-function lemma_name --profile --rlimit 1
```

For a proof that succeeds slowly:

```sh
verus file.rs --verify-root --verify-function lemma_name --profile-all
```

The guide recommends a small resource limit for timeout profiling to bound trace size.
Inspect instantiation counts and estimated downstream cost, not just the first quantifier reported.
A highly instantiated quantifier may only be responding to terms generated elsewhere.
See [quantifier profiling](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/profiling.md).

## Trigger groups have different costs

Each `#![trigger term_a, term_b]` is one group requiring both terms.
Two separate trigger attributes create alternatives; either can activate the quantifier.
Every group must cover all bound variables.
An instantiation that produces fresh matching terms can loop indefinitely.

Use `--triggers` to inspect automatic choices before changing them.
Add a useful ground instance or call a lemma explicitly when that is enough.
Do not silence a warning with `#![auto]` and call that a semantic repair.
See [trigger semantics](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/trigger-annotations.md) and
[matching loops](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/multitriggers.md).

## Select a local solver with explicit inputs

| Obligation | Useful approach | Constraint |
| --- | --- | --- |
| Linear integer facts | Default solver | Keep bounds and casts explicit |
| Nonlinear products | Existing arithmetic lemma or `nonlinear_arith` | Supply needed context through local `requires` |
| Fixed-width bit operations | `bit_vector` | Preserve machine widths and operation semantics |
| Concrete spec evaluation | `compute_only` | Must reduce fully without ambient facts |
| Evaluation with a residual symbolic goal | `compute` | SMT must still prove the residual expression |
| Polynomial or modular identities | `integer_ring` where supported | Needs Singular; check supported forms and nonzero divisors |

Example complete items, inside `verus!` with the prelude imported:

```rust
proof fn bounded_product(x: int, y: int)
    requires 0 <= x <= 7, 0 <= y <= 9,
    ensures x * y <= 63,
{
    assert(x * y <= 63) by(nonlinear_arith)
        requires 0 <= x <= 7, 0 <= y <= 9;
}

proof fn bit_mask(x: u32)
    ensures (x & 15u32) <= 15u32,
{
    assert((x & 15u32) <= 15u32) by(bit_vector);
}

spec fn sum_to(n: nat) -> nat
    decreases n,
{
    if n == 0 { 0 } else { n + sum_to((n - 1) as nat) }
}

proof fn concrete_sum() {
    assert(sum_to(6) == 21) by(compute_only);
}
```

Sources: [nonlinear arithmetic](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/nonlinear.md),
[bit vectors](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/bitvec.md), and
[computation](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/assert_by_compute.md).
A local variable's preceding assignment is not part of `compute_only`'s evaluation context.

## Use counterexamples to repair invariants

ExVerus (ICML 2026) motivates a useful workflow: generate a candidate bad state, validate it, and generalize the missing invariant relation.
Its [extracted counterexample validator](https://github.com/claudeyj/exverus/blob/c1470dca3535b0fdd4d885e63bc58776a7308806/vinv/pipeline/validator_extracted.py)
separates assertion failures from compilation errors and identifies where the failure occurs.
Its [README](https://github.com/claudeyj/exverus/blob/c1470dca3535b0fdd4d885e63bc58776a7308806/README.md) records an older Verus pin; do not assume its tooling works unchanged on current Verus.

Apply the idea without requiring the research pipeline:

1. Write the candidate state and the exact obligation it violates.
1. Check whether it satisfies the current precondition or loop invariant.
1. Determine whether it is reachable from program entry or merely allowed at an abstract loop head.
1. Add a general invariant relation, then prove entry, preservation, and exit.

A failed SMT query is not automatically a concrete execution counterexample. Check the source semantics before changing a contract.

## Treat stability as evidence, not luck

[Mariposa](https://github.com/secure-foundations/mariposa/blob/8646e7b1c601f7154e93d85ebb1083ce1ed47f95/README.md) tests SMT queries under semantics-preserving mutations such as renaming, reordering, and reseeding.
Its repository also hosts the Cazamariposas work listed on the Verus publications page.
The practical deduction is to suspect context sensitivity when irrelevant edits change proof results.

For an affected proof, compare a few seeds with `--smt-option smt.random_seed=N`, keeping all other settings fixed.
This is a limited diagnostic, not a proof of stability. Use query-level tooling only when deeper investigation is warranted.
Prefer smaller contexts, explicit useful instances, and predictable lemma boundaries over retaining a lucky seed or unexplained resource increase.
