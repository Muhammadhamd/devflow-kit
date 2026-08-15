---
name: devops-deploy
description: Promote merged work from the base branch to production and deploy it — asking first whether to run QA, and blocking the promotion if QA fails. Verifies health before declaring success, closes the resolved issues, and prunes the QA register. Use for "deploy", "ship it", "promote dev to main", or a production incident. Delegates to the `devops` agent. Ships already-merged code; it does not implement or review.
---

# DevOps deploy (QA gate → promote → deploy → verify → close)

The last checkpoint before users are affected. Everything earlier is revertible with a `git revert`;
this is where mistakes reach real people and real data. Bias toward stopping.

Read `.claude/devflow.md` — base branch, production branch, deploy pipeline, health check — and
**`.claude/deploy-config.json`** (gitignored) for the connection details.

## 1. The QA gate — always ask

Summarise what's pending from `docs/qa/pending-qa.md`, then ask:

> "N changes merged and undeployed, M not yet QA'd: `#12 publish flow`, `#15 image picker`.
> Two are marked *cannot be automated* (a live third-party publish, a cross-tenant check).
> Run QA first? (yes / no / show me)"

Ask **every promotion** — not once per session, not "if it seems needed".

| Answer | Action |
|---|---|
| **yes** | Run `run-qa` (the `qa` agent). Wait. Act on the verdict. |
| **no** | Proceed — and **record in the deploy notes that QA was skipped, and that the user chose it.** Skipping is their call; hiding it isn't. |
| **show me** | Print the register, ask again. |

### A `failed` QA blocks the promotion
Do not promote. Instead:
1. Identify the issue the failing change belongs to — the register entry's issue, or the one the
   merged PR closed.
2. Comment on it:
   ```
   ## QA
   <failing behaviour · observed vs expected · exact reproduction · "this blocked the <date> deploy">
   ```
3. Tell the user what failed, which issue you commented on, and that the deploy is stopped.

Only an **explicit, unambiguous override** ("deploy anyway despite the QA failure") releases the
block. Silence, "ok", or a change of subject is not consent. When overridden, deploy and record the
override, the failure, and who authorised it.

**`passed with exceptions` is not a pass** — surface the exceptions and ask.

## 2. Promote
- Production **mirrors** the base branch. Fast-forward.
- No fast-forward → the profile's non-destructive merge rule, **aborting on a real conflict**. A
  conflict means the branches diverged: a question for a human, not something to resolve under
  deploy pressure.
- **Never force-push production.** Never promote a non-base branch or unmerged work.

## 3. Deploy
Follow the profile's pipeline. Regardless of specifics:
- **Back up before any schema or data mutation** — and confirm the backup exists before proceeding.
- Run migrations the project's way. Never hand-edit production data to make one pass.
- **Verify health before declaring success.** A started process is not a working service: hit the
  health endpoint the profile names, and check the worker/queue side too if one exists.
- Not green → **roll back first, diagnose after.** A fast rollback beats a clever fix at 2am.

## 4. Close out (only on green)
- Close the issues this deploy resolved, commenting with the deploy reference. An issue closed on a
  broken deploy is a lie that outlives the incident.
- Prune `Verified — awaiting deploy` entries from `docs/qa/pending-qa.md` — they've shipped.
- Update the KB index issue if the project keeps one.
- Report: what promoted, what deployed, health result, issues closed, and **whether QA ran**.

## Secrets
Connection details live in **`.claude/deploy-config.json`** (**gitignored**) — host, user, key
**path**, deploy script, health URL, backup + rollback. Read it; never print it, never put it in a
tracked file, an issue, a commit message, or the transcript.

That file holds **paths and names only**. A key, password, or token *inside* it is a
misconfiguration — say so rather than using it. Never rotate, echo, or "just check" a value; if one
is missing, name the field and stop. A secret found committed is an incident: stop and say so.

## Delegation
Spawn the **`devops`** agent. It owns promotion, deploy, health, rollback, and issue closing.

## Anti-patterns
- ❌ Deploying without asking about QA. ❌ Proceeding past a `failed` QA without an explicit override.
- ❌ Force-pushing production, or resolving a real conflict unilaterally.
- ❌ Declaring success on "the process started". ❌ Closing issues before health is green.
- ❌ Writing a hotfix yourself when the deploy breaks — roll back and hand it to `developer`.
- ❌ Guessing at infrastructure the profile doesn't describe.
