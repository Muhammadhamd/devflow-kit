---
name: create-issue
description: Turn a requirement, idea, bug, or audit finding into a self-contained GitHub issue with an architect-level design comment and a linked knowledge-base doc. Use when the user says "create an issue for X", "file this", "turn this into an issue", "we should build/fix Y", or when audit-codebase hands off findings. Reads .claude/devflow.md for repo, labels, and KB location.
---

# Create issue (+ architect design + KB doc)

Produce a **self-contained issue** an implementer can act on from its text alone, an **architect-design comment** deep enough to build from, and a **KB doc** that captures the durable "why." Read `.claude/devflow.md` first for the repo, labels, and `docs/kb/` location.

## Rules
1. **Dedup first:** `gh issue list --state open --search "<keywords>"`. If it exists, comment/update instead of creating.
2. **One problem per issue.** Independent sub-problems → a parent (EPIC) + children, linked.
3. **Self-contained + evidence-backed:** every issue carries its own `file:line` evidence, impact, scope, and acceptance criteria.
4. **Never leak secrets** into issues/KB (describe the location + fix, never the value).
5. **Body = WHAT/WHY (stable); the design comment = HOW (opinionated).** Keep them separate.
6. **Right-size the depth.** A chore gets a short issue + short comment; anything with data/concurrency/security/money/cross-service impact gets the full design (below).

## Procedure
1. Draft the **issue body** to a file using `.claude/devflow-templates/issue.md`, then:
   ```bash
   gh issue create --title "<imperative title>" --body-file <body.md> --label <labels from profile>
   ```
   Include in the body: `KB: docs/kb/<topic>.md` and `Part of #<kb-index>` (the Knowledge Base index issue from the profile).
2. Produce the **architect-design comment**. For any non-trivial change, delegate the design to the **`architect`** agent (give it the requirement + the issue number + the profile); it returns the proposal in the section format below. For a chore, write a short design inline. Then attach it:
   ```bash
   gh issue comment <n> --body-file <design.md>
   ```
3. Create/append the **KB doc** at `docs/kb/<topic>.md` from `.claude/devflow-templates/kb-doc.md` (several issues may share one doc — append, don't overwrite). Add the new issue to the KB index issue's map.
4. Report the issue URL and what it covers.

## Architect-design comment — sections (scale to the change)
`## Architect design` →
- **Context & goals** — what we optimize for (SLOs, scale, non-goals).
- **Options** — a comparison **table** across the dimensions that differ; then the decision.
- **Recommended design** — topology (ASCII/mermaid) · **interface/code sketch** (the real seam) · **migration SQL** for any schema change · numbered **sequence** (happy + failure + crash) · **idempotency/consistency** argument with the concrete key · **config** keys.
- **Failure-modes matrix** — failure → detection → recovery.
- **Observability** — metrics + alerts.
- **Security & data** — authz on new surfaces; what data lives where.
- **Backwards compatibility** — feature flag / cutover (name it, default off).
- **Rollout** — phases, canary, kill switch.
- **Rollback** — ideally a flag flip, no redeploy.
- **Testing** — unit/integration + a **chaos/kill test that proves the invariant**.
- **What stays the same** · **Effort** (breakdown) · **Open decisions** (each with a recommendation).

## Depth checklist (non-trivial change — if any is missing it's "too basic")
- [ ] Interface/code sketch (the actual seam), not a description
- [ ] Migration SQL for every schema change
- [ ] Failure-modes matrix
- [ ] Idempotency/consistency argument with the concrete key
- [ ] Observability (named metrics + alerts)
- [ ] Rollout AND rollback (a flag flip)
- [ ] A chaos/kill test proving the invariant

## Anti-patterns
- ❌ "See the doc" with no self-contained detail. ❌ Ten issues that are one problem (or vice-versa). ❌ Secrets/tokens in text. ❌ Creating before searching. ❌ Claiming it's filed without the `gh`-returned URL.
