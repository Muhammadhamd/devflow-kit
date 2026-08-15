---
name: write-tests
description: Author test cases for an issue or PR — one or more tests per acceptance criterion, including the negative/race/idempotency/security-bypass paths — using the project's own test framework and command. Runs the automated tests and reports exact results; leaves live-provider and browser tests to the human. Use when the user says "write tests for #N", "add test cases", "cover this with tests", or when implement-issue/review-pr flags missing tests. Reads .claude/devflow.md for the test command + framework.
---

# Write tests (to the acceptance criteria)

Turn an issue's **acceptance criteria** (and the design's invariant) into real tests in the project's own framework. Read `.claude/devflow.md` for the test command, framework, and file conventions.

## Principle
For each requirement, add **at least one test that would FAIL before the change** — cover the negative path, not only success. The most important test is the one that proves the design's **invariant** (the CAS race, the double-charge guard, the owner-scope check).

## Procedure
1. **Load the contract:** `gh issue view <N> --comments` — the acceptance criteria + the design's Testing section (it usually names the chaos/kill/invariant test).
2. **Locate the framework:** from the profile + the existing test dir; match the project's naming, structure, and helpers. Don't introduce a new framework.
3. **Write tests**, mapped 1-to-1+ to criteria. Include, where relevant:
   - **Negative** — the failure/validation path returns the right error.
   - **Race / concurrency** — two concurrent operations → the invariant holds (e.g. `Promise.all` on the same row → exactly one effect).
   - **Idempotency** — running the handler twice with the same key → one side effect.
   - **Security-bypass** — a foreign/owner-mismatched ID, missing auth, or malformed input is rejected (fail-closed).
4. **Run them** with the profile's command; report **exact** pass/fail/skip. If a test can't run without real credentials/provider, mark it and leave it for the human.
5. **Coverage table** in your report:

   | Acceptance criterion | Test (file::name) | Kind | Ran? |
   |---|---|---|---|

## Boundary
- **You run:** unit / service / invariant / negative / race tests, plus build/lint if relevant.
- **Human runs:** live-provider, browser/mobile, and anything needing real secrets or human judgment — list these explicitly as pending.

## Anti-patterns
- ❌ Only happy-path tests. ❌ Tests that pass on the unchanged code (they must fail before the fix). ❌ A new test framework/style that doesn't match the repo. ❌ Claiming coverage of the invariant without the actual race/idempotency test. ❌ Reporting "all pass" when some were skipped for missing credentials.
