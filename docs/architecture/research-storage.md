# Research: the local storage engine

- **Status:** research note for ADR-0001 (desk research; benchmarks are run separately)
- **Date:** 2026-10-02. All links accessed 2026-10-02 unless a date is given.
- **Question:** what database holds the graph (objects, relations, events, ontology, chunks) on the Mac, later on the phone and web, and on the optional hub? And where does integrity, encryption and schema evolution live?

Note on sources: sqlite.org, pglite.dev, turso.tech and a few blogs were blocked from this sandbox, so some facts about them come from search-result snippets or from the projects' GitHub repos. Those are marked *(snippet)*. Anything I could not confirm is marked **unverified**.

## 1. What the workload needs

From Sprint's ADR-0001, ADR-0004, ADR-0005 and `context-graph.md`:

- A universal `objects` table plus typed `relations` (with provenance and validity), typed extension tables, an append-only `events` log, ontology tables (types, relation types, fields), and `object_chunks` with 512-dim embeddings.
- Strict, fast reads for Home and sprint metrics (dates, statuses, counts); 1–3 hop traversals (recursive CTEs).
- Full-text search (Polish and English text) and search by meaning over thousands to tens of thousands of chunks per user.
- Integrity rules that Sprint enforces in Postgres today: FKs to the ontology, `check_relation()` (from/to types, cardinality under advisory locks), `check_object_update()` (conversions), deferred `check_object_fields()`, limits (30 fields/types), event triggers, RLS for tenancy.
- One user, one device writing at a time in practice; a background worker (embeddings, AI agent) writing too.
- Runs on macOS first, then iOS and web; the same engine (or at least the same schema) on an optional hub that later may serve many tenants.

Scale for one person: roughly 10^4–10^5 objects, 10^5–10^6 events, 10^4–10^5 chunks. Every candidate below is fast enough at this size; the choice is about correctness, portability, durability and integrity, not raw speed.

## 2. SQLite and its ecosystem

### 2.1 Core engine

