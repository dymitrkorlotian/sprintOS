# ADR-0001: The founding architecture

- **Status:** Proposed (2026-10-02). The user decides after Sprint's testing phase; a red-team review comes first.
- **Scope:** everything needed to start building Sprint OS: the shell, the language, the UI, the editor, the local database, sync, security, identity, background work, AI, files, backups, distribution, licensing and the order of work.
- **Detail lives in:**
  - [`architecture/sync.md`](../architecture/sync.md): the sync design;
  - [`architecture/security.md`](../architecture/security.md): the threat model and keys;
  - [`architecture/benchmarks.md`](../architecture/benchmarks.md): measured numbers;
  - [`architecture/research/`](../architecture/research/): sourced research notes, dated 2026-10-02.

## Summary

Sprint OS runs **the whole app on each device**: the domain logic, the database, the jobs and the AI calls. Devices stay in step through **an ordered, end-to-end encrypted log of domain operations**. A server is optional, in one of two modes:
- a **blind relay** that only orders ciphertext;
- a **trusted hub** (your own box) that holds the key, runs jobs around the clock and serves the phone and web.

| # | Decision | Choice | Runner-up | Why |
|---|---|---|---|---|
| 1 | Language and core | **TypeScript** everywhere; one domain core shared by desktop, hub, web and phone | Rust core + UniFFI | One language with the editor and UI; Sprint's tested domain logic (about 10k lines) carries over; SQLite in Node is already fast enough (benchmarks) |
| 2 | Desktop shell | **Electron**, with the core in a `utilityProcess` and a sandboxed renderer | Tauri 2 (Tauri 3's Chromium runtime once stable) | Same Chromium as the web build and as Playwright; no WebKit editor risk; the core stays in TypeScript with no sidecar. It is what Notion, Obsidian, AFFiNE, Anytype, Logseq and Claude Desktop ship. |
| 3 | UI | **React 19 + Vite + TanStack Router**, a pure SPA; Tailwind | Svelte 5 | The editor ecosystem is React-first. The same SPA runs in Electron, a browser and a phone webview. Next.js is server-centric and adds nothing locally. |
| 4 | Editor | **BlockNote** (MPL core only) on **Yjs** | Tiptap 3 (same ProseMirror and Yjs base) | Notion-like out of the box; collaboration is Yjs-only, and Yjs is the most mature ProseMirror CRDT binding |
| 5 | Local database | **SQLite** (FTS5; vectors via `sqlite-vec` or a SIMD/HNSW index in the core) | PGlite | 4–60× faster than PGlite in our tests; opens in 7 ms vs 0.5 s; runs natively on Mac, iOS and the browser (OPFS); decades of durability |
| 6 | Sync | **Own protocol: a server-ordered log of deterministic domain operations**, optimistic apply and rebase; Yjs for page bodies | PowerSync (if E2EE and serverless use are dropped); LiveStore as build-or-adopt | Keeps every rule (uniqueness, cardinality, no cycles, one sprint close) on every device, without a trusted server. A prototype converged under fuzzing. |
| 7 | Encryption | **E2EE for sync, files and backups from day one**; keys in the data protection keychain; recovery key; passkey-PRF for the web | Server-side encryption only | A relay breach then leaks only metadata. Adding E2EE later is a rewrite; starting with it is cheap. |
| 8 | Server topology | **None required.** Optional **hub**: the same Node program on a Mac mini, home box, VPS or hosted. Trusted or blind. | Always-on hosted backend | Never requires hardware; about $0; selling later means hosting blind relays |
| 9 | Device-to-hub networking | **Iroh** (QUIC peer-to-peer with relays; pair by QR code) | Tailscale (power users); Cloudflare Tunnel (only on purpose) | No account, no VPN app and no open ports for a non-technical buyer. 1.0 since 2026-06 (MIT/Apache). |
| 10 | Identity and tenancy | **Local-first identity** (device keys plus the workspace key); accounts only for a hosted relay; `workspace_id` on every row | Server accounts from day one | Works with no server. Tenancy is the workspace key, enforced by cryptography rather than RLS. |
| 11 | Background work | **A durable job queue in SQLite** inside the core, the same core as a login-item agent (`SMAppService`) while the window is closed, and on the hub when present | Inngest-like cloud jobs | Jobs run where the data is; lazy, idempotent catch-up stays the basis of correctness (Sprint's `ensureSprintState`) |
| 12 | AI | **Bring your own key** in the keychain; one `complete()` gateway with a usage ledger in the core; Voyage `voyage-4-lite` for embeddings, with a local model (`voyage-4-nano`) as a spike; index in SQLite; an **MCP server** with scoped tools | Bundled local LLM | Claude stays the quality bar; local inference is too weak and heavy for summaries |
| 13 | Files | **Content-addressed** (SHA-256) blobs in the app container, deduplicated; encrypted chunks synced lazily | Files inside the database | Large blobs stay out of SQLite and the log |
| 14 | Backups | **Automatic, encrypted, versioned:** `VACUUM INTO` snapshots plus files, through **restic** to a folder, an S3-compatible bucket (R2 or B2 free tiers) or the hub; a weekly restore drill shown in the app | Litestream (no client-side encryption) | Local-first without backups loses everything with the laptop |
| 15 | Distribution | **Developer ID, notarized, GitHub Releases**, signed auto-update; Mac App Store later | Mac App Store first | No review delays, own update channel. Code kept sandbox-clean, so the App Store and iOS stay a packaging step. |
| 16 | Licence | **All rights reserved for now**; decide at the first public release between AGPL + commercial and an MIT client + FSL relay. Never BlockNote's GPL `xl-*` packages without a licence. | Open source from day one | Keeps every option open while there is one author |

**Top risks** (more in [Risks](#risks-and-unknowns)):

1. **Our own sync protocol** is the hardest code in the project. Mitigations:
   - spike S3, with 10,000-run fuzzing as the gate;
   - LiveStore as a build-or-adopt alternative;
   - a small protocol on SQLite rather than a framework.
2. **Electron's upkeep:** a major every 8 weeks and native-module rebuilds. Mitigations: few native modules, an upgrade job in CI, and the shell kept thin so Tauri remains an exit.
3. **Losing the key means losing the data.** Mitigations: recovery key, every device as a key holder, encrypted backups, and the app showing how many key copies exist.
4. **Scope:** this rebuilds everything Sprint does, for one developer with agents. Mitigation: build in parity slices in Sprint's own order, with Sprint running until the desktop app covers daily use.
5. **BlockNote churn:** pre-1.0, with breaking minors and a Yjs 14 move in progress. Mitigation: pin exact versions; Tiptap 3 is the fallback on the same base.

---

## Context

Sprint OS is a personal operating system: weekly sprints with automatic close and rollover, tasks and ideas, Notion-like pages and a daily journal, a library of links, files, an editable ontology (custom fields, types and relations), and AI that runs the rituals. The product's requirements are Sprint's [`vision.md`](https://github.com/dymitrkorlotian/sprint/blob/main/docs/vision.md), [`context-graph.md`](https://github.com/dymitrkorlotian/sprint/blob/main/docs/architecture/context-graph.md) and [`specs/`](https://github.com/dymitrkorlotian/sprint/tree/main/docs/specs).

**Decided before this ADR (by the user, 2026-10-02):**
- local-first, Mac first, then web and phone;
- a new project and repo;
- an empty database, with no migration from Sprint;
- no export feature for now;
- never require a home server;
- about $0 running cost for the user;
- a data model that can serve many tenants later.

**The bar:** top-notch choices (optimized, secure, performant, stable), made from first principles and on evidence. Sprint's choices (Next.js on Vercel, Supabase Postgres + RLS, Drizzle, Inngest, BlockNote) are inputs, not constraints.

**What the domain needs, measured:**
- **Size:** one heavy user reaches 10k–100k objects and a few hundred thousand edges and events over years. That is small for a database; the bottleneck is latency, not scale.
- **Rules:** many cross-record rules (see `sync.md` → The problem). Today Postgres enforces them with 45 functions, 27 triggers and partial unique indexes.
- **Rich text:** Notion-style pages with `@` mentions.
- **Search:** full-text plus search by meaning, in several languages (English and Polish at least, so `ł` must fold to `l`).
- **AI:** calls to Claude, and embeddings at 512 dimensions.
- **Background work:** the weekly close, the daily catch-up, embeddings and link fetching.

## First principles

1. **The device is the computer.** Reads and writes never wait for a network. A cold start shows your data in under a second.
2. **The rules live in one place: shared domain code over a database that also refuses bad data.** The same code runs on every device, so every device agrees.
3. **Servers are optional and dumb by default.** A relay that can't read your data can't leak it. Trusting a server is opt-in: your own hub.
4. **One language, one UI codebase.** For one developer working with AI agents, every extra language or UI stack is a tax on every feature.
5. **Boring storage, careful sync.** Use the most proven embedded database; spend the novelty budget on sync, where nothing off the shelf fits.
6. **Measure before trusting.** Each risky choice has a spike with numbers to hit before the build starts.

---

## 1. Language and where the logic lives

| Option | Perf | Security | Maturity | Dev speed | Fit | Cost | Lock-in | Total |
|---|---|---|---|---|---|---|---|---|
| **TypeScript core** (Node on desktop and hub, browser worker on web, JS in the phone webview) | 4 | 4 | 5 | 5 | 5 | 5 | 4 | **32** |
| Rust core + UniFFI (Swift/Kotlin) + WASM (web) | 5 | 5 | 4 | 2 | 3 | 5 | 4 | 28 |
| Swift core (Apple only) | 5 | 5 | 5 | 2 | 2 | 5 | 1 | 25 |

*Scores are 1–5; for lock-in, 5 means least.*

- **Choice: TypeScript.**
  - The editor, the UI and the AI SDKs are TypeScript.
  - Sprint's pure domain code ports almost unchanged, with its tests: ISO weeks and time zones, the sprint engine's planning, RRULE recurrences, canonical URLs, tag names, field validation, chunking, rank fusion and suggestion votes.
  - SQLite through Node runs every relational query in our benchmark in under 15 ms at 100k objects. The one heavy loop, vector search, gets a native or WASM SIMD kernel (decision 5).
- **What would change it:** a phone app that must be fully native (Swift UI over shared logic), or measured CPU-bound hot paths that a SIMD kernel can't fix. Then move the core to Rust behind the same operation API. The operation-log design keeps that possible.

## 2. Desktop shell

| Option | Perf | Security | Maturity | Dev speed | Fit | Cost | Lock-in | Total |
|---|---|---|---|---|---|---|---|---|
| **Electron 44** | 3 | 4 | 5 | 5 | 5 | 5 | 4 | **31** |
| Tauri 2.12 | 4 | 5 | 4 | 3 | 3 | 5 | 4 | 28 |
| Native Swift/SwiftUI | 5 | 5 | 5 | 1 | 2 | 5 | 1 | 24 |
| Electrobun 2 / Wails 3 / Flutter | 3–4 | 3–4 | 2–4 | 1–3 | 2–3 | 5 | 1–3 | ≤24 |

*Versions as on npm, 2026-10-02.*

- **Choice: Electron.** Three processes:
  - a **thin main process** for windows, menus and updates;
  - a **core `utilityProcess`** running the domain engine, SQLite, sync, jobs, AI calls and key access;
  - a **sandboxed renderer** (context isolation, no Node) that talks to the core over a typed `MessagePort` API and never sees a key.
- **Why it wins for this app:**
  - **Same engine as the web build and the tests:** the editor (ProseMirror via BlockNote) runs on the same Chromium as the web build and Playwright.
  - **WebKit risk:** WebKit-only contenteditable and drag-and-drop bugs are the top regret reported by Tauri developers.
  - **No sidecar:** the TypeScript core needs no separate runtime, whereas Tauri would need either a Node sidecar ("Electron with extra steps") or a Rust rewrite.
  - **Production record:** every Notion-class block editor shipped on desktop runs on Electron (research, §5).
- **Electron's costs, plainly:**
  - a 100–200 MB download;
  - a new major every 8 weeks, three supported at a time;
  - native modules (`better-sqlite3`) rebuilt per major;
  - security that is "safe if you follow the checklist" rather than deny-by-default.

  Tauri's smaller RAM use mostly disappears under fair accounting (tauri#5889); its smaller download is real.
- **Hardening:**
  - fuses (`RunAsNode` off, ASAR integrity, only-load-from-ASAR);
  - the 20-item checklist;
  - a separate session for untrusted web content.

  More in `security.md`.
- **What would change it:**
  - Tauri 3's Chromium (CEF) runtime reaches stable (alpha on 2026-10-01);
  - download size becomes a selling problem;
  - spike S1 shows BlockNote works flawlessly in WKWebView.

  Keep the shell layer thin (one `platform` adapter) so a switch costs weeks, not months.

## 3. UI framework

| Option | Perf | Security | Maturity | Dev speed | Fit | Cost | Lock-in | Total |
|---|---|---|---|---|---|---|---|---|
| **React 19 (+ Compiler) + Vite + TanStack Router** | 4 | 4 | 5 | 5 | 5 | 5 | 4 | **32** |
| Next.js (static export in the shell) | 3 | 4 | 5 | 3 | 2 | 5 | 3 | 25 |
| Svelte 5 / Solid | 5 | 4 | 4 | 3 | 2 | 5 | 4 | 27 |

- **Choice:** a pure SPA built with Vite, with type-safe routes and URL state in TanStack Router.
  - The same build runs in the Electron renderer, a browser and a phone webview.
  - Styling is Tailwind with Sprint's Stone scale and Cobalt signal, carried over after the UI/UX pass.
  - Data comes from the core through **live queries**: the core tells the UI which queries changed after each applied operation.
- **Why not Next.js:** a static export loses server actions, route handlers and middleware, which is where Sprint's whole data path lives (29 server-action files). Locally it adds nothing.
- **What would change it:** a native phone UI (then React Native for the phone only, hosting the editor in a webview).

## 4. Editor

| Option | CRDT fit | Licence | Maturity | Notion-like out of the box | Total /20 |
|---|---|---|---|---|---|
| **BlockNote 0.55** (MPL core) | 4 (Yjs) | 4 | 3 | 5 | **16** |
| Tiptap 3 (MIT core) | 4 (Yjs) | 4 | 5 | 2 | 15 |
| ProseMirror directly | 5 | 5 | 5 | 1 | 16 (months of UI work) |
| Lexical 0.52 | 3 | 5 | 3 | 2 | 13 |
| Native Swift editor | 2 | 5 | 1 | 1 | 9 |

- **Choice: BlockNote on Yjs.**
  - Bodies are stored as Yjs documents. JSON and plain text are derived from them for search and embeddings.
  - Collaboration in BlockNote is Yjs-only, and `y-prosemirror` is the most mature ProseMirror CRDT binding. Loro's binding is 0.4 and Automerge's calls itself beta.
  - Pin exact versions.
  - Use only the MPL packages: the `xl-*` packages are GPL-3.0 or commercial (Business tier $390/month).
- **What would change it:** BlockNote stalls, or its churn costs more than a month a year. Move to Tiptap 3, which shares the same ProseMirror and Yjs base, so documents survive the move.

## 5. Local database

| Option | Perf | Security | Maturity | Dev speed | Fit | Cost | Lock-in | Total |
|---|---|---|---|---|---|---|---|---|
| **SQLite** (`better-sqlite3`/`node:sqlite` on Mac and hub; official `sqlite-wasm` on OPFS in the browser; native on iOS) | 5 | 4 | 5 | 4 | 4 | 5 | 5 | **32** |
| PGlite 0.5.8 (Postgres 18.3 in WASM) | 2 | 4 | 3 | 4 | 4 | 5 | 4 | 26 |
| Embedded native Postgres | 4 | 4 | 4 | 3 | 4 | 5 | 4 | 28 (but a server per device, major upgrades on user machines, no iOS or browser) |
| DuckDB / Kuzu / SurrealDB / RxDB | — | — | — | — | 1–2 | — | — | ruled out: analytics engines, an archived graph DB, or document stores without relational rules |

**Measured at 100k objects** (details in `benchmarks.md`):

| p50 / p95 ms | SQLite | PGlite | Native PG |
|---|---|---|---|
| Open | **7** | 390–550 | 30 |
| Full-text top 20 | **13 / 51** | 150 / 601 | 79 / 155 |
| Board week + tags | **5 / 10** | 30 / 78 | 14 / 35 |
| 2-hop graph | **2 / 4** | 5 / 12 | 3 / 6 |
| Write with event trigger | **0.1 / 3** | 2.4 / 6.5 | 1.9 / 3.6 |
| Exact vector top-10 (125k × 512) | 115 / 125 | 617 / 637 | **31 / 50** |

- **Choice: SQLite.**
  - **Speed:** fastest on everything but brute-force vectors, and 8× faster to load than PGlite.
  - **Platforms:** one engine for Mac, hub, browser (OPFS, as Notion and Logseq ship) and iOS.
  - **Maturity:** WAL durability with decades of field use.
  - **Fit with sync:** the sync design wants one engine everywhere, so every replica applies operations identically.
- **Why not PGlite:** PGlite ran Sprint's whole schema unchanged, which mattered when the plan was to migrate. With an empty start that advantage is gone, and what remains is:
  - 4–60× slower queries;
  - a half-second open;
  - a 1.6 GB footprint for the same data;
  - 32-bit WASM memory limits;
  - a younger durability record.
- **Integrity in SQLite** (each checked):
  - `STRICT` tables and CHECKs;
  - partial unique indexes for cardinality (one parent, one project, one journal per day);
  - triggers with recursive CTEs and `raise(abort)` for structure (no page cycles);
  - triggers for the event log.
  - The gap is that SQLite has no deferred constraint triggers. Designs that need them (Sprint's "every task has its typed row by commit") are designed away: typed columns live on the object row, and the mutator writes them in the same statement.
- **Search:**
  - **Full text:** FTS5 over a **folded** search column. The core normalises text (NFKD, strip marks, then map `ł→l` and the other letters without a decomposition), because FTS5's `remove_diacritics` doesn't fold `ł`.
  - **Meaning:** vectors are kept in SQLite and searched by:
    - an exact SIMD scan in the core (WASM SIMD or a small native addon) for up to about 50k chunks;
    - an HNSW index (`usearch`, int8) beyond that.

    Fused with full text by reciprocal rank, as in Sprint R4.2.
- **Encryption at rest:** FileVault by default. Optional SQLite3 Multiple Ciphers (the `better-sqlite3-multiple-ciphers` build), with the key in the keychain, if spike S2 shows the overhead is under 15%.
- **What would change it:** the hub had to serve many tenants with server-side SQL across workspaces. In that case the hosted hub would use Postgres, with devices still on SQLite; the operation log is engine-neutral.

## 6. Sync and the data model

**Choice: our own small protocol: a server-ordered log of deterministic domain operations, applied optimistically and rebased, with Yjs for page bodies.** It is described in [`architecture/sync.md`](../architecture/sync.md).

| Option | Perf | Security | Maturity | Dev speed | Fit | Cost | Lock-in | Total |
|---|---|---|---|---|---|---|---|---|
| **Own ordered operation log (+ Yjs bodies)** | 5 | 5 | 2 | 2 | 5 | 5 | 5 | **29** |
| LiveStore 0.4 (the same model, pre-1.0) | 4 | 4 | 2 | 4 | 4 | 5 | 3 | 26 |
| PowerSync + own write API | 4 | 3 | 4 | 4 | 3 | 3 | 3 | 24 |
| Electric + TanStack DB | 4 | 3 | 4 | 4 | 2 | 4 | 4 | 25 |
| Automerge / Loro documents | 3–5 | 5 | 3 | 2 | 2 | 5 | 4 | 24–26 |
| Evolu (E2EE, per-cell LWW) | 4 | 5 | 3 | 4 | 2 | 5 | 3 | 26 |
| CloudKit | 4 | 5 | 4 | 3 | 1 | 5 | 1 | 23 |
| Zero | — | — | — | — | 1 | — | — | ruled out: no offline writes, no native |

- **Why:**
  - Only a total order plus deterministic rules keeps *cross-record* invariants on every device without trusting a server.
  - PowerSync, the best off-the-shelf fit, needs a trusted Postgres and service per user. That rules out E2EE, rules out a Mac-only mode, and makes cost grow with users.
- **Evidence:** a throwaway prototype converged on every run, with every invariant holding. The prototype was SQLite replicas, a blind sequencer, trigger-captured undo for rebase, and concurrent Yjs edits. A negative control with the rebase off diverged in every run (`benchmarks.md`).

**Data model, fresh, keeping Sprint's ontology concepts** (context-graph.md):
- **Objects:** `objects` with a UUIDv7 id (or UUIDv5 for singletons), `workspace_id`, `type`, `title`, a derived `body_text` and a folded `search_text`, `properties` as JSON with a validity check per type, and time and provenance columns.
- **Typed columns:** typed columns for hot fields of work items and sprints, on the object row or in 1:1 tables written by the same mutator.
- **Relations:** first-class rows with `valid_from`/`valid_to`, `source` and `status` (suggested, accepted, rejected). Ended, never deleted.
- **Ontology:** built-in types and relations in code; custom fields, types and relation types as rows (Sprint's FB-3, ADR-0005 concepts).
- **Event log:** written by triggers, with the operation id.
- **Derived tables:** a `body` table of Yjs documents; derived indexes (FTS, vectors) that are not synced but rebuilt locally from synced data.
- **What would change it:**
  - S3 fails its gates and LiveStore passes them: adopt LiveStore.
  - The user drops E2EE and a server-optional design: PowerSync on a hosted Postgres becomes the faster path.

## 7. Security and encryption

See [`architecture/security.md`](../architecture/security.md) for the threat model:
- a lost laptop;
- a sync-server or hub breach;
- malicious page or file content;
- prompt injection;
- SSRF;
- the supply chain;
- updates;
- MCP agents;
- key loss.

**Choice:**
- **E2EE everywhere data leaves the device** (operations, snapshots, files, backups), with XChaCha20-Poly1305 via libsodium.
- **A random workspace key**, wrapped by:
  - per-device keys in the **data protection keychain**;
  - a recovery key;
  - a passkey (PRF) for the web client.
- **AI keys** stay in the keychain and are read only by the core process.

What would change it: the user prefers server-side search and AI over privacy. Then the trusted hub covers that need without dropping E2EE for the relay.

## 8. Topology: Mac, plus an optional hub

| Setup | What it gives | Cost |
|---|---|---|
| **Mac only** | Everything, offline. Jobs run while the Mac is awake (login-item agent). Backups to a folder or bucket. | $0 |
| **+ blind relay** (hosted by us later, or any VPS) | Multi-device sync, phone and web; jobs run on devices | $0 (a free tier we host; or the user's own) |
| **+ trusted hub** (Mac mini, old PC, NAS, $4 VPS) | All of the above plus 24/7 jobs: the daily rollover, embeddings, link fetching, AI rituals, the always-on agent and MCP over the private network. It also serves the web client. | Hardware the user already has; never required |

- **The hub is the core plus a sequencer, packaged three ways:**
  - a launchd agent on a Mac;
  - a systemd service;
  - a Docker image.

  It deploys from code (one `compose.yaml` or a bootstrap script) and reports its health to the app.
- **Networking:** **Iroh** by default:
  - pair by scanning a QR code;
  - QUIC with hole punching;
  - the hub runs its own relay, with n0's public relays as fallback.

  **Tailscale** (or Headscale) is the power-user option. **Cloudflare Tunnel** is used only when someone wants a public URL. No open ports.
- **What would change it:** Iroh's relays prove unreliable in spike S6 (Tailscale becomes the default), or a hosted business needs a public endpoint (an HTTPS relay behind Cloudflare).

## 9. Identity and tenancy

- **No account is needed to use the app.** A device has a key pair; a workspace has a random key. Joining a device means receiving the workspace key, wrapped for that device, by QR pairing or the recovery key.
- **Accounts exist only for a hosted relay:** a passkey login and billing later.
- **Tenancy:** `workspace_id` is on every row, and a device can hold several workspaces. On a shared relay, tenancy is enforced by ciphertext per workspace plus per-account access to its logs. Sharing a workspace with another person (teams, later) is wrapping its key for them.
- **What would change it:** team workspaces with roles become a product goal. Per-member permissions would then need a trusted server or capability-based keys (Ink & Switch's Keyhive work).

## 10. Background work

- **Where jobs run:** a **job queue table in SQLite** in the core: durable, retried with backoff, idempotent, one at a time per key. Jobs:
  - the daily catch-up (`ensureSprintState`: close weeks, create sprints and recurring tasks);
  - embedding changed objects;
  - fetching link metadata;
  - AI rituals.
- **When jobs run:**
  - **While the app is open,** in the core process.
  - **When the window is closed,** the same core runs as a **login-item agent** registered with `SMAppService` (macOS 13+; Electron's `setLoginItemSettings` maps onto it).
  - **On a trusted hub, around the clock.**
- **Correctness:** this stays **lazy and idempotent** (Sprint's ADR-0002 design). Opening any device catches up; a missed job only delays, never corrupts. With several devices, jobs that change data are operations with deterministic ids, so two devices doing the same catch-up converge.
- **Crash reports:** opt-in, crash-only, scrubbed (`security.md` → Telemetry).

## 11. AI

- **Bring your own key:** the Anthropic and Voyage keys are pasted once and kept in the keychain. Anthropic's terms allow end users to use their own API keys; there is no "Sign in with Claude" for third-party apps.
- **The gateway:** `complete()` in the core, as in Sprint:
  - every call passes a `usage` argument and lands in an `ai_usage` table, which drives the spend view and monthly caps;
  - the model is chosen per feature;
  - prompt caching is on;
  - it runs with no tools when untrusted content is present.
- **Embeddings:** `voyage-4-lite` at 512 dimensions through the API by default ($0.02 per million tokens after 200M free).
  - **Spike S5** tests `voyage-4-nano` (open weights, Apache-2.0), reportedly in the same embedding space, on device. If it matches at 512 dimensions on an English and Polish test set, embeddings become free, offline and private, with no re-index.
  - Fallbacks: Qwen3-Embedding-0.6B or EmbeddingGemma-300M, with a re-index.
- **Local LLMs:** not now. A useful model is a 5+ GB download and RAM hog, and is weaker at Polish and multi-document summaries. Apple's on-device model has no Polish. The model is pluggable, so revisit yearly.
- **MCP server** with scoped, audited tools (`security.md` → MCP and agents). "Ask your OS" and the rituals can run as an agent on the hub, with capture and chat through optional messaging adapters.
- **Automations** (roadmap → Later): expose them, don't build an engine. Webhooks out (signed), MCP in and operations as the API cover n8n, Shortcuts and agents. A small built-in rule set ("when X, do Y") comes later if real use asks for it.

## 12. Files

- **Storage:** blobs live in the app container, named by their SHA-256, so a file saved twice is stored once.
- **Metadata:** the `file` object holds the hash, name, type and size.
- **Sync:** blobs travel as encrypted chunks to the relay, hub or the user's bucket, and devices fetch them lazily. The phone gets thumbnails first.
- **Rendering:** previews follow `security.md` (SVG as an image, PDFs in a sandboxed viewer).

## 13. Backups

Export is not a feature, but backups are a must.

- **What and where:** every hour, a `VACUUM INTO` snapshot plus new file blobs, sent through **restic** (encrypted, deduplicated, versioned; BSD-2) to the user's choice:
  - a folder (iCloud Drive or an external disk);
  - an S3-compatible bucket (Cloudflare R2 has 10 GB free and free egress);
  - the hub's append-only restic server.
- **Retention:** 48 hourly, 30 daily, 12 monthly.
- **Restore drill:** weekly and automatic. Restore the latest snapshot to a temporary place, run `integrity_check`, compare row counts and hashes, and show "Last verified backup: 2 days ago" in the app.
- **A design property, not a feature:** the database is plain SQLite with a documented schema, so it can always be read outside the app.

## 14. Distribution and operations

- **Distribution:** Apple Developer Program ($99/yr), Developer ID, hardened runtime, `notarytool`, stapled. Shipped on GitHub Releases.
- **Updates:** Electron's `autoUpdater` (Squirrel.Mac verifies the Apple signature), with a feed signed by our own Ed25519 key, no downgrades, and the update key kept offline.
- **Mac App Store:** later and optional. The code stays sandbox-clean (data in the container, bookmarks for any outside folder, no downloaded code), so the App Store and iOS remain a packaging step.
- **Testing:**
  - Vitest for the core and mutators;
  - **property tests** (fast-check) that every mutator keeps every invariant;
  - a **deterministic sync simulator** (seeded devices, network faults, crashes) run on every PR, plus 10k seeds nightly;
  - Playwright for the SPA, and its `_electron` driver for the desktop app;
  - crash-injection tests for durability.
- **Supply chain:** see `security.md`. In short: pnpm 11 defaults, a release-age gate, actions pinned by SHA, and a protected signing environment.

## 15. Licensing and business

| Option | For | Against |
|---|---|---|
| **All rights reserved now** (chosen) | Every later option stays open; one author can relicense anything | Nobody else can contribute yet |
| AGPL-3.0 client + commercial licence | Stops closed forks; proven (AppFlowy, Logseq, Standard Notes) | Needs a CLA for contributions; GPL and the App Store conflict |
| MIT/Apache client + FSL relay (Sentry/PowerSync style) | Widest adoption; the relay business is protected for two years per version | Anyone can fork the client |

- **Dependency check:** every recommended dependency is permissive:
  - MIT, Apache or BSD: Electron, React, Vite, Yjs, SQLite (public domain), sqlite-vec, libsodium (ISC), restic (BSD-2), Iroh;
  - MPL: BlockNote core.
- **Never include `@blocknote/xl-*`** without a commercial licence.
- **The business shape this enables:** a free local app; paid hosted relay, backup storage and hub; users bring their own AI key. It is the Obsidian pattern, with E2EE.

---

## Repository and what carries over

**From Sprint into sprintOS:**

| What | How |
|---|---|
| `docs/vision.md`, `docs/modules.md` | Copied into `docs/product/`, as requirements |
| `docs/architecture/context-graph.md`, ADR-0001 and ADR-0005 concepts (objects, typed edges with validity and provenance, event log, editable ontology) | Copied as **concepts**; the physical design here replaces Postgres |
| `docs/specs/*` (sprints, home, capture, recurring, task detail, pages, library, projects, objects, smarter) | Copied as requirements, each marked with what Sprint built and what changes locally |
| `docs/brand/` and ADR-0003 (Graphite, Stone, Cobalt) | Carried over after the UI/UX pass, which also sets the desktop look |
| **Pure code in `packages/core`, with tests:** time and ISO weeks, sprint engine planning, recurrence (RRULE), `canonicalUrl`, `normalizeTagName`, field definitions and values, collection queries, chunking, rank fusion, suggestion votes, Markdown conversion | Copied and adapted (about 10k lines plus 8k of tests). It has no framework imports, so it carries over best. |
| `packages/db` (Postgres, Drizzle, RLS), `apps/web` (Next.js), Inngest, Supabase Auth | **Not reused.** The data path is rewritten as mutators over SQLite. Components may be lifted one by one after the UI/UX pass. |

**What happens to Sprint, the web app:**
- It keeps running, unchanged, until Sprint OS reaches the **daily-use gate** (step B4 below): sprints, Home, Capture, pages and the journal work on the desktop for two real weeks.
- Then the user switches. Sprint stays read-only for a grace period and is then shut down, along with the Supabase project and the Vercel app. There is no migration (decided).
- **In the long run, Sprint OS's own web and phone clients replace it.** The web client is the same SPA, running the core in a browser worker on SQLite/OPFS and syncing through the relay or hub, with the key unwrapped by a passkey. The phone is the same SPA in Capacitor with native SQLite, plus Swift extensions for share and widgets.

---

## Execution plan

### Phase S: spikes (about 3 weeks; they run in parallel; each ends with a go/no-go note in `docs/spikes/`)

| Spike | Proves | Go if |
|---|---|---|
| **S1 Shell + editor** | Electron with the core `utilityProcess`, sandboxed renderer, BlockNote on Yjs. For comparison, the same editor in WKWebView. | Cold start to an interactive Home with 100k objects **< 1 s** on an M1; idle RAM **< 300 MB**; a 5,000-block page types at 60 fps; Safari-only editor bugs counted |
| **S2 Storage** | The SQLite schema with ontology triggers, folded FTS5, crash safety, optional encryption | Every Home, board, search and graph query **< 50 ms p95** at 100k objects on an M1; `kill -9` during writes 1,000× gives zero corruption; Polish folding passes; encryption overhead measured |
| **S3 Sync** | The protocol in `sync.md`; build or adopt LiveStore | **Zero divergence in 10,000 fuzz runs** (3–5 devices, offline, crashes, rich text); rebase p95 **< 50 ms**; 100k-object bootstrap **< 30 s** |
| **S4 Keys + crypto** | Data protection keychain from Electron under Developer ID; the E2EE envelope; recovery; passkey PRF in Safari and Chrome | Keys never reachable from the renderer; restore from the recovery key alone works |
| **S5 Local embeddings** | `voyage-4-nano` on Apple Silicon against `voyage-4-lite` | Same space at 512 dimensions (cosine ≥ 0.95 on the same texts); recall@10 within 3 points on an English/Polish set; **< 20 ms** per chunk on an M1 |
| **S6 Hub + networking** | The hub binary; Iroh pairing from Mac and phone; the relay | Pairing in under 1 minute with no account; sync works across two different home networks; the hub deploys from one file |
| **S7 Vector search** | A SIMD scan, `usearch` HNSW and `sqlite-vec` int8 at 125k chunks | Top-10 **< 50 ms p95** with recall@10 ≥ 0.95 |

**If a spike fails:**
- **S1:** look at Tauri again.
- **S2:** a narrower schema, or reconsider PGlite.
- **S3:** LiveStore; failing that, PowerSync, with the user's agreement to drop E2EE.
- **S5:** stay on the API.

### Phase B: build (each step is a PR or a few; steps on one line can run in parallel)

| Step | Content | Size |
|---|---|---|
| B0 | Repo, pnpm 11 workspace, CI (lint, types, tests, the sync simulator), signing and notarization pipeline, update feed | M |
| B1 | Core: schema, ontology, mutators, operation log, job queue, live queries · Shell: Electron processes, typed RPC, keychain helper | L · M (parallel) |
| B2 | Sprint loop: sprints, board, Home, Capture, recurring, review (Sprint R1 specs) | L |
| B3 | Pages and journal (BlockNote/Yjs, tree, mentions, templates) · Search (FTS5, folding) | L · M (parallel) |
| **B4** | **Daily-use gate:** two real weeks on the desktop app; then the Sprint switch-over begins | — |
| B5 | Library (links, guarded fetcher) · Files (content-addressed) · Editable ontology (FB-3) | M · M · L (parallel) |
| B6 | Sync: relay, Iroh, pairing, E2EE · Backups with restore drill | L · M (parallel) |
| B7 | AI: gateway, ledger, embeddings, related items, suggestions, insights, Ask your OS · MCP server | L · M (parallel) |
| B8 | Hub packaging (launchd, systemd, Docker), health in app, messaging adapter | M |
| B9 | Web client (SPA + core in a worker, OPFS, passkey) | L |
| B10 | Phone (Capacitor, native SQLite, share extension) | L |

### What happens to the web app during and after

- **During:** Sprint runs as is, with `#feedback` fixes only.
- **After B4:** Sprint is the fallback.
- **After B9:** the Sprint OS web client replaces Sprint's web app, and Sprint is shut down.

## Risks and unknowns

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| The sync protocol has a subtle bug (lost or diverged data) | Medium | High | S3 fuzz gate; nightly 10k-seed simulation; operations are idempotent and logged; backups with drills |
| Electron upkeep (6 majors a year) slows feature work | High | Low | Automate upgrades in CI; few native modules; consider `node:sqlite` to drop the native rebuild |
| Key loss locks the user out | Low | High | Recovery key at setup, key copies shown in the app, encrypted backups with their own recovery |
| BlockNote breaking changes, Yjs 14 move | High | Medium | Pin versions; one adapter module; Tiptap fallback |
| OPFS durability in browsers and webviews (web, phone) | Medium | Medium | Only a client: the device holds pending operations and the relay has the log; tested in B9 and B10 |
| Vector search too slow at scale | Low | Low | S7; most users have under 30k chunks |
| Iroh relays or NAT traversal fail in the field | Medium | Medium | Hub-hosted relay; Tailscale option |
| The rebuild takes longer than hoped | High | Medium | Parity slices in Sprint's order; Sprint keeps running; the B4 gate |
| Facts marked unverified in the research (`voyage-4-nano`'s shared space, Tailscale's free limits, Supabase news) | — | Low | None is load-bearing except `voyage-4-nano`, which S5 tests |

**Not settled here, on purpose:** pricing; team workspaces; the licence (at the first public release); whether a hosted relay is run by us.

## Sources

Every external claim is sourced and dated in the research notes:
- [`research/01-shell.md`](../architecture/research/01-shell.md): shells and UI;
- [`research/03-sync.md`](../architecture/research/03-sync.md): sync and CRDTs;
- [`research/04-editor-ai.md`](../architecture/research/04-editor-ai.md): editors and local AI;
- [`research/05-security-dist.md`](../architecture/research/05-security-dist.md): security, distribution, supply chain, licences;
- [`research/06-hub-mcp-backup.md`](../architecture/research/06-hub-mcp-backup.md): hub, networking, MCP, backups.

**Caveats on the evidence:**
- Many vendor sites were blocked from the research environment. Their facts come from GitHub sources, package registries or search summaries, and each note marks which; anything unverified is marked.
- Versions are as of 2026-10-02.
- Measured numbers are in [`benchmarks.md`](../architecture/benchmarks.md). They come from a cloud VM and are to be repeated on a Mac in S2.
