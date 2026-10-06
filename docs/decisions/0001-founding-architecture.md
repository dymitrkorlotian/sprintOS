# ADR-0001: The founding architecture

- **Status:** Proposed
- **Date:** 2026-10-02 (revised the same day after the red-team review on PR #1)
- **Scope:** every technology choice sprintOS starts with, for the Mac app, the optional hub, and later the web and the phone.
- **Evidence:** research notes and benchmarks in [`docs/architecture/`](../architecture/README.md); the sync contract in [`sync-protocol.md`](../architecture/sync-protocol.md). Each section links its note.

## Summary

**The shape in one paragraph.** sprintOS is a Mac app whose data lives on the Mac in one SQLite file per workspace. A Rust core owns everything that must be correct: the ontology, every write, the event log, sync, search, encryption and the AI gateway. The UI is React in a Tauri 2 window, so the same UI can run in a browser later. Every change is a signed event in an append-only log; a deterministic reducer folds the log into the SQLite tables the app queries. Devices sync by exchanging encrypted events, directly or through an optional **hub**: the same Rust core in a server binary, which the user can run on a home box, a Mac mini or a €6 VPS, or later rent from us. The hub is **blind by default** (it stores and forwards ciphertext) and becomes **trusted** for a workspace only when the user hands it that workspace's key, so it can run rituals, link fetching and the AI agent while the Mac sleeps. Nothing requires a server; the Mac alone is a complete product, and the first milestone is exactly that.

```
 Mac app (Tauri 2)                                  optional hub (same Rust core, headless) — M2
 ┌──────────────────────────────┐                   ┌──────────────────────────────────────┐
 │ React UI · ProseMirror editor│                   │ blind: store + forward ciphertext    │
 │ ─────── typed commands ───── │   encrypted       │ trusted (per workspace, by key grant):│
 │ Rust core                    │◀── events over ──▶│  reducer + SQLite replica            │
 │  ontology · reducer · jobs   │   iroh (QUIC,     │  jobs: rituals, link fetch, embeddings│
 │  sync · crypto · AI gateway  │   P2P or relay)   │  AI agent · MCP (HTTP)               │
 │  SQLite: events + projection │                   │  backups of its own state            │
 │  + FTS5 + vectors · blobs/   │                   └──────────────────────────────────────┘
 └──────────────────────────────┘                        ▲ later: web (M3) and phone (M2)
```

### The stack

| # | Decision | Choice | Runner-up | Why |
|---|---|---|---|---|
| 1 | App shell | **Tauri 2** (system WebKit); spike S2 also tries Tauri's CEF (Chromium) runtime | Electron | Deny-by-default IPC, a Rust backend in-process, small bundles, the same UI on the web, an iOS path. Memory is *not* a reason: with WebKit's processes counted, Tauri may use as much as Electron (tauri#5889); S2 measures it |
| 1 | Where the logic lives | **Rust core** (one workspace of crates), used by the app, the hub, wasm for web and UniFFI for phone | TypeScript core on Bun | One copy of every rule on every device and the hub; the CRDT, crypto, SQLite and networking libraries are Rust-native |
| 1 | UI | **React 19 + React Compiler, Vite, strict TypeScript** | Svelte 5 | The editor kits and keyboard tooling are React-first |
| 1 | Editor | **ProseMirror via Tiptap 3 (MIT)** with our own block UI, bound to **Loro** through loro-prosemirror | BlockNote + Yjs | ProseMirror has bindings for Loro and Yjs. Neither binding moves a block while keeping a concurrent edit (loro-prosemirror uses a plain list); S2 decides between forking it and accepting delete + insert |
| 2 | Local database | **SQLite 3.53+** via rusqlite: WAL, `synchronous=FULL` with `fullfsync`, `STRICT`, FTS5, vectors in the same file | Turso Database (same file format) when it reaches 1.0 | No server; runs on Mac, iOS, Linux and wasm; a file format stable since 2004; fastest or equal on every measured query (1.2–10× over native Postgres, mostly under 2× for indexed graph queries; writes ≈1.3× at equal durability) |
| 2 | Integrity | **One write path in the core** (typed commands → signed events → reducer), SQLite `CHECK`s, FKs and triggers as a backstop, conflict rules in the reducer | Triggers only | Sync merges need deterministic rules a trigger can't express; triggers still catch core bugs |
| 2 | Encryption at rest | **SQLCipher on by default** (rusqlite `bundled-sqlcipher`) if S3 shows acceptable cost on cold reads and writes; key in the data-protection Keychain (`AfterFirstUnlockThisDeviceOnly`), wrapped by the Secure Enclave where there is one; on top of FileVault | SQLite3MultipleCiphers; FileVault only | Protects copies and backups of the file, not just a powered-off disk |
| 3 | Sync and data model | **Own event log** (signed intent events in a per-device hash chain, ordered by author HLC) **+ deterministic reducer → SQLite**; rituals derived from log time; **Loro** for page bodies, carried as event payloads | Loro/Yjs documents as the truth | One home for cross-object rules and conflicts; needs no server; works with a blind hub; the log *is* the context graph's timeline. Contract: [`sync-protocol.md`](../architecture/sync-protocol.md) |
| 4 | Topology | **Mac alone works fully (M1). Optional hub = the same binary; blind by default, trusted per workspace by key grant (M2).** Async delivery without a hub through a "mailbox" | Trusted hub only | Meets "never require a server" and "$0", and still offers background AI to those who want it |
| 4 | Network | **iroh 1.x** (QUIC, dial by public key, hole punching, relays); the hub runs its own relay | Tailscale (an advanced option) | Embeds in the app and the hub, no account, no VPN profile |
| 5 | Security | Workspace keys with hash-identified epochs, device keys (Ed25519/X25519), HPKE, XChaCha20-Poly1305, signed and hash-chained events, revocation by seq cutoff, Argon2id recovery kit; parsers in a sandboxed helper; signed updates | – | See the threat model |
| 6 | Identity and tenancy | **No account to use the app.** Devices pair by QR code. Accounts (passkeys) only for a hosted hub. **Workspace = sync unit = encryption unit = tenant unit**; one SQLite file per workspace on every node | Postgres + RLS on a hosted hub | Physical isolation per tenant; the same code on the Mac and the hub |
| 7 | Background work | **A durable job table in SQLite** (leases, retries, idempotency keys), run by the trusted hub or else the Mac; the sprint close and recurrences are **derived by the reducer**, not jobs | Inngest-like hosted queue | No service to run; works offline; nothing to duplicate |
| 8 | AI | **One AI gateway in the core**: bring-your-own Anthropic key, per-feature models, usage ledger, budgets, prompt caching, Batch API for rituals. **Agent = deterministic jobs + one bounded, read-only tool loop for Ask**; it writes only suggestions. **MCP server off by default**, web-derived content excluded by default | Claude Agent SDK | Small attack surface; "AI proposes, you confirm" by construction |
| 8 | Embeddings | **Local by default, model chosen by S5** from a shortlist of Qwen3-Embedding-0.6B and EmbeddingGemma-300M; weights downloaded on first use (signed, pinned hash); Voyage API as an opt-in | Voyage API by default | Offline, private, $0. Public scores don't settle Polish retrieval, so S5 does |
| 9 | Files | Content-addressed blob store (BLAKE3); chunking, per-chunk encryption and lazy sync arrive with sync in M2 | Whole-file sync | Dedupe now; resumable sync later |
| 10 | Backups | **`VACUUM INTO` snapshot + restic-format repository** (disk or bucket; hub in M2), weekly automatic restore test shown in Settings | Litestream (kept for the hub) | Encrypted, versioned, deduplicated; restore is proven, not assumed |
| 11 | Distribution | **Developer ID + notarisation, direct download, Tauri updater** (signed), Homebrew cask; model weights outside the bundle; App Store later; **no telemetry**, opt-in crash reports | Sparkle 2 (switch if the bundle passes 50 MB or ships CEF) | No review risk for agent features; sandbox-ready anyway |
| 11 | Hub deployment (M2) | One OCI image + compose file, a NixOS module and a cloud-init script; health shown in the app | – | "Everything as code" |
| 12 | Automations | **No rules engine in v1**: built-in rituals; signed webhooks and MCP later | Built-in rules engine | Small surface |
| 13 | Licence | **FSL-1.1-ALv2** (source available; each release becomes Apache-2.0 after two years), protocol libraries MIT/Apache — **the user's call** | AGPL-3.0 + CLA | Shows the code, allows free self-hosting, blocks resale of the hosted hub |
| 15 | Plan | **Spikes that gate M1 first** (6 spikes, about 8–10 weeks), then a foundation and parallel tracks; hub, mailbox and transport spikes gate M2 | – | Every risky choice above has a go/no-go test that can fail |

### Top risks

1. **Building our own sync protocol.** It is the heart of the product and the place where data loss would happen. The red-team review found identity and ordering holes in the first draft (receiver-dependent clock clamping, duplicate ritual events, seq reuse after power loss or restore, backdating by a revoked device, offline key-rotation forks); [`sync-protocol.md`](../architecture/sync-protocol.md) closes each one. Mitigation: a small protocol, a pure reducer, and a deterministic simulation that must pass before any user data is trusted to it (S1a).
2. **WebKit inside Tauri:** IME and shortcut conflicts, the page process suspended in the background, the engine version following macOS. Mitigation: S2 on real input methods, with a Tauri CEF arm; Electron is a contained swap because the UI talks to the core through one interface.
3. **Block moves in the editor.** No ProseMirror CRDT binding keeps a concurrent edit to a moved block today: loro-prosemirror 0.4.4 stores blocks in a plain `LoroList`. Mitigation: S2 sizes a fork onto `LoroMovableList`; otherwise we accept delete + insert, as every Yjs-based editor does.
4. **Young dependencies:** loro-prosemirror (v0.4, last two releases six months apart), iroh (1.0 in June 2026), sqlite-vec (one maintainer). Each sits behind an interface with a named fallback.
5. **Key loss is permanent.** Nobody can recover end-to-end encrypted data without a device or the recovery kit. Mitigation: recovery kit at setup, reminders, and a trusted hub that can hold a copy of the key if the user wants.
6. **Rust slows a one-person team.** Mitigation: S4 ports one module in both Rust and TypeScript and compares; domain rules could move to TypeScript later without touching storage, sync or crypto.
7. **Prompt injection** has no complete fix, and MCP hands data to *other* agents that do have send tools. Mitigation: our agent can't send anything; MCP is off by default and excludes web-derived content unless the user includes it.
8. **The query planner, not the engine, is SQLite's weak spot**: three naive queries were 25–2,000 ms until fixed. Every query lives in the core with a plan test and a latency budget.

## Context

- Sprint, the web app, proved the product: sprints as first-class objects, an ontology and context graph with provenance and an event log, an editable ontology (ADR-0005 in Sprint), and AI that drafts rituals. Its stack (Next.js on Vercel, Supabase Postgres with RLS, Drizzle, Inngest, BlockNote, Voyage and Claude) is an input, not a constraint.
- Agent 6 started this ADR before handing it over. Its handoff on sprint#57 (2026-10-02) is reused where re-checked; its draft and scripts stay on the branch `claude/adr-0001-agent6-draft`, not in this repo, so figures from it are marked as Agent 6's.
- What Sprint's domain needs is in [`domain-requirements.md`](../architecture/domain-requirements.md): the integrity rules the database enforces today, every background job, every AI call, volumes (10k–100k objects for a heavy user) and the lessons not to relearn.
- Decided by the user: an empty database (no migration from Sprint), no export feature for now, never require a home server, about $0 running cost, a data model that can serve many tenants later.
- The bar: optimized, secure, performant and stable, decided from first principles and on evidence.
- This revision answers the red-team review (Agent 4, 38 comments). Its fact checks confirmed many claims (Tiptap's MIT extensions, BlockNote `xl-*` GPL, FTS5 not folding `ł`, PGlite `fsync=off`, iroh 1.0 and its relay policy, App Store 2.5.2 enforcement, SQLCipher on 3.53, R2's free tier, the Batch discount) and corrected others, all folded in below.

## 1. App architecture and shell

Research: [`research-shell-ui-editor.md`](../architecture/research-shell-ui-editor.md).

| Shell | Perf | Security | Stability | Dev speed | Fit | Cost | Lock-in | Mobile | Web reuse |
|---|---|---|---|---|---|---|---|---|---|
| **Tauri 2** (2.12, Sept 2026) | 4 | 5 | 4 | 4 | 5 | 5 | 4 | 3 | 4 |
| Electron 44 | 3 | 3 | 5 | 5 | 4 | 5 | 4 | 1 | 5 |
| Native SwiftUI + AppKit | 5 | 5 | 5 | 2 | 2 | 5 | 2 | 4 | 1 |
| Flutter / Compose MP / Electrobun / Dioxus | 3–4 | 3–4 | 2–4 | 3–4 | 2–4 | 5 | 2–3 | 1–5 | 2–4 |

**Choice: Tauri 2.** The UI is a web app in the system WebKit view; the backend is our Rust core, linked into the app. Native SwiftUI would be the best Mac app, but it gives nothing for the web, and no CRDT-aware block editor exists for it; we would build one on TextKit 2 (months) and a second UI anyway. Electron is the most predictable runtime (one Chromium everywhere) but forces a major upgrade every 8 weeks (three supported), and its Node runtime has to be fenced off with fuses. **Memory is not a reason to pick Tauri:** the popular "4× less memory" figures come from thin blog methods, and tauri#5889 measured Tauri at 421–429 MB against Electron's 332–337 MB on macOS once WebKit's processes were counted. S2 measures our own app as the sum of the app, WebContent, Networking and GPU processes. Tauri 3 (alpha since 2026-09-13) has an alpha CEF runtime, so S2 also runs the same UI on it; if it is solid, it removes WebKit's IME and drift risks without leaving Tauri.

**Where the logic lives: a Rust core.** It owns storage, the ontology, every domain command (`create_task`, `close_sprint`…), the reducer, sync, crypto, search, embeddings, jobs and the AI gateway. TypeScript owns views, view state, optimistic UI and the editor. The rule: **anything the hub or another device needs goes in Rust.** The core is used as a library by the Tauri app, as a binary by the hub, as wasm by the web client, and through UniFFI by a phone app. TypeScript types for the command API are generated from Rust. Sprint's `packages/core` (about 10.6k lines of pure TypeScript, 520 tests) is the specification to port; its tests are ported first.

**UI: React 19 with the React Compiler** (stable since October 2025), Vite, strict TypeScript, query caching over core queries, cmdk, a drag-and-drop library, and Radix-style primitives styled with Tailwind. The UI talks to the core through one `CoreClient` interface (Tauri IPC on the Mac, a worker with the wasm core on the web), so the shell can change without touching the UI.

**Editor: ProseMirror, via Tiptap 3 (MIT), with our own block UI.** ProseMirror has bindings for Loro (loro-prosemirror 0.4.4; releases on 2026-02-19 and 2026-08-22) and Yjs (y-prosemirror, stable). The block UI (slash menu, side menu, drag handles, nesting) is built on Tiptap's MIT UniqueID and DragHandle, an estimated 3–6 weeks; S2 first tries BlockNote's MPL-2.0 UI on top of loro-prosemirror. BlockNote's GPL `xl-*` packages are never used. The page document lives twice: in the webview (bound to ProseMirror) and in the core (durable copy, sync, search); both run the same Loro code (Rust, and its wasm build).

**Block moves, stated plainly.** loro-prosemirror stores a page's blocks in a plain `LoroList` and turns a block drag into delete + insert. So a block moved on one device while edited on another loses or duplicates the edit, exactly as with Yjs. Loro's `LoroMovableList` can do better, but only if the binding uses it. S2 decides between:
- (a) forking loro-prosemirror onto `LoroMovableList` (ProseMirror steps have to be mapped to moves; S2 sizes it), with the go criterion "a block dragged on A while edited on B ends up moved and edited, through the real binding, over 1,000 fuzzed runs";
- (b) accepting delete + insert, as every Yjs-based editor does today.

Either way, Loro stays the default for page bodies because the core is Rust: one implementation on both sides (Yjs would mean Yjs in the webview and the separate yrs port in the core), shallow snapshots, history, and Swift bindings. The choice between Loro and Yjs is confirmed in S2, not on move semantics.

**What would change it:**
- WebKit IME, shortcut or background-suspend problems that S2 can't fix → Tauri's CEF runtime if stable, else Electron (same UI, the core as a Node addon).
- loro-prosemirror fails S2 → the Yjs path: BlockNote + Yjs in the webview and yrs in the core.
- Rust slows the team badly (S4) → domain rules move to TypeScript on Bun; storage, sync and crypto stay in Rust.
- The user wants a fully native Mac feel and accepts a separate web UI → SwiftUI shell with the core over UniFFI and the editor as a web view.

## 2. Local storage

Research: [`research-storage.md`](../architecture/research-storage.md). Benchmark: [`bench-storage.md`](../architecture/bench-storage.md) (100k objects, 200k edges, 300k events, 100k 512-dimension vectors; a shared 4-vCPU x86 VM, so a pessimistic bound for a Mac, warm caches, one run per engine).

| Engine | Perf | Security | Stability | Dev speed | Fit | Cost | Lock-in |
|---|---|---|---|---|---|---|---|
| **SQLite 3.53** (+ FTS5, sqlite-vec) | 5 | 4 | 5 | 4 | 5 | 5 | 5 |
| Turso Database 0.8 (Rust rewrite, SQLite format) | 4 | 4 | 2 | 4 | 5 | 5 | 4 |
| PGlite 0.5.8 (Postgres 18 in wasm) | 2 | 3 | 2 | 4 | 4 | 5 | 3 |
| Embedded native Postgres | 4 | 4 | 4 | 3 | 2 | 5 | 4 |
| SurrealDB, CozoDB, Kùzu, Realm, a custom KV engine | – | – | 1–3 | – | – | – | – |
| Minigraf 2.0.3 (embedded bi-temporal graph, Datalog) | – | 3 | 1 | 3 | 2 | 5 | 3 |

Measured, p50 / p95 in ms:

| | SQLite | PGlite | Postgres 16 |
|---|---|---|---|
| Sprint board | 0.50 / 1.37 | 2.51 / 16.1 | 0.75 / 3.79 |
| All neighbours of an object | 0.05 / 0.15 | 1.61 / 6.92 | 0.51 / 1.62 |
| Home "today" | 1.45 / 2.39 | 8.73 / 16.7 | 2.53 / 7.67 |
| Full text, one common word | 22.3 / 39.4 | 142 / 309 | 52 / 98 |
| New task + 2 edges, triggers on, **every commit fsynced** | 1.25 / 7.9 (`FULL`) | – (PGlite runs `fsync=off`) | 1.63 / 10.7 |
| Cold open + first query (warm OS cache) | 35–46 (+ 30–50 to load the native module) | 850–970 | 100–107 (server already running) |

**Choice: SQLite**, bundled at ≥ 3.53 (for `ALTER TABLE` adding and dropping `NOT NULL` and `CHECK`; it also includes the fix for the WAL-reset corruption bug, which landed in 3.51.3), through rusqlite, with WAL, `synchronous=FULL` and `fullfsync` (section 3: an event leaves the device only once durable), `STRICT` tables, `foreign_keys=ON` checked at open, one writer connection and a small reader pool.

**Why, honestly:** the decisive reasons are no server, the same engine on Mac, iOS, a Linux hub and wasm, and a file format stable for 20 years. On speed, SQLite was fastest or equal everywhere, but the margin over native Postgres is mostly 1.2–1.7× for indexed queries, about 1.3× for writes at equal durability, and larger only for full text (2–3×). Our naive SQLite queries were *slower* than Postgres until fixed. PGlite was 5–20× slower.

- **PGlite is rejected:** slowest on everything, about 0.9 s to open, and it runs with **`fsync=off`** (we checked `current_setting('fsync')` on 0.5.8; PGlite issue #1107), so an acknowledged write can be lost on a crash. Its 0.4 → 0.5 upgrade couldn't read old data files.
- **An embedded Postgres is rejected:** a second process, 6 s recovery after an unclean stop, impossible on iOS (apps can't spawn processes) and in the browser.
- **Minigraf is rejected as the store, for now** (2.0.3, checked 2026-10-06; suggested by the user). It's an embedded Rust graph database queried in Datalog, with bi-temporal facts, MIT/Apache, and bindings for Swift, Android and wasm.
  - **No schema or constraints.** "Optional schema validation" is only on its roadmap, so every ontology rule (field types, relation ends, cardinality, no cycles) would be ours to write anyway.
  - **A live data-integrity bug.** Two values of one attribute written in a single transaction can read back as one (its issue #371). The fix changes the file format and ships in v3.0.0.
  - **One maintainer.** About 1,100 of roughly 1,110 commits are by one person (since 2023).
  - **No full text, vectors, encryption or sync,** so SQLite would still be needed beside it.
  - **Two ideas we take from it:**
    - *Bi-temporal facts*: valid time (when it was true) as well as transaction time (when we learned it). The event log already gives transaction time; the schema adds valid time to relations and corrected values, so the graph can answer "what did we believe when the AI drafted that plan?" (section 3).
    - *Datalog as a read-only query language* over the projection, for "Ask your OS" and future rules: small, recursive, and easy for a model to write correctly. It's a later option, not an M1 dependency.
  - **We'd revisit it** once v3 ships the integrity fix and schema validation, and only if a spike shows Datalog over our graph earns its place.
- **Full text:**
  - FTS5 with `unicode61 remove_diacritics 2`, titles weighted over bodies, plus a trigram index on titles for substring search (0.4 ms p95).
  - **`ł` isn't folded:** U+0142 has no Unicode decomposition. We checked on 3.53.4: `lodz` misses "Łódź", while `gesla` finds "gęślą". So the core writes a pre-folded search column (NFKD, marks stripped, `ł → l`) and folds queries the same way.
  - **Polish stemming:** Snowball's Polish stemmer first shipped in a tagged release with Snowball 3.1.0 (2026-05-22). FTS5 has no Snowball tokenizer, so it would be a pre-stemmed column filled by the core (if the Rust stemmer crate carries the new algorithm) or a custom tokenizer. S5 measures whether it helps.
- **Vectors in the same file.**
  - Exact search with sqlite-vec 0.1.9 took 122 ms over 100k × 512 float32 on the VM, and 64 ms at 256 dimensions. A heavy user has about 15k chunks (Sprint's estimate), about 18 ms.
  - A numpy matrix product took 25 ms on the same VM, so S3 tries a SIMD scan in the core.
  - Our native Postgres used pgvector 0.6.0 (443 ms), and Agent 6's used 0.8.1 (31–37 ms on 125k). The gap is unexplained (version, parallel workers or setup), so neither number is used as evidence.
  - Storage per chunk at 512 dimensions: 2 KB as float32, 0.5 KB as int8. sqlite-vec has no float16.
  - An approximate index (sqlite-vec DiskANN or SQLite's `vec1` once stable, or HNSW in the core) comes past about 50k chunks.
- **Query discipline:** three naive queries were slow (a recursive CTE joined in the wrong order, an `OR` join over hub nodes, a partial index that couldn't serve a query) and were 100–600× faster once fixed; the probe scripts are in the benchmark note. Every query is written in the core with an `EXPLAIN QUERY PLAN` test and a latency budget per class:
  - graph lookups under 2 ms;
  - Home, board and collections under 10 ms;
  - full text under 50 ms;
  - vectors under 30 ms.

**Integrity**, in four layers (Sprint enforces its rules with Postgres triggers, checks, RLS and advisory locks; none of that exists on a phone, and sync merges need rules a row-level trigger can't express):
1. **One write path.** Every write (UI, AI suggestion accepted, import, sync) is a typed command validated against the ontology, turned into a signed event, and applied by the reducer in one `BEGIN IMMEDIATE` transaction.
2. **The reducer enforces cross-object rules deterministically** and records every intent it can't apply as written in a `conflicts` table (section 3).
3. **SQLite backstop:** FKs to the ontology tables, `CHECK`s (enums, ranges, key formats, `json_valid`), and a few triggers for invariants whose breach corrupts the graph: relation from/to types, cardinality, append-only events, no deleting ontology rows. On the benchmark's trigger design they cost under a millisecond per write; the full command path (event, signature, reducer, FTS) is measured in S3.
4. **Checks:** `PRAGMA integrity_check`, `foreign_key_check` and an ontology consistency check at start-up and nightly.

**Encryption at rest:**
- **SQLCipher on by default**, through rusqlite's `bundled-sqlcipher` (SQLCipher 4.18 is on SQLite 3.53.4; Community Edition is BSD-3).
  - S3 measures it where it costs: cold reads, writes and checkpoints, with a 64 MB cache against a ~600 MB file. Hot queries hit already-decrypted pages, so measuring them proves nothing.
  - S3 also checks that it links with sqlite-vec and FTS5, and with CommonCrypto rather than OpenSSL, for a smaller supply chain.
- **The fallback is SQLite3MultipleCiphers** (MIT): it has no rusqlite feature, so it means custom linking.
- **The key** is random, 256-bit, in the **data-protection Keychain** with a keychain access group (TN3137), class **`AfterFirstUnlockThisDeviceOnly`**. Background jobs run from a login item while the screen is locked, and the `WhenUnlocked` classes would fail there.
  - It is wrapped by a Secure Enclave key on Apple silicon and T2 Macs.
  - Older Intel Macs without a Secure Enclave keep the Keychain item alone, and the app says so.
- **Why at all:** FileVault alone protects a powered-off Mac, but not a copy of the file, a backup, or another process run by the same user.
- **If the cost is too high,** we fall back to FileVault plus encrypted backups, and say so.

**Schema evolution:**
- `PRAGMA user_version` and migrations embedded in the binary, run at open after a `VACUUM INTO` backup.
- The local schema is private to each device and can be rebuilt from the log; what must stay compatible is the **event format** (section 3).
- The editable ontology is data, so most product changes need no migration.

**What would change it:**
- Minigraf ships v3 with schema validation and the #371 fix, and a spike shows Datalog over the graph is worth a second engine.
- Turso Database reaches 1.0 with stable concurrent writes and encryption. It reads the same files, so switching is cheap.
- `vec1` or sqlite-vec ANN goes stable when users pass ~50k chunks.
- A hosted product needs shared, many-writer workspaces. Then it's Postgres on the hosted hub and SQLite on devices.

## 3. Sync and the data model

Research: [`research-sync.md`](../architecture/research-sync.md). Contract: [`sync-protocol.md`](../architecture/sync-protocol.md). Benchmark: [`bench-crdt.md`](../architecture/bench-crdt.md).

| Design | No server needed | Offline writes | Cross-object rules | Rich text | Blind hub | Timeline | Build effort | Lock-in |
|---|---|---|---|---|---|---|---|---|
| A. CRDT documents are the truth (Loro/Yjs per object), SQLite as an index | 5 | 5 | 2 | 5 | 4 | 3 | 3 | 4 |
| **B. Event log + deterministic reducer → SQLite; Loro for page bodies** | 5 | 5 | 5 | 5 | 5 | 5 | 2 | 5 |
| C. Server-authoritative (PowerSync, Zero, Electric) | 1 | 1–3 | 5 | 2 | 1 | 3 | 4 | 2 |

| Ready-made option | Licence | Status (Oct 2026) | Why not as our core |
|---|---|---|---|
| Evolu | MIT | Active (`@evolu/common` 8.17.0, 2026-10-02); SQLite, end-to-end encrypted, self-hostable blind relay | TypeScript only, per-cell last-writer-wins, no rich text. **The closest prior art**: read its protocol and key handling before S1a and S6 |
| LiveStore | Apache-2.0 | 0.4.0 (June 2026); event-sourced SQLite with rebase | TypeScript only; needs a sync provider for ordering |
| Jazz | MIT | 2.0 alpha, rewritten | Unstable |
| Turso sync (`@tursodatabase/sync` 0.8.1) | MIT | Beta | Whole-database sync, tied to Turso Cloud; can't run on a blind hub |
| automerge-repo | MIT | 2.6.0-alpha | Only a document transport; no cross-document rules |
| cr-sqlite | MIT | Last release v0.16.3 (2025-01); build-only commits since | Stalled |
| PowerSync | Service FSL-1.1, SDKs Apache-2.0 | Stable | Needs a trusted Postgres and service; no blind mode |
| Zero | Apache-2.0 | 1.x | No offline writes |
| Ditto · Instant · Replicache | Closed · Apache · open | Metered · cloud closes 2027-08-31 · archived 2026-06-10 | Lock-in or winding down |

**Choice: B.** The truth is an append-only log of **intent events** ("add task T to sprint S", "move page P under Q", "set field `budget` on X to 40", "page body update: <Loro bytes>"). A pure, versioned reducer folds the events, in one total order, into the SQLite tables the app queries. The full rules are in [`sync-protocol.md`](../architecture/sync-protocol.md); the essentials:

- **Identity.** Each event has a UUIDv7 id, `workspace_id`, `device_id`, a gap-free per-device `seq`, the hash of the device's previous event (a per-device hash chain), the author's version vector at writing time (`deps`), an HLC, a type and `schema_version`, the actor, and a payload, all signed with the device key. Events are applied only after their `deps` (causal delivery).
- **Order is a pure function of signed fields:** `(hlc, device_id, seq)`. Receivers never clamp or rewrite it, so devices that receive the same events at different times sort them identically. A device's HLC must strictly increase; an event that goes backwards is a conflict everywhere. Wrong clocks are detected (devices compare wall clocks on contact and through signed heads) and shown to the user rather than silently repaired; a device won't push its own HLC more than an hour past its wall clock because of a remote event, so one bad clock doesn't spread.
- **Durable before sent.** The event and its effects commit in one transaction with `synchronous=FULL` and `fullfsync`; nothing leaves the device before its commit returns. A restored or migrated Mac (detected because its `ThisDeviceOnly` device key is missing, or peers know a higher seq) fetches its old events from peers and mints a **new device id**, so a `(device, seq)` pair can never name two events. If one ever does, it is a security alarm: the device is frozen at the last agreed seq.
- **Rituals are derived, not events.** The reducer tracks **log time**, the highest HLC applied. When log time passes Monday 00:00 in the workspace zone in effect, it applies the week's close (completed, carried over, missed); a device whose clock passes a boundary first writes a tiny `tick` event to move log time on. Recurring occurrences appear the same way, with ids from (series, occurrence time). There is nothing to write twice. AI and human artefacts (a retro draft, a plan) are ordinary events with a `logical_key`; if two devices write one, both are kept and the first in order is shown.
- **Late events.** Field writes are last-writer-wins by order and edge additions are idempotent, so most late events apply without replay. One that lands before an applied close or under an ordering rule makes the reducer rewind to a checkpoint (kept at least at every week boundary) and replay.
- **Every intent takes effect or becomes a visible conflict.** Cardinality (the later edge wins; the earlier ends as `superseded`), cycles (skipped, by Kleppmann's move algorithm), sibling order (fractional index, ties by event id), field type changes racing late values, writes to archived things (applied; archive is soft), a many → one cardinality change racing new links, recurrence edits: each has a rule in the contract, and anything the reducer can't apply as written lands in a `conflicts` table with its payload and a restore button.
- **Undo is `revert(event_id)`**, resolved at its place in the order: a field goes back only if it still holds the reverted value, else it is a conflict.
- **App versions.** Event shapes only grow; old versions are upcast. A `min_reducer_version` event makes older devices read-only (except plain field and body edits), drops their job leases and asks for an update; they rebuild after upgrading.
- **Page bodies are Loro documents**; their update bytes travel as event payloads, coalesced every few seconds of typing, and the reducer derives plain text, mentions, links and block ids into SQLite. Old history is trimmed with shallow snapshots.
- **The ontology is data in the log**, so custom fields, types and relation types sync and need no app update.
- **The log is the context graph's timeline.** One structure serves sync, history, undo, audit, "what changed this week" and AI context.

**Evidence.** Desk research ([`research-sync.md`](../architecture/research-sync.md)) and Agent 6's prototype, which shows that a deterministic reducer converges *given one order* (60 of 60 and 10 of 10 seeded runs with a sequencer, 0 of 5 without rebase; scripts on Agent 6's branch, not here). It does not test this design's hard part, ordering without a sequencer; that is S1a.

**CRDT for page bodies: Loro 1.16** (MIT, Rust, Swift bindings, shallow snapshots, time travel). Measured in Node (JavaScript and wasm) on a real 260k-keystroke trace of one flat LaTeX string, so no blocks, marks or concurrency ([`bench-crdt.md`](../architecture/bench-crdt.md)):

| | Loro 1.16 | Yjs 13.6 | Automerge 3.5 |
|---|---|---|---|
| Apply the trace | **1.08 s** | 2.0 s | 37.8 s |
| Load the page / memory after load | 14 ms lazy / +1.6 MB | 51 ms full decode / +3.3 MB | 3.6 s / +207 MB |
| Size with history; shallow snapshot | 231 KB; **65 KB** | 160–311 KB (default `gc` drops deleted text) | **129 KB**; none |

Loro loads fast, keeps history and has shallow snapshots; Automerge is too slow in JavaScript (possibly its JS layer; native was not measured) for an editor that runs there. Yjs is the fallback. Per-keystroke wire size is irrelevant here, because body edits are coalesced and wrapped in a signed, encrypted envelope (about 150 bytes of overhead). Two lessons for the build: free Loro handles created in loops, and after a long offline period send a snapshot rather than thousands of small updates. S1b reruns the comparison natively in Rust with a block-structured trace (splits, marks, moves) through both ProseMirror bindings.

**What would change it:** the user accepts that a server must always be up (then PowerSync with Postgres is less to build); a mature native framework that does B appears; S1a shows replay too slow on a phone (then A for bulk data, B only for edges); Loro shows merge bugs under fuzzing that aren't fixed quickly (page bodies to Yjs; the log design stays).

## 4. Topology: the Mac plus an optional hub

Research: [`research-topology-security.md`](../architecture/research-topology-security.md).

| Option | Perf | Security | Maturity | Dev speed | Fit | Cost | Lock-in |
|---|---|---|---|---|---|---|---|
| No hub (devices sync directly) | 4 | 5 | 4 | 5 | 2 | 5 | 5 |
| Blind hub only (relay) | 4 | 5 | 3 | 3 | 3 | 4 | 5 |
| Trusted hub only | 5 | 2 | 4 | 4 | 5 | 4 | 5 |
| **Hybrid: no hub works; hub blind by default, trusted per workspace** | 4 | 4 | 3 | 3 | 5 | 5 | 5 |

**Choice: the hybrid.** The Mac alone is M1; everything below the first bullet is M2 or M3, with its own go/no-go.

- **The Mac alone is complete:** editing, search, rituals when the Mac is awake, AI with the user's key, backups to a disk or bucket.
- **The hub is the same Rust core** in a headless binary, on a home server, a Mac mini, a cheap VPS (Hetzner's smallest is about €5.50–6 a month after the June 2026 price rise; re-check before quoting) or, later, our hosted service. Setting it up is pairing, like adding a phone.
- **Blind by default:** it stores and forwards encrypted events and blobs and acts as an always-online peer for the phone. It learns sizes, timing and workspace ids. It can withhold events, which signed per-device heads and hash chains detect (section 5), but it can't forge or reorder them.
- **Trusted by explicit grant, per workspace:** the user wraps that workspace's key to the hub's device key. The hub then keeps a plaintext replica and runs the jobs: rituals on schedule, link fetching, embeddings, the AI agent and MCP over HTTP. Revoking trust rotates the key. Obsidian lets each vault pick end-to-end or provider-managed encryption, and Tana and Capacities read data to run server-side AI, but **none lets the user grant a self-hosted server read access per workspace so it can run AI and jobs while the device sleeps.**
- **With no hub and the Mac asleep,** the phone keeps working on its own replica and changes meet when both are online. A **mailbox** can carry encrypted event batches in between: the user's S3-compatible bucket (Cloudflare R2's free tier is 10 GB) or, for Apple-only users, the iCloud private database through CloudKit. Spike S7 (M2) decides.
- **The web client (M3)** is the same React UI with the core in wasm, keeping an encrypted replica in the browser (OPFS) and unlocking the key with a passkey (PRF) or a pairing code. **Whoever serves the web app's code can steal the key**, so the code is never served by the hub: it comes from a separate static origin we publish, with signed releases, Subresource Integrity and per-release hashes the Mac app can check; the hub only carries ciphertext. A self-hoster who builds and serves their own web client trusts themselves. The web still needs an online peer (a hub), because browsers can't hole-punch.

**Network: iroh 1.x** (1.0 shipped 2026-06-15): QUIC connections addressed by public key, hole punching, encrypted relays when a direct path fails, Swift and Kotlin bindings. n0 describes its public relays as "for development and testing", so in production the relay is the user's hub or, with no hub, a small relay we host or n0's paid tier; S6 measures how often connections go direct. Tailscale is a documented option for power users, never a requirement. Embedding a tailnet (tsnet) is Go-only.

**What would change it:** iroh failing on mobile carrier networks in S6 (self-hosted relay first, then Tailscale); most users wanting the trusted hub (make trusted the default for hosted plans, keep blind for self-hosters); an Apple-only audience (CloudKit becomes the default mailbox).

## 5. Security model

Research: [`research-topology-security.md`](../architecture/research-topology-security.md) → threat model; [`sync-protocol.md`](../architecture/sync-protocol.md) sections 7–8.

| Threat | Main mitigations |
|---|---|
| Lost or stolen laptop or phone | FileVault / iOS Data Protection; SQLCipher with the key in the data-protection Keychain, wrapped by the Secure Enclave; remote revoke from another device |
| A revoked (stolen) device keeps writing, backdating its clock | Revocation names a **seq cutoff** (the highest seq any surviving device saw), not a time; every peer and the blind hub reject the device's later events whatever their HLC; the key rotates |
| Breached or malicious blind hub | Ciphertext only, names and paths included; events signed and hash-chained per device so it can't forge or reorder; **withholding is detected** through signed device heads compared on direct contact or via the mailbox |
| Malicious web-client code from whoever serves it | The web app is served from a separate static origin with signed releases and SRI, never from the hub; self-hosters serve their own build |
| Trusted hub breached | It can read the workspaces granted to it; that is documented at the grant, per workspace; hardened like the Mac (SSRF guards, parser sandbox, encrypted disk, signed updates) |
| Malicious link or file content (parser and image-decoder bugs) | Fetching and parsing run in a **separate sandboxed helper** (an XPC service on the Mac, a locked-down process on the hub) with no keys and no database access; memory-safe parsers; images re-encoded; size and time limits. Saved pages are never rendered as live HTML in an app window |
| SSRF from the link fetcher | Port Sprint's guarded fetcher: block loopback, private, link-local, CGNAT and cloud-metadata addresses at connect time and on every redirect; http(s) only; no cookies; 5 s and 1 MB caps |
| Prompt injection into our agent | It never has untrusted content, private data and a way to send data out in one session: no web-fetch or send tool, its only write is `create_suggestion`, answers render without loading remote images or links, anything derived from untrusted text shows its provenance and needs the user's accept |
| Prompt injection *through MCP* into other agents | External MCP clients (Claude Desktop, Claude Code, n8n) have their own send and shell tools. So MCP is **off by default**; each client's scope **excludes link and web-derived content by default** (or returns it in clearly labelled untrusted blocks); writes are `create_suggestion` only, rate-limited; tool descriptions are static and reviewed (against tool poisoning) |
| Telegram | Bot chats are not end-to-end encrypted: Telegram can read captured text and answers. It is opt-in per workspace, only with a trusted hub, and Ask answers with workspace content go back over Telegram only if the user turns that on |
| Malicious dependency | pnpm 11 (defaults `minimumReleaseAge: 1440`, `strictDepBuilds`, `blockExoticSubdeps`) raised to a 7-day release age, plus the opt-in `trustPolicy: no-downgrade` (pnpm 10.21+); a small JavaScript tree; `cargo-deny`, and `cargo-vet` with crates that have build scripts or proc macros as a separate audit criterion **gated in CI**; committed lockfiles; pinned CI actions. Recent incidents: npm worms (Shai-Hulud Sep and Nov 2025, "Mini" May 2026, keyv Aug 2026), the axios maintainer takeover (2026-03-31) and the crates.io `arrayref` hijack through a `build.rs` (2026-08-20) |
| Compromised update channel | Signed updates (Ed25519) with the key offline, Apple notarisation, the public key pinned in the app; model weights signed and pinned by hash |

**Keys.**
- **Workspace key.** Each workspace has a random 256-bit key. Synced data is sealed with XChaCha20-Poly1305.
- **Epochs** are identified by a hash of their parent and the new key's commitment.
  - Rotation happens on revoking a device or untrusting a hub. It wraps the new key to every remaining member with HPKE (RFC 9180).
  - If two devices rotate while offline, the fork is merged by one more rotation that excludes every device revoked in either branch. This converges because the revoked set only grows (`sync-protocol.md` section 8).
- **Device keys.** Each device has Ed25519 (for signing, and as its iroh identity) and X25519 keys. They are `ThisDeviceOnly` and never synced. Only the workspace key may sync through iCloud Keychain, and only as an opt-in.
- **Recovery kit:** 24 words or a printable page, which unwraps the key through Argon2id. Losing every device and the kit loses the data, and the app says so plainly.
- **Libraries:**
  - RustCrypto (`chacha20poly1305`, `ed25519-dalek`, `x25519-dalek`, `argon2`) plus the `hpke` crate.
  - Audit coverage is partial (an older external audit of `chacha20poly1305`, none of `hpke`), and our key module gets an external review before the hosted hub launches.
  - aws-lc-rs is not an alternative here: it has neither XChaCha20-Poly1305 nor public HPKE.
- **MLS** is not needed until real shared workspaces exist.

**Sandboxing.** Developer ID, Hardened Runtime and notarisation from day one. App Sandbox on if S2 shows file watching, the parser helper and the network stack all work inside it (that also keeps the Mac App Store open). Tauri: deny-by-default capabilities per window, a strict CSP, the isolation pattern, and no remote origin ever loaded in a window that can call the core.

## 6. Auth and tenancy

- **No account to use the app.** First run creates a workspace (with its key) and a device key pair. The person's identity is the set of their paired devices, as in Signal.
- **Pairing** a phone or a self-hosted hub is a QR code or one-time code; no server account is involved.
- **A hosted hub (later, paid)** has accounts for billing and routing only, never for decrypting data: passkeys first, an email link to recover the *account* (not the data), Sign in with Apple or Google optional.
- **Workspace = sync unit = encryption unit = tenant unit.** A person can have several (Work, Personal), each with its own key and its own hub trust setting. Every event carries `workspace_id`; objects keep UUIDv7 ids, globally unique, as in Sprint.
- **Many tenants later:** the hosted hub keeps **one SQLite file per workspace**, running the same core as the Mac, plus a small control database for accounts, devices and billing. Isolation is physical, and backup, restore and delete per tenant are file operations (Cloudflare Durable Objects with SQLite, Turso and 37signals' `activerecord-tenanted` use the same pattern). Postgres with RLS returns only if shared workspaces with many concurrent writers appear.

## 7. Background work

Replaces Inngest and Vercel cron. Research: [`research-ai-jobs.md`](../architecture/research-ai-jobs.md) → background work.

| Option | Perf | Security | Maturity | Dev speed | Fit | Cost | Lock-in |
|---|---|---|---|---|---|---|---|
| **Job table in the workspace's SQLite** (leases, retries, backoff, idempotency keys) | 5 | 5 | 4 | 4 | 5 | 5 | 5 |
| Hosted queue (Inngest-like) | 4 | 3 | 5 | 4 | 2 | 4 | 2 |
| Redis-backed queue (BullMQ, apalis + Redis) | 4 | 4 | 5 | 3 | 2 | 4 | 3 |

- **What is not a job:** the sprint close and recurring occurrences are derived by the reducer from log time (section 3), so they can't run twice or be missed. A Mac closed all weekend closes the week the moment it writes its first event on Monday.
- **Jobs** (from Sprint): embeddings (debounced per object), link metadata fetching with retries, the weekly summary, insights and plan drafts, the nightly integrity check, backups and the weekly restore test.
- **Where jobs run:** on the trusted hub when there is one, else on the Mac while it's awake. The phone never runs rituals. One runner holds a lease per workspace; a device below `min_reducer_version` gives its lease up; AI artefacts carry a `logical_key`, so a duplicate is kept as an alternative, not applied twice.
- **On the Mac,** an optional login item (`SMAppService`) keeps the core running with the window closed, which is why the database key uses `AfterFirstUnlockThisDeviceOnly` (S3 tests a locked-screen night run). On iOS nothing is relied on in the background.
- **Crash and error reports:** off by default. Apple's crash reports and MetricKit on the Mac and phone; for the hub, an opt-in Sentry-protocol endpoint the user can point at a self-hosted Bugsink or GlitchTip. No product analytics.

## 8. AI

Research: [`research-ai-jobs.md`](../architecture/research-ai-jobs.md); embeddings: [`bench-embeddings.md`](../architecture/bench-embeddings.md).

- **Keys:** bring your own Anthropic key (and an embeddings key if the API is chosen), in the Keychain (not synced) or the trusted hub's secret store. A hosted plan later would use our own metered key, never a proxy of users' keys.
- **One gateway** in the core for every model call: per-feature model and effort settings (the model is config, as in Sprint), a budget check before each call (notice at 80%, pause at 100%), a ledger row from each response's `usage` fields, prompt caching on by default, and the Batch API (half price) for rituals. The Anthropic API is called over HTTP from Rust; there is no official Rust SDK. Rough cost for one heavy user: about $0.12 a week for rituals; Ask, at a few cents a question, is what the budget is for.
- **The agent is mostly deterministic code.** Rituals (retro draft, weekly plan, triage) are jobs: code gathers the facts and the model drafts text or picks moves from a closed list through structured outputs, which code validates, as Sprint's Monday plan does. **Ask your OS** is one bounded tool loop on the plain Messages API: read-only tools (`search`, `get_object`, `neighbours`, `sprints`, `journal`, `insights`), a turn cap, a token budget, and an answer with object citations. The only write tool anywhere is `create_suggestion`; accepting it runs the same command a manual edit would. Every run writes an audit row. The Claude Agent SDK is not used: it brings a coding harness (files, shell, web) that is the wrong shape and attack surface for a personal data server.
- **Local LLMs don't carry the rituals.** Apple's on-device model has a 4K context on macOS 26 (8K on 27), and Apple Intelligence doesn't support Polish. Tag, kind and duplicate suggestions stay embedding-neighbour votes with no LLM, as in Sprint.
- **Embeddings: local by default, behind the gateway, model chosen by S5.**
  - The model is pinned per workspace and stored on every row; the Voyage API is an opt-in.
  - Vectors are derived data, never events. The Mac or trusted hub computes them; other devices either receive the vector table as an encrypted derived snapshot or embed only their queries.
  - **The shortlist:**
    - **Qwen3-Embedding-0.6B:** best MMTEB mean (64.3; Qwen's figure). Apache-2.0 on the model card; confirm before shipping weights, since the code repo has no LICENSE file.
    - **EmbeddingGemma-300M:** 2–3 points lower, 2–3× faster, and **first on the only Polish retrieval task** (Belebele: Gemma 92.95, bge-m3 92.90, Qwen3 89.43). Gemma licence, not OSI.
  - **How firm the numbers are:** the "Polish slice" and "English retrieval" means in the benchmark were computed in-house from leaderboard subsets. Voyage's lead of about 5.6 points is from vendor-submitted RTEB results, and Voyage has no Polish scores. No model has been run on real data yet.
  - **S5 decides,** with a labelling protocol, 200 real queries, recall@10 with a paired-bootstrap confidence interval, and the actual Rust-side runtime (candle, llama.cpp Metal or ONNX Runtime with Core ML) on a base M1 with real weights and tokenizer.
  - **Weights** (about 0.3–0.6 GB) are downloaded on first use, not bundled. They are signed and pinned by hash.
- **The semantic index lives in the same SQLite file** (section 2), re-embedded in the background when the model changes; hybrid search (FTS + vectors, reciprocal rank fusion, as in Sprint) covers the gap.
- **MCP server (M2), off by default.**
  - Built against the MCP spec of 2026-07-28 (stateless core, stricter authorization).
  - Scoped tools: read tools and `create_suggestion`, never delete.
  - Each client's scope excludes link and web-derived content unless the user includes it, because the client agent, not ours, holds send tools.
  - Transports: stdio on the Mac for Claude Desktop and Claude Code; Streamable HTTP with OAuth on a trusted hub.
  - Clients are allowlisted per workspace.
  - The research note lists the 2025–2026 MCP incidents (tool poisoning, command injection, CVE-2025-6514 on the client side).
- **Telegram (M2), opt-in per workspace, trusted hub only:** capture and chat by long polling; only allowlisted user ids; replies only to the asker; Telegram can read what passes through it.

## 9. Files

- **M1:** a **content-addressed blob store**: files named by their BLAKE3 hash under the workspace folder, metadata in SQLite, so identical files are stored once. Thumbnails and previews via QuickLook.
- **M2, with sync:** large files are split with FastCDC into chunks, so uploads resume and an edit resends only changed chunks. Each chunk is sealed with XChaCha20-Poly1305 under a per-blob key; on a blind store, objects are named by a keyed hash. Metadata syncs at once, bytes on demand. Garbage collection uses tombstones and a grace period so a blob still referenced by an unsynced device is never deleted.
- **Later:** Spotlight indexing (Core Spotlight).
- **The live database never sits in iCloud Drive or Dropbox**: they copy the database, WAL and shared-memory files separately and corrupt it. It lives in Application Support (or the sandbox container).

## 10. Backups

Distinct from export. Research: [`research-files-ops-licence.md`](../architecture/research-files-ops-licence.md) → backups.

| Option | Perf | Security | Stability | Dev speed | Fit | Cost | Lock-in |
|---|---|---|---|---|---|---|---|
| **restic-format repository** (bundled restic 0.19 now, `rustic_core` in-process later) | 4–5 | 5 | 5 | 4 | 5 | 5 | 5 |
| Kopia | 4 | 5 | 4 | 4 | 3 | 5 | 4 |
| Litestream only | 5 | 3 | 4 | 5 | 2 | 5 | 4 |
| Time Machine only | 3 | 4 | 4 | 5 | 2 | 5 | 3 |

- **When:** every hour when something has changed, and at quit, `VACUUM INTO` a consistent snapshot of each workspace file.
- **Where:** back up the snapshot and the blob store into a restic-format repository: an external disk or the user's bucket in M1, and the hub (rest-server in append-only mode) in M2. Blobs dedupe for free.
- **Restoring creates a new device id** (section 3), so a restored Mac never reuses sequence numbers its peers have already seen.
- **Encryption and retention:** encrypted with a repository password in the Keychain, which has its own recovery path. Keep 48 hourly, 30 daily, 12 weekly and 24 monthly snapshots.
- **A weekly automatic restore test:** restore the latest snapshot to a temporary folder, open it, run `integrity_check`, compare counts and a sample of blob hashes, and show "last verified restore: 3 days ago" in Settings.
- **Other layers:** the hub also streams its own databases with Litestream. Time Machine is a bonus layer; the live database is excluded from it, and the snapshots are included.
- **Export** stays out of scope, but it comes nearly free: the event log, the SQLite tables and the blob folder are an open, documented format.

## 11. Distribution and operations

- **Apple Developer Program** ($99 a year): Developer ID signing, hardened runtime, `notarytool` and stapling in CI.
- **Direct download first**, plus a Homebrew cask. The Mac App Store is possible later because we build sandbox-ready, but its guideline 2.5.2 (no running downloaded code) was used against AI app-builders in March 2026, so agent features would face review risk.
- **Updates:** the Tauri updater (Ed25519/minisign signatures). It has no delta updates, which is fine for a small app because model weights live outside the bundle. If the bundle passes 50 MB or ships CEF, we switch to Sparkle 2 (EdDSA, deltas).
- **Telemetry:** none. The only call home is the update check, which can be turned off. Crash reports are opt-in.
- **The hub from code (M2):** one small OCI image (static Rust binary), a compose file, a NixOS module and a cloud-init script for a VPS, with Tailscale optional. The app shows the hub's health (version, last sync, last job, last backup and restore test, disk) from a `/status` endpoint.
- **Testing:**
  - unit and property tests in Rust (proptest), and Vitest for the UI;
  - the **deterministic simulation** from S1a: 10⁴ seeds on every pull request and 10⁶ nightly, each run being 3–5 devices × 3,000–10,000 steps;
  - `cargo-fuzz` on the reducer, the event decoder and the parsers;
  - **power-loss tests** for SQLite and the blob store (a VM hard reset or dm-flakey, plus a SQLite VFS that drops unsynced writes), because `kill -9` can't lose WAL commits;
  - Playwright against the web build of the UI, plus a short macOS smoke suite through an embedded WebDriver (the official tauri-driver has no macOS support);
  - restore tests in CI.

## 12. Automations

No rules engine in v1. The built-in rituals cover what Sprint does by hand today. From M2, power users get signed outbound webhooks and the MCP server, which lets any MCP-capable assistant or n8n act through the same scoped tools. n8n's Sustainable Use License forbids bundling it in a paid product, so it stays outside. **What would change it:** repeated requests for "when X, do Y" inside the app; then a small declarative rules engine that emits suggestions, built on the event log.

## 13. Licensing and business

| Model | Trust | Stops resale of the hosted hub | Contributions | Fits dependencies |
|---|---|---|---|---|
| Closed | 2 | 5 | 1 | 5 |
| MIT / Apache-2.0 | 5 | 1 | 5 | 4 |
| AGPL-3.0 + CLA | 4 | 4 | 3 | 3 |
| **FSL-1.1-ALv2** (Apache-2.0 two years after each release) | 4 | 5 | 2 | 4 |
| Open core | 3 | 3 | 3 | 4 |

- **Recommendation: FSL-1.1-ALv2** for the app and the hub, with all copyright owned by the user (a CLA for any outside contribution), and MIT/Apache-2.0 for libraries we split out (the event format, the sync protocol, a CLI).
  - It shows the code and lets anyone self-host the hub for free.
  - It stops a third party selling our hosted hub for two years.
  - It never strands users, because each release becomes Apache-2.0.

  **This is the user's call.** AGPL-3.0 + CLA is the pick if an "open source" label matters from day one.
- **Dependencies:** everything chosen is permissive:
  - SQLite (public domain);
  - Loro and iroh (MIT/Apache);
  - Tiptap core (MIT);
  - sqlite-vec (MIT/Apache);
  - SQLCipher Community (BSD-3);
  - restic (BSD-2).

  Model weights are checked one by one: the Gemma licence isn't OSI, and Qwen3's Apache-2.0 must be confirmed on its model card. BlockNote's `xl-*` packages (GPL-3.0 or commercial) are not used. CI checks licences with `cargo-deny` and a JavaScript licence checker.
- **Business:** the hosted hub (sync, backup, background AI) as a subscription, priced against Obsidian Sync ($4–8 a month); self-hosting stays free. The data model already serves many tenants. "Sprint" is a crowded trademark, so search "Sprint OS" in EUIPO and USPTO before any public launch.

## 14. Repository and what carries over

- **Copy from Sprint into `docs/requirements/`** (a separate small PR): `vision.md`, `architecture/context-graph.md`, ADR-0005 and `specs/objects.md`, all of `docs/specs/`, `modules.md`, and the Resolved table of `open-questions.md`. They are requirements, not design.
- **Port, tests first** (TypeScript → Rust, from Sprint's `packages/core`):
  - **time:** ISO weeks, time zones and DST;
  - **sprints:** the sprint engine (close, carry-over, metrics) and recurrences (the RRULE subset);
  - **capture and the ontology:** quick-capture parsing, and ontology and field validation;
  - **links:** URL canonicalisation, kind detection and the metadata parsers;
  - **pages:** mentions extraction and the page tree;
  - **search:** chunking and reciprocal rank fusion;
  - **text:** tag normalisation and the Markdown converter.

  The guarded fetcher (`fetch-page.ts`, `address.ts`) is ported with every guard. Sprint's SQL migrations are the integrity specification: each rule becomes a reducer rule (with a conflict rule) or a SQLite backstop, each with a test.
- **Not ported:** Next.js, Supabase, Drizzle, Inngest, BlockNote wiring, RLS policies.
- **Sprint's web app** keeps running until sprintOS covers it. sprintOS's own web client and phone app then replace it; there is no data migration, as decided.
- **Repo layout** (when building starts):
  - `crates/`:
    - `core`, `store`, `reducer` (per-domain modules behind a registry), `sync`, `crypto`, `index`, `ai`, `jobs`;
    - `net` and `hub` from M2;
  - `app/` (the Tauri shell);
  - `ui/` (React);
  - `docs/`.

  One workspace, one CI.

## 15. Execution plan

Building starts after Sprint's testing phase (5 Oct – 1 Nov 2026) and its UI/UX pass. Spikes come first; each has a go/no-go that can fail. A "no-go" switches to the named fallback, not to a new search. Every number is measured on a base Apple Silicon Mac (M1 or M2) on the oldest supported macOS, with p50, p95, p99 and max over at least 5 runs and caches purged for cold figures.

### Phase 0: spikes that gate M1 (about 8–10 weeks, partly in parallel)

M1 is a single device: the Mac app with the event log, sync-ready ids and every rule, but no transport, hub, mailbox, web client, MCP or Telegram.

| # | Spike | Proves | Go if | Fallback if no-go |
|---|---|---|---|---|
| S1a | **Event log, order and reducer** | The event format, signing, hash chain, HLC rules, causal delivery, version-vector sync between in-process devices, the reducer with cardinality, tree moves, conflicts, revert, derived closes and occurrences, `min_reducer_version`; a deterministic simulation (drops, duplicates, reordering, partitions, power loss between append and fsync, ±1 day skew both ways) | Over 10⁶ runs (3–5 devices × 3,000–10,000 steps): zero lost acknowledged events; after all devices run the same reducer, byte-identical tables; every invariant holds; every user event has an effect or a conflict row; rebuild from empty equals incremental; applying twice changes nothing; same order whatever the receive time; plus every scenario in `sync-protocol.md` section 10. Replay of a synthetic year (with Loro payloads) from empty < 60 s on the Mac and < 3 min on a mid-range iPhone; a week-late event < 1 s on both; log size per year reported | Design A for bulk data with the reducer only for edges |
| S2 | **Shell, editor and WebKit** | Tauri 2.12 + React 19 + Tiptap (and BlockNote's UI) on loro-prosemirror, mirrored in the core over a binary channel, App Sandbox on; the same UI on Tauri 3's CEF runtime; a sized fork of loro-prosemirror onto `LoroMovableList` | Cold start to an editable page < 1 s (target 600 ms), caches purged; memory (app + WebContent + Networking + GPU processes) < 250 MB with a 1,000-block page, reported for both runtimes; input-to-paint p95 < 16 ms from a performance trace in a 10,000-block page; Polish, Ukrainian, Japanese and Chinese input with no doubled or dropped characters and no shortcut firing during composition; recovers after 20 min in the background under memory pressure; a block dragged on A while edited on B ends up moved and edited over 1,000 fuzzed runs (if the fork is taken) | CEF runtime or Electron; the Yjs path; delete + insert for block moves |
| S3 | **Storage on a Mac, on the real command path** | Sprint's schema and integrity rules in SQLite; typed command → signed event → reducer → tables + FTS + folded column; SQLCipher; a SIMD vector scan; a locked-screen night run | At 100k objects: graph lookups < 2 ms, Home, board and collections < 10 ms, full text < 50 ms, exact top-10 over 50k × 512 vectors < 30 ms (all p95); a command commits with `FULL` + `fullfsync` < 15 ms p99, and < 25 ms p99 while a job writes 10k chunks (group commit if needed; durability never lowered); bytes written per command reported; SQLCipher costs ≤ 20% on cold reads, writes and checkpoints (64 MB cache, ~600 MB file); 1,000 simulated power losses with zero corruption and zero lost acknowledged commits; the login item opens the encrypted database and runs a job with the screen locked overnight | FileVault only, with encrypted backups; group commit |
| S4 | **Rust for one person** | Port `recurrence` (about 1–2 days) in both Rust and TypeScript with Sprint's tests, timing both; generate TypeScript types from Rust | The Rust port takes ≤ 1.5× the TypeScript port's measured time, with all tests passing | Domain rules in TypeScript on Bun; storage, sync, crypto in Rust |
| S5 | **Embeddings and search** | Qwen3-0.6B, EmbeddingGemma-300M and Voyage on 200 real queries (Polish and English) over a fixed ~5M-token corpus, labelled by a written protocol; the Rust-side runtime with real weights; FTS5 with the folded column, with and without Polish stemming | The chosen local model's recall@10 is within 5 points of Voyage with a paired-bootstrap 95% interval reported; query embedding < 10 ms p95 on a base M1, model load < 2 s, resident memory reported; indexing the corpus < 10 min | Voyage by default when a key exists, the local model as the offline fallback |
| S8 | **Hostile input and prompt injection** | The sandboxed fetch-and-parse helper; Ask with read-only tools | Fuzzed HTML, PDF and images never crash outside the helper; every SSRF case blocked; 20 poisoned saved pages cause zero unconfirmed tool calls and zero outbound requests; a poisoned page read through MCP by a client that has web fetch returns no web-derived content unless the user's scope includes it, and then only inside labelled untrusted blocks | No AI over web content until fixed; MCP stays off |

After these, this ADR is updated with the results and fallbacks taken, and goes to the user for **Accepted** for M1.

### Spikes that gate M2 (before the phone and the hub)

| # | Spike | Go if | Fallback |
|---|---|---|---|
| S1b | **Loro bodies at scale and natively**: loro, yrs and automerge-rs in Rust on a block-structured trace through both ProseMirror bindings; coalesced, signed, encrypted body events; shallow-snapshot compaction keeping "rebuild from empty equals incremental" | Time to first render of a 1,000-block page < 100 ms; bytes per coalesced body event and per year reported; memory with 50 pages open on a mid-range iPhone reported; sync of 10k missing events over a blind relay timed; compaction keeps every S1a property | Yjs for bodies |
| S6 | **iroh, keys and revocation**: Mac (home Wi-Fi) ↔ iPhone (mobile network) ↔ VPS hub; pairing, granting and revoking a hub, rotation, recovery kit, concurrent rotation on two offline devices, a revoked device's backdated events, a hub hiding one device's events | Connects in < 3 s in 95% of 50 tries; direct in ≥ 70%; the revoked hub can't read the new epoch; backdated events rejected by every peer and the hub; withholding detected within one direct contact; restore with no server account | Self-hosted relay, then Tailscale |
| S7 | **Mailbox**: the phone writes encrypted batches to R2 and (with a provisioning profile) CloudKit while the Mac sleeps | No loss over 1,000 batches; defined behaviour when the quota is full | Hub only for async delivery |

The web client (M3) gets its own spike for OPFS limits, passkey PRF and the signed static origin.

### Phase 1: foundation (about 4–6 weeks, two or three agents)

Sequential, because everything builds on it:
- the Rust workspace and CI: lint, tests, `cargo-deny`, `cargo-vet`, the licence check and the simulation;
- the event format and reducer from S1a, with **per-domain reducer modules behind a registry and event-type ranges reserved per track** (the way Sprint reserves migration numbers), so tracks add files instead of editing one big `match`;
- **the ontology core** (built-in and custom types, fields and relation types, with cardinality read from data), because every track's rules read it;
- the SQLite schema and migrations from S3;
- the command API and TypeScript type generation;
- the Tauri shell with the `CoreClient` interface;
- the design tokens from Sprint's brand (ADR-0003 in Sprint).

### Phase 2: parallel build tracks

Each track owns its reducer module, its event-type range, its command files and its UI paths.

| Track | Contents | Milestone | Size |
|---|---|---|---|
| T1 Sprints and tasks | Board, Home, Capture, the sprint engine (ported), recurrences, derived closes, metrics | M1 | L |
| T2 Pages and journal | The editor (from S2), page tree, mentions and backlinks, templates, journal | M1 | L |
| T3 Library and files | Links (guarded fetcher in the helper, metadata, kinds), the blob store, previews | M1 | M |
| T4 Ontology admin UI | Custom fields, types and relation types (the rules are in Phase 1) | M1 | M |
| T5 Search and AI | FTS, the semantic index and embeddings, hybrid search, the AI gateway and ledger, rituals, Ask, suggestions | M1 | L |
| T8 Backups and ops | Snapshots, the restic repository, restore tests, signing, notarisation, updater, Homebrew cask | M1 | M |
| T6 Sync, devices and keys | Pairing, iroh transport, key management, revocation, recovery kit, the mailbox | M2 | L |
| T7 Hub | The headless binary, jobs, MCP over HTTP, Telegram, `/status`, container, NixOS module, cloud-init | M2 | M |

**Milestones:** M1, the Mac app replaces Notion and Sprint for planning, used daily on one Mac. M2, phone and hub. M3, the web client; Sprint's web app is retired.

## Consequences

- ✅ Fast, private and offline by default, with no server in the loop. Measured on a pessimistic VM at 100k objects: graph lookups under 2 ms p95, Home and board under 10 ms, full text under 50 ms.
- ✅ One Rust core with every rule, on every device and the hub; one signed event log that is sync, history, undo, audit and AI context.
- ✅ No running cost unless the user chooses a VPS hub or a paid plan; no vendor whose shutdown would strand the data.
- ✅ The hub can be blind or trusted per workspace, and a self-hosted trusted hub can run AI while the Mac sleeps.
- ⚠️ We own the sync protocol and the reducer. Their correctness rests on the simulation harness, which must exist before user data does.
- ⚠️ Every commit is fsynced with `fullfsync` before it can be sent; S3 decides whether group commit is needed.
- ⚠️ Moving a block while another device edits it loses that edit unless S2's fork succeeds.
- ⚠️ Two languages (Rust and TypeScript) and, unless the CEF runtime proves itself, a WebKit runtime that changes with macOS.
- ⚠️ End-to-end encryption makes lost keys unrecoverable.
- ⚠️ The web client needs a hub and trusts whoever serves its code; we serve it from a separate signed origin.
- ⚠️ Sprint's TypeScript domain code is a specification to port, not code to reuse.

## Open items for the user

1. The licence (section 13): FSL-1.1-ALv2 (recommended) or AGPL-3.0 + CLA.
2. Whether the default for a new user's own hub is blind (recommended) or trusted.
3. Whether to enrol in the Apple Developer Program now, so spikes can test the sandbox, notarisation, CloudKit and a TestFlight phone build.
4. Whether moving a block while another device edits it must keep the edit (then S2's loro-prosemirror fork is required work), or delete + insert is acceptable for M1.
