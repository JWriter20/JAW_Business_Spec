# Business Report — `business-report/1`

A portable contract for getting the operational numbers out of a business and
onto a dashboard. **One authenticated endpoint per business, one JSON document,
one shape for all of them.**

It is written for the case where one person or one small team runs several
independent products — different stacks, different clouds, different databases —
and wants a single page that says what is happening across all of them: growth,
money, customers, machines, service health, cost, outages.

The model is deliberately dull:

- Every business serves `GET /_dashboard/report` behind a bearer token.
- A collector polls each one on an interval, stores every snapshot, and graphs
  the history whether or not the business keeps history of its own.
- Concepts that businesses genuinely share — users, revenue, cost, uptime —
  are standardized, so "revenue across all companies" is a chart rather than a
  research project. Concepts that do not generalize get an escape hatch.
- Onboarding business number three is a config entry and an endpoint, not a
  change to the dashboard.

Nothing here is tied to a particular language, framework, cloud, or dashboard
implementation. The document is the interface; anything that can serve JSON over
HTTP can implement it.

**Status:** v1. Additive change is free within a major version — see
[§12 Versioning](#12-versioning).

**License:** Apache-2.0.

---

## 1. The one endpoint

```
GET  {base}/_dashboard/report
Authorization: Bearer <token>
```

Returns `200 application/json` with a **Business Report** document.

That is the whole surface. There is no POST, no mutation, no second endpoint.
A GET that changes state is a GET that a retry can break, and this thing gets
polled every 60 seconds forever.

### Why one document instead of many endpoints

Five businesses × eight concerns is forty endpoints to keep alive. One document
means a business is either reporting or it isn't, the dashboard has exactly one
failure mode to render, and adding a metric is a code change in one place with
no route to register. The cost is a fatter payload — capped at 1 MB in §10, and
in practice ~20–60 KB.

### Path

`/_dashboard/report` is the canonical path. The leading underscore keeps it out
of the way of real product routes.

A business may serve it somewhere else. If it already has a hardened internal
surface — an mTLS-terminated internal listener, a loopback-bound admin port, a
VPC-internal route — mounting the report there is better than opening a new
surface on the public site, and it costs one line of dashboard config. The
dashboard records the real URL per business; the path is a default, not a
requirement.

### Query parameters (all optional)

| Param | Meaning | Default |
|---|---|---|
| `sections` | Comma-separated list; return only these top-level sections. | all |
| `since` | ISO-8601 instant; `series` points before it may be omitted. | 30 days ago |

A business may ignore both and always return everything. The dashboard never
depends on them being honoured — they exist so an expensive report can be
narrowed during an incident.

### Response codes

| Code | When | Dashboard behaviour |
|---|---|---|
| `200` | A report was produced, even a partial one. | Render it. Show `errors[]` as a banner. |
| `401` | Missing or bad bearer token. | Red tile, "unauthorized". Does not retry-storm. |
| `429` | Rate limited. | Backs off, honours `Retry-After`. |
| `503` | The business cannot produce any report at all. | Red tile, keeps last good snapshot visible and marks it stale. |

**Prefer `200` with `errors[]` over `503`.** If the database read for `costs`
times out but everything else worked, return everything else and name the
failure. A dashboard that goes blank because one sub-query died is worse than
useless during the incident it exists to show you.

### Timing

- Respond in **under 2 s** at p95. The dashboard's client timeout is 10 s.
- Default poll interval is **60 s** per business, configurable.
- `Cache-Control: no-store`. The dashboard does its own storage.
- If the report is expensive, **serve a precomputed snapshot.** Compute it on a
  cron into a table or a file and have the endpoint read that. Set
  `business.ttlSeconds` to how stale that snapshot may be, and the dashboard
  will display the age instead of pretending the number is live.

---

## 2. Document shape

```jsonc
{
  "spec": "business-report/1",   // REQUIRED, exact string
  "business": { … },             // REQUIRED
  "status":   { … },             // REQUIRED
  "metrics":     [ … ],          // optional
  "series":      [ … ],          // optional
  "services":    [ … ],          // optional
  "hosts":       [ … ],          // optional
  "incidents":   [ … ],          // optional
  "inbox":       [ … ],          // optional
  "costs":       { … },          // optional
  "vendors":     [ … ],          // optional
  "deployments": [ … ],          // optional
  "extra":       { … },          // optional
  "errors":      [ … ]           // optional
}
```

Three required fields, and they are the three that make the tile render at all:
what this is, who it is, and whether it is on fire. Everything else is optional
so a business can ship a conforming report in an afternoon and grow it.

Unknown top-level keys are **ignored**, not rejected. Unknown enum values inside
a known field are treated as `"unknown"` and surfaced in the UI as such.

---

## 3. `business` — identity and freshness

```jsonc
"business": {
  "id": "acme-analytics",                   // REQUIRED. stable slug, [a-z0-9-]
  "name": "Acme Analytics",                 // REQUIRED. display name
  "env": "prod",                            // REQUIRED. prod | dev | staging
  "generatedAt": "2026-08-21T18:00:00.000Z",// REQUIRED. when THIS document was built
  "ttlSeconds": 60,                         // how stale the underlying data may be
  "revision": "a302ff66",                   // commit of the code that produced it
  "links": [ { "label": "Admin", "url": "https://…" } ]
}
```

`id` is the join key for everything the dashboard stores. Never change it.

`generatedAt` is when the *document* was assembled. If the numbers inside come
from a snapshot taken earlier, say so per-item with `asOf` — do not backdate
`generatedAt`.

---

## 4. `status` — the one-line roll-up

```jsonc
"status": {
  "level": "ok",              // REQUIRED. ok | warn | down | unknown
  "summary": "All services up. 1 warning: proxy bandwidth at 82%.",
  "since": "2026-08-19T04:00:00.000Z"
}
```

**The business computes this, not the dashboard.** Only the business knows that
a worker showing DOWN is a machine that was deliberately drained this morning,
or that a zero-order hour at 3am is normal. A generic rule over generic fields
gets that wrong and trains you to ignore the page.

Rule of thumb: `down` means customers are affected right now. `warn` means
something needs a human today. `ok` means go do something else.

---

## 5. `metrics` — everything that is a number

A **flat list**, not a nested object. The dashboard renders every business
uniformly without knowing any of their shapes, so every number arrives with
enough metadata to draw itself.

```jsonc
{
  "id": "revenue.mrr",           // REQUIRED. registry id (§5.1) or custom
  "label": "MRR",                // REQUIRED. always present, so the UI never
                                 //           needs to know what an id means
  "value": 128400,               // REQUIRED. number, or null for "not available"
  "unit": "usd_cents",           // REQUIRED. §5.2
  "kind": "gauge",               // gauge | counter | ratio    (default gauge)
  "group": "Revenue",            // free-text; the UI groups tiles by this
  "window": "30d",               // period the value covers
  "previous": 119050,            // same metric over the immediately prior window
  "direction": "up_good",        // up_good | down_good | neutral  (default neutral)
  "signed": false,               // may the value be negative? see §5.2
  "severity": "ok",              // ok | warn | crit
  "target": 200000,              // goal, drawn as a reference line
  "featured": true,              // promote to the top row of the business page
  "service": "api",              // attach to a service in `services`
  "asOf": "2026-08-21T17:00:00.000Z",
  "note": "Excludes lifetime plans; they have no recurring component."
}
```

### `direction` is the field that earns its keystrokes

It tells the UI whether green is up or down. Revenue up is good, cost up is bad,
p95 latency up is bad, error rate up is bad. Without it every delta is grey and
you have to read the label to know if you are winning. Set it on anything with a
`previous`.

### `previous` and deltas

`previous` is the same measurement over the immediately preceding window of the
same length — last 30 days vs the 30 days before that. The dashboard computes
and colours the delta. If a business cannot cheaply produce it, omit it; the
dashboard falls back to its own recorded history, which is worse only in that it
starts empty.

### 5.1 Well-known metric ids

**If a business has the concept, it uses the registry id.** That is what makes
"revenue across all companies" a chart instead of a research project. Anything
not in the registry is custom and must be namespaced with the business id:
`acme-analytics.reports.rendered`, `acme-api.solve.accuracy`.

| id | unit | kind | direction | meaning |
|---|---|---|---|---|
| `revenue.mrr` | `usd_cents` | gauge | up_good | Monthly recurring, normalized to a month. |
| `revenue.gross` | `usd_cents` | counter | up_good | Money collected in the window, before fees. |
| `revenue.net` | `usd_cents` | counter | up_good | After processor fees and refunds. |
| `revenue.arpu` | `usd_cents` | gauge | up_good | Net revenue ÷ paying users, per month. |
| `revenue.pending_payout` | `usd_cents` | gauge | neutral | Owed out but not yet paid (e.g. partner share). |
| `payments.count` | `count` | counter | up_good | Successful charges in the window. |
| `payments.failed` | `count` | counter | down_good | Declined or errored charges. |
| `payments.refunds` | `usd_cents` | counter | down_good | Refunded in the window. |
| `users.total` | `count` | gauge | up_good | All registered accounts, ever. |
| `users.new` | `count` | counter | up_good | Signed up in the window. |
| `users.active` | `count` | gauge | up_good | Did the core action in the window (set `window`). |
| `users.paying` | `count` | gauge | up_good | Currently on a paid plan or holding a balance. |
| `users.churned` | `count` | counter | down_good | Lapsed in the window. |
| `users.waitlist` | `count` | gauge | up_good | Waiting for access. |
| `usage.requests` | `count` | counter | up_good | Core-service requests served. |
| `usage.units` | `count` \| `credits` \| `tokens` | counter | up_good | Billable units consumed. A billable unit is a delivered job at one business and a prepaid credit at another, so this is the one id whose unit is not pinned. |
| `usage.success_rate` | `ratio` | ratio | up_good | Successful ÷ attempted, 0–1. |
| `usage.error_rate` | `ratio` | ratio | down_good | Errored ÷ attempted, 0–1. |
| `usage.latency_p50` | `ms` | gauge | down_good | Median end-to-end latency. |
| `usage.latency_p95` | `ms` | gauge | down_good | p95 end-to-end latency. |
| `cost.total` | `usd_cents` | counter | down_good | All spend in the window. |
| `cost.per_unit` | `usd_cents` | gauge | down_good | `cost.total` ÷ `usage.units`. |
| `margin.gross` | `ratio` | ratio | up_good | (net revenue − cost) ÷ net revenue. Set `signed: true` — it goes below zero. |
| `uptime.window` | `ratio` | ratio | up_good | Fraction of the window the core service was up. |
| `incidents.open` | `count` | gauge | down_good | Currently open, unresolved. |
| `vendors.at_risk` | `count` | gauge | down_good | External accounts projected to hit zero before they reset or renew. |
| `queue.depth` | `count` | gauge | down_good | Items waiting. Attach with `service`. |
| `queue.oldest_age` | `seconds` | gauge | down_good | Age of the oldest waiting item. |

A registry id carries a meaning, so reporting one with a different unit or
`direction` is an **error**, not a style difference — `cost.total` with
`direction: up_good` paints rising spend green.

Nothing here is required. A pre-revenue business reports the user and usage rows
and omits the money ones; that is not a conformance failure, and the linter says
"missing recommended" rather than "invalid". A business selling prepaid credits
legitimately has no `revenue.mrr` — there is no recurring component — and a
warning you have read and dismissed beats a fabricated number that makes the
cross-company revenue chart lie.

### 5.2 Units

`count`, `usd_cents`, `ratio` (0–1), `percent` (0–100), `ms`, `seconds`,
`bytes`, `credits`, `tokens`, `other`.

**Money is always integer cents.** Never floats, never dollars. A float that
drifts by a cent is a payout statement that fails to reconcile, and you find out
during an audit rather than at the keyboard.

`ratio` is 0–1, `percent` is 0–100. Pick one per metric and never mix — a "94"
that might be 94% or might be 9400% is a metric nobody trusts.

The upper bound is the half of that check that earns its keep: it catches a
percent posted into a ratio field, which would otherwise render as 8710%. The
lower bound is relaxed by **`signed: true`**, because some ratios are genuinely
negative — a gross margin below zero is a real number a business can post, and
most pre-launch months are one. Rejecting it would force the months you most
want to see off the chart. `margin.gross` is the only registry id that sets it.

When `unit` is `other`, set `unitLabel` to something short ("jobs/hr").

---

## 6. `series` — history for graphs

The dashboard records every snapshot it polls, so **every** metric graphs over
time whether or not a business serves history. `series` exists for two things
the dashboard cannot do for itself: bootstrapping a chart on day one, and
serving a resolution it never polled at.

```jsonc
{
  "metricId": "revenue.gross",
  "interval": "1d",              // 1m | 5m | 1h | 1d | 1w
  "points": [
    ["2026-07-23", 41200],
    ["2026-07-24", 38900],
    ["2026-07-25", null]         // null = no data, NOT zero
  ]
}
```

Points are `[timestamp, value]`. Timestamp is an ISO-8601 date (`2026-07-24`)
or instant (`2026-07-24T00:00:00.000Z`). Points must be sorted ascending.

**Where a series is served, it wins.** For any range a series covers, the served
points replace the dashboard's own recorded history for that metric id. The
business has the real rollup table; the dashboard has whatever it happened to
catch. Outside that range the dashboard's history is used.

`null` and "missing" are different from zero, and conflating them turns a quiet
weekend into a crash on the chart. Zero-fill in SQL where zero is the truth —
`generate_series` — and use `null` where it isn't.

---

## 7. `services` — core service health

```jsonc
{
  "id": "worker-pool",           // REQUIRED. stable within the business
  "name": "Batch worker pool",   // REQUIRED
  "status": "up",                // REQUIRED. up | degraded | down | unknown
  "kind": "worker",              // api | worker | inference | db | queue | cron
                                 // | frontend | external | other
  "critical": true,              // does `down` here mean customers are affected?
  "checkedAt": "2026-08-21T17:59:40.000Z",
  "latencyMs": 42,
  "message": "3/4 workers active; one drained for maintenance.",
  "dependsOn": ["load-balancer", "postgres"],   // ids of other services
  "uptimeWindow": { "window": "30d", "ratio": 0.9987 },
  "url": "https://…",
  "env": "prod"
}
```

`dependsOn` is what lets the dashboard draw the flow diagram — ingest feeds
processing feeds delivery — instead of an unordered grid of dots. It is also how
a single root failure gets rendered as one red node with greyed dependents
rather than nine red alerts.

`critical: false` is for the things that are allowed to be down at 3am.

---

## 8. `hosts` — machine health

```jsonc
{
  "id": "gpu-01",                // REQUIRED
  "name": "gpu-01",              // REQUIRED
  "status": "up",                // REQUIRED. up | degraded | down | unknown
  "role": "inference",           // free text: inference | polling | ci | vps | mail
  "lastSeenAt": "2026-08-21T17:58:00.000Z",
  "uptimeSeconds": 918273,
  "cpuPct": 12.5,
  "memPct": 61.0,
  "loadAvg": [1.2, 1.1, 0.9],
  "disks": [
    { "mount": "/",     "usedPct": 96.1, "freeBytes": 8123456789 },
    { "mount": "/data", "usedPct": 3.2,  "freeBytes": 3891234567890 }
  ],
  "gpus": [
    { "name": "NVIDIA RTX 4090", "utilPct": 88, "memUsedMb": 21000,
      "memTotalMb": 24564, "tempC": 71, "powerW": 420 }
  ]
}
```

Disks are a list because the interesting fact is usually about one mount, not the
box. A root filesystem at 96% while a 4 TB data volume sits empty is the normal
way a machine dies, and a single averaged `diskPct` hides exactly that.

GPUs are first-class for anyone running their own inference, because on those
machines the GPU is the thing that runs out.

A host that hasn't reported in is `unknown`, not `down`. The dashboard decides
what to do about a stale `lastSeenAt`; the business shouldn't guess.

---

## 9. `incidents`, `inbox`, `costs`, `vendors`, `deployments`, `extra`, `errors`

### 9.1 `incidents` — issues and outages

```jsonc
{
  "id": "ingest-failure:northwind",        // REQUIRED. STABLE across polls
  "title": "Northwind ingest returned 0 rows for 3 consecutive runs",
  "severity": "warn",            // REQUIRED. info | warn | critical
  "status": "open",              // REQUIRED. open | acknowledged | resolved
  "openedAt": "2026-08-20T05:00:00.000Z",
  "resolvedAt": null,
  "source": "monitoring",        // monitoring | deploy | cron | manual | external
  "service": "ingest",
  "count": 3,                    // occurrences since openedAt
  "detail": "…",
  "url": "https://…"
}
```

`id` must be **stable across polls** — the same underlying problem must produce
the same id every minute, or the dashboard shows you the same outage 1,440 times
a day and you stop reading it. Key it on the thing that is broken
(`ingest-failure:northwind`), never on a timestamp or a random id.

Resolved incidents may stay in the list for a while so the UI can show a
recently-resolved strip. Drop them after 24h.

### 9.2 `inbox` — customer email and support

```jsonc
{
  "id": "msg-8f21",              // REQUIRED. stable
  "subject": "Can't cancel my subscription",   // REQUIRED
  "receivedAt": "2026-08-21T14:02:00.000Z",    // REQUIRED
  "from": "j****@example.com",
  "fromName": "Jane",
  "status": "unread",            // unread | open | waiting | closed
  "priority": "high",            // low | normal | high
  "snippet": "First 200 characters, plain text…",
  "tags": ["billing"],
  "url": "https://mail…",        // deep link to the real thread
  "customerId": "…"              // join key into your own admin
}
```

**Never put full message bodies in the report.** A snippet plus a deep link is
enough to triage, and the report is a document that gets cached, logged and
snapshotted into a database on a different machine. Same reasoning for
attachments (don't), and for anything a customer sent you that you would not
want sitting in a backup of the dashboard's own history.

Cap at the 50 most recent items that need attention. This is a triage queue, not
a mail client.

### 9.3 `costs`

```jsonc
"costs": {
  "window": "30d",               // REQUIRED
  "currency": "usd",             // REQUIRED. always "usd" today
  "totalCents": 412300,          // REQUIRED
  "previousTotalCents": 388000,
  "projectedMonthEndCents": 450000,
  "asOf": "2026-08-21T06:00:00.000Z",
  "lines": [
    { "id": "cloud.compute", "label": "Cloud compute", "vendor": "cloud",
      "category": "compute", "cents": 12000, "previousCents": 9000 }
  ]
}
```

Categories: `compute`, `storage`, `network`, `ai`, `saas`, `proxy`, `payments`,
`marketing`, `other`.

`cost.total` in `metrics` and `costs.totalCents` must agree. The metric is what
graphs and alerts; `costs` is the breakdown behind it.

Cloud billing lags by up to a day. Set `asOf` honestly and let the UI show
"as of yesterday 06:00" rather than implying live spend.

### 9.4 `vendors` — external accounts and what is left in them

```jsonc
{
  "id": "proxy-provider",        // REQUIRED. joins to costs.lines[].vendor
  "name": "Proxy provider (residential)",  // REQUIRED
  "billing": "prepaid",          // REQUIRED. prepaid | quota | postpaid | free
  "status": "warn",              // ok | warn | crit | unknown
  "category": "proxy",
  "balance": { "value": 41000000000, "total": 200000000000, "unit": "bytes" },
  "resetsAt": null,              // prepaid: nothing resets
  "renewsAt": null,
  "burn": { "window": "7d", "value": 63000000000 },
  "exhaustsAt": "2026-08-25T00:00:00.000Z",
  "planCents": 14500,
  "accountUrl": "https://example.com/dashboard",
  "asOf": "2026-08-21T17:59:00.000Z",
  "note": "Pay-as-you-go. At zero the gateway stops serving until it is topped up."
}
```

#### `billing` is the field that carries the meaning

The two exhaustion cases are genuinely different, and the difference is what the
section exists to capture:

- **`prepaid`** — the balance drains and does not come back. A prepaid traffic
  balance stops serving the moment it hits zero and stays stopped until someone
  tops it up. **That is an outage whose fix is a credit card**, and it deserves
  to page you days ahead of time.
- **`quota`** — the allowance resets every cycle. A monthly bandwidth quota
  exhausted on the 28th is degraded service until the 1st. Bad, bounded,
  self-healing.
- **`postpaid`** — no balance; spend just accrues. Most cloud providers. There is
  nothing to run out of, so `balance` is forbidden rather than optional — a
  balance on a postpaid account means the billing mode is wrong.
- **`free`** — no money involved. Tracked so a free-tier limit is visible.

Ranking every external account by "days until zero" only works if the ones that
never reset are distinguishable from the ones that do. Without `billing` you get
one list where half the entries are urgent and half are noise, which is the same
as having no list.

#### `exhaustsAt` and who computes it

Omit it and **the dashboard projects it from its own recorded balance history**,
which is usually the better estimate — it has the real curve across weeks
instead of one window's average. Send it when the vendor's own API projects it,
or when burn is spiky in a way a linear fit gets wrong.

`null` means "not projected to run out". That is different from omitted.

The comparison that matters is `exhaustsAt` against `resetsAt`: a quota that
exhausts *after* it resets is fine forever, and a quota that exhausts before is
this month's problem. For a prepaid account there is no `resetsAt`, so
`exhaustsAt` stands alone and is simply a deadline.

#### Money lives in `costs`, not here

`costs.lines[].vendor` joins to `vendors[].id`. Spend and balance land on one
row in the UI without either section duplicating — and therefore contradicting —
the other. `planCents` is the exception: it is the recurring price of the
subscription, not spend in a window, and it is what tells you a $180/mo proxy
plan is being 18% used.

Not every vendor has a cost line (a free tier), and not every cost line has a
vendor — electricity for a machine you own is real spend with no account, and it
simply omits the field. But a `vendor` that is **set** must join to a
`vendors[].id`. A typo'd key is not a harmless mismatch: it splits that vendor's
spend from its balance and the UI renders both halves as though each were whole.

#### Roll it into `status`

A prepaid account projected to hit zero within 72 hours is a `warn` at minimum.
`vendors.at_risk` is the registry metric that counts them, so the number is
graphable and alertable rather than something you notice by scrolling.

### 9.5 `deployments` — what is actually running

```jsonc
{
  "component": "ingest-worker",            // REQUIRED. stable id
  "env": "prod",                           // REQUIRED
  "repo": "acme/ingest",                   // REQUIRED. owner/name
  "status": "healthy",           // healthy | degraded | failed | deploying | unknown
  "ref": "main",
  "sha": "f48f630b…",
  "version": "0.0.181",
  "deployedAt": "2026-08-19T21:04:42.000Z",
  "url": "https://…",
  "previous": { "sha": "…", "version": "0.0.180", "deployedAt": "…" }
}
```

**This section reports what is running, and nothing else.** It does not describe
what *should* ship next, and it does not carry merge-ordering rules.

Ordering constraints belong in the dashboard's own configuration, not in this
document, for two reasons. They are cross-repo knowledge that no single business
owns — "the collector's prod deploy must land before the API's" is a fact about
a pair of repositories, and neither of them is the right place to keep it. And
they have to be trustworthy exactly when a service is too broken to answer its
endpoint, which is the moment you most need to know what order to ship things in.

`previous` is what a rollback would return you to. It is what makes a rollback
button able to say *where* it is rolling back to before you press it.

### 9.6 `extra`

```jsonc
"extra": {
  "Model": { "name": "ranker v1.1", "accuracy": 0.941 },
  "Bandwidth": { "residentialGb": 412, "datacenterGb": 88 }
}
```

Free-form. Keys are display labels; values are any JSON. The dashboard renders
them generically — scalars as rows, arrays of flat objects as tables — under a
collapsed "Details" panel.

Use it for things that genuinely don't fit, and when something in `extra` turns
out to be one of the first things you look at every morning, promote it to a
custom metric so it graphs.

### 9.7 `errors` — partial failure

```jsonc
"errors": [
  { "section": "costs", "message": "Cost query timed out after 5s",
    "at": "2026-08-21T17:59:58.000Z" }
]
```

Present means the report is incomplete and says so. The dashboard renders a
banner naming the sections it could not get, and keeps showing the last good
value for those, marked stale.

---

## 10. Limits

| Thing | Cap | Over the cap |
|---|---|---|
| Whole document | 1 MB | Dashboard rejects, logs, keeps last good. |
| `metrics` | 200 | Truncated. |
| `series` | 50 series × 400 points | Truncated. |
| `services` | 100 | Truncated. |
| `hosts` | 50 | Truncated. |
| `incidents` | 100 | Truncated, `critical` first. |
| `inbox` | 50 | Truncated, most recent first. |
| `costs.lines` | 100 | Truncated. |
| `vendors` | 100 | Truncated, `crit` first. |
| `deployments` | 50 | Truncated. |
| Any string field | 2000 chars | Truncated. |

Sort before you truncate — send the 50 inbox items that matter, not the 50 the
query happened to return first.

---

## 11. Security

The report is a complete operational picture of a company behind one bearer
token, and it is fetched by a machine that holds the same for every other
company. The contract:

1. **Bearer token, required, always.** `Authorization: Bearer <token>`. Minimum
   32 characters. Compared in constant time.
2. **Fail closed when unconfigured.** No token in the environment means *deny
   every request*, never "skip the check". The alternative is the shape of bug
   that ships inside a container whose env file didn't mount and looks perfectly
   healthy while being wide open. Write that check once and copy it between
   businesses rather than reimplementing it per stack.
3. **One token per business**, generated by the dashboard, rotatable without
   coordinating across companies — and separate from any existing service
   credential. A credential that authorizes a service to spend money should not
   be the same string that authorizes a reader to look at a chart.
4. **Network restriction on top of the token, wherever the deployment allows
   it.** Bind to an internal interface, or allowlist the dashboard host's source
   IP, or reach a loopback-bound listener through a tunnel. For services that
   are inherently internet-facing — a function behind a managed API gateway —
   the token is the primary control and a source-IP condition on the resource
   policy is the secondary one. Be precise about which of those you actually
   applied rather than assuming a network rule you did not.
5. **No secrets in the payload.** No API keys, no tokens, no passwords, no
   connection strings, no full email bodies, no customer documents. Mask
   customer email addresses and put the real address behind the deep link. The
   report is cached, logged, and written to a database on another machine;
   anything in it is in all three places, and a backup of the third is a backup
   of everything you ever put in a report.
6. **`Cache-Control: no-store`** on the response.
7. **Rate limit** to something like 10 req/min. The dashboard polls once a
   minute; anything more is a bug or an intruder.

---

## 12. Versioning

`spec` is `"business-report/<major>"`. The dashboard accepts the current major
and the one before it.

**Additive changes are free within a major**: new optional fields, new registry
metric ids, new enum values. Consumers ignore unknown keys and treat unknown
enum values as `unknown`, so adding is never breaking.

**A major bump is required for**: removing or renaming a field, changing a
field's type or unit, making an optional field required, or changing the meaning
of an existing metric id. Retire a metric id rather than redefining it — a
silently-redefined metric produces a chart that lies about last month.

During a major transition a business may serve both from the same handler and
pick on `?spec=`, but the simpler path is that the dashboard supports both and
businesses migrate whenever.

---

## 13. Conformance

A conformance checker takes a report — from a file, or from a live endpoint with
a token — and runs four passes in order. The first three apply to any document;
the fourth needs a live URL.

**1. Valid.** The document matches §2–§9: required fields present, enums known,
money integral, ratios within 0–1 unless `signed`, `unitLabel` present whenever
`unit` is `other`, no `balance` on a `postpaid` or `free` vendor and one present
on every `prepaid` or `quota` vendor. *Failures are errors.*

**2. Consistent.** The document agrees with itself:

- `cost.total` matches `costs.totalCents`, over the same `window`.
- Every `series[].metricId` resolves to a metric in the same document.
- Every `metrics[].service` and `incidents[].service` resolves to a service.
- Every `services[].dependsOn` id resolves to a service.
- Every `costs.lines[].vendor` that is set resolves to a vendor.
- `vendors.at_risk` equals the number of vendors actually projected to hit zero
  before anything refills them, and no such vendor reports `status: ok`.
- Registry ids carry their registry unit and direction.
- Custom metric ids are namespaced with the business id.
- `series` points are sorted ascending.
- Ids are unique within their list.

*Failures are errors.* Each of these is a case where the UI would otherwise
render something confidently wrong rather than visibly broken.

**3. Complete.** Recommended registry metrics are present, `direction` is set
wherever there is a `previous`, counters carry a `window`, featured metrics have
a series to draw on day one, and `generatedAt` is actually recent. *Failures are
warnings* — a business with no revenue is not a malformed business.

**4. Live.** Against a real endpoint: responds under 2 s, sets
`Cache-Control: no-store`, and — the check that matters — a request with **no**
token and a request with a **wrong** token both get `401`. Schema validation
will never catch a fail-open endpoint; this is the only pass that will.

Exit non-zero on any error and zero on warnings alone, so the checker can gate a
PR in each business's own CI without blocking on soft advice. Run it there, and
a refactor that drops a metric fails a pull request instead of quietly blanking
a chart.
