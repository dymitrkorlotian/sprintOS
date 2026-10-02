# ADR-0001: The founding architecture

- **Status:** Proposed
- **Date:** 2026-10-02
- **Scope:** every technology choice sprintOS starts with, for the Mac app, the optional hub, and later the web and the phone.
- **Evidence:** research notes and benchmarks in [`docs/architecture/`](../architecture/README.md). Numbers below come from there; each section links its note.

## Summary

**The shape in one paragraph.** sprintOS is a Mac app whose data lives on the Mac in one SQLite file per workspace. A Rust core owns everything that must be correct: the ontology, every write, the event log, sync, search, encryption and the AI gateway. The UI is React in a Tauri 2 window, so the same UI runs in a browser later. Every change is an event in an append-only log; a deterministic reducer folds the log into the SQLite tables the app queries. Devices sync by exchanging encrypted events, directly or through an optional **hub**: the same Rust core in a server binary, which the user can run on a home box, a Mac mini or a €6 VPS, or later rent from us. The hub is **blind by default** (it stores and forwards ciphertext) and becomes **trusted** for a workspace only when the user hands it that workspace's key, so it can run rituals, link fetching and the AI agent while the Mac sleeps. Nothing requires a server; the Mac alone is a complete product.

```
 Mac app (Tauri 2)                                  optional hub (same Rust core, headless)
 ┌──────────────────────────────┐                   ┌──────────────────────────────────────┐
 │ React UI · ProseMirror editor│                   │ blind: store + forward ciphertext    │
 │ ─────── typed commands ───── │   encrypted       │ trusted (per workspace, by key grant):│
 │ Rust core                    │◀── events over ──▶│  reducer + SQLite replica            │
 │  ontology · reducer · jobs   │   iroh (QUIC,     │  jobs: rituals, link fetch, embeddings│
 │  sync · crypto · AI gateway  │   P2P or relay)   │  AI agent · MCP (HTTP) · Telegram    │
 │  SQLite: events + projection │                   │  backups of its own state            │
 │  + FTS5 + vectors · blobs/   │                   └──────────────────────────────────────┘
 └──────────────────────────────┘                        ▲ later: web (wasm core) and phone
```

### The stack

