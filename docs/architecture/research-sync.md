# Research: sync and the data model

Status: research input for ADR-0001. Written 2026-10-02. All URLs accessed 2026-10-02 unless a source date is given.

## TL;DR

> **Update after review:** loro-prosemirror 0.4.4 uses a plain `LoroList`, not `LoroMovableList`, so "moved *and* edited" (section 4a) needs a fork of the binding. The protocol rules that came out of the red-team review are in [`sync-protocol.md`](sync-protocol.md).
>
> **Update after measuring** ([`bench-crdt.md`](bench-crdt.md)): Automerge 3.5 in JavaScript took 38 s to replay one page's editing trace and 3.6 s to load it, so the fallback for page bodies is **Yjs**, not Automerge. Loro stays the pick.

- **Recommendation: an event log as the source of truth, reduced by a deterministic reducer into SQLite (the queryable store). Rich-text bodies are Loro documents whose update blobs travel as payloads inside the same log.** The hub, when there is one, is just another peer: it stores and forwards encrypted events. It can be blind (E2EE) or trusted (holds the key and runs AI jobs); the user chooses.
- Do **not** adopt a server-authoritative engine (PowerSync, Zero, Electric, Convex, Supabase Realtime). They all need an always-on server with Postgres (or similar), which breaks "never require a home server", and two of them have no offline writes.
- Do **not** adopt a packaged local-first framework as the core. The good ones are JS-only (LiveStore), alpha (Jazz 2.0), proprietary (Ditto), restrictively licensed (SQLite Sync), being wound down (Instant Cloud, Triplit) or unmaintained (cr-sqlite upstream). Copy their ideas, not their runtimes.
- **Loro** is the CRDT for page bodies: MIT, fastest in published traces, rich text with Peritext-style mark expansion, movable tree, shallow snapshots, time travel, Swift bindings. **Automerge 3** is the fallback (now memory-sane, stronger research lineage, E2EE sync work in progress).
- Cardinality rules (one parent, one active `in_sprint` per sprint, one `part_of` project) are enforced by the reducer after a total order (HLC + device id). Every device computes the same winner. The loser is kept as an ended edge with a "superseded" reason, never silently dropped.

## 1. CRDT libraries

### Status (Oct 2026)

| Library | Latest | Licence | Core | Native bindings | Notes |
|---|---|---|---|---|---|
| Loro | 1.16.x (Sept 2026) | MIT | Rust | Swift (`loro-swift`, tracks core: 1.16.0), Kotlin generatable via UniFFI (not packaged), Python, RN, C# | Movable tree, movable list, rich text, shallow snapshots, version checkout |
| Automerge | 3.4.1 (Aug 2026) | MIT | Rust | `automerge-swift` (0.7.x), `automerge-repo-swift`; no official Kotlin found (unverified) | 3.0 rebuilt memory model; same file format as 2.x |
| Yjs / Yrs | Yjs 13.6.31 stable (2026-05-28); v14 still RC (rc.26, Sept); Yrs 0.26.0 (2026-05-04) | MIT | JS; Rust port | `yswift` is "work in progress" | Largest editor ecosystem (ProseMirror, Tiptap, Lexical) |
| Diamond types / Eg-walker | research-grade | ISC/MIT (unverified) | Rust | none | Text only. Plain text + event graph; CRDT built only while merging |

Sources: Loro releases https://github.com/loro-dev/loro/releases, loro-swift https://github.com/loro-dev/loro-swift/releases/tag/1.16.0, loro-ffi https://github.com/loro-dev/loro-ffi, Loro MIT https://crates.io/crates/loro; Automerge 3.3–3.4 https://automerge.org/blog/2026-august/ and https://automerge.org/blog/2026-july/; automerge-swift https://github.com/automerge/automerge-swift/releases; Yjs versions https://github.com/yjs/yjs/releases and https://github.com/iasakura/cert-yjs/issues/208; Yrs https://lib.rs/crates/yrs; yswift https://github.com/y-crdt/yswift; Eg-walker (Gentle and Kleppmann, EuroSys 2025) https://github.com/josephg/eg-walker-reference.

### Performance (published numbers)

