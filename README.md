# Verus Skills

This folder packages the repo's Verus guides as agent-usable `SKILL.md` files with YAML frontmatter.

Skills:

- `verus-proof-writing/`: use when writing new Verus specs, invariants, lemmas, or ghost-state structure from scratch.
- `verus-proof-repair/`: use when a Verus proof or verification obligation is failing and the goal is to make a minimal, proof-focused fix.
- `verus-style/`: use when reviewing or editing Verus code for maintainability, module organization, naming, trigger discipline, and `vstd` usage.
- `verus-concurrency/`: use when working on `tokenized_state_machine!`, `AtomicInvariant`, `atomic_with_ghost!`, `GhostVar` / `GhostVarAuth`, shared-state protocols, or verified threading.

The skills are intentionally concise. Their frontmatter is the trigger surface; the body is the procedural workflow the agent should follow after the skill triggers.
# verus-skills
