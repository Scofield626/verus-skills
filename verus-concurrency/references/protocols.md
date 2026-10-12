# Ownership and logical atomicity

Reviewed 2026-10-12. The APIs here follow current Verus sources; older project pins can use different module paths and contracts.

## Resource identity and update authority

Current `GhostVarAuth` and `GhostVar` are exported through `vstd::resource::ghost_var`.
They hold two agreeing halves. `agree` requires equal IDs, and changing the value requires both halves.
The [implementation and contracts](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/vstd/resource/impls/ghost_var.rs) make these requirements explicit.

Complete item, inside `verus!` with the prelude imported:

```rust
proof fn update_agreeing_pair() {
    let tracked (mut auth, mut frag) =
        vstd::resource::ghost_var::GhostVarAuth::<int>::new(5);
    auth.agree(&frag);
    auth.update(&mut frag, 12);
    assert(auth@ == 12 && frag@ == 12);
    assert(auth.id() == frag.id());
}
```

For more complex sharing, inspect the chosen resource's validity, split, join, and update conditions.
The current [resource module](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/vstd/resource/mod.rs) contains fractional, authoritative, agreement, and collection resources.
Do not copy an old `vstd::pcm` path without checking the pinned library.

## State-machine storage and real permissions

The verified allocator's [protocol](https://github.com/verus-lang/verified-memory-allocator/blob/77d1fb6fc1907c903764863a479479a0a0ed04f5/verus-mimalloc/tokens.rs)
uses separate maps for thread, page, block, and segment ownership and storage maps for shared access permissions.
Its IDs retain provenance and allocation-related identity, beyond raw addresses.

The reusable pattern is to keep protocol ownership separate from the physical resource it authorizes.
A transition must establish their agreement when depositing, withdrawing, or guarding a permission.
For a read-only guard, prove that the held capability prevents an incompatible update for the borrow's lifetime.
Do not assume all set-like tokens have the same duplication or multiplicity rules.

Use the official [storage-map reference](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/state_machines/src/strategy-storage-map.md)
and [tokenization reference](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/state_machines/src/tokenization-reference.md)
for the generated API, including instance identity and token-exchange requirements.

## Invariant scope, masks, and credits

[Invariant contracts](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/vstd/invariant.rs) distinguish `AtomicInvariant` from `LocalInvariant`.
The atomic form allows a single supported atomic operation while open; the local form gives up cross-thread shared access.
Restore the predicate after ghost rearrangement, including failure branches and partially moved fields.

An invariant namespace identifies which invariant may be opened. A mask records which namespaces a scope permits.
Helpers must respect the remaining mask; `opens_invariants none` is appropriate only if their bodies satisfy it.

Opening an invariant in proof mode consumes an `OpenInvariantCredit` obtained through the supported executable interface.
Do not manufacture credits or hide a missing credit behind an assumption.

## Logically atomic APIs: current and experimental

Logical atomicity lets a client reason about a multi-step operation as one abstract atomic effect.
The [current guide](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/logatom.md) explicitly marks this feature experimental.
The public interface uses an `atomically` specification and a tracked `AtomicUpdate`.

Keep four states separate:

1. Function entry.
1. Just before the linearization point.
1. Just after the linearization point.
1. Function return.

A function's ordinary precondition need not describe the shared state at the later linearization point.
Use the atomic precondition and postcondition for that interval.
The [specification guide](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/logatom-spec.md) describes inputs, outputs, and masks.

The implementation opens its update with `try_open_atomic_update!`.
A committed update is consumed; an aborted attempt returns the update for retry.
The function must establish that its update has resolved before return.
The [opening guide](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/logatom-open.md) and
[complete reset and CAS examples](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/examples/guide/logatom.rs) show the exact syntax.
Use those version-matched examples rather than inventing a generic macro skeleton.

At the call site, `atomically loop` supports retry after abort.
The client must restore the resources required for another attempt, break on commit, and continue on abort.
For invariant-based clients, account for a credit on each opening; the example abort path returns one for the next attempt.
Close invariants before control flow leaves their blocks.
See the [client guide](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/logatom-call.md).

**Masks:** the client can open invariants in the outer mask, while the inner mask constrains which can remain open around `update`.
The implementation must be allowed to open the outer mask and may open only the inner mask inside its update body.
Do not widen masks solely to suppress a proof error.

**Helping:** another thread may commit an operation's update. Transfer the tracked update through a suitable invariant and retain completion evidence.
See the [helping example](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/examples/helping.rs).
This protocol proves the claimed atomic behavior; it does not automatically prove progress or wait freedom.

## Existing projects may use another interface

The guide distinguishes older ghost-callback interfaces, including `vstd::logatom`, from the newer atomic-update syntax.
Preserve an existing callback contract unless migration is part of the task.
Node replication's [history refinement](https://github.com/verus-lang/verified-node-replication/blob/6aed637f56043e508d3d187c4980431a7fce9373/verified-node-replication/src/spec/linearization.rs)
is another established approach. An invariant-only argument is not a substitute for its history relation.
