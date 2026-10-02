# Architecture research

Supporting research and benchmarks for [ADR-0001](../decisions/0001-founding-architecture.md). Written 2026-10-02. Web sources carry their own dates; facts that couldn't be checked are marked "unverified".

| Note | What it covers |
|---|---|
| [domain-requirements.md](domain-requirements.md) | What Sprint's product really needs: types, relations, integrity rules, jobs, AI calls, volumes, lessons, what to port |
| [research-shell-ui-editor.md](research-shell-ui-editor.md) | App shell, where the logic lives, UI framework, the editor judged on CRDT fit |
| [research-storage.md](research-storage.md) | SQLite, PGlite, embedded Postgres and others; integrity, encryption at rest, schema evolution, many tenants |
| [research-sync.md](research-sync.md) | CRDTs, local-first frameworks, server sync engines, the domain's conflict cases, the recommended design |
| [research-topology-security.md](research-topology-security.md) | Mac plus optional hub, networking, threat model, keys, sandboxing, supply chain, identity and tenancy |
| [research-ai-jobs.md](research-ai-jobs.md) | AI gateway, embeddings, local models, the agent, MCP, messaging, background jobs, automations, crash reports |
| [research-files-ops-licence.md](research-files-ops-licence.md) | Files, backups, distribution, hub deployment, telemetry, testing, licensing and business |
| [bench-storage.md](bench-storage.md) | Measured: SQLite vs PGlite vs Postgres at 100k objects, with scripts |
| [bench-crdt.md](bench-crdt.md) | Measured: Loro vs Automerge vs Yjs, with scripts |
| [bench-embeddings.md](bench-embeddings.md) | Measured: local embedding models on CPU, plus published scores and Apple Silicon numbers |

The benchmarks ran on a shared 4-vCPU x86 Linux VM, not a Mac. Treat absolute numbers as a pessimistic bound and compare engines by ratio; spike S3 reruns them on Apple Silicon.
