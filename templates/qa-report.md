# QA templates

Two shapes. Both are maintained by agents, not by hand.

---

## A. `docs/qa/pending-qa.md` — the register

**Written by the `developer` agent / `implement-issue`** the moment a PR merges. **Consumed by the
`qa` agent.** One entry per merged change, oldest first. Entries move to `## Verified` when QA
passes and are pruned after a successful deploy.

```markdown
# Pending QA

Changes merged to `<base branch>` and not yet verified. The `qa` agent reads this to know what
changed; `devops` refuses to promote while anything here is `status: pending` or `status: failed`.

## Awaiting QA

### #<issue> — <short title>   (PR #<pr>, merged <YYYY-MM-DD>)
- **status:** pending | passed | failed
- **surfaces:** <the user-visible places this touches — routes, pages, jobs. "none (internal)" is valid>
- **risk:** <one line: what breaks if this is wrong>
- **verify:**
  - <a concrete, checkable behaviour — not "test the feature">
  - <...>
- **cannot be automated:** <live-provider / real-credential / multi-process checks, and why>
- **KB:** `docs/kb/<topic>.md`

## Verified — awaiting deploy
<entries that passed, with the QA run date>
```

**Rules**
- `verify` items are behaviours a human or a spec can check, phrased so a failure is unambiguous.
  "Toggling an image then publishing within the save window reflects the toggle" — not "check images".
- `cannot be automated` is required, and `none` is a valid answer. It is the honest boundary of what
  QA can claim, and it is what the human gets handed.
- An internal-only change still gets an entry with `surfaces: none (internal)` — the register is the
  record of *what shipped*, not only of what has a UI.

---

## B. `## QA report` — a section appended to the issue's KB doc

**Written by the `qa` agent** into `docs/kb/<topic>.md`, under the issue it verified. Appended,
never overwritten: a re-run adds a new dated block so the history of what was checked survives.

```markdown
## QA report — #<issue> (<YYYY-MM-DD>)

**Verdict:** passed | failed | passed with exceptions

| Check | Result | Evidence |
|---|---|---|
| Unit/service suite | pass / FAIL | `<command>` → <n> passed, <n> failed |
| Build | pass / FAIL | `<command>` |
| `<spec name>.spec.js` | pass / FAIL | <what it drove, and what it asserted> |

**What was verified**
- <behaviour> → <observed result>

**What failed** *(omit when nothing did)*
- <behaviour> → <observed>, expected <expected>. Reproduce: <exact steps or command>

**Not covered — still needs a human**
- <check> — <why it can't be automated: real credentials, a second tenant, N processes, a live provider>

**Specs added/updated:** `<path>` — <what it locks in>
```

**Rules**
- **Never report a check you did not execute.** An unrun check is "not covered", not "pass".
- A `pass` line must carry evidence — the command and its actual numbers. "Looks correct" is not a result.
- `Not covered` is the most important section. It is what stops a green report from being read as
  "fully verified", and it is the list the human works from.
- On `failed`, the same content goes to the originating **issue** as a `## QA` comment, so the
  failure is tracked where the work is tracked rather than only in a doc.
