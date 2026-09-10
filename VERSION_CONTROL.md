# Version Control Spec

`main` is production. `dev` is what production becomes next. Everything else is
temporary.

---

## 1. Environments

Any product that is user-facing — or that touches a service which can affect
production for users — runs two environments bound to two long-lived branches:

| Branch | Environment |
|---|---|
| `main` | production |
| `dev` | development |

## 2. Branching

1. **No direct pushes to `main`. Ever.** The only path into `main` is a PR from
   `dev`.
2. **Direct pushes to `dev` are allowed and strongly discouraged.** The standard
   path is: branch off `dev` → PR into `dev` → PR `dev` into `main` when ready.
3. **Delete merged branches**, both remote and local.

## 3. Deployments

1. **Cross-repo release order belongs to the dashboard.** In multi-repo projects
   one repo's deployment can depend on another's. The dashboard is programmed to
   merge and release in the correct order; that order is never coordinated by
   hand.
2. **Every deployment is revertible** in a single action, without waiting on a
   forward fix.
3. **Rolling deployments are available per project.** When enabled, a release
   reaches 10% of customers, holds ~6 hours, then 50%, holds ~6 hours, then
   100%. The monitoring service watches error rates across each hold and warns
   the dashboard if they climb; a warned rollout does not advance on its own.