| # | Decision | Choice | Runner-up | Why |
|---|---|---|---|---|
| 1 | App shell | **Tauri 2** (system WebKit) | Electron | Published (rough) measurements: bundles ~25× smaller and idle memory ~4× lower than Electron; deny-by-default IPC; Rust backend; same UI runs on the web; iOS path exists. Electron stays the fallback if WebKit's IME or suspend bugs can't be fixed |
| 1 | Where the logic lives | **Rust core** (one workspace of crates), used by the app, the hub, wasm for web and UniFFI for phone | TypeScript core on Bun | One copy of every rule on every device and the hub; the CRDT, crypto, SQLite and networking libraries are Rust-native |
| 1 | UI | **React 19 + React Compiler, Vite, strict TypeScript** | Svelte 5 | The editor kits and keyboard tooling are React-first |
| 1 | Editor | **ProseMirror via Tiptap 3 (MIT) + loro-prosemirror**, with our own block UI; try BlockNote's UI on Loro first | BlockNote + Yjs | ProseMirror is the only engine with bindings to Loro, Yjs and Automerge; Loro gives movable blocks |
| 2 | Local database | **SQLite 3.53+** via rusqlite: WAL, `STRICT`, FTS5, vectors in the same file | Turso Database (same file format) when it reaches 1.0 | Measured fastest on every query and write; 35–46 ms cold open; stable file format since 2004; runs on Mac, iOS, Linux and wasm |
| 2 | Integrity | **One write path in the core** (typed commands → events → reducer), SQLite `CHECK`s, FKs and triggers as a backstop, post-merge rules in the reducer | Triggers only | Sync merges need deterministic rules a trigger can't express; triggers still catch core bugs |
| 2 | Encryption at rest | **Encrypted SQLite (SQLCipher or SQLite3MultipleCiphers) on by default** if the spike shows ≤ 20% overhead; key in the Keychain wrapped by the Secure Enclave; on top of FileVault | FileVault only | Protects copies and backups of the file, not just a powered-off disk |
| 3 | Sync and data model | **Own event log** (intent events, HLC order, per-device sequence numbers) **+ deterministic reducer → SQLite**; **Loro** documents for page bodies, carried as event payloads | Loro/Automerge documents as the truth | One home for the cross-object rules (one parent, one active sprint edge…); needs no server; works with a blind hub; the log *is* the context graph's timeline |
| 4 | Topology | **Mac alone works fully. Optional hub = the same binary; blind by default, trusted per workspace by key grant.** Async delivery with no hub through a "mailbox" (the user's bucket or iCloud) | Trusted hub only | Meets "never require a server" and "$0", and still offers background AI to those who want it |
| 4 | Network | **iroh 1.x** (QUIC, dial by public key, hole punching, relays); the hub runs its own relay | Tailscale (kept as an advanced option) | Embeds in the app and the hub, no account, no VPN profile, $0 |
| 5 | Security | Workspace keys with epochs, device keys (Ed25519/X25519), HPKE key wrapping, XChaCha20-Poly1305, signed events, Argon2id recovery kit; parsers in a sandboxed helper; signed updates | – | See the threat model |
| 6 | Identity and tenancy | **No account to use the app.** Devices pair by QR code. Accounts (passkeys) only for a hosted hub. **Workspace = sync unit = encryption unit = tenant unit**; one SQLite file per workspace on every node | Postgres + RLS on a hosted hub | Physical isolation per tenant; the same code on the Mac and the hub |
| 7 | Background work | **A durable job table in SQLite** (leases, retries, idempotency keys), run by the trusted hub or else the Mac; sprint close and recurrences are **deterministic events** | Inngest-like hosted queue | No service to run; works offline; duplicates are harmless by design |
| 8 | AI | **One AI gateway in the core**: bring-your-own Anthropic key in the Keychain, per-feature models, usage ledger, budgets, prompt caching, Batch API for rituals. **Agent = deterministic jobs + one bounded, read-only tool loop for Ask**; it writes only suggestions. **MCP server** (stdio on the Mac, HTTP + OAuth on the hub). Telegram capture on the hub | Claude Agent SDK | Small attack surface; "AI proposes, you confirm" by construction |
| 8 | Embeddings | **Local Qwen3-Embedding-0.6B by default** (Apache-2.0; 8-bit weights; 512 dimensions), built on the Mac or trusted hub; Voyage API as an opt-in | EmbeddingGemma-300M; voyage-4-lite by default | Best open model on every public slice (MMTEB 64.3, Polish 71.8); offline, private, $0. voyage-4-lite scores ~5.6 points higher on shared tasks, so the spike decides with real Polish queries |
| 9 | Files | Content-addressed blob store (BLAKE3), FastCDC chunks, per-chunk encryption, lazy sync | Whole-file sync | Dedupe and resumable sync of large files |
| 10 | Backups | **`VACUUM INTO` snapshot + restic-format repository** (hub, bucket or disk), weekly automatic restore test shown in Settings | Litestream (kept on the hub) | Encrypted, versioned, deduplicated; restore is proven, not assumed |
| 11 | Distribution | **Developer ID + notarisation, direct download, Tauri updater** (signed), Homebrew cask; App Store later; **no telemetry**, opt-in crash reports | Sparkle 2; Mac App Store | No review risk for agent features; sandbox-ready anyway |
| 11 | Hub deployment | One OCI image + compose file, a NixOS module and a cloud-init script; health shown in the app | – | "Everything as code" |
| 12 | Automations | **No rules engine in v1**: built-in rituals, signed webhooks, MCP; n8n stays outside | Built-in rules engine | Small surface; MCP covers power users |
| 13 | Licence | **FSL-1.1-ALv2** (source available; each release becomes Apache-2.0 after two years), protocol libraries MIT/Apache — **the user's call** | AGPL-3.0 + CLA | Shows the code, allows free self-hosting, blocks resale of the hosted hub |
| 15 | Plan | **Spikes first** (8 spikes, about 5–6 weeks), then a foundation and parallel build tracks | – | Every risky choice above has a go/no-go test |

### Top risks

1. **Building our own sync protocol.** It is the heart of the product and the place where data loss would happen. Mitigation: a small protocol (events + version vectors), a pure reducer, and a deterministic simulation and fuzz harness that must pass before any user data is trusted to it (spike S1).
2. **WebKit inside Tauri:** IME and shortcut conflicts, the page process being suspended in the background, and the engine version following macOS. Mitigation: spike S2 on real input methods; Electron is a contained swap because the UI talks to the core through one interface.
3. **Young dependencies:** loro-prosemirror (v0.4), iroh (1.0 in June 2026), sqlite-vec (one maintainer). Mitigation: each sits behind an interface, has a named fallback (Yjs path, Tailscale or self-hosted relay, plain BLOB scan), and gets a spike.
4. **Key loss is permanent.** End-to-end encryption means nobody can recover data without a device or the recovery kit. Mitigation: recovery kit at setup, reminders, and a "trusted hub" that can hold a copy of the key if the user wants.
5. **Rust slows a one-person team.** Mitigation: the spike records hours per feature; domain rules could move to TypeScript on Bun later without touching storage, sync or crypto.
6. **Prompt injection from saved pages** has no complete fix. Mitigation: the agent can't send data anywhere and can't write except through suggestions the user accepts.
7. **The query planner, not the engine, is SQLite's weak spot**: three naive queries in the benchmark were 25–2,000 ms until fixed. Mitigation: every query lives in the core with a plan test and a latency budget at 100k objects.
8. **A trusted hub on a cheap VPS is the most valuable target.** Mitigation: blind by default, SSRF guards, parsers in a sandbox, encrypted disk, signed updates.

## Context

- Sprint, the web app, proved the product: sprints as first-class objects, an ontology and context graph with provenance and an event log, an editable ontology (ADR-0005 in Sprint), and AI that drafts rituals. Its stack (Next.js on Vercel, Supabase Postgres with RLS, Drizzle, Inngest, BlockNote, Voyage and Claude) is an input, not a constraint.
- What Sprint's domain needs, in detail, is in [`domain-requirements.md`](../architecture/domain-requirements.md): the integrity rules the database enforces today, every background job, every AI call, volumes (10k–100k objects for a heavy user) and the lessons not to relearn.
- Decided by the user: an empty database (no migration from Sprint), no export feature for now, never require a home server, about $0 running cost, a data model that can serve many tenants later.
- The bar: optimized, secure, performant and stable, decided from first principles and on evidence.

## 1. App architecture and shell

Research: [`research-shell-ui-editor.md`](../architecture/research-shell-ui-editor.md).

| Shell | Perf | Security | Stability | Dev speed | Fit | Cost | Lock-in | Mobile | Web reuse |
|---|---|---|---|---|---|---|---|---|---|
| **Tauri 2** (2.12, Sept 2026) | 4 | 5 | 4 | 4 | 5 | 5 | 4 | 3 | 4 |
| Electron 44 | 2 | 3 | 5 | 5 | 4 | 5 | 4 | 1 | 5 |
| Native SwiftUI + AppKit | 5 | 5 | 5 | 2 | 2 | 5 | 2 | 4 | 1 |
| Flutter / Compose MP / Electrobun / Dioxus | 3–4 | 3–4 | 2–4 | 3–4 | 2–4 | 5 | 2–3 | 1–5 | 2–4 |

**Choice: Tauri 2.** The UI is a web app in the system WebKit view; the backend is our Rust core, linked into the app. Native SwiftUI would be the best Mac app, but it gives nothing for the web, and no CRDT-aware block editor exists for it; we would build one on TextKit 2 (months) and a second UI anyway. Electron is the most predictable runtime (one Chromium everywhere) but costs 3–4× the memory and start time and forces a major upgrade every few months.

**Where the logic lives: a Rust core.** It owns storage, the ontology, every domain command (`create_task`, `close_sprint`…), the reducer, sync, crypto, search, embeddings, jobs and the AI gateway. TypeScript owns views, view state, optimistic UI and the editor. The rule: **anything the hub or another device needs goes in Rust.** The core is used as a library by the Tauri app, as a binary by the hub, as wasm by the web client, and through UniFFI by a phone app. TypeScript types for the command API are generated from Rust, never written by hand. Sprint's `packages/core` (about 10.6k lines of pure TypeScript, 520 tests) is the specification to port; its tests are ported first.

**UI: React 19 with the React Compiler** (stable since October 2025), Vite, strict TypeScript, TanStack Query-style caching over core queries, cmdk, a drag-and-drop library, and Radix-style primitives styled with Tailwind. The UI talks to the core through one `CoreClient` interface (Tauri IPC on the Mac, a worker with the wasm core on the web), so the shell can change without touching the UI.

**Editor: ProseMirror.** It is the only engine with bindings for all three serious CRDTs (y-prosemirror is stable; loro-prosemirror v0.4.4 is young but active; automerge-prosemirror is beta). Because the sync design picks Loro (section 3), the editor is **Tiptap 3 (MIT) + loro-prosemirror**. The spike first tries BlockNote's MPL-2.0 UI on top of loro-prosemirror; if that needs a fork, we build the block UI (slash menu, side menu, drag handles, nesting) on Tiptap's MIT UniqueID and DragHandle, an estimated 3–6 weeks. BlockNote's GPL `xl-*` packages are never used. The page document lives twice: in the webview (bound to ProseMirror) and in the core (durable copy, sync, search); both run the same Loro code (Rust, and its wasm build), and exchange binary updates over a Tauri channel.

**What would change it:**
- WebKit IME, shortcut or background-suspend problems that the spike can't fix → Electron, same UI, the core as a Node addon.
- loro-prosemirror fails the spike → the Yjs path: BlockNote + Yjs in the webview and yrs in the core.
- Rust slows the team badly (measured in the spike) → domain rules move to TypeScript on Bun; storage, sync and crypto stay in Rust.
- The user wants a fully native Mac feel and accepts a separate web UI → SwiftUI shell with the core over UniFFI and the editor as a web view.

## 2. Local storage

Research: [`research-storage.md`](../architecture/research-storage.md). Benchmark: [`bench-storage.md`](../architecture/bench-storage.md) (100k objects, 200k edges, 300k events, 100k 512-dimension vectors; a 4-vCPU x86 VM, so a pessimistic bound for a Mac).

| Engine | Perf | Security | Stability | Dev speed | Fit | Cost | Lock-in |
|---|---|---|---|---|---|---|---|
| **SQLite 3.53** (+ FTS5, sqlite-vec) | 5 | 4 | 5 | 4 | 5 | 5 | 5 |
| Turso Database 0.8 (Rust rewrite, SQLite format) | 4 | 4 | 2 | 4 | 5 | 5 | 4 |
| PGlite 0.5.8 (Postgres 18 in wasm) | 2 | 3 | 2 | 4 | 4 | 5 | 3 |
| Embedded native Postgres | 3 | 4 | 4 | 3 | 3 | 5 | 4 |
| SurrealDB, CozoDB, Kùzu, Realm, a custom KV engine | – | – | 1–3 | – | – | – | – |

Measured, p50 / p95 in ms (SQLite · PGlite · native Postgres 16):

| | SQLite | PGlite | Postgres |
|---|---|---|---|
| Sprint board | 0.50 / 1.37 | 2.51 / 16.1 | 0.75 / 3.79 |
| All neighbours of an object | 0.05 / 0.15 | 1.61 / 6.92 | 0.51 / 1.62 |
| Full text, one common word | 22.3 / 39.4 | 142 / 309 | 52 / 98 |
| New task + 2 edges, with triggers | 0.19 / 1.20 | 3.13 / 12.8 | 1.63 / 10.7 |
| Cold open + first query | 35–46 | 850–970 | 100–107 (server already running) |

**Choice: SQLite**, bundled (≥ 3.53.x, which also has the fix for the 2026 WAL-reset corruption bug), through rusqlite in the core, with WAL, `STRICT` tables, `foreign_keys=ON` checked at open, one writer connection and a small reader pool. It is the fastest engine on every query and write we measured, usually by 2–10× over native Postgres and 5–20× over PGlite, with no server to run and a file format stable for 20 years. The same file and code run on the Mac, the iPhone, a Linux hub and (through SQLite's wasm build) the web.

- **PGlite is rejected:** slowest on everything, 0.9 s to open, and it runs with **`fsync=off`** (we checked `current_setting('fsync')` on 0.5.8; PGlite issue #1107), so an acknowledged write can be lost on a crash. Its 0.4 → 0.5 upgrade could not read old data files.
- **An embedded Postgres is rejected:** a second process, 6 s recovery after an unclean stop, impossible on iOS, and slower for this workload.
- **Full text:** FTS5 with `unicode61 remove_diacritics 2`, titles weighted over bodies, plus a trigram index on titles for substring search (0.4 ms p95). One common word ranks in 20–40 ms; search-as-you-type ranks titles first. Polish stemming is a spike item (Snowball gained Polish only in October 2025).
- **Vectors in the same file.** Exact search with sqlite-vec 0.1.9: 122 ms over 100k × 512 float32 on the VM, 64 ms at 256 dimensions; a heavy user has about 15k chunks (Sprint's estimate), about 18 ms. Store int8 or 256-dimension vectors; add an approximate index (sqlite-vec DiskANN or SQLite's `vec1` once stable, or HNSW in the core) past about 50k chunks.
- **Query discipline:** three naive queries were slow (a recursive CTE joined in the wrong order, an `OR` join over hub nodes, a partial index that couldn't serve a query) and were 100–600× faster once fixed. Every query is written in the core with an `EXPLAIN QUERY PLAN` test and a latency budget at 100k objects.

**Integrity**, in four layers (Sprint enforces its rules with Postgres triggers, checks, RLS and advisory locks; none of that exists on a phone, and sync merges need rules that a row-level trigger can't express):
1. **One write path.** Every write (UI, AI suggestion accepted, import, sync) is a typed command validated against the ontology, turned into an event, and applied by the reducer in one `BEGIN IMMEDIATE` transaction.
2. **The reducer enforces cross-object rules deterministically** (section 3), so merged states from two offline devices converge to the same valid state.
3. **SQLite backstop:** FKs to the ontology tables, `CHECK`s (enums, ranges, key formats, `json_valid`), and a few triggers for the invariants whose breach corrupts the graph: relation from/to types, cardinality, append-only events, no deleting ontology rows. Measured cost: under a millisecond per write.
4. **Checks:** `PRAGMA integrity_check`, `foreign_key_check` and an ontology consistency check at start-up and nightly, with results in the log.

**Encryption at rest:** encrypted SQLite on by default, SQLCipher or SQLite3MultipleCiphers (ChaCha20), chosen by spike S3, as long as hot queries cost no more than 20% extra. The key is a random 256-bit key in the Keychain (`WhenUnlockedThisDeviceOnly`), wrapped by a Secure Enclave key. FileVault alone protects a powered-off Mac but not a copy of the file, a backup or another process of the same user. If the overhead is too high, we fall back to FileVault plus encrypted backups and say so.

**Schema evolution:** `PRAGMA user_version` and migrations embedded in the binary, run at open after a `VACUUM INTO` backup; SQLite 3.53 can now add and drop `NOT NULL` and `CHECK` without the 12-step rebuild. The local schema is private to each device and can be rebuilt from the log; what must stay compatible across app versions is the **event format** (section 3). The editable ontology is data, so most product changes need no migration.

**What would change it:** Turso Database reaching 1.0 with stable concurrent writes and encryption (it reads the same files, so switching is cheap); `vec1` or sqlite-vec ANN going stable when a user passes ~50k chunks; a hosted product with shared, many-writer workspaces (then Postgres on the hosted hub, SQLite on devices).

## 3. Sync and the data model

Research: [`research-sync.md`](../architecture/research-sync.md). Benchmark: [`bench-crdt.md`](../architecture/bench-crdt.md).

| Design | No server needed | Offline writes | Cross-object rules | Rich text | Blind hub | Timeline | Build effort | Lock-in |
|---|---|---|---|---|---|---|---|---|
| A. CRDT documents are the truth (Loro/Automerge per object), SQLite as an index | 5 | 5 | 2 | 5 | 4 | 3 | 3 | 4 |
| **B. Event log + deterministic reducer → SQLite; Loro for page bodies** | 5 | 5 | 5 | 5 | 5 | 5 | 3 | 5 |
| C. Server-authoritative (PowerSync, Zero, Electric) | 1 | 1–3 | 5 | 2 | 1 | 3 | 4 | 2 |
| Packaged frameworks (LiveStore, Jazz, Evolu, Triplit, cr-sqlite, Ditto, Instant) | varies | varies | varies | 1–3 | varies | varies | – | 2–3 |

**Choice: B.** The truth is an append-only log of **intent events** ("add task T to sprint S", "move page P under Q", "set field `budget` on X to 40", "page body update: <Loro bytes>"). A pure, versioned reducer folds the events, in one total order, into the SQLite tables the app queries.

- **Each event** carries a UUIDv7 id, `workspace_id`, `device_id`, a gap-free per-device `seq`, a hybrid logical clock (HLC), a type and `schema_version`, the actor (user, system, AI, automation) and a payload. Events are signed with the device key. Total order: (HLC, device id, seq). HLCs more than a day in the future are clamped, so one bad clock can't win every conflict.
- **Sync** is exchanging version vectors (`device → highest seq`) and sending the missing events. It is resumable, works over any transport, and needs no plaintext on the relay.
- **Most events commute.** Field writes are last-writer-wins registers keyed by HLC, and edge additions are idempotent, so a late event usually applies without replay. Only events under an ordering rule (cardinality, tree moves, sprint close) make the reducer rewind to the last checkpoint before the event and replay. Checkpoints are snapshots at known log positions.
- **Cardinality rules** (one active `in_sprint` per task and sprint, one parent per page, one `part_of` project, and custom relation types' *one* sides) are applied by the reducer in total order: the later edge wins, the earlier one is ended with `outcome = superseded` and shown to the user. Nothing is silently dropped. **Tree moves** use Kleppmann's move algorithm: a move that would create a cycle is skipped and recorded. The rules come from the ontology tables, so custom relation types get the same treatment with no code.
- **Rituals are deterministic events.** The sprint close for week W is an event with a fixed id (from workspace and W) and a fixed HLC (Monday 00:00 in the workspace time zone). Whichever device writes it first, every device sorts it at the same place and the reducer computes its effects (completed, carried over, missed) from the state at that point. An offline edit made on Sunday and synced on Tuesday sorts *before* the close, and the replay recomputes the close correctly. Recurring occurrences get ids from (series, occurrence time) in the same way, which keeps Sprint's "one occurrence per series and moment, forever" rule.
- **Page bodies are Loro documents** (a movable list of blocks, each with Peritext-style rich text). A block moved on one device and edited on another ends up moved *and* edited, which a plain text CRDT can't do. Loro update bytes travel as event payloads; the reducer stores the document and derives plain text, mentions, links and block ids into SQLite. Old Loro history is trimmed with shallow snapshots.
- **Event volume stays small.** Typing doesn't produce an event per keystroke: the editor's Loro updates are coalesced into one body event every few seconds of editing (and at blur), and old body history is compacted into shallow snapshots. Graph events (fields, edges, rituals) are small and kept forever, because they are the timeline. Rough size: one heavy user writes tens of thousands of graph events a year, a few MB.
- **Time zone changes are events too.** The sprint boundary for week W is computed from the workspace time zone in effect at that point in the log, so a device that learns of a time zone change late still computes the same boundary.
- **A blind hub can withhold events but not forge them.** Version vectors show a device what it is missing, and every event is signed by its author's device key.
- **The ontology is data in the log** (custom fields, types and relation types), so it syncs and needs no app update. Event shapes only grow; old versions are upcast; a device keeps events it doesn't understand, skips them in its tables, asks for an update, and rebuilds after upgrading.
- **The log is the context graph's timeline.** One structure serves sync, history, undo, audit, "what changed this week" and AI context, which Sprint built separately with triggers.

**Why not the others.** Server-authoritative engines need an always-on server with Postgres, which breaks "never require a server" and makes a blind hub impossible. CRDT documents as the truth make cross-object rules fragile: two devices repairing a merged state can disagree and fight. The packaged frameworks are TypeScript-only, alpha, proprietary, restrictively licensed or winding down (Instant's cloud closes in 2027, Replicache was archived, cr-sqlite upstream stalled). After 2025–2026's churn among sync startups, owning a *small* protocol reduces risk.

**CRDT for page bodies: Loro 1.16** (MIT, Rust, Swift bindings, movable tree and list, shallow snapshots, time travel). Measured on a real 260k-keystroke editing trace ([`bench-crdt.md`](../architecture/bench-crdt.md)):

| | Loro 1.16 | Yjs 13.6 | Automerge 3.5 |
|---|---|---|---|
| Apply the trace | **1.08 s** | 2.0 s | 37.8 s |
| Load the page / memory after load | **14 ms / +1.6 MB** | 51 ms / +3.3 MB | 3.6 s / +207 MB |
| Size with history; shallow snapshot | 231 KB; **65 KB** | 160–311 KB, no history kept | **129 KB**; none |
| One keystroke on the wire | 85 B | **15 B** | 93 B |
| Concurrent page moves (X under Y while Y under X) | **valid tree** | cycle or lost pages | cycle or lost pages |

Loro is fastest to load, keeps history, has shallow snapshots and is the only one with a real move. **Fallback: Yjs** (the BlockNote path in section 1): smaller payloads and faster merges, but no history and no block move. Automerge is out for the JavaScript side: 38 s to replay one page and 3.6 s to load it. No library should hold the whole graph in one document (Yjs needed 651 MB and 3.4 s to load 100k objects), which confirms the graph belongs in SQLite and only bodies in CRDTs. Two lessons for the build: free Loro handles created in loops (each holds ~7 KB until freed), and after a long offline period send a snapshot or one combined update rather than thousands of small ones.

**What would change it:** the user accepts that a server must always be up (then PowerSync with Postgres is less to build); a mature native framework that does B appears; the spike shows replay too slow on a phone (then A for bulk data, B only for edges); Loro shows merge bugs under fuzzing that aren't fixed quickly (swap page bodies to Yjs; the log design stays).

## 4. Topology: the Mac plus an optional hub

Research: [`research-topology-security.md`](../architecture/research-topology-security.md).

| Option | Perf | Security | Maturity | Dev speed | Fit | Cost | Lock-in |
|---|---|---|---|---|---|---|---|
| No hub (devices sync directly) | 4 | 5 | 4 | 5 | 2 | 5 | 5 |
| Blind hub only (relay) | 4 | 5 | 3 | 3 | 3 | 4 | 5 |
| Trusted hub only | 5 | 2 | 4 | 4 | 5 | 4 | 5 |
| **Hybrid: no hub works; hub blind by default, trusted per workspace** | 4 | 4 | 3 | 3 | 5 | 5 | 5 |

**Choice: the hybrid.**

- **The Mac alone is complete.** Editing, search, rituals (when the Mac is awake), AI with the user's key, backups to a disk or bucket.
- **The hub is the same Rust core** in a headless binary. It runs on a home server, a Mac mini, a cheap VPS (Hetzner's smallest is about €5.50–6 a month after the June 2026 price rise; to re-check) or, later, our hosted service. Setting it up is pairing, like adding a phone.
- **Blind by default:** it stores and forwards encrypted events and blobs and acts as an always-online peer for the phone and the web. It learns sizes, timing and workspace ids, nothing else.
- **Trusted by explicit grant, per workspace:** the user wraps that workspace's key to the hub's device key. The hub then keeps a plaintext replica and runs the jobs: rituals on schedule, link fetching, embeddings, the AI agent, the MCP server over HTTP and Telegram capture. Revoking trust rotates the workspace key. No comparable product offers this switch: Tana and Capacities read everything to do server-side AI; Obsidian, Anytype, Standard Notes and Ente are end-to-end encrypted and do everything on the device.
- **With no hub and the Mac asleep,** the phone keeps working on its own replica, and changes meet when both are online. For delivery while the Mac sleeps at $0, a **mailbox** holds encrypted event batches: the user's own S3-compatible bucket (Cloudflare R2's free tier is 10 GB) or, for Apple-only users, their iCloud private database through CloudKit (spike S7). Either is just storage for ciphertext.
- **The web client** is the same React UI with the core compiled to wasm, keeping an encrypted replica in the browser's private storage (OPFS) and unlocking the workspace key with a passkey (PRF) or a pairing code. That keeps it working with a blind hub; its storage limits are an M3 spike.
- **The web client needs an online peer** (browsers can't open raw UDP or hole-punch), so it talks to a hub over HTTPS or WebSockets. That is honest: the web is a hub feature.

**Network: iroh 1.x** (1.0 shipped June 2026): QUIC connections addressed by public key, hole punching, encrypted relays when a direct path fails, Swift and Kotlin bindings, $0 public relays (rate-limited, no SLA), and a relay the hub can run itself. The device's iroh identity is its Ed25519 key. Tailscale is a documented option for power users (the hub can listen on a tailnet address) but never a requirement: it means another app, an account and a VPN profile on the phone. Embedding a tailnet (tsnet) is Go-only.

**What would change it:** iroh failing on mobile carrier networks in spike S6 (self-hosted relay first, then Tailscale); most users wanting the trusted hub (make trusted the default for hosted plans, keep blind for self-hosters); an Apple-only audience (CloudKit becomes the default mailbox).

## 5. Security model

Research: [`research-topology-security.md`](../architecture/research-topology-security.md) → threat model.

| Threat | Main mitigations |
|---|---|
| Lost or stolen laptop or phone | FileVault / iOS Data Protection; encrypted SQLite, key in the Keychain wrapped by the Secure Enclave; remote revoke from another device, which rotates the workspace key |
| Breached hub or sync server | Blind by default: ciphertext only, names and paths included; events signed by device keys so a hub can't forge or reorder history; a trusted hub is opt-in per workspace and documented as able to read that workspace |
| Malicious link or file content (parser and image-decoder bugs) | Fetching and parsing run in a **separate sandboxed helper** (an XPC service on the Mac, a locked-down process on the hub) with no keys and no database access; memory-safe parsers; images re-encoded; size and time limits. Saved pages are never rendered as live HTML in an app window |
| SSRF from the link fetcher | Port Sprint's guarded fetcher: block loopback, private, link-local, CGNAT and cloud-metadata addresses at connect time and on every redirect; http(s) only; no cookies; 5 s and 1 MB caps |
| Prompt injection from saved pages | The agent never has untrusted content, private data and a way to send data out in one session: it has no web-fetch or send tool, its only write is `create_suggestion`, answers render without loading remote images or links, and anything derived from untrusted text shows its provenance and needs the user's accept |
| Compromised agent or MCP misuse | Per-tool scopes (read, suggest); no delete tools; every write is an event and can be undone; MCP clients allowlisted; rate limits; the agent never sees keys |
| Malicious dependency | pnpm with a 7-day `minimumReleaseAge`, `trustPolicy: no-downgrade` and blocked install scripts; a small JavaScript tree; `cargo-deny`, `cargo-vet` and a review of any new crate with a build script or proc macro; committed lockfiles; pinned CI actions. 2025–2026 saw repeated npm worms (Shai-Hulud, its "Second Coming" and "Mini" waves, axios, keyv) and a crates.io hijack (`arrayref`, August 2026) |
| Compromised update channel | Signed updates (Ed25519) with the key offline, Apple notarisation, the public key pinned in the app |

**Keys.** Each workspace has a random 256-bit key with an epoch number; synced data is sealed with XChaCha20-Poly1305. Each device has Ed25519 (signing, and its iroh identity) and X25519 keys. Adding a device or granting a hub wraps the workspace key to it with HPKE (RFC 9180) after a QR or short-code pairing. Removing a device or untrusting a hub creates a new epoch. A recovery kit (24 words or a printable page) unwraps the key through Argon2id; losing every device and the kit loses the data, and the app says so plainly. Local keys sit in the Keychain, wrapped by a Secure Enclave P-256 key. Syncing keys through iCloud Keychain is opt-in. Libraries: one audited stack (RustCrypto or aws-lc-rs), never hand-rolled constructions. MLS is not needed until real shared workspaces exist.

**Sandboxing.** Developer ID, Hardened Runtime and notarisation from day one. App Sandbox on if spike S2 shows file watching, the parser helper and iroh all work inside it (that also keeps the Mac App Store open). Tauri: deny-by-default capabilities per window, a strict CSP, the isolation pattern, and no remote origin ever loaded in a window that can call the core.

## 6. Auth and tenancy

- **No account to use the app.** First run creates a workspace (with its key) and a device key pair. The person's identity is the set of their paired devices, as in Signal.
- **Pairing** a phone or a self-hosted hub is a QR code or one-time code; no server account is involved.
- **A hosted hub (later, paid)** has accounts for billing and routing only, never for decrypting data: passkeys first, an email link to recover the *account* (not the data), Sign in with Apple or Google optional. The passkey PRF extension can unlock the workspace key in the web client.
- **Workspace = sync unit = encryption unit = tenant unit.** A person can have several (Work, Personal), each with its own key and its own hub trust setting. Every event carries `workspace_id`; objects keep UUIDv7 ids, globally unique, as in Sprint.
- **Many tenants later:** the hosted hub keeps **one SQLite file per workspace**, running the same core as the Mac, plus a small control database for accounts, devices and billing. Isolation is physical (a bug can't leak rows across files), and backup, restore and delete per tenant are file operations. This is now a mainstream pattern (Cloudflare Durable Objects with SQLite, Turso, 37signals' `activerecord-tenanted`). Postgres with RLS returns only if shared workspaces with many concurrent writers appear.

## 7. Background work

Replaces Inngest and Vercel cron. Research: [`research-ai-jobs.md`](../architecture/research-ai-jobs.md) → background work.

| Option | Perf | Security | Maturity | Dev speed | Fit | Cost | Lock-in |
|---|---|---|---|---|---|---|---|
| **Job table in the workspace's SQLite** (leases, retries, backoff, idempotency keys) | 5 | 5 | 4 | 4 | 5 | 5 | 5 |
| Hosted queue (Inngest-like) | 4 | 3 | 5 | 4 | 2 | 4 | 2 |
| Redis-backed queue (BullMQ, apalis + Redis) | 4 | 4 | 5 | 3 | 2 | 4 | 3 |

- **Where jobs run:** on the trusted hub when there is one, else on the Mac while it's awake. The phone never runs rituals. One runner holds a lease per workspace for scheduled AI work; if two devices both run a job anyway, its deterministic key makes the second a no-op.
- **The jobs** (from Sprint): the sprint close and rollover, recurring occurrences, embeddings (debounced per object), link metadata fetching with retries, the weekly summary, insights and plan drafts, the nightly integrity check, backups and the weekly restore test.
- **Time:** the sprint boundary stays lazy and idempotent (Sprint's `ensureSprintState`), and the close is a deterministic event (section 3), so a Mac that was closed all weekend closes the week correctly on Monday at 9:00. Sprint's tested ISO-week, time-zone and DST functions are ported first.
- **On the Mac,** an optional login item (`SMAppService`) keeps the core running with the window closed; `NSBackgroundActivityScheduler` spreads heavy work. On iOS nothing is relied on in the background.
- **Crash and error reports:** off by default. Apple's crash reports and MetricKit on the Mac and phone; for the hub, an opt-in Sentry-protocol endpoint the user can point at a self-hosted Bugsink or GlitchTip. No product analytics.

## 8. AI

Research: [`research-ai-jobs.md`](../architecture/research-ai-jobs.md); embeddings benchmark: [`bench-embeddings.md`](../architecture/bench-embeddings.md).

- **Keys:** bring your own Anthropic key (and an embeddings key if the API is chosen), stored in the Keychain (not synced) or in the trusted hub's secret store. A hosted plan later would use our own metered key, never a proxy of users' keys.
- **One gateway** in the core for every model call. It holds per-feature model and effort settings (the model is config, as in Sprint), checks the budget before each call (notice at 80%, pause at 100%), records a ledger row from each response's `usage` fields (feature, model, tokens, cost, prompt version), turns prompt caching on by default, and sends rituals through the Batch API at half price. Rough cost for one heavy user: about $0.12 a week for rituals; Ask, at a few cents a question, is what the budget is for.
- **The agent is mostly deterministic code.** Rituals (sprint close, retro draft, rollover, weekly plan, triage) are jobs: code gathers the facts and the model drafts text or picks moves from a closed list through structured outputs, which code validates, as Sprint's Monday plan does. **Ask your OS** is one bounded tool loop on the plain Messages API: read-only tools (`search`, `get_object`, `neighbours`, `sprints`, `journal`, `insights`), a turn cap and a token budget, an answer with object citations. The only write tool anywhere is `create_suggestion`; accepting it runs the same command a manual edit would. Every run writes an audit row. The Claude Agent SDK is not used: it brings a coding harness (files, shell, web) that is the wrong shape and the wrong attack surface for a personal data server.
- **Local LLMs don't carry the rituals.** Apple's on-device model has an 8K context and Apple Intelligence doesn't support Polish; tag, kind and duplicate suggestions stay embedding-neighbour votes with no LLM, as in Sprint.
- **Embeddings: local by default, behind the gateway**, with the model pinned per workspace and stored on every row; the Voyage API is an opt-in. Vectors are derived data, never events: the Mac or trusted hub computes them, and other devices either receive the vector table as an encrypted derived snapshot or embed only their queries. The default is **Qwen3-Embedding-0.6B**: it leads the open models on multilingual MTEB (64.3), its Polish slice (71.8) and English retrieval (59.4), is Apache-2.0, takes 32K tokens and can be cut to 512 dimensions (about 1 KB a chunk, about 20 MB for 20k chunks). On a CPU it is slow (about 200 tokens/s on the test VM, so a first index of 5M tokens would take hours), but published MLX numbers on Apple Silicon are about 44K tokens/s, so the Mac builds the index and a CPU-only hub doesn't. EmbeddingGemma-300M is 2–3 points lower and 2–3× faster (Gemma licence, not OSI). voyage-4-lite is about 5.6 points higher on the tasks both were scored on (vendor-run) and costs about $0.10 for a whole personal corpus, so cost doesn't decide; privacy and offline use do. Spike S5 runs the real models on real Polish and English queries before this is final. See [`bench-embeddings.md`](../architecture/bench-embeddings.md); its speed numbers come from the published architectures with random weights, because the test VM couldn't download model weights.
- **The semantic index lives in the same SQLite file** (section 2), re-embedded in the background when the model changes; hybrid search (FTS + vectors, reciprocal rank fusion, as in Sprint) covers the gap.
- **MCP server:** the core exposes objects and the graph through scoped tools: read tools and `create_suggestion`, never delete. Locally over stdio for Claude Desktop and Claude Code; on a trusted hub over Streamable HTTP with OAuth. Inbound MCP clients are allowlisted per workspace; the lessons from 2025–2026's MCP vulnerabilities (command injection in stdio servers, over-broad tokens) are listed in the research note.
- **Telegram** for capture and chat, from the hub (or the Mac) by long polling, so no public address is needed; only allowlisted user ids; replies go only to the asker. WhatsApp, Signal and iMessage are out for now.

## 9. Files

- A **content-addressed blob store**: files named by their BLAKE3 hash under the workspace folder, metadata in SQLite, so identical files are stored once.
- Large files are split with **FastCDC** into chunks, so uploads resume and an edit resends only changed chunks. Each chunk is sealed with XChaCha20-Poly1305 under a per-blob key; on a blind store, objects are named by a keyed hash so the store can't tell whether it holds a known file.
- **Lazy sync:** metadata syncs at once, bytes on demand or by a per-device policy. Garbage collection uses tombstones and a grace period so a blob still referenced by an unsynced device is never deleted.
- Thumbnails and previews via QuickLook; objects are indexed in Spotlight (Core Spotlight) for system-wide search.
- **The live database never sits in iCloud Drive or Dropbox**: they copy the database, WAL and shared-memory files separately and corrupt it. It lives in Application Support (or the sandbox container).

## 10. Backups

Distinct from export. Research: [`research-files-ops-licence.md`](../architecture/research-files-ops-licence.md) → backups.

| Option | Perf | Security | Stability | Dev speed | Fit | Cost | Lock-in |
|---|---|---|---|---|---|---|---|
| **restic-format repository** (bundled restic 0.19 now, `rustic_core` in-process later) | 4–5 | 5 | 5 | 4 | 5 | 5 | 5 |
| Kopia | 4 | 5 | 4 | 4 | 3 | 5 | 4 |
| Litestream only | 5 | 3 | 4 | 5 | 2 | 5 | 4 |
| Time Machine only | 3 | 4 | 4 | 5 | 2 | 5 | 3 |

- Every hour when something changed, and at quit: `VACUUM INTO` a consistent snapshot of each workspace file. Then back up the snapshot and the blob store into a restic-format repository: on the hub (rest-server in append-only mode), the user's bucket, or an external disk. Blobs dedupe for free.
- Encrypted with a repository password in the Keychain, with its own recovery path. Retention: 48 hourly, 30 daily, 12 weekly, 24 monthly.
- **A weekly automatic restore test:** restore the latest snapshot to a temporary folder, open it, run `integrity_check`, compare counts and a sample of blob hashes, and show "last verified restore: 3 days ago" in Settings.
- The hub also streams its own databases with Litestream. Time Machine is a bonus; the live database is excluded from it (snapshots are included).
- Export stays out of scope, but it comes nearly free: the event log, the SQLite tables and the blob folder are an open, documented format.

## 11. Distribution and operations

- **Apple Developer Program** ($99 a year): Developer ID signing, hardened runtime, `notarytool` and stapling in CI.
- **Direct download first**, plus a Homebrew cask. The Mac App Store is possible later because we build sandbox-ready, but its guideline 2.5.2 (no running downloaded code) was used against AI app-builders in March 2026, so agent features would face review risk.
- **Updates:** the Tauri updater (Ed25519/minisign signatures, full downloads of a small app). Sparkle 2 is the runner-up if delta updates start to matter.
- **Telemetry:** none. The only call home is the update check, which can be turned off. Crash reports are opt-in.
- **The hub from code:** one small OCI image (static Rust binary), a compose file, a NixOS module and a cloud-init script for a VPS, with Tailscale optional. The app shows the hub's health (version, last sync, last job, last backup and restore test, disk) from a `/status` endpoint.
- **Testing:**
  - unit and property tests in Rust (proptest) and Vitest for the UI;
  - a **deterministic simulation** of 3–5 devices running the real reducer and Loro over a network that drops, duplicates, delays and reorders messages, with crashes and clock skew (spike S1 builds it; it then runs in CI on every change to the core);
  - `cargo-fuzz` on the reducer, the event decoder and the parsers;
  - a `kill -9` crash harness for SQLite and the blob store;
  - Playwright against the web build of the UI, plus a short macOS smoke suite through an embedded WebDriver (the official tauri-driver has no macOS support);
  - restore tests in CI.

## 12. Automations

No rules engine in v1. The built-in rituals cover what Sprint does by hand today. Power users get signed outbound webhooks (events they choose), an inbound capture endpoint on the hub, and the MCP server, which lets any MCP-capable assistant or n8n act through the same scoped tools. n8n's Sustainable Use License forbids bundling or hosting it for others, so it stays outside. **What would change it:** repeated requests for "when X, do Y" inside the app; then a small, declarative rules engine that emits suggestions, built on the event log.

## 13. Licensing and business

| Model | Trust | Stops resale of the hosted hub | Contributions | Fits dependencies |
|---|---|---|---|---|
| Closed | 2 | 5 | 1 | 5 |
| MIT / Apache-2.0 | 5 | 1 | 5 | 4 |
| AGPL-3.0 + CLA | 4 | 4 | 3 | 3 |
| **FSL-1.1-ALv2** (Apache-2.0 two years after each release) | 4 | 5 | 2 | 4 |
| Open core | 3 | 3 | 3 | 4 |

- **Recommendation: FSL-1.1-ALv2** for the app and the hub, with all copyright owned by the user (a CLA for any outside contribution), and MIT/Apache-2.0 for libraries we split out (the event format, the sync protocol, a CLI). It shows the code, lets anyone self-host the hub for free, stops a third party selling our hosted hub for two years, and never strands users. **This is the user's call**; AGPL-3.0 + CLA is the pick if an "open source" label matters from day one.
- **Dependencies:** everything chosen is permissive (SQLite public domain, Loro and iroh MIT/Apache, Tiptap core MIT, sqlite-vec MIT/Apache, SQLCipher BSD-style, restic BSD-2). BlockNote's `xl-*` packages (GPL-3.0 or commercial) are not used. CI checks licences with `cargo-deny` and a JavaScript licence checker.
- **Business:** the hosted hub (sync, backup, background AI) as a subscription, priced against Obsidian Sync ($4–8 a month); self-hosting stays free. The model already serves many tenants. "Sprint" is a crowded trademark; search "Sprint OS" in EUIPO and USPTO before any public launch.

## 14. Repository and what carries over

- **Copy from Sprint into `docs/requirements/`** (a separate small PR): `vision.md`, `architecture/context-graph.md`, ADR-0005 and `specs/objects.md`, all of `docs/specs/`, `modules.md`, and the Resolved table of `open-questions.md`. They are requirements, not design.
- **Port, tests first** (TypeScript → Rust, from Sprint's `packages/core`): ISO weeks, time zones and DST; the sprint engine (close, carry-over, metrics); recurrences (the RRULE subset); quick-capture parsing; the ontology and field validation; link URL canonicalisation, kind detection and metadata parsers; mentions extraction; the page tree; chunking and reciprocal rank fusion; tag normalisation; the Markdown converter. The guarded fetcher (`fetch-page.ts`, `address.ts`) is ported with every guard. Sprint's SQL migrations are the integrity specification: each rule becomes a reducer rule or a SQLite backstop with a test.
- **Not ported:** Next.js, Supabase, Drizzle, Inngest, BlockNote wiring, RLS policies.
- **Sprint's web app** keeps running until sprintOS covers it. sprintOS's own web client (through a hub) and phone app then replace it; there is no data migration, as decided.
- **Repo layout** (when building starts): `crates/` (`core`, `store`, `reducer`, `sync`, `crypto`, `index`, `ai`, `jobs`, `net`, `hub`), `app/` (Tauri shell), `ui/` (React), `docs/`. One workspace, one CI.

## 15. Execution plan

Building starts after Sprint's testing phase (5 Oct – 1 Nov 2026) and its UI/UX pass. Spikes come first; each has a go/no-go. A "no-go" switches to the named fallback, not to a new search.

### Phase 0: spikes (about 5–6 weeks, mostly in parallel)

| # | Spike | Proves | Go if | Fallback if no-go |
|---|---|---|---|---|
| S1 | **Sync core and simulation** | The event format, HLC, version-vector sync, the reducer with cardinality and tree-move rules, the sprint close as a deterministic event, Loro bodies | Over ≥ 10⁶ simulated runs (3–5 devices; drops, duplicates, reordering, partitions, crashes between append and fsync, ±1 day clock skew, two reducer versions): **zero lost acknowledged events**, byte-identical projections, invariants hold, rebuild from empty equals incremental, applying twice changes nothing. Replay of 1M events < 60 s on an M1; a week-late event applies in < 1 s | Design A for bulk data with the reducer only for edges; or Yjs instead of Loro for page bodies |
| S2 | **Shell, editor and WebKit** | Tauri 2.12 + React 19 + Tiptap/BlockNote on loro-prosemirror, the document mirrored in the core over a binary channel; App Sandbox on | Cold start to an editable page < 1 s (target 600 ms); idle memory with the WebKit process < 150 MB with a 1,000-block page; typing p95 < 16 ms in a 10,000-block page; Polish, Ukrainian, Japanese and Chinese input with no doubled or dropped characters and no shortcut firing during composition; recovers after 20 min in the background under memory pressure; offline Mac + browser edits converge with block ids and mentions intact | Electron; or the Yjs path |
| S3 | **Storage on a Mac** | Sprint's schema and integrity rules in SQLite; the benchmark rerun on Apple Silicon with rusqlite; encryption overhead; crash safety | At 100k objects: every interactive query < 50 ms p95 (most < 5 ms); a command commits < 5 ms p99 while a job writes 10k chunks; SQLCipher or SQLite3MultipleCiphers within +20% on hot queries; 1,000 `kill -9` runs with zero corruption and zero lost acknowledged commits | FileVault only, with encrypted backups |
| S4 | **Rust core speed for one person** | Port the sprint engine, recurrences and time functions with their tests; generate TypeScript types | Ported with all of Sprint's tests passing in ≤ 1.5× the time a TypeScript port is estimated to take | Domain rules in TypeScript on Bun; storage, sync, crypto in Rust |
| S5 | **Embeddings and search** | Local models vs Voyage on 200 real queries in Polish and English; FTS5 with and without Polish stemming | Local model recall@10 within 5 points of the API; query embedding < 50 ms on an M1; indexing a heavy user's corpus < 10 min | API by default when a key exists, local model as the offline fallback |
| S6 | **iroh and keys** | Mac (home Wi-Fi) ↔ iPhone (mobile network) ↔ VPS hub; the whole key flow (pair, grant and revoke a hub, rotate, restore from the recovery kit) | Connects in < 3 s in 95% of 50 tries; direct in ≥ 70%; the revoked hub can't read the new epoch; restore works with no server account | Self-hosted relay, then Tailscale |
| S7 | **Mailbox** | Phone writes encrypted batches to R2 and to CloudKit while the Mac sleeps; the Mac drains them on wake | No loss over 1,000 batches; defined behaviour when the quota is full | Hub only for async delivery |
| S8 | **Hostile input and prompt injection** | The sandboxed fetch-and-parse helper; Ask with read-only tools | Fuzzed HTML, PDF and images never crash outside the helper; every SSRF case blocked; 20 poisoned saved pages cause zero unconfirmed tool calls and zero outbound requests | Stricter: no AI over web content until fixed |

After the spikes, this ADR is updated with the results and the fallbacks taken, and goes to the user for **Accepted**.

### Phase 1: foundation (about 4 weeks, two or three agents)

Sequential, because everything builds on it: the Rust workspace, CI (lint, tests, `cargo-deny`, `cargo-vet`, the licence check, the simulation), the event format and reducer from S1, the SQLite schema and migrations from S3, the ontology tables seeded with Sprint's built-in types and relations, the command API and TypeScript type generation, the Tauri shell with the `CoreClient` interface, the design tokens from Sprint's brand (ADR-0003 in Sprint).

### Phase 2: parallel build tracks

Each track owns its paths and works against the command API, so several agents can run at once. Sizes are rough, for one agent.

| Track | Contents | Size |
|---|---|---|
| T1 Sprints and tasks | Board, Home, Capture, the sprint engine (ported), recurrences, the close and rollover as deterministic events, metrics | L |
| T2 Pages and journal | The editor (from S2), page tree, mentions and backlinks, templates, journal | L |
| T3 Library and files | Links (guarded fetcher in the helper, metadata, kinds), the blob store, previews, Spotlight | M |
| T4 Ontology admin | Custom fields, types and relation types (Sprint's ADR-0005 rules, in the reducer) | M |
| T5 Search and AI | FTS, the semantic index and embeddings, hybrid search, the AI gateway and ledger, rituals, Ask, suggestions | L |
| T6 Sync, devices and keys | Pairing, iroh transport, version-vector sync, key management, recovery kit, the mailbox | L |
| T7 Hub | The headless binary, jobs, MCP over HTTP, Telegram, `/status`, container, NixOS module, cloud-init | M |
| T8 Backups and ops | Snapshots, the restic repository, restore tests, signing, notarisation, updater, Homebrew cask | M |

**Milestones:** M1, the Mac app replaces Notion and Sprint for planning, used daily on one Mac (T1–T5, T8). M2, phone and hub (T6, T7, a phone app on the same core). M3, the web client through a hub; Sprint's web app is retired.

## Consequences

- ✅ Fast, private and offline by default: measured sub-2 ms p95 for interactive queries at 100k objects, no server in the loop.
- ✅ One Rust core with every rule, on every device and the hub; one event log that is sync, history, undo, audit and AI context.
- ✅ No running cost unless the user chooses a VPS hub or a paid plan; no vendor whose shutdown would strand the data.
- ✅ The hub can be blind or trusted, per workspace, which no comparable product offers.
- ⚠️ We own the sync protocol and the reducer. Their correctness rests on the simulation harness, which must exist before user data does.
- ⚠️ Two languages (Rust and TypeScript) and a WebKit runtime that changes with macOS.
- ⚠️ End-to-end encryption makes lost keys unrecoverable.
- ⚠️ The web client needs a hub; a Mac-only user has no browser access.
- ⚠️ Sprint's TypeScript domain code is a specification to port, not code to reuse.

## Open items for the user

1. The licence (section 13): FSL-1.1-ALv2 (recommended) or AGPL-3.0 + CLA.
2. Whether the default for a new user's own hub is blind (recommended) or trusted.
3. Whether to enrol in the Apple Developer Program now, so spikes can test the sandbox, notarisation and a TestFlight phone build.
