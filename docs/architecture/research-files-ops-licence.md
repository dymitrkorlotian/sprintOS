# Research: files, backups, distribution, operations, testing, licence

Input to ADR-0001. Facts checked on 2026-10-02 unless a date is given. "Unverified" means I could not confirm it from a primary source.

Scores are 1 (bad) to 5 (good). Columns: Perf = performance, Sec = security, Stab = stability/maturity, Dev = dev speed, Fit = fit to sprintOS, Cost = running cost for the user, Lock = freedom from lock-in.

---

## 1. Files

### 1.1 Blob store on disk

Design: content-addressed store, one file per blob, named by its BLAKE3 hash, under the app's data folder (`~/Library/Application Support/<app>/blobs/ab/cd/<hash>`). The DB keeps only metadata: `file` object (name, mime, size, created, owner, tenant) -> `blob_hash`. Same bytes = same blob, so dedupe is free. Delete is "unreference, then GC blobs with zero refs after a grace period".

- **BLAKE3**: fast (SIMD, multi-threaded), 256-bit, a Rust crate and C lib, CC0/Apache-2.0. Hash once on import; the hash is also the integrity check on read and after sync. SHA-256 is the boring fallback if a FIPS-like need ever comes up.
- **Write path**: write to a temp file in the same volume, `fsync`, rename into place (atomic on APFS), then insert the DB row. A crash leaves at worst an orphan blob, which GC removes. Never the reverse (a DB row without a blob).
- **APFS clones**: on import from Finder, `clonefile()` costs no extra disk until the user edits the original. Worth using.

| Option | Perf | Sec | Stab | Dev | Fit | Cost | Lock | Note |
|---|---|---|---|---|---|---|---|---|
| CAS dir + BLAKE3 + metadata in DB | 5 | 4 | 5 | 4 | 5 | 5 | 5 | Recommended |
| Blobs inside SQLite (BLOB column) | 3 | 4 | 5 | 5 | 2 | 5 | 4 | Bloats DB, backups and WAL; fine only for <100 KB thumbnails |
| User-visible folder tree (Obsidian style) | 4 | 3 | 4 | 3 | 3 | 5 | 5 | Nice "open format", but renames/moves fight sync; could be a later "mirror" view |

### 1.2 Chunked sync of large blobs

- Small blobs (< ~4 MB): send whole, keyed by hash. Most notes attachments are small.
- Large blobs: split with **FastCDC** (content-defined chunking; `fastcdc` Rust crate, MIT). Each chunk is BLAKE3-named; the blob is a manifest (list of chunk hashes). Edits to a big file then resend only changed chunks, and uploads are resumable for free: the receiver says which chunk hashes it already has.
- This is the same model restic, Kopia and rustic use, so it is well proven.
- Sync of blobs is lazy: metadata syncs eagerly through the normal sync path; bytes are fetched on open, or prefetched on Wi-Fi/power for pinned items. Phones must not pull everything.

### 1.3 Blob encryption

- If the hub/bucket is a blind store (E2EE), encrypt each chunk before upload with **XChaCha20-Poly1305**. Per-blob key derived from the content hash plus a vault secret (convergent encryption keyed by a secret: `key = HKDF(vault_key, blob_hash)`), so dedupe still works inside one vault but nobody without the vault key can confirm a file's presence. Name the object in the bucket by `HMAC(vault_key, blob_hash)`, not the raw hash, to avoid leaking "has this known file".
- Alternative: random per-blob key stored in the (synced, encrypted) DB row. Stronger (no convergence), but loses cross-device dedupe until metadata arrives. Fine; pick during the sync spike.
- Local disk: rely on FileVault. Do not encrypt local blobs by default (QuickLook, Spotlight and thumbnails need plaintext). Keys live in the macOS Keychain.

### 1.4 Thumbnails, previews, Spotlight

