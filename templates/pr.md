Closes #<N>

## Summary
<What changed and the approach (which design option from the issue). 2–4 sentences.>

## Acceptance criteria
- [ ] <criterion 1> — code: `file:line` · test: `file::name`
- [ ] <criterion 2> — code: `file:line` · test: `file::name`

## Feature flag & rollout
- **Flag:** `<FLAG_NAME>` (default: **off**) — enable with <how>.
- **Rollout:** <staging → canary → default-on, if applicable>.

## Human testing required (NOT run by the agent)
- [ ] <chaos/kill test that proves the invariant>
- [ ] <live-provider / real-credential check>
- [ ] <browser / mobile flow>

## Docs updated
- <doc(s) + KB doc changed>

## Rollback
<The exact revert — ideally a flag flip with no redeploy. What a rollback must reconcile, if anything.>

## Residual risks
<Anything unverified, deferred, or worth watching after merge.>
