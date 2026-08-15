---
name: review-pr
description: Review a Pull Request against its source issue's acceptance criteria and a quality/security rubric, then post inline PR comments, a PR verdict, and a "required changes" comment on the issue. Static/criteria review only — a human runs the tests. Use when the user says "review PR #N", "review the PR for issue #N", "check this PR". Reads .claude/devflow.md for the base branch; delegates to the `reviewer` agent.
---

# Review PR (criteria audit; humans test)

Audit a PR against the issue it implements. You verify **intent-correctness, criteria coverage, security, and that meaningful tests exist** — you do **not** execute tests, run the app, or claim pass/fail. A human runs the tests and merges. Read `.claude/devflow.md` for the project's base branch and conventions.

## Hard rules
- **Read-only on code.** You comment; you never edit the PR's code.
- **Do NOT run tests, the app, or migrations.** Read the tests to judge they *exist and cover the invariant*; report execution as "human-pending," never as passed.
- **Base branch must equal the profile's base branch.** If the PR targets a different branch → automatic "changes requested."
- **Comment on BOTH the PR and the issue.** Inline/line comments + a verdict on the PR; a concise numbered "Required changes" comment on the ISSUE (issues track the change requests).
- **No rubber-stamp.** Strongest verdict = *"correct on review — pending human testing of X/Y/Z."*

## Procedure
### 0. Load both sides
```bash
cat .claude/devflow.md
gh issue view <ISSUE> --comments        # Problem, Evidence, Acceptance criteria (contract), Architect design
gh pr view <PR> --json title,baseRefName,headRefName,files,additions,deletions,body,url
gh pr diff <PR>
```
Confirm the PR links the issue (`Closes #<ISSUE>`) and `baseRefName` == the profile's base branch.

### 1. Criteria-coverage table
For every acceptance criterion, find the code + the test in the diff:

| Acceptance criterion | Code (file:line) | Test (file:line) | Status |
|---|---|---|---|

`Status ∈ met / partial / missing / not-testable-by-code`. Any `missing`/`partial` is a required change.

### 2. Rubric (check each; cite file:line)
- **Base branch** == profile base branch (else auto changes-requested).
- **Design fidelity** — implements the issue's recommended approach; deviations justified in the PR, not silent.
- **Scope** — one issue; no unrelated changes; the design's "What stays the same" is untouched.
- **Invariant in code, not prompt** — the concrete mechanism the design names is present (CAS / dedup key / owner-scoped query / atomic reservation). A comment/prompt is not enforcement.
- **Security & data** — owner-scoped IDs, auth at every entry/side-effect, atomic credit/quota where applicable, retries idempotent (no double-charge/effect), fail-closed on security/paid checks, **no secrets/PII in code or logs**.
- **Schema** — any schema change has **both** the ORM model **and** a migration.
- **Feature flag / rollback** — flag present + default off where the design requires; a rollback path exists.
- **Tests exist & are meaningful** — the negative/race/idempotency test the design names is present and would fail before the change. (Judge existence + intent; a human runs them.)
- **Docs** — updated for what changed.
- **Backwards compatibility** — preserved unless the issue required removal.

### 3. What a human must test (state explicitly)
List everything you did NOT execute: the full/focused test run, the **chaos/kill test proving the invariant**, live-provider validation, browser/mobile flow. These are the human gate before merge.

### 4. Post the review
```bash
gh pr comment <PR> --body-file <review.md>       # criteria table + rubric findings + human-testing list + verdict
gh issue comment <ISSUE> --body-file <required_changes.md>   # numbered blockers + notable non-blockers
```
(Use `gh pr review <PR> --comment`/`--request-changes` for line-level comments where useful.)
**Verdict** (pick one, say why): `Changes requested` · `Correct on review — pending human testing of <list>` · `Comment only (non-blocking notes)`.

## Severity guide
- **Blocker:** wrong base branch · an acceptance criterion unmet · a security/ownership/idempotency hole · an invariant left prompt-only · a schema change missing its migration · a missing invariant test.
- **Non-blocker:** style, naming, doc gaps, extra hardening ideas — note, don't gate.

## Delegation
Spawn the **`reviewer`** agent (read-only) for the audit; synthesize its severity-ranked findings into the PR + issue comments. Never accept the PR author's summary in place of the diff.

## Anti-patterns
- ❌ "LGTM/approved" when you never ran the tests — say "pending human testing." ❌ Editing the PR's code to fix it yourself. ❌ Reporting a test as passing because it *looks* right. ❌ Reviewing only the happy path.
