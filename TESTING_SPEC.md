# Testing Spec

A test exists to catch a defect. A test that cannot fail is a liability with a
green check next to it.

---

## 1. What a test must prove

1. **Test the behavior, not the happy path.** Cover the contract and its edges:
   empty, boundary, malformed, concurrent, and failing inputs.
2. **Never write a test just to pass, and never write code just to pass a
   test.** Either one produces a suite that proves nothing and blocks nothing.
3. **No flakes.** A test that fails intermittently is telling you the code is
   non-deterministic — fix the behavior underneath it. Never retry it, loosen
   the assertion, skip it, or write it off as a flake.
4. **Documented behavior is tested.** Every code snippet in the README and
   `docs/` has a test that runs it as written and asserts what it claims. A
   snippet without one is an untested public API that strangers copy and paste
   (GENERAL_RULES.md §6).

## 2. Toolchain

One runner per language, the same one in every repo, so a test is written and
read the same way everywhere.

| Language | Runner | Coverage |
|---|---|---|
| TypeScript / JS | Vitest | `@vitest/coverage-v8` |
| Python | pytest | `pytest-cov` |

Deviating is a decision, and it goes in `TRIBAL_KNOWLEDGE.md` with the reason.

## 3. Coverage

Coverage is reported to a coverage service — **Codecov**, the same one across
every repo — on every PR. A number nobody looks at changes nothing; it has to
land on the PR as a status check.

1. **Core services are well tested: ≥ 90% line and branch coverage.** Core means
   any service with `audience: external` (MONITORING_SPEC.md §9.1) and anything
   it depends on. Everything else has a floor of 80%.
2. **Coverage never goes down**, and the lines a PR changes are ≥ 90% covered.
   A large untested addition cannot hide behind a healthy repo-wide average.
3. **Coverage is a floor, not a goal.** It measures which lines ran, not which
   behavior was verified — a suite at 100% that asserts nothing is worth
   nothing (§1).

## 4. Every bug gets a regression test

A bug fix is not finished until the bug cannot return:

1. A test reproducing the bug, in the existing test file covering that area,
   **verified failing first**.
2. The fix.
3. The same test, verified passing.
4. That test running in CI, so the bug cannot be reintroduced.

A fix that skips step 1 has not been shown to fix anything.

## 5. Fail fast

Tests run cheapest first, everywhere they run — CI and locally. A run that is
going to fail should fail in the first seconds, on the stage that names the fix
exactly, not twenty minutes later underneath an end-to-end timeout.

No stage starts until the one before it passes:

| Stage | Gate | Cost |
|---|---|---|
| 1 | Lockfile matches the manifest (DEPENDENCIES.md §2); repomix output freshly generated; `FILE_PURPOSES.md` matches `git ls-files` exactly, in both directions; every README and `docs/` snippet claimed by a test; lint and typecheck. | seconds |
| 2 | Unit tests — no network, no database, no filesystem. | seconds |
| 3 | Integration tests — real database, containers, service boundaries. | minutes |
| 4 | End-to-end tests. | long |
| 5 | Coverage thresholds (§3), measured over the run above. | free |

Stage 1 exists because those failures are the cheapest to hit and the cheapest
to fix — a stale repomix or a missing `FILE_PURPOSES.md` entry is one command,
and there is no reason to spend a full test run discovering it. Order within a
stage follows the same rule: fastest first.

The `FILE_PURPOSES.md` gate enumerates `git ls-files`, never the directory tree,
and fails on an entry with no file just as it fails on a file with no entry. A
map that describes `dist/` and `node_modules/` is not a stricter map — it is
thousands of lines that go stale on the next build and bury the ones that matter
(AGENTS_SPEC.md §4).

## 6. CI gates

CI runs the full suite on every PR — that is what the full suite is for. The
build fails on any gate in §5, and merging is blocked until it is green. The
stage-1 gates keep the codebase navigable (AGENTS_SPEC.md §4); a stale map is
worse than no map, because it still gets trusted.
