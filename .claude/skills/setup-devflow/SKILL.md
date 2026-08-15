---
name: setup-devflow
description: First-run setup for DevFlow in a project. Analyzes the codebase, interviews the user (which branch PRs target, test/build commands, conventions), then GENERATES a personalized set of skills + agents with this project's real context baked in, writes the .claude/devflow.md profile, sets up git/GitHub (labels, base branch), and files the "📚 Knowledge Base" index issue plus initial issues extracted from the project's docs. Run once per project (idempotent — safe to re-run to reconfigure). Triggered by the kit's README bootstrap or "set up devflow".
---

# Setup DevFlow (first run)

**Your job is to turn a generic kit into a project-specific toolset.** You analyze THIS codebase, then rewrite the installed skills and agents so they carry this project's real architecture, conventions, and danger zones **inline** — the way a senior who knows the repo would have written them. The `.claude/devflow.md` profile is the machine-readable source of truth; the personalized skills are what make the agents *read* like they belong to this project.

Run once (re-running reconfigures — detect and update rather than duplicate). **You are configuring the user's project, not the kit repo.**

## 0. Preflight
- Confirm you're in the **target project** (its own git history / source), not the DevFlow kit. If unsure, ask.
- Confirm the skills/agents were copied in (`.claude/skills/`, `.claude/agents/`, `.claude/devflow-templates/`); if not, do the README bootstrap first.
- Check tooling:
  ```bash
  git rev-parse --is-inside-work-tree && git remote -v
  gh auth status        # needed to file issues; if missing, tell the user to `gh auth login` and pause
  ```

## 1. Auto-detect what you can (don't ask what you can read)
- **Stack:** `package.json` / `pyproject.toml` / `go.mod` / `Cargo.toml` / `composer.json` → language + framework; read scripts for test/build/lint/dev commands.
- **Repo:** `owner/name` from `git remote`. **Branches:** `git branch -a` (is there a `dev`/`develop`?).
- **Docs:** `README*`, `docs/`, `ROADMAP*`, `TODO*`, `docs/plans/`, ADRs, spec files, any `CLAUDE.md` / `SYSTEM_MAP.md` / project-instruction files.

## 2. Deep codebase analysis (the heart of setup — do NOT skip)
This is what makes the output personalized. Read enough of the codebase to write project-specific rules with confidence. Prefer a subagent fan-out (Explore/general-purpose) so you can read broadly, then synthesize. Extract:

- **Architecture & layering** — the real request/data flow (entry points → services → data layer → external calls); the folder layout; where business logic lives; the dominant patterns to reuse (not reinvent).
- **Persistence & schema** — the ORM/driver, where models live, **how a schema change is made** (e.g. "model + a migration file in `X/`"), and any migration-runs-at-boot / push convention. This becomes a hard rule in the developer/reviewer.
- **Security & cost model** — auth mechanism; the owner-scoping/tenant rule; any paid/metered/AI operations and how they're gated (idempotency, quota, atomic charge, charge-after-work); where secrets live and how they're loaded. Capture concrete invariants ("every non-admin query filters by userId"; "credit charge must be idempotent + after successful work").
- **Testing** — framework, the exact test/build/lint commands, where tests live, the shape of an existing test to mirror. **Also record what's *missing*:** is there a unit runner at all? an e2e/browser runner? a CI workflow that runs them? These gaps drive the QA interview in step 3.
- **Runnable surface** — how the app starts locally (dev command, port, health endpoint), whether it needs a database/queue up first, and what a logged-in session requires. QA can't drive an app it can't start.
- **Deployment** — is there a target at all (host, PaaS, container registry, CI/CD workflow)? Which branch is production? Any existing deploy script, and how a release currently reaches users. If you find none, say so — don't invent one.
- **Conventions** — commit/PR style, naming, error handling, the doc-sync rules if any, and any commit trailer.
- **Danger zones** — directories/files the agents must NOT touch (dead prototypes, vendored code, generated bundles), and known fragile areas.
- **Project-instruction files** — if a `CLAUDE.md`/equivalent exists, treat its rules as authoritative and fold them into the baked context.

Write the synthesis to a scratch note; you'll bake it into the profile AND the skills.

## 3. Interview (ask ONLY what you couldn't determine or must confirm)
One short batched question set, detected values pre-filled as the recommended option:
1. **Which branch should PRs target?** (REQUIRED — **no default**; offer detected branches + "create a new `dev`" + "target `main`"). The single most important answer.
2. **Test / build / lint / run commands** — confirm the detected ones or correct.
3. **Merge policy** — human merges after review + tests (recommended) vs auto.
4. **Anything the agents must NOT touch** — confirm the danger zones you found.
5. **Security/cost rule** — confirm the invariants you inferred (auth model, owner-scoping, paid ops) or correct them.

