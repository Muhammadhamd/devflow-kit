---
name: audit-codebase
description: Scan the codebase and docs for gaps, bugs, security holes, performance and concurrency risks, missing tests, dead code, and stale docs; propose the findings; then (on confirmation) file them as issues via create-issue. Use when the user says "audit the codebase", "find work", "what needs fixing", "review the code for issues", or wants a backlog generated from the current state. Reads .claude/devflow.md; proposes before filing.
---

# Audit codebase → findings → issues

Turn the current state of a project into a **prioritized, evidence-backed backlog** — without spamming. Read `.claude/devflow.md` first (stack, conventions, security rules, base branch, KB index issue).

## Lenses (scan each; cite `file:line`)
- **Correctness / bugs** — logic errors, unhandled edges, wrong assumptions.
- **Security & owner-scoping** — unauthenticated side-effects, IDs trusted without owner checks, secrets in code/logs, injection, SSRF, missing authz.
- **Performance** — N+1 queries, unbounded loops/fetches, missing indexes, blocking work on hot paths.
- **Concurrency / idempotency** — check-then-act races, non-atomic claims, retries that double-charge/duplicate, fire-and-forget with no durability.
- **Test coverage** — critical paths (money, auth, data-loss) with no negative/invariant tests.
- **Docs staleness** — docs that contradict the code; undocumented behavior.
- **Dead code / tech-debt** — unreachable code, duplicate pipelines, stale wrappers, TODOs.

## Procedure
1. **Scope** with the user (whole repo vs a subsystem). For a large repo, optionally **fan out read-only subagents — one per lens** — and collect their findings (each returns `finding · file:line · impact · suggested fix`).
2. **Dedup** against existing issues (`gh issue list --search`) so you don't re-file known work.
3. **Rank** findings by severity (data-loss/security/money first) and group by topic/feature.
4. **PROPOSE to the user** — a concise table (finding · severity · file:line · suggested issue title). **Never mass-file unconfirmed.** Let them cut/merge/keep.
5. On confirm, for each kept finding call **`create-issue`** (self-contained issue + architect design + KB doc), grouped, each `Part of #<kb-index>`. Update the KB index issue with the new numbers.
6. Report the filed issue URLs + what was intentionally NOT filed (and why).

## Right-sizing
- A finding must be **real and evidence-backed** — no vague "could be better."
- Prefer a few high-value issues over a wall of nitpicks; roll trivial cleanups into one "housekeeping" issue.
- Verify each claim in code before proposing it; distinguish `verified` from `suspected`.

## Anti-patterns
- ❌ Filing 40 guessed issues without the user's review. ❌ Findings with no `file:line`. ❌ Re-filing something already tracked. ❌ Style-only nitpicks dressed up as issues.
