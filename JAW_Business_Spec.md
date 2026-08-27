# JAW Business Reporting Spec — `jaw-business-report/1`

One authenticated endpoint per business. One JSON document. One shape, whatever
the business does.

Four rules the rest of this document follows from:

1. **One endpoint, one document.** No second route, no mutation, no fan-out.
2. **Point in time only.** The report says what is true *now*. It never carries
   history, trends, or deltas. Storing snapshots and drawing graphs is the
   consumer's job.
3. **Incremental where it matters.** The consumer sends back the cursor it got
   last time. Anything that is a stream of things that happened — errors,
   events, mail, CI failures — returns only what is new since that cursor.
4. **Every number has the same shape.** One `metrics[]` list, labelled by what
   the number is about. A dashboard renders any business without knowing what
   the business does.

The goal is a document rich enough that a dashboard — or a local model reading
it every minute — can tell something is wrong: spend climbing, a cron that
processed 212,000 rows instead of 20,000, memory creeping in one service, a
domain expiring, a filing overdue, a GPU throttling, support mail piling up.

**Status:** v1. **License:** Apache-2.0.

---

## 1. The endpoint

```
GET  {base}/_dashboard/report?since=2026-08-26T17:04:03.114Z
Authorization: Bearer <token>
```

Returns `200 application/json` with a Business Report.

`GET` only. It gets polled forever; a retry must be free.

### Parameters

| Param | Meaning |
|---|---|
| `since` | ISO-8601 instant. One watermark for every stream. Use on the first call, or from a consumer that keeps one clock. |
| `cursor` | URL-encoded JSON: the `cursor.streams` object from the previous response. Per-stream watermarks. Wins over `since` for any stream it names. |
| `sections` | Comma-separated top-level keys to return. Optional; a business may ignore it. A filtered response still returns a complete `cursor` (§2, rule 5). |

`since` and `cursor` are the whole request contract; what to collect is the
business's decision.

No `since` and no `cursor` means a first call: return current state plus a
bounded backfill for the streams — 24 hours by default — and say how far back
you went in `cursor.from`.

### Response codes

| Code | When | Consumer does |
|---|---|---|
| `200` | A report was produced, even a partial one. | Render it. `unavailable[]` becomes a banner. |
| `401` | Missing or bad token. | Red tile. No retry storm. |
| `403` | The caller is not on the allowlist (§27). The token was never read. | Red tile, worded differently from `401`. Do not rotate the token. |
| `429` | Rate limited. | Back off, honour `Retry-After`. |
| `503` | No report is possible at all. | Red tile, keep the last snapshot, mark it stale. |

**Prefer `200` with `unavailable[]` over `503`.** One dead sub-query should cost
one section, not the page.

### Timing

- Under **2 s** at p95. Consumer timeout is 10 s.
- Default poll **60 s**.
- `Cache-Control: no-store`. Compress the body; send `Retry-After` with every `429`.
- Expensive to assemble? Serve a precomputed snapshot and set
  `business.ttlSeconds` to how stale it may be. The consumer shows the age
  instead of implying the number is live.

---

## 2. Cursors

Sections are one of two kinds, and the kind decides how `since` applies.

| Kind | Sections | `since` |
|---|---|---|
| **Snapshot** | `metrics`, `services`, `vendors`, `accounts`, `hosts`, `endpoints`, `sources`, `funnel`, `apis`, `jobs`, `deployments`, `domains`, `compliance`, `incidents` | Ignored. Always current and complete. |
| **Stream** | `events`, `errors`, `inbox`, `issues`, `ci` | Return only items after the watermark. |

Each stream is ordered by one timestamp field. That field is what "after the
cursor" means, and what a checker compares against:

| Stream | Field |
|---|---|
| `events` | `at` |
| `errors` | `lastSeenAt` — a bucket still happening returns again each poll, counting only since the cursor |
| `inbox` | `receivedAt` |
| `issues` | `updatedAt` — an issue returns when it is commented on, not only when opened |
| `ci` | `startedAt` |

### The response reports what it covered

```jsonc
"cursor": {
  "requestedSince": "2026-08-26T17:04:03.114Z",  // echoed, or null on a first call
  "from": "2026-08-26T17:04:03.114Z",            // start of what this response covers
  "streams": {
    "events": "2026-08-26T18:03:58.902Z",
    "errors": "2026-08-26T18:03:59.417Z",
    "inbox":  "2026-08-26T18:03:57.001Z",
    "issues": "2026-08-26T18:03:52.660Z",
    "ci":     "2026-08-26T18:03:59.980Z"
  },
  "complete": true
}
```

The consumer stores `cursor.streams` verbatim and sends it back as `cursor` on
the next request. It never invents a watermark from its own clock.

`from` is where coverage begins: the requested cursor, or the backfill start on a
first call. Without it a first response cannot say whether it carried 24 hours or
20 minutes.

### Rules

1. **Per-stream watermarks, not one.** Each stream's query finishes at a
   different instant. A single "now" taken when the response is serialized
   silently drops everything written to the first stream while the last one was
   still running.
2. **A watermark is the latest instant that stream can guarantee is fully
   visible** — normally when that query finished, earlier if writes can commit
   out of order. Under-reporting the watermark costs a duplicate. Over-reporting
   costs a permanent gap. Choose duplicates.
3. **Delivery is at-least-once, and `id` means one of two things.**
   - `events`, `inbox`, `ci` — one `id` per item. A repeat is a duplicate; drop it.
   - `errors`, `issues` — `id` names a bucket or an issue, and it comes back every
     time that thing changes. Upsert on `id`. What separates a duplicate from an
     update is `(id, lastSeenAt)` and `(id, updatedAt)`.

   Never add an error bucket's `count` to what you already stored for the same
   `lastSeenAt`. Rule 2 makes duplicates deliberate, and summing them inflates the
   number every time one arrives.
4. **Truncation is not loss — except for `errors`.** A stream over its cap returns
   the oldest items up to the cap, sets that stream's watermark to the last item
   returned, and sets `cursor.complete: false`. The consumer polls again
   immediately rather than waiting for the next tick.

   `errors` is ranked by `count`, not by time (§18), so the buckets it drops are
   not the newest ones and polling again will not bring them back. It advances its
   watermark as usual, sets `complete: false`, and declares the overflow in
   `unavailable[]`. Ranked truncation loses the tail; say so.
5. **`sections` never shortens the cursor.** A filtered response returns
   `cursor.streams` for every stream the business supports, echoing the requested
   watermark unchanged for any stream it did not collect. A cursor with keys
   missing overwrites what the consumer stored, and that is a silent gap.

### If a stream lives only in memory

Most do not. If a stream is durably stored and queryable by timestamp — rows in
a table, events in an index — answer from the query and skip this. A producer
that cannot keep state between requests at all, such as a serverless function,
has no buffer available and must persist its streams somewhere it can query.

For anything held in memory, such as error counters in a long-lived process:

- **Flush on acknowledgement, not on read.** Discard what is older than a cursor
  the consumer has *since sent back*. Discarding on read loses the buffer
  whenever the consumer fails to store the response it just drained.
- **Bound the buffer anyway.** If it overflows before acknowledgement, drop
  oldest and say so in `unavailable[]`. A silent gap is the one failure this
  design exists to prevent.

---

## 3. Document shape

```jsonc
{
  "spec": "jaw-business-report/1",  // REQUIRED, exact string
  "cursor":   { … },            // REQUIRED
  "business": { … },            // REQUIRED
  "status":   { … },            // REQUIRED

  "metrics":     [ … ],   // every number, point in time            §6
  "sources":     [ … ],   // where customers come from              §7
  "funnel":      [ … ],   // the stages they pass through           §8
  "services":    [ … ],   // what we run                            §9
  "vendors":     [ … ],   // what we pay for                        §10
  "accounts":    [ … ],   // what we hold, and what it owes  §11
  "hosts":       [ … ],   // machines we are responsible for        §12
  "endpoints":   [ … ],   // URLs under health check               §13
  "apis":        [ … ],   // every API surface and operation       §14
  "jobs":        [ … ],   // crons and batches, with normal ranges  §15
  "deployments": [ … ],   // what is running                       §16

  "events":    [ … ],     // stream: notable things that happened  §17
  "errors":    [ … ],     // stream: bucketed application errors   §18
  "inbox":     [ … ],     // stream: new customer mail             §19
  "issues":    [ … ],     // stream: public repo issues            §20
  "ci":        [ … ],     // stream: failed CI and test runs       §21

  "incidents":  [ … ],    // open and recently resolved            §22
  "domains":    [ … ],    // registrations and expiry              §23
  "compliance": [ … ],    // filings, registrations, insurance     §24

  "extra":       { … },   // free-form                             §25
  "unavailable": [ … ]    // sections this report could not produce §25
}
```

Four required fields. Everything else is optional, so a business can ship a
conforming report in an afternoon and grow it.

**Where a number comes from is not this document's business.** A live query, a
nightly cron, a hand-maintained YAML — all legitimate. `asOf` is what tells the
consumer how far to trust it. A renewal date typed in by hand with an honest
`asOf` is worth more than an omitted section, and most businesses will hand-
maintain their vendor plans, domains, and filings for a long time before any of
it has an API behind it.

Unknown top-level keys are ignored, not rejected. Unknown enum values are read
as `"unknown"`.

### Conventions every section shares

**Ids** are stable, unique within their section, at most 128 characters, and
printable ASCII with no newlines and no leading or trailing space. They are join
keys in the consumer's database, so a renamed id is a new object with no
history.

**Timestamps** are RFC 3339, UTC, milliseconds, `Z` —
`2026-08-26T18:04:00.112Z`. Watermarks get compared as strings, and one local
offset breaks that.

**Derived fields** — `daysRemaining` and its like — are conveniences computed from
the absolute date beside them. Where the two disagree, the absolute date wins.

**Two enums, everywhere:**

| | Values | Used by |
|---|---|---|
| Availability | `up` · `degraded` · `down` · `unknown` | `services`, `hosts`, `endpoints`, `apis`, `deployments` (which adds `deploying`) |
| Severity | `ok` · `warn` · `crit` · `unknown` | `metrics[].severity`, `vendors[].status`, `events[].severity`, `incidents[].severity` |

`status.level` (§5) is the one exception: it adds `down` for "customers are
affected right now". On an event or an incident, `ok` is where you would
otherwise have written "info". `errors[].level` is the log level your tracker
already assigned, not this scale.

---

## 4. `business`

```jsonc
"business": {
  "id": "acme",                              // REQUIRED. stable slug [a-z0-9-]
  "name": "Acme Corp",                       // REQUIRED
  "env": "prod",                             // REQUIRED. prod | dev | staging
  "generatedAt": "2026-08-26T18:04:00.112Z", // REQUIRED. when this document was assembled
  "ttlSeconds": 60,                          // how stale the data underneath may be
  "revision": "a302ff66",                    // commit that produced it
  "links": [
    { "label": "Admin", "url": "https://admin.acme.example" },
    { "label": "Status", "url": "https://status.acme.example" }
  ]
}
```

`id` is the join key for everything the consumer stores. Never change it.

`generatedAt` is when the document was assembled. If a number inside is older,
say so on that item with `asOf`. Do not backdate `generatedAt`.

---

## 5. `status`

```jsonc
"status": {
  "level": "warn",                              // REQUIRED. ok | warn | down | unknown
  "summary": "Ingest ran 10x normal volume. Proxy balance exhausts in 3 days.",
  "since": "2026-08-26T04:12:00.000Z"
}
```

**The business computes this.** Only it knows that a worker showing DOWN was
drained on purpose this morning. A generic rule over generic fields gets that
wrong and trains you to ignore the page.

`down` = customers affected now. `warn` = a human today. `ok` = go do something
else.

---

## 6. `metrics`

Every number in the report is a metric. Same shape for revenue, VRAM, and
support response time.

```jsonc
{
  "id": "usage.latency_p95",   // REQUIRED. registry id (§6.2) or namespaced custom
  "label": "API p95 latency",  // REQUIRED. the UI never needs to know what an id means
  "value": 412,                // REQUIRED. number, or null for "not available"
  "unit": "ms",                // REQUIRED. §6.3
  "kind": "gauge",             // gauge | counter                (default gauge)
  "window": "5m",              // REQUIRED for counters: what period the value covers
  "service": "api",            // scope: §6.1
  "instance": null,            // dimension within a scope: a mount, a GPU, a queue
  "outcome": null,             // success | failure — split a metric by result
  "group": "Usage",            // free text; the UI groups tiles by it
  "direction": "down_good",    // up_good | down_good | neutral  (default neutral)
  "severity": "ok",            // ok | warn | crit
  "target": 300,               // goal, drawn as a reference line
  "expected": { "min": 80, "max": 600 },  // normal range. outside it is an anomaly
  "featured": true,            // top row of the business page
  "signed": false,             // may go negative — only relaxes the ratio floor
  "asOf": "2026-08-26T18:03:12.000Z",
  "note": "Excludes the /export route."
}
```

**No history.** No `previous`, no series, no deltas. The consumer stores every
snapshot and does its own arithmetic. A business that already has a rollup table
does not need to serve it — the consumer will have the same curve within a day.

**`expected` is how anomalies get caught.** The producer knows what normal looks
like; the consumer does not, on day one. A cron that normally moves 20,000 rows
declares `expected: {min: 15000, max: 25000}`, and 212,000 is then a fact the
dashboard can act on rather than a number a human has to recognize. Omit it and
the consumer learns the band from its own history; declare it when you already
know it, or when day-one alerting matters more than a week of watching.

**`direction`** tells the UI whether green is up or down. Revenue up is good,
cost up is bad, latency up is bad. Set it on everything.

### 6.1 Scope

A metric may name what it is about. All are optional; more than one may be set.

| Field | Points at | Example |
|---|---|---|
| `service` | `services[].id` | memory used by the API |
| `host` | `hosts[].id` | CPU on `gpu-01` |
| `vendor` | `vendors[].id` | requests billed to the LLM provider |
| `job` | `jobs[].id` | rows processed by the nightly ingest |
| `endpoint` | `endpoints[].id` | check latency for the health URL |
| `api` | `apis[].id` | calls to `POST /v1/solve` |
| `source` | `sources[].id` | signups from the newsletter |
| `stage` | `funnel[].id` | people who finished onboarding |
| `instance` | free text, within the scope above | `/data`, `gpu0`, `queue:apply` |
| `outcome` | `success` \| `failure` | attempts that succeeded vs failed |

Unscoped means business-wide. `revenue.net` with no scope is the company's;
`revenue.net` with `service: "api"` is that product line's.

**Identity is `id`, `window`, and every scope label above**, and it must be
unique in the document. One id and scope may carry only one window per report —
`revenue.gross` at `30d` and at `mtd` are two rows, and a consumer that ignores
`window` overwrites one with the other. Everything else — which services are
busy, which are idle, which earn money, which leak memory — is a group-by on the
consumer's side.

### 6.2 Registry

If the business has the concept, it uses the registry id. That is what makes
"revenue across all companies" a chart instead of a research project. Anything
else is custom and must be namespaced with the business id: `acme.solve.accuracy`.

Reporting a registry id with a different unit or direction is an error, not a
style choice — `cost.total` with `direction: up_good` paints rising spend green.

**Revenue and payments** — `usd_cents`, unless noted.

| id | kind | direction | meaning |
|---|---|---|---|
| `revenue.mrr` | gauge | up_good | Recurring, normalized to a month. |
| `revenue.gross` | counter | up_good | Collected in the window, before fees. |
| `revenue.net` | counter | up_good | After processor fees and refunds. |
| `revenue.arpu` | gauge | up_good | Net revenue ÷ paying users, per month. |
| `revenue.pending_payout` | gauge | neutral | Owed out, not yet paid. |
| `payments.count` (`count`) | counter | up_good | Successful charges. |
| `payments.failed` (`count`) | counter | down_good | Declined or errored charges. |
| `payments.refunds` | counter | down_good | Refunded in the window. |
| `payments.disputed` | counter | down_good | Chargebacks opened in the window. |

**Users** — all `count`.

| id | kind | direction | meaning |
|---|---|---|---|
| `users.total` | gauge | up_good | Registered accounts, ever. |
| `users.new` | counter | up_good | Signed up in the window. |
| `users.active` | gauge | up_good | Did the core action in the window. |
| `users.paying` | gauge | up_good | On a paid plan or holding a balance. |
| `users.trial` | gauge | neutral | In trial. |
| `users.delinquent` | gauge | down_good | Payment failed, not yet cancelled. |
| `users.suspended` | gauge | down_good | Blocked, banned, or over limit. |
| `users.churned` | counter | down_good | Lapsed in the window. |
| `users.waitlist` | gauge | up_good | Waiting for access. |

**Engagement** — how hard the people you have are using the thing.

| id | unit | kind | direction | meaning |
|---|---|---|---|---|
| `engagement.dau` | `count` | gauge | up_good | Distinct users active today. |
| `engagement.wau` | `count` | gauge | up_good | Last 7 days. |
| `engagement.mau` | `count` | gauge | up_good | Last 30 days. |
| `engagement.stickiness` | `ratio` | gauge | up_good | DAU ÷ MAU. |
| `engagement.sessions_per_user` | `count` | gauge | up_good | Per active user, in the window. |
| `engagement.actions_per_user` | `count` | gauge | up_good | Core actions per active user. |
| `engagement.session_seconds_p50` | `seconds` | gauge | up_good | Median session length. |
| `engagement.retention_d30` | `ratio` | gauge | up_good | Of a cohort 30 days old, still active. |

**Usage and speed** — scope to a service to compare services.

