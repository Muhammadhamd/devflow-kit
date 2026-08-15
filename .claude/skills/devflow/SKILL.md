---
name: devflow
description: The DevFlow orchestrator and session entrypoint — the one skill that knows all the others. Run /devflow at the start of any session to see project state (open issues, base branch, knowledge base) and route your request to the right skill/agent. Use when the user says "/devflow", "start a devflow session", "what should I work on", "let's build/review something", or just opens the project and wants to work.
---

# DevFlow orchestrator

The **front door**. Every DevFlow session starts here. It reads the project profile + GitHub state, tells the user where things stand, and routes to the right skill or agent. It also keeps the knowledge base honest.

**If `.claude/devflow.md` doesn't exist yet, this project isn't set up — run `setup-devflow` first.**

## On invocation

1. **Load context**
   ```bash
   cat .claude/devflow.md                 # project profile: base branch, commands, repo, conventions
   gh issue list --state open --limit 40
   gh pr list --state open --limit 20
   ```
   Read the **📚 Knowledge Base index issue** (the `kb`-labelled issue named in the profile) for the map of docs ↔ issues.

2. **Brief the user** (short): project name, base branch PRs target, # open issues (by label), any open PRs awaiting review, and the top of the backlog. Don't dump everything — a 4–6 line status.

3. **Ask what they want**, then route:

| The user wants… | Route to | Notes |
|---|---|---|
| A new feature / requirement / captured idea | **`create-issue`** | Turns it into a self-contained issue + architect design + a KB doc. |
| A deep design / proposal before coding | **`architect`** agent | Options + trade-offs + implementation sketch + schema/security/rollout. `create-issue` calls it for the design; call it directly for "design/propose X". |
| To find work / audit quality | **`audit-codebase`** | Scans code+docs → proposes findings → files issues (on confirm). |
| To build a specific issue | **`implement-issue`** (uses the **`developer`** agent) | Issue → branch off the profile's base → PR. One issue per PR. |
| To review an open PR | **`review-pr`** (uses the **`reviewer`** agent) | Criteria + security audit → comments on PR + issue. |
| More/better tests for an issue or PR | **`write-tests`** | Maps acceptance criteria → tests. |
| To verify what's merged but not deployed | **`run-qa`** (uses the **`qa`** agent) | **Executes** the suite, the build, and Playwright specs against the running app. Writes a `## QA report` on each issue's KB doc; comments `## QA` on the issue when something fails. |
| To deploy / ship / promote to production | **`devops-deploy`** (uses the **`devops`** agent) | **Asks whether to run QA first.** A failed QA blocks the promotion. Then promote → deploy → verify health → close issues. |
| A capability that **doesn't exist yet** (a role/skill no current agent covers) | **`create-capability`** | Authors a new **personalized** agent/skill from the requirement + codebase + KB, registers it here, then resumes the workflow. |
| To just talk through a decision | Stay here, or capture it as a KB doc / ADR. |

**When a request names an agent/skill that doesn't exist** (e.g. "use the data-migration agent"), don't fail and don't fake it — route to **`create-capability`** to mint it (personalized), then continue. Reuse an existing capability if one already fits.

4. **After any skill runs**, update the **KB index issue** if issues were created/closed (keep the map current), and tell the user the next logical step.

## The loop this enforces

```
requirement/idea ─► create-issue ─► issue + KB doc (linked to the KB index)
                                         │
                              implement-issue ─► PR to <base branch>   (developer agent)
                                         │                └─ on merge: appends to docs/qa/pending-qa.md
                                         │
                                 review-pr ─► criteria/security audit    (reviewer agent — reads, never runs)
                                         │
                                    HUMAN ─► merges
                                         │
                                   run-qa ─► EXECUTES suite + build + Playwright   (qa agent)
                                         │      └─ ## QA report on the KB doc; ## QA comment on the issue if it fails
                                         │
                            devops-deploy ─► "run QA first?" → promote → deploy → health → close issues   (devops agent)
```

Every piece of work ends up as: an **issue** (why + acceptance criteria + design), a **KB doc**
(durable context + its QA report), a **PR** (the change, reviewable), a **review**, and a record of
**what was actually executed** — so months later you can search GitHub to find, change, or remove any
logic with its full rationale *and* know whether it was ever verified.

## Who executes what — the distinction that matters
- **`developer`** writes the code and marks its own homework.
- **`reviewer`** reads the code and is **forbidden** from running it — it never claims a test passed by reading it.
- **`qa`** is the **only** agent that executes. Its verdict is the only one grounded in observation, so it must never overstate: a check it didn't run is `not covered`, never `pass`.
- **You (human)** still own what QA structurally cannot: live-provider flows, real credentials, a second tenant, and true concurrency across **separate processes**. Every QA report ends with that list.

Without `qa`, nobody in the loop has ever seen the software behave — `developer` is not impartial and
`reviewer` is not allowed. That gap is what this role exists to close.

## The pieces (so the user knows the toolbox)
- **Skills:** `setup-devflow` (once), `devflow` (this — each session), `create-issue`, `implement-issue`, `review-pr`, `run-qa`, `devops-deploy`, `audit-codebase`, `write-tests`, `create-capability` (mint new agents/skills on demand).
- **Agents:** `architect` (design/proposal), `developer` (implements → PR), `reviewer` (independent audit, read-only), `qa` (executes and reports), `devops` (promote + deploy).
- **The kit grows itself:** if a workflow needs a role/skill that isn't here, `create-capability` authors it — personalized to this project — instead of forcing you to hand-prompt a generic one.
- **Config:** `.claude/devflow.md` (the project profile — the one place project specifics live).
- **KB:** `docs/kb/<topic>.md` + the pinned 📚 Knowledge Base index issue.

## Guardrails
- Never bypass the profile's base branch or the one-issue-per-PR rule.
- If the profile is missing/stale, fix it (or re-run `setup-devflow`) before routing to build/review.
- Prefer routing to a skill over doing the work ad-hoc here — the skills carry the discipline.
- **Never route a deploy request straight to promotion.** `devops-deploy` asks about QA every time; that question is the gate, not a formality.
- **Never let `reviewer` or `developer` claim an executed result.** If someone needs to know whether it works, that's `run-qa`.
- If `docs/qa/pending-qa.md` has entries with `status: pending` or `failed`, say so in the opening brief — undeployed unverified work is the thing most likely to bite.