- **crdt-benchmarks** (https://github.com/dmonad/crdt-benchmarks), real editing trace B4. *Old versions*: Yjs 13.6.11, Loro 0.10.1, Automerge 2.1.10, so Automerge numbers predate 3.0.

  | B4 | Yjs | Loro | Automerge 2 |
  |---|---|---|---|
  | apply time | 5.7 s | 3.1 s | 14.3 s |
  | doc size | 160 KB | 258 KB | 129 KB |
  | parse (load) time | 39 ms | 13 ms | 1,805 ms |

  B4x100 (16–26 MB docs): Yjs parse 2.6 s and 327 MB memory, Loro parse 1.3 s; Automerge skipped.
- **Automerge 3.0** (https://automerge.org/blog/automerge-3/, 2025): uses the compressed columnar format in memory. Memory down >10x, sometimes 100x. Moby Dick pasted: 700 MB in 2.x vs 1.3 MB in 3.0. A doc that took 17 hours to load now opens in 9 s. So the 1.8 s B4 parse above is stale; a re-run of current versions is needed (spike item).
- **Loro 1.0** (https://loro.dev/blog/v1.0): shallow snapshot import ~0.37 ms on their example; shallow snapshot 869 B vs 5,421 B full. (loro.dev blocked from this sandbox; figures via search excerpt, so treat as vendor numbers.)
- Eg-walker paper claims an order of magnitude less steady-state memory than CRDTs and much faster load, because it keeps plain text plus an event log.

Takeaway: for a personal workspace (thousands of pages, each a few KB to a few hundred KB), all three are fast enough. Load time and memory per open doc matter more than raw merge speed. Loro and Automerge 3 both load in milliseconds at our sizes.

### Rich-text merge semantics

- **Peritext** (Litt et al., CSCW 2022, https://www.inkandswitch.com/peritext/) defines how formatting marks merge: bold/italic *expand* when you type at their edge; links and comments do *not*. Implemented in Automerge (marks API) and Loro.
- **Loro** stores marks as "style anchors" with per-style expansion behaviour (before/after/both/none), configured per key (https://loro.dev/blog/loro-richtext). Text uses Fugue, which avoids interleaving of concurrent inserts.
- **Yjs** formatting is attributes on items; concurrent format at boundaries is less principled than Peritext, but it is battle-tested inside editors.
- **Block moves**: none of them gives "move a block with its content" in a text sequence. Model blocks as a **movable list or movable tree of block containers** (Loro `LoroMovableList`/`LoroTree`, each block holding a `LoroText`). Then a concurrent move + edit keeps both: the block moves and the edit lands in it. Moving by delete-and-reinsert loses concurrent edits. This is the decisive reason to prefer Loro for a Notion-style editor.

### History and time travel

- Loro: `checkout(frontiers)` to any version, `fork_at`, diff between versions; shallow snapshots drop old history (like a git shallow clone).
- Automerge: every change hash-linked; view at any `heads`; full history kept (no shallow mode in core).
- Yjs: snapshots only work with GC disabled, which grows docs.

### Encryption-friendliness

All three exchange **opaque binary updates** that a relay can store as ciphertext. What the relay cannot do is merge or compact them; it can only store and forward. Compaction must happen on a device with the key (a device writes a new snapshot, encrypts it, uploads it, and the relay drops older blobs it supersedes). Automerge's Beelay/Keyhive (Ink & Switch, 2024–2026) is designing exactly this (E2EE sync with server-blind relays), but is **pre-alpha** (https://www.inkandswitch.com/project/keyhive/, https://github.com/automerge/beelay).

### Scoring (1–5, higher is better)

| | Perf | Security | Maturity | Dev speed | Domain fit | Cost | Lock-in |
|---|---|---|---|---|---|---|---|
| Loro | 5 | 4 | 4 | 4 | 5 (tree, movable list, rich text) | 5 | 4 |
| Automerge 3 | 4 | 4 | 4 | 4 | 4 (no movable list/tree) | 5 | 4 |
| Yjs/Yrs | 4 | 4 | 5 (JS) / 3 (Swift) | 4 | 3 | 5 | 4 |
| Eg-walker/DT | 5 | 4 | 2 | 2 | 2 (text only) | 5 | 3 |

**Pick Loro.** What would change it: Loro's team (small, VC-funded startup) goes quiet, or a fuzz spike finds merge bugs (there are recent ones, e.g. tree snapshot panic https://github.com/loro-dev/loro/issues/1088, fixed in 1.16.2–1.16.4). Fallback is Automerge 3, which has the deepest research backing.

## 2. Local-first sync frameworks

| Framework | Status Oct 2026 | Licence | Server needed? | Native (Swift) | Verdict |
|---|---|---|---|---|---|
| **LiveStore** | 0.4.0 (2026-06-02); event-sourced, reactive SQLite; sync backends incl. Cloudflare DO, S2 | Apache-2.0 (unverified) | Sync provider needed for multi-device; app works alone | No, TS only (web, Expo, Node) | **Best model to copy.** Library itself doesn't fit a native Mac app |
| **Jazz** | 2.0 alpha (alpha.5x), new relational API, rewrite | MIT; single-tenant server open source | Sync server (self-hostable) | No | Too unstable; just rewrote itself |
| **Triplit** | Acquired by Supabase (Oct 2025); not integrated, "open source everything" | open source | Server | No | Orphaned product |
| **Evolu** | Active; SQLite + CRDT, E2EE, blind relay, P2P possible | MIT (unverified) | Relay (blind, optional self-host) | No (TS) | Closest fit on values (E2EE, blind relay); TS only, LWW per cell, no rich text |
| **DXOS** (ECHO/Composer) | Very active (PRs on 2026-10-02); built on Automerge | MIT (unverified) | Edge/agent optional | No | Big framework, many opinions |
| **Ditto** | Commercial P2P mesh (BLE, Wi-Fi, LAN); Swift/Kotlin SDKs | Closed source, commercial | No (P2P), Big Peer optional | Yes | Proprietary and metered: fails $0 / lock-in |
| **cr-sqlite** | Upstream vlcn-io last npm release ~2 years ago; Fly.io fork (0.17, breaking) maintained for Corrosion | MIT | No | C extension | Upstream effectively unmaintained |
| **SQLite Sync** (sqliteai) | Active | Elastic License 2.0: non-production only without commercial licence | SQLite Cloud / Postgres | Yes | Licence disqualifies |
| **Turso** (embedded replicas + offline sync) | Offline writes + bidirectional sync in beta; RN bindings Jan 2026 | MIT core; sync via Turso Cloud | Turso Cloud (self-host sync unverified) | Partial | Whole-DB sync, conflict detection only, cloud-tied |
| **Instant** | Team joined OpenAI (2026-08-22); cloud shuts 2027-08-31; open source "but unmaintained" | Apache-2.0 (unverified) | Server | No | Dead end |
| **Convex** (local/self-host) | Open-source backend | FSL→Apache (unverified) | Server | No | Online-first; optimistic updates only |
| **PouchDB/CouchDB** | Mature, slow-moving | Apache-2.0 | CouchDB peer | No | Document revs + manual conflicts; no rich text |

Sources: LiveStore changelog https://docs.livestore.dev/changelog/ and event sourcing https://docs.livestore.dev/evaluation/event-sourcing/; Jazz https://github.com/garden-co/jazz and https://jazz.tools/docs/getting-started/server-setup; Triplit https://supabase.com/blog/triplit-joins-supabase; Evolu https://www.evolu.dev/ and https://github.com/finitoapp/evolu-live-relay; DXOS https://github.com/dxos/dxos/pull/13593; Ditto https://github.com/api-evangelist/ditto-live and https://ditto.live/pricing/cloud-sync; cr-sqlite https://github.com/vlcn-io/cr-sqlite and https://github.com/superfly/cr-sqlite; SQLite Sync https://github.com/sqliteai/sqlite-sync; Turso https://turso.tech/blog/turso-offline-sync-public-beta and https://turso.tech/blog/react-native-bindings-for-turso; Instant https://www.instantdb.com/essays/instant_team_joins_openai and https://runtimewire.com/article/instant-team-joins-openai-cloud-shutdown.

The pattern: 2025–2026 was hard on sync startups (Instant, Triplit, Replicache all wound down or archived). **Owning our sync protocol is a risk reducer, not gold-plating**, as long as the protocol stays small.

## 3. Server-authoritative sync engines

| Engine | Status | Licence | Always-on server? | Offline writes | Self-host cost |
|---|---|---|---|---|---|
| **ElectricSQL** | Active; read path only (shapes over HTTP) | Apache-2.0 | Yes, Electric + Postgres | Not built in; you build the write path | Postgres + Electric service |
| **PowerSync** | Stable; backends Postgres, MongoDB, MySQL, SQL Server (beta Mar 2026); SQLite client | Service: FSL ("Open Edition", source-available); SDKs Apache-2.0 incl. Swift and Kotlin | Yes, service + source DB + bucket storage DB | Yes (upload queue, server decides) | 3 processes minimum |
| **Zero** | 1.0 (June 2026), Rocicorp | Apache-2.0 | Yes, zero-cache + Postgres 15+ with logical replication | **No**: writes rejected while offline | Postgres + zero-cache |
| **Replicache** | Free, maintenance mode; repo archived 2026-06-10 | open source | Yes (your backend) | Yes | Your backend |
| **Supabase Realtime** | Active | Apache-2.0 | Yes | No (broadcast/changes only) | Supabase stack |

Sources: Electric writes guide https://electric-sql.com/docs/guides/writes; PowerSync open edition https://powersync.com/blog/powersync-open-edition-release, SDKs https://powersync.com/open-source, self-host config https://docs.powersync.com/configuration/powersync-service/self-hosted-instances, SQL Server beta https://releases.powersync.com/announcements/sql-server-backend-database-support-beta-release; Zero 1.0 https://www.infoq.com/news/2026/06/zero-version-1/, offline https://zero.rocicorp.dev/docs/offline, licence https://zero.rocicorp.dev/docs/open-source, Postgres needs https://zero.rocicorp.dev/docs/connecting-to-postgres; Replicache https://github.com/rocicorp/replicache.

All of these make the server the truth. Two devices can only converge through it, so the Mac and phone cannot sync while the hub is off, and a "hub" would need Postgres plus a sync service. That contradicts "never require a home server" and adds ops cost. **PowerSync is the only one worth keeping in mind**, for a later paid hosted tier with many tenants, because it is mature, has native Swift/Kotlin SDKs and real offline writes. Even then, it cannot hold E2EE data it must query.

## 4. Domain problems

### (a) The same page edited on Mac and phone offline

With Loro: each page is a `LoroDoc` with a movable list of blocks; each block has a `LoroText` with marks. Both devices edit offline; on reconnect, each imports the other's update blob. Result: all inserted characters kept, no interleaving of concurrent runs (Fugue), marks merged by Peritext rules, a block moved on one device and edited on the other ends up moved *and* edited. Concurrent delete + edit of the same block: the delete wins for the container; to avoid losing typed text, use soft delete (a `deleted` flag on the block) so the edit is still visible as "restored by concurrent edit". This is a product rule the spike must test.

### (b) Graph edges with cardinality rules

The rules: a task has at most one *active* `in_sprint` per sprint; a page has at most one `child_of` parent; an item is `part_of` at most one project. Concurrent offline edits can violate them (Mac moves page P under A, phone moves P under B; or both create the same edge).

Options:

1. **CRDT + post-merge repair.** Each device repairs the merged state (e.g. keep the edge with the highest timestamp). Repair writes are new ops, so two devices can repair differently and then fight. Works only if repair is a pure function of merged state and writes nothing (a *view-time* repair). Fragile.
2. **Hub-authoritative validation with rejection.** The hub orders writes and rejects losers. Needs the hub online and trusted (not blind), and the phone's offline work can bounce later. Breaks "no server required".
3. **Event log + deterministic reducer.** All devices sort events by the same total order (HLC, then device id, then sequence) and fold them through the same pure reducer, which enforces the rules. Same events in, same state out, on every device. The loser of a conflict is recorded as an ended edge with `outcome = superseded` and surfaced as a gentle notice. Nothing is lost, nothing needs a server.
4. **Intent operations.** Store intent ("move P under A", "add task T to sprint S") rather than state diffs. Combined with option 3 this gives meaningful conflict handling: a move that would create a cycle is skipped, exactly the Kleppmann et al. tree-move algorithm (undo-do-redo in timestamp order, proved in Isabelle; https://martin.kleppmann.com/papers/move-op.pdf).

Worked example (option 3 + 4):

```
Mac   (offline)  hlc=10:00:05.001 mac#41  edge.add  child_of(P → A)
Phone (offline)  hlc=10:00:07.300 ph#17   edge.add  child_of(P → B)
Both sync. Every device sorts: mac#41, then ph#17.
Reducer:
  mac#41 → P has no parent → add child_of(P → A), active
  ph#17  → P already has an active parent → end child_of(P → A) with outcome=superseded_by ph#17,
           add child_of(P → B), active
Result on every device: P under B; the A link is kept as history; UI shows
"P was moved to B on your phone after you moved it to A on your Mac".
```

Same pattern for `in_sprint` (rule key = task + sprint) and `part_of` (rule key = item). A move that would create a cycle (P under its own descendant) is skipped and recorded as `rejected: cycle`, as in Kleppmann's algorithm. Rule keys come from the ontology (relation type has `max_out = 1`, optional scope), so custom relation types get the same treatment without code.

**Choose 3 + 4.** The page tree uses the Kleppmann move algorithm inside the reducer (Loro's `LoroTree` implements the same idea, so either the reducer or a single workspace-level `LoroTree` can own the hierarchy; the reducer is preferred so all graph rules live in one place).

### (c) The append-only event log across devices

- Each event: `id` (UUIDv7 or device id + seq), `device_id`, `seq` (per device, gap-free), `hlc` (Kulkarni et al. 2014, https://cse.buffalo.edu/tech-reports/2014-04.pdf), `type`, `schema_version`, `payload`, `actor` (user / system / AI) for provenance.
- Total order = (hlc, device_id, seq). HLC stays close to wall time, so "what happened when" in the Timeline layer is readable, and it respects causality across devices that have synced.
- Sync = exchange per-device version vectors (`device_id → max seq`) and send missing events. Simple, resumable, works over any transport, and the relay needs no plaintext.
- A late event (phone was offline a week) sorts into the past. The reducer rewinds to the last checkpoint before it and replays. LiveStore does the same ("rebase"). At personal scale (estimate tens of thousands of events per year; to verify in spike) replay is cheap. Checkpoints = SQLite snapshots at known log positions.
- Clock guard: reject or clamp HLCs far in the future (e.g. > 1 day ahead) to stop one bad clock from winning every conflict forever.
- The domain's Timeline layer *is* this log. One structure serves sync, history, undo, audit and AI context.

### (d) Schema and ontology evolution across app versions

- The editable ontology (custom fields, types, relation types) is **data in the log**, so it syncs like anything else and needs no app update.
- Event *shapes* evolve only additively. Each event carries `schema_version`; the reducer has upcasters for old versions.
- An old app that receives an event type or version it does not know **must keep it in the log, never drop it**, skip it in the projection, and show "update the app to see recent changes". After upgrade it rebuilds the projection from the log.
- Consequence to accept: two devices on different reducer versions can show different state for a while. They converge once both run the same reducer. Projections record the reducer version; a mismatch triggers rebuild.
- Never change the meaning of an existing event type. New meaning = new type.

### (e) Peer-to-peer vs hub

- The protocol is symmetric, so both work: Mac ↔ phone directly on a LAN (Bonjour / Network.framework; transport out of scope here), or via any always-on peer.
- Because the hub only stores and forwards events, it can be a tiny binary on a Mac mini or VPS, or a **dumb blob store** (an S3-compatible bucket, or possibly the user's own iCloud via CloudKit at $0; CloudKit fit for web is unverified). This keeps "never require a home server" and "about $0" true.
- A hub that runs AI rituals must read data, so it is a *trusted* peer holding the key.

### (f) E2EE: can the hub be blind?

Yes, with this design. Events are encrypted per workspace (e.g. XChaCha20-Poly1305, key held on devices). Only `workspace_id`, `device_id`, `seq` need to be plaintext for routing and gap detection; HLC and type can be inside the ciphertext. A blind hub cannot: run queries, compute embeddings, run AI agents, or compact logs. So compaction (snapshot upload) is done by a device. Key management (device enrolment, revocation, recovery) is the hard part and needs its own ADR; Keyhive is the reference to watch but is pre-alpha. Server-authoritative engines cannot be blind at all.

## 5. Recommended architecture

### Three concrete designs

**A. CRDT documents as truth.** Every object (task, page, sprint…) is a Loro/Automerge doc; edges live in docs too; SQLite is a materialised index rebuilt from docs. Sync = doc updates.

**B. Event log + deterministic reducer (recommended).** The log of intent events is truth; a pure reducer folds it into SQLite in a total order; page bodies are Loro docs whose update blobs are event payloads (`page.body_updated { doc_id, loro_update }`), materialised into SQLite as plain text for search plus the Loro snapshot for editing.

**C. Server-authoritative.** Postgres on a hub/cloud is truth; PowerSync (or Zero) keeps a SQLite replica on each device; server validates writes.

| Criterion | A. CRDT docs | B. Event log + reducer | C. Server-authoritative |
|---|---|---|---|
| Works with no server (Mac alone, or P2P) | 5 | 5 | 1 |
| Offline writes on every device | 5 | 5 | 3 (PowerSync) / 1 (Zero) |
| Cardinality rules, cross-object invariants | 2 (per-doc only; repair is fragile) | 5 (one reducer, deterministic) | 5 (server checks) |
| Rich text | 5 | 5 (Loro inside) | 2 (needs a CRDT anyway) |
| Blind hub / E2EE | 4 | 5 | 1 |
| Timeline / audit / AI context | 3 (derive from doc histories) | 5 (it is the log) | 3 (CDC) |
| Query performance | 4 (SQLite index) | 5 (SQLite projection) | 5 |
| Storage growth | 3 (per-doc history) | 3 (log + checkpoints; needs compaction policy) | 4 |
| Schema evolution | 3 (doc shape migrations are hard) | 4 (upcasters, rebuild) | 4 (server migrations) |
| Many tenants later | 3 | 4 (log per workspace) | 5 |
| Build effort | 3 | 3 (own reducer + sync; ~small protocol) | 4 |
| Lock-in | 4 | 5 | 2 |
| Running cost | 5 | 5 | 2 |

**Decision: B.** It is the only design where the hardest domain rules (cardinality, tree moves, provenance) have one obvious home, it needs no server, and it lets the hub be blind or trusted by choice. It also matches the domain: the context graph already *has* an append-only event log as a layer, so B uses one structure where A and C need two.

Shape of B:

```
UI ──intent──▶ append event (local SQLite: events table, fsync) ──▶ reducer ──▶ projection tables (objects, edges, fields, FTS)
                    │                                                   ▲
                    ▼                                                   │
             sync: version vectors, missing events (encrypted) ◀──▶ peer / hub / blob store
page editor ──▶ LoroDoc ──export update──▶ event payload (same log)
```

- One SQLite file per workspace on device: `events` (append-only), projection tables, checkpoints. Open, inspectable on-disk format.
- Reducer: pure, versioned, no clock or randomness, same code on every platform (Rust core shared via UniFFI to Swift/Kotlin and via WASM to web is the natural fit; platform choice is a separate research note).
- Many tenants later: `workspace_id` on every event and row; a hosted hub stores logs per workspace.

### What would change the decision

- If the user drops offline writes on the phone or accepts "a server must be up", C (PowerSync) becomes simpler to build.
- If a mature native framework appears that does B well (e.g. LiveStore ships a native/Rust core, or Evolu adds rich text and native SDKs), adopt it instead of building.
- If the spike shows late-event replay is too slow on phone for realistic logs, move to A for bulk data and keep the reducer only for graph edges.
- If Loro shows merge bugs in fuzzing that are not fixed quickly, swap the body CRDT to Automerge 3 (the log design does not change).

### Risks and unknowns

- Log growth and compaction with a blind hub (devices must do it). Policy: checkpoint + prune Loro history with shallow snapshots after N months; keep graph events forever (they are small and are the Timeline).
- Reducer bugs are global: a nondeterministic reducer diverges every device. Mitigation: property tests below, and a projection hash exchanged during sync to detect divergence early.
- Reducer version skew (d) makes temporary divergence by design. Must be visible to the user, not silent.
- Key management for E2EE is unsolved here.
- Benchmarks above use 2023-era versions for Automerge/Loro; numbers must be re-measured.

### What the fuzz / property-test spike must prove

Harness: 3–5 simulated devices running the real reducer and real Loro, a simulated network that drops, duplicates, delays and reorders messages, random partitions, random crashes mid-write (kill between append and fsync), clock skew up to ±1 day, and two reducer versions mixed. Random ops: create/edit/delete objects, add/end edges including conflicting cardinality edits, tree moves incl. cycle attempts, concurrent rich-text edits incl. block move + edit and delete + edit, custom-field/ontology changes.

Properties (all must hold over ≥10⁶ random runs):

1. **Zero data loss**: every event a device acknowledged to the user exists in every device's log after full sync; every character typed appears in the merged text or in a recorded deletion event.
2. **Convergence**: after full exchange, every device's projection is byte-identical (hash of a canonical dump) and Loro docs have equal version vectors and equal text.
3. **Invariants**: in every projection, at most one active `in_sprint` per (task, sprint), one parent per page, no cycles, one `part_of` project.
4. **Replay equivalence**: rebuilding from an empty DB equals the incrementally maintained projection, for any arrival order.
5. **Idempotence**: applying any event twice changes nothing.
6. **Crash safety**: after a crash at any point, the log is a valid prefix and the projection rebuilds.
7. **Version skew**: an old reducer keeps unknown events; after upgrade it converges with the others.

Performance targets to measure (on an M1 Mac and a mid-range phone): replay 1M events from empty; rewind-and-replay for a late event 1 week old; open a 200-block page; memory with 50 pages open; sync of 10k missing events over a blind relay.
