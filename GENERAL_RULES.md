# General Engineering Rules

How code gets written in every repository, whoever is writing it.

---

## 1. Less code

More code is a cost, not an achievement. A change is measured by what it leaves
behind in the codebase, not by what it took to write.

1. **Check for an existing function before writing a new one.** Duplicated logic
   is the most expensive code we own — two places to fix, one of which gets
   forgotten.
2. **Extend, don't fork.** If something close already exists but is subtly
   different, generalize it. A second near-identical helper is two behaviors
   waiting to drift apart.
3. **Keep the diff minimal.** A small general change beats a large specific one.
4. **Delete dead code when you find it** — after confirming nothing outside the
   repo depends on it. External consumers (`../FlashApply`) do not appear in a
   local grep.

## 2. No band-aids

1. **Fix the main flow.** A hard-coded value, a special case for one input, or a
   patch at the call site instead of inside the function is a bug relocated, not
   a bug fixed.
2. **Fallbacks are a desperate measure, not a design.** A fallback that quietly
   produces a plausible answer converts a loud failure into a silent wrong one
   and hides the real defect indefinitely. Fail loudly instead. On the rare
   occasion one is warranted, it is a decision — record it in
   `TRIBAL_KNOWLEDGE.md`.
3. **Never satisfy a test by weakening it** (TESTING_SPEC.md §1).

## 3. Third-party integrations

Read the provider's current documentation before writing against their API.
Endpoints, field names, error shapes, and rate limits are never inferred from
memory or from how a similar service behaves.

## 4. Concurrency

Independent work runs concurrently — `Promise.all()` over sequential awaits, one
batched query over N in a loop. Serial work with no ordering requirement is
latency we chose to pay.

## 5. Comments

Comments carry **tribal knowledge**: why the code is this way and what it trades
off. What the code does is already written down, in the code.

- One or two sentences. Simple functions often need none.
- State the reason for a non-obvious decision, then stop.
- Anything longer than that is a repo-level decision — it belongs in
  `TRIBAL_KNOWLEDGE.md`, not in a comment block.

## 6. Documentation

Every repository has a root `docs/` folder, kept deliberately small. One page
that is current beats five that are thorough; a document nobody finishes is a
document nobody reads.
