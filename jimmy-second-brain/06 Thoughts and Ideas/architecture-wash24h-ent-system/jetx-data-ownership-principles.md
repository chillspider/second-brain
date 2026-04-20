# JetX Data Ownership & Events — Principles

> One page. How data flows between apps. When to use an API, when to use a queue.

---

## North Star

**The source system owns its data. Always. Everything else is a transport detail.**

Readers read freely. Writers go through the source — but *how* they reach the source depends on whether a human is waiting.

---

## The Three Channels

```
                  ┌──────────────────────────────────┐
                  │   SOURCE SYSTEM (owns the data)  │
                  │        Only writer. Ever.        │
                  └──────────────────────────────────┘
           ▲                  ▲                     │
           │ 1. Sync API      │ 2. Command topic    │ 3. Change events
           │    (user waits)  │    (async, fanout)  │    (after any write)
           │                  │                     ▼
      Forms, UI edits    Batch jobs,            DW, dashboards,
                         cross-system signals    other apps' caches
```

**Channel 1 — Sync API.** A human is waiting. Validate, apply, return. 200ms feedback. Examples: editing a customer, activating a site, assigning a permission.

**Channel 2 — Command topic.** No human waits. Publisher drops a command; source consumes at its own pace. Examples: ALPR plate sighting updates customer last-seen; dashboard "refresh from source" button; nightly reconciliation; Camera detecting a site offline.

**Channel 3 — Change events.** Always emitted after a successful write on channel 1 or 2. This is the MDM backbone. The DW subscribes. Other apps subscribe. Caches stay fresh.

---

## The Eight Rules

**1. Source system is the only writer to its own data.**
No exceptions. Not for admin scripts, not for migrations, not "just this once." If something else needs to write, you're violating ownership — either move the data, or add an API/command.

**2. User-facing writes go through sync APIs.**
If a human is watching a spinner, don't put a queue between them and the answer. Commands are for system-to-system, not forms.

**3. System-to-system writes go through command topics.**
When no human waits, publish a command addressed to the source. Source validates, applies, rejects, or queues — on its own terms. This gives you backpressure, retries, and durability for free.

**4. Every successful write emits a change event.**
Regardless of how the write arrived. Change events are the only way other systems learn about updates. No polling, no direct DB reads across services.

**5. Command handlers must be idempotent.**
Every command carries a `command_id`. Source deduplicates. Retries are safe. If you can't make it idempotent, it's probably a sync API, not a command.

**6. Failures go to a DLQ, never silently dropped.**
Every command topic has a companion dead-letter stream. A steward UI can inspect, fix, and replay. Silent failure is worse than no async at all.

**7. Schema is versioned and explicit.**
CloudEvents envelope. JSON Schema per event type, in git. Breaking changes = new event type (`customer.updated.v2`), not a mutation of the old one.

**8. ACKs are informational, never gating.**
Command results (success, rejection, deferred) flow through a durable result stream keyed by `command_id`. Subscribers use them for observability, state updates, retries, and user notifications — never to block forward progress. If your code cannot proceed without the ACK, you need a sync API, not a command. Sync-over-queue is worse than sync HTTP in every way.

---

## Decision tree

```
I need to change data in another system
   │
   ├─ Is a human waiting for the result? ──── YES ──► Sync API. Done.
   │          NO
   │
   ├─ Can my code proceed without the result? ─ NO ──► Sync API. You need the ack
   │          YES                                      to move forward; don't fake it
   │                                                   with a blocking queue listener.
   │
   ├─ Could this be retried safely? ─────────── NO ──► Make it idempotent first.
   │          YES
   │
   └─► Command topic. Publish and continue.
       Subscribe to the result stream if you
       want to *observe* completion later —
       but never to gate progress on it.


I need to read data from another system
   │
   ├─ Is it authoritative right-now data? ──── YES ──► Sync API read from source.
   │          NO
   │
   ├─ Is a stale view acceptable? ───────────── YES ──► Subscribe to change events,
   │                                                    keep a local cache, read local.
```

---

## Message envelopes (three shapes, one spec)

All messages share the CloudEvents envelope. Only the `type` and `data` differ.

