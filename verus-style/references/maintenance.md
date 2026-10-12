# Maintenance evidence and review rules

Reviewed 2026-10-12. These sources support decisions about proof maintenance; they do not impose one universal project layout.

## Treat automation as a configurable proof interface

The Verus-team paper [Tunable Automation in Automated Program Verification (FMCAD 2026)](https://doi.org/10.34727/2026/isbn.978-3-85448-093-8_38)
studies control over ambient quantified facts through broadcasts.
A useful workflow is to enable a relevant group during development, then trim it when profiling shows excess context.
The paper also describes trait-level groups that expose mathematical laws without exposing a representation.
This supports deliberate module-wide defaults as well as local imports; always-local is too strong a rule.

The current [broadcast guide](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/broadcast_proof.md) documents
`-V axiom-usage-info` as experimental tooling that overapproximates which broadcast facts were used.
Use its output as a guide, then reverify after removing imports.
It is not an exact dependency proof or a complete audit of trusted code.

[Trigger annotations](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/trigger-annotations.md) distinguish alternative trigger groups from groups requiring several terms.
Inspect automatic selection before overriding it. Make activation terms match real clients, and avoid self-generating matching loops.

## Distinguish information hiding from solver opacity

An open spec body can expose representation details to clients. A closed body protects that boundary.
Opacity controls automatic unfolding where a definition is available; it is a separate choice.
Use local reveal and small proof contracts to manage context rather than exposing private representation details.
See [spec visibility](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/spec_functions.md) and
[reveal and fuel](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/reference-reveal-hide.md).

## Inspect the actual theorem and trusted boundary

[Verdict's validator](https://github.com/secure-foundations/verdict/blob/9bc18bc5a287c1608dcb74a6eee7b9ca94a62b67/verdict/src/validator.rs) provides a concrete example of a result specification conditional on `Ok`.
Review the implication direction before describing what a wrapper proves.
Also inspect cryptographic and external interfaces separately from verified caller code.

[Verified-storage](https://github.com/microsoft/verified-storage/blob/41cab95051229ebd457b4eed7fc111e3e7c9099c/pmemlog/src/main_t.rs) restricts capability construction in its trusted interface.
[NrOS page tables](https://github.com/matthias-brun/verified-nrkernel/blob/1c861090537664b78544cda5d53711b1fcfa05e2/README.md) document `_t` and `_u` conventions, while verified-storage uses `_t` and `_v`.
Suffixes organize review; declarations, tool configuration, and contracts determine the actual proof boundary.

The official [trusted computing base guide](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/guide/src/tcb.md)
explains the components assumed for verification to be meaningful.
Keep project-specific hardware and environment assumptions visible alongside those tool assumptions.
Do not expand a trusted wrapper to make an ordinary proof failure disappear.

[VerusBelt (PLDI 2026)](https://iris-project.org/pdfs/2026-pldi-verusbelt.pdf)
provides a semantic foundation for a substantial subset of Verus, including proof-oriented types, lifetimes, concurrency, and mutable borrows.
It does not verify the Verus implementation or every standard-library specification. Do not turn a subset soundness result into an end-to-end toolchain claim.
Use current migration notes for syntax: a paper can describe planned capabilities that have since shipped.

## Pin versions and preserve verification scope

Read migration notes before applying current examples to an old artifact.
The [mutable-reference migration](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/migration-mut-ref.md) changes postcondition syntax and tracked-place rules.
The [iterator migration](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/migration-iterators.md) replaces iterator-specific ghost fields with a common interface.
Current resource modules also differ from older imports.

A current project branch is evidence of practice at its pinned revision, not proof that the published artifact had the same interface.
A successful Rust build can erase proof annotations without checking them.
Use the project's verification command and required features, followed by its lint and relevant runtime tests.
See [Cargo Verus](https://github.com/verus-lang/verus/blob/4bfc2b4484efc35ed373372a35f69b45f2483ea5/source/docs/CARGO-VERUS.md) and
[Verdict's separate build instructions](https://github.com/secure-foundations/verdict/blob/9bc18bc5a287c1608dcb74a6eee7b9ca94a62b67/README.md).
