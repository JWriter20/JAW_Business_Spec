# Security Spec

Nothing gets more access, or more exposure, than it needs. Two things leak, and
neither can be un-leaked: credentials, and the contents of private repos.

---

## 1. Credentials

1. **Never commit a credential.** Not in source, not in config, not in a test
   fixture, not in a commit message, not in a log line, not in an error string.
2. **Secrets live in a managed store** — AWS Secrets Manager or Parameter Store
   as SecureStrings, or the platform equivalent. That store is the source of
   truth; nothing else is.
3. **`.env` is local development only, and only once you have verified it is
   gitignored.** Verify per repo. Never assume.
4. **Anything exposed is rotated.** A key removed in a later commit is still in
   the history and still compromised; deleting it from `HEAD` is not a
   rotation.
5. **Report keys by expiry, never by value.** The business report describes a
   credential by provider, kind, blast radius, and expiry, and carries no
   material of any kind (MONITORING_SPEC.md §26).

## 2. Least privilege

Every identity — person, service, CI job, agent, token — holds the narrowest
permission that lets it do its work, and nothing kept "just in case."

1. **Scope by default.** No wildcard IAM actions or resources (`"*"`), no
   admin/owner role where a scoped one works, no write scope on a credential
   that only reads (MONITORING_SPEC.md §26 reports that scope as blast radius).
2. **One identity per workload.** A shared credential makes the blast radius the
   union of everything using it, and makes revocation impossible without an
   outage.
3. **Elevated access is time-boxed.** Standing admin is the exception, and the
   exception is recorded in `TRIBAL_KNOWLEDGE.md`.
4. **Widening a permission is a reviewed change, never a debugging step.** If
   opening a policy made it work, the real fix has not been found
   (GENERAL_RULES.md §2).

## 3. Network exposure

1. **Internal, admin, and machine-to-machine endpoints are IP-gated.** An
   explicit allowlist of addresses, CIDR ranges, and hostnames; every other
   caller is refused *before* authentication is attempted. MONITORING_SPEC.md
   §29 is the worked example.
2. **Never `0.0.0.0/0`.** Not in a security group, ingress rule, firewall, or
   allowlist config — and no `::/0`, no wildcard hostname, no "allow all" value.
   An empty, missing, or unparseable allowlist denies every request rather than
   defaulting open.
3. **A deliberately public surface is the exception, and it stays narrow.** Only
   the routes customers actually need are exposed, still behind authentication
   and rate limiting; no admin, internal, or dashboard route is ever among them.
4. **Bind to the narrowest interface.** Databases, caches, queues, and SSH
   listen on localhost or a private subnet, never on a public one.

## 4. Public repositories

**IMPORTANT — a public repository is a FINISHED PRODUCT, and it is written as
one.** Whoever finds it — a customer, an investor, a candidate, a competitor —
judges the business by what is in it, and judges it without asking what any file
was for. There is no "temporarily", no "I'll clean it up later", and no "nobody
will look at that file." If it is not part of the finished product, it is not
committed.

Confirm whether a repository is public **before** writing anything into it.

1. **Nothing from a private repo appears in a public one.** Not in code, not in
   comments, not in commits, not in PRs, not in issues, not in branch names, not
   anywhere.
2. **No proprietary results.** Benchmarks, model outputs, internal metrics,
   customer data, pricing, and vendor terms from private work stay private.
3. **IMPORTANT — no garbage, ever.** None of this is ever committed to a public
   repo: scratch files, `tmp/`, `test.py`, `notes.md`; diagnostic or one-off
   scripts; saved test output, logs, coverage dumps, `.DS_Store`, editor
   directories; debug logging, commented-out code, dead code, unused
   dependencies; half-finished features; placeholder text, stub docs, or `TODO`
   and `FIXME` left as a note to self; and filler documentation written to look
   substantial rather than to be read (GENERAL_RULES.md §5, §6).
4. **Experimentation has exactly one home:** `src/experiments/`, gitignored.
   That is the only place unfinished work may exist in a public repository, and
   it never ships.
5. **Everything visible is written for a stranger.** README, naming, commit
   messages, PR titles, issues, releases. No `wip`, no `fix`, no `final v2`.
   Nothing merges that you would not be happy to see quoted.
6. **Publishing a private repo publishes its entire history**, every branch and
   every commit — not just `HEAD`. Audit the history before flipping the switch,
   and rotate anything it ever contained (§1.4).