**Change event** (channel 3, fanout after any successful write):
```json
{
  "specversion": "1.0",
  "id": "01HY3K...",
  "type": "customer.updated.v1",
  "source": "jetx/crm",
  "time": "2026-04-18T12:30:00Z",
  "correlation_id": "req_abc123",
  "causation_id": "cmd_xyz789",
  "data": {
    "entity_id": "cust_abc123",
    "version": 42,
    "changes": { "phone": { "from": "...", "to": "..." } },
    "snapshot": { /* full entity at this version */ }
  }
}
```

**Command** (channel 2, addressed to a source):
```json
{
  "specversion": "1.0",
  "id": "cmd_xyz789",
  "type": "customer.update_requested.v1",
  "source": "jetx/alpr",
  "time": "2026-04-18T12:29:59Z",
  "correlation_id": "req_abc123",
  "data": {
    "entity_id": "cust_abc123",
    "changes": { "last_seen_at": "2026-04-18T12:29:58Z" }
  }
}
```

**ACK / command result** (durable result stream, keyed by `command_id`):
```json
{
  "specversion": "1.0",
  "id": "01HY...",
  "type": "customer.update_completed.v1",
  "source": "jetx/crm",
  "time": "2026-04-18T12:30:04.521Z",
  "correlation_id": "req_abc123",
  "causation_id": "cmd_xyz789",
  "data": {
    "command_id": "cmd_xyz789",
    "status": "success",
    "entity_id": "cust_abc123",
    "version_before": 41,
    "version_after": 42,
    "duration_ms": 47
  }
}
```

`status` is one of `success`, `rejected`, `deferred`. Rejections include a `reason` and optional `retry_after`. Subscribers build dashboards, steward-review queues, and downstream triggers from these — but never block on them.

---

## Stack for JetX

| Component      | Choice                      | Why                                          |
|----------------|-----------------------------|----------------------------------------------|
| Broker         | **NATS JetStream**          | Lighter than Kafka, has streams + work queues, matches the pattern |
| Envelope       | CloudEvents 1.0             | Boring, standard, tooling exists             |
| Schemas        | JSON Schema in git repo     | No schema registry server to run             |
| DLQ            | NATS stream per topic       | Same infra, no new moving parts              |
| Observability  | Prometheus + Grafana        | Already in place                             |

Skip Kafka, Avro, and Schema Registry until you actually hit their justifying scale. You won't.

---

## What to build, and when

1. **Now (with `platform-core`):** Sync APIs + change events on every write. No command topics yet.
2. **First real "signal source from elsewhere" use case lands:** Add one command topic for that one entity. Learn the patterns in production.
3. **Expand deliberately:** Add command topics only where sync APIs are causing actual pain (tight coupling, bursts, rate limits, batch jobs). Not speculatively.

Async infrastructure is cheap to add later, expensive to get wrong early. The sync API + change event baseline is what mature MDM deployments converge on; commands are the advanced layer on top.

---

## Red flags

- A consumer app reads directly from another app's database → ownership violated.
- A command handler is not idempotent → you'll lose or duplicate data on retry.
- A sync API is used because "we don't have a queue" → you'll regret it during a traffic spike.
- A command topic is used because "async is cooler" → you'll regret it when debugging at 2am.
- Code publishes a command and then blocks waiting for the ACK → you've built sync-over-queue; use a sync API instead.
- Change events are emitted before the DB commit → consumers will see data that doesn't exist.
- Two apps subscribe to the same command topic → only the source should consume its own commands.
- Schema changes land without a version bump → downstream breakage, blame spiral.

---

## One-line summary

**Source owns writes. Sync for humans, commands for systems, events for everyone else.**

---

## Applied In

Every active project maps onto these three channels. New projects must declare their channel position in their Project Overview.

- [[Project Overview (Identity)]] — Channel 1 (admin console writes: users/groups/roles); Channel 3 publisher (token contents read by every app); never Channel 2
- [[Project Overview (FinOps)]] — source of truth for reconciliation, VAT, P&L; Xero is a Channel 3 downstream consumer (journal entry push); Vietinbank / Cybersource / EasyInvoice are Channel 3 upstream feeds; finance team edits exceptions via Channel 1 (Sync API)
- [[Project Overview (Jetx-Lakehouse)]] — Channel 3 reader for every source system (PG → Parquet → Iceberg); never writes back to sources; Gold tables are Channel 3 outputs for FinOps, BI, CLI

## See Also

- [[jetx-platform-principles]] — the platform rules these data channels operate under
- [[00-architecture]] — stack-level view
- [[README]] — framework docs in reading order
