# 03: Sync and the data model (research for the local-first Mac/phone ADR)

Research date: **2026-10-02**. Research only; nothing in `/home/user/sprint` was changed.

## How this was researched, and its limits

- **Network limits.** The sandbox proxy blocked most vendor sites (powersync.com, electric-sql.com, loro.dev, automerge.org, zero.rocicorp.dev, jazz.tools, infoq.com, supabase.com, webkit.org, caniuse, news.ycombinator.com, x.com). Three kinds of evidence were used instead:
  1. **Primary sources I read directly:** npm registry metadata (versions, publish dates, licences), GitHub raw READMEs and docs sources (PowerSync docs repo, Zero docs repo, BlockNote docs, Actual Budget source code, crdt-benchmarks READMEs, a reverse-engineering write-up of Linear's sync engine), and Apple's developer documentation JSON.
  2. **Web-search snippets** for vendor pages I could not open. These are marked *(search snippet)*. Treat them as weaker evidence.
  3. **Things I could not verify.** These are marked **UNVERIFIED**.
- The web-search budget ran out near the end. Gaps are listed in the last section.
- **Dates.** "npm, 2026-09-16" means the npm registry `time` field for that version, read on 2026-10-02. "Accessed 2026-10-02" applies to every URL unless a publication date is given.

---

## 0. TL;DR for the ADR

1. **The domain is mostly records, not documents.** It has tasks, sprints, objects, edges, an event log, and server-side invariants enforced by Postgres today. Examples: no cycles in `child_of` (migration 0013), at most one `part_of` project (0022), one journal per day (0012), one link per canonical URL (0016). Pure CRDTs cannot enforce cross-record invariants. The data layer should stay **server-authoritative**: clients make optimistic local writes, and the server validates, orders and can reject them. The one exception is **page bodies (rich text)**, which need a text CRDT.
2. **BlockNote only supports Yjs** for collaboration (`withCollaboration` in `@blocknote/core/yjs`; docs list only Yjs providers). So page bodies should be Yjs documents, stored as binary updates in a row and synced like any other row. Loro and Automerge have ProseMirror bindings but no BlockNote integration.
3. **Best fit with the current stack:** PowerSync (Postgres → client SQLite; writes go through *your* backend; Swift, Kotlin, JS and Node SDKs are GA; free self-host under FSL; Supabase integration) or a hand-built Linear/Replicache-style op-log on Supabase. ElectricSQL plus TanStack DB is a credible web-first option, but it has no native Swift client. Zero is web-only and does not support offline writes.
4. **E2EE conflicts with three things the product does today:** server-side AI (Voyage embeddings, Claude calls), Postgres full-text search, and server-enforced invariants. The web app can still decrypt with a passkey-PRF key, since PRF ships in Chrome, Safari 18+ and Firefox 135–148+. But the whole AI and search pipeline would have to move to the clients. Don't adopt it by default. If needed, use per-field E2EE ("vault" notes) later.
5. **CloudKit** is free and supports E2EE with Advanced Data Protection. However, encrypted fields cannot be read from CloudKit JS. It would mean leaving Postgres and RLS, and it rules out multi-tenant selling to non-Apple users. Reject it as the primary transport.

---

## 1. CRDT libraries

### 1.1 Automerge

| Fact | Source |
|---|---|
| **3.0** (mid-2025) re-implemented the in-memory representation on the columnar on-disk format. Memory use fell by more than 10×, and by about 100× in some cases. Pasting *Moby Dick* takes 700 MB in Automerge 2 and 1.3 MB in Automerge 3. A document that had not loaded after 17 h loads in 9 s. File format unchanged; API nearly backward-compatible. | https://automerge.org/blog/automerge-3/ (search snippet; HN thread https://news.ycombinator.com/item?id=44777086, Aug 2025) |
| Latest is **@automerge/automerge 3.5.0** (MIT). | npm, 2026-09-16 |
| **automerge-repo**: the `latest` npm dist-tag points to **2.6.0-alpha.3**, i.e. an alpha is published as "latest". | npm, 2026-08-07 |
| The reference **automerge-repo-sync-server** is "an unsecured Express app", "partly for demonstration purposes". It has no auth, so you must add your own. | https://github.com/automerge/automerge-repo-sync-server README (accessed 2026-10-02) |
| **Rich text:** Automerge 2.2 added marks and block markers; `@automerge/prosemirror` is the reference binding, using a `SchemaAdapter` to map ProseMirror nodes. | https://automerge.org/blog/rich-text/, https://automerge.org/docs/reference/documents/rich-text/ (search snippets) |
| **Swift:** `automerge-swift` (MIT) supports iOS 13+, macOS 10.15+ and Swift 6 mode. A 2026 PR updates the core to 0.12 (**the PR seen was in a fork, `duncan/automerge-swift`, so treat as UNVERIFIED upstream**). There is also `automerge-repo-swift`. | https://swiftpackageindex.com/automerge/automerge-swift (search snippet) |
| **Trees:** no native movable-tree type. Moves are delete plus re-insert (cycles are not addressed). | General knowledge; not re-verified this session |

### 1.2 Loro

| Fact | Source |
|---|---|
| **Loro 1.0** has a stable data format. Text and lists use **Fugue**. It has a rich-text CRDT (Peritext-like), a **Movable Tree** ("ensures no cyclic dependencies exist in the tree after merging concurrent move operations"), a Movable List and an LWW Map. Bindings: Rust, JS (WASM) and Swift. | https://loro.dev/blog/v1.0 (search snippet; released 2024), GitHub README https://github.com/loro-dev/loro (accessed 2026-10-02) |
| Movable tree is based on Kleppmann et al., "A highly-available move operation for replicated trees": all moves are sorted, and a move that would create a cycle is skipped. | https://loro.dev/blog/movable-tree (snippet), https://martin.kleppmann.com/papers/move-op.pdf |
| Latest is **loro-crdt 1.16.4** (MIT). | npm, 2026-09-30 |
| **loro-prosemirror 0.4.4** (MIT) provides sync, undo and cursor plugins. It is still 0.x. | npm 2026-08-22; README https://github.com/loro-dev/loro-prosemirror |
| Bundle is about 0.9 MB gzipped (the 1.0-beta measurement). Yjs is about 25 KB. | zxch3n/crdt-benchmarks README (below) |

### 1.3 Yjs / yrs

| Fact | Source |
|---|---|
| The stable line is **yjs 13.6.33**. | npm, 2026-09-23 |
| **v14 is still in release candidates.** The latest is v14.0.0-rc.28 (2026-09-29), with no final v14.0.0 yet. v14 drops the experimental move feature, adds an `AttributionManager` (per-client authorship) and a new YType. The binary format stays parseable. | https://github.com/yjs/yjs/releases/tag/v14.0.0-rc.28 (search snippet), https://github.com/yjs/yjs/issues/694 |
| **BlockNote 0.55.0** (2026-09-22) depends on *both* `yjs ^13.6.27` / `y-prosemirror ^1.3.7` *and* `@y/y ^14.0.0-rc.23` / `@y/prosemirror ^2.0.0-6`. BlockNote is mid-migration to Yjs 14. | npm `@blocknote/core@latest` dependencies, read 2026-10-02 |
| y-prosemirror 1.3.7 (2025-07-03); **Hocuspocus 4.7.0** (MIT, 2026-09-09); **y-sweet client 0.9.1** (MIT, 2025-09-16, no release in a year). | npm |
| `yrs` (Rust port) and `ywasm` exist. A Swift binding exists (y-crdt/y-uniffi) but its maturity was **UNVERIFIED** this session. | crdt-benchmarks lists ywasm 0.17.4 |
| **Trees:** no native tree type. You model a tree with a parent pointer in a Y.Map (LWW), which can create cycles under concurrent moves, or with nesting, where a move means delete plus copy. | General; consistent with v14 dropping move |

### 1.4 Benchmarks

Two primary tables, read directly from GitHub:

- **dmonad/crdt-benchmarks** (the Yjs author). Old versions: yjs 13.6.11, ywasm 0.9.3, loro 0.10.1, automerge 2.1.10. Node 20, i5-8400. Real-world editing trace B4 (≈260k ops):

  | B4 | yjs | ywasm | loro 0.10 | automerge 2.1 |
  |---|---|---|---|---|
  | time | 5,714 ms | 28,675 ms | 3,089 ms | 14,326 ms |
  | docSize | 159,929 B | 159,929 B | 258,228 B | 129,116 B |
  | parseTime | 39 ms | 16 ms | 13 ms | 1,805 ms |

  Source: https://github.com/dmonad/crdt-benchmarks README (accessed 2026-10-02). Its notes: memUsed excludes WASM memory. Loro's parse time benefits from its "snapshot" (which stores the in-memory state). Automerge does B4 in about 1 s if all ops are in a single `change`.

- **zxch3n/crdt-benchmarks** (the Loro author's fork). Newer versions: yjs 13.6.15, loro 1.0.0-beta.2, automerge 2.1.10.

  | B4 | yjs | loro 1.0β | automerge 2.1 |
  |---|---|---|---|
  | time | 2,616 ms | 2,271 ms | 7,109 ms |
  | docSize | 226,981 B | 230,556 B | 129,116 B |
  | parseTime | 27 ms | 6 ms | 1,185 ms |
  | gz bundle | 25 KB | 894 KB | 591 KB |

  Source: https://github.com/zxch3n/crdt-benchmarks README.
- **Neither table includes Automerge 3.x**, whose load-time and memory changes are large (§1.1). A "2026" comparison table on pkgpulse.com (https://www.pkgpulse.com/guides/yjs-vs-automerge-vs-loro-crdt-libraries-2026) gives different numbers (e.g. Loro 68 kB doc, Yjs 160 kB). **UNVERIFIED and probably unreliable**, so don't cite it.
- **For this app none of this matters much.** A personal page is kilobytes to low megabytes, and all three libraries load such documents in milliseconds. Bundle size (Yjs is the smallest) and editor integration matter more.

### 1.5 What CRDTs can and cannot do for this domain

| Need | CRDT behaviour |
|---|---|
| Rich text (page body) | Yjs (y-prosemirror), Loro (Fugue + Peritext-like marks), Automerge (marks + block markers). All are good. Only **Yjs** works with BlockNote. |
| Page tree with "at most one parent" and no cycles | **Loro Movable Tree** guarantees one parent and no cycles by construction. Yjs and Automerge do not: a parent-pointer register gives one parent, but concurrent moves can form a cycle. A server-ordered op-log handles it by rejecting the move, which is what the DB trigger in migration 0013 does today. |
| "Item part of at most one project" | Can be made true *by construction*: store `project_id` as a single-valued LWW field on the item, not as a set of edges. Any CRDT or LWW row sync then keeps the invariant. If kept as edge rows, two offline devices can each add an edge, and only a server can reject one. |
| Uniqueness (one journal per day, one link per canonical URL) | CRDTs **cannot** enforce this. Two offline creates give two objects. Workaround: **deterministic IDs** (e.g. `uuidv5(workspace, 'journal', date)`, `uuidv5(workspace, canonicalUrl)`), so concurrent creates *merge into the same object* instead of conflicting. |
| Cross-object rules (sprint close/rollover moves N tasks atomically; status transitions; "cannot reopen a closed sprint") | CRDTs cannot express this: each field merges on its own, so a half-applied close can merge with a concurrent edit. Needs a server-ordered mutation (Linear/Replicache/Zero/PowerSync-backend style) or client-side rebase. Rule of thumb: **automatic rituals run on one authority** (the server or a designated device), not on every replica. |
| Append-only event log | Easy in every model: a grow-only set keyed by event id. Ordering needs an HLC (Actual) or a server sync id (Linear). |

---

## 2. Sync engines

Legend: **SA** = server-authoritative. **Lic** = licence.

### 2.1 ElectricSQL (Electric)
- **1.0 GA on 2025-03-17**: "APIs are stable… ready for mission critical, production apps". https://electric.ax/blog/2025/03/17/electricsql-1.0-released (search snippet)
- **Read path only.** Electric is "a read-path sync engine for Postgres". It syncs **Shapes** (partial replication) over an HTTP API that works with CDNs. Writes go through your own API; Electric streams the result back via logical replication. https://github.com/electric-sql/electric README (accessed 2026-10-02)
- Recommended client stack is **TanStack DB** with optimistic mutations: the API returns a txid, and the client waits for it on the stream and then drops its optimistic state. https://electric-sql.com/blog/2025/07/29/local-first-sync-with-tanstack-db (snippet)
- `@electric-sql/client` 1.5.28 (Apache-2.0, npm 2026-09-09). The server is open source and self-hostable (Elixir).
- **Electric Cloud pricing** (2026-04-02): reads are free and unlimited; $1 per million writes; $0.10/GB-month retention; pay-as-you-go is $0/month and bills under $5 are waived. https://electric-sql.com/blog/2026/04/02/electric-cloud-pricing (snippet)
- **Offline/native:** no Swift SDK. The client persists shapes, but durable offline write queues are left to the app or TanStack DB. **Conflict model:** whatever your API does (SA). **E2EE:** not built in.
- **Production users:** e.g. Trigger.dev realtime (https://trigger.dev/blog/how-we-built-realtime, snippet).

### 2.2 PowerSync
- **Architecture:** Postgres (also MongoDB, MySQL beta, SQL Server beta, Convex experimental) → PowerSync Service → **SQLite on every client**. Client writes go into an ordered **upload queue** (PUT/PATCH/DELETE with a per-client op id) that **your backend** drains. The docs call this "server-authoritative reconciliation". The default behaviour is per-field LWW (as received by the server, deletes win). The backend may **reject or validate** writes, e.g. reject updates to a completed order. Yjs CRDT data can be stored and synced as rows. Source: powersync-docs repo `handling-writes/handling-update-conflicts.mdx` (GitHub, accessed 2026-10-02).
- **SDK status** (powersync-docs `resources/feature-status.mdx`, accessed 2026-10-02):
  - GA: **Swift**, Kotlin, JS/Web, Node.js, Dart/Flutter, and Sync Streams.
  - Beta: Rust, .NET, Capacitor, Drizzle, Room.
  - Alpha: **Tauri SDK**, GRDB (Swift), TanStack DB.
  - The Swift SDK was rewritten in pure Swift in v1.14.0, with no Kotlin XCFramework (search snippet from https://powersync.com/blog/powersync-changelog-may-2026).
- npm versions: `@powersync/web` 2.4.2, `@powersync/node` 1.1.1 (both 2026-10-01, Apache-2.0); `@powersync/tauri-plugin` 0.0.6 (2026-08-03).
- **Licence:** the service is **FSL-1.1-ALv2** (Functional Source License). Use is free except for a competing service, and each version converts to Apache-2.0 later (2 years under FSL's standard terms; the conversion period is **not re-read**). "Open Edition" self-host is feature-equivalent to Cloud. Client SDKs are Apache-2.0. Sources: https://github.com/powersync-ja/powersync-service LICENSE (read 2026-10-02), https://powersync.com/blog/powersync-open-edition-release (snippet).
- **Cloud pricing** *(search snippet via G2/powersync.com/pricing, not opened)*: Free tier with 2 GB synced/month, Pro from $49/month, Team from $599/month.
- **Supabase:** there is a first-class Supabase integration guide plus templates (https://docs.powersync.com/integrations/supabase/local-development, snippet; https://github.com/powersync-community).
- **E2EE:** possible at the app level, either by encrypting columns client-side or by mirroring them into local-only decrypted tables. There is a demo E2EE chat with Supabase (https://powersync.com/blog/building-an-e2ee-chat-app-with-powersync-supabase, snippet). Encryption at rest on the client: SQLCipher/SQLite3MultipleCiphers (beta).
- **Maturity:** about 10+ years of the JourneyApps lineage (copyright "2023-2026 Journey Mobile, Inc."). Production use is broad; specific named customers were **not verified** this session.

### 2.3 Zero (Rocicorp)
- **1.0 shipped around June 2026**, after about 2 years and 50+ releases. 1.0 was "very minor" over 0.26.2. It added a schema-change hook for **Supabase** publications. https://www.infoq.com/news/2026/06/zero-version-1/, https://zero.rocicorp.dev/docs/release-notes/1.0 (snippets)
- Now `@rocicorp/zero` **1.9.0** (Apache-2.0, npm 2026-08-14).
- **Model:** SA. **Custom mutators** run optimistically on the client and authoritatively on your server, and can run arbitrary validation and auth. Synced queries carry permissions. The client store is IndexedDB with ZQL/IVM (incremental view maintenance).
- **Not a fit for native or offline:** the Zero docs say "**Zero doesn't support offline writes**" and list "You are building a native mobile app" under when *not* to use it. Source: rocicorp/zero-docs `when-to-use.mdx` (GitHub, accessed 2026-10-02).
- **Governance signal:** the rocicorp/mono README says tests for zero-cache, zql and zqlite "were removed from the published repository on **September 30, 2026**" and that history was rewritten. The published repo is now a mirror of an internal repo. https://github.com/rocicorp/mono README (accessed 2026-10-02)

### 2.4 Replicache
- `replicache` 15.3.0 (npm 2025-07-02). The licence field points to https://roci.dev/terms.html. Rocicorp's focus is Zero; Replicache is effectively in maintenance. **Whether it was fully open-sourced is UNVERIFIED this session.**
- Its **push/pull model is still the template** for a hand-built op-log: named mutators, a server-assigned version, client rebase of pending mutations onto the server state.

### 2.5 LiveStore
- Event-sourcing on reactive SQLite. Two local DBs: an **event log** and a **materialised state DB**. Sync uses git-like push and rebase of events. https://docs.livestore.dev/evaluation/how-livestore-works/ (snippet)
- **0.4.0 on 2026-06-02**; still pre-1.0 (breaking changes in minors). Apache-2.0 (npm). It has a Cloudflare Durable Objects sync provider (`@livestore/sync-cf`). It was built for the Overtone music app. https://docs.livestore.dev/changelog/ (snippet)
- **Fit:** conceptually the closest to an append-only domain op-log, but it is JS/TS-only (web, Expo, Node) with no Swift, and no Postgres source.

### 2.6 Jazz (Garden Computing)
- The README says "**this is the Jazz 2.0 alpha with an entirely new API**" (Classic Jazz docs are separate). It is now a "local-first relational database" with a Rust core and RocksDB server. https://github.com/garden-co/jazz README (accessed 2026-10-02)
- `jazz-tools` 0.20.19 (MIT, 2026-07-03). The server is self-hostable. Jazz Cloud pricing is compute at $0.039/h for 2 GB RAM (1 GB included) and storage at $0.45/GB-month (1 GB included) (snippet, https://jazz.tools/).
- **E2EE** is a core design goal, but issue #3125 (2026-09-18) is titled "Track E2EE release qualification…", so **E2EE in 2.0 is not yet qualified** (snippet). It would replace Postgres entirely.

### 2.7 Triplit
- **Acquired by Supabase on 2025-10-08.** The founder joined to work on integrations, and the company pledged to "open source everything". https://supabase.com/blog/triplit-joins-supabase, https://x.com/kiwicopple/status/1976017503823491220 (snippets)
- `@triplit/client` 1.0.50, **AGPL-3.0**, last published **2025-07-31**, so effectively frozen. **Do not adopt.**

### 2.8 Evolu
- A TypeScript local-first platform: SQLite plus a CRDT with HLC timestamps. **E2EE by default**. Identity is a mnemonic-derived owner (SLIP-21 → OwnerId, EncryptionKey, WriteKey), and you self-host the relay with `npx @evolu/relay`. https://www.evolu.dev/docs/local-first, https://deepwiki.com/evoluhq/evolu (snippets)
- `@evolu/common` 8.15.1 (MIT, 2026-10-01): active.
- The relay is blind, so it has **no server-side invariants, no server AI and no Postgres**. iOS and Android work through the React Native/Expo examples.

### 2.9 InstantDB
- Open source; self-host from about $30/month. Offline cache plus a persistent mutation outbox. JS/React/React Native SDKs. `@instantdb/core` 1.0.67 (Apache-2.0, 2026-08-31). https://www.instantdb.com/docs/self-hosting (snippet)
- **UNVERIFIED and important:** a search snippet said "Instant is sunsetting, with services continuing until August 31st, 2027". It was attributed to instantdb.com docs, but the GitHub README (read 2026-10-02) does not mention it and a second search found nothing. **Check before relying on Instant.** Either way it is a Firebase-style triple store and would replace Postgres.

### 2.10 Convex
- `convex` 1.46.0 (Apache-2.0; the backend is open source and self-hostable). It has optimistic updates but **no first-party offline write queue**.
- Offline options: **PowerSync's Convex source** (experimental, announced 2026-06-10, https://releases.powersync.com/announcements/announcing-convex-backend-support-experimental, snippet) or community "replicate" with Yjs. Convex has `prosemirror-sync` for server-authorised collaborative editing (https://github.com/get-convex/prosemirror-sync).

### 2.11 Ditto
- A proprietary CRDT engine with P2P mesh (Bluetooth, Wi-Fi) and an optional cloud ("Big Peer"). SDKs for Swift, Kotlin, JS, Rust and more.
- Pricing per aggregators: free tier of 10 cloud connections and 2 GB storage; Pro 1,000 connections. **UNVERIFIED**; https://ditto.live/pricing/cloud-sync not opened.
- Closed source and enterprise-oriented. **Poor fit** for a $0 hobby project with a sellable future.

### 2.12 cr-sqlite (vlcn)
- A CRDT SQLite extension (multi-master, causal event logs). The last npm release was `@vlcn.io/crsqlite-wasm` 0.16.0 on **2023-12-16**, so it is **dormant**. https://github.com/vlcn-io/cr-sqlite. Do not adopt.

### 2.13 SQLite Sync (sqlite.ai)
- A CRDT SQLite extension that syncs to SQLite Cloud, **PostgreSQL** or **self-hosted Supabase**. It has a "Block-Level LWW" mode for markdown. `@sqliteai/sqlite-sync` 1.2.0 (2026-09-28). https://github.com/sqliteai/sqlite-sync README
- **Licence:** **Elastic License 2.0 (modified)**. Free only inside OSI-licensed open-source projects, and *you may not modify or replace the network layer*. https://github.com/sqliteai/sqlite-sync/blob/main/LICENSE.md (read 2026-10-02). **That blocks a closed commercial product**; licensing it needs a deal.

### 2.14 Turso (embedded replicas, Turso Sync)
- **Offline Sync** public beta on 2025-03-31. The newer "Turso Sync" uses CDC: local changes are pushed as logical row mutations, and remote changes are pulled as physical pages, so the local copy becomes byte-identical to the remote. Turso claims syncs up to 312× faster than embedded replicas. https://turso.tech/blog/turso-offline-sync-public-beta, https://turso.tech/blog/sync-benchmark (snippets)
- `@tursodatabase/sync` 0.8.1 (MIT, 2026-09-29).
- **News today:** "Supabase Announces $150M in New Funding and **Turso Acquisition**", dated **2026-10-02**. It is framed around "agentic workloads"; the Turso founders join as "Head of Agentic Services". http://www.prnewswire.com/news-releases/supabase-announces-150m-in-new-funding-and-turso-acquisition-302896752.html (snippet; the press release itself could not be opened).
  - **Implication (speculative):** Supabase may gain a first-party SQLite edge/offline story. Watch this before committing, but don't plan around it.
- Turso is SQLite-to-SQLite, not Postgres. Its conflict handling is limited (row-level).

### 2.15 Supabase's own offline story
- **There is no built-in offline sync.** Supabase points to partners: PowerSync, RxDB's Supabase replication plugin, SQLite Sync, and WatermelonDB (https://supabase.com/blog/react-native-offline-first-watermelon-db). Discussion: https://github.com/orgs/supabase/discussions/357 (snippets).
- Now owns Triplit (2025) and Turso (announced 2026-10-02). No product announcement on offline sync was found.

### 2.16 Summary of engines

| Engine | Status (2026-10) | Lic | Self-host | Postgres source | Native Swift | Offline writes | Conflict model | E2EE |
|---|---|---|---|---|---|---|---|---|
| PowerSync | GA (Swift/JS/Kotlin GA) | FSL→Apache (svc), Apache (SDK) | yes (free) | **yes** | **GA** | **yes (upload queue)** | SA, your backend; LWW default | app-level |
| Electric + TanStack DB | 1.x GA | Apache-2.0 | yes | **yes** | no | app-managed | SA, your API | app-level |
| Zero | 1.9 | Apache-2.0 (tests now private) | yes | **yes** | no | **no** | SA custom mutators | no |
| LiveStore | 0.4 | Apache-2.0 | yes (CF DO) | no | no | yes | event rebase | possible |
| Jazz 2.0 | alpha | MIT | yes | no | no (RN scaffold) | yes | CRDT | goal, not qualified |
| Evolu | 8.x active | MIT | relay | no | via RN | yes | CRDT/HLC LWW | **yes, built in** |
| Automerge-repo | 3.5 core / repo alpha | MIT | toy server | no | yes (swift) | yes | CRDT | possible (blind relay) |
| Turso Sync | beta | MIT | partial | no (SQLite) | ? | yes | row-level | no |
| SQLite Sync | 1.2 | ELv2-modified | via PG | yes | yes (ext) | yes | CRDT | no |
| Triplit | frozen | AGPL | – | – | – | – | – | – |
| cr-sqlite | dormant (2023) | – | – | – | – | – | – | – |
| Instant | 1.0 (**sunset? unverified**) | Apache | yes | no | no | yes | SA | no |
| Ditto | commercial | proprietary | enterprise | no | yes | yes | CRDT | ? |

---

## 3. Op-log / event-sourcing approaches

### 3.1 Actual Budget: CRDT over SQLite, Merkle trie, E2EE (open source, read from source)
- **Message** (protobuf, `packages/crdt/src/proto/sync.proto`): `{dataset, row, column, value}`. It is wrapped in `MessageEnvelope {timestamp, isEncrypted, content}`, with `EncryptedData {iv, authTag, data}`. https://github.com/actualbudget/actual/blob/master/packages/crdt/src/proto/sync.proto (accessed 2026-10-02)
- **Clock:** a Hybrid Unique Logical Clock that serialises into a 46-character sortable string (`2015-04-24T22:23:42.123Z-1000-<nodeid>`) and is based on the HLC paper. (`packages/crdt/src/crdt/timestamp.ts`)
- **Merkle:** a ternary radix trie keyed by minute-resolution timestamps. Each node's hash is the XOR of murmurhashes, so `diff()` finds the earliest divergent time window and the client re-sends from there. A code comment admits a known weakness: if nothing matches, it falls back to the front window. (`merkle.ts`)
- **Conflict rule:** **per-cell LWW**. `compareMessages` looks up `messages_crdt` for each `(dataset,row,column)` and only applies a message if its timestamp is newer. (`packages/loot-core/src/server/sync/index.ts`)
- **Server role:** the sync server is a **dumb relay and store**. It stores envelopes per file and group, returns messages since a timestamp plus its Merkle hash, and never interprets values. With E2EE the payloads are AES-GCM, using a key derived from a user password (`createKey({password, salt})`). (`packages/sync-server/src/app-sync.ts`, `packages/loot-core/src/server/encryption/index.ts`)
- **Invariants:** **none enforced by sync.** Cross-row consistency (account exists, budget totals) is recomputed locally from the merged cells. The Merkle comment includes a TODO about checking whether an account exists. Original design essay: James Long, "Using CRDTs in the Wild", https://archive.jlongster.com/using-crdts-in-the-wild (could not open; snippet).
- **Lesson for Sprint:** this is the simplest E2EE-capable design with a blind server, and it suits single-user multi-device. The price is that every invariant becomes "derived on read" or "repaired after merge".

### 3.2 Linear sync engine (SA, total order)
- Clients make optimistic changes to in-memory models. They are packaged as **transactions** (create/update/delete/archive) and stored in an IndexedDB `__transactions` table while offline. They are sent in batches and **executed on the server**, which may add side effects, and they are **reversible on the client** if they fail.
- The server assigns a global, monotonically increasing **`syncId`** (a **total order**, so closer to OT than to CRDTs). It broadcasts **delta packets** of sync actions to every client, including the sender. Clients track `lastSyncId` and `firstSyncId`; bootstrap is full or partial, with lazy hydration. **Sync groups** carry permissions.
- Sources: https://github.com/wzhudev/reverse-linear-sync-engine (README read 2026-10-02); Tuomas Artman talks https://www.youtube.com/watch?v=Vk15EYX6C8g; https://linear.app/now/scaling-the-linear-sync-engine (snippets).
- **Invariants:** enforced on the **server** at transaction execution. A rejected transaction is rolled back on the client. Conflicts default to last write wins *in server order*. Text (issue descriptions) uses a separate Yjs-style CRDT (**not re-verified**).
- **Lesson:** this is the model that matches Sprint's current Postgres-enforced invariants and RLS. The existing `@sprint/db` functions become the "transaction executors".

### 3.3 Things Cloud
- Proprietary. Conflict logic is "inspired by version control systems… simultaneous edits merge to a shared consensus". Rewritten in **server-side Swift** (in production since early 2024; blog 2025-05). Devices are told to pull via local-network broadcast and push. https://culturedcode.com/things/blog/2025/05/a-swift-cloud/ (snippet), https://culturedcode.com/things/cloud/
- Not E2EE (to my knowledge; **UNVERIFIED**). It is a custom SA service, not CloudKit.

### 3.4 Bear 2 (CloudKit)
- Bear syncs through CloudKit and supports per-note encryption. **Not re-verified this session.** It is the canonical "Apple-only, no web app" case.

### 3.5 Obsidian Sync
- E2EE by default: AES-256-GCM, with the key from PBKDF2-SHA-512 (650k iterations) plus HKDF. Optionally Obsidian holds the key.
- Markdown conflicts use a **diff-match-patch three-way merge**. Users report occasional lost or duplicated text. https://forum.obsidian.md/t/robust-sync-conflict-resolution/93544, https://deepwiki.com/obsidianmd/obsidian-help/2.3-synchronization-and-conflict-resolution (snippets)
- **Lesson:** file-level merging is the weakest option for structured data.

### 3.6 Takeaway for an op-log design in Sprint
- **Log entries should be *domain operations*** (`task.move_to_sprint`, `sprint.close`, `edge.add(part_of)`), not raw row diffs. Each carries a client id, a client sequence number, an HLC and a base `syncId`.
- **The server applies them** in one Postgres transaction through the existing `@sprint/db` functions, under RLS and inside `withUser()`. It assigns a `syncId` and appends the result to the event log, which the app already has as a concept.
- **Clients** keep a local SQLite projection and **rebase** their pending ops onto server deltas. A rejected op is rolled back and surfaced to the user, e.g. "this page can't move into its own child".
- **Page bodies** are a per-page Yjs document. Updates are appended as binary rows and compacted on the server, which can decode Yjs to derive `body_text` for search and embeddings.
- Replicache's push/pull protocol and Linear's design are the blueprints. PowerSync gives you the read replication and the client upload queue for free, so you write only the server "executor".

---

## 4. CloudKit / iCloud (CKSyncEngine)

- **CKSyncEngine:** iOS/iPadOS/tvOS 17+, **macOS 14+**. It manages push/pull scheduling, change tokens and retries. You persist its opaque state. Batches are limited to **250 records per request** (saves plus deletes). Apple says not to use it for the public DB, and the sync schedule depends on system conditions. Apple docs JSON https://developer.apple.com/documentation/cloudkit/cksyncengine-5sie5 (read 2026-10-02).
  - Forum thread on API design and maintenance concerns: https://developer.apple.com/forums/thread/771941 (snippet).
- **Cost:** private-database usage counts against the **user's** iCloud quota, and developers are not billed for it. Per-user limits are cited as 10 GB assets, 100 MB data, 2 GB transfer and 40 requests/s. That figure is **old and UNVERIFIED**; it comes from forum threads https://developer.apple.com/forums/thread/665612 and the rambo.codes "CloudKit 101" post from 2020. Public DB: "up to 1PB" (https://developer.apple.com/icloud/cloudkit/).
- **E2EE:** `CKRecord.encryptedValues` (iOS 15/macOS 12+) encrypts on-device. With **Advanced Data Protection** the keys belong only to the owner and share participants. Encrypted fields **cannot be indexed or queried**. Apple docs JSON `cloudkit/ckrecord/encryptedvalues` (read 2026-10-02).
- **Web:** CloudKit JS (`cdn.apple-CloudKit.com/ck/2/CloudKit.js`) can reach public and private DBs with Apple ID sign-in (https://developer.apple.com/documentation/cloudkitjs).
  - However, **encrypted fields are not available through CloudKit JS or Web Services** (forum https://developer.apple.com/forums/thread/82810 and Tact blog https://blog.justtact.com/advanced-data-protection/, snippets; **partly verified**: Apple's docs say values are decrypted only on-device).
  - ADP users may also have iCloud web access turned off.
- **Interaction with the web app and multi-tenant selling:**
  - CloudKit would make Apple the system of record. The Next.js web app could only use CloudKit JS: it would need Apple ID sign-in, could not read E2EE fields, and could not run server jobs (Inngest, AI, embeddings) on private data without a device relay.
  - Android, non-Apple users and team workspaces are excluded. RLS, Postgres FTS and pgvector cannot run on CloudKit data.
  - **Verdict:** at most an *optional extra transport* (e.g. a device-to-device backup). Not the sync backbone.

---

## 5. End-to-end encryption alongside a web app

- **Feasible pattern:** like Proton, Bitwarden and Standard Notes, the web app derives a key client-side and the server stores ciphertext.
- **Passkey-PRF status:**
  - **Chrome/Chromium:** supported. Intent to Ship: https://groups.google.com/a/chromium.org/g/blink-dev/c/iTNOgLwD2bI. The exact version was **not re-verified**; it shipped around 2023–24. A 2026 article mentions Chrome 147 adding PRF at credential creation (https://www.corbado.com/blog/passkeys-prf-webauthn, snippet).
  - **Safari 18 / iOS 18 / macOS 15:** PRF works with iCloud Keychain passkeys. Over hybrid/QR it was broken in 18.0.1 and partly fixed in 18.2, but returns *different* PRF output over hybrid than on-device. iOS does **not** pass PRF to external security keys. https://developer.apple.com/forums/thread/774112, https://developers.yubico.com/WebAuthn/Concepts/PRF_Extension/Developers_Guide_to_PRF.html (snippets)
  - **Firefox:** baseline support in 135 (Windows), macOS platform authenticator in 139, Windows Hello creation and authentication fixed in 148, Android targeted for 149. https://bugzilla.mozilla.org/show_bug.cgi?id=1863819 (snippet)
  - **Production proof:** Bitwarden supports passkey login with PRF, and since server release **2026.1.1** it can *unlock the vault* with a PRF passkey. https://bitwarden.com/help/login-with-passkeys/, https://community.bitwarden.com/t/you-can-now-unlock-your-vault-with-a-passkey/93556 (snippets)
- **Design if wanted:**
  - Wrap a random data key (DEK) with several key-encryption keys: one per passkey via PRF → HKDF, plus a recovery phrase or password.
  - **Do not** derive the data key directly from a single passkey's PRF output: passkeys get lost, and hybrid PRF output differs.
- **Cost to Sprint (the real decision):** with E2EE the server cannot
  1. compute embeddings or call Claude on content. The AI could move client-side, with the client sending plaintext to Claude itself, which breaks "server never sees plaintext" anyway;
  2. run Postgres FTS (`body_text`) or pgvector;
  3. enforce content invariants. It can still enforce structural ones if edges and IDs stay plaintext;
  4. run cron rituals (sprint close, rollover) on content. Metadata-only rituals still work.

  Hybrid options: E2EE only for opt-in "private" pages or fields, or E2EE with server AI disabled for those items.
- **Anytype** (any-sync, E2EE, CRDT, self-hostable) and **Standard Notes** are reference designs; their details were **not re-verified** this session (search budget ran out).

---

## 6. Rich-text merging for block-editor documents

- **BlockNote** (MPL-2.0 core; `@blocknote/xl-ai` is GPL-3.0 or proprietary):
  - Collaboration is **Yjs-only**: `withCollaboration({collaboration: {provider, fragment: doc.getXmlFragment(...)}})`. The documented providers are Liveblocks, PartyKit, Y-Sweet, Hocuspocus, y-websocket, y-indexeddb, y-webrtc, Matrix and Nostr. Source: BlockNote repo `docs/content/docs/features/collaboration/index.mdx` (read 2026-10-02).
  - `@blocknote/core/yjs` has converters between blocks and Y.Doc / Y.XmlFragment (https://www.blocknotejs.org/docs/reference/editor/yjs-utilities).
  - Liveblocks also offers its own "LiveText" engine for BlockNote (`@liveblocks/react-blocknote` 3.24.3, 2026-10-01).
  - An npm search found **no Loro or Automerge BlockNote binding**. Using Loro or Automerge would mean writing your own BlockNote collaboration extension on top of loro-prosemirror or @automerge/prosemirror. BlockNote's block schema (nested blocks, block ids, props) must round-trip, which is non-trivial.
- **How the merges behave** (from the docs above plus general CRDT knowledge, not re-tested):
  - **Yjs (y-prosemirror):** the ProseMirror tree maps to Y.XmlFragment / XmlElement / XmlText. Concurrent inserts interleave by YATA order. Block attributes are LWW per key. A block "move" is delete plus insert: a concurrent edit to a moved block can be lost or duplicated, and nested-block moves are the classic weak spot. v14 does not add move.
  - **Loro (loro-prosemirror):** text uses Fugue, which minimises interleaving. Marks use Peritext-style expand semantics. The ProseMirror tree maps to Loro containers. Loro has a movable tree and list, so block moves *could* be true moves, but **whether loro-prosemirror uses them for block moves is UNVERIFIED**.
  - **Automerge:** text with marks plus *block markers* inline in the text, giving a flat representation of structure. Nested structures go through the SchemaAdapter.
- **For single-user multi-device** the concurrency is mostly "same person edited the same page on two offline devices". Any of the three merges that acceptably well. **Yjs wins on integration cost** because BlockNote already speaks it.
- **Storage:** store a Yjs update log per page (append updates, compact periodically into a state snapshot). Derive `objects.body` (BlockNote JSON) and `body_text` on the server by decoding Yjs in Node. The server never needs to merge-edit, only to read.

---

## 7. Scored comparison (1 = poor, 5 = best; "lock-in" 5 = least lock-in)

Scores are judgement calls for *this* app: single user, multi-device, Mac then phone, the web app stays, Postgres+RLS today, multi-tenant later, about $0 cost.

| Option | Perf | Security | Maturity | Dev speed | Domain fit | Cost | Lock-in | Notes |
|---|---|---|---|---|---|---|---|---|
| **PowerSync + own write API (+ Yjs page bodies)** | 4 | 4 | 4 | 4 | **5** | 4 | 3 | Postgres stays the source of truth; invariants stay in DB and backend; Swift GA; FSL self-host |
| **Hand-built op-log (Linear/Replicache style) on Supabase (+ Yjs)** | 4 | 4 | 3 | 2 | **5** | 5 | **5** | Most control and fit; most work (client store, rebase, bootstrap, Swift client) |
| Electric + TanStack DB (+ Yjs) | 4 | 4 | 4 | 4 | 3 | 5 | 4 | Excellent for web; no Swift client; offline write queue is DIY |
| Zero | 5 | 4 | 3 | 4 | 2 | 4 | 3 | No offline writes, no native; governance shift (tests private) |
| Automerge 3 + automerge-repo (own server) | 3 | 4 | 3 | 3 | 2 | 5 | 4 | Document-centric; invariants and server AI need a Postgres projection; repo is alpha |
| Loro (+ own sync) | 5 | 4 | 3 | 2 | 3 | 5 | 4 | Best tree semantics; no BlockNote; you build all transport and projection |
| Evolu | 4 | **5** | 3 | 4 | 2 | 5 | 3 | E2EE built in; blind relay means no server AI, FTS or invariants; no Postgres |
| LiveStore | 4 | 3 | 2 | 3 | 3 | 5 | 3 | Event-sourcing fits the model; pre-1.0; no Swift or Postgres |
| Jazz 2.0 | 4 | 4 | 1 | 3 | 2 | 4 | 2 | Alpha rewrite |
| CloudKit / CKSyncEngine | 4 | **5** | 4 | 3 | 1 | **5** | 1 | Free, ADP E2EE; breaks the web app, AI, RLS and multi-tenant |
| Turso Sync / SQLite Sync | 4 | 3 | 2 | 3 | 2 | 3 | 2 | SQLite-centric; SQLite Sync licence blocks closed commercial use; Turso is being acquired |

---

## 8. Recommendation

**Keep Postgres (Supabase) as the authority and add a server-authoritative sync layer: PowerSync first, with a hand-built op-log as the fallback. Use Yjs only for page bodies. Do not adopt E2EE or CloudKit as the base.**

Concretely:
1. **Clients** (Mac, phone, and optionally the web app) hold a SQLite replica via PowerSync Sync Streams, scoped per workspace. That fits the existing `workspace` column and RLS mindset.
2. **Writes are named domain mutations**, not row CRUD. Put them in a local `pending_mutations` table that PowerSync's upload queue sends to a Next.js route. The route runs the existing `@sprint/db` functions inside `withUser()`, so DB constraints and triggers (0013, 0022, 0012, 0016) keep enforcing invariants. A rejection reverts the optimistic state.
3. **Make invariants merge-friendly anyway**, so that offline conflicts are rare:
   - deterministic IDs for singletons (journal per day, link per canonical URL);
   - single-valued fields for cardinality-1 relations (`parent_id`, `project_id`);
   - sprint close and rollover run only on the server (cron/Inngest) or as a single server mutation.
4. **Page bodies** are a Yjs doc per page, synced as an append-only update table (`page_updates`) through the same channel. The server decodes and compacts them and derives `body`, `body_text` and embeddings. BlockNote is already moving to Yjs 14 (`@y/y` rc), so plan for v14.
5. **Security** relies on TLS, Supabase at-rest encryption, RLS and local SQLite encryption (SQLCipher on device). Optional later: client-side encrypted "vault" fields, using a passkey-PRF-wrapped data key. Those fields are excluded from AI and search.

**What would change this recommendation:**
- **E2EE becomes a hard requirement** (e.g. as a selling point) → Actual/Evolu-style blind relay with per-cell LWW plus Yjs, client-side AI and search, and a passkey-PRF key hierarchy. Postgres becomes a ciphertext store, and server-enforced invariants go away.
- **Apple-only and no web app** → CloudKit with CKSyncEngine (free, ADP) becomes attractive.
- **PowerSync's FSL licence or pricing becomes a problem for the paid product** (FSL forbids a *competing sync service*, not ordinary SaaS use, but get that confirmed) → the hand-built op-log (Replicache protocol), or Electric for reads plus your own upload queue.
- **Supabase ships first-party offline sync** (it now owns Triplit's know-how and Turso as of 2026-10-02) → re-evaluate; it would likely be the lowest-friction path.
- **Real-time multi-user co-editing comes into scope** → still Yjs (Hocuspocus or Y-Sweet for live sessions), so it does not change the core choice.
- **The page tree becomes the core UX with heavy offline re-parenting** → consider Loro's Movable Tree for the tree only. It is rarely worth the cost for one user.

---

## 9. What I could not verify (follow up before the ADR is final)
- PowerSync pricing page (only a G2 snippet), the exact FSL→Apache conversion period, and named production users.
- Whether **InstantDB is sunsetting** (a snippet claimed services until 2027-08-31; not confirmed).
- Details of the Supabase–Turso acquisition (press release dated 2026-10-02, not opened).
- Current per-user CloudKit quotas (numbers are from old forum and blog posts).
- Exactly which Chrome version shipped PRF; Chrome's PRF-on-create in 147.
- Whether Replicache is fully open source now.
- Whether loro-prosemirror uses movable-list moves for blocks.
- Status of the Yjs Swift bindings (y-uniffi / yswift).
- Anytype, Standard Notes and Things E2EE details; Bear's CloudKit encryption specifics.
- Jazz 2.0 E2EE qualification outcome (issue #3125).
- That automerge-swift upstream has moved to core 0.12 (only a fork PR was seen).
- No benchmark covers Automerge 3.x against Loro 1.x and Yjs 13 head-to-head. The published tables are for older versions.