- **Version.** Latest release 3.53.4 (2026-07-24); 3.54.0 is in draft, dated 2026-10-15 *(snippet, https://sqlite.org/ and https://www.sqlite.org/draft/releaselog/3_54_0.html)*.
- **3.53.0 (2026-04-09)** added `ALTER TABLE` to add and remove `NOT NULL` and `CHECK` constraints, `REINDEX EXPRESSIONS`, and `json_array_insert()`/`jsonb_array_insert()` *(snippet, https://daily.dev/posts/sqlite-release-3-53-0-on-2026-04-09-qfesmnxqk, https://sqlite.org/releaselog/3_53_1.html)*. This removes one of the most common reasons for the 12-step rebuild.
- **The WAL-reset bug.** A data race between a checkpoint and a concurrent WAL reset could silently and permanently corrupt a database. It was in every release from 3.7.0 (2010) to 3.51.2, needs WAL mode plus two or more connections (threads or processes) writing or checkpointing at the same instant, and is fixed in 3.51.3 (2026-03-13), with backports to 3.50.7 and 3.44.6. Tailscale found it after 19 production corruptions (https://mjtsai.com/blog/2026/08/14/sqlite-wal-reset-bug/, https://theconsensus.dev/p/2026/08/23/another-look-at-sqlite-wal-reset.html). Lessons for us: **bundle our own SQLite (≥ 3.53.x)**, never rely on the OS copy, and keep the number of writing connections to one.
- **Features we need, all long stable:** WAL (one writer, many readers, readers never block the writer); `STRICT` tables (3.37, 2021) so a column really holds its declared type; JSON functions and binary **JSONB** (3.45, 2024) for `properties`; generated columns (3.31) to pull hot JSON fields into indexed columns; triggers; `CHECK`; foreign keys (off by default, `PRAGMA foreign_keys=ON` per connection; `DEFERRABLE INITIALLY DEFERRED` FKs are supported); recursive CTEs; partial and expression indexes; `RETURNING`; window functions; the session extension (changesets) for sync experiments.
- **Limits that matter:** one writer at a time per file (`BEGIN IMMEDIATE` serializes; fine for one user, plan a single writer task); no stored procedures or plpgsql; triggers are SQL only and fail with `RAISE(ABORT, '...')`; `CHECK` cannot run subqueries; no deferred triggers or deferred `CHECK` (only FKs defer); no RLS (tenancy is the file boundary); no `ALTER COLUMN TYPE`.

### 2.2 Full-text search (FTS5)

- `unicode61` with `remove_diacritics 2` folds "zażółć" to "zazolc" and handles Polish case folding. It does no stemming, so "zadania" does not match "zadanie".
- `trigram` (3.34+) gives substring and `LIKE`/`GLOB` matches, and partly hides the lack of stemming for a highly inflected language, at roughly 3x the index size (**unverified** multiplier; the benchmark agent should measure).
- **Polish stemming:** FTS5's built-in `porter` is English only. `fts5-snowball` wraps Snowball's libstemmer as an FTS5 tokenizer (https://github.com/abiliojr/fts5-snowball). Snowball only gained a Polish algorithm in October 2025, after Snowball 3.0 *(snippet, https://snowballstem.org/algorithms/)*, so we would have to build fts5-snowball against a current libstemmer ourselves (**unverified** that it builds cleanly). Other options: a custom tokenizer in Rust (FTS5 tokenizers are a C API; rusqlite exposes enough to register one, **unverified** ergonomics), or no stemming plus trigram plus search by meaning, which covers most of what stemming buys.
- Postgres is not better here: core Postgres has no Polish snowball config either; it needs an ispell dictionary.

### 2.3 Vector search

| Option | Status (2026-10) | Index | Notes |
|---|---|---|---|
| **sqlite-vec** (asg017) | Stable v0.1.9 (2026-03-31); ANN only in v0.1.10-alpha.4 (2026-05-18): DiskANN, "rescore", IVF (disabled) (https://github.com/asg017/sqlite-vec/releases) | Brute force in stable; ANN alpha | Pure C, no deps, runs everywhere incl. wasm and iOS. float32, int8, bit vectors (README). Mozilla Builders sponsored; development paused, then resumed "with Mozilla's support" in v0.1.7. 159 open issues; docs lag. Single maintainer. |
| **SQLite `vec1`** (by the SQLite team) | Pre-1.0, "version 0.7", "no further features required before 1.0" *(snippet, https://sqlite.org/vec1, forum post)* | IVFADC + OPQ (ANN), AVX2/NEON | Official, portable C. Worth watching: if it ships 1.0 it becomes the default choice. Accuracy/recall at our scale **unverified**. |
| **libSQL native vectors** | In libSQL (fork of C SQLite). `F32_BLOB(n)`, `F16_BLOB`, `F8_BLOB`, `F1BIT_BLOB`; `libsql_vector_idx` (DiskANN); `vector_top_k()` *(snippet, https://docs.turso.tech/features/ai-and-embeddings)* | DiskANN | libSQL README now says: "If you're starting a new project, you probably want to look into Turso … new features are being developed in Turso" (https://github.com/tursodatabase/libsql). So it is maintenance-track. |
| **Turso Database** (Rust rewrite) | Vector indexing is on the roadmap, not shipped as an index (https://github.com/tursodatabase/turso) | Scalar distance functions | See 2.4. |
| **sqlite-vss** | Deprecated by its author in favour of sqlite-vec | Faiss | Do not use. |
| **vectorlite** | Beta, "could be breaking changes"; hnswlib-based; prebuilt for macOS/Linux/Windows x64/arm64 (https://github.com/1yefuwang1/vectorlite) | HNSW (in memory) | C++; no iOS/wasm builds published. |
| **USearch SQLite extension** | Distance functions only, no index; past macOS arm64 segfault and missing-binary issues (https://github.com/unum-cloud/usearch/issues/371) | none | Not a vector store. |

**What we actually need.** Sprint's ADR-0004 already decided "no vector index until ~50,000 chunks; exact scan is fast and always correct". The same holds locally: 50,000 × 512 float32 = 100 MB to scan; at memory bandwidth of 10+ GB/s with SIMD that is on the order of 10 ms, half that with float16 or int8 (my estimate; the benchmark agent should confirm). So: store embeddings as a plain `BLOB` column in a normal table (`object_chunks`), compute distance with sqlite-vec's scalar `vec_distance_cosine()` (stable) or a few lines of SIMD Rust, and keep the option of an ANN index later (sqlite-vec DiskANN, `vec1`, or an in-memory HNSW built at start-up with the `usearch` or `hnsw_rs` crates). A plain BLOB column keeps the format open and the vectors under the same FKs and cascades as everything else, which a `vec0` virtual table does not (virtual tables cannot be FK targets or have triggers of their own).

### 2.4 Turso Database (formerly "Limbo")

- Rust rewrite of SQLite; MIT. Latest 0.8.1 (2026-09-29), 0.8.2-pre.2 on 2026-10-02; not 1.0 (https://github.com/tursodatabase/turso/releases).
- README: compatible with SQLite's SQL dialect, file format and C API (tracking 3.50.4); "existing SQLite database files work as-is"; full compatibility is a 1.0 requirement; used in production by Turso Cloud, Kin and Spice.ai (https://github.com/tursodatabase/turso).
- `BEGIN CONCURRENT` with MVCC (Hekaton-style). 0.8.0 release notes "stop calling MVCC experimental". In 0.5 the MVCC mode could not have indexes and loaded the whole database into memory at first access *(snippet, https://betterstack.com/community/guides/databases/turso-explained/)*; whether that is still true in 0.8 is **unverified**.
- Experimental: encryption at rest (AEGIS-256, AES-GCM, page level, vendor-reported 6% read / 14% write overhead, *(snippet, https://turso.tech/blog/introducing-fast-native-encryption-in-turso-database)*), FTS via tantivy, Postgres dialect and wire protocol, CDC, multi-process WAL.
- **Verdict:** the most interesting thing to watch (async I/O, CDC for sync, encryption, concurrent writes), but pre-1.0 and fast-moving. Because it reads and writes SQLite files, choosing stock SQLite now keeps a cheap switch open later.

### 2.5 Bindings

| Binding | Status | Notes |
|---|---|---|
| **rusqlite** (Rust) | 0.40.2 (2026-08-08), bundles SQLite 3.53.2 *(snippet, https://crates.io/crates/rusqlite)* | Synchronous, thin, full API (functions, hooks, sessions, backup, blob I/O, load_extension). The best fit for a Rust core. Use the `bundled` feature (or `bundled-sqlcipher`). |
| **sqlx** (Rust) | 0.9.0 (2026-05-06); repo moved to `transact-rs` (https://github.com/launchbadge/sqlx/discussions/4271) | Async, compile-time checked queries, works for both SQLite and Postgres. SQLite driver runs each connection on a worker thread; fewer SQLite-specific hooks than rusqlite. Useful if the hub must speak Postgres too. |
| **better-sqlite3** (Node) | Mature, synchronous, native addon | Needs a native build per Electron/Node ABI. |
| **node:sqlite** | Built in; no flag since 22.13/23.4; Stability 1.2 "release candidate" since v25.7.0; has `loadExtension`, sessions/changesets, custom functions, backup (https://nodejs.org/api/sqlite.html, docs for v26.10.0) | Node 24.15+ also RC *(snippet)*. Good for a TS hub without native addons. |
| **bun:sqlite** | Built into Bun; vendor claims 3–6x faster than better-sqlite3, disputed (https://github.com/oven-sh/bun/issues/4776) | Ties the hub to Bun. |
| **GRDB** (Swift) | 7.x, 7.11.1 by June 2026; 7.10 added SQLCipher via SwiftPM plus Android/Linux/Windows *(snippet, https://forums.swift.org/t/grdb-v7-10-0-android-linux-windows-and-sqlcipher-swiftpm/84754)* | Best Swift SQLite library: WAL, `DatabasePool`, observation. Relevant if the iOS app is native Swift reading the same file format. |

## 3. PGlite (Postgres in WebAssembly)

- **Version:** `@electric-sql/pglite` 0.5.8 (package.json on `main`). 0.5.x is **Postgres 18.3** and cannot open data directories written by 0.4.x (PG 17), and PGlite has no `pg_upgrade` *(snippet, https://github.com/electric-sql/pglite/discussions/766)*. Still 0.x after 2.5 years.
- **Extensions:** pgvector is an external package (`@electric-sql/pglite-pgvector`) supporting "single-precision, half-precision, binary, and sparse vectors", so `halfvec` yes; HNSW and IVFFlat come with pgvector (https://github.com/electric-sql/pglite/blob/main/docs/extensions/extensions.data.ts). Also pg_trgm, unaccent, ltree, pgcrypto, pg_ivm, pg_uuidv7, Apache AGE, PostGIS, live queries. plpgsql is part of core Postgres, so triggers in plpgsql work.
- **Process model:** Postgres single-user mode, one connection, one thread. A background job and the UI share one connection and queue behind each other.
- **Durability by storage:**
  - *In-memory:* none.
  - *IndexedDB:* the whole database is loaded into memory at start, and changed files (one per table/index) are flushed whole to IndexedDB after each query; `relaxedDurability` returns before the flush (https://github.com/electric-sql/pglite/blob/main/docs/docs/filesystems.md). Memory use equals database size; write cost grows with table size.
  - *OPFS AHP:* Chrome and Firefox only, in a Worker; **does not work in Safari** (limit of 252 sync access handles vs 300+ Postgres files). So it does not work in WKWebView, which is what Tauri uses on macOS and iOS.
  - *NodeFS:* open issue #1107 (2026-09-14, no maintainer reply): Postgres is started with `-F` (fsync off) and NodeFS's fsync is a no-op, so a write acknowledged before an OS crash or power loss may be lost (https://github.com/electric-sql/pglite/issues/1107). Earlier reports: "PANIC: could not locate a valid checkpoint record" after restart (#327); multi-process corruption without a data-dir lock (PR #892); "stack depth limit exceeded" after ~1,800 failed statements (#1115).
- **Speed (vendor, M2 Air):** in memory, PGlite is 2–7x slower than wa-sqlite on bulk ops (25,000 inserts in a transaction: 0.29 s vs 0.08 s); single-row writes to IndexedDB are ~14–24 ms with full durability vs ~0.08 ms relaxed (https://github.com/electric-sql/pglite/blob/main/docs/benchmarks.md). Native SQLite vs native Postgres in the same doc: 25,000 indexed inserts 0.04 s vs 0.38 s.
- **Native build:** `libpglite` and `pglite-bindings` are WIP (Python via wasmtime, Go and Kotlin experiments) (https://github.com/electric-sql/pglite-bindings). A third-party `pglite-rs` links a native single-process Postgres fork into Rust *(snippet, https://lib.rs/crates/pglite-rs)*; maturity **unverified**.
- **Where it could run for us:** in a Tauri webview (yes, but IndexedDB only on WebKit, so whole DB in memory); in Node on a hub (yes, but see durability above); on iOS only inside a webview.
- **Verdict:** excellent for tests and for a browser demo; not a durable primary store for a personal OS in 2026. Its main attraction (same SQL as Postgres everywhere) is real, but it costs durability, concurrency and memory.

## 4. Other engines

| Engine | Facts | Fit |
|---|---|---|
| **Real Postgres embedded** (`postgresql_embedded` Rust crate, Postgres.app) | Downloads or bundles Postgres binaries (~10 MB compressed bundles) and runs a **separate server process** (https://github.com/theseus-rs/postgresql-embedded). | Works on macOS (sandboxed App Store builds can spawn helpers only with care). Impossible on iOS: apps may not fork or spawn processes, and Postgres is multi-process with shared memory (https://developer.apple.com/forums/thread/747499). ~50–100 MB RSS idle (**unverified**). Two engines across platforms. |
| **DuckDB** | 1.5.6 (2026-09-28), 1.4.5 LTS; 2.0 planned 2026-10-21 (https://duckdb.org/release_calendar) | Columnar OLAP; single-writer, weak at many small transactions. Good as an optional analytics reader of SQLite files, not the store. |
| **Kùzu** (embedded graph DB) | Repo archived 2025-10-10; company acquired by Apple (Oct 2025, reported Feb 2026) (https://uwaterloo.ca/computer-science/news/waterloo-based-graph-database-start-up-kuzu-acquired-apple, https://dbdb.io/db/kuzu). Community fork exists. | Dead upstream. Rule out. |
| **CozoDB** | Last release v0.7.6 (Dec 2024), no commits since; an open issue asks if it is maintained (https://github.com/cozodb/cozo) | Abandoned. Rule out. |
| **SurrealDB embedded** | BSL 1.1; 3.0's change date is 2030-01-01 to Apache-2.0 (https://github.com/surrealdb/surrealdb/blob/main/LICENSE). Embedding in your own app is allowed; offering it as a DB service is not. | Licence is workable but a risk for "hosted service later"; young storage engine; we'd lose SQL tooling. No. |
| **Realm** | MongoDB deprecated Atlas Device SDK and Device Sync; EOL 2025-09-30. The local DB stays open source but MongoDB no longer maintains it (https://www.mongodb.com/community/forums/t/device-sync-and-edge-server-are-deprecated/296035/45). | Rule out. |
| **Core Data / SwiftData** | Apple-only, SQLite underneath with a private schema. | No web/hub story; rule out as the store (fine for nothing here). |
| **KV engines** (LMDB, redb, fjall, sled) | redb: stable file format, "stable and maintained" (https://github.com/cberner/redb). fjall 3.0 (Jan 2026), LSM, forward-compatible format (https://fjall-rs.github.io/post/fjall-3/). sled: long-stalled beta. LMDB: rock-solid, C. | We would have to build indexes, queries, FTS, migrations and integrity ourselves. Only worth it for a CRDT op log side store, and SQLite does that too. |

## 5. Scoring

Scores 1–5 (5 best). "Fit" = fit to a typed graph with an ontology, FTS, vectors, metrics.

| Option | Perf | Security | Stability | Dev speed | Fit | Cost | Lock-in (5 = none) | Portability (Mac/iOS/web/hub) | Total /40 |
|---|---|---|---|---|---|---|---|---|---|
| **SQLite (stock, bundled ≥3.53) + FTS5 + BLOB vectors** | 5 | 4 | 5 | 4 | 4 | 5 | 5 | 5 | **37** |
| Turso Database (Rust) | 4 | 4 | 2 | 4 | 4 | 5 | 4 | 4 | 31 |
| libSQL (C fork) | 5 | 4 | 4 | 4 | 4 | 5 | 4 | 4 | 34 |
| PGlite | 2 | 3 | 2 | 4 | 5 | 5 | 4 | 3 | 28 |
| Embedded real Postgres | 4 | 4 | 5 | 3 | 5 | 5 | 4 | 1 | 31 |
| SurrealDB embedded | 3 | 3 | 2 | 3 | 4 | 4 | 2 | 3 | 24 |
| Custom engine on redb/fjall | 4 | 4 | 3 | 1 | 2 | 5 | 3 | 4 | 26 |

Why SQLite scores 4 on fit, not 5: no plpgsql, no RLS, no deferred checks, so some integrity moves to code (section 6). Why libSQL is below stock SQLite: its own README steers new projects away, and its vector index ties the file to libSQL.

Vector add-on choice (on top of SQLite):

| Option | Perf at 10^4–10^5 | Stability | Portability | Lock-in | Pick |
|---|---|---|---|---|---|
| BLOB column + exact scan (sqlite-vec scalar fns or Rust SIMD) | 4 | 5 | 5 | 5 | **now** |
| sqlite-vec DiskANN | 5 | 2 (alpha) | 5 | 4 | later, if > ~100k chunks |
| SQLite `vec1` | 5 | 2 (pre-1.0) | 5 | 4 | watch |
| In-memory HNSW (usearch/hnsw_rs) rebuilt from BLOBs | 5 | 4 | 4 | 5 | fallback |

## 6. Integrity: where the ontology rules live

Sprint enforces the ontology in Postgres with FKs, `CHECK`s, plpgsql triggers (`check_relation`, `check_object_update`, deferred `check_object_fields`), RLS and advisory locks. Locally there is no RLS (one file = one tenant), no plpgsql and no advisory locks (there is only one writer anyway). What SQLite can still do:

- **FKs** to the ontology tables (`objects.type → object_types.key`, `relations.type → relation_types.key`), with `PRAGMA foreign_keys=ON` on every connection (easy to forget; assert it at open) and deferrable FKs for imports and sync batches.
- **`CHECK`** on single rows: enums, ranges, `json_valid(properties)` / `json_type(properties,'$.x')='integer'`, key formats (`GLOB`), and with 3.53 they can be changed by `ALTER TABLE`.
- **Triggers** in SQL: `BEFORE INSERT ... WHEN (SELECT ...) THEN SELECT RAISE(ABORT,'relation_type_mismatch')` can enforce from/to types and cardinality (`max_active_per_from`) by counting, the 30-field limit, append-only `events` (`BEFORE UPDATE/DELETE ON events → RAISE(ABORT)`), and writing history rows. These cover most of Sprint's checks, but are verbose and have no deferred mode, so a check that must see a "finished" transaction (like `check_object_fields`) cannot be a trigger.
- **Application functions:** the core can register a deterministic Rust function, e.g. `ontology_valid(type, properties)`, and call it from `CHECK` or a trigger. Strong, but then the file can only be written by a process that registered the function (the `sqlite3` CLI fails with "no such function" on writes). Reads stay open. Acceptable if we document it; keep such checks to triggers so they can be dropped for repair tools.

**Recommended layering:**

1. **One domain core write path** (a Rust crate, also used by the hub; TS only if the whole stack is TS). Every write (UI, AI agent, importer, sync apply) is a typed command validated against the ontology (the Zod-schema equivalent in Rust, generated from the ontology tables), run in one `BEGIN IMMEDIATE` transaction that also appends the event. This is the primary guard and the only place with full context (deferred-style checks run at the end of the command).
2. **Database backstop:** FKs, `STRICT`, `CHECK`s and a small set of SQL triggers for the invariants whose breach corrupts the graph: type FKs, relation from/to types, cardinality, append-only events, no delete of ontology rows. Cheap and catches bugs in the core and in hand-written repair SQL.
3. **Post-sync validation and repair:** sync merges can produce states no single command would (two devices each add the "one allowed" `in_sprint` edge while offline). The core must run the same validators on merged state and repair deterministically (e.g. keep the edge with the lowest HLC/op id, end the other with `outcome=conflict`, log an event), so every device converges to the same repair. This needs the sync design to pick a deterministic rule; the DB backstop must not reject an incoming merged op outright, or devices diverge. So: backstop triggers enforce on local commands; the sync apply path runs in a mode that applies, then validates and repairs inside the same transaction.
4. **Periodic `PRAGMA integrity_check` / `foreign_key_check`** plus an ontology consistency check at start-up or nightly, with results in the event log.

## 7. Encryption at rest

| Option | Licence / cost | Overhead | Notes |
|---|---|---|---|
| **FileVault (macOS) / Data Protection (iOS)** | free, on by default on most Macs; iOS file protection classes | ~0 for the app | Protects a lost or stolen powered-off device. Does not protect against other processes of the same user, backups copied elsewhere, or a hub disk. |
| **SQLCipher Community** | BSD-style; must ship its copyright notice (https://www.zetetic.net/sqlcipher/community/). 4.18.0 (2026-08-18) (https://www.zetetic.net/blog/2026/08/18/sqlcipher-4.18.0-release/) | Vendor: "as little as 5–15%" *(snippet)*; independent numbers vary (https://discuss.zetetic.net/t/spiking-sqlcipher-for-room-surprising-benchmark-results/6961) | AES-256-CBC + HMAC-SHA512 per page. rusqlite `bundled-sqlcipher`; GRDB via SwiftPM since 7.10. Lags upstream SQLite by weeks. |
| **SQLite3MultipleCiphers** | MIT; Ulrich Telle; active (Sept 2026 HW acceleration) (https://utelle.github.io/SQLite3MultipleCiphers/) | ChaCha20-Poly1305 default, said to beat AES-CBC in software; AEGIS and Ascon available | Reads/writes SQLCipher v1–4 formats. Drop-in amalgamation; fewer language packages than SQLCipher. |
| **SEE** (SQLite team) | US $2,000 perpetual source licence, $3,500 with a year of support (https://sqlite.org/com/see.html) | low (**unverified**) | Official, closed source; licence forbids redistributing source. |
| **libSQL encryption / Turso Database encryption** | MIT | Turso: 6% read / 14% write (vendor) | Ties the file to that engine; Turso's is still experimental. |

**Recommendation:** v1 relies on FileVault/Data Protection plus keychain-held keys for secrets (API tokens) outside the DB. Build the core so the SQLite library is swappable (`bundled` → `bundled-sqlcipher` or SQLite3MultipleCiphers) and turn on page encryption for the **hub** and for any "blind relay" copy, where the disk is not under the user's control. Real protection for the hub question is end-to-end encryption of the sync payloads, which is a sync-design decision, not a storage one. Measure the overhead on our workload before enabling it on the Mac.

## 8. Schema evolution

- **Tools in SQLite:** `PRAGMA user_version` (an integer in the header) to track the applied migration; `ALTER TABLE ADD COLUMN` (no PK/UNIQUE, constant default), `RENAME COLUMN` (3.25), `DROP COLUMN` (3.35, fails if indexed or in a constraint), add/drop `NOT NULL` and `CHECK` (3.53). Everything else uses the documented **12-step rebuild** (create new table, copy, drop, rename, recreate indexes/triggers/views, with `foreign_keys=OFF` and a `foreign_key_check` after), all in one transaction. Migrations are embedded in the binary and run at open, after a backup via the online backup API or `VACUUM INTO`.
- **Fewer migrations:** this is where ADR-0005 already points: custom fields, custom types and relation types are **rows**, values are in `properties` JSON (`properties.fields`), and hot fields get generated columns + indexes. Core types keep typed columns where metrics need them. With this design most product changes are data, not DDL.
- **Devices on different app versions:** the local schema is private to each device; what must be compatible is the **sync payload**. Rules: (a) ops carry a `schema_version`; (b) a device preserves unknown JSON keys and unknown op kinds rather than dropping them (store-and-forward); (c) additive changes only within a major version; (d) a breaking change bumps a "minimum app version" that the hub or peers announce, and older apps go read-only until updated; (e) never reuse a field or type key. An old app must never rewrite a row and lose fields it doesn't know, so updates should be field-level patches, not whole-row writes.
- **Engine upgrades:** SQLite's file format is stable and backwards compatible since 2004 (format 4). PGlite, by contrast, had a hard break between 0.4 (PG 17) and 0.5 (PG 18) with no upgrade path, which is a serious risk for a long-lived personal store.

## 9. Multi-tenant later

- **SQLite file per tenant** is now a mainstream pattern: Cloudflare Durable Objects with SQLite are GA (April 2025) at 10 GB per object, and D1 recommends per-tenant databases (https://developers.cloudflare.com/changelog/2025-04-07-sqlite-in-durable-objects-ga); 37signals' `activerecord-tenanted` gives each tenant a SQLite file and is used in production (https://github.com/basecamp/activerecord-tenanted); Turso Cloud sells many small databases. Isolation is physical (no RLS bug can leak data across files), backup/restore/delete per user is a file operation, and the hub runs **the same schema and the same core code** as the Mac.
- **Postgres on the hosted hub** wins for cross-tenant queries, heavy concurrent writers per tenant, and managed HA, none of which a personal OS needs early. Keeping one schema for both would force the lowest common denominator (no plpgsql, `STRICT` types, JSON via functions on both sides); possible with sqlx or a thin query layer, but it doubles test matrices.
- **Recommendation:** hub = the same Rust core with one SQLite file per tenant (plus a small control DB for accounts). Revisit Postgres only for a paid multi-user product with shared workspaces, where one workspace has many concurrent writers.

## 10. Recommendation

**Use stock SQLite, bundled at ≥ 3.53.x, through rusqlite in a Rust domain core, on Mac, iOS, hub and (via the official wasm build with OPFS) web.**

- WAL, `STRICT`, `foreign_keys=ON`, `synchronous=NORMAL` in WAL (durable to the last checkpointed commit on power loss; use `FULL` if we want every commit fsynced; benchmark both), one writer connection plus a small reader pool.
- Schema close to Sprint's: `objects`, `relations`, typed extension tables, `events` (append-only by trigger), ontology tables, `properties` as JSONB-in-BLOB or JSON text with generated columns for hot fields.
- FTS5 with `unicode61 remove_diacritics 2`, plus a trigram index on titles; Polish stemming deferred (spike it).
- Embeddings as `BLOB` in `object_chunks` with exact scan; sqlite-vec loaded for its distance functions; ANN deferred.
- Integrity layered as in section 6; encryption as in section 7; per-tenant files on the hub.

**What would change this:**

- Turso Database reaches 1.0 with MVCC + indexes, encryption and CDC stable → switch engines (same file format, little code change), mainly for sync and concurrent background writes.
- SQLite `vec1` or sqlite-vec ANN ships 1.0 and we pass ~100k chunks per user → add the index.
- A hard requirement that the hosted product run shared multi-user workspaces with many concurrent writers → Postgres on the hub, SQLite on devices, with a sync layer translating.
- PGlite gets a native build with real fsync and multi-connection support → reconsider "Postgres everywhere".
- The sync research picks an engine that owns storage (e.g. a CRDT library storing its own docs) → SQLite stays as the query index, written from the CRDT state.

**Risks and unknowns:**

- Single writer: a long AI-agent write transaction blocks UI writes. Mitigation: one writer task with short transactions; measure p99 commit latency under a background embedding job.
- FTS quality for Polish without stemming.
- Integrity in triggers vs merges from sync (section 6, point 3) needs the sync design to define deterministic repair.
- Custom functions in triggers make the file less "open" for third-party writers.
- sqlite-vec has one maintainer and alpha ANN.
- WAL-class bugs: rare but real (2026's WAL-reset). Keep SQLite current, keep one writer, keep backups (`VACUUM INTO` daily snapshot).

## 11. Spike criteria (what must be proven before Accepted)

1. **Schema port:** Sprint's core tables, ontology tables and the `check_relation` / cardinality / conversion / append-only rules as SQLite FKs, `CHECK`s and triggers; a test suite that each bad write is refused with a named error. Pass: all of Sprint's integrity tests that apply have a SQLite equivalent or a documented core-only rule.
2. **Latency (Mac M-series, 50k objects, 200k relations, 1M events, 50k chunks):** single command commit p99 < 5 ms with `synchronous=NORMAL` (and report `FULL`); Home/sprint metric queries < 10 ms; 3-hop recursive CTE < 20 ms; FTS query < 10 ms; exact top-20 vector scan over 50k × 512 < 30 ms (float32) and the same at int8/float16.
3. **Concurrency:** UI writes stay < 20 ms p99 while a background job inserts 10k chunks in batches.
4. **Crash safety:** kill -9 and forced power-off simulation (e.g. `dm-flakey` or a VM) during writes, 1,000 iterations: zero corruption, zero lost acknowledged commits under the chosen `synchronous` level.
5. **Polish FTS:** on 200 real Sprint notes, recall of unicode61 vs unicode61+trigram vs a Snowball-Polish tokenizer for 30 inflected queries.
6. **Encryption:** the same benchmarks with SQLCipher and SQLite3MultipleCiphers (ChaCha20); pass if overhead ≤ 15% on writes.
7. **Portability:** the same Rust core and file open on macOS, iOS (simulator), Linux hub, and the web via SQLite wasm + OPFS, with the same migration set.
8. **Migration drill:** a 12-step rebuild of `objects` on the 50k dataset inside a transaction, timed, with `foreign_key_check` clean afterwards.
