# Sync: an ordered log of domain operations

> Part of [ADR-0001](../decisions/0001-founding-architecture.md). Status: **Proposed**; spike S3 must prove it before it's accepted.

## The problem

One person, several devices: the Mac, later a phone, a browser, and optionally an always-on hub. Each device must work offline at full speed, and they must end up with the **same** data. The data is records with rules, not free-form documents:

| Rule | Example |
|---|---|
| Uniqueness | one journal entry per day, one sprint per ISO week, one link per canonical URL, one live tag per name |
| Cardinality | a page has at most one parent; an item is part of at most one project |
| Structure | no page inside itself; an `in_sprint` edge starts from a task |
| Multi-object steps | the sprint close moves N tasks and writes metrics in one step; it must happen once |
| Rich text | page bodies edited on two offline devices must merge |

CRDTs (Automerge, Loro, Yjs) merge each value independently, so they **cannot** keep cross-record rules: two offline devices can each give a page a parent, or each close the same sprint (research notes, `research/03-sync.md` §1.5). Server-authoritative engines (PowerSync, Electric, Zero) keep the rules but need a trusted server that can read the data, which rules out end-to-end encryption and a server-optional design. They also make the device a cache, not the engine.

## The design

**Every change is a named, deterministic domain operation, and every device applies the same operations in the same order.**

```
 device                                   sequencer (hub or relay)
 ───────────────────────────              ─────────────────────────
 UI → mutator(args) ─┐
                     ├─ apply now (optimistic), keep undo log
                     └─ outbox ── encrypt ──▶  append, assign seq ──▶ log (ciphertext)
                                                                         │
 pull since cursor ◀── decrypt ◀──────────────── ops seq > cursor ◀──────┘
   undo pending · apply ordered ops · re-apply still-pending ops
```

1. **Operations, not row diffs.** An operation is `{ id, device, n, name, args, schema }`, e.g. `task.moveToSprint { task, sprint, at }` or `page.setParent { page, parent, edge }`. **Mutators** are pure TypeScript functions in the shared core that turn an operation into database writes. They read only the database and their arguments: ids, timestamps and random values are generated **before** the operation, by the device that creates it, and travel in `args`. Given the same state, a mutator always does the same thing on every device.
2. **One total order per workspace.** A sequencer assigns each pushed operation the next sequence number. It does not need to read the operation: in **blind** mode it orders ciphertext.
3. **Optimistic apply and rebase.** A device applies its own operations at once and keeps them as *pending*, with an undo log captured by triggers. When ordered operations arrive, it:
   - undoes its pending operations;
   - applies the ordered ones, its own included;
   - re-applies the ones still pending.

   This is Replicache's and Linear's model, done on SQLite.
4. **Rules hold everywhere.** The database enforces the rules (checks, partial unique indexes, triggers). If an operation breaks a rule, its transaction fails, and **it fails the same way on every device**, because every device has the same state at that point in the order. A failed operation is a no-op everywhere and is shown to its author: "This page can't move inside its own sub-page." Nobody has to arbitrate.
5. **Designs that make conflicts rare.**
   - **Deterministic ids for singletons:** the journal for a date, the sprint for a week and the link for a URL get an id derived from what they are (UUIDv5 of workspace + kind + key). Two offline "open today's journal" operations become the same object, and the second is a no-op.
   - **Find-or-create mutators** for anything unique.
   - **Rituals are idempotent:** `sprint.close { week }` does nothing if that week is already closed. Two devices that closed the same week offline converge; the second close is a no-op.
6. **Rich text is a CRDT inside an operation.** A page body is a Yjs document. `body.update { object, update }` carries a Yjs update, and applying updates in any order gives the same document. The editor (BlockNote) speaks Yjs natively. The derived JSON and plain text (`body_text`, used by search and embeddings) are recomputed on apply.
7. **Bootstrap and compaction.**
   - A new device downloads an **encrypted snapshot**: a `VACUUM INTO` copy of the database at sequence *n*, plus the ordered operations after *n*.
   - Any device holding the key can write a snapshot.
   - The sequencer keeps the operations since the latest snapshot, plus the snapshot. That way neither storage nor bootstrap time grows forever.
8. **Versioning.**
   - Each operation records the schema version of the device that made it.
   - A device never applies an operation from a newer schema than its own: it stops syncing and asks to update.
   - Migrations are mutators too, run at a known sequence number, so old and new devices still agree.
9. **The event log is derived.** The timeline (who changed what, when) is written by triggers as each operation is applied, exactly as in Sprint. It records the operation id, so history can always be traced to an intent.

## The two server modes

The sequencer is the same program in both modes. What differs is whether it holds the workspace key.

| | **Trusted hub** (your own box, Mac mini, VPS) | **Blind relay** (hosted, or a hub without the key) |
|---|---|---|
| Holds the workspace key | yes: it is just another device | no |
| Stores | the log, snapshots and a materialised database | ciphertext operations, snapshots and file chunks |
| Validates operations before ordering | yes: it rejects rule-breaking operations early | no: devices reject them deterministically |
| Runs jobs (daily rollover, embeddings, link fetching, AI rituals) | yes, around the clock | no: the devices run them |
| Serves the web client | yes, over the private network | only as static files plus the log |
| A breach exposes | everything on that box (it is yours) | metadata only: sizes, timing, counts |

- **Both modes speak one protocol.** A user can start with no server, add a relay, and later add a trusted hub, with no migration.
- **Selling later:** the hosted service is the blind relay. It can't read customers' data, which keeps costs and liability low.

## Why this and not …

| Option | Why not (for this project) |
|---|---|
| **PowerSync** + own write API | Needs a trusted Postgres and a PowerSync service for every user. That means no E2EE, no "Mac only" mode, a server bill that grows with users, and the device is a replica rather than the engine. It is the best choice if we wanted a server-centric app. |
| **ElectricSQL** / **Zero** | Read-path or online-only engines (Zero has no offline writes); same trusted-server model |
| **Automerge / Loro documents** | Cross-record rules can't be expressed; we'd rebuild a relational projection beside them anyway |
| **Actual Budget-style per-cell LWW** | Proven and E2EE, but rules become "repair after merge" |
| **LiveStore** (event sourcing on SQLite, rebase, Cloudflare sync) | **Closest to this design.** Pre-1.0 (0.4.0, 2026-06), TypeScript only. Spike S3 evaluates adopting it instead of building. |
| **CloudKit** | Apple only, no web, no tenancy of our own |

## What spike S3 must prove

- **Zero divergence:**
  - 10,000 seeded fuzz runs with 3–5 devices (random offline periods, random operations, concurrent rich-text edits) end with identical state hashes and identical page texts on every device;
  - every invariant holds, and no operation is lost.
  - *A first prototype passes this; see [`benchmarks.md`](benchmarks.md) → Sync prototype.*
- **Rebase cost:** undo 50 pending operations, apply 500 incoming, re-apply 50, with p95 under 50 ms on a 100k-object database on an M-series Mac.
- **Crash safety:** killing the app at random points during apply, rebase and snapshot (1,000 runs) never corrupts the database or loses an acknowledged operation.
- **Bootstrap:** a 100k-object workspace reaches a new device in under 30 s on a 50 Mbit/s link.
- **Build or adopt:** the same fuzz suite run on LiveStore. Adopt it only if it passes, supports the encryption hook and doesn't fight the schema.