| id | unit | kind | direction | meaning |
|---|---|---|---|---|
| `usage.requests` | `count` | counter | up_good | Requests served in the window. |
| `usage.units` | `count`\|`credits`\|`tokens` | counter | up_good | Billable units. Unit is not pinned — a delivered job at one business, a credit at another. |
| `usage.success_rate` | `ratio` | gauge | up_good | Succeeded ÷ attempted. |
| `usage.error_rate` | `ratio` | gauge | down_good | Errored ÷ attempted. |
| `usage.latency_p50` | `ms` | gauge | down_good | Median end to end. |
| `usage.latency_p95` | `ms` | gauge | down_good | p95 end to end. |
| `usage.latency_p99` | `ms` | gauge | down_good | p99 end to end. |
| `usage.inflight` | `count` | gauge | neutral | Requests in flight right now. |
| `queue.depth` | `count` | gauge | down_good | Items waiting. |
| `queue.oldest_age` | `seconds` | gauge | down_good | Age of the oldest waiting item. |
| `queue.dlq_depth` | `count` | gauge | down_good | Dead-lettered items. |

**Reliability**

| id | unit | kind | direction | meaning |
|---|---|---|---|---|
| `uptime.window` | `ratio` | gauge | up_good | Fraction of the window up. Set `window`. |
| `uptime.outage_seconds` | `seconds` | counter | down_good | Time down in the window. |
| `incidents.open` | `count` | gauge | down_good | Open and unresolved. |
| `errors.count` | `count` | counter | down_good | Errors in the window. |
| `errors.buckets` | `count` | gauge | down_good | Distinct error groups in the window. |
| `jobs.late` | `count` | gauge | down_good | Scheduled jobs past due. |
| `jobs.failed` | `count` | counter | down_good | Job runs that failed in the window. |
| `ci.failed_runs` | `count` | counter | down_good | CI runs that failed in the window. |
| `ci.pass_rate` | `ratio` | gauge | up_good | Passed ÷ total runs. |

**Machines** — scope with `host`, `service`, and `instance`. This is where a
leak, a full disk, or a saturated NIC shows up.

| id | unit | kind | direction | meaning |
|---|---|---|---|---|
| `resource.cpu_pct` | `percent` | gauge | down_good | CPU. Scope by host, service, or task via `instance`. |
| `resource.load_1m` | `other` | gauge | down_good | Load average. `unitLabel: "load"`. |
| `resource.mem_bytes` | `bytes` | gauge | down_good | Resident memory. Per service is how a leak is seen. |
| `resource.mem_pct` | `percent` | gauge | down_good | Memory used of total. |
| `resource.swap_bytes` | `bytes` | gauge | down_good | Swap in use. |
| `resource.psi_mem` | `percent` | gauge | down_good | Memory pressure stalled time (PSI avg60). |
| `resource.psi_cpu` | `percent` | gauge | down_good | CPU pressure. |
| `resource.psi_io` | `percent` | gauge | down_good | I/O pressure. |
| `resource.vram_used_bytes` | `bytes` | gauge | down_good | VRAM in use. `instance` is the GPU. |
| `resource.vram_total_bytes` | `bytes` | gauge | neutral | VRAM fitted. |
| `resource.gpu_util_pct` | `percent` | gauge | neutral | GPU busy. |
| `resource.gpu_temp_c` | `other` | gauge | down_good | GPU temperature. `unitLabel: "°C"`. |
| `resource.gpu_power_w` | `other` | gauge | neutral | GPU draw. `unitLabel: "W"`. |
| `resource.disk_used_pct` | `percent` | gauge | down_good | Per mount, via `instance`. |
| `resource.disk_free_bytes` | `bytes` | gauge | up_good | Free on that mount. |
| `resource.disk_io_pct` | `percent` | gauge | down_good | Device utilization. |
| `resource.net_rx_bps` | `other` | gauge | neutral | Download. `unitLabel: "bit/s"`. |
| `resource.net_tx_bps` | `other` | gauge | neutral | Upload. `unitLabel: "bit/s"`. |
| `resource.sockets` | `count` | gauge | down_good | Open sockets or connections. |
| `resource.fds` | `count` | gauge | down_good | Open file descriptors. |
| `resource.threads` | `count` | gauge | down_good | Threads or goroutines. |
| `resource.restarts` | `count` | counter | down_good | Process restarts in the window. |
| `resource.processes` | `count` | gauge | neutral | Live processes we manage. Scope by host or service, and set `expected` — the right number is a known number. |
| `resource.orphans` | `count` | gauge | down_good | Processes matching a managed service that no supervisor owns: reparented to init when their parent died, or left behind by a previous release. Expected zero. |

**Data stores** — scope with `service` and `instance`, where `instance` is the
table or collection.

| id | unit | kind | direction | meaning |
|---|---|---|---|---|
| `db.rows` | `count` | gauge | neutral | Live row estimate. |
| `db.bytes` | `bytes` | gauge | neutral | Size on disk, table plus indexes. |
| `db.reads` | `count` | counter | neutral | Rows or read units consumed in the window. |
| `db.writes` | `count` | counter | neutral | Inserts, updates, and deletes in the window. |
| `db.seq_scans` | `count` | counter | down_good | Sequential scans. Climbing while `db.rows` is flat is a missing index. |
| `db.reads_per_request` | `count` | gauge | down_good | `db.reads` ÷ requests served. |
| `db.dead_rows` | `count` | gauge | down_good | Dead tuples awaiting vacuum. |
| `db.query_p95` | `ms` | gauge | down_good | p95 query time. |
| `db.connections` | `count` | gauge | down_good | Open connections, against the pool limit. |
| `db.replication_lag` | `seconds` | gauge | down_good | How far a replica trails. |

Take all of these from the database's own statistics view —
`pg_stat_user_tables`, `performance_schema`, the platform's per-table metrics —
never by querying the data. **`db.rows` is an estimate on purpose.**
`SELECT count(*)` is a full scan in most engines and gets slower every day,
while the live-row estimate sitting in the stats view is free and is accurate
enough to notice a table that stopped growing.

**`db.reads_per_request` is the one that catches a code change.** A raw read
rate rises with traffic, so a busy Tuesday and an accidental N+1 look identical.
Divide by requests and traffic cancels out: a deploy that turns one query into
forty moves this number and nothing else. Put an `expected` band on it, and do
the same per job run for anything batch.

One `instance` per table. A row per query shape belongs in your APM — the
cardinality is unbounded, and this document has a 1000-metric cap.

**Money out**

| id | unit | kind | direction | meaning |
|---|---|---|---|---|
| `cost.total` | `usd_cents` | counter | down_good | All spend in the window. |
| `cost.spend` | `usd_cents` | counter | down_good | Spend attributable to one label. Scope with `service`, `host`, `job`, or `vendor`, and give it a display `label`. |
| `cost.per_unit` | `usd_cents` | gauge | down_good | `cost.total` ÷ `usage.units`. |
| `cost.per_user` | `usd_cents` | gauge | down_good | `cost.total` ÷ active users. |
| `margin.gross` | `ratio` | gauge | up_good | (net revenue − cost) ÷ net revenue. Set `signed: true`. |
| `vendors.at_risk` | `count` | gauge | down_good | Accounts projected to hit zero before a reset, a renewal, or a working auto top-up refills them. |
| `vendors.topup_failed` | `count` | gauge | down_good | Accounts on auto top-up whose payment method is not `ok`, or whose last top-up failed. |
| `finance.cash` | `usd_cents` | gauge | up_good | Total across `accounts[]`. |
| `finance.burn_net` | `usd_cents` | counter | down_good | Money out minus money in, across accounts, in the window. Negative when cash grew. |
| `finance.runway_days` | `other` | gauge | up_good | Cash ÷ net burn. `unitLabel: "days"`. `null` while cash is growing. |
| `finance.unreconciled` | `usd_cents` | gauge | down_good | Account outflow that no tracked vendor accounts for. |

`cost.total` is everything. `cost.spend` rows are the breakdown, and there may
be several axes — by our service, by host, by job, by vendor. **Sum within one
label, never across.** The same dollar appears once per axis, so adding a
by-service row to a by-host row counts it twice.

The vendor axis is usually free, since bills arrive itemised. Attributing spend
to your *own* services needs resource tagging you may not have — an estimate
with a `note` beats an omission, and no breakdown at all is fine.

**Acquisition and funnel** — scope with `source` and `stage`.

| id | unit | kind | direction | meaning |
|---|---|---|---|---|
| `funnel.count` | `count` | gauge | up_good | People who reached this stage. Scope with `stage`, and with `source` for the cross-tab. |
| `funnel.conversion` | `ratio` | gauge | up_good | Reached this stage ÷ reached the one before it. |
| `funnel.time_to_convert_p50` | `seconds` | gauge | down_good | Median time to get from the previous stage to this one. |
| `acquisition.cac` | `usd_cents` | gauge | down_good | Spend on a source ÷ paying customers it produced. |
| `acquisition.ltv` | `usd_cents` | gauge | up_good | Net revenue expected from a customer, by source. |

**Support, code, and paperwork**

| id | unit | kind | direction | meaning |
|---|---|---|---|---|
| `inbox.unread` | `count` | gauge | down_good | Customer messages needing a reply. |
| `inbox.oldest_age` | `seconds` | gauge | down_good | Longest anyone has waited. |
| `inbox.response_p50` | `seconds` | gauge | down_good | Median first response time. |
| `issues.open` | `count` | gauge | neutral | Open issues on public repos. |
| `domains.expiring` | `count` | gauge | down_good | Domains with status `expiring` or `expired`, plus endpoints whose TLS certificate has 30 days or less left. |
| `compliance.due` | `count` | gauge | down_good | Entries in `compliance[]` with status `due` or `overdue`. |

#### Recommended

Five ids, and only these five, are what a checker warns about when they are
missing:

`users.total` · `usage.requests` · `usage.success_rate` · `incidents.open` ·
`cost.total`

Short deliberately, and **missing is a warning, not a failure**. Most businesses
will dismiss one or two forever — a business with no cost API cannot report
`cost.total` — which is the intended outcome. A warning you have read beats a
fabricated number that makes the cross-company chart lie. Nothing else in the
registry is expected of anyone.

### 6.3 Units

`count`, `usd_cents`, `ratio` (0–1), `percent` (0–100), `ms`, `seconds`,
`bytes`, `credits`, `tokens`, `other`.

- **Money is integer cents.** Never floats, never dollars. A cent of float drift
  is a payout that fails to reconcile, found during an audit.
- **`ratio` is 0–1, `percent` is 0–100.** Never mix. A "94" that might be 94% or
  9400% is a metric nobody trusts. The ceiling catches a percent posted into a
  ratio field, which would otherwise render as 8710%.
- **`signed: true`** relaxes the ratio floor, and only that. Gross margin goes
  below zero and most pre-launch months are there; rejecting it hides the months
  you most want. Every other unit may go negative without it — net burn is
  negative in a good month.
- **`unit: "other"` requires `unitLabel`** — short, like `"°C"` or `"jobs/hr"`.
- **`null` is "not available"**, which is not zero. Never send zero for missing.

### 6.4 Windows

`5m`, `1h`, `24h`, `7d`, `30d`, `90d`, `mtd` (month to date), `all` (since the
business began).

**`all` is the honest answer when a business holds only a lifetime figure** — a
user table with no signup date, a lifetime-spend column with no ledger behind
it. Report the lifetime total with `window: "all"` and let the consumer
difference its own snapshots into a rate. A 30-day number derived from a
lifetime total is fabricated, and it lands on the one chart that has to be
trusted.

`mtd` is what most billing APIs actually give you, so it is what most cost
figures should carry. Do not convert it to `30d` by arithmetic.

---

## 7. `sources` — where customers come from

One row per acquisition source. Traffic, signups, and revenue attach to it as
metrics scoped with `source` (§6.1).

```jsonc
{
  "id": "paid-search",             // REQUIRED. stable
  "name": "Paid search — launch",  // REQUIRED
  "kind": "paid",                  // REQUIRED. search | social | referral | direct
                                   // | paid | affiliate | email | partner | other
  "attribution": "last_touch",     // REQUIRED. first_touch | last_touch
                                   //           | linear | none
  "utm": {
    "source": "search",            // the utm_* tuple, as recorded
    "medium": "cpc",
    "campaign": "launch-q3",
    "content": null,
    "term": null
  },
  "referrerDomain": null,          // for referral traffic
  "paid": true,                    // does money go out for this source
  "active": true,                  // is it currently running
  "since": "2026-07-01T00:00:00.000Z",
  "note": "Attributed by the ad platform, so last touch. Organic rows are first touch."
}
```

### Rules

- **One row per source, not per URL.** Group by the utm tuple you actually plan
  to read. A row per landing page or per ad creative makes cardinality unbounded
  and the section unreadable, the same failure as concrete paths in §13.
- **State the attribution model on every row.** First touch and last touch
  produce different numbers from the same week, and a mixed set that does not say
  which is a set nobody can add up. Mixing is legitimate — ad platforms report
  last touch, your own database usually knows first touch — but say so per row.
- **`direct` is a real row, not a gap.** It is where everything unattributable
  lands, and its size is the honest measure of how much of this section to trust.
- **No per-person attribution.** No click ids, no visitor ids, no IPs, no
  landing-page URLs with identifiers in them. This section is counts by source;
  the individual journey stays in whatever tool recorded it.
- Spend on a paid source is `cost.spend` scoped with `source`, which is what
  makes `acquisition.cac` a number rather than a guess.

---

## 8. `funnel` — the stages between finding you and paying you

Stages in order. Counts attach as `funnel.count` scoped with `stage`, and the
cross-tab with `source` is what says which channel sends people who actually
convert.

```jsonc
{
  "id": "signup",                  // REQUIRED. stable
  "name": "Created an account",    // REQUIRED
  "position": 3,                   // REQUIRED. ascending, unique. conversion is
                                   //           measured against position - 1
  "kind": "signup",                // REQUIRED. impression | visit | signup
                                   // | activation | habit | paid | expansion
                                   // | other
  "basis": "cohort",               // REQUIRED. window | cohort
  "cohort": {                      // REQUIRED when basis is cohort
    "window": "30d",               // who entered the first cohort stage then
    "observedForDays": 30          // how long they have been watched since
  },
  "optional": false,               // may a person skip this stage
  "note": null
}
```

`kind` is a small fixed vocabulary so a dashboard can line up funnels from
businesses whose stages are named nothing alike. `name` stays yours.

### `basis` is the field that decides whether the rate means anything

- **`window`** — people who did this in the window. Cheap, current, and it mixes
  cohorts: the people who paid today are not the people who visited today.
- **`cohort`** — of the people who entered the first cohort-basis stage during
  `cohort.window`, how many have since reached this one. Correct, and it lags by
  however long conversion takes.

A funnel that quietly mixes the two produces conversion rates that are wrong in
a direction nobody notices — growing traffic makes window-basis conversion look
like it is falling, and shrinking traffic makes it look like it is improving.

**Stages before you can identify a person are window-basis by necessity.** You
cannot cohort an impression. Mark the boundary and treat the crossing rate as an
estimate: impressions-to-visits is a ratio of two populations, not a conversion.
Everything from the first identified stage onward should share one basis.

`funnel.time_to_convert_p50` on a stage is what tells you whether the cohort has
been watched long enough. A 30-day cohort reporting a paid conversion whose
median time to convert is 11 days is mature; one whose median is 40 days is
reporting a number that will keep rising after you read it.

### Rules

- Positions are unique and ascending. Gaps are fine; two stages sharing a
  position is not.
- `funnel.conversion` on a stage is against the previous position, and must agree
  with the two counts where both are present.
- Per-source counts never exceed the unscoped count for the same stage.
- Report the stages you have. Four honest stages beat six where two are guesses.

---

## 9. `services` — what we run

Identity, health, and shape. Every number about a service is a metric scoped to
its `id` (§6.1).

```jsonc
{
  "id": "api",                   // REQUIRED. stable within the business
  "name": "Public API",          // REQUIRED
  "status": "up",                // REQUIRED. up | degraded | down | unknown
  "kind": "api",                 // api | worker | inference | db | queue | cron
                                 // | frontend | mobile | external | other
  "parent": null,                // §9.1 — id of the service this belongs to
  "critical": true,              // does down here mean customers are affected?
  "serverless": false,           // true = no host to report resources for
  "hosts": ["app-01"],           // hosts[].id it runs on
  "dependsOn": ["postgres", "queue"],   // other services[].id
  "since": "2026-08-24T09:00:00.000Z",  // when status last changed
  "startedAt": "2026-08-24T09:00:00.000Z", // last start. resets leak baselines
  "version": "1.4.2",
  "message": "3/4 workers active; one drained for maintenance.",
  "checkedAt": "2026-08-26T18:03:58.000Z",
  "url": "https://api.acme.example",
  "env": "prod"
}
```

`dependsOn` draws the flow graph, so one root failure renders as one red node
with greyed dependents instead of nine red alerts. `critical: false` is for what
is allowed to be down at 3am. `startedAt` is what lets a consumer tell a memory
leak from a restart sawtooth.

### 9.1 Sub-services

`parent` nests a service inside another. Use it when one logical service is
really N of the same thing and you want both the roll-up and the parts.

A collection service with one worker per upstream source:

```jsonc
{ "id": "ingest",            "name": "Data collection", "kind": "worker", "status": "degraded",
  "critical": true, "message": "1 of 4 sources failing" },

{ "id": "ingest.source-a",   "name": "Source A", "kind": "worker", "parent": "ingest",
  "status": "up",   "critical": false },
{ "id": "ingest.source-b",   "name": "Source B", "kind": "worker", "parent": "ingest",
  "status": "down", "critical": false,
  "message": "0 rows for 3 consecutive runs; selector change suspected" },
{ "id": "ingest.source-c",   "name": "Source C", "kind": "worker", "parent": "ingest",
  "status": "up",   "critical": false },
{ "id": "ingest.source-d",   "name": "Source D", "kind": "worker", "parent": "ingest",
  "status": "up",   "critical": false }
```

