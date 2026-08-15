---
name: create-capability
description: Author a NEW agent or skill, personalized to THIS project, when a workflow needs a role or capability the kit doesn't already have. Uses three inputs — the knowledge base, the user's requirement, and the codebase/profile — to write a correct, project-specific `.claude/agents/<name>.md` or `.claude/skills/<name>/SKILL.md`, then registers it with the orchestrator. Use when the user asks for a capability that no existing agent/skill covers ("we need a data-migration agent", "make a skill for release notes", "use the X agent" when X doesn't exist yet). Reads .claude/devflow.md.
---

# Create a capability (agent or skill) — personalized to this project

This is the kit's **meta-skill**: it lets DevFlow *grow itself*. Claude Code already lets anyone create an agent or skill — but only if the human knows how to prompt it into being project-specific. **This skill captures that know-how**: given a need, it authors a new agent/skill that already knows this codebase, follows the kit's conventions, and slots into the loop — no expert prompting required.

Use it when a requested workflow references a role/capability that **doesn't exist yet** (e.g. "use the proposal architect agent", "run the data-migration agent", "make a release-notes skill"). If an existing agent/skill already covers it, **use that instead — don't duplicate.**

## Decide first: agent or skill? (and does it already exist?)
1. **Check the toolbox** — list `.claude/agents/` and `.claude/skills/`. If something covers the need, stop and use it. The base set: agents `architect`, `developer`, `reviewer`; skills `create-issue`, `implement-issue`, `review-pr`, `audit-codebase`, `write-tests`, `create-capability`.
2. **Agent vs skill:**
   - **Agent** = an autonomous *actor/role* with a bounded tool set that does work and reports back (implementer, auditor, migrator, designer). Reusable "team member."
   - **Skill** = a *procedure/workflow* Claude follows in the main thread (a repeatable multi-step recipe: create-issue, cut-a-release, triage-a-bug).
   - Rule of thumb: a distinct **persona with its own least-privilege tools** → agent. A **repeatable set of steps** the main assistant runs → skill. When both, a skill often *orchestrates* one or more agents.

## The three inputs (this is what makes it personalized — use all three)
1. **The requirement** — what the new capability must do, in the user's words. Pin its scope and its definition of done.
2. **The codebase + profile** — read `.claude/devflow.md` and the relevant code so the capability carries **real** facts: the stack, base branch, schema-change rule, security/cost invariants, danger zones, test/build commands, the patterns to reuse. Bake these **inline**, don't leave them generic.
3. **The knowledge base** — read the pinned 📚 KB index issue and the `docs/kb/<topic>.md` docs relevant to this capability, so it inherits the project's durable context and vocabulary.

If any input is thin (vague requirement, empty repo, no KB yet), say so and bake only what you can confirm — mark the rest as an explicit TODO inside the new file rather than inventing it.

## Procedure
### 1. Draft the spec (propose before writing)
State back: **name** (kebab-case), **agent or skill**, **one-line purpose**, **the tools it needs** (least privilege — see below), **the project invariants it must uphold**, and **how it plugs into the loop** (who calls it, what it hands to next). Get a quick yes/edits. **Don't create files unconfirmed.**

### 2. Author the file
- **Agent** → `.claude/agents/<name>.md` with frontmatter `name`, `description` (say when to use it), and `tools` scoped to **least privilege** (a reviewer/auditor gets read-only `Read, Grep, Glob, Bash`; an implementer gets `"*"`; a designer gets read-only + web). Body: role identity, "read `.claude/devflow.md` first", a **`## This project (baked context)`** block with the real invariants/commands/danger zones, its procedure, its boundaries, and a **structured report** format. Never let it claim "done" without evidence; state the testing boundary (agents run automated tests, humans run live/browser/chaos).
- **Skill** → `.claude/skills/<name>/SKILL.md` with frontmatter `name` + `description` (with trigger phrases). Body: reads the profile first; a clear numbered procedure; the baked project specifics; guardrails; and how it hands off to agents/other skills.
- **Model the base files.** Match the voice, structure, and discipline of the existing `architect`/`developer`/`reviewer` agents and the existing skills. Reuse their conventions (base-branch rule, one-issue-per-PR, invariant-in-code-not-prompt, propose-before-side-effects, no secrets).

### 3. Uphold the kit's non-negotiables (bake into every capability)
- Reads `.claude/devflow.md`; never hardcodes a stack/branch that belongs in the profile.
- PRs target the profile's **base branch**; never merge autonomously.
- Owner-scope queries; make paid ops idempotent/atomic; **never log or write secrets**.
- Schema change ⇒ model **+** migration.
- Respects the danger zones (won't touch what the profile says not to).
- **Testing boundary** stated: automated tests are the agent's; live/browser/chaos + merge are the human's.

### 4. Register it (so the orchestrator knows it exists)
- Add it to the **`devflow`** orchestrator's routing table + "The pieces" list, so `/devflow` can route to it.
- Add a row to the **profile's** capability list in `.claude/devflow.md` (and the README "What's inside" table if the user maintains one).
- If it belongs in a workflow chain (e.g. an architect step before implement), note the hand-off in `devflow`.

### 5. Hand off
Tell the user the capability exists, how to invoke it, and where it sits in the loop. If it was created mid-workflow (the user's request needed it), continue that workflow using the new capability.

## Example (the user's own scenario)
> "…use the proposal architect-level expert agent to create a proposal, then ask the reviewer agent to validate…"

If `architect` exists → use it. If it didn't, this skill would: confirm it's an **agent** (a designer persona, read-only tools + web), author `.claude/agents/architect.md` with this project's schema rule + security invariants baked in, register it in `devflow`, then resume the chain: `create-issue` → `architect` (proposal) → `reviewer` (validate) → `developer` (PR) → human merge.

## Guardrails
- **Reuse over create** — one capability per real need; never spawn a near-duplicate of an existing agent/skill.
- **Propose before writing files**; least-privilege tools always.
- **Personalized, not generic** — if the new file could have been written for any project, you didn't use the three inputs. Put real `file:line`-level project facts in it.
- Never bake a secret into a capability; describe + remediate instead.
