# Benchmarks behind ADR-0001

> Run on 2026-10-02 for [ADR-0001](../decisions/0001-founding-architecture.md). The scripts were throwaway and are not in the repo. This page records the method and the numbers so they can be rerun on a Mac in spike S2.

## Machine and versions

- A cloud VM: 4 vCPU Intel Xeon @ 2.1 GHz, 16 GB RAM, Linux, Node 22.22.
- This is **slower than an Apple Silicon Mac** (expect M-series single-thread speed to be roughly 2× this), so treat every number as an upper bound. The ratios between engines matter more than the absolute values.
- **SQLite** 3.53.4 via `better-sqlite3` 13.0.3, with FTS5 and `sqlite-vec` 0.1.9. Settings: WAL, `synchronous=NORMAL`, 64 MB cache, 256 MB mmap.
- **PGlite** 0.5.8, which is PostgreSQL 18.3 compiled to 32-bit WASM, with `@electric-sql/pglite-pgvector` 0.0.9 (pgvector 0.8.1) and `unaccent`, persisted to a directory on disk.
- **Native PostgreSQL** 16 with pgvector 0.8.1, over a local socket with postgres.js. This is the reference point for "real Postgres".

## The dataset

A deterministic synthetic workspace the size of a heavy user after several years (`architecture/context-graph.md` in Sprint puts one person at 10k–100k objects):

| | Count |
|---|---|
| Objects (45% tasks, 25% ideas, 30% pages, 200 tags) | 100,000 |
| Edges (`tagged_with`, `child_of` page tree, `mentions`, `blocks`) | 159,768 |
| Embedding chunks, 512 dimensions, clustered around 64 topics | 124,835 |
| Event-log rows written by triggers | 260k (SQLite) / 334k (Postgres) |

- Titles and bodies come from a 6,000-word vocabulary with Polish diacritics and a Zipf distribution. Page bodies have 3–28 paragraphs as BlockNote-style JSON.
- **The Postgres variants used Sprint's real schema.** All 25 migration files, unchanged, so every trigger, check and RLS policy was live during the load.
- **The SQLite variant used an equivalent hand-written schema:**
  - `STRICT` tables, CHECKs, JSON validity checks;
  - triggers that write the event log;
  - a partial unique index for "one parent per page";
  - a recursive-CTE trigger that refuses page cycles;
  - an FTS5 index kept in step by triggers;
  - a `vec0` table for the vectors.

## Results (100k objects)

### Load, size, opening

| | SQLite | PGlite | Native Postgres |
|---|---|---|---|
| Load everything (batches of 2,000 in transactions) | **48 s** | 6 min 28 s | 2 min 14 s |
| Size on disk | **0.89 GB** | 1.6 GB | 0.59 GB |
| Open the database (new process) | **7 ms** | 390–550 ms | 30 ms (connect to a running server) |
| First query after opening | **0.5 ms** | 55–68 ms | n/a |
| Process memory after the query run | 386 MB (mostly the 256 MB mmap) | 404–420 MB | server process, not comparable |

### Queries (30 runs each; p50 / p95 in ms)

