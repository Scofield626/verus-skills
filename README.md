# Verus Skills

Four agent skills for writing, repairing, and maintaining Verus proofs:

- [verus-proof-writing](verus-proof-writing/SKILL.md): specifications, invariants, mutable borrows, refinement, and new proofs.
- [verus-proof-repair](verus-proof-repair/SKILL.md): failed obligations, proof regressions, and solver performance.
- [verus-style](verus-style/SKILL.md): proof interfaces, automation scope, ownership, and trusted boundaries.
- [verus-concurrency](verus-concurrency/SKILL.md): VerusSync protocols, permissions, invariants, and verified threading.

Each skill keeps its main workflow in `SKILL.md`. Read its linked references only when their subject applies.
References stay inside their skill directory so each skill can be installed separately.

## Sources

The skills draw on official documentation and projects from the
[Verus publications and projects list](https://verus-lang.github.io/verus/publications-and-projects/).
References link to exact source revisions, which can differ from published artifacts.

| Source | Guidance |
| --- | --- |
| [Verus](https://github.com/verus-lang/verus/tree/4bfc2b4484efc35ed373372a35f69b45f2483ea5) | Language, vstd, and diagnostics |
| [Verified allocator](https://github.com/verus-lang/verified-memory-allocator/tree/77d1fb6fc1907c903764863a479479a0a0ed04f5) | Storage permissions and protocol ownership |
| [Node replication](https://github.com/verus-lang/verified-node-replication/tree/6aed637f56043e508d3d187c4980431a7fce9373) | History refinement and linearization |
| [NrOS page tables](https://github.com/matthias-brun/verified-nrkernel/tree/1c861090537664b78544cda5d53711b1fcfa05e2) | Hardware refinement and trusted boundaries |
| [PoWER / verified-storage](https://github.com/microsoft/verified-storage/tree/41cab95051229ebd457b4eed7fc111e3e7c9099c) | Crash-safe writes and recovery |
| [Anvil](https://github.com/anvil-verifier/anvil/tree/a597f9de7dbeb056403bd60b0a2ca67f686a5e1f) | Step refinement and liveness assumptions |
| [Vest](https://github.com/secure-foundations/vest/tree/11af01ddaef76cceded03517cd48764adb9d35b1) | Parser and serializer proof interfaces |
| [Verdict](https://github.com/secure-foundations/verdict/tree/9bc18bc5a287c1608dcb74a6eee7b9ca94a62b67) | End-to-end contracts and failure cases |
| [CortenMM artifact](https://github.com/TELOS-syslab/CortenMM-Artifact/tree/604592d815cb1cb7d4619f202b5d7ff471a3a7af) | Representation helpers, including unfinished proofs |
| [Mariposa / Cazamariposas code](https://github.com/secure-foundations/mariposa/tree/8646e7b1c601f7154e93d85ebb1083ce1ed47f95) | Proof stability diagnostics |
| [ExVerus](https://github.com/claudeyj/exverus/tree/c1470dca3535b0fdd4d885e63bc58776a7308806) | Counterexample validation and invariant repair |

### Papers

- [Tunable Automation, FMCAD 2026](https://doi.org/10.34727/2026/isbn.978-3-85448-093-8_38): configurable broadcast groups and measured context reduction.
- [VerusBelt, PLDI 2026](https://iris-project.org/pdfs/2026-pldi-verusbelt.pdf): semantic foundations and the limits of the soundness claim.
- [ProofPlumber, CAV 2024](https://www.andrew.cmu.edu/user/bparno/papers/proof-plumber.pdf): source-level assertions as diagnostic probes.
