# JAW Business Specs

The set of specs that every business I build must follow. One concern per file.

These are normative: **must** is a requirement, **should** is a default that
needs a reason to break, and a repo that breaks one records why in its
`TRIBAL_KNOWLEDGE.md`.

| Spec | Covers |
|---|---|
| [GENERAL_RULES.md](GENERAL_RULES.md) | How code is written: minimal, general, no band-aids. |
| [SECURITY.md](SECURITY.md) | Credentials, least privilege, network exposure, public repos. |
| [TESTING_SPEC.md](TESTING_SPEC.md) | What a test must prove, the toolchain, coverage, CI gates. |
| [DEPENDENCIES.md](DEPENDENCIES.md) | Package managers, pinning, the monthly upgrade. |
| [VERSION_CONTROL.md](VERSION_CONTROL.md) | Branches, environments, deployments, rollback. |
| [MONITORING_SPEC.md](MONITORING_SPEC.md) | `jaw-business-report/1` — the one reporting endpoint every business serves. |
| [AGENTS_SPEC.md](AGENTS_SPEC.md) | How coding agents operate in these repos. |
| [AGENTS.template.md](AGENTS.template.md) | The `AGENTS.md` an internal repo copies to its root. |

The first six say what a repository must **be**. `AGENTS_SPEC.md` says how an
agent working in one **behaves**; where it restates a rule, it is the same rule
applied to an agent's workflow, not a second rule. `AGENTS.template.md` is the
condensed version an agent actually reads at work — it is copied into every
internal repo, so a rule change lands in the spec and in the template together.

Who reads a repo's `AGENTS.md` decides what goes in it (AGENTS_SPEC.md §1). In
an **internal** repo it is an agent we sent to change the code, and the template
is the whole file. In a **distributed** one — a public dev tool other people
install — it is an agent helping a developer *use* the product, so `AGENTS.md`
documents the product and `CONTRIBUTING.md` carries the rules for working on it.

**Precedence.** A repo's own `AGENTS.md` may add constraints, never relax one.
Where a repo conflicts with this set, the repo is wrong.

**Status:** v1. **License:** Apache-2.0.
