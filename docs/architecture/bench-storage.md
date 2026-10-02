# Benchmark: local storage at 100k objects

Supports [ADR-0001](../decisions/0001-founding-architecture.md) → Local storage. Measured 2026-10-02. Throwaway scripts, included at the end.

## Question

Which local database holds one heavy user's context graph best: SQLite (with FTS5 and sqlite-vec), PGlite (Postgres compiled to WebAssembly), or a real Postgres run next to the app? "Best" here means cold start, query latency at p50 and p95, write latency with the ontology's triggers on, memory and disk.

## Machine and versions

- A shared cloud VM: Linux x86_64, 4 vCPU Intel Xeon at 2.8 GHz (AVX-512), 15 GB RAM, network disk. **Not a Mac.** An M-series Mac has faster cores and far more memory bandwidth, so treat every number here as a pessimistic bound; the *ratios* between engines are the point.
- Node 22.22.0.
- SQLite 3.53.4 (bundled by better-sqlite3 13.0.3), sqlite-vec 0.1.9. WAL, `synchronous=NORMAL` unless stated, 64 MB page cache, 256 MB mmap.
- PGlite 0.5.8 (PostgreSQL 18.3 on wasm32, 32-bit), pgvector 0.8.1 (`@electric-sql/pglite-pgvector` 0.0.9), Node filesystem storage.
- PostgreSQL 16.14 (Ubuntu package) with pgvector 0.6.0, `shared_buffers=256MB`, reached over a Unix socket with `pg` (so each query pays a round trip, as an app talking to a bundled server would). pgvector 0.6 has no `halfvec`, so it stores `vector` (float32).

## Dataset

Generated deterministically to look like Sprint after about ten years of heavy use ([`domain-requirements.md`](domain-requirements.md) puts one heavy user at 10k–100k objects):

- **100,000 objects:** 45% tasks, 15% ideas, 12% pages (100–800 words), 12% links, 5% objects of a custom type (`c_book`, with `properties.fields`), journal entries, people, files, insights, 520 sprints, 500 tags, 300 projects. Titles of 3–8 words from a 6,000-word Zipf-like vocabulary, so some words are very common and most are rare.
- **200,000 relations:** `in_sprint` (with `outcome` in properties), `tagged_with`, `mentions`, `child_of` (a page tree), `part_of`, `related_to`; 8% of them ended (`valid_to` set).
- **300,000 events** with JSON payloads.
- **100,000 embeddings** of 512 dimensions (one per object; clustered random unit vectors, so latency is realistic but recall numbers on this data mean nothing).
- The ontology's rules as **triggers**: one active `child_of` / `part_of` per source, `in_sprint` only from a task, and the event log written by triggers, as in Sprint.

## Results

Latency in milliseconds, p50 / p95, 200 runs each with random parameters (vectors: 20–30 runs).

| Query | SQLite | PGlite | Postgres 16 (native) |
|---|---|---|---|
| Sprint board: tasks in a sprint, ordered | **0.50 / 1.37** | 2.51 / 16.1 | 0.75 / 3.79 |
| Home "today": tasks planned on a date | **1.45 / 2.39** | 8.73 / 16.7 | 2.53 / 7.67 |
| One hop: all live neighbours of an object, both directions | **0.05 / 0.15** | 1.61 / 6.92 | 0.51 / 1.62 |
| Page subtree (recursive CTE) | 26.9 / 38.3 naive → **0.03 / 0.19** fixed ¹ | 1.14 / 5.72 | 0.39 / 0.65 |
| Two hops, skipping hub nodes (tags, sprints, projects) | 49 / 109, max 2.1 s naive → **0.08 / 0.25** in app code ² | – | – |
| Sprint metrics: outcomes for one sprint | 17.6 / 25.2 naive → **0.11 / 0.36** with the right index ³ | 0.79 / 3.75 | 0.39 / 0.85 |
| Custom-type table: filter on a JSON field, sort on another, 5k objects | **5.2 / 7.8** | 9.1 / 16.5 | 6.2 / 10.0 |
| Full text, one common word, top 20 by rank | **22.3 / 39.4** | 142 / 309 | 52 / 98 |
| Full text, one mid-frequency word | **6.2 / 11.0** | 50.9 / 154 | 26.0 / 63.5 |
| Full text, two words | **0.95 / 5.7** | 14.2 / 39.7 | 10.4 / 29.4 |
| Full text, 3-letter prefix | 30.0 / 58.6; **21.5 / 41.5** with a prefix index | 252 / 327 | 84 / 128 |
| Substring in titles (trigram index) | **0.08 / 0.39** | – | – |
| Vectors: exact top 10 of 100k × 512 | 122 / 142 (float32); 149 / 177 (int8); **64 / 70** (256 dims) | 511 / 734 (halfvec) | 443 / 569 (float32) |
| Write: new task + 2 edges, triggers on, one transaction | **0.19 / 1.20** (`NORMAL`); 1.25 / 7.9 (`FULL`) | 3.13 / 12.8 (**fsync off**) | 1.63 / 10.7 |
| Write: rename an object (event and FTS triggers) | **0.11 / 1.22**; 1.07 / 4.06 (`FULL`) | 1.19 / 7.56 | 0.74 / 6.16 |
| Bad edge refused by trigger | 50 / 50 | – | – |

¹ SQLite joined the recursive CTE to `objects` in the wrong order and scanned the table. `CROSS JOIN` fixes the order. ² The query-planner can't push "don't expand through tags" into an `OR` join well; doing the second hop as a loop of indexed one-hop queries in the core is 600× faster. ³ The partial index `where valid_to is null` can't serve a query about ended edges; a plain `(to_id, type)` index can.

| Startup, memory, size | SQLite | PGlite | Postgres 16 |
|---|---|---|---|
| Cold start: open + first board query | **35–46 ms** (62–80 ms counting loading the native module) | 850–970 ms | 100–107 ms with the server already up; the server itself starts in 140 ms warm, 6.2 s after an unclean stop |
| Memory | 72 MB process RSS at cold start; 356 MB after the full run (256 MB of it is mmap of the file, given back under pressure) | 330–406 MB | 75 MB client + 71 MB server |
| Bulk load: rows / indexes / vectors | 19.2 s / 3.1 s FTS / 10.7 s | 48.3 s / 8.7 s / 128 s | 28.4 s / 5.0 s / 31.0 s |
| On disk | 573 MB in one file, of which float32 vectors 206, int8 vectors 51, objects 96, events 54, FTS 50, relations 47 | 980 MB directory (492 MB database + WAL) | 659 MB |

