# Dependency Spec

A dependency is code we did not write, running with our permissions. It changes
when we decide it changes, and never on its own.

---

## 1. Package managers

| Language | Manager | Registry |
|---|---|---|
| TypeScript / JS | npm | npm |
| Python | pip | PyPI |

One manager per language per repo — never two resolving the same tree. The
lockfile is committed, and CI and production install from the lockfile only
(`npm ci`, never `npm install`).

## 2. Pinning

1. **Every dependency is pinned to an exact version.** No `^`, no `~`, no `*`,
   no `latest`, no floating tags in images or CI actions.
2. **No automatic upgrades.** No auto-merged bot PRs, no `npm update` in a build
   step, no resolution at deploy time. A version changes only through a reviewed
   PR — including the monthly one in §3.
3. **Adding a dependency is a decision.** Check the standard library and
   existing code first (GENERAL_RULES.md §1). A package pulled in for one
   function is a permanent supply-chain surface bought for a few lines.

## 3. The monthly bump

Once a month a bot upgrades everything, on the normal path to production:

1. **One PR into `dev`**, raising every dependency to its latest version with
   the lockfile regenerated.
2. **Full CI runs.** If a package breaks the build, the bot drops that one back
   to its pinned version and names it in the PR; the rest of the upgrade still
   lands rather than blocking on it.
3. **It soaks in `dev`.** Merged to `dev` and exercised there, with the business
   report showing no new errors (MONITORING_SPEC.md §19) before it goes further.
4. **`dev` → `main` by the usual PR** (VERSION_CONTROL.md §2), rolling if the
   project has rolling deployments enabled (§3.3).

**Security patches do not wait for the cycle.** An advisory affecting a package
we run is upgraded immediately, down the same path at speed.
