---
name: verus-concurrency
description: Design and repair concurrent Verus proofs with VerusSync tokenized state machines, ghost tokens, invariants, atomic_with_ghost!, storage permissions, and verified threading.
---

# Verus Concurrency

Use VerusSync as the default approach for shared-state protocols. Follow the repository's existing proof interfaces and pinned Verus version.

## Conceptual stack

Read the proof as four layers:

1. A VerusSync state machine defines the protocol.
1. Generated ghost tokens enforce ownership of protocol fragments.
1. `AtomicInvariant` or `LocalInvariant` stores tokens and permissions between operations.
1. `atomic_with_ghost!` or `open_atomic_invariant!` connects a concrete step to the protocol.

Define the protocol before wiring its tokens into executable code.
An existing proof around one atomic location may need only `atomic_with_ghost!`; preserve that simpler interface when it fits.

## VerusSync: define the protocol first

### Sharding choices

Sharding determines how a state-machine field is represented by owned tokens.

- Use `constant` for immutable configuration or IDs.
- Use `variable` for one authoritative current value.
- Use `map` for per-thread, per-request, per-page, or per-key ownership.
- Use `storage_map` or `storage_option` for resources accessed through `guard`, `withdraw`, or `deposit`.
- Use `set` for distinct elements and `multiset` when multiplicity matters.

Preserve each token's instance identity as well as its value.
For raw-pointer permissions, address equality alone does not establish allocation identity or provenance.

### Transition discipline

Use `init!` to establish the initial state, `transition!` for state changes, and `property!` for safety facts or guards.
Use `readonly!` for read-only observations supported by the pinned macro API.

This complete state machine keeps a counter within a fixed limit:

```rust
use verus_state_machines_macros::tokenized_state_machine;
use vstd::prelude::*;

tokenized_state_machine! {
    Counter {
        fields {
            #[sharding(constant)] pub limit: nat,
            #[sharding(variable)] pub value: nat,
        }

        init! {
            initialize(limit: nat) {
                init limit = limit;
                init value = 0;
            }
        }

        transition! {
            increment() {
                require pre.value < pre.limit;
                update value = pre.value + 1;
            }
        }

        #[invariant]
        pub fn bounded(&self) -> bool {
            self.value <= self.limit
        }

        #[inductive(initialize)]
        fn initialize_inductive(post: Self, limit: nat) {}

        #[inductive(increment)]
        fn increment_inductive(pre: Self, post: Self) {}
    }
}
```

### How to write the invariant

Split the invariant into domain, range, phase, storage, and ownership conditions.
Prove initialization and preservation for every transition.

For quantified post-state facts, use `assert forall ... by` and split modified and unchanged entries.
If the invariant is not inductive, add the missing protocol relation before tuning assertions or triggers.

For storage protocols and generated token APIs, read [ownership and storage](references/protocols.md#state-machine-storage-and-real-permissions).

## Auth-frag patterns

An authoritative resource records the shared state; fragments describe what individual holders may rely on.

### State-machine auth/frag

Keep agreement in the protocol when it is part of the state-machine invariant:

- Use a `variable` field for the authoritative value.
- Use keyed `map` entries for fragments when ownership is per key.
- Express permitted updates and agreement through transitions and invariants.

### Other resource types

Use an existing resource algebra when its split, join, and update rules fit the task.
Check the pinned library: current APIs use `vstd::resource`, while older code may use `vstd::pcm`.

`GhostVarAuth<T>` and `GhostVar<T>` form one agreeing pair.
`agree` requires matching IDs; `update` requires mutable access to both tokens.
The pair does not provide arbitrary reader fragments.
See [resource contracts and an example](references/protocols.md#resource-identity-and-update-authority).

## Connecting to exec code

### `atomic_with_ghost!`

Use it when the atomic location carries the relevant ghost token.
Keep proof work beside the atomic operation and preserve the relation between its value and the token.

For compare-and-exchange, handle success and failure separately.
An atomic step may advance the protocol or preserve its abstract state; it need not perform exactly one abstract update.

Identify the linearization point: the logical instant when an operation takes effect.
Linearizability needs a relation to the abstract history, beyond an inductive state invariant.

### `AtomicInvariant` and `LocalInvariant`

Store protected tokens and permissions in the invariant and restore its complete payload before closing it.

`AtomicInvariant` supports sharing across threads when its payload meets the required bounds.
Its ordinary open block permits at most one supported atomic executable operation, together with proof work.
The standard invariant API uses sequentially consistent atomics; follow the pinned API's memory model.

`LocalInvariant` has no atomic-operation restriction but is not `Sync`.
It cannot provide cross-thread sharing through a shared reference.

### `open_atomic_invariant!` discipline

- Check namespace masks before opening invariants or calling helpers.
- Do not reopen the same invariant while it is open.
- Keep loops and unrelated executable code outside the block.
- Restore the predicate and all payload fields on every branch.
- Supply an `OpenInvariantCredit` for proof-mode opening; do not manufacture credits in proof code.

### `deposit` / `withdraw` / `guard`

Use `guard` for shared access justified by held tokens.
Use `withdraw` to take ownership from storage and `deposit` to return a resource.

For `storage_map`, prove that withdrawn keys exist and values match.
Prove freshness for deposits and preserve agreement for unchanged entries.
Use a `by { ... }` clause when the transition's safety condition needs an explicit proof.

## Threading

Put shareable executable state in `Arc`. Keep tracked thread capabilities separate.
Use the repository's verified spawn and join wrappers when the proof crosses thread boundaries.

Move each thread's token into its closure and return the completion evidence required by the caller.
Preserve produced tokens on success, retry, and error paths.
Do not create missing capabilities with `Tracked::assume_new()`.

## Concurrent proof repair

Classify the failure by layer:

| Failure | Repair |
| --- | --- |
| Non-inductive invariant | Find the missing relation; prove entry and preservation |
| Missing token | Store produced tokens immediately; track each branch's ownership |
| Namespace conflict | Check masks, shorten open scopes, and use distinct namespaces when required |
| Atomic block violation | Move non-atomic executable work outside the block |
| Storage mismatch | Prove key, value, and instance agreement; check deposit freshness |
| Trigger miss | Inspect inferred triggers; expose the relevant membership or lookup instance |

Use `opens_invariants none` only when the helper's body satisfies that restriction.
Run focused verification, then the repository's full verification, lint, and relevant tests.
Safety or linearizability alone does not establish termination, fairness, lock freedom, or liveness.

## Optional: logically atomic APIs

Keep an existing linearization interface when the repository provides one.
For a task that needs compositional logically atomic contracts, read [the experimental API guidance](references/protocols.md#logically-atomic-apis-current-and-experimental).
Check version support before using `atomically` specifications.

## Escalation rule

For sequential proofs without shared-state protocol machinery, use `verus-proof-writing` or `verus-proof-repair` if available.