Rules:

- `parent` must be another `services[].id` in the same document. No cycles.
- Any depth is legal. Past three levels, a dashboard stops being readable.
- **The parent's `status` is the business's own roll-up**, not a computed
  maximum. One dead source out of forty may be `degraded`; one dead source out
  of two is `down`. Only the business knows which.
- **Metrics attach to the level they describe.** `usage.requests` on
  `ingest.source-b` is that source's. The same id on `ingest` is the parent's own
  total. Do not double count: emit the children, or emit the parent, and let the
  consumer sum — say which in `note` if it is not obvious.
- `dependsOn` is for things a service *calls*. `parent` is for what it is *part
  of*. A child may also `dependsOn` a service outside its parent.
- **A child inherits `hosts` and `serverless` from its parent** unless it sets
  its own. Four workers in one process do not each repeat the machine.

---

## 10. `vendors` — what we pay for

One entry per external account: spend, usage, what is left, and when the period
rolls. This is the section that answers "are expenditures getting out of hand".

```jsonc
{
  "id": "proxy",                       // REQUIRED. stable
  "name": "Residential proxy",         // REQUIRED
  "billing": "prepaid",                // REQUIRED. prepaid | quota | postpaid | free
  "status": "warn",                    // ok | warn | crit | unknown
  "category": "proxy",                 // compute | storage | network | ai | saas
                                       // | proxy | payments | marketing | legal | other
  "parent": null,                      // id of a parent vendor, for line items
  "services": ["ingest"],              // services[].id that depend on this account
  "period": {
    "start": "2026-08-01T00:00:00.000Z",
    "end":   "2026-08-31T23:59:59.000Z",
    "renewsAt": null,                  // when the subscription next charges
    "resetsAt": null                   // when a quota refills. prepaid: null
  },
  "plan": { "name": "Pay as you go", "cents": null, "interval": null },
                                       // interval: monthly | yearly | none
  "spend": {
    "periodToDateCents": 4500,
    "projectedPeriodCents": 6000,
    "lastInvoiceCents": 12000,
    "asOf": "2026-08-26T06:00:00.000Z"
  },
  "usage": {
    "used": 159000000000,              // in the period
    "included": 200000000000,          // what the plan covers. null if metered
    "remaining": 41000000000,          // what is left. null if there is no cap
    "unit": "bytes",
    "unitLabel": null
  },
  "topUp": {                           // how the balance gets refilled
    "mode": "manual",                  // REQUIRED for prepaid. auto | manual | none
    "thresholdValue": null,            // balance that triggers an auto top-up
    "amountCents": null,               // what each top-up charges
    "paymentMethod": "ok",             // ok | expiring | failed | missing
    "lastAt": "2026-08-04T09:12:00.000Z",
    "lastFailedAt": null
  },
  "exhaustsAt": "2026-08-29T00:00:00.000Z",   // null = not projected to run out
  "accountUrl": "https://example.com/billing",
  "asOf": "2026-08-26T18:03:40.000Z",
  "note": "At zero the gateway stops serving until it is topped up."
}
```

### `billing` — does the balance come back?

| Value | `usage.remaining` | Running out means |
|---|---|---|
| `prepaid` | required | The balance drains and does not come back. Service stops until someone tops it up. **An outage whose fix is a credit card** — page days ahead. |
| `quota` | required | The allowance resets each cycle. Exhausted on the 28th is degraded until the 1st. Bad, bounded, self-healing. |
| `postpaid` | must be null | Nothing to run out of; spend accrues. Something that can hit zero is not postpaid. |
| `free` | optional | No money, but a free tier still has a ceiling worth watching. `spend` must be zero or absent. |

`usage.remaining` is the balance the table constrains. `usage.used` and
`usage.included` are legal on any billing mode — metering and a plan allowance
are not a balance.

### `topUp` — does anybody have to act?

Hitting zero on an account that refills itself is a charge. Hitting zero on one
that needs a human is an outage. Same balance, same burn rate, completely
different night.

| `mode` | Hitting zero means |
|---|---|
| `auto` | The provider charges the card and service continues. A cost event, not an availability one. |
| `manual` | Somebody has to notice and pay. **This is the outage case**, and it deserves to page days ahead. |
| `none` | Nothing refills it. The account ends when the balance does. |

**`auto` is only as good as the card behind it.** An auto top-up with an expired
card is worse than manual, because nobody is watching a thing that has always
worked. That is what `paymentMethod` is for, and why anything other than `ok`
puts the account back in the at-risk set whatever `mode` says.

The mirror image is worth saying too: auto top-up converts an availability risk
into a spend risk. A runaway consumer against a manual account stops; against an
auto account it bills. Put an `expected` band on that vendor's projected spend.

**At risk** means projected to hit zero before anything refills it — a reset, a
renewal, or a working auto top-up. The comparison is `exhaustsAt` against
`resetsAt`: a quota that exhausts *after* it resets is fine forever; one that
exhausts before is this month's problem. Prepaid has no reset, so the projection
is simply a deadline. Omit `exhaustsAt` and the consumer projects it from its own
history — weeks of curve rather than one window's average.

Ranking accounts by "days until zero" only means something if the ones that
never reset, and the ones that refill themselves, are distinguishable from the
ones that do neither.

**Recurring and metered spend only.** One-time charges — a domain purchase, a
GPU, an annual filing fee — stay out by default; this section is for monitoring
what is running, not for bookkeeping. A producer that wants them anyway puts
them in `spend.lastInvoiceCents` for the period they hit and says so in `note`.

**`parent` breaks a bill into line items, and you should break it.** One
provider with compute, functions, storage, a database, and egress is six
entries: the account and five children. A lump per provider says spend went up;
line items say which component did it, which is the only version you can act on.
Billing APIs will group by component if you ask.

A parent's `spend` is the whole account. Children sum to no more than the parent;
the gap is spend the provider did not attribute.

---

## 11. `accounts` — what we hold

Bank accounts, processor balances, reserves. Money in, against §10's money out.
Like vendors, an account carries its own figures inline; the business-wide
roll-ups are metrics.

```jsonc
{
  "id": "bank-operating",            // REQUIRED. stable
  "name": "Operating account",       // REQUIRED
  "kind": "bank",                    // REQUIRED. bank | processor | card
                                     // | reserve | treasury | other
  "connection": "read_only",         // REQUIRED. read_only | manual | none
  "role": "operating",               // operating | tax | payouts | reserve
  "currency": "usd",                 // REQUIRED
  "balanceCents": 4821900,           // REQUIRED. denominated in `currency`
  "usdCents": null,                  // REQUIRED when `currency` is not usd
  "availableCents": 4712300,         // minus holds and pending debits
  "pendingInCents": 312400,          // settling toward this account
  "pendingOutCents": 109600,         // authorised, not yet settled
  "flow": {                          // movement over the period
    "window": "30d",
    "inCents": 5120400,
    "outCents": 233250
  },
  "institutionLast4": "4471",        // identification, never the account number
  "asOf": "2026-08-26T06:00:00.000Z",
  "note": null
}
```

### Read-only, and balances only

- **`connection: "read_only"`, or nothing.** A monitoring endpoint has no reason
  to hold a credential that can move money, and this document is cached, logged,
  and stored on a machine that is not the one holding the money.
- **No account numbers, no routing details, no card numbers.** `institutionLast4`
  is enough to tell two accounts apart.
- **No transaction lists.** Balances and period totals only. Itemising
  transactions turns a monitoring document into a copy of your bank statement,
  and every finding here is an aggregate anyway.
- `manual` is a legitimate connection. A balance typed in weekly with an honest
  `asOf` still answers "how much runway is there".
- **One currency in the roll-ups.** `balanceCents` is in the account's own
  currency; anything that is not `usd` also carries `usdCents`, converted, with
  the rate's source in `note`. `finance.cash` sums `usdCents` where present and
  `balanceCents` otherwise. An account with neither is left out of the total and
  named in `unavailable[]` — adding euro cents to dollar cents is a number that
  looks right and is not.

### Reconciliation is the point

Two independent numbers should nearly agree: what the accounts say went out, and
what the vendors say they charged.

```
finance.unreconciled  =  Σ accounts[].flow.outCents  −  Σ vendors[].spend, same period
```

Sum only top-level vendors — children are a breakdown of their parent, not extra
spend. **Transfers between your own accounts are not flow.** A processor
settling into the operating account is the same money arriving twice; net those
out or the residual means nothing.

A small residual is normal: one-time charges are deliberately out of §10 — a
domain renewal, a laptop, a filing fee — and payroll and tax are not vendors at
all. **A large or growing residual is the finding.** Spend you cannot name is
either a vendor nobody is tracking or a charge nobody authorised, and both are
expensive to learn about a month late.

Put an `expected` band on it, so the alert fires when the number leaves the range
it has held all year rather than every time it moves.

---

## 12. `hosts` — machines we are responsible for

Identity only. CPU, memory, VRAM, disk, network, sockets are metrics scoped with
`host` (§6.2, Machines).

```jsonc
{
  "id": "gpu-01",                // REQUIRED
  "name": "gpu-01",              // REQUIRED
  "status": "up",                // REQUIRED. up | degraded | down | unknown
  "kind": "bare-metal",          // bare-metal | vm | vps | container-host | other
  "role": "inference",           // free text
  "services": ["inference"],     // services[].id running here
  "lastSeenAt": "2026-08-26T18:03:00.000Z",
  "uptimeSeconds": 918273,
  "bootedAt": "2026-08-16T02:15:00.000Z",
  "os": "Ubuntu 24.04",
  "location": "office",          // free text: a region, a rack, a room
  "note": "Thermal throttling above 80°C on the top card."
}
```

A host that has not reported in is `unknown`, not `down`. The consumer decides
what a stale `lastSeenAt` means.

**Count your processes.** Every non-serverless host reports
`resource.processes` — with `expected`, because the right number is a number you
know — and `resource.orphans`. Processes counts every live process matching a
service you manage, orphans included. An **orphan** is one no supervisor owns:
reparented to init when its parent died, or left by a release that
half-restarted. It holds memory, file handles, and sometimes a port, and the
supervisor that would restart it cannot see it — which is how a machine sits at
78% memory with every service reporting healthy.

On a container platform, ask the orchestrator the same question: a task or pod
running that no current deployment accounts for. If there is no supervisor to
compare against, report `resource.processes` and omit orphans rather than
guessing at one.

Serverless services set `serverless: true` and report no host. That is not a
gap — there is nothing to watch.

---

## 13. `endpoints` — URLs under health check

```jsonc
{
  "id": "api-health",                          // REQUIRED
  "url": "https://api.acme.example/health",    // REQUIRED
  "status": "up",                              // REQUIRED. up | degraded | down | unknown
  "service": "api",                            // services[].id this proves
  "checkedAt": "2026-08-26T18:03:55.000Z",
  "statusCode": 200,
  "latencyMs": 84,
  "from": "us-west",                           // where the check ran
  "expectStatus": 200,
  "tls": {
    "expiresAt": "2026-11-02T00:00:00.000Z",
    "issuer": "Example CA",
    "daysRemaining": 68
  }
}
```

An expired certificate is a total outage with a week of warning, so TLS expiry
belongs on the thing that serves it, not in a calendar somewhere.

---

## 14. `apis` — every API surface and operation

What the business exposes and how hard it is being called. One row per surface,
one row per operation, linked by `parent` — the same nesting as services.

```jsonc
{
  "id": "api-public:POST /v1/solve",   // REQUIRED. stable
  "name": "Submit a solve",            // REQUIRED
  "parent": "api-public",              // the surface this operation belongs to
  "service": "api",                    // services[].id that serves it
  "method": "POST",                    // REQUIRED for rest and webhook surfaces
  "path": "/v1/solve",                 // REQUIRED. route TEMPLATE, never a real path
  "kind": "rest",                      // rest | graphql | grpc | websocket
                                       // | webhook | mcp | other
  "auth": "bearer",                    // none | bearer | key | oauth | mtls | signature
  "visibility": "public",              // public | partner | internal
  "status": "up",                      // up | degraded | down | unknown
  "deprecated": false,
  "since": "2026-03-01T00:00:00.000Z",
  "url": "https://api.acme.example/v1/solve"
}
```

Traffic is metrics scoped with `api` (§6.1):

| Metric | |
|---|---|
| `usage.requests` with `outcome: "success"` and `outcome: "failure"` | required for every operation |
| `usage.latency_p50`, `usage.latency_p95`, `usage.error_rate` | where you have them |

### Rules

- **Route templates, never concrete paths.** `/v1/solve/{id}`, not
  `/v1/solve/8812`. Concrete paths make cardinality unbounded and the section
  worthless within a day.
- **Every operation, including the ones nobody called.** A route at zero is
  information — it is dead, it is broken, or nobody found it. Omitting it says
  nothing; `value: 0` says something.
- **Counts, not just a rate.** A rate cannot distinguish 1 failure in 3 from
  40,000 in 120,000. Emit both outcomes and let the consumer divide.
- **Define failure once and say which.** 5xx is always failure; 4xx is a
  judgement call — a 429 you imposed and a 401 from a token you rotated are
  arguably yours. Note the convention on the surface row.
- A surface row's own metrics are that surface's total. Emit them or emit the
  operations, not both, or the totals double.
- **The metric cap binds before the api cap.** 500 operations reporting both
  outcomes is the entire 1000-metric budget (§26) and leaves room for nothing
  else. Past a couple of hundred operations, report the surfaces and the
  operations that earn their row.
- **No per-route instrumentation? Report the surfaces.** Per-operation counts
  are where the value is, but they usually mean new middleware. A gateway that
  only knows its own total is still worth reporting: emit the surface rows, skip
  the operations, and add them when the counters exist.

Most gateways, load balancers, and cloud CLIs already report request and error
counts grouped by route. Pull that grouping — the account-wide total is the
number you already have and cannot act on.

---

## 15. `jobs` — crons and batches

Point-in-time state of every scheduled thing, plus the range that counts as
normal.

```jsonc
{
  "id": "nightly-ingest",              // REQUIRED. stable
  "name": "Nightly ingest",            // REQUIRED
  "status": "anomalous",               // REQUIRED. ok | running | late | failed
                                       //           | anomalous | disabled | unknown
  "service": "ingest",
  "schedule": "0 4 * * *",             // cron expression, or free text
  "nextRunAt": "2026-08-27T04:00:00.000Z",
  "lateBySeconds": null,
  "lastRun": {
    "startedAt": "2026-08-26T04:00:03.000Z",
    "finishedAt": "2026-08-26T05:41:22.000Z",
    "durationMs": 6079000,
    "outcome": "success",              // success | failure | partial | timeout | killed
    "processed": 212418,
    "failed": 91,
    "url": "https://ci.acme.example/runs/8821"
  },
  "expected": {
    "processed":  { "min": 15000, "max": 25000 },
    "durationMs": { "min": 300000, "max": 1500000 }
  },
  "consecutiveFailures": 0,
  "note": "10x normal volume; upstream re-listed its whole catalogue."
}
```

`expected` is the point. A run processing ten times its normal volume passes
every check the job itself can make, and is still the most interesting thing that
happened last night. `status: "anomalous"` means it finished with numbers outside
the declared range.

---

## 16. `deployments` — what is running

```jsonc
{
  "id": "api",                           // REQUIRED. stable
  "env": "prod",                         // REQUIRED
  "repo": "acme/api",                    // REQUIRED. owner/name
  "status": "up",                        // up | degraded | down
                                         // | deploying | unknown
  "service": "api",
  "ref": "main",
  "sha": "f48f630b",
  "version": "1.4.2",
  "deployedAt": "2026-08-24T09:00:00.000Z",
  "url": "https://github.com/acme/api/releases/tag/v1.4.2",
  "previous": { "sha": "1c9a04e2", "version": "1.4.1",
                "deployedAt": "2026-08-19T21:04:42.000Z" }
}
```

What is running, nothing else. Not what should ship next, and not merge
ordering — that is cross-repo knowledge no single business owns, and it has to
stay readable when the service is too broken to answer this endpoint. Keep it in
the consumer's own config.

`previous` is what a rollback returns you to, which is what lets a rollback
button say where it is going before you press it.

---

## 17. `events` — stream: notable things that happened

Everything a human would want told about after the fact. Only items after the
`events` cursor.

```jsonc
{
  "id": "evt-2026-08-26-8f21c4",   // REQUIRED. stable and unique. dedupe key
  "at": "2026-08-26T05:41:22.000Z",// REQUIRED
  "kind": "job",                   // REQUIRED. signup | payment | churn | refund
                                   // | deploy | rollback | job | scale | config
                                   // | security | vendor | support | manual | other
  "severity": "warn",              // ok | warn | crit            (default ok)
  "title": "Nightly ingest processed 212,418 rows",  // REQUIRED
  "service": "ingest",
  "job": "nightly-ingest",
  "value": 212418,
  "unit": "count",
  "expected": { "min": 15000, "max": 25000 },
  "detail": "Upstream re-listed its full catalogue after a schema migration.",
  "url": "https://ci.acme.example/runs/8821"
}
```

Emit an event for what you would want to find when reconstructing a bad day:
deploys and rollbacks, plan changes and cancellations, a job outside its range,
a scaling action, a config or secret change, a login from somewhere new, a
vendor topped up.

Do not emit one per request, per signup, or per row. If it happens more than a
few hundred times a day it is a metric, not an event.

---

## 18. `errors` — stream: bucketed application errors