**QA** (the `qa` agent executes; nothing else in the workflow does):
6. **Missing test tooling — install it?** Report what you found and what's absent, then offer to add
   only what they approve. Never add a dependency silently.
   - No unit runner → offer the ecosystem-standard one (Jest/Vitest for JS, pytest, go test…), wired
     into a `test` script.
   - No browser/e2e runner → offer **Playwright**, plus a spec directory and an `e2e` script.
   - Already present → confirm the commands and move on. **Don't replace a working stack.**
7. **How does QA reach a logged-in app?** The dev command, the URL/port, and the **names** of the env
   vars holding QA account credentials (e.g. `QA_EMAIL` / `QA_PASSWORD`).
   > **Ask for variable NAMES, never values.** If the user offers a password, decline it and explain
   > that specs read `process.env.*` and the values belong in a gitignored file only they hold. A
   > credential pasted into chat is in the transcript forever.
8. **Is there a staging/test environment**, or does QA run against local only? QA must never run
   destructive checks against production data — if there's no safe target, record that limit.

**Deployment** (the `devops` agent):
9. **Do you want the deploy role at all?** If no, skip 10–11 and record it in the profile — the
   router should then not offer `devops-deploy`.
10. **Production branch + how a release reaches users** — the pipeline steps, the health endpoint to
    verify, and the rollback move.
11. **Connection details** — host, user, key path, registry, whatever the target needs.
    > These go in **`.claude/devflow.local.md`**, which you create **and gitignore**. Never in the
    > profile, an issue, a commit, or the transcript. Ask for a **key path**, never a key. If the
    > user pastes a secret, tell them it's now in the transcript and should be rotated.

Keep it to what changes behavior; state assumptions for the rest.

