# Security model

> Part of [ADR-0001](../decisions/0001-founding-architecture.md). Status: **Proposed**. Sources are in [`research/05-security-dist.md`](research/05-security-dist.md) and [`research/06-hub-mcp-backup.md`](research/06-hub-mcp-backup.md).

## What we protect

Private notes, journal entries, tasks, saved pages and files; the user's AI keys (Claude, Voyage); the workspace key that encrypts sync and backups.

## Threats and controls

| Threat | Control |
|---|---|
| **Lost or stolen Mac** | FileVault (the app checks and warns if it's off). The app container is protected by macOS. Optional database encryption (SQLite3 Multiple Ciphers) with the key in the keychain, measured in spike S2. The workspace key never sits in a plain file. |
| **Breach of the sync server** | **End-to-end encryption.** The relay stores and orders ciphertext only. Every operation, snapshot and file chunk is sealed with the workspace key (XChaCha20-Poly1305, libsodium) and authenticated, so a malicious server can't read, forge or reorder unnoticed: each operation carries its device and counter inside the ciphertext, and devices check the sequence for gaps. What leaks is metadata: sizes, timing, counts. |
| **Breach of a trusted hub** | It holds the key by design, so it is as sensitive as the Mac. Defences: it lives on the user's own private network (no open ports), and it runs from code, so it can be rebuilt (see the ADR → Hub). |
| **Malicious saved-page or file content** (XSS into the app, then to IPC and keys) | The renderer never holds keys and has no Node. Saved pages render in a sandboxed `<iframe>` without `allow-same-origin`, from a separate session with no preload, under a strict CSP. SVG renders as `<img>`, PDFs in a sandboxed viewer. External links open only on click, and only http, https and mailto. |
| **Prompt injection from saved pages into the AI** | Break the "lethal trifecta" (private data + untrusted content + a way out). Summaries of web content run with **no tools**. No tool that reaches the network may run while web content is in the context. Model output never renders remote images or auto-links. Every write the AI proposes is a suggestion the user accepts (Sprint's rule). Web content is wrapped and marked untrusted, never put in the system prompt. |
| **SSRF from the link fetcher** (it now runs on the user's network) | Resolve once, refuse loopback, private, link-local, CGNAT and ULA addresses, connect to the pinned IP, and re-check on every redirect; allow only http(s); cap size and time. macOS's Local Network prompt is not a defence: it doesn't cover localhost. Trilium shipped this exact bug in 2026. |
| **Malicious dependency** (npm worms 2025–26: chalk/debug, Shai-Hulud, axios, TanStack, ChainDrop) | pnpm 11 defaults (`minimumReleaseAge` ≥ 1 day, build scripts blocked unless allow-listed, `blockExoticSubdeps`), frozen lockfile, few dependencies in the core process. GitHub Actions pinned by commit SHA, no `pull_request_target`, signing secrets only in a protected release environment. Changes under `.claude/` reviewed in PRs (ChainDrop planted Claude Code hooks). |
| **Malicious update** (the Notepad++ hijack, 2025) | Updates are verified against keys built into the app: the Apple code signature (Squirrel.Mac checks it) plus our own Ed25519 signature on the feed. No downgrades. The update key is kept offline with a backup. |
| **Compromised renderer reaching the core** | Electron fuses: `RunAsNode` off, `NodeOptions` off, ASAR integrity and only-load-from-ASAR on. Context isolation and sandbox on. The core process checks the sender of every message, and the API is typed and narrow (no "run SQL"). |
| **An AI agent or MCP client doing damage** | See below. |
| **Losing the key = losing the data** | A recovery key (printed or saved, 24 words), plus every device as a key holder, plus encrypted backups. The app shows how many key copies exist. |

## Keys

```
workspace key (random 256-bit)  ── seals ──▶ operations, snapshots, file chunks, backups
   ├─ wrapped by each device's key   → macOS keychain (data protection keychain, this device only)
   ├─ wrapped by the recovery key    → shown once, 24 words
   └─ wrapped by a passkey (PRF)     → web client only (Safari 18+, Chrome, Firefox 139+ on macOS)
AI keys (Claude, Voyage)            → keychain; read only by the core process
```

- **Keychain.**
  - Use the **data protection keychain** (Apple TN3137), not the legacy file keychain that Electron's `safeStorage` uses. Code running inside the app could read the legacy keychain without a prompt.
  - It is reached through a small signed native helper (Swift, N-API), which spike S4 proves under Developer ID.
  - Optionally the device key is wrapped by a Secure Enclave key and protected by Touch ID.
- **Never derive the workspace key from a passkey directly.** Passkeys get lost, and PRF output differs over hybrid sign-in. The key is random; passkeys and the recovery key only wrap it.
- **Sharing a workspace later** (teams, selling) means wrapping the workspace key for another person's public key. The model allows it; nothing is built for it in R5.

## MCP and agents

The app exposes its graph to Claude and other agents through an **MCP server** (spec 2026-07-28, TypeScript SDK v2). It runs over stdio on the Mac, packaged as an `.mcpb` bundle for Claude Desktop and Claude Code. Over HTTP it runs only on the private network, with Origin and Host checks and a per-client token.

| Rule | Why |
|---|---|
| **Scopes per client:** read-only by default; writes granted per tool (`tasks.create`, `pages.append`…), never "run SQL" | 404 GitHub-reviewed MCP advisories by 2026-10, mostly command injection and open HTTP endpoints |
| **Destructive or bulk actions need a confirmation in the app** (archive, delete, close a sprint, edits to more than 20 objects) | An agent can be talked into anything |
| **Every MCP call becomes an operation with `actor = agent:<client>`** in the event log | Audit: "what did the agent do?" is a timeline filter, and each change can be undone like any other |
| **Content from saved pages and files is marked untrusted** in tool results | The realistic attack is injection through stored content |
| **No outbound network tools in the same server** | Cuts the exfiltration leg |

**Messaging channels** (Telegram and others) are optional adapters on the trusted hub:
- capture and chat only;
- a chat-ID allow-list;
- confirmation for writes;
- no shell tools.

Telegram bot messages are not end-to-end encrypted, so the setup screen says so. We don't embed OpenClaw: it has 763 GitHub-reviewed advisories. n8n's licence forbids bundling it in a paid product.

## Telemetry

None by default. Crash reports are asked for once and are crash-only, scrubbed of titles, URLs and text, with no session replay. "Copy diagnostics" is the offline alternative.