**Grouped, never raw.** One entry per distinct failure, with a count. A thousand
identical timeouts is one bucket with `count: 1000`.

```jsonc
{
  "id": "api:TimeoutError:upstream-fetch",  // REQUIRED. stable bucket key
  "service": "api",                         // REQUIRED
  "kind": "TimeoutError",                   // REQUIRED. class, code, or type
  "message": "upstream fetch timed out after 5000ms",  // REQUIRED. representative
  "level": "error",                         // warn | error | fatal   (default error)
  "count": 412,                             // occurrences since the cursor
  "usersAffected": 37,
  "firstSeenAt": "2026-07-02T11:04:00.000Z",// first ever. a regression vs a classic
  "startedAt": "2026-08-26T17:04:03.114Z",  // first since the cursor
  "lastSeenAt": "2026-08-26T18:02:51.000Z",
  "release": "1.4.2",
  "sample": { "requestId": "b12f…", "path": "/v1/solve", "statusCode": 504 },
  "url": "https://errors.acme.example/g/8812"
}
```

- **Bucket on `(service, kind, normalized message)`.** Strip ids, numbers, and
  paths from the message before grouping, or every request becomes its own bucket
  and the section is noise.
- `id` must be **stable across polls** — the same failure produces the same key
  tomorrow, or the consumer cannot tell "still happening" from "happened again".
- `firstSeenAt` earlier than the current release is a long-standing bug;
  `firstSeenAt` inside it is a regression you just shipped. Cheap to record, and
  it is the field that says which.
- **No PII and no payloads.** `sample` carries identifiers, not bodies.
- Cap at 100 buckets sorted by `count` descending. Over the cap the tail is
  dropped for good, not deferred: set `cursor.complete: false` and say so in
  `unavailable[]` (§2, rule 4).

---

## 19. `inbox` — stream: new customer mail

Every support channel that receives customer messages, in one queue. Items after
the `inbox` cursor.

```jsonc
{
  "id": "msg-8f21",                             // REQUIRED. stable
  "receivedAt": "2026-08-26T17:42:00.000Z",     // REQUIRED
  "subject": "Can't cancel my subscription",    // REQUIRED
  "channel": "email",                           // email | chat | form | social | phone
  "from": "j****@example.com",                  // masked
  "fromName": "Jane",
  "status": "unread",                           // unread | open | waiting | closed
  "priority": "high",                           // low | normal | high
  "snippet": "First 200 characters, plain text…",
  "tags": ["billing"],
  "waitingSeconds": 1320,                       // since it arrived unanswered
  "customerId": "cus_8812",                     // join key into your own admin
  "url": "https://mail.acme.example/t/8f21"     // deep link to the real thread
}
```

**Never full bodies, never attachments.** A snippet and a link are enough to
triage, and this document is cached, logged, and written to a database on
another machine. Mask the address; the real one is behind the link.

Be clear-eyed about what is left: `fromName`, `subject`, and `snippet` are text a
customer wrote, so a report carrying an inbox is carrying customer data, and the
consumer's database inherits that. A business that would rather not spread it
omits all three and keeps the queue — `inbox.unread` and `inbox.oldest_age`
answer the question the dashboard is actually asking.

---

## 20. `issues` — stream: public repository issues

Links and titles only, and **public repositories only**. Issues opened or
updated since the `issues` cursor.

```jsonc
{
  "id": "acme/api#412",                              // REQUIRED. stable
  "repo": "acme/api",                                // REQUIRED
  "number": 412,                                     // REQUIRED
  "title": "Rate limit headers missing on 429",      // REQUIRED
  "url": "https://github.com/acme/api/issues/412",   // REQUIRED
  "state": "open",                                   // open | closed
  "openedAt": "2026-08-26T16:20:00.000Z",
  "updatedAt": "2026-08-26T17:55:00.000Z",
  "author": "octocat",
  "labels": ["bug"],
  "comments": 3,
  "isPullRequest": false
}
```

The point is knowing a stranger filed something, not mirroring the tracker.
Private repositories stay out — their titles leak roadmap and customers.

---

## 21. `ci` — stream: failed builds and tests

Runs since the `ci` cursor that did not pass. Successes are a metric
(`ci.pass_rate`), not a list.

```jsonc
{
  "id": "acme/api:build:1892",           // REQUIRED. stable
  "repo": "acme/api",                    // REQUIRED
  "workflow": "build",                   // REQUIRED
  "status": "failed",                    // REQUIRED. failed | cancelled | timed_out
  "branch": "main",
  "sha": "f48f630b",
  "runNumber": 1892,
  "startedAt": "2026-08-26T17:31:00.000Z",
  "durationMs": 214000,
  "failedJobs": ["test (node 22)"],
  "failedTests": ["applies coupons before tax", "rejects expired tokens"],
  "url": "https://github.com/acme/api/actions/runs/1892",
  "blocking": true                       // is this on a branch that ships?
}
```

`blocking` separates a red main branch from a red experiment.

---

## 22. `incidents` — open and recently resolved

```jsonc
{
  "id": "ingest-failure:source-b",       // REQUIRED. STABLE across polls
  "title": "Source B returned 0 rows for 3 consecutive runs",  // REQUIRED
  "severity": "warn",                    // REQUIRED. ok | warn | crit
  "status": "open",                      // REQUIRED. open | acknowledged | resolved
  "openedAt": "2026-08-25T05:00:00.000Z",
  "resolvedAt": null,
  "source": "monitoring",                // monitoring | deploy | cron | vendor
                                         // | manual | external
  "service": "ingest.source-b",
  "customerImpact": false,
  "count": 3,                            // occurrences since openedAt
  "detail": "Selector change suspected; other three sources unaffected.",
  "url": "https://status.acme.example/i/412"
}
```

`id` must be **stable across polls**. Key it on what is broken, never on a
timestamp or a random id, or the same outage arrives 1,440 times a day and you
stop reading the list.

Keep resolved incidents for 24h so a UI can show a recently-resolved strip, then
drop them.

---

## 23. `domains`

```jsonc
{
  "id": "acme.example",              // REQUIRED
  "name": "acme.example",            // REQUIRED
  "status": "ok",                    // REQUIRED. ok | expiring | expired | unknown
  "registrar": "Example Registrar",
  "expiresAt": "2027-03-14T00:00:00.000Z",
  "daysRemaining": 200,
  "autoRenew": true,
  "renewalCents": 1400,
  "locked": true,                    // transfer lock
  "dnssec": true,
  "primary": true,                   // is the business served from it
  "note": null
}
```

`autoRenew: false` on a domain the business is served from is a self-inflicted
outage with a date on it. Certificates live on `endpoints[].tls` (§13).

---

## 24. `compliance` — filings, registrations, insurance

The paperwork that quietly expires: entity good standing, annual reports,
franchise tax, registered agent, insurance, licences.

```jsonc
{
  "id": "de-llc-good-standing",              // REQUIRED
  "name": "Delaware LLC — good standing",    // REQUIRED
  "kind": "registration",                    // REQUIRED. registration | filing | tax
                                             // | insurance | licence | agent | other
  "status": "ok",                            // REQUIRED. ok | due | overdue
                                             //           | lapsed | unknown
  "authority": "Delaware Division of Corporations",
  "goodThrough": "2027-06-01T00:00:00.000Z", // when the current standing ends
  "dueAt": "2027-06-01T00:00:00.000Z",       // when the next action is due
  "daysRemaining": 279,
  "lastFiledAt": "2026-05-28T00:00:00.000Z",
  "amountCents": 30000,                      // what the next filing costs
  "autoFiled": false,                        // does an agent handle it
  "url": "https://icis.corp.delaware.gov",
  "note": "Registered agent files the annual report; franchise tax is manual."
}
```

Losing good standing is slow, silent, expensive to unwind, and nobody remembers
to check. Say it out loud months early.

---

## 25. `extra` and `unavailable`

```jsonc
"extra": {
  "Model": { "name": "ranker v1.1", "accuracy": 0.941 },
  "Feature flags": { "newCheckout": "10%", "asyncExport": "off" }
}
```

Free-form. Keys are display labels, values are any JSON. Rendered generically —
scalars as rows, arrays of flat objects as tables. When something in `extra`
becomes one of the first things you look at every morning, promote it to a
metric so it graphs.

```jsonc
"unavailable": [
  { "section": "vendors", "reason": "billing API returned 502", "at": "2026-08-26T18:03:44.000Z" },
  { "section": "metrics", "metric": "engagement.retention_d30",
    "reason": "cohort table rebuilding", "at": "2026-08-26T18:03:20.000Z" },
  { "section": "errors", "reason": "buffer overflowed; 1,200 oldest dropped", "at": "2026-08-26T12:00:00.000Z" }
]
```

`section` is always a top-level key, because that is what the consumer keys its
last-good values on. To name one missing number rather than a whole section, add
`metric`.

Present means the report is incomplete and says which part. The consumer shows a
banner and keeps the last good value for those sections, marked stale. A
dropped buffer (§2) is declared here too.

---

## 26. Limits

| Thing | Cap | Over the cap |
|---|---|---|
| Whole document | 2 MB | Consumer rejects, logs, keeps last good. |
| `metrics` | 1000 | Truncated, `featured` first. |
| `services` | 200 | Truncated, `critical` first. |
| `vendors` | 100 | Truncated, `crit` first. |
| `accounts` | 50 | Truncated, largest balance first. |
| `hosts` | 50 | Truncated. |
| `endpoints` | 100 | Truncated, `down` first. |
| `apis` | 500 | Truncated, most-called first, surfaces before operations. The metric cap bites first (§14). |
| `sources` | 200 | Truncated, most traffic first. |
| `funnel` | 20 | Truncated from the bottom of the funnel up. |
| `jobs` | 100 | Truncated, not-`ok` first. |
| `deployments` | 100 | Truncated. |
| `events` | 500 | §2, rule 4. |
| `errors` | 100 buckets | By `count` descending, and the tail is lost (§2, rule 4). |
| `inbox` | 100 | §2, rule 4. |
| `issues` | 100 | §2, rule 4. |
| `ci` | 100 | §2, rule 4. |
| `incidents` | 100 | Truncated, `crit` first. |
| `domains` | 200 | Truncated, soonest expiry first. |
| `compliance` | 100 | Truncated, soonest due first. |
| Any string | 2000 chars | Truncated. |
| `snippet`, `message` | 500 chars | Truncated. |

Sort before you truncate. Send the 100 error buckets that matter, not the 100
the query returned first.

Streams truncate with a watermark (§2, rule 4) and are therefore never lossy —
except `errors`, which is ranked by `count` and drops its tail. Snapshots
truncate by dropping the least important rows, and are lossy by design.

---

## 27. Security

This document is a complete operational picture of a company behind one bearer
token, fetched by a machine that holds the same for every other company. Two
controls, in this order: who may connect, then who may read.

1. **Allowlist the caller, and check it before the token.** The endpoint accepts
   connections only from an explicit list of addresses, CIDR ranges, and
   hostnames, and answers everything else with `403` before it reads the
   `Authorization` header — an off-list caller must not get far enough to time
   the comparison. There is no wildcard and no "allow all" value, and an empty,
   missing, or unparseable list denies every request.

   - **Hostnames resolve at runtime** — every 60 s, or the record's TTL if that
     is longer — and the last good answer is cached. A dynamic-DNS name is how an
     address on DHCP gets on the list. Resolution failure keeps serving the cached
     set; NXDOMAIN never empties the list, and never opens it.
   - **Match the peer address of the connection.** Trust `X-Forwarded-For` only
     when the peer is itself on a separate trusted-proxy list, and then read
     exactly one hop from the right. A forwarded header from an untrusted peer is
     attacker-controlled text, and believing it bypasses this entire section in
     one line.
   - **Allow an IPv6 prefix, not an address.** Residential ISPs delegate a /56 or
     /64 and rotate the low bits, so a /128 entry locks you out at the next lease.
   - **Keep one way in that does not depend on the list** — a loopback listener
     reached over SSH, or console access — and write it in the deployment notes.
     The failure mode of this control is locking yourself out on the morning the
     dashboard is the thing you need.
   - **Log every denial with its source address.** A denial from an address you do
     not recognise is the only place this document will ever tell you that
     somebody found the endpoint.

2. **Bearer token, always.** `Authorization: Bearer <token>`, minimum 32
   characters, compared in constant time. Never in the query string, never in a
   log line.
3. **Fail closed when unconfigured.** No token in the environment, or no
   allowlist, means deny every request — never "skip the check". That is the bug
   that ships inside a container whose env file did not mount and looks healthy
   while wide open.
4. **One token per business**, rotatable without coordinating across companies,
   and separate from any credential that can spend money or write.
5. **TLS, or a tunnel.** Bearer tokens go over HTTPS. The one exception is a
   listener bound to loopback and reached through SSH or a VPN, where the tunnel
   is the transport security. Never a plaintext port on a public interface.
6. **No secrets, no bodies, no PII.** No keys, tokens, passwords, connection
   strings, customer documents, or message bodies. No account, routing, or card
   numbers, and no transaction lists — §11 is balances and totals, and the
   connection behind it is read-only. Mask email addresses, and see §19 for what
   an inbox section does still carry and what that costs. Error samples carry
   identifiers, not payloads. Private repository issues stay out.
   The report is cached, logged, and stored on another machine; a backup of that
   machine is a backup of everything you ever put in a report.
7. **`Cache-Control: no-store`.**
8. **Rate limit** to roughly 10/min, with `Retry-After` on every `429`. The
   consumer polls once a minute, plus a few immediate re-polls to drain a
   truncated stream (§2, rule 4); more than that is a bug or an intruder.

---

## 28. Versioning

`spec` is `"jaw-business-report/<major>"`, the major a bare integer — this document
is `"jaw-business-report/1"`, and that exact string is what §3 requires. Consumers
accept the current major and the one before it.

**Free within a major:** new optional fields, new registry ids, new enum values.
Consumers ignore unknown keys and read unknown enum values as `unknown`.

**Needs a major bump:** removing or renaming a field, changing a type or unit,
making an optional field required, or changing what an existing metric id means.
Retire an id rather than redefining it — a redefined metric produces a chart
that lies about last month.

---

## 29. Conformance

A checker takes a report from a file, or from a live endpoint with a token, and
runs four passes.

**1. Valid.** Required fields present, enums known, ids and timestamps in the §3
shapes, money integral, ratios in 0–1 unless `signed`, `unitLabel` set whenever
`unit` is `other`, `window` set on counters and drawn from §6.4, `cohort` present
on every cohort-basis funnel stage, balance present on `prepaid` and `quota`
vendors and absent on `postpaid` and `free`, `usdCents` present on every account
not denominated in `usd`. *Errors.*

**2. Consistent.** The document agrees with itself:

- Every `service`, `host`, `vendor`, `job`, `endpoint`, `api`, `source`, `stage`
  on a metric resolves.
- Every `services[].parent`, `vendors[].parent`, `apis[].parent`, `dependsOn`,
  `hosts[].services`, `apis[].service`, and `endpoints[].service` resolves. No
  parent cycles.
- The metric identity tuple (§6.1) is unique.
- `vendors.at_risk` equals the vendors projected to hit zero before a reset, a
  renewal, or an auto top-up with a healthy payment method refills them, and
  none of those reports `status: ok`.
- `vendors.topup_failed` equals the accounts on `auto` whose `paymentMethod` is
  not `ok` or whose last top-up failed.
- `finance.cash` equals the sum of `accounts[].usdCents`, falling back to
  `balanceCents` for accounts already in `usd`.
- `finance.unreconciled` equals account outflow minus top-level vendor spend for
  the same period, where both are present.
- Every account declares a `connection`, and none carries an account number or a
  transaction list.
- `incidents.open` equals the unresolved entries in `incidents[]`.
- `domains.expiring` and `compliance.due` agree with their lists.
- Every `expected` has `min <= max`.
- Where an operation reports both outcomes and `usage.error_rate`, the rate
  matches the counts within 1% relative.
- `funnel[].position` values are unique. Where a stage reports
  `funnel.conversion` and both counts exist, the rate matches within 1%.
- A stage count scoped to a source never exceeds that stage's unscoped count.
- No single-label group of `cost.spend` exceeds `cost.total` for the same
  window, and child vendor spend sums to no more than the parent's.
- Every stream item falls after the requested cursor and at or before that
  stream's returned watermark, measured on that stream's ordering field (§2).
- Every stream item has a stable, unique `id`.
- No `series`, `points`, or `previous` on a metric — history is not this
  document's job (§6). `deployments[].previous` is the one legitimate use of the
  name, and is not history.
- `cursor.streams` names every stream the business supports, whatever `sections`
  asked for, and `cursor.from` is present on a first call.

*Errors.* Each is a case where a UI renders something confidently wrong rather
than visibly broken.

**3. Complete.** The five recommended metrics (§6.2) present; `direction` set;
`expected` on jobs, on `resource.processes`, and on any metric with a known
normal range; both outcomes of `usage.requests` on every operation in `apis[]`;
`resource.processes` and `resource.orphans` on every non-serverless host;
`featured` metrics present, at most eight; `generatedAt` recent; every service
either has a host, inherits one, declares `serverless`, or is `kind: external`.
*Warnings* — a business with no revenue is not a malformed business.

**4. Live.** Against a real endpoint:

- Responds in under 2 s.
- Sets `Cache-Control: no-store`.
- A request with **no** token and a request with a **wrong** token both get
  `401`. Schema validation never catches a fail-open endpoint; this is the only
  check that does.
- **Allowlist.** A correct token from an address that is not on the list gets
  `403`, and the response says nothing a `401` would not. With the allowlist
  unset, every request gets `403`. Starting up with the dynamic-DNS hostname
  unresolvable leaves the endpoint closed, not open.
