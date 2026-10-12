# Project proof patterns

These are practical deductions from the linked code, not requirements imposed by Verus.
Source links pin revisions, which can differ from the published artifacts.

## Compose a model proof with an implementation proof

Use two interfaces: a pure model with proved laws, and executable operations proved to implement that model.
Callers can then reuse model laws without reopening representation details.

Vest illustrates this separation in [proof traits](https://github.com/secure-foundations/vest/blob/11af01ddaef76cceded03517cd48764adb9d35b1/vest_lib/src/core/proof.rs) and
[executable parser contracts](https://github.com/secure-foundations/vest/blob/11af01ddaef76cceded03517cd48764adb9d35b1/vest_lib/src/core/exec/parser.rs).
Its current parser contract relates both success and failure to the pure parser and carries the consumed byte count and deep view.

For serialization code, separate these obligations:

- Serializing and then parsing recovers the original valid value.
- Parsing and then serializing reproduces the consumed bytes.
- Composition preserves the needed input-prefix behavior.
- Repetition makes progress when its proof requires positive consumption.

These are distinct properties. A successful roundtrip in one direction does not automatically establish the other.
Current Vest factors them into proof traits and derives higher-level laws through blanket implementations.
For example, its destination-buffer roundtrip, serializer validity, and serializer-equivalence traits support a derived ordinary roundtrip.
Do not assume the 2025 artifact's combinator names remain current.

**Use when:** building a reusable verified parser, encoder, or compositional library.
First identify which laws composition needs; avoid forcing every component to prove unrelated properties.

## Check the strength of the end-to-end theorem

Verdict's [top-level validator](https://github.com/secure-foundations/verdict/blob/9bc18bc5a287c1608dcb74a6eee7b9ca94a62b67/verdict/src/validator.rs) connects a successful `Ok` result to a policy-based specification.
It also uses explicit shallow and deep view equalities to connect byte collections across layers.
Its contract is conditional on `Ok`; that alone does not specify every error's meaning or establish that an error cannot occur.

**Use when:** proving a wrapper around several verified components.
Track each conversion and each failure branch. Preserve the distinction between acceptance soundness, completeness, and full result equivalence.

## Put crash safety on the write capability

In verified-storage, [write-restricted persistent memory](https://github.com/microsoft/verified-storage/blob/41cab95051229ebd457b4eed7fc111e3e7c9099c/pmemlog/src/pmemspec_t.rs)
requires permission for every modeled partially flushed outcome of a write.
The [trusted wrapper](https://github.com/microsoft/verified-storage/blob/41cab95051229ebd457b4eed7fc111e3e7c9099c/pmemlog/src/main_t.rs) controls construction of the permission and relates storage to the recovered abstract state.
This places the crash obligation at the operation that can create an intermediate persistent state.

**Use when:** normal return postconditions cannot express failures during an operation.
Define the recovery interpretation, state which recovered outcomes are allowed, and require each physical write to preserve that condition.
Keep corruption assumptions and initialization requirements explicit; crash consistency does not by itself establish corruption detection.
The [PoWER repository](https://github.com/microsoft/verified-storage/blob/41cab95051229ebd457b4eed7fc111e3e7c9099c/README.md) identifies its OSDI 2025 artifact branch separately from current development.

## Prove adjacent refinement layers

Refinement means that concrete behavior implements an abstract specification.
A large system can need intermediate models rather than one direct concrete-to-abstract proof.

Node replication connects a replicated log model to an asynchronous singleton in
[linearization.rs](https://github.com/verus-lang/verified-node-replication/blob/6aed637f56043e508d3d187c4980431a7fce9373/verified-node-replication/src/spec/linearization.rs).
The NrOS page-table project connects OS and hardware state to a user-facing model in
[os_refinement.rs](https://github.com/matthias-brun/verified-nrkernel/blob/1c861090537664b78544cda5d53711b1fcfa05e2/page-table/src/impl_u/os_refinement.rs).

**Use when:** hardware behavior, batching, or concurrent histories make a direct proof too large.
Choose a relation for each adjacent model, prove initial-state correspondence, then map each concrete step to abstract progress or an unchanged abstract state.
Keep implementation invariants separate from the externally visible specification.

CortenMM's [node model helpers](https://github.com/TELOS-syslab/CortenMM-Artifact/blob/604592d815cb1cb7d4619f202b5d7ff471a3a7af/verification/common/src/spec/node_helper.rs)
provide another useful decomposition: conversion and inverse lemmas between node identifiers, tree traces, and levels.
Prove these representation bridges once before using them throughout traversal proofs.
Use this as a decomposition example; some helpers contain `admit()`.

## Keep liveness separate from step correctness

Anvil's [Reconciler contract](https://github.com/anvil-verifier/anvil/blob/a597f9de7dbeb056403bd60b0a2ca67f686a5e1f/src/reconciler/exec/reconciler.rs) connects each executable reconciliation step to its pure model.
The controller's [liveness specification](https://github.com/anvil-verifier/anvil/blob/a597f9de7dbeb056403bd60b0a2ca67f686a5e1f/src/controllers/vreplicaset_controller/proof/liveness/spec.rs)
adds temporal assumptions, including fairness and eventual disabling of modeled failures.
The [target property](https://github.com/anvil-verifier/anvil/blob/a597f9de7dbeb056403bd60b0a2ca67f686a5e1f/src/controllers/vreplicaset_controller/trusted/liveness_theorem.rs)
relates a stable desired state to eventual stable agreement of the cluster state.

**Use when:** the task asks for eventual progress in a distributed or reactive system.
First prove local refinement and safety invariants. Then state scheduling and environment assumptions and prove progress under them.
Do not infer liveness from a verified step function, inductive invariant, or terminating local loop.
A fairness assumption constrains scheduling; it is not established merely by writing it in a theorem's precondition.
