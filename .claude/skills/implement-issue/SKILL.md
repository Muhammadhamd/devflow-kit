---
name: implement-issue
description: Take a tracked GitHub issue (acceptance criteria + architect-design comment) and deliver it as a reviewable Pull Request against the project's configured base branch. One issue → one PR. Use when the user says "implement issue #N", "build #N", "start a PR for #N". Reads .claude/devflow.md for the base branch, commands, and conventions; delegates substantial work to the `developer` agent.
---

# Implement issue → PR

Turn a filed issue into a **single reviewable PR** against the branch this project targets. **Read `.claude/devflow.md` first** — the base branch, install/test/build commands, and conventions all come from it. Never hardcode a branch or stack.

## Hard rules
- **Base branch = the profile's base branch.** Branch off it; open the PR with `--base <that branch>`. Never target a different branch or commit to it directly without asking.
- **One issue per PR.** Small and reviewable. Respect any EPIC dependency order.
- **Never merge.** The `review-pr` skill + a human do that.
- **Testing boundary:** you write and run automated + invariant tests; the chaos/live/browser checks are the human handoff — list them, don't claim they passed.
- **Follow the issue's design, reconciled with current code** (it may be stale — flag drift before coding).

## Procedure
### 0. Preconditions
```bash
cat .claude/devflow.md                 # base branch, commands, conventions
gh auth status ; git fetch origin
git rev-parse --verify origin/<base>   # base branch must exist
git status --short                      # clean, or only intended changes
```

### 1. Load the issue as the contract
```bash
gh issue view <N> --comments
```
Extract **Acceptance criteria (the contract)** and the **`## Architect design` comment (the plan)**. Read its linked `KB: docs/kb/<topic>.md` for context. If criteria/design are missing, write them (or run `create-issue`) first.

### 2. Plan & gate
Trace the real flow end-to-end in code. Do a proportional **security/cost gate** (auth, owner-scoping, idempotency, secrets, atomic credit/quota if the project has paid ops — see the profile). Write a short TODO mapping each acceptance criterion → change → proof.

### 3. Branch off the base
```bash
git switch -c <type>/issue-<N>-<slug> origin/<base>   # type ∈ feat|fix|refactor|chore
```

### 4. Implement through the project's existing architecture
Follow the profile's conventions. Schema change ⇒ **both** the ORM model **and** a migration. Owner-scope every non-admin query. Honor the design's **feature flag** (default off) and its **"What stays the same."** Put invariants (CAS, jobId, owner-scope) **in code, not prompts.**

### 5. Tests (to the invariant)
Add at least one test that fails before the change, including the **negative/race/idempotency** test the design names. Run the profile's test + build (+ lint). Report exact pass/fail/skip. Do NOT claim the chaos/live/browser checks passed.

### 6. Docs
Update the relevant docs + the KB doc for what actually shipped (per the project's doc conventions).

### 7. Commit, push, open the PR against the base branch
Commit (with the profile's trailer if any), push, then:
```bash
gh pr create --base <base branch> --head <branch> --title "<title> (#N)" --body-file <pr_body.md>
```
Build the PR body from `.claude/devflow-templates/pr.md`: `Closes #N`; summary + approach; **acceptance-criteria checklist mapped to code + test**; **flag/rollout**; **human-testing required** (the chaos/live/browser checks you did NOT run); docs updated; **rollback**; residual risks.

### 8. Link back on the issue
```bash
gh issue comment <N> --body "PR opened against <base>: <url>. Automated tests: <result>. Pending human testing: <list>."
```

### 9. Register the change for QA
The `qa` agent can only verify what it knows changed, and it reads
**`docs/qa/pending-qa.md`** to find out. Add the entry when the PR opens; the merger confirms it.
Shape: `.claude/devflow-templates/qa-report.md`.

```markdown
### #<issue> — <short title>   (PR #<pr>, merged <YYYY-MM-DD>)
- **status:** pending
- **surfaces:** <routes / pages / jobs a user can reach. "none (internal)" is valid and still gets an entry>
- **risk:** <one line: what breaks if this is wrong>
- **verify:**
  - <a behaviour whose failure is unambiguous>
- **cannot be automated:** <live-provider / real-credential / multi-process checks — or `none`>
- **KB:** `docs/kb/<topic>.md`
```

Two things make this entry worth writing rather than boilerplate:

- **`verify` must be behaviours, not areas.** "Toggling an image then publishing within the save
  window reflects the toggle" is checkable; "test the image picker" is not. You just built it — you
  know the edge that matters, and nobody downstream does.
- **`cannot be automated` is required**, and `none` is a legitimate answer. It is the honest
  boundary of what QA will be able to claim, and it becomes the human's checklist. Under-declaring
  it produces a green report that nobody should have trusted.

An entry with no `verify` items is a claim that nothing observable changed. Sometimes true —
say so explicitly rather than leaving it blank.

## Delegation
For a substantial issue, spawn the **`developer`** agent with the frozen plan + the profile, then **independently audit the diff** (don't trust its summary) before opening the PR. If subagents are unavailable, keep the same separation sequentially.

## Anti-patterns
- ❌ Targeting a branch other than the profile's base, or committing straight to it.
- ❌ Bundling multiple issues / unrelated cleanup into one PR.
- ❌ "Done / all tests pass" when only automated tests ran and chaos/live are human-pending.
- ❌ Skipping the QA register entry, or filling it with "test the feature". QA verifies what you tell it changed — a vague entry buys a vague verdict.
- ❌ Declaring `cannot be automated: none` to look clean. Under-declaring it is how a green report gets trusted for something nobody checked.
- ❌ Implementing the design without reconciling it against current code.
- ❌ Enforcing an invariant in a prompt that the design says to put in code.
