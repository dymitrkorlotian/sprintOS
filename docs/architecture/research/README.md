# Research notes for ADR-0001

Gathered on 2026-10-02 for the [founding architecture](../../decisions/0001-founding-architecture.md). Each note cites a source and a date for its claims and marks how sure it is:

- **Verified:** read directly from the source (a package registry, a GitHub repo, Apple's docs).
- **Search summary:** a search engine's summary of a page that couldn't be opened, because the research environment's proxy blocked many vendor sites.
- **Unverified:** stated, but not checked against a source in that session.

Paths such as `/home/user/sprint` refer to the Sprint repo as checked out during the research.

| Note | Topic |
|---|---|
| [01-shell.md](01-shell.md) | Desktop shells (Tauri, Electron, native, others), UI framework, real local-first apps |
| 02 | The local-database note was stopped before it was written; the measured comparison is in [`../benchmarks.md`](../benchmarks.md) |
| [03-sync.md](03-sync.md) | CRDTs, sync engines, op-log designs, CloudKit, E2EE with a web app |
| [04-editor-ai.md](04-editor-ai.md) | Block editors and CRDT bindings; local embeddings and LLMs; bring-your-own-key; prompt injection |
| [05-security-dist.md](05-security-dist.md) | Apple distribution, updates, keys, webview hardening, supply chain, telemetry, licences, threat model |
| [06-hub-mcp-backup.md](06-hub-mcp-backup.md) | Private networking (Iroh, Tailscale), MCP, backups, always-on agents, Mac background jobs |
