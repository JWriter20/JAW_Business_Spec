# Agents Spec

How a coding agent operates inside these repositories. Everything here is agent
behavior; what the repository itself must be is in the other specs, and an agent
is bound by those too.

---

## 1. Required root files

| File | Contents |
|---|---|
| `AGENTS.md` | [AGENTS.template.md](AGENTS.template.md), copied verbatim, plus a short repo-specific section. The only one of these with real content. |
| `CLAUDE.md`, `.cursorrules`, `GEMINI.md`, and every other default-read harness file | Exactly one line: `Read the AGENTS.md file for all instructions on behavioral operation` |
| `TRIBAL_KNOWLEDGE.md` | Repo decisions and the reasons behind them. |
| `FILE_PURPOSES.md` | Every file in the repo, and what it is for. |

Every agent file a major harness reads by default must exist. A missing one
means that harness starts with no instructions at all.

### `AGENTS.md` is a copy, not a rewrite

An agent reads `AGENTS.md` and nothing else by default, so the recurring rules
from this spec set have to be *in* it — professionalism, secrets, branching,
test discipline, less code, no fallbacks. They are already condensed into
[AGENTS.template.md](AGENTS.template.md): copy it to the repo root as
`AGENTS.md` and fill in the repo-specific section at the bottom. Nothing else in
it is edited per repo.

1. **The specs are the source of truth; the template is the copy.** A rule
   change lands in the spec and in the template in the same PR, and repos pick
   it up on their next sync.
2. **Never paste a whole spec in.** `AGENTS.md` carries the rules that recur in
   day-to-day work. MONITORING_SPEC.md is a format reference — link it, do not
   inline it.
3. **A repo may add constraints, never relax one** (README.md).

### `TRIBAL_KNOWLEDGE.md`

When a key decision is made about the repo, add **2–3 sentences**: the rule, and
why we chose it. Not a changelog and not a design doc — the thing a new engineer
would otherwise learn by breaking something.

## 2. Codebase navigation

Two references, for two different questions. Both are gated by CI
(TESTING_SPEC.md §6).

**`FILE_PURPOSES.md` — "where does X live?"** Every directory and file, its
purpose, how modules flow into one another, and the DB schema. Read it when you
are unfamiliar with a subsystem, need to know which file owns a responsibility
before editing, or are planning a change that spans modules.

**`repomix-output.md` — "does a helper for this already exist?"** repomix is
installed in every repo. Its output is a compressed, comment-stripped view of
every source file with full function and type signatures, regenerated on every
`npm run build` via the `postbuild` hook (`repomix.config.json`). It is
gitignored; if it is missing or looks stale, `npm run repomix` rebuilds it in
~2s. Grep it directly — `grep -nE "function foo|const foo =" repomix-output.md`
beats walking the tree.

**Before writing any new helper, utility, or function:** grep
`repomix-output.md` for likely names, then check `FILE_PURPOSES.md` for the
canonical home of that behavior. Write new code only after confirming nothing
comparable exists. If something similar exists but is subtly different, extend
it rather than fork it — duplicating logic is a serious defect, not a shortcut.

## 3. Before you start

1. **Check whether the repo is public or private.** If it is public, everything
   you write into it — code, comments, commits, PR text — is governed by
   SECURITY.md §4.
2. **Branch off `dev`** (VERSION_CONTROL.md §2). Never commit to `main`.

## 4. Parallel work

| Situation | What to do |
|---|---|
| Several tasks that belong in **separate PRs** | Default to spawning one agent per task, each in its own worktree, running at the same time. |
| Several parts of **one PR** that are independent and slow | Spawn subagents in their own worktrees; as each finishes, merge its worktree into the branch you are on, then delete the worktrees. |
| Work that shares files or has an ordering requirement | One agent. Coordination costs more here than the parallelism returns. |

Every worktree follows the same branch discipline: off `dev`, in by PR.

## 5. Running tests

1. **Run the failing test, not the suite.** The full suite runs in CI on every
   PR; running it locally to chase one failure burns time and money.

   ```bash
   vitest run test/prodTests/LabelCollection/*.test.ts -t "specific test name"
   ```
2. **Write output to a log file, then query the file.** Piping a run straight
   into `grep` discards the output you will need thirty seconds later and forces
   a second full run. Redirect to a temp log, grep that, delete it when done.
3. **Fix bugs in the order TESTING_SPEC.md §4 requires** — reproduce failing,
   fix, verify passing, land the test in CI. Do not reorder it; a test written
   after the fix has never been seen to fail.

## 6. Working with the user

1. **Say when the direction is wrong.** If a suggestion will waste time or leads
   down a dead end, say so before starting, with the reason.
2. **Review your own code hardest.** Judge it the way a strict senior technical
   manager would — you are biased toward the code you just produced, and
   agreeing with the user is not the job either.
3. **Explain plainly.** The user is technical; concepts still get explained
   simply and briefly. Short and clear beats complete and long.
