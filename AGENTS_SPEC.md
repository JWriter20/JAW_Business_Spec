# Agents Spec

How a coding agent operates inside these repositories. Everything here is agent
behavior; what the repository itself must be is in the other specs, and an agent
is bound by those too.

---

## 1. Two kinds of repo

`AGENTS.md` has one job: be the file an agent reads when it has nothing else.
Who that agent is working *for* differs between repos, and that decides what the
file says.

| Kind | What it is | Who reads its `AGENTS.md` |
|---|---|---|
| **Internal** | Every private repo, and any public repo nobody installs. | An agent we sent to change this code. |
| **Distributed** | A public dev tool — a published package, CLI, SDK, or MCP server that other people install and build on. | An agent helping a developer *use* the product. It found the repo in `node_modules`, in a clone beside their own project, or through our MCP server. |

The split is not visibility, it is who the reader works for. `JAW_Business_Spec`
is public and internal; `CaptchaKraken` is distributed.

**In a distributed repo the consumer is the default reader, and `AGENTS.md` is
written for them** (§3). The contributor rules — branch discipline, the test
toolchain, `FILE_PURPOSES.md` — move into `CONTRIBUTING.md`, which is where the
ecosystem already looks and where GitHub surfaces them on every PR and issue.
`AGENTS.md` opens with two lines routing there.

### Why split by file, and not two sections or two folders

A consumer's agent loads `AGENTS.md` into every session inside *their* project.
Our contributor rules are dead weight there at best and actively wrong at worst:
"branch off `dev`", "fallbacks are a desperate measure", and "delete dead code
when you find it" read as instructions about the code the agent is holding —
which is not ours. They also publish how we work internally, in the one file
every user of the product is guaranteed to read.

Splitting by folder fails for a different reason: the source a contributor edits
*is* the product, so there is no subtree to move aside, and a harness that reads
the nearest `AGENTS.md` walking up from the repo root would hand a contributor
the consumer file anyway. Two audiences, two files, one router line.

## 2. Required root files

| File | Contents |
|---|---|
| `AGENTS.md` | **Internal:** [AGENTS.template.md](AGENTS.template.md), copied verbatim, plus a short repo-specific section. **Distributed:** written for whoever uses the product, covering §3. |
| `CONTRIBUTING.md` | **Distributed only, and required there:** the GENERAL RULES block of [AGENTS.template.md](AGENTS.template.md), under a heading saying it is for anyone — agent or human — changing this code, plus dev setup, how to run the gates, and the PR rules. |
| `CLAUDE.md`, `.cursorrules`, `GEMINI.md`, and every other default-read harness file | Exactly one line. **Internal:** `Read the AGENTS.md file for all instructions on behavioral operation`. **Distributed:** the same line naming `CONTRIBUTING.md` instead (§2.1). |
| `TRIBAL_KNOWLEDGE.md` | Repo decisions and the reasons behind them. |
| `FILE_PURPOSES.md` | Every tracked file in the repo, and what it is for (§4). |

Every agent file a major harness reads by default must exist. A missing one
means that harness starts with no instructions at all.

### 2.1 In a distributed repo the harness files point at `CONTRIBUTING.md`

A harness reads `CLAUDE.md` because the repo is its **working directory**, which
only happens to someone changing this code. A developer who installed the
package has it in `node_modules`, and their harness reads *their* `CLAUDE.md`,
never ours. So the one-liner naming `CONTRIBUTING.md` puts a contributor's agent
one hop from its rules and costs a consumer's agent nothing.

That matters because agents do not reliably follow pointers — the reason the
template is copied verbatim into a repo rather than linked from it. `AGENTS.md`
still opens with two lines routing to `CONTRIBUTING.md`, for the agent that
arrived by reading the repo instead of by working in it, but nothing depends on
those two lines being obeyed.

The GENERAL RULES block survives being published almost intact — less code, no
band-aids, no fallbacks, the test rules, never commit a credential, and pin
dependencies all apply to a stranger's PR exactly as they apply to ours. Three
lines are ours alone and are kept anyway rather than reworded: nothing from a
private repo reaches a public one, secrets live in the managed store, and
internal endpoints are IP-allowlisted. An outside contributor skips them; we are
the ones who would pay for their absence.

### `AGENTS.md` is a copy, not a rewrite

An agent reads `AGENTS.md` and nothing else by default, so the recurring rules
from this spec set have to be *in* it — professionalism, secrets, branching,
test discipline, less code, no fallbacks. They are already condensed into
[AGENTS.template.md](AGENTS.template.md): copy it to the repo root as
`AGENTS.md` and fill in the repo-specific section at the bottom. Nothing else in
it is edited per repo. In a distributed repo that same block is copied to
`CONTRIBUTING.md` instead, and `AGENTS.md` is written from scratch for the
product (§3).

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

## 3. Product-facing `AGENTS.md`

Distributed repos only. The reader is an agent that has been asked to make
something work: it will follow this file literally and then tell a developer
what it did. Optimize for exactly that — fewest steps to a working call, and no
claim it can get wrong.

There is no template for this file — the shape differs too much between a solver
API, a CLI, and a library for one to be worth copying. What does not differ is
what has to be covered, and the order, because a reader stops as soon as it can
act. `CaptchaKraken/AGENTS.md` is the worked example.

1. **What it is in two sentences**, and what it is not.
2. **The choice that has to be made before installing anything**, as a table
   keyed on what the reader can observe about their machine or account — hosted
   vs. self-hosted, which tier, which port. An agent picks badly from prose and
   well from a table.