- **QuickLook**: `QLThumbnailGenerator` makes thumbnails for any type macOS knows; `QLPreviewPanel`/`QLPreviewView` shows previews. Free, native, sandbox-safe. Cache thumbnails as small blobs keyed by `(blob_hash, size)`.
- **Core Spotlight**: index tasks, notes, files via `CSSearchableIndex` with a domain id per type; tapping a result deep-links into the app. Works on macOS and iOS and (since 2024) feeds Apple's semantic search ([WWDC24 "Support semantic search with Core Spotlight"](https://developer.apple.com/videos/play/wwdc2024/10131/)). Cheap win; must respect the "private" flag and deletes. It is a convenience, not our search engine.

### 1.5 iCloud Drive / Dropbox pitfall

Never keep the live SQLite DB (or the blob store) in iCloud Drive, Dropbox, OneDrive or a synced "Desktop & Documents". The sync client copies `db`, `-wal` and `-shm` one file at a time at arbitrary moments and ignores SQLite locks, which corrupts or rolls back data. SQLite's own list of causes includes file-copy during a transaction and broken locking on network/synced filesystems ([sqlite.org/howtocorrupt.html](https://www.sqlite.org/howtocorrupt.html)); WAL contents are committed data, not scratch ([sqlite.org/wal.html](https://www.sqlite.org/wal.html)); a practitioner write-up on agent state DBs repeats the "synced folder is a corruption machine" finding ([dev.to, 2026](https://dev.to/milkyway008/your-agents-sqlite-state-db-keeps-corrupting-what-actually-causes-it-and-how-to-recover-the-data-55hd)). Rules:
- Store under `~/Library/Application Support` (or the sandbox container), which iCloud "Desktop & Documents" does not touch.
- If a user picks a custom location, refuse cloud-synced paths (check `NSURLIsUbiquitousItemKey` and known Dropbox/OneDrive paths) and explain why.
- If they want a copy in iCloud, write a snapshot (`VACUUM INTO`) there, never the live file.

### 1.6 Where blobs live when no hub is present

S3-compatible bucket the user owns, chosen by the user, written directly by the app (encrypted as in 1.3):

