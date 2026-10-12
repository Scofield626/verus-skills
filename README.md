# Verus Skills

Four agent skills for writing, repairing, and maintaining Verus proofs:

- [verus-proof-writing](verus-proof-writing/SKILL.md): specifications, invariants, mutable borrows, refinement, and new proofs.
- [verus-proof-repair](verus-proof-repair/SKILL.md): failed obligations, proof regressions, and solver performance.
- [verus-style](verus-style/SKILL.md): proof interfaces, automation scope, ownership, and trusted boundaries.
- [verus-concurrency](verus-concurrency/SKILL.md): shared-state protocols, permissions, invariants, and logical atomicity.

Each skill keeps its main workflow in `SKILL.md`. Read its linked references only when their subject applies.
References stay inside their skill directory so each skill can be installed separately.

## Research basis

Refreshed on **2026-10-12** from current official documentation and selected repositories on the
[Verus publications and projects list](https://verus-lang.github.io/verus/publications-and-projects/).
Selection favored Verus-team projects and peer-reviewed systems, security, and verification work.
This is a focused source review, not an exhaustive survey or a reproduction of the papers' experiments.

The main additions are current mutable-borrow and iterator contracts, experimental logical atomicity,
compositional proof interfaces, crash consistency, liveness assumptions, and evidence-based proof diagnostics.
The update also corrects the old description of `GhostVarAuth`, trigger groups, and invariant restrictions.

### Inspected repository snapshots

The links below identify the reviewed revision. Detailed references link to the exact supporting files.
The venue identifies the associated publication; current branch code can postdate its evaluated artifact.

| Source | Revision | Evidence used |
| --- | --- | --- |
| [Verus](https://github.com/verus-lang/verus/tree/4bfc2b4484efc35ed373372a35f69b45f2483ea5) | `4bfc2b4484ef` | Official guide, migrations, vstd, and CLI; source through 2026-10-12 |
| [Verified allocator](https://github.com/verus-lang/verified-memory-allocator/tree/77d1fb6fc1907c903764863a479479a0a0ed04f5) | `77d1fb6fc190` | Verus-team project: storage permissions, identities, and protocol ownership |
| [Node replication](https://github.com/verus-lang/verified-node-replication/tree/6aed637f56043e508d3d187c4980431a7fce9373) | `6aed637f5604` | Verus-team project: layered history refinement and linearization |
| [NrOS page tables](https://github.com/matthias-brun/verified-nrkernel/tree/1c861090537664b78544cda5d53711b1fcfa05e2) | `1c8610905376` | Team-associated project: hardware/OS refinement and trusted boundaries |
| [PoWER / verified-storage](https://github.com/microsoft/verified-storage/tree/41cab95051229ebd457b4eed7fc111e3e7c9099c) | `41cab9505122` | OSDI 2025: crash-safe write capabilities and recovery views |
| [Anvil](https://github.com/anvil-verifier/anvil/tree/a597f9de7dbeb056403bd60b0a2ca67f686a5e1f) | `a597f9de7dbe` | OSDI 2024: executable step refinement and separate temporal assumptions |
| [Vest](https://github.com/secure-foundations/vest/tree/11af01ddaef76cceded03517cd48764adb9d35b1) | `11af01ddaef7` | USENIX Security 2025: current compositional parser and serializer proof traits |
| [Verdict](https://github.com/secure-foundations/verdict/tree/9bc18bc5a287c1608dcb74a6eee7b9ca94a62b67) | `9bc18bc5a287` | USENIX Security 2025: end-to-end contracts, errors, and nested views |
| [CortenMM artifact](https://github.com/TELOS-syslab/CortenMM-Artifact/tree/604592d815cb1cb7d4619f202b5d7ff471a3a7af) | `604592d815cb` | SOSP 2025: sampled representation-model helpers |
| [Mariposa / Cazamariposas code](https://github.com/secure-foundations/mariposa/tree/8646e7b1c601f7154e93d85ebb1083ce1ed47f95) | `8646e7b1c601` | Repository linked for CADE 2025: proof stability diagnostics |
| [ExVerus](https://github.com/claudeyj/exverus/tree/c1470dca3535b0fdd4d885e63bc58776a7308806) | `c1470dca3535` | ICML 2026: counterexample validation and invariant repair |

### Additional Verus-team papers

- [Tunable Automation, FMCAD 2026](https://doi.org/10.34727/2026/isbn.978-3-85448-093-8_38): configurable broadcast groups and measured context reduction.
- [VerusBelt, PLDI 2026](https://iris-project.org/pdfs/2026-pldi-verusbelt.pdf): semantic foundations and the limits of the soundness claim.
- [ProofPlumber, CAV 2024](https://www.andrew.cmu.edu/user/bparno/papers/proof-plumber.pdf): source-level assertions as diagnostic probes.

The update distinguishes documented language behavior from deductions drawn from project patterns.
It does not treat an arXiv listing or a future conference date as evidence of an already published peer-reviewed result.

## Validation

The repository contains skill documentation, with no existing build or test configuration.
For this refresh:

- All four skills pass the skill-creator metadata validator.
- Markdown lint passes with the line-length rule disabled, matching the existing long-line format.
- Local links and pinned GitHub file paths are checked against the inspected checkouts.
- All four Rust code blocks verify with `--no-cheating` on Verus `0.2026.10.04.426d8b0`.
- Four negative controls fail as expected: wrong borrow framing, a missing loop relation, missing solver inputs, and mismatched token IDs.
- The linked official logical-atomicity example also verifies on that release.

These checks validate the examples and document structure, not the behavior of an agent on every future task.
The project repositories were inspected, not rebuilt in full.
The supplied instructions refer to `docs/coding-style.md`, but that file is absent from this checkout.
