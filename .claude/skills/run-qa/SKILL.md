---
name: run-qa
description: Verify merged-but-undeployed changes by actually running them — the project's test suite, the build, and Playwright specs driving the real app — then record a QA report on each issue's KB doc and comment `## QA` on the issue when something fails. Use before promoting to production, or when the user says "QA this", "verify what changed", "test before deploy". Delegates to the `qa` agent, the only agent permitted to execute tests.
---

# Run QA (execute, then report what actually happened)

The workflow has two agents that never observe the software: `developer` marks its own homework,
`reviewer` is forbidden from running anything. This skill closes that gap — and its whole value is
that its verdict is **grounded in execution**, so it must never claim more than it ran.

Read `.claude/devflow.md` for commands, base branch, app-start, and the QA section.

## Hard rules
1. **Never report a check you didn't run.** Not-run is `not covered`, with a reason. Errored is `FAIL`.
2. **Every `pass` carries evidence** — the command and its real output. "Looks correct" is not a result.
3. **Never handle a credential in plaintext.** Specs reference `process.env.QA_*`; values live in a
   gitignored file. If they're absent, name the variables and stop. Never accept a password in chat.
4. **Never weaken a failing test to get green.** If the test is wrong, say so and why.
5. **Specs must pass twice from a clean start.** A flaky spec trains people to ignore red.

## Procedure

### 1. Establish what changed
```bash
cat docs/qa/pending-qa.md                 # the register `developer` maintains
git log --oneline <production>..<base>    # what's merged and undeployed
git diff <production>...<base> --stat
```
Read each referenced `docs/kb/<topic>.md` for the *why*. The register's `verify` list is a **floor,
not a ceiling** — if the diff touches a surface the entry doesn't mention, verify it and note the gap.
An entry that claims a surface the diff doesn't touch is also a finding.

### 2. Baseline, then run
Run the suite **before** judging anything: a pre-existing failure is not this change's fault, and
attributing it here buries the real signal. Then:
```bash
<test command>      # real numbers, both totals
<build command>
<lint command>      # if configured
```

### 3. Drive the app
Start it per the profile. **If it won't start, that's a FAIL on the whole batch** — report and stop;
there is nothing to verify.

For each `verify` item with a user-visible surface, write or extend a **Playwright spec**:
- Named for the behaviour, not the issue — `publish-reflects-image-toggle.spec.js` outlives `issue-55.spec.js`.
- Asserts the **observable outcome**, not the implementation.
- No fixed sleeps where a wait-for works; no dependence on leftover state.
- Commit it. These accumulate into the regression suite the project didn't have.

### 4. Record — two places, one truth
- Append `## QA report` to each issue's `docs/kb/<topic>.md` (shape: `.claude/devflow-templates/qa-report.md`).
  **Append, never overwrite** — a re-run adds a dated block so the history of what was checked survives.
- Update each register entry: `status: passed | failed`.
- **On any failure**, comment on the **originating issue** (the one the PR closed):
  ```
  ## QA
  <failing behaviour · observed vs expected · exact reproduction · what it blocks>
  ```
  Failures belong where the work is tracked, not only in a doc.

### 5. Hand over the boundary
Return `passed` · `passed with exceptions` · `failed`, plus the **not covered** list — the section
that stops a green report being read as "fully verified". It typically includes, and these are not
your failures:
- Real-credential flows (live OAuth, a real payment, publishing to a third party)
- Multi-tenant checks needing a second real account
- True concurrency — N requests from **separate processes**; `Promise.all` in one process proves
  nothing about a database-level race
- Anything whose failure mode is data loss on real data

## Delegation
Spawn the **`qa`** agent for the run; synthesise its findings into the reports and comments. Never
accept the register's claims in place of the diff, and never accept "the tests look right" in place
of running them.

## Anti-patterns
- ❌ Reporting the suite as passing because you read it. ❌ Marking `passed` on a check you skipped.
- ❌ Prompting for a password, or writing one into a spec.
- ❌ Deleting or `.skip`-ing a failing test to reach green.
- ❌ A report with no `not covered` section — there is essentially always something.
- ❌ Specs named after issue numbers, which nobody can interpret a year later.