| Bucket | Free tier | Paid | Egress | Note |
|---|---|---|---|---|
| Cloudflare R2 | 10 GB-month, 1M Class A, 10M Class B ops/month | $0.015/GB-month, Class A $4.50/M | free | [R2 pricing summaries, 2026](https://egresscost.com/cloudflare/), [freetier.co](https://freetier.co/directory/products/cloudflare-r2) |
| Backblaze B2 | first 10 GB free | $6.95/TB-month | free up to 3x stored, then $0.01/GB; Class A-C calls free | [backblaze.com/cloud-storage/pricing](https://www.backblaze.com/cloud-storage/pricing) |

R2 fits "about $0" best for one person (10 GB, no egress). B2 is cheaper per TB above that. Both need a card on file. Treat the bucket as a dumb blob store; it can also hold the backup repository (section 2).

---

## 2. Backups (not export)

Backups are for "my Mac died / I deleted a sprint". They are encrypted, versioned, automatic, and restored by the app itself.

### 2.1 Tools

| Tool | Status (2026-10) | Encryption | Backends | Note |
|---|---|---|---|---|
| **restic** | 0.19.0 (2026-06-09), 0.19.1 (2026-07) ([blog](https://restic.net/blog/2026-06-09/restic-0.19.0-released/), [releases](https://github.com/restic/restic/releases)) | AES-256-CTR + Poly1305, always on | local, SFTP, rest-server, S3 (R2/B2), B2, rclone | BSD-2. Go binary. Very mature, huge user base. Format stable with compression (v2 repo). |
| **rustic** | 0.11.x ([releases](https://github.com/rustic-rs/rustic/releases)) | same as restic (repo-compatible) | same + OpenDAL | Apache/MIT. Rust **library** (`rustic_core`) we could embed. Still beta-labelled; "compatible with restic" ([comparison](https://rustic.cli.rs/docs/comparison-restic.html)). |
| **Kopia** | 0.21.1 (2026-07-22) ([releases](https://github.com/kopia/kopia/releases)) | AES-GCM / ChaCha20 | local, SFTP, S3, B2, GCS, WebDAV, rclone | Apache-2.0. Good GUI, server mode. Not restic-compatible. |
| **BorgBackup** | 1.4 stable; 2.0 still beta (2.0.0b25, 2026-09) ([releases](https://github.com/borgbackup/borg/releases)) | AES/ChaCha | SSH only (borg on server) | Needs borg on the far side; no S3. Poor fit. |
| **Litestream** | v0.5.0 (late 2025): LTX format, PITR, compaction, pure Go ([Fly blog](https://fly.io/blog/litestream-v050-is-here/), [how it works](https://litestream.io/how-it-works/)) | relies on bucket (SSE); no client-side E2EE built in (unverified for v0.5) | S3, GCS, Azure, SFTP, file | Continuous WAL shipping, seconds of RPO. One DB file, not blobs. |
| **sqlite3_rsync** | in SQLite since 3.47; WAL/page-size limits lifted in 3.50.0 (2025-05-29) ([sqlite.org/rsync.html](https://sqlite.org/rsync.html)) | none (SSH transport) | SSH | Live DB to live replica copy; not versioned, not a backup by itself. |
| **Online Backup API / `VACUUM INTO`** | core SQLite | n/a | local file | The right way to take a consistent snapshot of a live DB. |

### 2.2 SQLite consistency rules

- Never copy `db` + `-wal` with a file tool while the app runs. Take a snapshot with `VACUUM INTO 'snap.db'` (compact, consistent, one file) or the Backup API, then back up the snapshot.
- **Time Machine** copies files while the app runs, so a TM restore of a live WAL DB may be torn. Mitigation: exclude the live DB and `-wal/-shm` from TM (`NSURLIsExcludedFromBackupKey` / `tmutil addexclusion`) and instead write an hourly `VACUUM INTO` snapshot into a folder TM does back up. Blobs are immutable, so TM handles them fine.
- If the DB is SQLCipher-encrypted, `VACUUM INTO` keeps the encryption (with the same key) – check in the spike.

### 2.3 Design

1. Every N minutes when changed (default 60) and at quit: `VACUUM INTO` a snapshot.
2. Back up `{snapshot, blobs/}` with an embedded restic-format engine to one or more targets: hub (rest-server endpoint on the hub, append-only mode), the user's S3 bucket, or an external disk. Blobs dedupe naturally because they are already content-addressed.
3. Retention: hourly 48, daily 30, weekly 12, monthly 24 (restic `forget --keep-*` semantics).
4. Repository password in Keychain, plus a printed/recovery-key flow. Losing it = losing backups; say so in the UI.
5. **Automated restore test**, weekly: restore latest snapshot to a temp dir, open it, `PRAGMA integrity_check`, compare row counts and a sample of blob hashes against the live DB, show "Last verified restore: 3 days ago" in Settings. Plus `restic check --read-data-subset=5%` monthly.
6. Hub, if present, runs its own backup of its own DB the same way (cron/systemd timer), to a bucket.

Litestream is a good **addition on the hub** (seconds of RPO for the server DB) but not the main backup: one DB only, no blobs, no client-side encryption.

| Option | Perf | Sec | Stab | Dev | Fit | Cost | Lock |
|---|---|---|---|---|---|---|---|
| restic CLI bundled as a helper | 4 | 5 | 5 | 4 | 4 | 5 | 5 |
| rustic_core embedded (restic format) | 5 | 5 | 3 | 3 | 5 | 5 | 5 |
| Kopia | 4 | 5 | 4 | 4 | 3 | 5 | 4 |
| Litestream only | 5 | 3 | 4 | 5 | 2 | 5 | 4 |
| Time Machine only | 3 | 4 | 4 | 5 | 2 | 5 | 3 |

**Recommendation: the restic repository format.** Start by bundling the signed `restic` binary as a helper (proven, simple). Move to `rustic_core` in-process once it is out of beta, keeping the same repos. Hub additionally runs Litestream to the bucket. Time Machine is a bonus layer with the exclusion rule above.

What would change it: rustic declares 1.0 (embed from day one); a Mac App Store build (a bundled helper binary must be signed and sandboxed; still allowed, but simpler in-process); Litestream adding client-side encryption (then use it for the Mac too).

---

## 3. Distribution (Mac)

### 3.1 Signing basics

- Apple Developer Program: $99/year. Needed for Developer ID, notarisation, the App Store and TestFlight.
- Direct download: sign with **Developer ID Application** (hardened runtime), submit with `xcrun notarytool submit --wait`, then `xcrun stapler staple` the `.app`/`.dmg` so Gatekeeper passes offline ([notarytool man page](https://keith.github.io/xcode-man-pages/notarytool.1.html)). All in CI (GitHub Actions macOS runner, App Store Connect API key).

### 3.2 Mac App Store vs direct

| Topic | Mac App Store | Direct (Developer ID) |
|---|---|---|
| Sandbox | required | optional (we can still opt in) |
| Helper processes (hub binary, restic, AI agent tools) | only if embedded, signed, sandboxed, inheriting the sandbox | free |
| Login items / background agent | `SMAppService` (allowed) | `SMAppService` or launchd |
| Local server / LAN peer | needs `com.apple.security.network.server` + local network prompt; fine | fine |
| File access outside container | user picks via open panel; keep via security-scoped bookmarks | anything, with TCC prompts |
| Running code at runtime | Guideline 2.5.2 forbids downloading/executing code that changes functionality; Apple enforced it on AI app-builders in March 2026 ([guidelines](https://developer.apple.com/app-store/review/guidelines/), [Gadget Hacks](https://apple.gadgethacks.com/news/apple-app-store-takedown-lawsuit-explained-guideline-252-and-ai-dev-apps/)) | no such rule |
| Updates | App Store | our own (Sparkle/Tauri) |
| Revenue cut | 15% (small business) / 30% | 0% + payment processor |
| Review delay & risk | days; risk of rejection for agent features | none |

Review risks for us: an AI agent that runs shell commands or scripts, user-defined automations, a plugin system, or a bundled server can trip 2.5.2 or sandbox rules. Data-only automations (no code) are fine.

**Recommendation: direct download first** (Developer ID + notarisation + auto-update), built sandbox-friendly anyway (security-scoped bookmarks, `SMAppService`, no writes outside the container) so an App Store build stays possible later with agent tools trimmed. Homebrew cask as a second channel (`brew install --cask`; cask points at our notarised DMG; auto-updated apps set `auto_updates true`).

### 3.3 Auto-update

| Option | Perf | Sec | Stab | Dev | Fit | Note |
|---|---|---|---|---|---|---|
| **Sparkle 2** | 5 | 5 | 5 | 4 | 5 | MIT. EdDSA-signed appcasts, binary delta updates, XPC services for sandboxed apps ([sparkle-project.org](https://sparkle-project.org/)). Recent fixes for TCC bypass via the downloader XPC (2.7.3) and a root-helper LPE (2.9.6) ([security notes](https://sparkle-project.org/documentation/security-and-reliability/)): keep it current. |
| Tauri updater plugin | 4 | 4 | 4 | 5 | 4 | Minisign signature, cannot be disabled; ships `App.app.tar.gz` + `.sig`; static JSON on any host ([v2.tauri.app/plugin/updater](https://v2.tauri.app/plugin/updater/)). No deltas: full download every update. |
| Electron `autoUpdater` / electron-updater | 3 | 4 | 4 | 5 | 3 | Squirrel.Mac; requires signed app; large downloads (whole Chromium). |

Pick follows the shell choice (another note). If the shell is native or Tauri, Sparkle 2 is the best-in-class Mac updater and works with any `.app`; Tauri's updater is acceptable and simpler. Host the appcast and archives on R2 or GitHub Releases ($0).

### 3.4 iOS later

App Store only for normal users (EU alternative marketplaces exist but are a niche). TestFlight for beta (up to 10,000 external testers, builds expire after 90 days). iOS kills background work, so the phone must be a thin peer that syncs on open/push; it cannot be the hub. Guideline 2.5.2 applies fully on iOS.

---

## 4. Hub deployment from code

Goal: "a €70 home server running an always-on agent, Tailscale, restic, everything as code", but never required.

### 4.1 Packaging

- **One OCI image**: static Rust binary on `gcr.io/distroless/static` (or `scratch` + CA certs), non-root, read-only root FS, one volume `/data` (DB + blobs). Multi-arch (amd64 + arm64). Image signed with cosign, SBOM attached.
- **docker compose** file: hub + optional `tailscale` sidecar + optional `litestream` sidecar.
- **NixOS module**: `services.sprintos-hub = { enable; dataDir; tailscale; backup.restic = {...}; }`, reusing `services.restic.backups`. Good for the "everything as code" crowd.
- **cloud-init bootstrap**: one script that installs Docker (or the bare binary + systemd unit), joins Tailscale with an auth key, sets up restic timer to the user's bucket, enables unattended-upgrades. Works on Hetzner, any VPS, a Raspberry Pi.
- **Health endpoint**: `GET /healthz` (process up) and `GET /status` (version, DB size, last sync per device, last backup, last restore test, disk free). The Mac app shows this as a hub card with a red/amber/green dot.

### 4.2 Where to run it

| Host | Price (2026-10) | Reliability | Note |
|---|---|---|---|
| Hetzner CAX11 (2 vCPU Arm, 4 GB) | €5.99/month excl. IPv4, new orders after 2026-06-15 (was €4.49) | good | Two price rises in 2026 (April, June 15) ([Northflank breakdown](https://northflank.com/blog/hetzner-cloud-server-price-increases), [wz-it](https://wz-it.com/en/blog/hetzner-price-increase-june-2026-cpx-ccx-alternatives/)). Hetzner's own page was blocked to me: verify before quoting to users. |
| Hetzner CX23 (2 vCPU x86, 4 GB; replaced CX22) | €5.49/month excl. IPv4 (was €3.99) | good | Same sources. IPv4 adds a small monthly fee; IPv6-only + Tailscale avoids it. |
| Fly.io | no free tier for new accounts since 2024; 7-day/2-VM-hour trial, then pay as you go ([srvrlss.io](https://www.srvrlss.io/provider/fly/)) | good | Not $0. |
| Oracle Cloud Always Free (Ampere A1) | $0 | weak | Idle instances get reclaimed (95th-pct CPU < 20% over 7 days); capacity often unavailable; blogs report the allowance halved to 2 OCPU/12 GB in June 2026 ([terminalbytes](https://terminalbytes.com/oracle-cloud-free-tier-changes-2026/)) – unverified with Oracle. A hub idles most of the time, so it is exactly what gets reclaimed. |
| Mac mini / home box | ~€0 extra | depends on home | The "€70 home server" model. |
| Our hosted hub (later) | paid | ours | Business, section 7. |

Honest conclusion: "about $0" is met by **no hub** (Mac + bucket) or **a box the user already owns**. The cheapest reliable VPS is about €6/month now. Oracle Free is a "try it" option with a warning, not a recommendation.

### 4.3 Tailscale

Personal plan: free, 6 users, unlimited user devices, 50 tagged resources (checked 2026-08-17 per [SSD Nodes](https://www.ssdnodes.com/learn/is-tailscale-free-plan-limits)). Use it as the default private network between Mac, phone and hub: no open ports, WireGuard, MagicDNS, HTTPS certs via `tailscale cert`. Make it optional (plain TLS + device keys must also work), and keep Headscale as the escape hatch from the vendor. Do not embed `tsnet` in the Mac app at first; rely on the user's Tailscale app.

---

## 5. Telemetry

- **None by default.** No analytics SDK, no phone-home except the update check (which sends app version and OS version only; document it, allow turning it off).
- Free from Apple without any SDK: crash reports and energy reports via Xcode Organizer/App Store Connect for App Store and TestFlight builds, only from users who opted in at OS level. For direct downloads, Apple gives nothing; add **opt-in** crash reporting (self-hosted Sentry-compatible endpoint, or just "Send report" that opens a prefilled email/issue with the `.ips` file).
- Opt-in, anonymous, aggregate usage counters only if a real question needs them; show the exact payload before sending.
- AI calls go to the user's chosen model provider with their key; say so plainly in the privacy page.

---

## 6. Testing strategy

| Layer | Tooling | What it proves |
|---|---|---|
| Unit | `cargo test` + `cargo nextest`; Vitest for TS | logic, schema, ontology rules |
| Property | **proptest** (Rust), fast-check (TS) | CRDT/sync laws: convergence, commutativity, idempotence of merges; any op order gives same state |
| Fuzz | `cargo fuzz` (libFuzzer), run in CI nightly | parsers, sync wire format, blob manifests, import |
| Deterministic simulation | **turmoil** or **madsim** for tokio code ([turmoil docs](https://docs.rs/turmoil), [madsim docs](https://docs.rs/madsim)), FoundationDB/TigerBeetle style: seeded RNG, fake clock, fake network with drops, partitions, reorders, restarts | sync correctness under failure; failures replay from a seed |
| Jepsen-style checks | own checker on simulation histories (e.g., "every acked write is on all peers after heal") | no lost acked writes |
| Antithesis | sales-only, no self-serve ([qaskills](https://qaskills.sh/blog/antithesis-deterministic-simulation-testing)) | later, if the sync engine becomes the product |
| Crash consistency | harness that runs writes in a child process and `kill -9` at random points (and at fsync boundaries via a failpoint crate), then reopens: `PRAGMA integrity_check`, invariants, blob/DB agreement | SQLite + blob store + sync journal survive power loss |
| Backup | CI job: create data, back up, wipe, restore, diff | restore actually works (same code as the in-app weekly test) |
| Desktop e2e | see below | real app flows |
| Native UI | XCUITest, if the shell has native parts | menus, Share extension, Spotlight deep links |

**Desktop e2e for Tauri on macOS:** the official `tauri-driver` supports only Windows and Linux on desktop because macOS has no WKWebView driver ([v2.tauri.app WebDriver](https://v2.tauri.app/develop/tests/webdriver/)). Options on macOS: WebdriverIO's Tauri service with an embedded WebDriver plugin inside the app (`tauri-plugin-wdio-webdriver`) ([webdriver.io platform support](https://webdriver.io/docs/desktop-testing/tauri/platform-support/)); CrabNebula's `@crabnebula/tauri-driver` (proprietary macOS part) ([npm](https://www.npmjs.com/package/@crabnebula/tauri-driver)); community `tauri-webdriver` ([GitHub](https://github.com/danielraffel/tauri-webdriver)). Practical plan: run most UI e2e with **Playwright against the web build** of the same frontend (fast, any OS), plus a thin macOS smoke suite via WebdriverIO embedded driver. Electron has first-class Playwright support, which is one point in its favour in the shell decision.

CI: GitHub Actions; macOS runner for signing, notarisation and the smoke suite; Linux for everything else, including hub image tests.

---

## 7. Licensing and business

### 7.1 What similar products chose

| Product | Licence | Business |
|---|---|---|
| Obsidian | closed (free app) | Sync $4/month (annual, 1 GB) or $5 monthly; Plus $8/month (10 GB) ([Obsidian blog](https://obsidian.md/blog/standard-plan/)), Publish, commercial licence |
| Logseq | AGPL-3.0 | sync service (beta) |
| Anytype | protocol MIT; apps "Any Source Available" (non-commercial) ([Anytype blog](https://blog.anytype.io/our-open-philosophy/)) | paid storage/network tiers |
| AppFlowy | AGPL-3.0 | cloud plans |
| AFFiNE | MIT for editor and app; backend under "AFFiNE EE" source-available ([discussion](https://github.com/toeverything/AFFiNE/discussions/5947)) | cloud + team plans |
| Standard Notes | AGPL-3.0 | E2EE sync subscription |
| Bitwarden | clients GPL-3.0, server AGPL-3.0, some features under Bitwarden License (source-available) | subscriptions |
| Zed | editor GPL-3.0, server AGPL-3.0, GPUI Apache-2.0 | paid AI/collab |

Pattern: the ones that earn money sell **sync/hosting**, not the client. Licence choice mostly decides who may run a competing hosted service.

### 7.2 Options

| Licence model | Trust | Stops hosted clones | Contributions | Dep. compatibility | Simplicity |
|---|---|---|---|---|---|
| Closed | 2 | 5 | 1 | 5 | 5 |
| MIT/Apache-2.0 | 5 | 1 | 5 | 4 | 5 |
| MPL-2.0 | 5 | 1 | 4 | 4 | 4 |
| GPL-3.0 | 4 | 2 | 3 | 3 | 3 |
| AGPL-3.0 | 4 | 4 | 3 | 3 | 3 |
| Open core (MIT core + closed hub features) | 3 | 3 | 3 | 4 | 2 |
| FSL-1.1-ALv2 (becomes Apache-2.0 after 2 years per release) ([fsl.software](https://fsl.software/)) | 4 | 5 | 2 | 4 | 4 |
| BSL (default 4 years) | 3 | 5 | 2 | 4 | 3 |

AGPL and GPL need a CLA (or DCO plus copyright assignment) if we ever want to dual-license or sell an App Store build. GPL code on the Mac App Store has a known conflict (Apple's terms add restrictions); owning all copyright avoids it.

### 7.3 Dependency licences that matter

| Dependency | Licence | Effect |
|---|---|---|
| BlockNote core | MPL-2.0 | fine for any model |
| BlockNote `@blocknote/xl-*` (AI, multi-column, exporters) | GPL-3.0 OR commercial (Business subscription) ([pricing](https://www.blocknotejs.org/pricing), [npm xl-ai](https://www.npmjs.com/package/@blocknote/xl-ai)) | using them forces GPL-compatible app or a paid licence; avoid or pay |
| Tiptap core | MIT; 10 former Pro extensions MIT since June 2025 ([Tiptap blog](https://tiptap.dev/blog/release-notes/were-open-sourcing-more-of-tiptap)); collab/AI/conversion still paid | fine; avoid paid cloud parts |
| Loro, Automerge, Yjs | MIT | fine |
| SQLite | public domain | fine |
| SQLCipher Community | BSD-style, needs attribution | fine |
| sqlite-vec | MIT/Apache-2.0 | fine |
| Sparkle | MIT | fine |
| restic | BSD-2-Clause; rustic Apache/MIT | fine |
| Tailscale client | BSD-3 (client); service is SaaS | fine |

### 7.4 Business

- Revenue later: **hosted hub + sync + backup** as a subscription (benchmarks: Obsidian Sync $4–8/month; Standard Notes, Anytype similar range). Self-hosted hub stays free.
- Many-tenant data model is already decided, so a hosted hub is a matter of ops.
- **Trademarks**: "Sprint" is a weak, crowded mark (former US carrier, Scrum term). Check "Sprint OS" in EUIPO/USPTO before any public launch; register the final name and logo; keep the trademark separate from the code licence (a trademark policy like Mozilla's or Zed's lets forks exist under other names).

### 7.5 Recommendation

**Client and hub under FSL-1.1-ALv2** (source available now, Apache-2.0 two years after each release), all copyright owned by the user (CLA for outside contributions), trademark registered. Reasons: shows the code (trust, security review), lets anyone self-host the hub for free, blocks a third party from selling our hosted hub for two years, converts to a real open-source licence so users are never stranded, and has no copyleft friction with the App Store or commercial deps. Libraries we split out (sync protocol, file format, CLI) go **MIT/Apache-2.0** to encourage reuse, like Anytype's protocol.

What would change it: wanting outside contributors and an "open source" label from day one -> **AGPL-3.0 + CLA** (the Standard Notes/Logseq path). No interest in a hosted business -> MIT/Apache. Planning a VC-funded company with enterprise features -> open core. Using BlockNote xl-* without paying -> must be GPL-3.0-compatible (AGPL or GPL), which rules out FSL.

---

## 8. Summary of recommendations

1. Files: CAS dir with BLAKE3, metadata in SQLite, FastCDC chunks for big blobs, lazy blob sync, XChaCha20-Poly1305 per-chunk encryption with keyed (HMAC) object names for blind stores; QuickLook + Core Spotlight; DB never in iCloud/Dropbox.
2. Hubless blob store: user's R2 bucket (10 GB free, zero egress); B2 for bigger sets.
3. Backups: `VACUUM INTO` snapshot + restic-format repo (restic helper now, rustic_core later) to hub/bucket/disk; weekly automatic restore test shown in UI; Litestream on the hub; TM exclusion for the live DB.
4. Distribution: Developer ID + notarytool + staple, direct download, Sparkle 2 (or Tauri updater if Tauri), Homebrew cask; App Store build only later and sandbox-ready from the start.
5. Hub: distroless OCI image, compose, NixOS module, cloud-init script; Tailscale default; `/status` in the app. Cheapest reliable VPS is now ~€5.50–6/month (Hetzner); $0 only on own hardware or Oracle Free (unreliable).
6. Telemetry: none by default; opt-in crash reports.
7. Testing: proptest + fuzz + deterministic simulation (turmoil/madsim) for sync; kill -9 crash harness; restore tests in CI; Playwright on web build + WebdriverIO embedded driver for macOS Tauri smoke.
8. Licence: FSL-1.1-ALv2 for app and hub, MIT/Apache for protocol libs, CLA, trademark; business = hosted hub/sync.

## 9. Risks and spike criteria

| Risk | Spike must prove |
|---|---|
| Blob GC deletes a blob still referenced by an unsynced peer | GC with grace period + tombstones passes a simulation with 3 peers, partitions and restarts; zero lost blobs over 10k seeds |
| FastCDC + encryption breaks dedupe or resumability | 1 GB file, 1% edit: resend < 5% of bytes; upload resumes after kill at random point |
| `VACUUM INTO` too slow on a large DB | 1 GB DB snapshot in < 10 s on an M1 without blocking writers > 100 ms; works with SQLCipher |
| restic helper in sandbox / notarisation | bundled restic signed with our team id runs from the sandboxed app, backs up to R2 and rest-server, restores; or rustic_core does the same in-process |
| Crash consistency | 1,000 random `kill -9` runs during mixed writes: `integrity_check` ok, no DB row without blob, sync journal resumes |
| macOS e2e for Tauri | WebdriverIO embedded driver runs 5 core flows in CI on a macOS runner, < 10 min, < 2% flake |
| Hub cost assumptions | re-check Hetzner prices on hetzner.com (blocked here) and Oracle Free limits before quoting numbers to users |
| Licence | lawyer-free check: every runtime dependency is FSL/Apache compatible (`cargo deny`, `license-checker`) in CI; no xl-* BlockNote packages |
| App Store later | a sandboxed build with agent tools removed passes a TestFlight/Mac App Store review |