## 4. Generate the project profile
Fill `.claude/devflow-templates/project-profile.md` with the detected + answered + analyzed values and write it to **`.claude/devflow.md`**. Include the base branch, commands, conventions, the security/cost invariants, the schema-change rule, danger zones, and a KB-index pointer. Every other skill reads this. If it exists, update in place (preserve hand-edits you can't re-derive).

## 5. Personalize the skills & agents (bake this project's context in)
Now rewrite the installed generic files so they speak this project. For **each** skill in `.claude/skills/` and **each** agent in `.claude/agents/`:

- Keep the generic discipline/procedure intact (it's good), but **inject a `## This project (baked context)` section** near the top with the concrete, analyzed specifics that skill/agent needs — pulled from step 2. Replace vague phrasing ("follow the profile's conventions", "if the project has paid ops") with the **actual** rule when you know it.
  - `architect` agent + `create-issue` → the real schema-change rule, security/cost invariants, and the modules/patterns a design must route through, so proposals are grounded in this codebase.
  - `developer` agent + `implement-issue` → the real base branch, exact test/build/lint commands, the schema-change rule, the owner-scoping invariant, the credit/quota idempotency rule (if paid ops exist), the danger zones, the commit trailer. It should read like your existing project-tailored agents, not a template.
  - `reviewer` agent + `review-pr` → the same invariants as *blocker* checks stated concretely ("PR base must be `<branch>`"; "any schema change must include a migration in `<dir>`"; "every new non-admin query must filter by `<owner field>`").
  - `create-issue` / `audit-codebase` → this project's feature areas, doc locations, and label set; the security/perf lenses that actually apply here.
  - `write-tests` → the real framework + command + where tests live + the shape to mirror.
  - `qa` agent + `run-qa` → the exact test/build/e2e commands, **how to start the app** (command, URL, prerequisites), the spec directory, the env-var **names** for QA credentials, and the project's genuinely un-automatable checks (live provider, second tenant, multi-process concurrency) so the `not covered` list is concrete rather than generic.
  - `devops` agent + `devops-deploy` → the real production branch, promotion rule, pipeline steps, health endpoint, rollback move, and the backup-before-mutation rule. If the user declined the deploy role, **don't generate these** — and say so.
  - `create-capability` → the project facts a *newly-minted* agent/skill must inherit (invariants, danger zones, base branch), so anything the kit generates later is born personalized too.
- **Still keep `.claude/devflow.md` as the source of truth** for anything that may change (branch, commands): the baked sections should *reference and agree with* it, not contradict it. When in doubt, bake the durable architectural facts inline and point to the profile for the operational values.
- Optionally, if the project already has strong tailored agents (e.g. a `<project>-developer`), reconcile: either fold their good rules into the generated `developer`, or note the overlap for the user — don't silently leave two conflicting implementers.

**Show the user a short diff-summary** of what you baked into each file (a few bullets each) before finalizing, so they can correct a wrong inference cheaply.

## 6. Git / GitHub setup
- **Base branch:** if the user chose a new branch (e.g. `dev`), create from the current default and push after confirming: `git switch -c dev && git push -u origin dev`. Record it in the profile.
- **Labels** (idempotent): `kb`, `feature`, `bug`, `tech-debt`, `security`, `docs`.
  ```bash
  gh label create feature --color 1d76db 2>/dev/null || true   # (repeat per label)
  ```
- **Branch protection** (optional): suggest protecting the base branch (require PR + review) — describe how; don't force it.

## 6b. QA + deploy scaffolding
Only what the user approved in step 3.

- **`docs/qa/pending-qa.md`** — create it with the header and empty `## Awaiting QA` /
  `## Verified — awaiting deploy` sections (shape: `.claude/devflow-templates/qa-report.md`).
  `developer` appends on every PR; `qa` reads it; `devops` refuses to promote while anything is
  `pending` or `failed`.
- **Test tooling** (only if approved): install it, add the scripts, and add **one working example
  spec** that passes — an empty harness nobody has ever seen go green is not a harness. Confirm the
  commands run before recording them in the profile.
- **Playwright** (if approved): install, create the spec dir, add the `e2e` script, and write one
  smoke spec that loads the app and asserts something real. If the app needs auth, the spec reads
  `process.env.<the names from step 3>` — **never a literal**.
- **Credentials scaffold:** create `.env.qa.example` listing the variable **names** with empty
  values, and ensure `.env.qa` is gitignored. The user fills the real file; you never see it.
- **`.claude/devflow.local.md`** (if the deploy role was accepted): connection details, **gitignored**.
  Verify the ignore actually matches — a `.gitignore` entry that doesn't match is worse than none,
  because it looks handled.

```bash
grep -qxF '.env.qa' .gitignore || echo '.env.qa' >> .gitignore
grep -qxF '.claude/devflow.local.md' .gitignore || echo '.claude/devflow.local.md' >> .gitignore
git check-ignore -v .env.qa .claude/devflow.local.md    # prove it, don't assume it
```

## 7. Knowledge base + initial issues (PROPOSE, then file on confirm)
The KB is the project's durable memory: `docs/kb/<topic>.md` docs + a pinned index issue.
1. **Scan docs + code** for planned-but-unimplemented work, TODOs, and distinct feature areas → candidate list grouped by topic.
2. **Show the proposed list** (topics → KB docs → issues); get a yes / edits. **Never mass-file unconfirmed.**
3. On confirm:
   - Create `docs/kb/<topic>.md` per area (seed from source docs — the "why + context").
   - Create the **📚 Knowledge Base index issue** (label `kb`, pinned): a map linking every KB doc + child issue, grouped by area.
   - Create one issue per work item via `create-issue`, each with `KB: docs/kb/<topic>.md` and `Part of #<kb-index>`.
   - Edit the index issue to list the created numbers.

## 8. Verify + hand off
- Confirm: **codebase analyzed → skills/agents personalized** (name the files), profile written, base branch set, labels created, KB index + initial issues filed (URLs).
- Confirm the QA/deploy scaffolding: register created, tooling installed (naming what), example spec
  **run and passing**, ignore rules **proven** with `git check-ignore`.
- Tell the user, verbatim:
  > **Setup complete. Your skills and agents are now tailored to this codebase.** Start every new session with `/devflow` — it routes what you want to do (new feature, implement an issue, review a PR, QA, deploy). All PRs target `<base branch>`; you review + merge. Before anything reaches production, `devops-deploy` will ask whether to run QA first — and a QA failure blocks the promotion.
- If QA credentials are needed, tell them which variables to fill in `.env.qa` — **by name**, and
  note that you will never ask for the values.

## Guardrails
- The personalization step is the point — **never finish setup having only copied generic files.** If you couldn't analyze the codebase (empty repo, no access), say so and bake in only what you confirmed, marking the rest as TODO in each file.
- Idempotent: re-running re-analyzes and updates the baked sections/profile/labels/KB issue rather than duplicating.
- Propose-before-file for issues; keep the initial batch tight (the backlog grows later via `create-issue`/`audit-codebase`).
- Never write secrets into the profile, skills, issues, or KB docs — describe a leaked secret and its remediation; never reproduce it.
- **Never ask for a credential value.** Collect variable *names* and file *paths*. If the user volunteers a password or key, decline it, explain it's now in the transcript, and recommend rotating it. This holds even when they insist — the leak is the transcript, not your caution.
- **Never install a dependency the user didn't approve**, and never replace a working test stack with a preferred one.
- **Don't generate the deploy role if the user declined it**, and don't invent a pipeline you couldn't detect. Guessing at someone's infrastructure is the one setup error that isn't cheap to undo.
- An installed test harness with no passing example isn't set up — run it before you claim it.
