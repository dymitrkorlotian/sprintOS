# Benchmark: Loro vs Automerge vs Yjs

> **Read with care** (red-team review, PR #1): this ran in Node (JavaScript/wasm), not natively in Rust, on a flat single-user text trace with no blocks, marks or concurrency. Loro's 14 ms load is lazy, while Yjs's 51 ms is a full decode. The page-move rows test a job the ADR gives to the reducer, not to Loro. And loro-prosemirror 0.4.4 stores blocks in a plain `LoroList`, so the movable-list advantage doesn't reach the editor through the existing binding. Spike S1b reruns this natively on a block-structured trace.

Supports [ADR-0001](../decisions/0001-founding-architecture.md) → Sync. Measured 2026-10-02. Throwaway scripts; the core is at the end.

## Question

Which CRDT library should hold sprintOS data: rich-text pages, the object graph, and the sync payloads between Mac, hub, web and phone? We measure speed, size, memory, merge behaviour and what happens with a "move" conflict in a page tree.

## Machine and versions

- Shared cloud VM: Linux x86_64, 4 vCPU Intel Xeon at 2.8 GHz, 15 GB RAM. **Not a Mac.** An M-series Mac is faster; read the ratios, not the absolute numbers.
- Node 22.22.0. All three libraries run as WebAssembly (Loro, Automerge) or plain JS (Yjs) inside Node.
- `loro-crdt` 1.16.4, `@automerge/automerge` 3.5.0, `yjs` 13.6.33 (latest on npm that day).
- Rust crates were not measured (optional; the JS bindings are what the app will call first).
- Each number is the **median of 5 runs**, each run in a fresh `node --expose-gc` process, unless marked "1 run". Memory = growth of `heapUsed + external` after load (WebAssembly memory counts as external); a cross-check with RSS is noted where it differs.

## Method

- **A. Rich-text page.** The real "automerge-paper" editing trace from [josephg/editing-traces](https://github.com/josephg/editing-traces) (`sequential_traces/automerge-paper.json.gz`): 259,778 single-character edits (182k inserts, 77k deletes) that end in a 104,852-character LaTeX paper with 1,173 lines. That is more ops but fewer paragraphs than the 150k / 2,000 target; it is the standard trace, so results compare with public ones. **One transaction per keystroke** (the realistic case for a live editor). Then: save, load in a fresh process, add 100 bold ranges.
- **B. Object graph.** 100,000 objects (title, type, status, priority, createdAt, updatedAt, dueAt and a `properties` map of 3 entries) and 200,000 edges stored as map entries `from>type>to → timestamp`. Built in transactions of 1,000 objects / 2,000 edges.
  - Layout 1: one big doc. Then 10,000 random single-field updates (status, priority or title), one transaction each, capturing each delta; then import the deltas one by one into a replica loaded from the base snapshot, and once more as a single batch. **Automerge did only 500 updates** (see why below).
  - Layout 2: one doc per object, holding the object and its outgoing edges; create and encode 100,000 of them, then load the first 1,000.
- **C. Concurrent merge.** A small doc (900-character paragraph + a map of 20 keys) is forked to two peers. Each does 1,000 random edits (60% text insert/delete in that same paragraph, 40% writes to the same 20 keys), one transaction each. They swap "updates since fork" and import. Convergence = identical JSON on both sides. 7 runs.
- **Move conflict.** Pages X, Y, Z under the root. Peer A moves X under Y; peer B concurrently moves Y under X. Second case: A moves X under Y, B moves X under Z. Loro uses its movable tree (`LoroTree`) and `LoroMovableList`. Yjs and Automerge have **no move operation**, so we test the two usual workarounds: a `parent` field per node, and a nested tree where move = copy + delete.
- **D. Sync payload.** On a doc with some history: bytes of the update for one typed character and for one changed map field. Loro: `export({mode:'update', from: versionBefore})`. Yjs: `encodeStateAsUpdate(doc, stateVectorBefore)`. Automerge: `saveSince(doc, headsBefore)`.

## Results

| | Loro 1.16.4 | Yjs 13.6.33 | Automerge 3.5.0 |
|---|---|---|---|
| **A. Rich text (paper trace, 260k ops)** | | | |
| Apply all ops, 1 transaction per op | **1.08 s** | 2.0 s | 37.8 s |
| Encode full doc | 24 ms | 18 ms | 48 ms |
| Size, full doc | 231 KB (all history) | 311 KB V1 / 160 KB V2 (**no history**: deleted text is dropped) | **129 KB** (all history) |
| Shallow / compacted snapshot | **65 KB** (shallow snapshot) | n/a, Yjs keeps current state only | not supported |
| Load in fresh process | **14 ms** (lazy) | 51 ms | 3.6 s (1.0–2.4 s when repeated warm) |
| Memory after load | +1.6 MB (RSS +3.6 MB) | +3.3 MB (RSS +10.5 MB) | **+207 MB** (RSS +229 MB) |
| Marks (bold) | yes, `mark`; 100 ranges 74 ms | yes, `format`; 24 ms | yes, `mark`; 46 ms |
| **B1. One big doc (100k objects + 200k edges)** | | | |
| Build | 8.7 s | **2.0 s** | 90 s |
| Encode | 3.4 s | 0.75 s | 3.9 s |
| Size (raw JSON of the data is 31.1 MB) | 27.2 MB; shallow 18.2 MB | 39.9 MB V1 / 28.7 MB V2 | **8.9 MB** |
| Load in fresh process | 0.19 s, lazy; full `toJSON` +10 s (1 run) | 3.4 s; `toJSON` +0.6 s | 13.7 s |
| Memory after load | +53 MB (lazy, before touching data) | +651 MB | +1,215 MB |
| One field update (local change) | 0.05 ms | 0.06 ms | **163 ms** |
| Delta size per change | 88 B | **39 B** | 152 B |
| Import one delta | 0.09 ms | 0.02 ms | 143 ms |
| Import all deltas as one batch | 4.3 s (10k) | 4.0 s (10k, via `mergeUpdates`) | 0.35 s (500) |
| **B2. One doc per object (100k docs)** | | | |
| Create + encode all | 21 s ¹ | **14.7 s** | 60 s |
| Total size | 81.7 MB snapshots (817 B each); 34 MB as `update` export (340 B each) | **33.6 MB** (336 B each) | 47.6 MB (476 B each) |
| Load 1,000 docs | **78 ms** | 218 ms | 385 ms |
| **C. Concurrent merge (1,000 + 1,000 edits)** | | | |
| Converged, 7 of 7 runs | yes | yes | yes |
| Merge time (both directions) | 14.5 ms | **4.7 ms** | 69 ms |
| Payload per side | 6.3 KB | 6.9 KB | 98 KB |
| **Move conflict (page tree)** | | | |
| X→Y vs Y→X | one move wins, Y > X on both peers, **no cycle** | no move op. Parent field: **cycle** X↔Y, both pages cut off from root. Copy+delete: **both X and Y lost** | same as Yjs: **cycle**, or **both lost** |
| X→Y vs X→Z | X ends under Z on both, **no duplicate** | parent field: last writer wins (fine). Copy+delete: **X duplicated** under Y and Z | same as Yjs |
| Concurrent move in a list | `LoroMovableList`: same item moved by both, ends once (`bacd`) | none (delete + insert duplicates) | none |
| **D. Sync payload** | | | |
| One character typed | 85 B | **15 B** | 93 B |
| One map field changed | 91 B | **22 B** | 128 B |

¹ Includes a second export per doc (the `update` mode size); about 30% of the time.

## Observations

1. **Automerge is too slow for this app in JS.** 37.8 s to replay a 105k-character page (Loro 1.1 s, Yjs 2.0 s), 3.6 s to load it and +207 MB of memory. Its files are the smallest (129 KB with full history), but the cost of getting there is high.
2. **Automerge local changes get slower as a map gets wider.** One field update costs 5.6 ms in a map of 5k objects, 24 ms at 20k and 163 ms at 100k: linear in the width of the parent map. Batching many changes into one `applyChanges` is fast (500 in 0.35 s), so the cost is per transaction. That is why B1 ran 500 updates for Automerge, not 10,000. The cause looks like the JS layer rebuilding its immutable view of the parent object (unverified). Sharding (many docs or nested maps) avoids it.
3. **Nobody should put 100k objects in one CRDT doc.** Yjs needs 651 MB and 3.4 s to load it; Automerge 1.2 GB and 13.7 s. Loro loads lazily in 0.19 s, but reading everything through the JS API is slow (`toJSON` 10 s; reading 100k fields one by one 7.3 s, 1 run). The object graph belongs in the local database; CRDT docs suit units that people edit together: a page, or one object.
4. **One doc per object works** for all three: 336–817 B per object, 1,000 docs load in 78–385 ms. Loro's snapshot format adds ~480 B of fixed overhead per doc; its `update` export is 340 B, on par with Yjs. So store small Loro docs as updates, or store field values in the database and keep a CRDT only for rich text.
5. **Yjs has the smallest sync payloads** (15 B per keystroke, 22 B per field change, vs 85–128 B for Loro and Automerge) and the fastest merges. But Yjs keeps **no history**: deleted text is garbage-collected, so no time travel, no "who changed what", no undo across sessions from the doc alone.
6. **Loro is the only one with a real move.** In both tree conflicts it converged to a valid tree with no cycle, no duplicate and no lost page. With Yjs or Automerge the app must build its own move on top: a parent field gives cycles (X under Y under X, both gone from the tree view), copy + delete loses or duplicates pages. A page tree with drag-and-drop between devices hits this.
7. **Loro has a shallow snapshot**: the paper shrinks from 231 KB to 65 KB, the graph from 27.2 to 18.2 MB, keeping current state and dropping old history. Automerge has no equivalent; Yjs is always "shallow".
8. **Loro JS pitfall: a detached container passed to `setContainer(key, new LoroMap())` holds ~7 KB of WebAssembly memory until `free()` is called or GC runs its finaliser.** Without freeing, building 100k per-object docs took 9 minutes per run instead of 21 s, and the big-doc build took about 2× longer (17–21 s vs 8.7 s). Any Loro code must free handles it creates in loops.
9. **Loro batch import was slower than one-by-one** for 10k small deltas into a 100k-object doc (4.3 s vs 0.9 s). Yjs `mergeUpdates` of 10k updates was also slow (4.0 s vs 0.2 s one by one). Catch-up after a long offline period should send a fresh snapshot or one combined update, not 10k tiny ones.
10. All three converged in every merge run, and all three support bold/italic marks on text.

## What this does not show

- Not a Mac, not a phone. Phone memory makes Automerge's numbers worse, not better.
- No Rust numbers. A Rust hub (if chosen) would be faster for Loro and Automerge; Yjs has `yrs`.
- The text is LaTeX, not Notion-like blocks. A block editor keeps each block in a list or tree container; this changes sizes a little, not the ranking.
- Automerge's sync protocol (`generateSyncMessage`) was not measured; it adds a Bloom-filter handshake on top of the change bytes.

## Script (core parts)

Full scripts: `a_text.mjs`, `b_graph.mjs`, `c_merge.mjs`, `common.mjs` (in the agent's scratch directory, not kept). Core:

```js
import * as A from '@automerge/automerge';
import { LoroDoc, LoroMap } from 'loro-crdt';
import * as Y from 'yjs';
// ---- A. replay the trace, one transaction per keystroke ----
// ops = [[pos, delCount, insText], ...] from automerge-paper.json
if (lib === 'loro') {
  const d = new LoroDoc(); const t = d.getText('t');
  for (const [p, del, ins] of ops) { if (del) t.delete(p, del); if (ins) t.insert(p, ins); d.commit(); }
  full = d.export({ mode: 'snapshot' });
  shallow = d.export({ mode: 'shallow-snapshot', frontiers: d.oplogFrontiers() });
} else if (lib === 'automerge') {
  let d = A.from({ t: '' });
  for (const [p, del, ins] of ops) d = A.change(d, x => A.splice(x, ['t'], p, del, ins));
  full = A.save(d);
} else {
  const d = new Y.Doc(); const t = d.getText('t');
  for (const [p, del, ins] of ops) d.transact(() => { if (del) t.delete(p, del); if (ins) t.insert(p, ins); });
  full = Y.encodeStateAsUpdate(d);            // V2: Y.encodeStateAsUpdateV2(d)
}
// load in a fresh process: LoroDoc.fromSnapshot(b) | A.load(b) | Y.applyUpdate(new Y.Doc(), b)
// marks: loro t.mark({start,end}, 'bold', true) | A.mark(x, ['t'], {start,end,expand:'after'}, 'bold', true)
//        | yjs t.format(start, len, {bold: true})

// ---- B. object graph, one big doc (Loro shown; Yjs uses Y.Map, Automerge plain objects) ----
const attach = (parent, key) => { const det = new LoroMap(); const h = parent.setContainer(key, det); det.free(); return h; };
const d = new LoroDoc(), os = d.getMap('objects'), es = d.getMap('edges');
for (let i = 0; i < 100_000; i++) {
  const m = attach(os, 'o' + i); for (const k of FIELDS) m.set(k, obj(i)[k]);
  const p = attach(m, 'properties'); for (const k in obj(i).properties) p.set(k, obj(i).properties[k]);
  p.free(); m.free(); if (i % 1000 === 999) d.commit();
}
for (let j = 0; j < 200_000; j++) { es.set(`o${rnd(N)}>${TYPE[j % 3]}>o${rnd(N)}`, 1.7e12 + j); if (j % 2000 === 1999) d.commit(); }
// 10k single-field updates, delta captured per transaction
let v = d.oplogVersion();
for (let k = 0; k < 10_000; k++) {
  os.get('o' + rnd(N)).set(['status', 'priority', 'title'][k % 3], value(k)); d.commit();
  deltas.push(d.export({ mode: 'update', from: v })); v = d.oplogVersion();  // take v AFTER commit:
}                                   // oplogVersion() already includes ops of the open transaction
// Yjs: d.on('update', u => deltas.push(u)); Automerge: A.getLastLocalChange(d) after each A.change
// import: replica.import(u) | Y.applyUpdate(replica, u) | [replica] = A.applyChanges(replica, [u])

// ---- C. move conflict (Loro) ----
const base = new LoroDoc(); const tr = base.getTree('pages');
const X = tr.createNode(), Yn = tr.createNode(), Z = tr.createNode(); base.commit();
const a = base.fork(), b = base.fork(); a.setPeerId(2n); b.setPeerId(3n);
a.getTree('pages').move(X.id, Yn.id); a.commit();      // X under Y
b.getTree('pages').move(Yn.id, X.id); b.commit();      // Y under X
a.import(b.export({ mode: 'update' })); b.import(a.export({ mode: 'update' }));
// both: Y > X, Z  (B's move is dropped: it would create a cycle)
// Yjs / Automerge workaround 1: nodes.X.parent = 'Y' vs nodes.Y.parent = 'X'  -> merged: cycle
// workaround 2: copy X into Y.kids and delete root.X, vs copy Y into X.kids and delete root.Y
//               -> merged: root has only Z; X and Y are both gone
```
