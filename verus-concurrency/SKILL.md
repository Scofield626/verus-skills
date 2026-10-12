---
name: verus-concurrency
description: Design and repair Verus shared-state proofs using tokenized state machines, tracked permissions, resource algebras, invariants, atomic_with_ghost, or logically atomic APIs. Use for concurrent ownership protocols, linearization, invariant masks, and verified threading.
---

# Verus Concurrency

Choose the smallest ownership model that expresses the protocol. Preserve token identity, atomicity, and the client-visible contract.

## Choose the proof mechanism

- Use `atomic_with_ghost!` when one atomic location carries the relevant ghost state.
- Use `tokenized_state_machine!` for a protocol with several ownership states and transitions.
- Use existing resource types for reusable agreement, fractional ownership, or authoritative state.
- Use an existing linearization interface when the repository provides one.
- Consider current `atomically` specifications for compositional logically atomic APIs. The feature is experimental and version-sensitive.

These mechanisms can be combined. A tokenized state machine is not required for every atomic proof.
For current API details and source examples, read [ownership and logical atomicity](references/protocols.md).

## Define protocol resources

Separate immutable configuration, current state, thread-owned capabilities, and stored executable permissions.
Choose sharding according to ownership:

| Sharding | Typical purpose |
| --- | --- |
| `constant` | Shared immutable configuration or identity |
| `variable` | One token for a changing authoritative field |
| `map` | Ownership of distinct keyed entries |
| `set` / `multiset` | Distinct elements / elements with multiplicity |
| `storage_option` / `storage_map` | Stored resources accessed through deposit, withdraw, or guard |

Use `transition!` for state changes and ownership transfer, and `property!` for consequences of held tokens.
Use `readonly!` for read-only observations according to the installed macro API.

Split invariants into domain, range, phase, storage, and ownership conditions.
Prove initialization and preservation for each transition. If a clause is not inductive, find the missing protocol relation before tuning triggers.

## Agreement is not unlimited sharing

`GhostVarAuth<T>` and `GhostVar<T>` form one matching pair.
`agree` requires matching IDs; `update` needs mutable access to both tokens.
Neither token alone permits an update. The pair does not supply arbitrary reader fragments.

For multiple readers or shards, choose a resource with the required split and update rules, or encode the protocol in a state machine.
Do not infer these rules from a resource's name.

For raw pointers, preserve the permission's allocation identity and provenance as well as its address and value.
Address equality alone is insufficient to establish ownership of an allocation.

## Open invariants correctly

An `AtomicInvariant` can share protected state across threads when its payload meets the required bounds.
Its ordinary open block permits at most one supported atomic executable operation, together with proof work.
Use the pinned API's memory model; the standard invariant documentation describes sequentially consistent atomics.

A `LocalInvariant` has no atomic-operation restriction but is not `Sync`.
It cannot provide cross-thread sharing through a shared reference.

For either invariant:

- Check the namespace mask before opening it or calling helpers.
- Do not reopen the same invariant while it is open.
- Restore the complete payload and invariant predicate before exit.
- For proof-mode opening, supply the required `OpenInvariantCredit`; do not invent credits in proof code.

Use `guard` for shared access justified by held tokens, `withdraw` for ownership transfer out of storage, and `deposit` to return a resource.
Prove key and value agreement for storage maps, including unchanged entries.

## Connect implementation steps to abstract steps

Identify the linearization point: the logical instant when an operation takes effect.
For compare-and-exchange, handle success and failure separately. A failed operation can still change the proof's knowledge or retry state.
Do not require every atomic instruction to perform exactly one abstract update; some steps preserve the abstract state.

Linearizability requires a relation between concrete execution and the abstract history, not only an inductive state invariant.
Helping can move another thread's operation to completion. Track request identity and completion evidence across that transfer.

## Thread and failure paths

Use `Arc` for shareable executable state and tracked capabilities for the ownership protocol.
Use the repository's verified spawn and join wrappers when the proof crosses thread boundaries.
Return or store produced tokens on every success, retry, and error path.
Do not create missing capabilities with `Tracked::assume_new()`.

For repair, classify the failed layer: token ownership, invariant preservation, mask, atomicity, or refinement.
Then run the focused proof and the repository's complete verification, lint, and relevant tests.
Safety or linearizability alone does not establish termination, fairness, lock freedom, or liveness.