- Served over HTTPS, or over a loopback listener reached through a tunnel.
- **Cursor round trip:** call once, send `cursor.streams` back immediately, and
  confirm the watermarks come back equal or later and no `events`, `inbox`, or
  `ci` item repeats. `errors` and `issues` may legitimately return the same id
  again, so compare `(id, lastSeenAt)` and `(id, updatedAt)` there (rule 3). It
  catches the race in §2: a producer stamping watermarks from a single "now"
  fails it.

Exit non-zero on errors, zero on warnings alone. Run passes 1–3 against a fixture
in each business's CI — a refactor that drops a metric should fail a pull request,
not quietly blank a chart. **Pass 4 cannot run in CI**, because a hosted runner's
egress address is not on the allowlist and never should be. Run it from the
consumer's own machine on a schedule. A red build "fixed" by widening the
allowlist has traded the whole of §27 for a green tick.

---

## Appendix A — a complete report

Acme Corp: a paid API with a web app, a self-hosted inference box, a collection
service with four upstream sources, and the paperwork of a Delaware LLC. Every
section is populated, every reference resolves, and the numbers agree.

The story it tells: source B has been dead for a day, `POST /v1/export` is
failing every call, the hourly rollup just processed ten times its normal volume,
the worker's memory is above its range with an orphaned process left over from a
restart, a GPU is over its temperature band, the proxy balance runs out in three
days, one domain is 21 days from expiry with auto-renew off, the franchise tax
is due, paid search is bringing a quarter of the traffic, a tenth of the signups
and an LTV only 1.9x its CAC, and $196 of card spend belongs to no tracked
vendor. The LLM account also runs dry this week and does not appear as a risk,
because it tops itself up. And since the 17:31 deploy, the database is serving
three times the reads per request against a flat row count — the shape of an
index that stopped being used.

