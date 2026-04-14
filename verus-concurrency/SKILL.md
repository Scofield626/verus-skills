______________________________________________________________________

## name: verus-concurrency description: Design and repair concurrent Verus proofs. Use when working on tokenized_state_machine! protocols, ghost tokens, AtomicInvariant or LocalInvariant, atomic_with_ghost!, GhostVar or GhostVarAuth, deposit/withdraw/guard storage patterns, or Arc and thread::spawn verified sharing.

# Verus Concurrency

Use this skill for concurrency, atomicity, and shared-state reasoning only.

## Conceptual stack

Read the proof as four layers:

1. VerusSync state machine defines the protocol.
1. Generated ghost tokens enforce ownership of protocol fragments.
1. `AtomicInvariant` or `LocalInvariant` stores tokens and permissions at rest.
1. `atomic_with_ghost!` or `open_atomic_invariant!` connects one concrete step to one ghost transition.

If the proof does not fit this stack yet, build the missing higher layer first. Do not start from raw atomics.

## VerusSync: define the protocol first

### Sharding choices

Use `constant` for immutable global configuration or IDs.

Use `variable` for one authoritative current value.

Use `map` for per-thread, per-request, per-page, or per-key ownership.

Use `storage_map` or `storage_option` when the field stores an exec permission that must support `guard`, `withdraw`, or `deposit`.

Use `multiset` or `set` for many interchangeable holders such as readers or pending requests.

### Transition discipline

Use `transition!` for ownership movement.

Use `property!` to derive a guard or safety fact from held tokens.

Use `readonly!` only for genuine read-only protocol observations.

Preferred skeleton:

```rust
tokenized_state_machine! {
    Protocol {
        fields {
            #[sharding(constant)] cfg: Cfg;
            #[sharding(variable)] phase: Phase;
            #[sharding(map)] owners: Map<Key, OwnerState>;
            #[sharding(storage_map)] store: Map<Key, Perm>;
        }

        transition!{
            start(k: Key) {
                ...
            }
        }

        property!{
            borrow_perm(k: Key) {
                have owners >= [k => let _];
                guard store >= [k => let p];
            }
        }

        #[invariant]
        pub fn wf(&self) -> bool {
            ...
        }
    }
}
```

### How to write the invariant

Split the invariant into small clauses:

- domain completeness
- ordering or range bounds
- phase agreement
- storage agreement
- ownership-count agreement

For nontrivial post-state repairs, use:

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

This is the default repair for non-inductive concurrent invariants.

## Auth-frag patterns

Use auth-frag when one component can update a logical value and many others must observe or locally carry fragments of it.

### Option A: state-machine auth/frag

Use this when agreement is intrinsic to the protocol itself.

- authoritative value is a `variable` field
- fragments are usually keyed `map` entries
- updates are protocol transitions

### Option B: `Resource<...>` from `vstd::pcm`

Use this when you need a reusable resource algebra independent of one protocol.

- good for pure agreement or monotone resources
- less direct for atomic protocol integration

### Option C: `GhostVar` / `GhostVarAuth`

Use this when you need one updater plus many agreeing readers or shards.

This is the standard pattern for concurrent logical views stored in invariants.

Template:

```rust
let tracked (mut auth, mut frag) = GhostVarAuth::<T>::new(v0);
auth.agree(&frag);
auth.update(&mut frag, v1);
```

Sharded update pattern:

```rust
open_atomic_invariant_in_proof!(credit => &inv => inner => {
    let tracked mut shard = inner.shards.tracked_remove(shard_id);
    auth_frag.agree(&shard.kv_state);
    auth_frag.update(&mut shard.kv_state, new_shard_view);
    completion = lin.apply(op, new_combined_view, result, &mut inner.combined);
    inner.shards.tracked_insert(shard_id, shard);
});
```

## Connecting to exec code

### `atomic_with_ghost!`

Use it when the atomic location itself carries the relevant ghost token.

Template:

```rust
let res = atomic_with_ghost!(
    &self.atomic => compare_exchange(expected, desired);
    update old_val -> new_val;
    returning ret;
    ghost g => {
        if ret.is_ok() {
            g = self.inst.borrow().transition(g, ...);
        }
    }
);
```

Keep the block tiny. One atomic op, one ghost update.

### `AtomicInvariant`

Store inside it any ghost state that must persist between operations:

- atomic permissions
- hidden `PointsTo<T>` permissions
- state-machine tokens at rest
- `GhostVarAuth<T>` plus matching fragments
- one-shot or refcount halves

Open it only around the atomic action or the short proof step that needs it.

### `open_atomic_invariant!` discipline

Inside the block:

- perform only atomic exec code
- do proof-only ghost rearrangement
- rebuild the full payload before exit

Do not:

- re-open the same invariant recursively
- call helpers that open arbitrary invariants unless masks allow it
- put loops or unrelated exec code inside the block

### `deposit` / `withdraw` / `guard`

Use `guard` when read-only stability is enough.

Use `withdraw` when a thread needs exclusive ownership of the exec resource.

Use `deposit` when returning ownership to shared storage.

Patterns:

```rust
property!{
    do_guard(v: (int, PointsTo<T>)) {
        have shared_guard >= {v};
        guard storage >= Some(v.1);
    }
}

transition!{
    writer_take() {
        withdraw storage -= Some(let p);
    }
}

transition!{
    writer_put_back(p: PointsTo<T>) {
        deposit storage += Some(p);
    }
}
```

For `storage_map`, prove domain and value agreement explicitly in the `by { ... }` block.

## Threading

Put shareable exec state in `Arc`.

Keep linear thread capabilities separate.

Typical split:

- `Arc<...>` for shared object
- one tracked token per thread or closure
- join returns an upgraded tracked witness

Patterns:

```rust
let shared = Arc::new(SharedState { ... });
let shared2 = shared.clone();
let handle = vstd::thread::spawn(
    move || -> (ret: Tracked<Witness>)
        ensures ret@@ is Complete,
    {
        thread_routine(shared2, Tracked(my_token), Ghost(thread_id))
    }
);
let done = handle.join().unwrap();
```

Node-replication style:

```rust
let nr = Arc::new(NodeReplicated::new(...));
std::thread::spawn(move || {
    let mut tkn = tkn;
    match nr.execute_mut(op, tkn, Tracked::assume_new()) {
        Ok((_, new_tkn, _)) | Err((new_tkn, _)) => { tkn = new_tkn; }
    }
});
```

## Concurrent proof repair

Classify the failure by layer.

### Non-inductive invariant

Repair with:

- smaller invariant clauses
- explicit `assert forall ... by`
- local trigger-forcing asserts

### Missing token

Repair with:

- `Option<Tracked<_>>` storage across success branches
- immediate save of produced tokens
- explicit unwrap only after success is known

### Namespace conflict

Repair with:

- distinct namespaces
- shorter open scopes
- helpers marked `opens_invariants none`

### Atomic block violation

Repair with:

- move non-atomic exec code outside
- leave only the atomic instruction and ghost update inside

### `storage_map` mismatch

Repair with:

- explicit proof that every withdrawn key exists
- explicit proof that every withdrawn value matches pre-state
- explicit freshness or admissibility proof for deposits

### Trigger miss inside an invariant proof

Repair by asserting the precise membership or lookup term first.

Typical pattern:

```rust
assert(inner.shards.contains_key(self.shard));
assert(shard_of_key(key, self.inv.constant().nshard()) == self.shard);
```

## Escalation rule

If the task is ordinary sequential proof construction or repair without shared-state protocol machinery, switch to `verus-proof-writing` or `verus-proof-repair`.
