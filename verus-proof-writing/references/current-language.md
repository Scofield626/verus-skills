# Current language patterns

Check the target repository's version before applying these patterns.
The examples below are complete items to place inside `verus!` with `vstd::prelude::*` imported.

## Mutable-borrow contracts

Current Verus treats mutable references as first-class values.
For an input `x: &mut T`, use `old(x)` for its entry state and `final(x)` for its final state in postconditions.
When a function returns a borrow, describe how the owner's final value depends on the returned borrow's final value.
Frame untouched fields explicitly.

```rust
fn first_mut(pair: &mut (u64, u64)) -> (r: &mut u64)
    ensures
        *r == old(pair).0,
        *final(pair) == (*final(r), old(pair).1),
{
    &mut pair.0
}

fn use_first_mut() {
    let mut pair = (4u64, 9u64);
    let r = first_mut(&mut pair);
    *r = 8;
    assert(pair == (8u64, 9u64));
}
```

The final value can remain unknown when the callee returns. The postcondition states a relation that survives later caller writes.
See the [mutable-reference guide](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/mutable-references.md).

Do not mix this syntax with a scope using `deprecated_postcondition_mut_ref_style(true)`.
Older versions may require `-V new-mut-ref`; current main enables the feature without that flag.
For tracked mutation, `tracked &mut T` alone does not imply that the target place is ghost state.
Use a tracked-only datatype or, where appropriate, `&mut Tracked<T>` and `mut_ref_tracked`.
See the [migration guide](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/migration-mut-ref.md).

## Carry borrow relations through loops

A loop can lose the relation between a mutable reference and its owner.
Record the relation using `after_borrow(owner)` and `final(reference)`.
`after_borrow` denotes the owner's value after outstanding borrows resolve; it is not a snapshot of its current contents.

```rust
fn set_through_loop() -> (result: u64)
    ensures result == 23,
{
    let mut value = 0u64;
    let r = &mut value;
    loop
        invariant after_borrow(value) == *final(r),
        decreases 0nat,
    {
        *r = 23;
        return value;
    }
}
```

For containers of mutable references, inspect `has_resolved` and the relevant container lemmas before adding assumptions about final values.
The [borrow guide](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/mutable-references.md) explains resolution and loop invariants.

## Iterator specifications have changed

Current iterator proofs use a common interface:

- `iter.index()` counts produced items. For a range starting at five, it is not the current range value.
- `iter.history()` describes items already produced.
- `iter.seq()` describes the full sequence; prefixes use `take(iter.index())`.
- Iterators yielding `&T` produce `Seq<&T>`; use `.unref()` when a sequence of values is needed.

Custom iterators use the new iterator specification traits rather than the old `ForLoopGhostIterator` interface.
Check [iterator migration](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/migration-iterators.md) and
[iterator specifications](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/iterator-specs.md) before porting old field-based invariants.

## Type invariants are an encapsulation mechanism

`#[verifier::type_invariant]` applies to exec and tracked objects, not arbitrary spec values.
The type must be local to the crate, with no fields public outside that crate.
Use `use_type_invariant(&value)` to make the fact available where allowed.
Do not rely on temporary invariant violations across construction or field updates; check the current enforcement rules.
See [type invariants](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/reference-type-invariants.md).

## Collection equality and domains

For `Seq`, prove length and element agreement; for `Map`, prove domain and value agreement.
`=~=` requests extensional equality; `=~~=` also handles nested collections.
Current Verus promotes equality in several proof positions, so not every equality failure needs a different operator.
See [extensional equality](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/extensional_equality.md).

`recommends` does not act like `requires` on a proof or exec function.
If a useful equation depends on a lookup domain, prove that domain condition.
See [spec versus proof](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/spec_vs_proof.md).