```json
{
  "spec": "jaw-business-report/1",

  "cursor": {
    "requestedSince": "2026-08-26T17:04:03.114Z",
    "from": "2026-08-26T17:04:03.114Z",
    "streams": {
      "events": "2026-08-26T18:03:58.902Z",
      "errors": "2026-08-26T18:03:59.417Z",
      "inbox":  "2026-08-26T18:03:57.001Z",
      "issues": "2026-08-26T18:03:52.660Z",
      "ci":     "2026-08-26T18:03:59.980Z"
    },
    "complete": true
  },

  "business": {
    "id": "acme",
    "name": "Acme Corp",
    "env": "prod",
    "generatedAt": "2026-08-26T18:04:00.112Z",
    "ttlSeconds": 60,
    "revision": "f48f630b",
    "links": [
      { "label": "Admin", "url": "https://admin.acme.example" },
      { "label": "Status", "url": "https://status.acme.example" }
    ]
  },

  "status": {
    "level": "warn",
    "summary": "Source B down 37h. /v1/export failing every call. Hourly rollup ran 10x normal. Proxy balance out in 3 days. Orphaned worker on app-01.",
    "since": "2026-08-25T05:00:00.000Z"
  },

  "metrics": [
    { "id": "revenue.mrr", "label": "MRR", "value": 4289000, "unit": "usd_cents", "kind": "gauge", "group": "Revenue", "direction": "up_good", "target": 5000000, "featured": true },
    { "id": "revenue.gross", "label": "Gross revenue", "value": 5120400, "unit": "usd_cents", "kind": "counter", "window": "30d", "group": "Revenue", "direction": "up_good" },
    { "id": "revenue.net", "label": "Net revenue", "value": 4712100, "unit": "usd_cents", "kind": "counter", "window": "30d", "group": "Revenue", "direction": "up_good" },
    { "id": "revenue.net", "label": "Net revenue — API", "value": 3980200, "unit": "usd_cents", "kind": "counter", "window": "30d", "service": "api", "group": "Revenue", "direction": "up_good" },
    { "id": "revenue.net", "label": "Net revenue — inference", "value": 731900, "unit": "usd_cents", "kind": "counter", "window": "30d", "service": "inference", "group": "Revenue", "direction": "up_good" },
    { "id": "revenue.arpu", "label": "ARPU", "value": 4287, "unit": "usd_cents", "kind": "gauge", "group": "Revenue", "direction": "up_good" },
    { "id": "payments.count", "label": "Payments", "value": 1204, "unit": "count", "kind": "counter", "window": "30d", "group": "Revenue", "direction": "up_good" },
    { "id": "payments.failed", "label": "Failed payments", "value": 38, "unit": "count", "kind": "counter", "window": "30d", "group": "Revenue", "direction": "down_good", "expected": { "min": 0, "max": 25 }, "severity": "warn" },
    { "id": "payments.refunds", "label": "Refunds", "value": 48200, "unit": "usd_cents", "kind": "counter", "window": "30d", "group": "Revenue", "direction": "down_good" },
    { "id": "payments.disputed", "label": "Chargebacks", "value": 0, "unit": "usd_cents", "kind": "counter", "window": "30d", "group": "Revenue", "direction": "down_good" },

    { "id": "users.total", "label": "Total users", "value": 18422, "unit": "count", "kind": "gauge", "group": "Users", "direction": "up_good" },
    { "id": "users.new", "label": "New users", "value": 812, "unit": "count", "kind": "counter", "window": "30d", "group": "Users", "direction": "up_good" },
    { "id": "users.active", "label": "Active users", "value": 6104, "unit": "count", "kind": "gauge", "window": "30d", "group": "Users", "direction": "up_good", "featured": true },
    { "id": "users.paying", "label": "Paying users", "value": 1099, "unit": "count", "kind": "gauge", "group": "Users", "direction": "up_good" },
    { "id": "users.trial", "label": "In trial", "value": 84, "unit": "count", "kind": "gauge", "group": "Users", "direction": "neutral" },
    { "id": "users.delinquent", "label": "Delinquent", "value": 12, "unit": "count", "kind": "gauge", "group": "Users", "direction": "down_good" },
    { "id": "users.suspended", "label": "Suspended", "value": 3, "unit": "count", "kind": "gauge", "group": "Users", "direction": "down_good" },
    { "id": "users.churned", "label": "Churned", "value": 41, "unit": "count", "kind": "counter", "window": "30d", "group": "Users", "direction": "down_good" },

    { "id": "engagement.dau", "label": "DAU", "value": 1840, "unit": "count", "kind": "gauge", "group": "Engagement", "direction": "up_good" },
    { "id": "engagement.wau", "label": "WAU", "value": 4210, "unit": "count", "kind": "gauge", "group": "Engagement", "direction": "up_good" },
    { "id": "engagement.mau", "label": "MAU", "value": 6104, "unit": "count", "kind": "gauge", "group": "Engagement", "direction": "up_good" },
    { "id": "engagement.stickiness", "label": "DAU/MAU", "value": 0.301, "unit": "ratio", "kind": "gauge", "group": "Engagement", "direction": "up_good" },
    { "id": "engagement.sessions_per_user", "label": "Sessions per user", "value": 3.2, "unit": "count", "kind": "gauge", "window": "7d", "group": "Engagement", "direction": "up_good" },
    { "id": "engagement.actions_per_user", "label": "API calls per active user", "value": 148.4, "unit": "count", "kind": "gauge", "window": "7d", "group": "Engagement", "direction": "up_good" },
    { "id": "engagement.session_seconds_p50", "label": "Median session", "value": 412, "unit": "seconds", "kind": "gauge", "window": "7d", "group": "Engagement", "direction": "up_good" },
    { "id": "engagement.retention_d30", "label": "D30 retention", "value": null, "unit": "ratio", "kind": "gauge", "group": "Engagement", "direction": "up_good", "note": "Cohort table is rebuilding; not available this hour." },

    { "id": "usage.requests", "label": "API requests", "value": 41484, "unit": "count", "kind": "counter", "window": "1h", "service": "api", "group": "Usage", "direction": "up_good", "note": "All surfaces. The per-operation breakdown is scoped by api." },
    { "id": "usage.success_rate", "label": "API success rate", "value": 0.9961, "unit": "ratio", "kind": "gauge", "window": "1h", "service": "api", "group": "Usage", "direction": "up_good" },
    { "id": "usage.error_rate", "label": "API error rate", "value": 0.0039, "unit": "ratio", "kind": "gauge", "window": "1h", "service": "api", "group": "Usage", "direction": "down_good" },
    { "id": "usage.latency_p50", "label": "API p50", "value": 84, "unit": "ms", "kind": "gauge", "window": "5m", "service": "api", "group": "Usage", "direction": "down_good" },
    { "id": "usage.latency_p95", "label": "API p95", "value": 412, "unit": "ms", "kind": "gauge", "window": "5m", "service": "api", "group": "Usage", "direction": "down_good", "target": 300, "expected": { "min": 80, "max": 600 }, "featured": true },
    { "id": "usage.latency_p99", "label": "API p99", "value": 1210, "unit": "ms", "kind": "gauge", "window": "5m", "service": "api", "group": "Usage", "direction": "down_good" },
    { "id": "usage.inflight", "label": "In flight", "value": 37, "unit": "count", "kind": "gauge", "service": "api", "group": "Usage", "direction": "neutral" },
    { "id": "usage.requests", "label": "Inference requests", "value": 2100, "unit": "count", "kind": "counter", "window": "1h", "service": "inference", "group": "Usage", "direction": "up_good" },
    { "id": "usage.units", "label": "Credits consumed", "value": 1240000, "unit": "credits", "kind": "counter", "window": "30d", "service": "inference", "group": "Usage", "direction": "up_good" },
    { "id": "usage.units", "label": "Tokens billed by the LLM provider", "value": 184000000, "unit": "tokens", "kind": "counter", "window": "30d", "vendor": "llm", "group": "Usage", "direction": "neutral" },
    { "id": "usage.units", "label": "Rows rolled up", "value": 212418, "unit": "count", "kind": "counter", "window": "24h", "job": "hourly-rollup", "group": "Jobs", "direction": "neutral", "expected": { "min": 15000, "max": 25000 }, "severity": "warn", "note": "10x the normal range. See events." },

    { "id": "usage.requests", "label": "Source A rows", "value": 8412, "unit": "count", "kind": "counter", "window": "24h", "service": "ingest.source-a", "group": "Ingest", "direction": "up_good" },
    { "id": "usage.requests", "label": "Source B rows", "value": 0, "unit": "count", "kind": "counter", "window": "24h", "service": "ingest.source-b", "group": "Ingest", "direction": "up_good", "expected": { "min": 4000, "max": 12000 }, "severity": "crit" },
    { "id": "usage.requests", "label": "Source C rows", "value": 6120, "unit": "count", "kind": "counter", "window": "24h", "service": "ingest.source-c", "group": "Ingest", "direction": "up_good" },
    { "id": "usage.requests", "label": "Source D rows", "value": 3902, "unit": "count", "kind": "counter", "window": "24h", "service": "ingest.source-d", "group": "Ingest", "direction": "up_good" },

    { "id": "queue.depth", "label": "Queue depth", "value": 1841, "unit": "count", "kind": "gauge", "service": "queue", "group": "Reliability", "direction": "down_good", "expected": { "min": 0, "max": 2500 } },
    { "id": "queue.oldest_age", "label": "Oldest queued item", "value": 92, "unit": "seconds", "kind": "gauge", "service": "queue", "group": "Reliability", "direction": "down_good" },
    { "id": "queue.dlq_depth", "label": "Dead letters", "value": 12, "unit": "count", "kind": "gauge", "service": "queue", "group": "Reliability", "direction": "down_good" },

    { "id": "uptime.window", "label": "API uptime", "value": 0.9993, "unit": "ratio", "kind": "gauge", "window": "30d", "service": "api", "group": "Reliability", "direction": "up_good", "featured": true },
    { "id": "uptime.outage_seconds", "label": "API downtime", "value": 1814, "unit": "seconds", "kind": "counter", "window": "30d", "service": "api", "group": "Reliability", "direction": "down_good" },
    { "id": "uptime.window", "label": "Marketing site uptime", "value": 0.9998, "unit": "ratio", "kind": "gauge", "window": "30d", "endpoint": "www", "group": "Reliability", "direction": "up_good" },
    { "id": "incidents.open", "label": "Open incidents", "value": 2, "unit": "count", "kind": "gauge", "group": "Reliability", "direction": "down_good" },
    { "id": "errors.count", "label": "Errors", "value": 596, "unit": "count", "kind": "counter", "window": "1h", "group": "Reliability", "direction": "down_good" },
    { "id": "errors.buckets", "label": "Distinct errors", "value": 3, "unit": "count", "kind": "gauge", "window": "1h", "group": "Reliability", "direction": "down_good" },
    { "id": "jobs.late", "label": "Late jobs", "value": 1, "unit": "count", "kind": "gauge", "group": "Jobs", "direction": "down_good" },
    { "id": "jobs.failed", "label": "Failed job runs", "value": 0, "unit": "count", "kind": "counter", "window": "24h", "group": "Jobs", "direction": "down_good" },
    { "id": "ci.failed_runs", "label": "Failed CI runs", "value": 1, "unit": "count", "kind": "counter", "window": "24h", "group": "Code", "direction": "down_good" },
    { "id": "ci.pass_rate", "label": "CI pass rate", "value": 0.92, "unit": "ratio", "kind": "gauge", "window": "7d", "group": "Code", "direction": "up_good" },

    { "id": "db.reads_per_request", "label": "DB reads per API request", "value": 41.2, "unit": "count", "kind": "gauge", "window": "5m", "service": "postgres", "group": "Data", "direction": "down_good", "expected": { "min": 8, "max": 18 }, "severity": "crit", "note": "Was 12 before web 2.8.0 shipped at 17:31. Same traffic, three times the reads." },
    { "id": "db.rows", "label": "users rows", "value": 18422, "unit": "count", "kind": "gauge", "service": "postgres", "instance": "users", "group": "Data", "direction": "neutral" },
    { "id": "db.rows", "label": "solves rows", "value": 4120880, "unit": "count", "kind": "gauge", "service": "postgres", "instance": "solves", "group": "Data", "direction": "neutral" },
    { "id": "db.rows", "label": "sessions rows", "value": 91204, "unit": "count", "kind": "gauge", "service": "postgres", "instance": "sessions", "group": "Data", "direction": "neutral" },
    { "id": "db.bytes", "label": "solves on disk", "value": 9019431322, "unit": "bytes", "kind": "gauge", "service": "postgres", "instance": "solves", "group": "Data", "direction": "neutral" },
    { "id": "db.reads", "label": "solves reads", "value": 1840000, "unit": "count", "kind": "counter", "window": "1h", "service": "postgres", "instance": "solves", "group": "Data", "direction": "neutral", "expected": { "min": 300000, "max": 700000 }, "severity": "warn" },
    { "id": "db.writes", "label": "solves writes", "value": 41200, "unit": "count", "kind": "counter", "window": "1h", "service": "postgres", "instance": "solves", "group": "Data", "direction": "neutral" },
    { "id": "db.reads", "label": "users reads", "value": 412000, "unit": "count", "kind": "counter", "window": "1h", "service": "postgres", "instance": "users", "group": "Data", "direction": "neutral" },
    { "id": "db.writes", "label": "users writes", "value": 812, "unit": "count", "kind": "counter", "window": "1h", "service": "postgres", "instance": "users", "group": "Data", "direction": "neutral" },
    { "id": "db.seq_scans", "label": "solves sequential scans", "value": 1840, "unit": "count", "kind": "counter", "window": "1h", "service": "postgres", "instance": "solves", "group": "Data", "direction": "down_good", "expected": { "min": 0, "max": 50 }, "severity": "crit", "note": "Row count is flat, so this is a query that stopped using an index." },
    { "id": "db.dead_rows", "label": "solves dead rows", "value": 184000, "unit": "count", "kind": "gauge", "service": "postgres", "instance": "solves", "group": "Data", "direction": "down_good" },
    { "id": "db.query_p95", "label": "Query p95", "value": 128, "unit": "ms", "kind": "gauge", "window": "5m", "service": "postgres", "group": "Data", "direction": "down_good", "expected": { "min": 5, "max": 80 }, "severity": "warn" },
    { "id": "db.connections", "label": "DB connections", "value": 74, "unit": "count", "kind": "gauge", "service": "postgres", "group": "Data", "direction": "down_good", "expected": { "min": 0, "max": 80 }, "severity": "warn", "note": "Pool limit is 80." },
    { "id": "db.replication_lag", "label": "Replica lag", "value": 0.4, "unit": "seconds", "kind": "gauge", "service": "postgres", "group": "Data", "direction": "down_good" },

    { "id": "resource.cpu_pct", "label": "app-01 CPU", "value": 34.2, "unit": "percent", "kind": "gauge", "host": "app-01", "group": "Machines", "direction": "down_good" },
    { "id": "resource.mem_pct", "label": "app-01 memory", "value": 78.4, "unit": "percent", "kind": "gauge", "host": "app-01", "group": "Machines", "direction": "down_good", "severity": "warn" },
    { "id": "resource.psi_mem", "label": "app-01 memory pressure", "value": 12.4, "unit": "percent", "kind": "gauge", "host": "app-01", "group": "Machines", "direction": "down_good", "expected": { "min": 0, "max": 5 }, "severity": "warn", "note": "PSI avg60. Rising with the worker's memory." },
    { "id": "resource.mem_bytes", "label": "API memory", "value": 812000000, "unit": "bytes", "kind": "gauge", "service": "api", "host": "app-01", "group": "Machines", "direction": "down_good", "expected": { "min": 400000000, "max": 900000000 } },
    { "id": "resource.mem_bytes", "label": "Worker memory", "value": 3910000000, "unit": "bytes", "kind": "gauge", "service": "worker", "host": "app-01", "group": "Machines", "direction": "down_good", "expected": { "min": 400000000, "max": 2000000000 }, "severity": "crit", "note": "Climbs steadily between restarts. See restarts below." },
    { "id": "resource.restarts", "label": "Worker restarts", "value": 2, "unit": "count", "kind": "counter", "window": "24h", "service": "worker", "group": "Machines", "direction": "down_good", "expected": { "min": 0, "max": 0 }, "severity": "warn" },
    { "id": "resource.sockets", "label": "API sockets", "value": 1842, "unit": "count", "kind": "gauge", "service": "api", "group": "Machines", "direction": "down_good" },
    { "id": "resource.fds", "label": "API file descriptors", "value": 2104, "unit": "count", "kind": "gauge", "service": "api", "group": "Machines", "direction": "down_good" },
    { "id": "resource.threads", "label": "Worker threads", "value": 96, "unit": "count", "kind": "gauge", "service": "worker", "group": "Machines", "direction": "down_good" },
    { "id": "resource.net_rx_bps", "label": "app-01 download", "value": 412000000, "unit": "other", "unitLabel": "bit/s", "kind": "gauge", "host": "app-01", "group": "Machines", "direction": "neutral" },
    { "id": "resource.net_tx_bps", "label": "app-01 upload", "value": 98000000, "unit": "other", "unitLabel": "bit/s", "kind": "gauge", "host": "app-01", "group": "Machines", "direction": "neutral" },
    { "id": "resource.load_1m", "label": "gpu-01 load", "value": 4.2, "unit": "other", "unitLabel": "load", "kind": "gauge", "host": "gpu-01", "group": "Machines", "direction": "down_good" },
    { "id": "resource.disk_used_pct", "label": "gpu-01 root disk", "value": 96.1, "unit": "percent", "kind": "gauge", "host": "gpu-01", "instance": "/", "group": "Machines", "direction": "down_good", "severity": "crit" },
    { "id": "resource.disk_free_bytes", "label": "gpu-01 root free", "value": 8123456789, "unit": "bytes", "kind": "gauge", "host": "gpu-01", "instance": "/", "group": "Machines", "direction": "up_good" },
    { "id": "resource.disk_used_pct", "label": "gpu-01 data disk", "value": 3.2, "unit": "percent", "kind": "gauge", "host": "gpu-01", "instance": "/data", "group": "Machines", "direction": "down_good" },
    { "id": "resource.gpu_util_pct", "label": "GPU 0 utilization", "value": 88, "unit": "percent", "kind": "gauge", "host": "gpu-01", "instance": "gpu0", "group": "Machines", "direction": "neutral" },
    { "id": "resource.vram_used_bytes", "label": "GPU 0 VRAM used", "value": 22548578304, "unit": "bytes", "kind": "gauge", "host": "gpu-01", "instance": "gpu0", "service": "inference", "group": "Machines", "direction": "down_good" },
    { "id": "resource.vram_total_bytes", "label": "GPU 0 VRAM", "value": 25757220864, "unit": "bytes", "kind": "gauge", "host": "gpu-01", "instance": "gpu0", "group": "Machines", "direction": "neutral" },
    { "id": "resource.gpu_temp_c", "label": "GPU 0 temperature", "value": 81, "unit": "other", "unitLabel": "°C", "kind": "gauge", "host": "gpu-01", "instance": "gpu0", "group": "Machines", "direction": "down_good", "expected": { "min": 30, "max": 80 }, "severity": "warn", "note": "Throttles above 80." },
    { "id": "resource.gpu_power_w", "label": "GPU 0 power", "value": 420, "unit": "other", "unitLabel": "W", "kind": "gauge", "host": "gpu-01", "instance": "gpu0", "group": "Machines", "direction": "neutral" },

    { "id": "cost.total", "label": "Spend", "value": 191400, "unit": "usd_cents", "kind": "counter", "window": "30d", "group": "Cost", "direction": "down_good", "expected": { "min": 150000, "max": 210000 }, "featured": true },
    { "id": "cost.per_unit", "label": "Cost per credit", "value": 15, "unit": "usd_cents", "kind": "gauge", "window": "30d", "group": "Cost", "direction": "down_good" },
    { "id": "cost.per_user", "label": "Cost per active user", "value": 31, "unit": "usd_cents", "kind": "gauge", "window": "30d", "group": "Cost", "direction": "down_good" },
    { "id": "margin.gross", "label": "Gross margin", "value": 0.9594, "unit": "ratio", "kind": "gauge", "window": "30d", "group": "Cost", "direction": "up_good", "signed": true },
    { "id": "vendors.at_risk", "label": "Accounts running out", "value": 1, "unit": "count", "kind": "gauge", "group": "Cost", "direction": "down_good", "note": "The proxy account, which is manual. The LLM account also hits zero this week but tops itself up." },
    { "id": "vendors.topup_failed", "label": "Auto top-ups at risk", "value": 0, "unit": "count", "kind": "gauge", "group": "Cost", "direction": "down_good", "expected": { "min": 0, "max": 0 } },
    { "id": "finance.cash", "label": "Cash on hand", "value": 6974300, "unit": "usd_cents", "kind": "gauge", "group": "Finance", "direction": "up_good", "featured": true },
    { "id": "finance.burn_net", "label": "Net burn", "value": -4887150, "unit": "usd_cents", "kind": "counter", "window": "30d", "group": "Finance", "direction": "down_good", "note": "Negative: cash grew by $48,871 over the period." },
    { "id": "finance.runway_days", "label": "Runway", "value": null, "unit": "other", "unitLabel": "days", "kind": "gauge", "group": "Finance", "direction": "up_good", "note": "Cash is growing, so runway does not apply." },
    { "id": "finance.unreconciled", "label": "Unaccounted spend", "value": 19600, "unit": "usd_cents", "kind": "gauge", "window": "30d", "group": "Finance", "direction": "down_good", "expected": { "min": 0, "max": 40000 }, "note": "$196 of card spend no tracked vendor claims — a domain renewal and a monitor. Inside the usual band." },

    { "id": "inbox.unread", "label": "Unanswered mail", "value": 2, "unit": "count", "kind": "gauge", "group": "Support", "direction": "down_good" },
    { "id": "inbox.oldest_age", "label": "Longest wait", "value": 1320, "unit": "seconds", "kind": "gauge", "group": "Support", "direction": "down_good" },
    { "id": "inbox.response_p50", "label": "Median first response", "value": 5400, "unit": "seconds", "kind": "gauge", "window": "7d", "group": "Support", "direction": "down_good" },
    { "id": "issues.open", "label": "Open public issues", "value": 7, "unit": "count", "kind": "gauge", "group": "Code", "direction": "neutral" },
    { "id": "domains.expiring", "label": "Domains expiring", "value": 1, "unit": "count", "kind": "gauge", "group": "Paperwork", "direction": "down_good", "severity": "warn" },
    { "id": "compliance.due", "label": "Filings due", "value": 1, "unit": "count", "kind": "gauge", "group": "Paperwork", "direction": "down_good", "severity": "warn" },

    { "id": "acme.solve.attempts", "label": "Solves — succeeded", "value": 41022, "unit": "count", "kind": "counter", "window": "1h", "service": "inference", "outcome": "success", "group": "Product", "direction": "up_good" },
    { "id": "acme.solve.attempts", "label": "Solves — failed", "value": 178, "unit": "count", "kind": "counter", "window": "1h", "service": "inference", "outcome": "failure", "group": "Product", "direction": "down_good", "expected": { "min": 0, "max": 400 } },
    { "id": "acme.model.accuracy", "label": "Model accuracy", "value": 0.941, "unit": "ratio", "kind": "gauge", "window": "24h", "service": "inference", "group": "Product", "direction": "up_good", "target": 0.95 },

    { "id": "funnel.count", "label": "Search impressions", "value": 412000, "unit": "count", "kind": "gauge", "window": "30d", "stage": "impression", "group": "Funnel", "direction": "up_good" },
    { "id": "funnel.count", "label": "Visits", "value": 18400, "unit": "count", "kind": "gauge", "window": "30d", "stage": "visit", "group": "Funnel", "direction": "up_good", "featured": true },
    { "id": "funnel.count", "label": "Signups", "value": 1240, "unit": "count", "kind": "gauge", "stage": "signup", "group": "Funnel", "direction": "up_good" },
    { "id": "funnel.count", "label": "Finished onboarding", "value": 812, "unit": "count", "kind": "gauge", "stage": "onboarded", "group": "Funnel", "direction": "up_good" },
    { "id": "funnel.count", "label": "Using it weekly", "value": 604, "unit": "count", "kind": "gauge", "stage": "active", "group": "Funnel", "direction": "up_good" },
    { "id": "funnel.count", "label": "Converted to paid", "value": 96, "unit": "count", "kind": "gauge", "stage": "paid", "group": "Funnel", "direction": "up_good" },

    { "id": "funnel.conversion", "label": "Impression → visit", "value": 0.0447, "unit": "ratio", "kind": "gauge", "stage": "visit", "group": "Funnel", "direction": "up_good", "note": "Two populations, not a conversion — both stages are window basis." },
    { "id": "funnel.conversion", "label": "Visit → signup", "value": 0.0674, "unit": "ratio", "kind": "gauge", "stage": "signup", "group": "Funnel", "direction": "up_good", "note": "Crosses the window/cohort boundary. Read as an estimate." },
    { "id": "funnel.conversion", "label": "Signup → onboarded", "value": 0.6548, "unit": "ratio", "kind": "gauge", "stage": "onboarded", "group": "Funnel", "direction": "up_good", "expected": { "min": 0.6, "max": 0.8 } },
    { "id": "funnel.conversion", "label": "Onboarded → weekly", "value": 0.7438, "unit": "ratio", "kind": "gauge", "stage": "active", "group": "Funnel", "direction": "up_good" },
    { "id": "funnel.conversion", "label": "Weekly → paid", "value": 0.1589, "unit": "ratio", "kind": "gauge", "stage": "paid", "group": "Funnel", "direction": "up_good", "target": 0.2, "featured": true },
    { "id": "funnel.time_to_convert_p50", "label": "Signup → onboarded, median", "value": 420, "unit": "seconds", "kind": "gauge", "stage": "onboarded", "group": "Funnel", "direction": "down_good" },
    { "id": "funnel.time_to_convert_p50", "label": "Weekly → paid, median", "value": 950400, "unit": "seconds", "kind": "gauge", "stage": "paid", "group": "Funnel", "direction": "down_good", "note": "11 days, so the 30-day cohort is mature enough to read." },

    { "id": "funnel.count", "label": "Visits — organic search", "value": 8900, "unit": "count", "kind": "gauge", "window": "30d", "stage": "visit", "source": "organic-search", "group": "Acquisition", "direction": "up_good" },
    { "id": "funnel.count", "label": "Visits — paid search", "value": 4200, "unit": "count", "kind": "gauge", "window": "30d", "stage": "visit", "source": "paid-search", "group": "Acquisition", "direction": "up_good" },
    { "id": "funnel.count", "label": "Visits — partner blog", "value": 2610, "unit": "count", "kind": "gauge", "window": "30d", "stage": "visit", "source": "partner-blog", "group": "Acquisition", "direction": "up_good" },
    { "id": "funnel.count", "label": "Visits — newsletter", "value": 1890, "unit": "count", "kind": "gauge", "window": "30d", "stage": "visit", "source": "newsletter", "group": "Acquisition", "direction": "up_good" },
    { "id": "funnel.count", "label": "Visits — direct", "value": 800, "unit": "count", "kind": "gauge", "window": "30d", "stage": "visit", "source": "direct", "group": "Acquisition", "direction": "up_good" },
    { "id": "funnel.count", "label": "Signups — organic search", "value": 700, "unit": "count", "kind": "gauge", "stage": "signup", "source": "organic-search", "group": "Acquisition", "direction": "up_good" },
    { "id": "funnel.count", "label": "Signups — paid search", "value": 121, "unit": "count", "kind": "gauge", "stage": "signup", "source": "paid-search", "group": "Acquisition", "direction": "up_good", "severity": "warn", "note": "23% of visits, 10% of signups." },
    { "id": "funnel.count", "label": "Signups — partner blog", "value": 240, "unit": "count", "kind": "gauge", "stage": "signup", "source": "partner-blog", "group": "Acquisition", "direction": "up_good" },
    { "id": "funnel.count", "label": "Signups — newsletter", "value": 139, "unit": "count", "kind": "gauge", "stage": "signup", "source": "newsletter", "group": "Acquisition", "direction": "up_good" },
    { "id": "funnel.count", "label": "Signups — direct", "value": 40, "unit": "count", "kind": "gauge", "stage": "signup", "source": "direct", "group": "Acquisition", "direction": "up_good" },
    { "id": "funnel.count", "label": "Paid — paid search", "value": 9, "unit": "count", "kind": "gauge", "stage": "paid", "source": "paid-search", "group": "Acquisition", "direction": "up_good" },
    { "id": "cost.spend", "label": "Spend — paid search", "value": 41850, "unit": "usd_cents", "kind": "counter", "window": "30d", "source": "paid-search", "group": "Acquisition", "direction": "down_good" },
    { "id": "acquisition.cac", "label": "CAC — paid search", "value": 4650, "unit": "usd_cents", "kind": "gauge", "window": "30d", "source": "paid-search", "group": "Acquisition", "direction": "down_good", "severity": "warn" },
    { "id": "acquisition.ltv", "label": "LTV — paid search", "value": 8900, "unit": "usd_cents", "kind": "gauge", "source": "paid-search", "group": "Acquisition", "direction": "up_good", "note": "1.9x CAC. Below the 3x the other channels clear." },

    { "id": "usage.requests", "label": "POST /v1/solve — succeeded", "value": 34042, "unit": "count", "kind": "counter", "window": "1h", "api": "api-public:POST /v1/solve", "outcome": "success", "group": "API", "direction": "up_good" },
    { "id": "usage.requests", "label": "POST /v1/solve — failed", "value": 126, "unit": "count", "kind": "counter", "window": "1h", "api": "api-public:POST /v1/solve", "outcome": "failure", "group": "API", "direction": "down_good" },
    { "id": "usage.error_rate", "label": "POST /v1/solve error rate", "value": 0.0037, "unit": "ratio", "kind": "gauge", "window": "1h", "api": "api-public:POST /v1/solve", "group": "API", "direction": "down_good" },
    { "id": "usage.latency_p95", "label": "POST /v1/solve p95", "value": 486, "unit": "ms", "kind": "gauge", "window": "5m", "api": "api-public:POST /v1/solve", "group": "API", "direction": "down_good" },
    { "id": "usage.requests", "label": "GET /v1/solve/{id} — succeeded", "value": 5102, "unit": "count", "kind": "counter", "window": "1h", "api": "api-public:GET /v1/solve/{id}", "outcome": "success", "group": "API", "direction": "up_good" },
    { "id": "usage.requests", "label": "GET /v1/solve/{id} — failed", "value": 8, "unit": "count", "kind": "counter", "window": "1h", "api": "api-public:GET /v1/solve/{id}", "outcome": "failure", "group": "API", "direction": "down_good" },
    { "id": "usage.requests", "label": "GET /v1/account — succeeded", "value": 1890, "unit": "count", "kind": "counter", "window": "1h", "api": "api-public:GET /v1/account", "outcome": "success", "group": "API", "direction": "up_good" },
    { "id": "usage.requests", "label": "GET /v1/account — failed", "value": 2, "unit": "count", "kind": "counter", "window": "1h", "api": "api-public:GET /v1/account", "outcome": "failure", "group": "API", "direction": "down_good" },
    { "id": "usage.requests", "label": "DELETE /v1/account/keys/{id} — succeeded", "value": 12, "unit": "count", "kind": "counter", "window": "1h", "api": "api-public:DELETE /v1/account/keys/{id}", "outcome": "success", "group": "API", "direction": "up_good" },
    { "id": "usage.requests", "label": "DELETE /v1/account/keys/{id} — failed", "value": 0, "unit": "count", "kind": "counter", "window": "1h", "api": "api-public:DELETE /v1/account/keys/{id}", "outcome": "failure", "group": "API", "direction": "down_good" },
    { "id": "usage.requests", "label": "POST /v1/export — succeeded", "value": 0, "unit": "count", "kind": "counter", "window": "1h", "api": "api-public:POST /v1/export", "outcome": "success", "group": "API", "direction": "up_good", "severity": "crit" },
    { "id": "usage.requests", "label": "POST /v1/export — failed", "value": 18, "unit": "count", "kind": "counter", "window": "1h", "api": "api-public:POST /v1/export", "outcome": "failure", "group": "API", "direction": "down_good", "severity": "crit" },
    { "id": "usage.error_rate", "label": "POST /v1/export error rate", "value": 1, "unit": "ratio", "kind": "gauge", "window": "1h", "api": "api-public:POST /v1/export", "group": "API", "direction": "down_good", "severity": "crit", "note": "Every call has failed since 17:05. Times out against the object store." },
    { "id": "usage.latency_p95", "label": "POST /v1/export p95", "value": 5012, "unit": "ms", "kind": "gauge", "window": "5m", "api": "api-public:POST /v1/export", "group": "API", "direction": "down_good", "severity": "crit" },
    { "id": "usage.requests", "label": "GET /admin/metrics — succeeded", "value": 60, "unit": "count", "kind": "counter", "window": "1h", "api": "api-admin:GET /admin/metrics", "outcome": "success", "group": "API", "direction": "up_good" },
    { "id": "usage.requests", "label": "GET /admin/metrics — failed", "value": 0, "unit": "count", "kind": "counter", "window": "1h", "api": "api-admin:GET /admin/metrics", "outcome": "failure", "group": "API", "direction": "down_good" },
    { "id": "usage.requests", "label": "POST /admin/replay — succeeded", "value": 3, "unit": "count", "kind": "counter", "window": "1h", "api": "api-admin:POST /admin/replay", "outcome": "success", "group": "API", "direction": "up_good" },
    { "id": "usage.requests", "label": "POST /admin/replay — failed", "value": 1, "unit": "count", "kind": "counter", "window": "1h", "api": "api-admin:POST /admin/replay", "outcome": "failure", "group": "API", "direction": "down_good" },
    { "id": "usage.requests", "label": "POST /hooks/payments — succeeded", "value": 214, "unit": "count", "kind": "counter", "window": "1h", "api": "api-hooks:POST /hooks/payments", "outcome": "success", "group": "API", "direction": "up_good" },
    { "id": "usage.requests", "label": "POST /hooks/payments — failed", "value": 6, "unit": "count", "kind": "counter", "window": "1h", "api": "api-hooks:POST /hooks/payments", "outcome": "failure", "group": "API", "direction": "down_good", "note": "Six retried deliveries from the payment processor. Same batch as the failed renewals." },

    { "id": "cost.spend", "label": "Spend — API", "value": 49000, "unit": "usd_cents", "kind": "counter", "window": "30d", "service": "api", "group": "Cost", "direction": "down_good" },
    { "id": "cost.spend", "label": "Spend — inference", "value": 70000, "unit": "usd_cents", "kind": "counter", "window": "30d", "service": "inference", "group": "Cost", "direction": "down_good" },
    { "id": "cost.spend", "label": "Spend — ingest", "value": 21550, "unit": "usd_cents", "kind": "counter", "window": "30d", "service": "ingest", "group": "Cost", "direction": "down_good" },
    { "id": "cost.spend", "label": "Spend — worker", "value": 9000, "unit": "usd_cents", "kind": "counter", "window": "30d", "service": "worker", "group": "Cost", "direction": "down_good" },

    { "id": "resource.processes", "label": "app-01 managed processes", "value": 14, "unit": "count", "kind": "gauge", "host": "app-01", "group": "Machines", "direction": "neutral", "expected": { "min": 12, "max": 13 }, "severity": "warn", "note": "One more than the supervisor accounts for." },
    { "id": "resource.orphans", "label": "app-01 orphaned processes", "value": 1, "unit": "count", "kind": "gauge", "host": "app-01", "group": "Machines", "direction": "down_good", "expected": { "min": 0, "max": 0 }, "severity": "warn", "note": "A worker from the 09:12 restart, reparented to init. Still holding 1.1 GB and a queue connection." },
    { "id": "resource.processes", "label": "Worker processes", "value": 4, "unit": "count", "kind": "gauge", "service": "worker", "host": "app-01", "group": "Machines", "direction": "neutral", "expected": { "min": 3, "max": 3 }, "severity": "warn" },
    { "id": "resource.processes", "label": "gpu-01 managed processes", "value": 4, "unit": "count", "kind": "gauge", "host": "gpu-01", "group": "Machines", "direction": "neutral", "expected": { "min": 3, "max": 5 } },
    { "id": "resource.orphans", "label": "gpu-01 orphaned processes", "value": 0, "unit": "count", "kind": "gauge", "host": "gpu-01", "group": "Machines", "direction": "down_good", "expected": { "min": 0, "max": 0 } }
  ],

  "services": [
    { "id": "web", "name": "Marketing site and app", "status": "up", "kind": "frontend", "critical": true, "serverless": true, "dependsOn": ["api"], "version": "2.8.0", "startedAt": "2026-08-26T17:31:44.000Z", "checkedAt": "2026-08-26T18:03:58.000Z", "url": "https://acme.example", "env": "prod" },
    { "id": "api", "name": "Public API", "status": "up", "kind": "api", "critical": true, "serverless": false, "hosts": ["app-01"], "dependsOn": ["postgres", "queue", "inference"], "since": "2026-08-24T09:00:00.000Z", "startedAt": "2026-08-24T09:00:00.000Z", "version": "1.4.2", "checkedAt": "2026-08-26T18:03:58.000Z", "url": "https://api.acme.example", "env": "prod" },
    { "id": "worker", "name": "Async worker", "status": "degraded", "kind": "worker", "critical": true, "serverless": false, "hosts": ["app-01"], "dependsOn": ["queue", "postgres"], "since": "2026-08-26T17:38:20.000Z", "startedAt": "2026-08-26T17:38:20.000Z", "version": "1.4.2", "message": "Restarted twice in 24h on memory; RSS climbing again.", "checkedAt": "2026-08-26T18:03:58.000Z", "env": "prod" },
    { "id": "inference", "name": "Inference pool", "status": "up", "kind": "inference", "critical": true, "serverless": false, "hosts": ["gpu-01"], "since": "2026-08-16T02:20:00.000Z", "startedAt": "2026-08-16T02:20:00.000Z", "version": "0.9.4", "message": "1 GPU at 81°C, throttling.", "checkedAt": "2026-08-26T18:03:58.000Z", "env": "prod" },
    { "id": "postgres", "name": "Postgres", "status": "degraded", "kind": "db", "critical": true, "serverless": false, "hosts": ["app-01"], "since": "2026-08-26T17:34:00.000Z", "message": "Reads per request tripled after web 2.8.0; connection pool near its limit.", "checkedAt": "2026-08-26T18:03:58.000Z", "env": "prod" },
    { "id": "queue", "name": "Job queue", "status": "up", "kind": "queue", "critical": true, "serverless": true, "checkedAt": "2026-08-26T18:03:58.000Z", "env": "prod" },

    { "id": "ingest", "name": "Data collection", "status": "degraded", "kind": "worker", "critical": true, "serverless": false, "hosts": ["app-01"], "dependsOn": ["postgres", "proxy-gateway"], "since": "2026-08-25T05:00:00.000Z", "message": "1 of 4 sources failing.", "checkedAt": "2026-08-26T18:03:58.000Z", "env": "prod" },
    { "id": "ingest.source-a", "name": "Source A", "status": "up", "kind": "worker", "parent": "ingest", "critical": false, "checkedAt": "2026-08-26T18:03:58.000Z", "env": "prod" },
    { "id": "ingest.source-b", "name": "Source B", "status": "down", "kind": "worker", "parent": "ingest", "critical": false, "since": "2026-08-25T05:00:00.000Z", "message": "0 rows for 3 consecutive runs; selector change suspected.", "checkedAt": "2026-08-26T18:03:58.000Z", "env": "prod" },
    { "id": "ingest.source-c", "name": "Source C", "status": "up", "kind": "worker", "parent": "ingest", "critical": false, "checkedAt": "2026-08-26T18:03:58.000Z", "env": "prod" },
    { "id": "ingest.source-d", "name": "Source D", "status": "up", "kind": "worker", "parent": "ingest", "critical": false, "checkedAt": "2026-08-26T18:03:58.000Z", "env": "prod" },

    { "id": "proxy-gateway", "name": "Proxy gateway", "status": "up", "kind": "external", "critical": false, "checkedAt": "2026-08-26T18:03:58.000Z", "note": "Third party. Depends on the prepaid proxy account." }
  ],

  "vendors": [
    { "id": "cloud", "name": "Cloud provider", "billing": "postpaid", "status": "ok", "category": "compute",
      "services": ["api", "worker", "postgres", "queue"],
      "period": { "start": "2026-08-01T00:00:00.000Z", "end": "2026-08-31T23:59:59.000Z", "renewsAt": null, "resetsAt": null },
      "plan": { "name": "On demand", "cents": null, "interval": "none" },
      "spend": { "periodToDateCents": 98200, "projectedPeriodCents": 118000, "lastInvoiceCents": 112400, "asOf": "2026-08-26T06:00:00.000Z" },
      "accountUrl": "https://example.com/billing", "asOf": "2026-08-26T18:03:40.000Z" },
    { "id": "cloud.compute", "name": "Cloud — compute instances", "billing": "postpaid", "status": "ok", "category": "compute", "parent": "cloud",
      "period": { "start": "2026-08-01T00:00:00.000Z", "end": "2026-08-31T23:59:59.000Z" },
      "spend": { "periodToDateCents": 42000, "asOf": "2026-08-26T06:00:00.000Z" } },
    { "id": "cloud.functions", "name": "Cloud — functions", "billing": "postpaid", "status": "ok", "category": "compute", "parent": "cloud",
      "period": { "start": "2026-08-01T00:00:00.000Z", "end": "2026-08-31T23:59:59.000Z" },
      "spend": { "periodToDateCents": 9200, "asOf": "2026-08-26T06:00:00.000Z" } },
    { "id": "cloud.storage", "name": "Cloud — object storage", "billing": "postpaid", "status": "ok", "category": "storage", "parent": "cloud",
      "period": { "start": "2026-08-01T00:00:00.000Z", "end": "2026-08-31T23:59:59.000Z" },
      "spend": { "periodToDateCents": 21400, "asOf": "2026-08-26T06:00:00.000Z" } },
    { "id": "cloud.database", "name": "Cloud — managed database", "billing": "postpaid", "status": "ok", "category": "storage", "parent": "cloud",
      "period": { "start": "2026-08-01T00:00:00.000Z", "end": "2026-08-31T23:59:59.000Z" },
      "spend": { "periodToDateCents": 9800, "asOf": "2026-08-26T06:00:00.000Z" } },
    { "id": "cloud.egress", "name": "Cloud — egress", "billing": "postpaid", "status": "warn", "category": "network", "parent": "cloud",
      "period": { "start": "2026-08-01T00:00:00.000Z", "end": "2026-08-31T23:59:59.000Z" },
      "spend": { "periodToDateCents": 15800, "projectedPeriodCents": 19000, "asOf": "2026-08-26T06:00:00.000Z" },
      "note": "Up 61% on last period; the rollup re-reads the bucket." },

    { "id": "proxy", "name": "Residential proxy", "billing": "prepaid", "status": "warn", "category": "proxy",
      "services": ["ingest"],
      "period": { "start": "2026-08-01T00:00:00.000Z", "end": null, "renewsAt": null, "resetsAt": null },
      "plan": { "name": "Pay as you go", "cents": null, "interval": "none" },
      "spend": { "periodToDateCents": 4500, "projectedPeriodCents": 6000, "lastInvoiceCents": 20000, "asOf": "2026-08-26T06:00:00.000Z" },
      "usage": { "used": 159000000000, "included": 200000000000, "remaining": 41000000000, "unit": "bytes" },
      "topUp": { "mode": "manual", "paymentMethod": "ok", "lastAt": "2026-08-04T09:12:00.000Z", "lastFailedAt": null },
      "exhaustsAt": "2026-08-29T00:00:00.000Z",
      "accountUrl": "https://example.com/proxy/billing", "asOf": "2026-08-26T18:03:40.000Z",
      "note": "At zero the gateway stops serving until it is topped up." },

    { "id": "bandwidth", "name": "Datacentre bandwidth", "billing": "quota", "status": "ok", "category": "network",
      "services": ["ingest"],
      "period": { "start": "2026-08-01T00:00:00.000Z", "end": "2026-08-31T23:59:59.000Z", "renewsAt": "2026-09-01T00:00:00.000Z", "resetsAt": "2026-09-01T00:00:00.000Z" },
      "plan": { "name": "500 GB monthly", "cents": 1800, "interval": "monthly" },
      "spend": { "periodToDateCents": 1800, "asOf": "2026-08-26T06:00:00.000Z" },
      "usage": { "used": 388000000000, "included": 500000000000, "remaining": 112000000000, "unit": "bytes" },
      "exhaustsAt": "2026-09-04T00:00:00.000Z", "asOf": "2026-08-26T18:03:40.000Z",
      "note": "Projected to exhaust after it resets, so it never actually runs out." },

    { "id": "llm", "name": "LLM API credits", "billing": "prepaid", "status": "ok", "category": "ai",
      "services": ["inference"],
      "period": { "start": "2026-08-01T00:00:00.000Z", "end": null, "renewsAt": null, "resetsAt": null },
      "plan": { "name": "Prepaid credits", "cents": null, "interval": "none" },
      "spend": { "periodToDateCents": 62400, "projectedPeriodCents": 74000, "asOf": "2026-08-26T06:00:00.000Z" },
      "usage": { "used": 184000000, "included": null, "remaining": 41000000, "unit": "tokens" },
      "topUp": { "mode": "auto", "thresholdValue": 20000000, "amountCents": 20000,
                 "paymentMethod": "ok", "lastAt": "2026-08-22T02:41:00.000Z", "lastFailedAt": null },
      "exhaustsAt": "2026-09-02T00:00:00.000Z", "asOf": "2026-08-26T18:03:40.000Z",
      "note": "Runs out in 7 days and tops itself up for $200. A charge, not an outage — so not at risk." },

    { "id": "email", "name": "Transactional email", "billing": "postpaid", "status": "ok", "category": "saas",
      "services": ["api"],
      "period": { "start": "2026-08-01T00:00:00.000Z", "end": "2026-08-31T23:59:59.000Z", "renewsAt": "2026-09-01T00:00:00.000Z", "resetsAt": null },
      "plan": { "name": "Growth", "cents": 4900, "interval": "monthly" },
      "spend": { "periodToDateCents": 4900, "asOf": "2026-08-26T06:00:00.000Z" },
      "usage": { "used": 41200, "included": 100000, "remaining": null, "unit": "count", "unitLabel": "emails" },
      "asOf": "2026-08-26T18:03:40.000Z" },

    { "id": "ads", "name": "Search ad platform", "billing": "postpaid", "status": "ok", "category": "marketing",
      "period": { "start": "2026-08-01T00:00:00.000Z", "end": "2026-08-31T23:59:59.000Z", "renewsAt": null, "resetsAt": null },
      "plan": { "name": "Auction", "cents": null, "interval": "none" },
      "spend": { "periodToDateCents": 41850, "projectedPeriodCents": 49000, "asOf": "2026-08-26T06:00:00.000Z" },
      "accountUrl": "https://example.com/ads", "asOf": "2026-08-26T18:03:40.000Z" },
    { "id": "errors-saas", "name": "Error tracking (free tier)", "billing": "free", "status": "warn", "category": "saas",
      "period": { "start": "2026-08-01T00:00:00.000Z", "end": "2026-08-31T23:59:59.000Z", "resetsAt": "2026-09-01T00:00:00.000Z" },
      "plan": { "name": "Free", "cents": 0, "interval": "monthly" },
      "usage": { "used": 4820, "included": 5000, "remaining": 180, "unit": "count", "unitLabel": "events" },
      "asOf": "2026-08-26T18:03:40.000Z",
      "note": "96% of the free event ceiling. Over it, errors stop being recorded." }
  ],

  "accounts": [
    { "id": "bank-operating", "name": "Operating account", "kind": "bank", "connection": "read_only",
      "role": "operating", "currency": "usd",
      "balanceCents": 4821900, "availableCents": 4712300,
      "pendingInCents": 312400, "pendingOutCents": 109600,
      "flow": { "window": "30d", "inCents": 5120400, "outCents": 233250 },
      "institutionLast4": "4471", "asOf": "2026-08-26T06:00:00.000Z" },
    { "id": "bank-tax", "name": "Tax reserve", "kind": "reserve", "connection": "read_only",
      "role": "tax", "currency": "usd", "balanceCents": 1840000,
      "institutionLast4": "9032", "asOf": "2026-08-26T06:00:00.000Z",
      "note": "Set aside quarterly. No flow reported — transfers in are from the operating account." },
    { "id": "processor", "name": "Payment processor balance", "kind": "processor", "connection": "read_only",
      "role": "payouts", "currency": "usd", "balanceCents": 312400,
      "asOf": "2026-08-26T18:03:30.000Z",
      "note": "Settles to the operating account every two days. Transfers are not flow." }
  ],

  "hosts": [
    { "id": "app-01", "name": "app-01", "status": "up", "kind": "vps", "role": "application",
      "services": ["api", "worker", "postgres", "ingest"],
      "lastSeenAt": "2026-08-26T18:03:50.000Z", "uptimeSeconds": 4218400,
      "bootedAt": "2026-07-08T14:20:00.000Z", "os": "Ubuntu 24.04", "location": "eu-west" },
    { "id": "gpu-01", "name": "gpu-01", "status": "degraded", "kind": "bare-metal", "role": "inference",
      "services": ["inference"],
      "lastSeenAt": "2026-08-26T18:03:00.000Z", "uptimeSeconds": 918273,
      "bootedAt": "2026-08-16T02:15:00.000Z", "os": "Ubuntu 24.04", "location": "office",
      "note": "Root filesystem at 96% while /data is empty. Top card throttles above 80°C." }
  ],

  "endpoints": [
    { "id": "www", "url": "https://acme.example", "status": "up", "service": "web",
      "checkedAt": "2026-08-26T18:03:55.000Z", "statusCode": 200, "latencyMs": 210, "from": "us-west", "expectStatus": 200,
      "tls": { "expiresAt": "2026-11-02T00:00:00.000Z", "issuer": "Example CA", "daysRemaining": 68 } },
    { "id": "api-health", "url": "https://api.acme.example/health", "status": "up", "service": "api",
      "checkedAt": "2026-08-26T18:03:55.000Z", "statusCode": 200, "latencyMs": 84, "from": "us-west", "expectStatus": 200,
      "tls": { "expiresAt": "2026-11-02T00:00:00.000Z", "issuer": "Example CA", "daysRemaining": 68 } },
    { "id": "docs", "url": "https://docs.acme.example", "status": "degraded", "service": "web",
      "checkedAt": "2026-08-26T18:03:55.000Z", "statusCode": 200, "latencyMs": 3120, "from": "us-west", "expectStatus": 200,
      "tls": { "expiresAt": "2026-11-02T00:00:00.000Z", "issuer": "Example CA", "daysRemaining": 68 } }
  ],

  "sources": [
    { "id": "organic-search", "name": "Organic search", "kind": "search", "attribution": "first_touch",
      "utm": { "source": "search", "medium": "organic" }, "paid": false, "active": true },
    { "id": "paid-search", "name": "Paid search — launch", "kind": "paid", "attribution": "last_touch",
      "utm": { "source": "search", "medium": "cpc", "campaign": "launch-q3" }, "paid": true, "active": true,
      "since": "2026-07-01T00:00:00.000Z",
      "note": "Attributed by the ad platform, so last touch. The organic rows are first touch." },
    { "id": "partner-blog", "name": "Partner blog", "kind": "referral", "attribution": "first_touch",
      "utm": { "source": "partner", "medium": "referral" }, "referrerDomain": "blog.partner.example",
      "paid": false, "active": true },
    { "id": "newsletter", "name": "Newsletter", "kind": "email", "attribution": "last_touch",
      "utm": { "source": "newsletter", "medium": "email", "campaign": "weekly" }, "paid": false, "active": true },
    { "id": "direct", "name": "Direct and unattributed", "kind": "direct", "attribution": "none",
      "paid": false, "active": true,
      "note": "Everything with no referrer or no utm tuple. 4% of visits, so the rest of this section is worth reading." }
  ],

  "funnel": [
    { "id": "impression", "name": "Saw us in search results", "position": 1, "kind": "impression",
      "basis": "window", "optional": false,
      "note": "Search console, window basis — there is no person to cohort yet." },
    { "id": "visit", "name": "Landed on the site", "position": 2, "kind": "visit",
      "basis": "window", "optional": false },
    { "id": "signup", "name": "Created an account", "position": 3, "kind": "signup",
      "basis": "cohort", "cohort": { "window": "30d", "observedForDays": 30 }, "optional": false,
      "note": "First identified stage. Everything below shares this cohort." },
    { "id": "onboarded", "name": "Finished onboarding", "position": 4, "kind": "activation",
      "basis": "cohort", "cohort": { "window": "30d", "observedForDays": 30 }, "optional": false },
    { "id": "active", "name": "Used it in 3 of the last 4 weeks", "position": 5, "kind": "habit",
      "basis": "cohort", "cohort": { "window": "30d", "observedForDays": 30 }, "optional": false },
    { "id": "paid", "name": "Converted to a paid plan", "position": 6, "kind": "paid",
      "basis": "cohort", "cohort": { "window": "30d", "observedForDays": 30 }, "optional": false }
  ],

  "apis": [
    { "id": "api-public", "name": "Public API", "service": "api", "kind": "rest", "auth": "bearer",
      "visibility": "public", "status": "degraded", "url": "https://api.acme.example",
      "note": "Failure = 5xx, plus 429s we imposed. 4xx from caller mistakes are not counted." },
    { "id": "api-public:POST /v1/solve", "name": "Submit a solve", "parent": "api-public", "service": "api",
      "method": "POST", "path": "/v1/solve", "kind": "rest", "auth": "bearer", "visibility": "public",
      "status": "up", "deprecated": false, "since": "2026-03-01T00:00:00.000Z" },
    { "id": "api-public:GET /v1/solve/{id}", "name": "Fetch a solve", "parent": "api-public", "service": "api",
      "method": "GET", "path": "/v1/solve/{id}", "kind": "rest", "auth": "bearer", "visibility": "public",
      "status": "up", "deprecated": false, "since": "2026-03-01T00:00:00.000Z" },
    { "id": "api-public:GET /v1/account", "name": "Account and balance", "parent": "api-public", "service": "api",
      "method": "GET", "path": "/v1/account", "kind": "rest", "auth": "bearer", "visibility": "public",
      "status": "up", "deprecated": false, "since": "2026-03-01T00:00:00.000Z" },
    { "id": "api-public:DELETE /v1/account/keys/{id}", "name": "Revoke an API key", "parent": "api-public",
      "service": "api", "method": "DELETE", "path": "/v1/account/keys/{id}", "kind": "rest", "auth": "bearer",
      "visibility": "public", "status": "up", "deprecated": false, "since": "2026-05-14T00:00:00.000Z" },
    { "id": "api-public:POST /v1/export", "name": "Export results", "parent": "api-public", "service": "api",
      "method": "POST", "path": "/v1/export", "kind": "rest", "auth": "bearer", "visibility": "public",
      "status": "down", "deprecated": false, "since": "2026-07-20T00:00:00.000Z",
      "note": "Every call has failed since 17:05." },

    { "id": "api-admin", "name": "Admin API", "service": "api", "kind": "rest", "auth": "mtls",
      "visibility": "internal", "status": "up" },
    { "id": "api-admin:GET /admin/metrics", "name": "Internal metrics", "parent": "api-admin", "service": "api",
      "method": "GET", "path": "/admin/metrics", "kind": "rest", "auth": "mtls", "visibility": "internal",
      "status": "up" },
    { "id": "api-admin:POST /admin/replay", "name": "Replay a job", "parent": "api-admin", "service": "api",
      "method": "POST", "path": "/admin/replay", "kind": "rest", "auth": "mtls", "visibility": "internal",
      "status": "up" },

    { "id": "api-hooks", "name": "Webhook receiver", "service": "api", "kind": "webhook", "auth": "signature",
      "visibility": "partner", "status": "up" },
    { "id": "api-hooks:POST /hooks/payments", "name": "Payment processor callbacks", "parent": "api-hooks",
      "service": "api", "method": "POST", "path": "/hooks/payments", "kind": "webhook", "auth": "signature",
      "visibility": "partner", "status": "up" }
  ],

  "jobs": [
    { "id": "hourly-rollup", "name": "Hourly usage rollup", "status": "anomalous", "service": "worker",
      "schedule": "0 * * * *", "nextRunAt": "2026-08-26T19:00:00.000Z", "lateBySeconds": null,
      "lastRun": { "startedAt": "2026-08-26T17:00:02.000Z", "finishedAt": "2026-08-26T17:58:31.000Z",
                   "durationMs": 3509000, "outcome": "success", "processed": 212418, "failed": 0,
                   "url": "https://ci.acme.example/runs/8821" },
      "expected": { "processed": { "min": 15000, "max": 25000 }, "durationMs": { "min": 120000, "max": 900000 } },
      "consecutiveFailures": 0,
      "note": "Succeeded, but moved 10x its normal volume and nearly overran its own hour." },
    { "id": "nightly-ingest", "name": "Nightly ingest", "status": "ok", "service": "ingest",
      "schedule": "0 4 * * *", "nextRunAt": "2026-08-27T04:00:00.000Z", "lateBySeconds": null,
      "lastRun": { "startedAt": "2026-08-26T04:00:03.000Z", "finishedAt": "2026-08-26T04:12:44.000Z",
                   "durationMs": 761000, "outcome": "partial", "processed": 18434, "failed": 1,
                   "url": "https://ci.acme.example/runs/8804" },
      "expected": { "processed": { "min": 15000, "max": 30000 }, "durationMs": { "min": 300000, "max": 1500000 } },
      "consecutiveFailures": 0,
      "note": "Inside its range only because the other three sources grew. Source B contributed nothing." },
    { "id": "weekly-backup", "name": "Weekly offsite backup", "status": "late", "service": "postgres",
      "schedule": "0 3 * * 0", "nextRunAt": "2026-08-30T03:00:00.000Z", "lateBySeconds": 7200,
      "lastRun": { "startedAt": "2026-08-23T03:00:01.000Z", "finishedAt": "2026-08-23T03:41:12.000Z",
                   "durationMs": 2471000, "outcome": "success", "processed": 1, "failed": 0 },
      "expected": { "durationMs": { "min": 1200000, "max": 3600000 } },
      "consecutiveFailures": 0,
      "note": "Runner did not pick up the schedule; 2h past due." }
  ],

  "deployments": [
    { "id": "api", "env": "prod", "repo": "acme/api", "status": "up", "service": "api",
      "ref": "main", "sha": "f48f630b", "version": "1.4.2", "deployedAt": "2026-08-24T09:00:00.000Z",
      "url": "https://github.com/acme/api/releases/tag/v1.4.2",
      "previous": { "sha": "1c9a04e2", "version": "1.4.1", "deployedAt": "2026-08-19T21:04:42.000Z" } },
    { "id": "web", "env": "prod", "repo": "acme/web", "status": "up", "service": "web",
      "ref": "main", "sha": "7b21ca90", "version": "2.8.0", "deployedAt": "2026-08-26T17:31:44.000Z",
      "url": "https://github.com/acme/web/releases/tag/v2.8.0",
      "previous": { "sha": "004ad112", "version": "2.7.3", "deployedAt": "2026-08-21T10:02:00.000Z" } },
    { "id": "ingest", "env": "prod", "repo": "acme/ingest", "status": "degraded", "service": "ingest",
      "ref": "main", "sha": "aa19c004", "version": "0.6.1", "deployedAt": "2026-08-22T12:00:00.000Z",
      "previous": { "sha": "88c1d0aa", "version": "0.6.0", "deployedAt": "2026-08-15T08:30:00.000Z" } }
  ],

  "events": [
    { "id": "evt-2026-08-26-a1f2", "at": "2026-08-26T17:12:04.000Z", "kind": "vendor", "severity": "warn",
      "title": "Proxy balance projected to run out in 3 days",
      "detail": "41 GB left, burning 13 GB/day. Prepaid, so it does not refill.",
      "url": "https://example.com/proxy/billing" },
    { "id": "evt-2026-08-26-b2c3", "at": "2026-08-26T17:31:44.000Z", "kind": "deploy", "severity": "ok",
      "title": "Deployed web 2.8.0 to prod", "service": "web",
      "url": "https://github.com/acme/web/releases/tag/v2.8.0" },
    { "id": "evt-2026-08-26-c3d4", "at": "2026-08-26T17:49:10.000Z", "kind": "payment", "severity": "warn",
      "title": "6 renewal charges failed in one batch", "value": 6, "unit": "count",
      "expected": { "min": 0, "max": 2 },
      "detail": "All six were the same card network. Retry scheduled for 04:00." },
    { "id": "evt-2026-08-26-d4e5", "at": "2026-08-26T17:58:31.000Z", "kind": "job", "severity": "warn",
      "title": "Hourly rollup processed 212,418 rows", "service": "worker", "job": "hourly-rollup",
      "value": 212418, "unit": "count", "expected": { "min": 15000, "max": 25000 },
      "detail": "Upstream re-listed its full catalogue after a schema migration.",
      "url": "https://ci.acme.example/runs/8821" }
  ],

  "errors": [
    { "id": "api:TimeoutError:upstream-fetch", "service": "api", "kind": "TimeoutError",
      "message": "upstream fetch timed out after 5000ms", "level": "error",
      "count": 412, "usersAffected": 37,
      "firstSeenAt": "2026-07-02T11:04:00.000Z", "startedAt": "2026-08-26T17:04:09.221Z",
      "lastSeenAt": "2026-08-26T18:02:51.000Z", "release": "1.4.2",
      "sample": { "requestId": "b12f8c04", "path": "/v1/solve", "statusCode": 504 },
      "url": "https://errors.acme.example/g/8812" },
    { "id": "ingest.source-b:ParseError:empty-result", "service": "ingest.source-b", "kind": "ParseError",
      "message": "expected results container, found none", "level": "error",
      "count": 183, "usersAffected": 0,
      "firstSeenAt": "2026-08-25T05:00:00.000Z", "startedAt": "2026-08-26T17:05:00.000Z",
      "lastSeenAt": "2026-08-26T18:00:00.000Z", "release": "0.6.1",
      "sample": { "requestId": "src-b-0912", "path": "/list" },
      "url": "https://errors.acme.example/g/8840" },
    { "id": "worker:OutOfMemory:rollup", "service": "worker", "kind": "OutOfMemory",
      "message": "JavaScript heap out of memory during rollup batch", "level": "fatal",
      "count": 1, "usersAffected": 0,
      "firstSeenAt": "2026-08-26T09:12:00.000Z", "startedAt": "2026-08-26T17:38:12.000Z",
      "lastSeenAt": "2026-08-26T17:38:12.000Z", "release": "1.4.2",
      "sample": { "requestId": "rollup-88213" },
      "url": "https://errors.acme.example/g/8851" }
  ],

  "inbox": [
    { "id": "msg-8f21", "receivedAt": "2026-08-26T17:42:00.000Z", "subject": "Can't cancel my subscription",
      "channel": "email", "from": "j****@example.com", "fromName": "Jane", "status": "unread",
      "priority": "high", "snippet": "I've tried the billing page three times and it just spins…",
      "tags": ["billing"], "waitingSeconds": 1320, "customerId": "cus_8812",
      "url": "https://mail.acme.example/t/8f21" },
    { "id": "msg-8f22", "receivedAt": "2026-08-26T17:51:12.000Z", "subject": "API returning 504 on /v1/solve",
      "channel": "email", "from": "d****@example.org", "fromName": "Dev", "status": "unread",
      "priority": "normal", "snippet": "Started around 17:05 UTC, roughly one in every 200 calls…",
      "tags": ["api", "incident"], "waitingSeconds": 768, "customerId": "cus_4410",
      "url": "https://mail.acme.example/t/8f22" }
  ],

  "issues": [
    { "id": "acme/api#412", "repo": "acme/api", "number": 412,
      "title": "Rate limit headers missing on 429", "url": "https://github.com/acme/api/issues/412",
      "state": "open", "openedAt": "2026-08-26T16:20:00.000Z", "updatedAt": "2026-08-26T17:55:00.000Z",
      "author": "octocat", "labels": ["bug"], "comments": 3, "isPullRequest": false }
  ],

  "ci": [
    { "id": "acme/api:build:1892", "repo": "acme/api", "workflow": "build", "status": "failed",
      "branch": "main", "sha": "f48f630b", "runNumber": 1892,
      "startedAt": "2026-08-26T17:31:00.000Z", "durationMs": 214000,
      "failedJobs": ["test (node 22)"],
      "failedTests": ["applies coupons before tax", "rejects expired tokens"],
      "url": "https://github.com/acme/api/actions/runs/1892", "blocking": true }
  ],

  "incidents": [
    { "id": "ingest-failure:source-b", "title": "Source B returned 0 rows for 3 consecutive runs",
      "severity": "warn", "status": "open", "openedAt": "2026-08-25T05:00:00.000Z", "resolvedAt": null,
      "source": "monitoring", "service": "ingest.source-b", "customerImpact": false, "count": 3,
      "detail": "Selector change suspected; the other three sources are unaffected.",
      "url": "https://status.acme.example/i/412" },
    { "id": "vendor-exhaustion:proxy", "title": "Proxy balance runs out in 3 days",
      "severity": "warn", "status": "open", "openedAt": "2026-08-26T17:12:04.000Z", "resolvedAt": null,
      "source": "vendor", "customerImpact": false, "count": 1,
      "detail": "Prepaid. At zero, collection stops until it is topped up.",
      "url": "https://status.acme.example/i/413" }
  ],

  "domains": [
    { "id": "acme.example", "name": "acme.example", "status": "ok", "registrar": "Example Registrar",
      "expiresAt": "2027-03-14T00:00:00.000Z", "daysRemaining": 200, "autoRenew": true,
      "renewalCents": 1400, "locked": true, "dnssec": true, "primary": true },
    { "id": "acme-cdn.example", "name": "acme-cdn.example", "status": "expiring", "registrar": "Example Registrar",
      "expiresAt": "2026-09-16T00:00:00.000Z", "daysRemaining": 21, "autoRenew": false,
      "renewalCents": 1400, "locked": true, "dnssec": false, "primary": false,
      "note": "Serves static assets for the app. Auto-renew is off." }
  ],

  "compliance": [
    { "id": "de-llc-good-standing", "name": "Delaware LLC — good standing", "kind": "registration",
      "status": "ok", "authority": "Delaware Division of Corporations",
      "goodThrough": "2027-06-01T00:00:00.000Z", "dueAt": "2027-06-01T00:00:00.000Z",
      "daysRemaining": 279, "lastFiledAt": "2026-05-28T00:00:00.000Z", "autoFiled": false,
      "url": "https://example.gov/entity/acme" },
    { "id": "de-franchise-tax", "name": "Delaware franchise tax", "kind": "tax", "status": "due",
      "authority": "Delaware Division of Corporations", "dueAt": "2026-10-10T00:00:00.000Z",
      "daysRemaining": 45, "lastFiledAt": "2025-10-02T00:00:00.000Z", "amountCents": 30000,
      "autoFiled": false, "url": "https://example.gov/tax",
      "note": "Filed manually. Late fees start the day after." },
    { "id": "registered-agent", "name": "Registered agent", "kind": "agent", "status": "ok",
      "authority": "Example Agent Services", "goodThrough": "2027-01-15T00:00:00.000Z",
      "daysRemaining": 142, "amountCents": 12900, "autoFiled": true },
    { "id": "liability-insurance", "name": "General liability insurance", "kind": "insurance", "status": "ok",
      "authority": "Example Insurance", "goodThrough": "2027-02-01T00:00:00.000Z",
      "daysRemaining": 159, "amountCents": 84000, "autoFiled": true }
  ],

  "extra": {
    "Model": { "name": "ranker v1.1", "trainedAt": "2026-08-04", "accuracy": 0.941 },
    "Feature flags": { "newCheckout": "10%", "asyncExport": "off" }
  },

  "unavailable": [
    { "section": "metrics", "metric": "engagement.retention_d30",
      "reason": "cohort table rebuilding since 17:20", "at": "2026-08-26T18:03:20.000Z" }
  ]
}
```