| Query | SQLite | PGlite | Native Postgres |
|---|---|---|---|
| Full-text prefix search, top 20 (Sprint's current query shape: rank computed from the expression) | — | 6,159 / 18,932 | 2,122 / 4,277 |
| Full-text prefix search, top 20, **stored index column** (FTS5 / stored `tsvector` + GIN) | **13 / 51** | 150 / 601 | 79 / 155 |
| Exact vector top-10 over 125k chunks (no ANN index) | 115 / 125 | 617 / 637 | **31 / 50** |
| Graph: 2-hop neighbourhood of a page | **1.9 / 4.1** | 5.1 / 11.6 | 2.7 / 5.7 |
| Board: a week's work items with their tags | **5.4 / 10.0** | 29.7 / 78.3 | 13.7 / 34.9 |
| A page's children (sidebar tree) | **0.04 / 0.10** | 0.98 / 1.69 | 0.49 / 0.85 |
| Write: rename a task (trigger writes an event), one transaction | **0.11 / 3.3** | 2.4 / 6.5 | 1.9 / 3.6 |
| Write: create a task + its row + a tag edge, one transaction | **0.21 / 1.3** | 5.4 / 8.9 | 3.9 / 7.8 |

### Vector search variants (SQLite, same 125k chunks)

| Variant | p50 / p95 (ms) | Recall@10 vs exact |
|---|---|---|
| `sqlite-vec` float32, exact | 117 / 126 | 1.0 |
| `sqlite-vec` int8 quantized | 72 / 83 | 0.78* |
| `sqlite-vec` binary quantized + rerank top 200 | 80 / 87 | 0.60* |
| Plain JavaScript loop over a `Float32Array` (no SIMD) | 120 / 136 | 1.0 |

\* The synthetic vectors are random noise around 64 centres, so many neighbours tie. Recall on real embeddings will be much higher. These figures only show that quantization alone does not reach the target here.

## What the numbers say

1. **SQLite is the fastest local engine by a wide margin** on everything except brute-force vectors:
   - 4–60× faster than PGlite on queries and writes;
   - opens in milliseconds;
   - loads 8× faster.
2. **PGlite works, but costs speed and size.**
   - It runs Sprint's whole schema unchanged (triggers, deferred constraint triggers, RLS with roles, advisory locks, pgvector `halfvec`, `unaccent`). That was the reason to consider it.
   - In WASM it is 2–5× slower than native Postgres on relational queries and about 20× slower on vectors (no SIMD).
   - Opening the database takes about half a second, against SQLite's 7 ms.
   - With no data migration (decided 2026-10-02), the one big advantage, reusing Sprint's schema, is gone.
3. **Vector search needs its own plan.** No engine's brute-force scan of 125k × 512 floats hits 50 ms on this VM except native Postgres with SIMD.
   - The realistic size is smaller. A few thousand objects a year gives about 10–30k chunks, which scan in about 10–30 ms.
   - The ADR's plan is a SIMD scan or an HNSW index (`usearch`) in the core process, measured in spike S7.
4. **Sprint's own full-text query does not scale.** It recomputes the document vector to rank every match, which took 2 s on native Postgres at 100k objects. The new project indexes a stored column from day one. (This is a note for Sprint too, though it is far from 100k objects.)

## SQLite capability checks

Each was checked directly in SQLite 3.53.4:

- A trigger can use a recursive CTE and `raise(abort, …)`, so the "no page inside itself" rule works in SQLite exactly as in Sprint's migration 0013.
- Partial unique indexes enforce "one active parent per page" and "one project per item".
- `pragma defer_foreign_keys` exists. There are **no deferred constraint triggers**, so rules of the form "every task has its row by commit time" must be designed away: one table, or checked by the mutator.
- `jsonb()` is available (3.45+).
- **FTS5 `unicode61 remove_diacritics 2` does not fold `ł` to `l`.** "zazolc" does not find "zażółć", although "gesla" finds "gęślą". Postgres `unaccent` does fold it. The new project must normalise text itself before indexing: NFKD, strip marks, then map the letters without a decomposition (`ł→l`, `ø→o`, `đ→d`, …), into a search column. This is the same rule as Sprint's R46.

## PGlite capability checks

Each was checked directly in PGlite 0.5.8:

- PostgreSQL 18.3; pgvector 0.8.1 with `halfvec`; `unaccent` folds `zażółć` to `zazolc`.
- `create role` and `set role` with RLS policies work. A non-owner role sees only its rows.
- Deferred constraint triggers fire at commit. `pg_try_advisory_xact_lock` works.
- **All 25 of Sprint's migration files apply unchanged.** The longest took 67 ms; the database itself is created in about 3.7 s on first open.
- `pglite-socket` serves the Postgres wire protocol but ignores the user in the connection string (every connection is the superuser), so RLS can't be tested through it.

## Sync prototype (spike S3 preview)

A throwaway prototype of the design in [`sync.md`](sync.md):

- Replicas: SQLite, in memory.
- Mutators: deterministic, with all ids and dates in their arguments.
- Server: a sequencer that only orders operations.
- Each replica:
  - applies its own operations at once;
  - on receiving the server's order, undoes its pending operations through a trigger-captured undo log, applies the ordered operations, then re-applies what is still pending.

The schema enforced, in SQLite, on every replica:

- one active parent per page;
- no page cycles (a trigger);
- one project per item;
- one journal entry per day, with a deterministic id and find-or-create;
- valid edge endpoints.

Page bodies were Yjs documents edited concurrently.

| Run | Result |
|---|---|
| 3 devices, 3,000 random steps each run, devices going offline and online at random, 200 seeds | see below* |
| 5 devices, 10,000 steps, 30 seeds | see below* |
| Negative control: the same fuzz with the rebase switched off | **0 of 5 converged**, so the test does detect divergence |

\* Filled in when the long runs finish. A first run of 3 seeds converged: every replica had the same state hash and the same page texts, every invariant held, no operation was lost (about 2,400 operations each, 4–5% rejected deterministically on every replica).
