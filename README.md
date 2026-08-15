# DevFlow Kit

A drop-in set of **Claude Code skills + agents** that gives any project a disciplined, trackable workflow:

> **requirement → docs → issues (with a shared knowledge base) → PR → review → merge**

Instead of "tell Claude to build a feature and push one giant commit," you get proper issues, per-feature knowledge docs, reviewable PRs against the branch you choose, an independent review pass, and a GitHub trail you can search, change, or roll back later.

**Self-tailoring, not static.** The kit ships generic, but its **first run analyzes YOUR codebase and rewrites the skills + agents to fit it** — baking in your real architecture, base branch, schema-change rule, security/cost invariants, test commands, and the areas the agents must not touch. After setup, your `developer`, `reviewer`, and `implement-issue` read like they were written for your repo, not a template. A machine-readable profile (`.claude/devflow.md`) holds the values that can change (branch, commands); the durable architecture is baked into the skills themselves. Same kit fits a Node SaaS, a Python API, or a Go service.

---

## Install (into your project)

1. Open **your project** in Claude Code.
2. Paste this repo's URL and say:

   > **"Follow this repo's README to set up DevFlow in this project."**

3. Claude will run the bootstrap below, then hand off to the `setup-devflow` skill, which **analyzes your codebase, rewrites the skills/agents to match it**, interviews you for what it can't detect (which branch to target, test command, etc.), and files your first issues + knowledge base.

That's it. From then on, start any session with **`/devflow`** — the orchestrator that knows every skill and routes your request.

---

## Bootstrap (what Claude does when you say "follow the README")

Claude, perform these steps in the user's current project (the target repo), then stop and follow the setup skill:

1. Clone this kit to a temp dir (do not clone into the project):
   ```bash
   git clone <THIS_REPO_URL> /tmp/devflow-kit
   ```
2. Copy the skills and agents into the target project:
   ```bash
   mkdir -p .claude/skills .claude/agents
   cp -r /tmp/devflow-kit/.claude/skills/*  .claude/skills/
   cp -r /tmp/devflow-kit/.claude/agents/*  .claude/agents/
   cp -r /tmp/devflow-kit/templates         .claude/devflow-templates
   ```
3. Load and follow **`.claude/skills/setup-devflow/SKILL.md`** — the first-run setup. Do not skip its interview; it must ask the user which branch PRs should target (there is no default).

---

## What's inside

| Skill | Role |
|---|---|
| `setup-devflow` | First run: **analyze the codebase → personalize every skill/agent to this project**, interview for the rest, generate `.claude/devflow.md`, git/GitHub setup, file the KB index issue + initial issues from your docs. |
| `devflow` | **Session entrypoint / orchestrator.** Call `/devflow` each session; it reads the profile + open issues and routes to the right skill/agent. |
| `create-issue` | Turn a requirement/finding into a self-contained issue + an architect-level design (via the `architect` agent) + a linked KB doc. |
| `implement-issue` | Issue → branch → implementation → **PR against your chosen branch** (one issue per PR). Uses the `developer` agent. |
| `review-pr` | Audit a PR against its issue's acceptance criteria + a quality/security rubric; comment on PR + issue. Uses the `reviewer` agent. |
| `run-qa` | **Actually runs the software.** Executes the suite, the build, and Playwright specs it writes against the live app; records a `## QA report` on each issue's KB doc and comments `## QA` on the issue when something fails. Uses the `qa` agent. |
| `devops-deploy` | Promote the base branch to production and deploy. **Asks whether to run QA first, every time — and a QA failure blocks the promotion.** Verifies health, then closes the resolved issues. Uses the `devops` agent. |
| `audit-codebase` | Scan code + docs for gaps/risks → propose → file issues. |
| `write-tests` | Author test cases mapped to acceptance criteria. |
| `create-capability` | **The kit grows itself.** When a workflow needs a role/skill that doesn't exist yet, this authors a **new agent or skill personalized to your project** (from the requirement + codebase + KB) and registers it — no expert prompting required. |

| Agent | Role |
|---|---|
| `architect` | Turns a requirement/issue into an architect-level design proposal (options, schema/security/rollout, invariant tests). Read-only — designs, doesn't implement. |
| `developer` | Implements an approved issue through the project's existing architecture; writes automated + invariant tests. Registers what changed for QA. |
| `reviewer` | Independent PR/issue reviewer; criteria + security audit. **Read-only — never runs anything.** |
| `qa` | **The only agent that executes.** Runs the suite, the build, and browser specs; reports what actually passed — and what it could not check. |
| `devops` | Promotes and deploys. Asks about QA first; blocked by a QA failure; closes issues only once production health is green. |

**Knowledge base:** each issue links a `docs/kb/<topic>.md` doc (many issues can share one). A single pinned **"📚 Knowledge Base" index issue** maps every doc + issue — your searchable memory of *why* each piece exists. After QA runs, that same doc carries the **`## QA report`** — so the *why* and the *was it ever verified* live together.

---

### Who executes what — the gap this closes

`developer` marks its own homework. `reviewer` is **forbidden** from running anything. Without a third
role, **nobody in the loop has ever observed the software behave** — every "correct on review" verdict
is a reading, and it's entirely possible to merge and ship a batch nobody executed.

`qa` closes that. Its verdict is the only one grounded in observation, which is exactly why it must
never overstate: **a check it didn't run is reported `not covered`, never `pass`.** Every report ends
with the boundary of what it *couldn't* verify — live-provider calls, real credentials, a second
tenant, true concurrency across separate processes. That list is what you work from.

**Credentials never touch an agent.** Specs read `process.env.QA_*`; the values live in a gitignored
`.env.qa` only you hold. Setup asks for variable *names*, never values — a password pasted into a
chat is in that transcript permanently.

```
issue ─► implement-issue ─► PR ─► review-pr ─► HUMAN merges
                             └─ registers the change in docs/qa/pending-qa.md
                                                    │
                                              run-qa ─► executes; report on the KB doc
                                                    │      (failure → ## QA comment on the issue)
                                                    │
                                          devops-deploy ─► "run QA first?" → promote → health → close
```