3. **Install, then one call that works.** Copy-pasteable, and tested
   (TESTING_SPEC.md §1.4).
4. **Credentials, obtained without ever printing one.** If we ship an MCP
   server, that is the path — it writes the key to disk and the transcript never
   holds it (SECURITY.md §1). Say so explicitly, because the obvious move for an
   agent is to echo the key it just created.
5. **The MCP tools, one line each: what it is for, not what it returns.** The
   tool schema already says what comes back. What it cannot say is which tool
   answers "am I out of credit?".
6. **Every error the service returns, and the fix for each.** This is the
   section that stops an agent wrapping a retry loop around a 403 that means
   "no licence".
7. **Environment variables**, with the one that switches endpoints called out.
8. **Rules for agents** (§3.1).
9. **A table of links into `docs/`** for everything else. Never inline a guide.

### 3.1 Rules for agents

Every product-facing `AGENTS.md` ends with these four, made concrete for the
product. They exist because an agent asked about our product in front of a
customer will produce an answer with or without us, and a plausible invented one
costs more than an admitted gap.

1. **Numbers come from one page, or from nowhere.** Name the single file holding
   every published figure — accuracy, latency, price, limits — and forbid
   deriving, rounding, or converting anything that is not on it. A count is not
   a percentage, and a measurement on one workload is not a promise about
   another.
2. **Name the capabilities that do not exist, and forbid routing around them.**
   Anything gated by licence, tier, or "hosted only" is listed here with what a
   request for it actually returns. An agent told only that a feature is
   unavailable will helpfully invent a workaround.
3. **Never print a credential** into a transcript, a log, or a commit, and name
   the local file that holds it as one that is never committed.
4. **What the licence permits building**, in one sentence, with a link. The
   reader is deciding whether to build on us right now.

Nothing else from [AGENTS.template.md](AGENTS.template.md) belongs in this file.

### 3.2 One `AGENTS.md` per published artifact

A monorepo that publishes `js/` to npm and `python/` to PyPI publishes neither
its root nor its root `AGENTS.md`. Whoever runs `npm i` gets `js/` and nothing
above it, so `js/AGENTS.md` must exist, be scoped to that package's surface, and
be listed in the manifest's published files. The root file carries what is true
across ports and routes to each.

## 4. Codebase navigation

Two references, for two different questions. Both are gated by CI
(TESTING_SPEC.md §6).

**`FILE_PURPOSES.md` — "where does X live?"** Every tracked directory and file,
its purpose, how modules flow into one another, and the DB schema. Read it when
you are unfamiliar with a subsystem, need to know which file owns a
responsibility before editing, or are planning a change that spans modules.

**IMPORTANT — tracked means `git ls-files`, and nothing else.** Build output
(`dist/`, `build/`, `js/dist/`), dependencies (`node_modules/`, `.venv/`),
coverage dumps, logs, and `src/experiments/` are gitignored, so they get no
entry, ever. Walking the directory tree to fill the map produces thousands of
lines describing files nobody edits, buries the hundred that matter, and goes
stale on the next build. Anything that generates or checks the map enumerates
`git ls-files`. The CI gate reads both directions: a tracked file with no entry
fails, and so does an entry for a file that has been deleted or newly ignored
(TESTING_SPEC.md §5).

**`repomix-output.md` — "does a helper for this already exist?"** repomix is
installed in every repo. Its output is a compressed, comment-stripped view of
every source file with full function and type signatures, regenerated on every
`npm run build` via the `postbuild` hook (`repomix.config.json`). It honors
`.gitignore`, so it maps source and not build output — keep it that way. It is
itself gitignored; if it is missing or looks stale, `npm run repomix` rebuilds
it in ~2s. Grep it directly — `grep -nE "function foo|const foo =" repomix-output.md`
beats walking the tree.

**Before writing any new helper, utility, or function:** grep
`repomix-output.md` for likely names, then check `FILE_PURPOSES.md` for the
canonical home of that behavior. Write new code only after confirming nothing
comparable exists. If something similar exists but is subtly different, extend
it rather than fork it — duplicating logic is a serious defect, not a shortcut.

## 5. Before you start

1. **Check whether the repo is public or private, and internal or distributed**
   (§1). If it is public, everything you write into it — code, comments,
   commits, PR text — is governed by SECURITY.md §4. If it is distributed, your
   rules are in `CONTRIBUTING.md`; `AGENTS.md` there is the product's own
   documentation, and it is a customer-facing surface you may not clutter.
2. **Branch off `dev`** (VERSION_CONTROL.md §2). Never commit to `main`.

## 6. Parallel work

| Situation | What to do |
|---|---|
| Several tasks that belong in **separate PRs** | Default to spawning one agent per task, each in its own worktree, running at the same time. |
| Several parts of **one PR** that are independent and slow | Spawn subagents in their own worktrees; as each finishes, merge its worktree into the branch you are on, then delete the worktrees. |
| Work that shares files or has an ordering requirement | One agent. Coordination costs more here than the parallelism returns. |

Every worktree follows the same branch discipline: off `dev`, in by PR.

## 7. Running tests

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

## 8. Working with the user

1. **Say when the direction is wrong.** If a suggestion will waste time or leads
   down a dead end, say so before starting, with the reason.
2. **Review your own code hardest.** Judge it the way a strict senior technical
   manager would — you are biased toward the code you just produced, and
   agreeing with the user is not the job either.
3. **Explain plainly.** The user is technical; concepts still get explained
   simply and briefly. Short and clear beats complete and long.