## What the numbers say

1. **SQLite was fastest or equal on every query, but the margin over native Postgres is modest for indexed queries.** It is mostly 1.2–1.7× (board, Home, collections), 2–3× for full text, and 5–20× over PGlite; most of it is in-process calls with no round trip and no wasm. Writes at **equal durability** (SQLite `FULL` 1.25 / 7.9 ms against Postgres 1.63 / 10.7 ms, both fsyncing every commit) differ by about 1.3×; the `NORMAL` row doesn't fsync, so it isn't comparable to Postgres. Per class, at 100k objects on this VM: graph lookups under 2 ms p95 (after the fixes below), Home, board and collections under 10 ms, full text under 50 ms.
2. **The danger in SQLite is the query planner, not the engine.** Three naive queries were 25–2,000 ms until fixed with a join order, an app-side loop, or the right index. So: every query lives in the core with a test that checks its plan (`EXPLAIN QUERY PLAN`) and a latency budget at 100k objects.
3. **Full-text search fits the 50 ms budget, but only just, for very common words.** Ranking every match of a word found in tens of thousands of objects costs 20–40 ms. Mitigations: rank titles first (title-only FTS is tiny), a trigram index for substring-in-title search (0.4 ms p95), and bodies on Enter or as you pause.
4. **Exact vector search over 100k × 512 floats is too slow for "as you type"** (122 ms here) but fine for "related items" in a side panel. A real heavy user has far fewer vectors: ADR-0004 estimates about 15k chunks, which is ~18 ms here. For headroom: store 256-dimension Matryoshka vectors (halves the time), and add an approximate index (sqlite-vec's DiskANN or `vec1` once stable, or HNSW in the Rust core) past ~50k chunks. A numpy matrix-vector product on the same machine took 25 ms p50 for 100k × 512, so a SIMD scan in Rust can be several times faster than sqlite-vec 0.1.9's prebuilt scan; that's a spike item.
5. **PGlite is not a good primary store for a desktop app.** It is the slowest on everything, takes ~0.9 s to open, and **runs with `fsync=off`** (checked with `current_setting('fsync')`; see PGlite issue #1107), so an acknowledged write can be lost on a crash or power cut. Its value is Postgres compatibility, which the new architecture doesn't need.
6. **A bundled native Postgres is fast enough but buys nothing here:** a second process to supervise, 6 s recovery after a crash, no iOS at all (apps can't spawn processes), and 2–10× slower than SQLite for this workload.
7. **Writes are cheap on this design**, even with `synchronous=FULL` (1.25 ms p50): the ontology's triggers (cardinality, event log, FTS sync) cost well under a millisecond. This is Sprint's trigger design with a tiny event payload; the ADR's real write path (command → signed event → reducer) and macOS `fullfsync` are measured in spike S3.

**Fairness caveats (from the red-team review):**
- **Tuning wasn't equal.** SQLite's slow queries were hand-fixed, while Postgres 16 ran untuned with default `work_mem` and got no index fixes. Unfixed, SQLite was *slower* than Postgres on the page tree (26.9 vs 0.39 ms) and the metrics query (17.6 vs 0.39 ms).
- **The full-text word lists differ.** SQLite's common and mid words came from `fts5vocab` over all objects; Postgres's came from `ts_stat` over tasks only.
- **"Cold" start was warm.** It was a new process on a file the load step had just written, with no cache purge.
- **One run per engine** on a VM shared with another benchmark, so there is no variance and no p99.
- **Postgres's write transaction** paid five awaited socket round trips.
- **Vector search on native Postgres used pgvector 0.6.0 (443 ms)**, while Agent 6's run used 0.8.1 (31–37 ms on 125k). The gap is unexplained, so neither number is used as evidence.

Spike S3 reruns this on a Mac with purged caches, equal tuning, one word list, and p50, p95, p99 and max over at least 5 runs.
8. **Disk is dominated by vectors.** int8 (51 MB per 100k × 512) or 256-dimension float32 (103 MB) are the sensible formats; sqlite-vec 0.1.9 has no float16.

## Not measured here (spike items)

- Apple Silicon numbers, SQLCipher overhead, and rusqlite (the Rust binding) instead of better-sqlite3.
- Concurrent readers while one writer syncs a large batch.
- Crash consistency (`kill -9` during writes) and `VACUUM INTO` backup time.
- Vector recall on real embeddings (the embedding benchmark covers quality).

## Scripts

Run with `npm i better-sqlite3 sqlite-vec @electric-sql/pglite @electric-sql/pglite-pgvector pg`, then `node sqlite.mjs load && node sqlite.mjs query && node sqlite.mjs cold`, and `ENGINE=pglite|native node pg.mjs load|query|cold`. The fixed queries (¹ ² ³) and the prefix and trigram indexes were measured with three small probe scripts, included below after the main scripts.

<details><summary><code>gen.mjs</code></summary>

```js
// Deterministic synthetic dataset shaped like one heavy Sprint user after years of use.
export const N_OBJECTS = +(process.env.N || 100_000);
export const DIM = +(process.env.DIM || 512);
let s = 42;
export const rnd = () => ((s = (s * 1103515245 + 12345) % 2147483648) / 2147483648);
export const pick = (a) => a[Math.floor(rnd() * a.length)];
const syl = ["ka","ro","mi","ta","pe","lo","su","an","de","wi","zy","ne","po","ra","ch","sz","bo","le","gi","ut","or","ex","in","co","ma"];
const VOCAB = Array.from({ length: 6000 }, () => Array.from({ length: 2 + Math.floor(rnd() * 3) }, () => pick(syl)).join(""));
// Zipf-ish word choice so some words are common and most are rare.
const word = () => VOCAB[Math.floor(VOCAB.length * Math.pow(rnd(), 2.2))];
const words = (n) => Array.from({ length: n }, word).join(" ");
const hex = (n) => Array.from({ length: n }, () => Math.floor(rnd() * 16).toString(16)).join("");
let t0 = Date.parse("2016-01-01");
const uuid7 = () => { t0 += 1000 + Math.floor(rnd() * 3e6); const ts = t0.toString(16).padStart(12, "0"); const r = hex(20);
  return `${ts.slice(0, 8)}-${ts.slice(8)}-7${r.slice(0, 3)}-8${r.slice(3, 6)}-${r.slice(6, 18)}`; };
const day = (i) => new Date(Date.parse("2016-01-04") + i * 86400000).toISOString().slice(0, 10);
export const WS = "0190a000-0000-7000-8000-000000000001";

export function generate() {
  const mix = [["task", .45], ["idea", .15], ["page", .12], ["link", .12], ["journal_entry", .03], ["c_book", .05], ["improvement", .01], ["person", .02], ["insight", .01], ["file", .02], ["recurrence", .002]];
  const objects = [];
  const sprints = Array.from({ length: 520 }, (_, i) => ({ id: uuid7(), type: "sprint", title: `Sprint ${2016 + Math.floor(i / 52)}-W${(i % 52) + 1}`, body: words(30 + Math.floor(rnd() * 120)), props: { iso_week: (i % 52) + 1, rating: rnd() } }));
  const tags = Array.from({ length: 500 }, () => ({ id: uuid7(), type: "tag", title: word(), body: "", props: {} }));
  const projects = Array.from({ length: 300 }, () => ({ id: uuid7(), type: "project", title: words(3), body: words(40), props: { status: pick(["active", "done", "paused"]) } }));
  objects.push(...sprints, ...tags, ...projects);
  while (objects.length < N_OBJECTS) {
    let x = rnd(), type = "task"; for (const [t, p] of mix) { if ((x -= p) < 0) { type = t; break; } }
    const o = { id: uuid7(), type, title: words(3 + Math.floor(rnd() * 6)), props: {} };
    if (type === "page" || type === "journal_entry") o.body = words(100 + Math.floor(rnd() * 700));
    else if (type === "link") o.body = words(20 + Math.floor(rnd() * 60));
    else o.body = rnd() < .5 ? words(Math.floor(rnd() * 40)) : "";
    if (type === "task" || type === "idea") Object.assign(o, { status: pick(["todo", "in_progress", "done", "done", "done"]), priority: Math.floor(rnd() * 6), planned_on: rnd() < .6 ? day(Math.floor(rnd() * 3650)) : null });
    if (type === "c_book") o.props = { fields: { stage: pick(["o_aaaaaa", "o_bbbbbb", "o_cccccc", "o_dddddd"]), rating: Math.floor(rnd() * 6), author: words(2) } };
    if (type === "link") o.props = { url: `https://ex${hex(6)}.com/${hex(8)}`, kind: pick(["article", "video", "product"]), status: pick(["inbox", "read", "archived"]) };
    objects.push(o);
  }
  const byType = {}; for (const o of objects) (byType[o.type] ??= []).push(o);
  const rel = []; const R = (type, from, to, props = {}) => rel.push({ id: uuid7(), type, from_id: from, to_id: to, props, valid_to: rnd() < .08 ? "2020-01-01" : null });
  for (const t of byType.task) { const k = Math.floor(rnd() * 520); R("in_sprint", t.id, sprints[k].id, { outcome: pick(["completed", "completed", "carried_over", "removed"]), state: "accepted" }); if (rnd() < .2) R("in_sprint", t.id, sprints[Math.min(519, k + 1)].id, { state: "accepted" }); }
  const target = 2 * N_OBJECTS;
  const pages = byType.page; for (let i = 1; i < pages.length; i++) if (rnd() < .7) rel.push({ id: uuid7(), type: "child_of", from_id: pages[i].id, to_id: pages[Math.floor(rnd() * i)].id, props: { position: rnd() }, valid_to: null });
  for (const o of objects) if (rnd() < .3 && o.type !== "tag") R("part_of", o.id, pick(projects).id);
  while (rel.length < target) {
    const x = rnd(); const a = pick(objects);
    if (x < .55) R("tagged_with", a.id, pick(tags).id);
    else if (x < .9) R("mentions", pick(pages.concat(byType.journal_entry)).id, a.id, { block_id: hex(8) });
    else R("related_to", a.id, pick(objects).id, { similarity: rnd() });
  }
  return { objects, relations: rel, sprints, tags, byType };
}
// Clustered unit vectors, so the data has neighbourhoods like real embeddings do.
export function vectors(n) {
  const C = 200, cent = Array.from({ length: C }, () => Float32Array.from({ length: DIM }, () => rnd() * 2 - 1));
  const out = new Array(n);
  for (let i = 0; i < n; i++) { const c = cent[Math.floor(rnd() * C)], v = new Float32Array(DIM); let norm = 0;
    for (let d = 0; d < DIM; d++) { v[d] = c[d] + (rnd() * 2 - 1) * 0.9; norm += v[d] * v[d]; } norm = Math.sqrt(norm); for (let d = 0; d < DIM; d++) v[d] /= norm; out[i] = v; }
  return out;
}
export function stats(arr) { const a = [...arr].sort((x, y) => x - y); const q = (p) => a[Math.min(a.length - 1, Math.floor(p * a.length))]; return { p50: +q(.5).toFixed(3), p95: +q(.95).toFixed(3), max: +a[a.length - 1].toFixed(3) }; }
export async function timeit(n, fn) { const t = []; for (let i = 0; i < n; i++) { const s = performance.now(); await fn(i); t.push(performance.now() - s); } return stats(t); }
```

</details>

<details><summary><code>sqlite.mjs</code></summary>

```js
// SQLite (better-sqlite3) + FTS5 + sqlite-vec. Usage: node sqlite.mjs load|query
import Database from "better-sqlite3";
import * as vec from "sqlite-vec";
import fs from "node:fs";
import { generate, vectors, timeit, rnd, pick, WS, DIM } from "./gen.mjs";
const FILE = process.env.FILE || "data/sprint.db";
const mode = process.argv[2];
const SYNC = process.env.SYNC || "NORMAL";

function open() {
  const db = new Database(FILE);
  vec.load(db);
  db.pragma("journal_mode = WAL"); db.pragma(`synchronous = ${SYNC}`); db.pragma("foreign_keys = ON");
  db.pragma("cache_size = -64000"); db.pragma("mmap_size = 268435456"); db.pragma("temp_store = MEMORY");
  return db;
}

if (mode === "load") {
  fs.mkdirSync("data", { recursive: true }); for (const f of [FILE, FILE + "-wal", FILE + "-shm"]) fs.rmSync(f, { force: true });
  const db = open(); const t0 = performance.now();
  db.exec(`
  create table objects (id text primary key, workspace_id text not null, type text not null, title text not null, body_text text not null default '',
    properties text not null default '{}' check (json_valid(properties)), status text, priority integer check (priority between 0 and 5),
    planned_on text, created_at text not null, updated_at text not null, archived_at text) strict;
  create index objects_ws_type on objects (workspace_id, type) where archived_at is null;
  create index objects_planned on objects (workspace_id, planned_on) where planned_on is not null;
  create table relations (id text primary key, workspace_id text not null, type text not null, from_id text not null references objects(id),
    to_id text not null references objects(id), properties text not null default '{}', source text not null default 'user',
    status text not null default 'accepted', valid_from text not null, valid_to text) strict;
  create index rel_from on relations (from_id, type) where valid_to is null;
  create index rel_to on relations (to_id, type) where valid_to is null;
  create table events (id integer primary key, workspace_id text not null, occurred_at text not null, actor text not null, object_id text, kind text not null, payload text not null) strict;
  create index events_object on events (object_id, id);
  create virtual table objects_fts using fts5 (title, body_text, content='objects', content_rowid='rowid', tokenize='unicode61 remove_diacritics 2');
  create virtual table vec_f32 using vec0 (embedding float[${DIM}] distance_metric=cosine);
  create virtual table vec_i8 using vec0 (embedding int8[${DIM}] distance_metric=cosine);
  `);
  const { objects, relations } = generate();
  const insO = db.prepare(`insert into objects values (?,?,?,?,?,?,?,?,?,?,?,null)`);
  const insR = db.prepare(`insert into relations values (?,?,?,?,?,?,'user','accepted',?,?)`);
  const insE = db.prepare(`insert into events (workspace_id, occurred_at, actor, object_id, kind, payload) values (?,?,?,?,?,?)`);
  const now = new Date().toISOString();
  db.transaction(() => {
    for (const o of objects) insO.run(o.id, WS, o.type, o.title, o.body || "", JSON.stringify(o.props), o.status ?? null, o.priority ?? null, o.planned_on ?? null, now, now);
    for (const r of relations) insR.run(r.id, WS, r.type, r.from_id, r.to_id, JSON.stringify(r.props), now, r.valid_to);
    for (let i = 0; i < 3 * objects.length; i++) { const o = objects[i % objects.length]; insE.run(WS, now, pick(["user", "system", "ai"]), o.id, pick(["created", "updated", "status_changed", "linked"]), JSON.stringify({ before: { status: "todo" }, after: { status: "done" } })); }
  })();
  const tRows = performance.now();
  db.exec(`insert into objects_fts (objects_fts) values ('rebuild')`);
  const tFts = performance.now();
  const V = vectors(objects.length);
  const insV = db.prepare(`insert into vec_f32 (rowid, embedding) values (?, ?)`), insV8 = db.prepare(`insert into vec_i8 (rowid, embedding) values (?, vec_quantize_int8(?, 'unit'))`);
  db.transaction(() => { for (let i = 0; i < V.length; i++) { insV.run(BigInt(i + 1), V[i]); insV8.run(BigInt(i + 1), V[i]); } })();
  const tVec = performance.now();
  // Ontology triggers, added after the bulk load: cardinality, and the event log.
  db.exec(`
  create trigger rel_cardinality before insert on relations when new.type in ('child_of','part_of') begin
    select raise(abort, 'cardinality: one active edge per source') where exists (select 1 from relations where from_id = new.from_id and type = new.type and valid_to is null);
  end;
  create trigger rel_in_sprint_task before insert on relations when new.type = 'in_sprint' begin
    select raise(abort, 'in_sprint must start from a task') where (select type from objects where id = new.from_id) <> 'task';
  end;
  create trigger obj_event_ins after insert on objects begin
    insert into events (workspace_id, occurred_at, actor, object_id, kind, payload) values (new.workspace_id, new.created_at, 'user', new.id, 'created', json_object('title', new.title));
    insert into objects_fts (rowid, title, body_text) values (new.rowid, new.title, new.body_text);
  end;
  create trigger obj_event_upd after update on objects begin
    insert into events (workspace_id, occurred_at, actor, object_id, kind, payload) values (new.workspace_id, new.updated_at, 'user', new.id, 'updated', json_object('before', old.title, 'after', new.title));
    insert into objects_fts (objects_fts, rowid, title, body_text) values ('delete', old.rowid, old.title, old.body_text);
    insert into objects_fts (rowid, title, body_text) values (new.rowid, new.title, new.body_text);
  end;
  create trigger rel_event_ins after insert on relations begin
    insert into events (workspace_id, occurred_at, actor, object_id, kind, payload) values (new.workspace_id, new.valid_from, 'user', new.from_id, 'linked', json_object('type', new.type, 'to', new.to_id));
  end;`);
  db.exec("analyze"); db.pragma("wal_checkpoint(TRUNCATE)"); db.close();
  const size = fs.statSync(FILE).size;
  console.log(JSON.stringify({ engine: "sqlite", load_rows_ms: Math.round(tRows - t0), fts_build_ms: Math.round(tFts - tRows), vec_load_ms: Math.round(tVec - tFts), file_mb: +(size / 1e6).toFixed(1), objects: objects.length, relations: relations.length }));
}

if (mode === "cold") { // process start → open → first real query
  const t = performance.now(); const db = open();
  const r = db.prepare(`select o.id, o.title from relations r join objects o on o.id = r.from_id where r.to_id = (select id from objects where type='sprint' order by id desc limit 1) and r.type='in_sprint' and r.valid_to is null`).all();
  console.log(JSON.stringify({ engine: "sqlite", open_and_first_query_ms: +(performance.now() - t).toFixed(1), since_process_start_ms: Math.round(performance.now()), rows: r.length, rss_mb: Math.round(process.memoryUsage().rss / 1e6) }));
}

if (mode === "query") {
  const db = open(); const N = 200;
  const ids = db.prepare(`select id from objects`).pluck().all();
  const sprints = db.prepare(`select id from objects where type='sprint'`).pluck().all();
  const pages = db.prepare(`select id from objects where type='page'`).pluck().all();
  const days = db.prepare(`select distinct planned_on from objects where planned_on is not null`).pluck().all();
  db.exec(`create virtual table if not exists objects_fts_v using fts5vocab(objects_fts, 'row')`);
  const vocab = db.prepare(`select term, doc from objects_fts_v order by doc desc`).all();
  const common = vocab.slice(0, 50).map((x) => x.term), mid = vocab.slice(500, 1500).map((x) => x.term), rare = vocab.slice(-3000).map((x) => x.term);
  const out = {};
  const q = {
    board: db.prepare(`select o.id, o.title, o.status, o.priority from relations r join objects o on o.id = r.from_id where r.to_id = ? and r.type = 'in_sprint' and r.valid_to is null order by o.priority`),
    today: db.prepare(`select id, title, status from objects where workspace_id = ? and planned_on = ? and archived_at is null`),
    hop1: db.prepare(`select r.type, o.id, o.title from relations r join objects o on o.id = r.to_id where r.from_id = @id and r.valid_to is null
                      union all select r.type, o.id, o.title from relations r join objects o on o.id = r.from_id where r.to_id = @id and r.valid_to is null`),
    hop2: db.prepare(`with recursive n(id, d) as (select @id, 0 union select case when r.from_id = n.id then r.to_id else r.from_id end, n.d + 1
                      from n join relations r on (r.from_id = n.id or r.to_id = n.id) and r.valid_to is null where n.d < 2)
                      select distinct o.id, o.title from n join objects o on o.id = n.id limit 200`),
    page_tree: db.prepare(`with recursive t(id, depth) as (select @id, 0 union all select r.from_id, t.depth + 1 from t join relations r on r.to_id = t.id and r.type = 'child_of' and r.valid_to is null where t.depth < 20)
                      select o.id, o.title, t.depth from t join objects o on o.id = t.id`),
    metrics: db.prepare(`select properties ->> '$.outcome' as outcome, count(*) from relations where to_id = ? and type = 'in_sprint' group by 1`),
    collection: db.prepare(`select id, title from objects where workspace_id = ? and type = 'c_book' and archived_at is null and properties ->> '$.fields.stage' = ? order by properties ->> '$.fields.rating' desc limit 50`),
    fts: db.prepare(`select o.id, o.title, bm25(objects_fts, 5.0, 1.0) as score from objects_fts join objects o on o.rowid = objects_fts.rowid where objects_fts match ? order by score limit 20`),
    vec_f32: db.prepare(`select rowid, distance from vec_f32 where embedding match ? and k = 10`),
    vec_i8: db.prepare(`select rowid, distance from vec_i8 where embedding match vec_quantize_int8(?, 'unit') and k = 10`),
  };
  out.board = await timeit(N, () => q.board.all(pick(sprints)));
  out.today = await timeit(N, () => q.today.all(WS, pick(days)));
  out.hop1 = await timeit(N, () => q.hop1.all({ id: pick(ids) }));
  out.hop2 = await timeit(N, () => q.hop2.all({ id: pick(ids) }));
  out.page_tree = await timeit(N, () => q.page_tree.all({ id: pick(pages) }));
  out.metrics = await timeit(N, () => q.metrics.all(pick(sprints)));
  out.collection = await timeit(N, () => q.collection.all(WS, pick(["o_aaaaaa", "o_bbbbbb"])));
  out.fts_common = await timeit(N, () => q.fts.all(pick(common)));
  out.fts_mid = await timeit(N, () => q.fts.all(pick(mid)));
  out.fts_2terms = await timeit(N, () => q.fts.all(`${pick(mid)} ${pick(mid)}`));
  out.fts_prefix = await timeit(N, () => q.fts.all(`${pick(mid).slice(0, 3)}*`));
  const Q = vectors(60);
  out.vec_f32_k10 = await timeit(30, (i) => q.vec_f32.all(Q[i]));
  out.vec_i8_k10 = await timeit(30, (i) => q.vec_i8.all(Q[i]));
  // Writes: one object + two edges + triggers (event log, FTS, cardinality), one transaction each.
  const insO = db.prepare(`insert into objects values (?,?,?,?,?,?,?,?,?,?,?,null)`);
  const insR = db.prepare(`insert into relations values (?,?,?,?,?,?,'user','accepted',?,null)`);
  const upd = db.prepare(`update objects set title = ?, updated_at = ? where id = ?`);
  let k = Math.floor(Math.random() * 1e9); const now = new Date().toISOString();
  const write = db.transaction(() => { const id = `ffffffff-0000-7000-8000-${String(k++).padStart(12, "0")}`;
    insO.run(id, WS, "task", "new task " + k, "", "{}", "todo", 2, null, now, now);
    insR.run(id + "a", WS, "in_sprint", id, pick(sprints), "{}", now); insR.run(id + "b", WS, "part_of", id, pick(ids), "{}", now); });
  out.write_txn = await timeit(N, write);
  out.update_title = await timeit(N, () => upd.run("renamed " + rnd(), now, pick(ids)));
  let rejected = 0; const bad = ids.slice(0, 50);
  for (const id of bad) { try { insR.run(id + "x" + rnd(), WS, "in_sprint", pick(pages), pick(sprints), "{}", now); } catch { rejected++; } }
  out.trigger_rejects_bad_in_sprint = `${rejected}/50`;
  out.rss_mb = Math.round(process.memoryUsage().rss / 1e6);
  console.log(JSON.stringify({ engine: "sqlite", sync: SYNC, ...out }, null, 1));
}

if (mode === "vec") { // faster vector paths: binary quantisation + exact re-rank, and 256 dims
  const db = open();
  db.exec(`drop table if exists vec_bit; create virtual table vec_bit using vec0 (embedding bit[${DIM}]);
           drop table if exists vec_256; create virtual table vec_256 using vec0 (embedding float[256] distance_metric=cosine);`);
  const V = vectors(+(process.env.N || 100000));
  const ib = db.prepare(`insert into vec_bit (rowid, embedding) values (?, vec_quantize_binary(?))`), i2 = db.prepare(`insert into vec_256 (rowid, embedding) values (?, vec_normalize(vec_slice(?, 0, 256)))`);
  db.transaction(() => { for (let i = 0; i < V.length; i++) { ib.run(BigInt(i + 1), V[i]); i2.run(BigInt(i + 1), V[i]); } })();
  const Q = vectors(60);
  const exact = db.prepare(`select rowid from vec_f32 where embedding match ? and k = 10`);
  const bitq = db.prepare(`with c as (select rowid from vec_bit where embedding match vec_quantize_binary(@q) and k = 200)
     select c.rowid, vec_distance_cosine(f.embedding, @q) d from c join vec_f32 f on f.rowid = c.rowid order by d limit 10`);
  const q256 = db.prepare(`select rowid from vec_256 where embedding match vec_normalize(vec_slice(?, 0, 256)) and k = 10`);
  const truth = Q.map((q) => new Set(exact.all(q).map((r) => r.rowid)));
  const recall = (fn) => { let hit = 0; Q.forEach((q, i) => fn(q).forEach((r) => truth[i].has(r.rowid) && hit++)); return +(hit / (10 * Q.length)).toFixed(3); };
  console.log(JSON.stringify({ bit_rerank200: { ...(await timeit(60, (i) => bitq.all({ q: Q[i] }))), recall10: recall((q) => bitq.all({ q })) },
    f32_256dims: { ...(await timeit(60, (i) => q256.all(Q[i]))), recall10_vs_512: recall((q) => q256.all(q)) },
    sizes_mb: db.prepare(`select name, round(sum(pgsize)/1e6,1) mb from dbstat where name like 'vec_%chunks%' group by name`).all() }, null, 1));
}
```

</details>

<details><summary><code>pg.mjs</code></summary>

```js
// PGlite (wasm, Node filesystem) or native Postgres 16, same schema and queries. Usage: ENGINE=pglite|native node pg.mjs load|query|cold
import fs from "node:fs";
import { generate, vectors, timeit, rnd, pick, WS, DIM } from "./gen.mjs";
const ENGINE = process.env.ENGINE || "pglite", mode = process.argv[2], DIR = "data/pglite";
const VT = process.env.VT || "halfvec";
async function connect() {
  if (ENGINE === "pglite") { const { PGlite } = await import("@electric-sql/pglite"); const { vector } = await import("@electric-sql/pglite-pgvector");
    const db = await PGlite.create(DIR, { extensions: { vector } }); return { q: (s, p) => db.query(s, p), exec: (s) => db.exec(s), close: () => db.close() }; }
  const pg = (await import("pg")).default; const c = new pg.Client({ host: process.env.PGHOST || "/tmp/claude-0/pgsock", port: 55432, database: "postgres", user: process.env.PGUSER || "postgres" }); await c.connect();
  return { q: (s, p) => c.query(s, p), exec: (s) => c.query(s), close: () => c.end() };
}
const vecLit = (v) => "[" + Array.from(v, (x) => x.toFixed(5)).join(",") + "]";

if (mode === "load") {
  if (ENGINE === "pglite") fs.rmSync(DIR, { recursive: true, force: true });
  const db = await connect(); const t0 = performance.now();
  await db.exec(`drop table if exists chunks, events, relations, objects cascade; create extension if not exists vector;
  create table objects (id uuid primary key, workspace_id uuid not null, type text not null, title text not null, body_text text not null default '',
    properties jsonb not null default '{}', status text, priority int check (priority between 0 and 5), planned_on date,
    created_at timestamptz not null default now(), updated_at timestamptz not null default now(), archived_at timestamptz,
    search tsvector generated always as (setweight(to_tsvector('simple', title), 'A') || setweight(to_tsvector('simple', left(body_text, 100000)), 'B')) stored);
  create table relations (id uuid primary key, workspace_id uuid not null, type text not null, from_id uuid not null references objects, to_id uuid not null references objects,
    properties jsonb not null default '{}', source text not null default 'user', status text not null default 'accepted', valid_from timestamptz not null default now(), valid_to timestamptz);
  create table events (id bigserial primary key, workspace_id uuid not null, occurred_at timestamptz not null default now(), actor text not null, object_id uuid, kind text not null, payload jsonb not null);
  create table chunks (object_id uuid not null, chunk int not null, embedding ${VT}(${DIM}) not null, primary key (object_id, chunk));`);
  const { objects, relations } = generate();
  const B = 5000;
  for (let i = 0; i < objects.length; i += B) await db.q(`insert into objects (id, workspace_id, type, title, body_text, properties, status, priority, planned_on)
    select id, '${WS}', type, title, body_text, properties, status, priority, planned_on from jsonb_to_recordset($1::jsonb) as x(id uuid, type text, title text, body_text text, properties jsonb, status text, priority int, planned_on date)`,
    [JSON.stringify(objects.slice(i, i + B).map((o) => ({ id: o.id, type: o.type, title: o.title, body_text: o.body || "", properties: o.props, status: o.status ?? null, priority: o.priority ?? null, planned_on: o.planned_on ?? null })))]);
  for (let i = 0; i < relations.length; i += B) await db.q(`insert into relations (id, workspace_id, type, from_id, to_id, properties, valid_to)
    select id, '${WS}', type, from_id, to_id, properties, valid_to from jsonb_to_recordset($1::jsonb) as x(id uuid, type text, from_id uuid, to_id uuid, properties jsonb, valid_to timestamptz)`,
    [JSON.stringify(relations.slice(i, i + B).map((r) => ({ ...r, properties: r.props })))]);
  await db.exec(`insert into events (workspace_id, actor, object_id, kind, payload) select workspace_id, 'user', id, k, '{"before":{"status":"todo"},"after":{"status":"done"}}' from objects, unnest(array['created','updated','linked']) k`);
  const tRows = performance.now();
  await db.exec(`create index objects_ws_type on objects (workspace_id, type) where archived_at is null; create index objects_planned on objects (workspace_id, planned_on) where planned_on is not null;
    create index rel_from on relations (from_id, type) where valid_to is null; create index rel_to on relations (to_id, type) where valid_to is null; create index rel_to_all on relations (to_id, type);
    create index events_object on events (object_id, id); create index objects_search on objects using gin (search);`);
  const tIdx = performance.now();
  const V = vectors(objects.length);
  for (let i = 0; i < objects.length; i += 2000) await db.q(`insert into chunks select x.id, 0, x.e::${VT} from jsonb_to_recordset($1::jsonb) as x(id uuid, e text)`,
    [JSON.stringify(objects.slice(i, i + 2000).map((o, j) => ({ id: o.id, e: vecLit(V[i + j]) })))]);
  const tVec = performance.now();
  await db.exec(`create or replace function check_cardinality() returns trigger language plpgsql as $$ begin
      if new.type in ('child_of','part_of') and exists (select 1 from relations where from_id = new.from_id and type = new.type and valid_to is null) then raise exception 'cardinality'; end if;
      if new.type = 'in_sprint' and (select type from objects where id = new.from_id) <> 'task' then raise exception 'in_sprint must start from a task'; end if;
      insert into events (workspace_id, actor, object_id, kind, payload) values (new.workspace_id, 'user', new.from_id, 'linked', jsonb_build_object('type', new.type, 'to', new.to_id));
      return new; end $$;
    create trigger rel_check before insert on relations for each row execute function check_cardinality();
    create or replace function log_object() returns trigger language plpgsql as $$ begin
      insert into events (workspace_id, actor, object_id, kind, payload) values (new.workspace_id, 'user', new.id, tg_op, jsonb_build_object('title', new.title)); return new; end $$;
    create trigger obj_log after insert or update on objects for each row execute function log_object(); analyze;`);
  let hnsw = null; if (process.env.HNSW) { const t = performance.now(); await db.exec(`create index chunks_hnsw on chunks using hnsw (embedding ${VT}_cosine_ops)`); hnsw = Math.round(performance.now() - t); }
  const size = (await db.q(`select pg_database_size(current_database()) s`)).rows[0].s;
  await db.close();
  console.log(JSON.stringify({ engine: ENGINE, load_rows_ms: Math.round(tRows - t0), index_build_ms: Math.round(tIdx - tRows), vec_load_ms: Math.round(tVec - tIdx), hnsw_build_ms: hnsw, db_mb: +(Number(size) / 1e6).toFixed(1) }));
}

if (mode === "cold") {
  const t = performance.now(); const db = await connect();
  const r = await db.q(`select o.id, o.title from relations r join objects o on o.id = r.from_id where r.to_id = (select id from objects where type='sprint' order by id desc limit 1) and r.type='in_sprint' and r.valid_to is null`);
  console.log(JSON.stringify({ engine: ENGINE, open_and_first_query_ms: +(performance.now() - t).toFixed(1), rows: r.rows.length, rss_mb: Math.round(process.memoryUsage().rss / 1e6) })); await db.close();
}

if (mode === "query") {
  const db = await connect(); const N = +(process.env.ITER || 200); const col = (r) => r.rows.map((x) => Object.values(x)[0]);
  const ids = col(await db.q(`select id from objects`)), sprints = col(await db.q(`select id from objects where type='sprint'`)), pages = col(await db.q(`select id from objects where type='page'`));
  const days = col(await db.q(`select distinct planned_on::text from objects where planned_on is not null`));
  const words = col(await db.q(`select word from ts_stat('select search from objects where type = ''task''') order by ndoc desc`));
  const common = words.slice(0, 50), mid = words.slice(500, 1500);
  const out = {};
  out.board = await timeit(N, () => db.q(`select o.id, o.title, o.status, o.priority from relations r join objects o on o.id = r.from_id where r.to_id = $1 and r.type = 'in_sprint' and r.valid_to is null order by o.priority`, [pick(sprints)]));
  out.today = await timeit(N, () => db.q(`select id, title, status from objects where workspace_id = $1 and planned_on = $2 and archived_at is null`, [WS, pick(days)]));
  out.hop1 = await timeit(N, () => db.q(`select r.type, o.id, o.title from relations r join objects o on o.id = r.to_id where r.from_id = $1 and r.valid_to is null
                      union all select r.type, o.id, o.title from relations r join objects o on o.id = r.from_id where r.to_id = $1 and r.valid_to is null`, [pick(ids)]));
  out.page_tree = await timeit(N, () => db.q(`with recursive t(id, depth) as (select $1::uuid, 0 union all select r.from_id, t.depth + 1 from t join relations r on r.to_id = t.id and r.type = 'child_of' and r.valid_to is null where t.depth < 20)
                      select o.id, o.title, t.depth from t join objects o on o.id = t.id`, [pick(pages)]));
  out.metrics = await timeit(N, () => db.q(`select properties ->> 'outcome', count(*) from relations where to_id = $1 and type = 'in_sprint' group by 1`, [pick(sprints)]));
  out.collection = await timeit(N, () => db.q(`select id, title from objects where workspace_id = $1 and type = 'c_book' and archived_at is null and properties #>> '{fields,stage}' = $2 order by (properties #>> '{fields,rating}') desc limit 50`, [WS, pick(["o_aaaaaa", "o_bbbbbb"])]));
  const fts = (q) => db.q(`select id, title, ts_rank(search, q) r from objects, to_tsquery('simple', $1) q where search @@ q order by r desc limit 20`, [q]);
  out.fts_common = await timeit(N, () => fts(pick(common)));
  out.fts_mid = await timeit(N, () => fts(pick(mid)));
  out.fts_2terms = await timeit(N, () => fts(`${pick(mid)} & ${pick(mid)}`));
  out.fts_prefix = await timeit(N, () => fts(`${pick(mid).slice(0, 3)}:*`));
  const Q = vectors(30).map(vecLit);
  if (process.env.HNSW) await db.exec(`set hnsw.ef_search = 100`); else await db.exec(`set enable_indexscan = on`);
  out.vec_halfvec_k10 = await timeit(20, (i) => db.q(`select object_id, embedding <=> $1::${VT} d from chunks order by d limit 10`, [Q[i]]));
  let k = Math.floor(Math.random() * 1e9);
  out.write_txn = await timeit(N, async () => { const id = `ffffffff-0000-7000-8000-${String(k++).padStart(12, "0")}`;
    await db.exec(`begin`); await db.q(`insert into objects (id, workspace_id, type, title, status, priority) values ($1, $2, 'task', 'new task', 'todo', 2)`, [id, WS]);
    await db.q(`insert into relations (id, workspace_id, type, from_id, to_id) values (gen_random_uuid(), $1, 'in_sprint', $2, $3)`, [WS, id, pick(sprints)]);
    await db.q(`insert into relations (id, workspace_id, type, from_id, to_id) values (gen_random_uuid(), $1, 'part_of', $2, $3)`, [WS, id, pick(ids)]); await db.exec(`commit`); });
  out.update_title = await timeit(N, () => db.q(`update objects set title = $1, updated_at = now() where id = $2`, ["renamed " + rnd(), pick(ids)]));
  out.rss_mb = Math.round(process.memoryUsage().rss / 1e6);
  console.log(JSON.stringify({ engine: ENGINE, hnsw: !!process.env.HNSW, ...out }));
  await db.close();
}
```

</details>

<details><summary><code>probe.mjs</code></summary>

```js
import Database from "better-sqlite3"; import * as vec from "sqlite-vec"; import { timeit, pick, vectors } from "./gen.mjs";
const db = new Database("data/sprint.db"); vec.load(db); db.pragma("mmap_size = 268435456"); db.pragma("cache_size = -64000");
const pages = db.prepare(`select id from objects where type='page'`).pluck().all();
const ids = db.prepare(`select id from objects`).pluck().all();
const sprints = db.prepare(`select id from objects where type='sprint'`).pluck().all();
const tree = db.prepare(`with recursive t(id, depth) as (select @id, 0 union all select r.from_id, t.depth + 1 from t join relations r on r.to_id = t.id and r.type = 'child_of' and r.valid_to is null where t.depth < 20) select o.id, o.title, t.depth from t join objects o on o.id = t.id`);
console.log("page_tree", await timeit(200, () => tree.all({ id: pick(pages) })));
const tree2 = db.prepare(`with recursive t(id, depth) as (select @id, 0 union all select r.from_id, t.depth + 1 from t join relations r on r.to_id = t.id and r.type = 'child_of' and r.valid_to is null where t.depth < 20) select count(*) from t`);
console.log("page_tree_ids_only", await timeit(200, () => tree2.all({ id: pick(pages) })));
// 2 hops: two indexed branches per step; don't expand through hub nodes (tags, sprints, projects); cap fan-out.
const hop2 = db.prepare(`with recursive n(id, d) as (
   select @id, 0
   union select x.id, n.d + 1 from n join (select from_id k, to_id id from relations where valid_to is null union all select to_id, from_id from relations where valid_to is null) x on x.k = n.id
   where n.d < 2 and (n.d = 0 or (select type from objects where id = n.id) not in ('tag','sprint','project')))
 select o.id, o.title, min(n.d) d from n join objects o on o.id = n.id group by o.id order by d limit 200`);
console.log("hop2_v2", await timeit(200, () => hop2.all({ id: pick(ids) })));
const metrics = db.prepare(`select properties ->> '$.outcome' outcome, count(*) from relations where to_id = ? and type = 'in_sprint' group by 1`);
db.exec(`create index if not exists rel_to_all on relations (to_id, type)`);
console.log("metrics_with_full_index", await timeit(200, () => metrics.all(pick(sprints))));
// Vector floor without SQL: contiguous Float32Array dot products in JS, k=10.
const n = 100000, D = 512, V = vectors(n), M = new Float32Array(n * D); V.forEach((v, i) => M.set(v, i * D));
const Q = vectors(30);
console.log("js_bruteforce_f32_512", await timeit(30, (i) => { const q = Q[i]; const top = []; for (let r = 0; r < n; r++) { let s = 0; const o = r * D; for (let d = 0; d < D; d++) s += M[o + d] * q[d]; if (top.length < 10 || s > top[9][0]) { top.push([s, r]); top.sort((a, b) => b[0] - a[0]); if (top.length > 10) top.pop(); } } return top; }));
```

</details>

<details><summary><code>probe2.mjs</code></summary>

```js
import Database from "better-sqlite3"; import { timeit, pick } from "./gen.mjs";
const db = new Database("data/sprint.db"); db.pragma("mmap_size = 268435456"); db.pragma("cache_size = -64000");
const pages = db.prepare(`select id from objects where type='page'`).pluck().all();
const ids = db.prepare(`select id from objects`).pluck().all();
const tree = db.prepare(`with recursive t(id, depth) as (select @id, 0 union all select r.from_id, t.depth + 1 from t join relations r on r.to_id = t.id and r.type = 'child_of' and r.valid_to is null where t.depth < 20)
  select o.id, o.title, t.depth from t cross join objects o on o.id = t.id`);
console.log("page_tree_crossjoin", await timeit(200, () => tree.all({ id: pick(pages) })));
// 2 hops as the app would do it: 1-hop query twice, skipping hub nodes on the second step.
const hop1 = db.prepare(`select r.type, o.id, o.type otype, o.title from relations r cross join objects o on o.id = r.to_id where r.from_id = @id and r.valid_to is null
                      union all select r.type, o.id, o.type, o.title from relations r cross join objects o on o.id = r.from_id where r.to_id = @id and r.valid_to is null`);
const HUB = new Set(["tag", "sprint", "project"]);
let sizes = [];
console.log("hop2_app", await timeit(200, () => { const first = hop1.all({ id: pick(ids) }); const seen = new Map(first.map((r) => [r.id, r]));
  for (const r of first) if (!HUB.has(r.otype)) for (const s of hop1.all({ id: r.id })) if (!seen.has(s.id)) seen.set(s.id, s); sizes.push(seen.size); }));
sizes.sort((a, b) => a - b); console.log("hop2 result size p50", sizes[100], "p95", sizes[190]);
```

</details>

<details><summary><code>probe3.mjs</code></summary>

```js
import Database from "better-sqlite3"; import { timeit, pick } from "./gen.mjs";
const db = new Database("data/sprint.db"); db.pragma("mmap_size = 268435456"); db.pragma("cache_size = -64000");
let t = performance.now();
db.exec(`drop table if exists fts_p; create virtual table fts_p using fts5 (title, body_text, content='objects', content_rowid='rowid', tokenize='unicode61 remove_diacritics 2', prefix='2 3');
         insert into fts_p (fts_p) values ('rebuild');`);
console.log("fts with prefix index build ms", Math.round(performance.now() - t));
t = performance.now();
db.exec(`drop table if exists fts_tri; create virtual table fts_tri using fts5 (title, content='objects', content_rowid='rowid', tokenize='trigram remove_diacritics 1'); insert into fts_tri (fts_tri) values ('rebuild');`);
console.log("trigram title index build ms", Math.round(performance.now() - t));
const vocab = db.prepare(`select term from objects_fts_v order by doc desc`).pluck().all(); const mid = vocab.slice(500, 1500), common = vocab.slice(0, 50);
const q = db.prepare(`select o.id, o.title, bm25(fts_p, 5.0, 1.0) s from fts_p join objects o on o.rowid = fts_p.rowid where fts_p match ? order by s limit 20`);
console.log("prefix3 with prefix index", await timeit(200, () => q.all(`${pick(mid).slice(0, 3)}*`)));
console.log("common term", await timeit(200, () => q.all(pick(common))));
// Typical search-as-you-type: every word a prefix, title-weighted, top 20 — like Sprint's search.
console.log("2 prefix terms", await timeit(200, () => q.all(`${pick(mid).slice(0, 4)}* ${pick(mid).slice(0, 3)}*`)));
const tri = db.prepare(`select o.id, o.title from fts_tri join objects o on o.rowid = fts_tri.rowid where fts_tri match ? limit 20`);
console.log("trigram substring on titles", await timeit(200, () => tri.all(`"${pick(mid).slice(1, 5)}"`)));
console.log(db.prepare(`select name, round(sum(pgsize)/1e6,1) mb from dbstat where name like 'fts_%' group by 1`).all());
```

</details>
