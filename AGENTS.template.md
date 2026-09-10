<!-- Copy to the root of a repo as AGENTS.md, then fill in "This repository".
     Everything above that section is the same in every repo — edit it here,
     in JAW_Business_Spec, never in a copy. -->

# AGENTS.md

Instructions for any agent working in this repository. Full specs:
`https://github.com/JWriter20/JAW_Business_Spec`.

## Before you touch anything

1. **Is this repo public or private?** If public, the Professionalism rules
   below govern every line, commit, and PR title you write.
2. **Read `TRIBAL_KNOWLEDGE.md`** — the decisions made here and why.
3. **Branch off `dev`.** Never commit to `main`, never push to `main`.

## Before you write a function

Grep `repomix-output.md` for likely names, then check `FILE_PURPOSES.md` for
where that behavior belongs. Write new code only after confirming nothing
comparable exists; if something close exists, extend it rather than fork it.
Duplicated logic is a serious defect. `npm run repomix` rebuilds the map (~2s)
if it is stale.

Update `FILE_PURPOSES.md` when you add a file — CI fails without the entry.

## Professionalism (public repos)

**IMPORTANT: a public repository is a FINISHED PRODUCT.** Customers, investors,
and candidates judge the business by what is in it, and they do not ask what a
file was for.

- **No garbage, ever.** No scratch files, `tmp/`, diagnostic scripts, saved test
  output, logs, debug logging, commented-out code, dead code, placeholder text,
  stub docs, or `TODO`s left as a note to self.
- **Nothing from a private repo** reaches a public one — not in code, comments,
  commits, PRs, issues, or branch names. No proprietary results or metrics.
- **Experiments live in `src/experiments/`, gitignored.** That is the only place
  unfinished work may exist here.
- Commits and PR titles are written for a stranger reading them in a year.

## Code

- **Less code.** More code is a cost, not an achievement. Minimal, general
  changes; delete dead code when you find it.
- **No band-aids.** Fix the main flow. A hard-coded value or a patch at the call
  site is a bug relocated, not fixed.
- **Fallbacks are a desperate measure.** They turn a loud failure into a silent
  wrong answer and hide the real defect. Fail loudly instead.
- **Read the provider's docs** before writing against a third-party API. Never
  infer an endpoint, field, or rate limit from memory.
- **Run independent work concurrently** — `Promise.all()`, one batched query
  over N in a loop.
- **Comments say why, in a sentence or two.** Simple code needs none. Anything
  longer is a decision — put it in `TRIBAL_KNOWLEDGE.md`.

## Tests

- **Run the failing test, not the suite.** CI runs the full suite on every PR.
- **Write test output to a log file and grep the file.** Piping a run into
  `grep` throws away output you will need and forces a second run. Delete the
  log afterward.
- **Every bug gets a regression test, in this order:** reproduce it with a test
  that you verify fails → fix → verify it passes → land it in CI.
- **No flakes.** An intermittent failure means the code is non-deterministic.
  Fix the behavior. Never retry, loosen, or skip.
- **Never write a test just to pass, or code just to pass a test.**

## Security

- **Never commit a credential** — not in code, config, fixtures, logs, or commit
  messages. Secrets live in the managed store; `.env` is local only, and only
  once you have verified it is gitignored.
- **Least privilege.** No wildcard permissions, no admin role where a scoped one
  works, no write scope on a read-only credential.
- **No `0.0.0.0/0`.** Internal and admin endpoints are IP-allowlisted before
  authentication.
- **Dependencies are pinned exactly** and never auto-upgraded. Adding one is a
  decision worth recording.

## Working with the user

- Say when the direction is wrong, before starting, with the reason.
- Review your own code the way a strict senior manager would. You are biased
  toward what you just wrote.
- Explain plainly and briefly. The user is technical; short beats complete.

## Parallel work

Separate PRs → one agent per task, each in its own worktree, at the same time.
One PR, independent slow parts → subagents in worktrees, merge each into your
branch as it finishes, then delete the worktrees. Shared files or ordering
requirements → one agent.

---

# This repository

<!-- Everything below is repo-specific. Keep it short. -->

**What it is:**
**Stack:**
**Public or private:**
**Run / build / test:**
**Layout:** see `FILE_PURPOSES.md`
**Gotchas:** see `TRIBAL_KNOWLEDGE.md`
