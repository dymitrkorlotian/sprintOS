# 05: Security, distribution, updates, supply chain, telemetry and licensing for a local-first Mac app

Research for the ADR on a local-first desktop shell (Tauri 2 / Electron / native Swift). Research date: **2026-10-02**. Everything cited says where it came from and, where the source gives one, its date. "Accessed 2026-10-02" means the page was read today. The sandbox egress proxy blocked many primary sites (tauri.app, electronjs.org, sparkle-project.org, pnpm.io, inkandswitch.com, gnu.org, wikipedia, cisa.gov, bleepingcomputer, doyensec). Where that happened I read the same docs from their GitHub source (`raw.githubusercontent.com`) or relied on search-result summaries, and I say which. The **"Unverified"** markers show the claims I could not check against a primary source in this session.

Repo facts I checked (read-only): the repo pins `pnpm@10.33.0` (`/home/user/sprint/package.json`). `pnpm-workspace.yaml` uses `onlyBuiltDependencies: [sharp, unrs-resolver]` and **has no `minimumReleaseAge`**. The web app uses `@blocknote/core|react|shadcn ^0.55.0` (MPL-2.0) and **no `@blocknote/xl-*` package** today.

---

## 1. Apple distribution

### 1.1 Apple Developer Program
- **USD 99 per membership year.** It includes App Store distribution, TestFlight, Certificates/Identifiers/Profiles, an Apple-verified Developer ID and "Eligibility for distribution options". The commission is "30% (15% if you're enrolled in the App Store Small Business Program …) and 15% for qualifying subscriptions". Fee waivers are only for nonprofits, education and government. Source: https://developer.apple.com/programs/whats-included/ (accessed 2026-10-02).
- Xcode Cloud (25 compute h/month) and CloudKit are included (search summary of https://developer.apple.com/programs/, accessed 2026-10-02).
- The program is the only real recurring cost of a signed Mac app. **Without it there is no Developer ID, so no notarization.** Since macOS 15 Sequoia, users can no longer Control-click to open an unnotarized app; they have to go to System Settings → Privacy & Security (AppleInsider 2024-08-06, https://appleinsider.com/articles/24/08/06/apple-removes-control-click-option-for-skipping-gatekeeper-in-macos-sequoia; MacRumors 2024-08-06). An unsigned app is therefore not a realistic option for anyone but the developer.

### 1.2 Developer ID signing + notarization + hardened runtime
From Apple, "Notarizing macOS software before distribution" (https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution, accessed 2026-10-02):
- Notarization is "not App Review". It is an automated malware and signing scan, and "usually takes less than an hour".
- Requirements: every executable code-signed with a **Developer ID** certificate, **Hardened Runtime** enabled, a **secure timestamp**, no `com.apple.security.get-task-allow`, and linked against the macOS 10.9+ SDK.
- "Beginning in macOS 10.15, all software built after June 1, 2019, and distributed with Developer ID must be notarized." MAS apps don't need it.
- Since 2023-11-01 only `notarytool` (or Xcode 14+) is accepted; `altool` is retired. The steps are `notarytool submit … --wait`, then `stapler staple` (stapling is recommended so Gatekeeper works offline).
- The notary keeps an audit trail, so if the signing key leaks Apple can revoke tickets for rogue builds.
- Hardened runtime and JS engines: if a host runs a JIT (as WebKit/V8 do) it may need `com.apple.security.cs.allow-jit` / `allow-unsigned-executable-memory` (same doc). Electron apps typically need `allow-jit`. Tauri uses the system WKWebView, which runs out of process, so the app itself usually needs no JIT entitlement. **Unverified** for every Tauri plugin.
- In CI, use an **App Store Connect API key** (`APPLE_API_ISSUER`, `APPLE_API_KEY`, `.p8`) rather than an Apple ID password. tauri-action supports both (search summaries of https://v2.tauri.app/distribute/sign/macos/ and the dev.to Tauri v2 shipping guides, accessed 2026-10-02).

### 1.3 Mac App Store (MAS) vs direct download

App Review Guidelines (https://developer.apple.com/app-store/review/guidelines/, "Last Updated: June 8, 2026", accessed 2026-10-02):
- **2.4.5(i)** MAS apps "must be appropriately sandboxed". **(ii)** "packaged and submitted using technologies provided in Xcode; no third-party installers". **(iv)** they may not download code that adds functionality. **(vi)** they "may not … require license keys, or implement their own copy protection". **(vii)** "They must use the Mac App Store to distribute updates; other update mechanisms are not allowed."
- **2.5.2** says apps must be self-contained and may not "download, install, or execute code which introduces or changes features". This limits user plugins/scripts that run downloaded code, and remotely loaded web UIs.
- **3.1.1** says unlocking features needs In-App Purchase. **3.1.3(b) Multiplatform Services** lets an app unlock purchases made on the web, "provided those items are also available as in-app purchases within the app". **3.1.1(a)**: in the US storefront, links and buttons to external purchase need no entitlement.
- Commission is 30%, or 15% under the Small Business Program (proceeds under USD 1M in the prior calendar year; search summary of the Apple newsroom 2020-11 and https://developer.apple.com/programs/whats-included/).
- Review time: Apple says "90% of submissions are reviewed in less than 24 hours". That is a search summary; the primary page was not fetched (**Unverified** exact current wording). Every hot-fix has to wait for review.

App Sandbox implications for this app:
- **Network**: `com.apple.security.network.client` for Claude/Voyage/sync. `network.server` only if you listen locally; the Tauri docs list both as the minimum (search summary of https://v2.tauri.app/distribute/app-store/).
- **Files**: user-chosen files via Powerbox (`files.user-selected.read-write`). Persistent access to a vault folder needs **security-scoped bookmarks** (`com.apple.security.files.bookmarks.app-scope`) plus `startAccessingSecurityScopedResource` / `stop…`. Leaking these "leaks kernel resources" (Apple archive doc https://developer.apple.com/library/archive/documentation/Miscellaneous/Reference/EntitlementKeyReference/Chapters/EnablingAppSandbox.html, via search summary). An app whose data lives in its own container (`~/Library/Containers/<id>`) needs none of this. A user-visible "vault folder" design (Obsidian-style) does need bookmarks.
- **Keychain**: sandboxed and provisioned apps get the Data Protection keychain naturally (§3.1).

**Can Tauri ship in the MAS?** Yes, there is an official guide (tauri-docs `src/content/docs/distribute/app-store.mdx`, read on GitHub 2026-10-02). It needs an `Entitlements.plist` with `com.apple.security.app-sandbox`, an App ID and Team ID, a Mac App Store Connect provisioning profile, a universal binary and a `.pkg` signed with a Mac Installer Distribution cert. The guide does not mention the updater; in a MAS build you have to compile the Tauri updater plugin out (guideline 2.4.5(vii)). Developers have filed issues about MAS uploads (tauri-apps/tauri#13118), so expect some friction.

**Can Electron ship in the MAS?** Yes, but only with the special **MAS build** of Electron. From the Electron docs (`docs/tutorial/mac-app-store-submission-guide.md`, read on GitHub 2026-10-02): "only the MAS build of Electron can run with the App Sandbox", and in the MAS build "`crashReporter` and `autoUpdater`" are disabled. "Video capture may not work for some machines", "certain accessibility features may not work", and apps "will not be aware of DNS changes".

**Native Swift** is the reference case: Xcode, sandbox, StoreKit, TestFlight and MetricKit all work as designed.

**TestFlight for Mac** has been available since macOS 12 Monterey (MacStories / XDA 2021, via search summary). It allows up to 10,000 external testers and collects crash logs and feedback, but **only for App Store-track builds** (sandboxed, MAS-signed). It is not available for Developer ID builds.

**Trade-off summary for this app**

| | Direct (Developer ID + notarized) | Mac App Store |
|---|---|---|
| Cost | $99/yr | $99/yr + 15–30% of sales |
| Sandbox | optional (recommended to adopt anyway) | mandatory |
| Updates | own updater (Sparkle / Tauri / electron-updater) | MAS only |
| Payments | own (Stripe/Paddle/Lemon Squeezy), licence keys OK | IAP (US: external links allowed) |
| Release latency | minutes (notarization) | review, usually ≤24h |
| GPL code | fine | conflict risk (§6.3) |
| Crash data | own (Sentry etc.) | App Store Connect / Xcode Organizer + own |
| Discovery/trust | low | higher |

**Recommendation:** start with direct distribution (Developer ID + notarization + hardened runtime), with the sandbox *enabled where cheap*. Keep the code sandbox-compatible: data in the app container, bookmarks for any external folder, no helper daemons, and no downloaded executable code. A MAS build then stays possible later, and so does iOS, which is always sandboxed.

---

## 2. Auto-update

### 2.1 Sparkle 2 (native / any Mac app)
- Sparkle README (https://raw.githubusercontent.com/sparkle-project/Sparkle/2.x/README.markdown, accessed 2026-10-02): "Updates are verified using EdDSA signatures and Apple Code Signing. Supports Sandboxed applications in Sparkle 2". It has delta updates, atomic installs, needs macOS 12+ and needs an **HTTPS** server.
- Sandboxed apps need the Installer XPC service (`SUEnableInstallerLauncherService = YES`). The public key goes in `SUPublicEDKey`. `SURequireSignedFeed = YES` also signs the appcast and release notes; it defaults to NO, so **turn it on** (search summaries of https://sparkle-project.org/documentation/sandboxing/ and /customization/).
- **Recent vulns:**
  - CVE-2025-0509: a signing-check bypass before 2.6.4 (GHSA-wc9m-r3v6-9p5h).
  - CVE-2025-10015 (TCC bypass via a globally registered Downloader.xpc) and CVE-2025-10016 (local root via the installer daemon race), fixed in **2.7.2** (CERT Polska, 2025-09, https://cert.pl/en/posts/2025/09/CVE-2025-10015/).
  - Sparkle 2.7.3 has further local-exploit fixes (GitHub discussion #2764).
  - Pin a current 2.x release.
- Licence: MIT. **Unverified** in this session; based on repo knowledge.
- Electron and Tauri apps *can* embed Sparkle, but it is unusual; each framework has its own updater.

### 2.2 Tauri updater plugin
From tauri-docs `plugin/updater.mdx` (read on GitHub 2026-10-02):
- "Tauri's updater needs a signature to verify that the update is from a trusted source. **This cannot be disabled.**" It uses minisign/Ed25519 keys made with `tauri signer generate`, and the public key is embedded in `tauri.conf.json`. **Losing the private key means installed apps can no longer be updated.**
- "TLS is enforced in production mode." `dangerousInsecureTransportProtocol` exists for development only.
- It accepts a static JSON (`latest.json`) or a dynamic server (204 means no update). The docs warn that "Tauri will validate the whole file before checking the version", so a broken entry for one platform breaks all of them.
- On macOS it ships a `.app.tar.gz` plus a `.sig`. tauri-action builds, signs, notarizes and uploads to GitHub Releases, including `latest.json` (`includeUpdaterJson`).
- Rollback: you can override the version comparator. That allows downgrades, so a downgrade policy is a deliberate choice.
- Crate `tauri-plugin-updater` 2.13.1 is `Apache-2.0 OR MIT` (crates.io API, 2026-09-30).

### 2.3 Electron
- Built-in `autoUpdater` on macOS = **Squirrel.Mac**. The app must be code-signed; it streams the ZIP to disk, checks SHA256 and supports optional delta patches (Electron `docs/api/auto-updater.md`, read on GitHub 2026-10-02). It is disabled in MAS builds.
- **electron-updater** (electron-builder) supports GitHub Releases, S3, R2 and generic HTTPS providers. The mac ZIP must be signed and notarized. Newer versions offer an Ed25519-signed manifest (search summary of https://www.electron.build/docs/features/auto-update/). Doyensec showed a signature-validation bypass leading to RCE in electron-updater in 2020 (https://blog.doyensec.com/2020/02/24/electron-updater-update-signature-bypass.html). On **2026-02-16** they published "Building a Secure Electron Auto-Updater" and the ElectronSafeUpdater reference design (Ed25519, SHA-512, explicit threat model). That is a sign the default stack needed hardening (search summary; the blog itself was blocked).
- **update.electronjs.org** is a free update service for open-source apps on GitHub Releases. **Unverified** current terms.

### 2.4 Hosting for $0
- **GitHub Releases** works for all three: Sparkle appcast as a release asset or GitHub Pages, Tauri `latest.json` as a release asset, electron-updater's `github` provider. It's free for public repos. A private repo needs a token in the client, which is a bad idea, so use a public "releases" repo or Cloudflare R2/Pages (free tier) for a closed-source app.
- The trust anchor is the **embedded public key**, not the host. A compromised host can only serve stale or withheld updates, unless the signing key also leaks.

### 2.5 Update-channel attacks and mitigations
- **Notepad++ (June–Dec 2025):** the hosting provider was compromised and update traffic for selected targets was redirected to malicious installers. It was attributed to a Chinese state actor and fixed in v8.8.9 (2025-12-09) with hardened verification (The Hacker News 2026-02; Help Net Security 2026-02-02; via search summaries). Lesson: **verify signatures client-side with a pinned key; never trust the transport or host.**
- Threat classes (The Update Framework spec, https://theupdateframework.github.io/specification/latest/, via search summary):
  - **rollback**: serving an older vulnerable version.
  - **freeze**: serving a stale "no update" answer forever.
  - **mix-and-match**: combining metadata from different releases.
  - **arbitrary install**: installing something the publisher never released.
  - **key compromise**: the signing key itself is stolen.
- Mitigations for a solo dev:
  1. A separate offline-ish signing key for updates (minisign/EdDSA), kept as an encrypted GitHub Actions secret or, better, signed locally on release.
  2. Refuse downgrades (monotonic version check).
  3. Sign the feed itself (`SURequireSignedFeed`), not only the binaries.
  4. Keep Apple code-sign verification as a second, independent check. Sparkle and Squirrel.Mac do this; for Tauri the notarized-bundle check is done by Gatekeeper on first launch. **Unverified** whether the Tauri updater re-verifies the Apple signature.
  5. Publish SHA-256s and GitHub artifact attestations (§4.4).
  6. Plan for key rotation. Sparkle supports rotation by shipping a new key in a release signed with the old one. **Unverified** for the Tauri updater: rotating means shipping a release that embeds the new pubkey.

---

## 3. Security model

### 3.1 Secrets (Claude/Voyage API keys, sync keys)
- **macOS has two keychains** (Apple TN3137, revised 2026-09-24, https://developer.apple.com/documentation/technotes/tn3137-on-mac-keychains):
  - The **file-based keychain** (legacy, ACL-based, "on the road to deprecation"). SecItem defaults to it.
  - The **data protection keychain** (iOS-style). It uses access groups taken from code-signing entitlements, which "must be authorized by a provisioning profile". You opt in with `kSecUseDataProtectionKeychain`. Biometrics, the Secure Enclave and iCloud Keychain **require** it.
  - New in **macOS 26.4**: file-based keychains may depend on protected entropy files in `/var/db/SystemKeys`.
  - Apple: "Default to targeting the data protection keychain".
- **Secure Enclave** (https://developer.apple.com/documentation/security/protecting-keys-with-the-secure-enclave, accessed 2026-10-02):
  - P-256 only, and keys must be created inside the Enclave.
  - Hardware: Macs with Touch ID/Touch Bar (T2) or M1+.
  - It can wrap a symmetric data key with ECIES (`eciesEncryptionCofactorX963SHA256AESGCM`).
  - Biometric gating via `SecAccessControl` (`biometryAny`).
  - Pattern: an SE-held P-256 key wraps the database encryption key, so a stolen disk image or backup without the device can't decrypt it. Don't make it the only copy: losing the Mac means losing the data, so pair it with a recovery key or passphrase.
- **Tauri**:
  - `tauri-plugin-stronghold`: "Stronghold is no longer recommended and will be deprecated and therefore removed in v3" (Tauri discussion #7846 / plugin changelog, via search summary). The crate still gets releases (2.4.0, 2026-09-30, crates.io).
  - Use the **`keyring` crate**: 4.2.0, MIT OR Apache-2.0, 2026-08-29.
  - Its Apple store `apple-native-keyring-store` (1.0.2) has a `keychain` module for the legacy keychain and a `protected` module for the Data Protection keychain, intended for provisioning-profile-signed apps (docs.rs summary).
  - Community plugins exist: tauri-plugin-keyring (HuakunShen) and tauri-plugin-keyring-store. **Unverified** maintenance quality.
  - For Secure Enclave use, write a small Swift or Rust (security-framework crate) bridge.
- **Electron `safeStorage`**:
  - The docs say keys are kept in Keychain "in a way that prevents other applications from loading them without user override", and that content is "protected from other users and other apps running in the same userspace" (`docs/api/safe-storage.md`, read on GitHub 2026-10-02).
  - Caveat from the community (search summaries; HN "macOS data protection keychain for Electron apps", the biw/keychain-store README): safeStorage uses the **legacy file-based keychain**. Code running *inside* the app (a poisoned npm dependency, an injected dylib) can call `safeStorage.decryptString` with no prompt.
  - Data Protection keychain wrappers for Electron exist (biw/keychain-store). **Unverified** maturity.
- **Swift**: the SecItem API / CryptoKit directly is the simplest and most correct.
- Overall: keychain storage protects keys **at rest and from other users and apps**. It does **not** protect against malicious code inside your own process. That is why supply chain (§4) and renderer isolation (§3.3) matter more than which keystore you pick.

### 3.2 Data at rest on macOS
- **FileVault** encrypts the whole volume. Apps can't require it but can detect it and warn. **Unverified**: no public API besides `fdesetup status`, which needs no root for status. I could not fetch Apple's Platform Security guide (support.apple.com was blocked).
- macOS does **not** apply iOS-style per-file Data Protection classes to ordinary app files. **Unverified**: the Platform Security guide was not reachable. TCC "data vaults" protect other apps' containers from non-entitled processes.
- **App Sandbox** (even outside the MAS) limits what a compromised app can read. It also puts the app's container under TCC protection: since macOS 14, other apps are prompted before accessing another app's container. **Unverified**: from memory of the WWDC23 "What's new in privacy" session, not fetched.
- Option: SQLCipher or app-level encryption of the database file with the key in the keychain. This mainly helps against backups and cloud sync of the folder, plus other-user scenarios; FileVault already covers a stolen laptop.

### 3.3 Webview hardening

**Tauri 2** (tauri-docs `security/*.mdx`, read on GitHub 2026-10-02):
- **Capabilities** grant permissions per window or webview *label* ("not titles"). Windows in several capabilities "merge the security boundaries".
- By default only bundled code can call commands. Remote URLs can be allowed per capability. "Tauri is unable to distinguish between requests from an embedded `<iframe>` and the window itself" on Linux and Android.
- Out of scope: "Malicious or insecure Rust code" and "0-days or unpatched 1-days in the system WebView".
- **CSP**: Tauri injects nonces and hashes into bundled assets at compile time. Recommended: the most restrictive policy and no CDN scripts.
- Use **scopes** on the fs and http plugins (allow-list paths and hosts).
- The **Isolation pattern** puts an isolation iframe between the frontend and IPC.
- Tauri 2 was pen-tested by Radically Open Security (2023-11 to 2024-08, NLnet-funded). "All findings … resolved in the release candidate" (report PDF in tauri-apps/tauri `audits/`, via search summary).
- The system WKWebView gets security patches through macOS updates and nothing has to be shipped.

**Electron**: the 20-item checklist (`docs/tutorial/security.md`, read on GitHub 2026-10-02):
- `contextIsolation` is the default since 12, the `sandbox` is the default since 20, and `nodeIntegration` has been off since 5.
- Also on the list: CSP, `setPermissionRequestHandler`, limiting navigation and `window.open`, no `shell.openExternal` on untrusted input, validating the IPC `sender`, preferring custom protocols to `file://`, and staying on a current Electron.
- **Fuses** (`docs/tutorial/fuses.md`):
  - Disable `runAsNode`, `nodeOptions`, `nodeCliInspect` and `grantFileProtocolExtraPrivileges`.
  - Enable `cookieEncryption`, `embeddedAsarIntegrityValidation` and `onlyLoadAppFromAsar`.
  - Use `@electron/fuses` to set them.
- Electron bundles its own Chromium, so **you** must ship each Chromium security fix. Electron 44.5.1 was current on npm at 2026-10-02.

**Rendering untrusted HTML** (saved web pages, link previews, SVG, PDF):
- Put archived HTML in an `<iframe sandbox>` **without `allow-same-origin`**, which gives an opaque origin. An unsandboxed `srcdoc` iframe is same-origin with the parent (MDN iframe reference; hatchlab5/sandbox-render README; via search summaries).
- Add an in-document CSP such as `default-src 'none'; img-src data: blob:; style-src 'unsafe-inline'; form-action 'none'; base-uri 'none'`. `form-action` and `base-uri` are not covered by `default-src`.
- **Strip scripts** at capture time (DOMPurify or a server-side sanitizer) and still keep `allow-scripts` off.
- Better: serve archived content from a **separate origin or custom scheme**. In Tauri that means a second webview, or a custom protocol whose window has *no* capability. In Electron it means a separate `session` / partition with no preload. That way a sandbox escape can't reach IPC.
- **SVG**: render as `<img src>`, which runs no scripts, never inline it. **PDF**: use the system viewer (PDFKit in Swift, WKWebView's built-in PDF) or pdf.js in a sandboxed frame.
- **Never** let archived content trigger `shell.openExternal` / `opener` without a user click and a scheme allow-list (http/https/mailto).

### 3.4 SSRF when fetching link metadata locally
- Moving the link fetcher from a server to the user's Mac changes the target. It can no longer hit cloud metadata, but it **can** reach the user's router admin, NAS, `localhost` dev servers, Docker, Ollama and so on.
- **macOS 15+ Local Network privacy** (Apple TN3179, revised 2026-02-17, https://developer.apple.com/documentation/technotes/tn3179-understanding-local-network-privacy):
  - An outgoing TCP connection to a *local network* address (one on a broadcast-capable interface: Wi-Fi/Ethernet) triggers a one-time prompt.
  - **Loopback is not a "local network"**, so `127.0.0.1` / `localhost` services are reachable without any prompt. **Inferred** from TN3179's definition; the note does not name loopback explicitly.
  - Traffic from **WKWebView / Safari is exempt**.
  - CLI tools run from Terminal are auto-allowed.
  - So the prompt is a partial backstop at most, and an unexpected prompt would confuse users. Add `NSLocalNetworkUsageDescription` only if you really need local access.
- Prior art: **Trilium** (a local-first notes app) had a DNS-rebinding SSRF in link preview (GHSA-8h94-9q2r-jhqp, published 2026-05-24, CVSS 4.9). It validated the DNS answer and then called `fetch()` with the hostname, which resolved again. A similar CVE-2026-61704 for `link-preview-js` is reported (SecureLayer7; **Unverified** details).
- Mitigations:
  - Resolve once, reject loopback, private, link-local, CGNAT, multicast and IPv6 ULA/link-local ranges, then **connect to the pinned IP** while keeping SNI and Host. Re-validate on every redirect, capping hops.
  - Allow only http and https, cap the response size and set timeouts.
  - Optionally fetch through a remote fetcher (the existing Vercel function) instead of locally.

---

## 4. Supply chain

### 4.1 npm incidents 2025–2026 (all via search summaries of the cited reporting)

| When | Incident | Vector |
|---|---|---|
| 2025-03-12/15 | **tj-actions/changed-files** (CVE-2025-30066), 23k repos | Compromised GitHub Action; **tags re-pointed** to a malicious commit, which leaked secrets to logs (Unit42, Sysdig) |
| 2025-08-26 | **Nx "s1ngularity"** | Stolen npm token via a GitHub Action. Malicious `postinstall` scanned for secrets and **ran installed AI CLIs with `--dangerously-skip-permissions` / `--yolo`**; 2,349 credentials leaked (The Hacker News 2025-08) |
| 2025-09-08 | **chalk/debug + 16 more** (≈2.6B weekly downloads) | Maintainer phished via `npmjs.help` 2FA-reset email; browser crypto-drainer; pulled in about 2h (Vercel, Semgrep, StepSecurity) |
| 2025-09 | **Shai-Hulud** worm, 500+ packages | Self-propagating via stolen npm tokens and lifecycle scripts (CISA alert 2025-09-23) |
| 2025-11-24 | **Shai-Hulud 2.0**, about 796 packages, 25k+ GitHub repos | Preinstall-stage worm, credential theft (Datadog, Check Point) |
| 2026-03-31 | **axios** 1.14.1 / 0.30.4 | Maintainer account takeover; phantom dep `plain-crypto-js` with a `postinstall` cross-platform RAT, including macOS (Datadog, Snyk, CISA 2026-04-20) |
| 2026-05-11 | **TanStack (42 packages, 84 versions) "Mini Shai-Hulud"**, 170+ npm/PyPI packages that day | `pull_request_target` "pwn request", then **Actions cache poisoning**, then the **OIDC token read from runner memory**. The first malicious npm packages with **genuine SLSA provenance** (TanStack postmortem) |
| 2026-06 | 30+ `@redhat-cloud-services` packages | GitHub account compromised via a malicious **VS Code extension** (RHSB-2026-006) |
| 2026-08-04 | **ChainDrop** (Shai-Hulud family), 452 packages / 2,251 versions, about 2B monthly downloads, incl. `keyv`, `cacheable`, **`flat-cache`, `file-entry-cache`** (ESLint deps) | Preinstall `setup.mjs` downloads Bun, steals credentials, **plants Claude Code and VS Code hooks** for persistence; exfiltration via GitHub repos and EtherHiding (Elastic Security Labs) |

Takeaways for this project:
1. Install-time scripts and **developer machines and agents** are the main target. The repo is built by several Claude sessions, and ChainDrop explicitly plants Claude Code hooks.
2. Provenance alone is not enough (TanStack).
3. A **release-age delay** would have blocked every one of these, because each was detected within hours.

### 4.2 npm/pnpm mitigations
- **pnpm 10**:
  - Dependency lifecycle scripts are **off by default**, and `onlyBuiltDependencies` is the allow-list (Socket 2025-01).
  - `allowBuilds` (10.26+) replaces the older build settings.
  - `minimumReleaseAge` is available but **defaults to 0** in v10.
  - The repo is on 10.33 with no age setting. **Action: add `minimumReleaseAge: 1440` (or 4320) now.**
- **pnpm 11.0** (2026-04-28, `pnpm.io/blog/releases/11.0.md` read on GitHub):
  - Defaults: `minimumReleaseAge: 1440` (one day), `blockExoticSubdeps: true`, `strictDepBuilds: true` (fail on unapproved build scripts) and `verifyDepsBeforeRun: install`.
  - Requires Node 22+, which the repo already uses. Upgrade.
  - `trustPolicy: no-downgrade` blocks versions published with weaker auth than earlier ones. It is opt-in (search summary of the pnpm settings docs).
- **npm registry hardening (GitHub, 2025-09-22 plan)**:
  - Classic tokens revoked (Nov 2025).
  - Granular write tokens default to 7 days and are capped at 90.
  - Publishing requires FIDO 2FA.
  - **Trusted publishing (OIDC)** (github.blog "Our plan for a more secure npm supply chain").
  - This matters if you publish packages; as a consumer, rely on lockfile + age + script blocking.
- Also: a committed lockfile with `--frozen-lockfile` in CI, Dependabot/Renovate with a **cooldown**, and keeping the dependency count low in the desktop shell. Tauri's frontend is the same web app; the Rust side replaces most Node server dependencies.

### 4.3 Rust crates
- Incidents:
  - `faster_log` / `async_println`: fast_log typosquats published 2025-05-25 that stole Solana and Ethereum keys from source files; removed 2025-09-24 (Socket).
  - **2026-08-20**: `arrayref` 0.3.10, `internment` 0.8.7 and `append-only-vec` 0.1.9 were published from a compromised account. They added the typosquat `proc-macro1`, whose **build.rs downloaded and ran a payload at compile time**. Removed in 86–107 minutes. arrayref has about 244M downloads and is in "roughly three-quarters of all Rust environments" (The Hacker News 2026-08; StepSecurity; Wiz).
  - crates.io updated its malicious-crate notification policy (blog.rust-lang.org 2026-02-13).
- Tools:
  - **cargo-audit**: RustSec advisories against `Cargo.lock`.
  - **cargo-deny**: advisories + **licences** + bans + **sources**. This is the recommended baseline.
  - **cargo-vet**: Mozilla's human-audit ledger. You can import audits from Mozilla, Google, Bytecode Alliance and others; it is high-friction.
  - (Search summaries: Microsoft RustTraining engineering book ch. 6; rustprojectprimer.com.)
- crates.io **Trusted Publishing** (OIDC, 30-minute tokens) since 2025-07 (Rust blog 2025-07-11), with GitLab added later (Rust blog 2026-01-21).
- Rust has **no equivalent of a release-age gate** (**Unverified**: no native cargo setting found). Mitigations:
  - Commit `Cargo.lock` and build with `cargo build --locked`.
  - Update crates deliberately, with a delay and `cargo-deny` in CI.
  - Use `cargo vet` imports for the crates Tauri pulls in.
  - Remember that `build.rs` and proc-macros run arbitrary code at build time.

### 4.4 Builds and signing in GitHub Actions
- **Pin actions to full commit SHAs.** Tags are mutable: tj-actions had its tags re-pointed.
- Avoid `pull_request_target`; don't share caches across trust boundaries (TanStack); use minimal `permissions:`.
- Keep signing secrets (Developer ID .p12, App Store Connect API key, updater private key) in a **protected environment** with required reviewers, used only by tag-triggered release jobs. Remember that OIDC and other tokens sit in runner memory.
- **GitHub artifact attestations** (`actions/attest-build-provenance`; verify with `gh attestation verify <file> -R owner/repo`; Sigstore-backed; github.blog 2024-05-02 and the 2025-02-18 changelog). They are free for public repos. **Unverified**: private-repo availability on the free plan.
- **Reproducible builds**: Tauri/Rust builds can be made close to reproducible (`--locked`, pinned toolchain via `rust-toolchain.toml`, `SOURCE_DATE_EPOCH`). Apple code signatures and notarization tickets add non-deterministic bytes, so compare the *unsigned* payloads. I found no evidence that bit-for-bit reproducible signed Tauri or Electron Mac builds are routine (**Unverified**). This is a nice-to-have, not a requirement.
- macOS GitHub-hosted runners are free for public repos. Private repos use minutes at a 10× multiplier. **Unverified** current multiplier and free minutes.

---

## 5. Crash and error reporting that respects privacy
- **Sentry SaaS Developer plan:** free, 5,000 errors/month, 1 user, 30-day retention (several 2026 pricing write-ups, via search; **Unverified** on sentry.io directly). The repo already uses Sentry for the web app.
- **Self-hosted Sentry:** 4 CPU, 16 GB RAM + 16 GB swap minimum, 32 GB recommended (develop.sentry.dev/self-hosted, via search summary). Licence is FSL. That is not $0 to run.
  - Lighter self-host alternatives: GlitchTip (Sentry-protocol compatible, MIT, **Unverified**) and Bugsink. Or no backend at all: write crash reports to disk and offer "Copy diagnostics" / "Send report" by email.
- **Tauri:** `tauri-plugin-sentry` (by timfish) covers JS errors via @sentry/browser, Rust panics, and **native crashes as minidumps** via `sentry-rust-minidump` (a separate crash-reporter process) (crates.io / lib.rs summaries).
- **Electron:** `@sentry/electron` and the built-in `crashReporter` (disabled in MAS builds).
- **Scrubbing:**
  - Keep `sendDefaultPii: false`, which is the default.
  - Add a `beforeSend` hook that strips breadcrumbs with note titles, URLs and query strings, and request bodies.
  - Turn on server-side scrubbing as defence in depth (docs.sentry.io "Scrubbing Sensitive Data", via search).
  - For a notes app, also **disable session replay**, console breadcrumbs and DOM breadcrumbs, since those can carry note text.
- **Apple:**
  - **MetricKit** (`MXCrashDiagnostic`) is available on macOS 12+ with immediate delivery. Reports say payloads arrive reliably for App Store and TestFlight installs; behaviour for Developer ID builds is **Unverified**.
  - The **Xcode Organizer / App Store Connect** crash feeds cover App Store and TestFlight users who opted into sharing analytics with developers.
  - Direct-download apps get nothing from Apple, so they need their own reporter.
- **Norms:**
  - **Obsidian:** "does not collect any telemetry", and the update check can be disabled (obsidian.md/privacy, via search summary).
  - **VS Code:** `telemetry.telemetryLevel` = `all`/`error`/`crash`/`off`; extensions are not covered by the setting (code.visualstudio.com/docs/configure/telemetry, via search).
  - Recommended: **off by default, ask once** ("Send crash reports?" with a preview of what gets sent), crash-only level, no usage analytics, and a setting to turn it off. Document this in the privacy policy.

---

## 6. Licensing

### 6.1 Licence families (from general knowledge unless a source is cited; GNU, OSI and fair.io sites were blocked)

| Licence | Copyleft | Effect for this app |
|---|---|---|
| MIT / Apache-2.0 | none (Apache adds a patent grant) | Anyone, including competitors, can fork and sell. Maximum adoption. |
| MPL-2.0 | file-level | Changes to MPL files must be shared; the rest of the app can stay closed. BlockNote core uses it. |
| GPL-3.0 | strong (distribution) | Any distributed combined work must be GPL; network use is not "distribution". |
| AGPL-3.0 | strong + network (§13) | SaaS hosts must offer source. Used by Logseq, AppFlowy, Standard Notes; it discourages commercial forks of a sync server. |
| **FSL-1.1** (Sentry, 2023-11) | source-available, non-compete | "Competing Use" = a commercial product that substitutes for the software or offers "the same or substantially similar functionality". **Each version becomes Apache-2.0 or MIT two years after release** (FSL-1.1-ALv2 template, getsentry/fsl.software, read 2026-10-02). |
| **BUSL-1.1** (MariaDB, HashiCorp) | source-available, licensor-defined "Additional Use Grant" | Converts to a GPL-compatible licence at the Change Date, at most four years later. **Unverified** in this session (mariadb.com was blocked). |
| "Fair Source" (fair.io, 2024) | an umbrella label for FSL-style delayed-open licences | Not OSI open source. **Unverified** current definition. |
| Open core | an OSS core plus proprietary features or services | Examples: AFFiNE (MIT core; `packages/backend` and `packages/common/native` under a separate server licence, per AFFiNE's LICENSE read 2026-10-02) and the BlockNote xl-* packages. |

The **sole copyright holder** can always dual-license their own code, including AGPL plus a commercial licence or a MAS exception. Contributions from other people need a CLA or DCO-plus-relicensing terms to keep that option open.

### 6.2 BlockNote
- `@blocknote/core`, `react` and `shadcn` 0.55.0 are **MPL-2.0** (npm registry, 2026-10-02). Note: MPL, not MIT.
- `@blocknote/xl-multi-column`, `xl-pdf-exporter`, `xl-docx-exporter` and `xl-ai` 0.55.0 are **"GPL-3.0 OR PROPRIETARY"** (npm registry). The BlockNote LICENSE.txt says: "The XL packages … are licensed under the GNU General Public License Version 3 (GPL-3.0). Additionally, a commercial license is available" (TypeCellOS/BlockNote `LICENSE.txt`, read 2026-10-02). **Unverified** price; the pricing page was blocked.
- (a) **Closed-source app:** you can't ship xl-* without buying the commercial licence. Distributing a desktop binary *is* distribution, whereas the web app arguably isn't under GPL (no network clause), but a downloadable bundle is.
- (b) **AGPL or GPL app:** compatible. GPLv3 §13 and AGPLv3 §13 explicitly allow combining them; the combined work carries GPL obligations for the xl-* parts and AGPL for yours.
- (c) **MAS-distributed GPL app:**
  - The FSF holds that the App Store's usage rules and DRM add restrictions that GPLv2/v3 forbid (§10 of GPLv3). That is why GNU Go was removed in 2010 and **VLC for iOS was pulled in January 2011** after a VLC copyright holder objected. VLC came back in 2013 after VideoLAN relicensed libVLC to LGPL/MPL. (**Unverified in this session**: wikipedia, gnu.org and fsf.org were blocked; well-documented history from training data.)
  - With third-party GPL code (BlockNote xl-*), you can't grant the needed exception yourself, so either **buy the commercial xl licence or keep xl-* out of the MAS build**.
  - Apple lets developers supply a custom EULA, but whether that cures the conflict is legally untested. **Opinion, not legal advice.**

### 6.3 Licences of candidate dependencies (checked 2026-10-02)

| Component | Licence | Source |
|---|---|---|
| Tauri (`tauri` 2.12.1, `@tauri-apps/api` 2.12.1) | Apache-2.0 OR MIT | crates.io API, npm |
| Electron 44.5.1 | MIT | npm |
| PGlite 0.5.8 | Apache-2.0 on npm. The repo LICENSE is Apache-2.0; bundled Postgres code is under the PostgreSQL Licence (**Unverified** exact dual statement) | npm, GitHub raw |
| SQLite | Public domain | **Unverified** in this session (sqlite.org blocked); long-standing |
| sqlite-vec 0.1.9 | "MIT OR Apache" | npm |
| Automerge 3.5.0 | MIT | npm |
| Loro (`loro-crdt` 1.16.4) | MIT | npm |
| Yjs 13.6.33 | MIT | npm |
| PowerSync service (Open Edition) | **FSL-1.1-ALv2** (Apache-2.0 after 2 years). `@powersync/service-core` 1.27.0 is FSL; client SDK `@powersync/web` 2.4.2 is Apache-2.0 | GitHub raw LICENSE, npm |
| ElectricSQL (`electric` repo; `@electric-sql/client` 1.5.28) | Apache-2.0 | GitHub raw, npm |
| Zero (`@rocicorp/zero` 1.9.0; `rocicorp/mono`) | Apache-2.0 | npm, GitHub raw |
| keyring 4.2.0 / apple-native-keyring-store 1.0.2 | MIT OR Apache-2.0 | crates.io |
| Sparkle | MIT (**Unverified**) | |
| Sentry server | FSL (Fair Source) | blog.sentry.io via search |

FSL on the PowerSync server matters only if *you* offer a competing sync service. Self-hosting it for your own app is a Permitted Purpose.

### 6.4 Local-first apps and business models

| App | Licence | Model |
|---|---|---|
| Obsidian | Proprietary, closed source | Free app; paid Sync, Publish and commercial licence. Plugins are the community's. No telemetry. |
| Anytype | **Any Source Available License 1.0**: use only for non-commercial purposes or commercial use on "Allowed Networks" approved by Any Association (anytype-ts LICENSE.md, read 2026-10-02) | Freemium network storage tiers |
| AFFiNE | MIT for the client, separate licence for backend/native dirs (AFFiNE LICENSE) | Cloud and self-host EE |
| AppFlowy | AGPL-3.0 (LICENSE read) | Cloud + paid plans |
| Logseq | AGPL-3.0 (LICENSE.md read) | Sync (paid), donations |
| Standard Notes | AGPL-3.0 (app LICENSE read) | Paid E2EE sync tiers. Acquired by Proton in 2024 (**Unverified** here) |

Pattern: **the client is free (often open), and revenue comes from sync, hosting or publishing**. The sync server is where a licence like AGPL, FSL or source-available protects the business.

---

## 7. Threat model for a local-first personal OS

### 7.1 Local-first guidance
- Kleppmann, Wiggins, van Hardenberg and McGranaghan, "Local-first software: you own your data, in spite of the cloud" (Onward! 2019, Ink & Switch). Its seven ideals: fast (no spinners), multi-device, offline, collaboration, longevity ("the long now"), **privacy and security by default**, and user ownership and control. The essay argues that E2EE sync servers can't read data and that servers become dumb relays. (**Unverified in this session**: inkandswitch.com and kleppmann.com were blocked; content from training data.)
- Ink & Switch's later work (Keyhive, Beelay: E2EE access control and sync for local-first, 2024–2025) tackles group key management and revocation. **Unverified** current status.

### 7.2 Assets, adversaries and controls for this app

| Asset | Threat | Control |
|---|---|---|
| Notes, tasks, files on disk | Stolen laptop | FileVault (prompt the user), optional DB encryption with an SE-wrapped key |
| | Other local apps or malware | App Sandbox + TCC container protection; no world-readable exports |
| API keys (Claude, Voyage) | Exfiltration by a malicious dependency in-process | Keychain (Data Protection) + **make the Rust/Swift core, not the webview, hold keys and make the API calls**; the webview never sees the key; strict CSP `connect-src` |
| Saved web pages / previews | XSS into the app, then IPC, then file and key access | Sandboxed opaque-origin iframe / separate webview with no capabilities; sanitize at capture |
| Link fetcher | SSRF to LAN/localhost | IP pinning + private-range deny list (§3.4) |
| Update channel | Malicious update (Notepad++-style) | Ed25519-signed feed and binaries, pinned key, no downgrades, notarization |
| Build pipeline | Worms in npm/crates, poisoned Actions | pnpm 11 defaults, cargo-deny/vet, SHA-pinned actions, protected release environment, attestations |
| **Sync server** (if any) | **Breach**: attacker reads or alters all users' data | **E2EE**: the server stores ciphertext CRDT ops or snapshots only. Keys derive from a device key plus a recovery phrase. Authenticate ops (sign or MAC) so a malicious server can't inject edits. Rollback/freeze protection via version vectors. Metadata (sizes, timing, doc counts) still leaks. |
| AI features | Plaintext sent to Anthropic and Voyage | Explicit per-feature consent; document the providers' retention; local embedding option later |

### 7.3 E2EE trade-offs
- **What you lose:**
  - Server-side features: full-text or semantic search on the server, server-run AI jobs (today's Inngest jobs and the `object_chunks` index).
  - Web-app access without the key: the web client needs the key in the browser, from a passphrase or WebAuthn PRF.
  - Easy account recovery: a lost recovery phrase means lost data.
  - Sharing and collaboration: needs group key management.
- **What you keep:** a sync-server breach becomes an availability and metadata incident instead of a disclosure of everyone's notes. For a product that might be sold, and that stores private notes, this is a strong differentiator (Standard Notes, Obsidian Sync and Anytype all market E2EE).
- **Middle path:**
  - Phase 1: local-only plus file-level backup (iCloud Drive or user folder), so no server exists to breach.
  - Phase 2: E2EE sync relay (CRDT ops encrypted client-side; Automerge, Loro and Yjs are all MIT).
  - AI calls go straight from the device to Anthropic and Voyage with the user's own key. Embeddings computed locally (sqlite-vec / PGlite+pgvector) never leave the device except to Voyage.

---

## 8. Recommendations (concrete)

1. **Distribution:**
   - Ship direct with Developer ID + hardened runtime + notarytool + staple, on GitHub Releases. Pay the $99/yr. Don't plan for the MAS in v1, but keep the code sandbox-clean (container storage, security-scoped bookmarks for any external vault, no helpers or daemons, no downloaded executable code). A MAS or iOS build then becomes a packaging task.
   - If you go to the MAS later: enrol in the Small Business Program (15%), compile out the updater, and use IAP or the 3.1.3(b) web-purchase route.
2. **Updates:** if Tauri, use the Tauri updater + `latest.json` on GitHub Releases via tauri-action. If Swift, use Sparkle ≥2.7.3 with `SURequireSignedFeed`. In either case:
   - Keep the update signing key offline or in a protected environment, with an encrypted backup (losing it strands every user).
   - Refuse downgrades.
   - Attach `gh` attestations and SHA-256s.
   - Avoid electron-updater unless you add a signed manifest.
3. **Secrets:**
   - Store the Claude and Voyage keys in the **Data Protection keychain**: `keyring` 4 with apple-native `protected` in Tauri; SecItem in Swift; not the plain `safeStorage` in Electron.
   - Make all API calls from the native core so the webview never holds keys.
   - Don't use Stronghold.
   - Optionally use an SE-wrapped database key plus a recovery phrase.
4. **Webview:**
   - Strict CSP, minimal Tauri capabilities per window label, fs and http plugin scopes.
   - Render saved pages in an opaque-origin sandboxed iframe or a separate capability-less webview. SVG as `<img>`, PDF via a native viewer.
   - If Electron: apply all 20 checklist items and set the fuses (runAsNode off, ASAR integrity + onlyLoadAppFromAsar on).
5. **Link fetching:** IP-pinned fetch with a private and loopback deny list, re-checked per redirect, plus size and time caps. Don't rely on the macOS Local Network prompt: it doesn't cover localhost or WKWebView traffic.
6. **Supply chain (do this now, even for the web app):**
   - Add `minimumReleaseAge: 1440` (ideally 4320) to `pnpm-workspace.yaml`, then move to pnpm 11 (`strictDepBuilds`, `blockExoticSubdeps`) and consider `trustPolicy: no-downgrade`.
   - Pin GitHub Actions by SHA and avoid `pull_request_target`.
   - For Rust: `cargo-deny` (advisories, licences, sources) in CI, `--locked`, and cargo-vet imports if time allows.
   - Treat agent sessions as a target: ChainDrop plants Claude Code hooks, so review `.claude/` changes in PRs.
7. **Telemetry:**
   - Off by default, asked once, crash-only.
   - Sentry SaaS free tier (5k events) with `beforeSend` scrubbing; no replay and no DOM or console breadcrumbs.
   - Offer "copy diagnostics" as the no-network fallback.
   - Publish a short privacy page modelled on Obsidian's.
8. **Licensing:**
   - Keep `@blocknote/xl-*` out of the desktop app unless you buy the commercial licence or the app is GPL/AGPL. Never ship xl-* in a MAS build without the commercial licence.
   - For "may sell later or open-source", the most flexible choice is to **stay closed or all-rights-reserved now** (sole author, so every option stays open). Later, either:
     - **AGPL client + commercial dual licence** (requires a CLA), or
     - **FSL-1.1-ALv2 for the sync server, MIT/Apache client** (Sentry/PowerSync style).
   - Avoid GPL-only if the MAS matters.
   - All candidate sync and CRDT libraries are permissive except the PowerSync service (FSL, fine for self-use).
9. **Sync:** design for E2EE from the start (encrypted CRDT ops, signed or MACed by the device, recovery phrase), so that a sync-server breach exposes only metadata. Accept that server-side AI and search then move on-device.

## 9. Not verified in this session (summary)
- Apple's exact current review-time statement.
- Whether MetricKit delivers payloads to Developer ID builds.
- Data Protection classes on macOS; the TCC app-container protection details.
- Sparkle licence and key-rotation procedure.
- Whether the Tauri updater re-verifies the Apple code signature.
- update.electronjs.org terms.
- The exact Sentry free-tier numbers on sentry.io.
- GlitchTip licence.
- BlockNote commercial price.
- BUSL and fair.io definitions.
- VLC/FSF App Store history and the local-first essay text (training data; the primary sites were blocked).
- PGlite's PostgreSQL-licence notice.
- SQLite public-domain page.
- GitHub-hosted macOS runner minute multipliers.
- CVE-2026-61704 details.

Searches were capped at 200 per session, and that cap was reached while the licensing section was being researched.
