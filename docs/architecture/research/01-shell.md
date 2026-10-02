# 01: App shell and UI stack for a local-first Mac app (later phone; web stays)

Research date: 2026-10-02. Scope: desktop/mobile shell + UI framework for the Sprint "personal OS" (today Next.js 16.3 + React 19.2 + BlockNote 0.55 on Vercel/Supabase).

## Method and caveats (read first)

- The research sandbox's egress proxy blocked direct fetches of most vendor sites (tauri.app, electronjs.org, notion.com, blackboard.sh, powersync.com, dolthub.com, gethopp.app, news.ycombinator.com and others). Evidence comes from: (a) the **npm registry** (versions and publish timestamps, queried live on 2026-10-02: hard facts), (b) **GitHub pages** fetched directly (releases, issues, discussions, PRs: hard facts), and (c) **web-search result summaries** of pages I could not open. Claims from (c) are marked **[search summary]**: the URL and the date the search engine gave are cited, but I did not read the full page. Treat their exact numbers as indicative.
- The session's web-search budget ran out near the end. A few items are therefore marked **[unverified]**.
- "Accessed" means 2026-10-02 unless a date is given.

---

## 1. Tauri 2

### Version and cadence
- **Current stable: Tauri 2.12** (announced 26 Sep 2026; "the biggest update so far in the 2.x releases"). `@tauri-apps/cli` and `@tauri-apps/api` **2.12.1** were published 2026-09-30 (npm registry, accessed). Announcement: https://v2.tauri.app/blog/tauri-2.12/ **[search summary, 2026-09-26]**. GitHub release: https://github.com/tauri-apps/tauri/releases/tag/tauri-v2.12.0 (fetched; the fetch tool printed "September 26, 2025", but the notes mention macOS 27 and npm dates it 2026, so it is 2026).
- Cadence: 2.10.1 (2 Feb 2026), 2.11.2 (16 May), 2.11.3 (17 Jun), 2.11.5 (1 Jul), 2.12 (26 Sep). That is a **minor release about every 3–4 months, with patches every few weeks**. Stable since 2 Oct 2024. https://tauri.app/release/core/ and https://docs.rs/crate/tauri/latest **[search summary]**.
- 2.12 highlights (GitHub release notes, fetched): a **security fix GHSA-w28w-mhc8-qvjv** ("Channel data IPC queue entries are now bound to specific webviews, preventing other webviews from accessing queued payloads"); a webview permission-request handler; `appDirectoriesOverride`; macOS glass window effects; Windows 7 dropped; MSRV 1.90.
- **Tauri 3 is in alpha** (tauri v3.0.0-alpha.4, 1 Oct 2026), with an official **`tauri-runtime-cef` (Chromium Embedded Framework) runtime** at v3.0.0-alpha.5 (1 Oct 2026; alpha.1 on 15 Sep bumped CEF to 152.3.0). Tauri 3 also removes the `macos-private-api` feature. Sources: https://github.com/tauri-apps/tauri/releases (fetched), https://github.com/tauri-apps/tauri/releases/tag/tauri-runtime-cef-v3.0.0-alpha.1 (fetched). **This matters:** Tauri's long-standing WebKit pain gets a **bundled-Chromium escape hatch**, but only in an alpha major release. I found no stable ETA.

### macOS WKWebView vs Chromium
- **Rendering and feature drift:** "Safari is the new IE" is the most common regret. Scott Tolinski (Syntax.fm) moved a project from Tauri/Electrobun to Electron "because of how many issues I was having that were entirely Safari based" (https://x.com/stolinski/status/2024134641364709506, Feb 2026 **[search summary]**). Figma reportedly could not use Tauri because of WebKit canvas differences (https://www.buildmvpfast.com/blog/tauri-v2-vs-electron-desktop-apps-2026 **[search summary, 2026]**). The CSS layout differs between WKWebView, WebView2 and WebKitGTK (scroll-driven animations, advanced grid) (https://tech-insider.org/tauri-tutorial-cross-platform-rust-app-2026/ **[search summary]**).
- **Linux WebKitGTK is worse than macOS:** a Tauri maintainer said they cannot "100% recommend tauri (for Linux) as of now". Users report 40 fps vs 240 fps in Chromium. CEF work resumed in Nov 2025. https://github.com/orgs/tauri-apps/discussions/8524 (fetched; thread Jan 2024 – Jul 2026). This is not relevant for a Mac-first app, but it is relevant if Linux ever matters.
- **Devtools:** the inspector on macOS is Safari Web Inspector, not Chrome DevTools/CDP. Enabling devtools in release builds uses a **private API and blocks Mac App Store acceptance** (https://v2.tauri.app/develop/debug/ **[search summary]**). Automated e2e tests are harder: someone had to build a WebDriver for WKWebView Tauri apps (https://danielraffel.me/2026/02/14/i-built-a-webdriver-for-wkwebview-tauri-apps-on-macos/ **[search summary, 2026-02-14]**). Today's repo uses Playwright on Chromium, which does not transfer directly.
- **Embedded-browser auth:** Google OAuth refuses embedded WKWebViews (https://getatrium.dev/blog/embedding-real-browser-tauri **[search summary, 2026]**). Sprint uses a Supabase magic link, so this matters only if Google sign-in is added. The usual workaround is a system-browser deep link.
- **WebGPU:** shipped and on by default in Safari 26 / macOS Tahoe 26 (https://webkit.org/blog/17333/webkit-features-in-safari-26-0/ **[search summary, Sep 2025]**). Whether it is available inside a WKWebView is unclear (one guide hints that embedded views may lack it) **[unverified]**. The app does not need WebGPU.
- **OPFS / SQLite-WASM:** OPFS `createSyncAccessHandle` works only in dedicated workers. The official `@sqlite.org/sqlite-wasm` (3.53.4, npm 2026-10-02) supports OPFS VFSes. Notion had to use a SharedWorker so only one tab writes, because concurrent OPFS writers corrupted the DB (https://www.notion.com/blog/how-we-sped-up-notion-in-the-browser-with-wasm-sqlite **[search summary]**). WKWebView-specific OPFS durability/eviction behaviour in a Tauri custom-protocol origin: **[unverified]**. In Tauri the safer choice is native SQLite in Rust (`@tauri-apps/plugin-sql` 2.5.0, npm 2026-09-26) or rusqlite, not OPFS.
- **Memory caveat:** the popular "Tauri uses ~50% less RAM" figures count Chromium's shared pages naively. A Tauri issue showed that with PSS/USS accounting **WebKit-based apps used more memory than Chromium ones** (>90 MB difference in tests). https://github.com/tauri-apps/tauri/issues/5889 (fetched; opened 2022-12-21, closed). A Hoppscotch user saw the Tauri app above 1 GB with 7 tabs (https://github.com/hoppscotch/hoppscotch/discussions/3775 **[search summary]**). **Bundle size is a real win; RAM mostly is not.**

### Mobile (iOS/Android) maturity
- Desktop is production-ready. Mobile works but is "relatively young": plugin coverage lags desktop, and platform-specific code is often needed. "Production-critical mobile apps are still better served by flutter_rust_bridge or UniFFI" (https://rustify.rs/articles/rust-tauri-vs-flutter-2026 **[search summary, 2026]**). 2.12 still ships Android fixes and breaking path changes (GitHub release notes, fetched). **Verdict: usable for a personal app's phone companion, and riskier than Capacitor or React Native for a polished mobile product.**

### Security model (capabilities/permissions)
- Default-deny ACL. **Capabilities** bind permission sets to window/webview labels. **Permissions** allow or deny commands. **Scopes** constrain arguments (for example fs paths). The Rust core enforces them at runtime and checks them at build time. The webview is treated as untrusted. https://v2.tauri.app/security/permissions/, https://v2.tauri.app/reference/acl/capability/ **[search summary]**. This model is stronger by default than Electron, where the developer has to wire up `contextBridge` safely.

### IPC performance
- v2 rewrote IPC. It supports **raw (binary) requests and responses** that skip JSON. **Channels** stream data; above 8 KB JSON or 1 KB raw they switch from `eval` to the fetch-based custom protocol (https://v2.tauri.app/blog/tauri-20/ and docs.rs channel.rs **[search summary]**).
- Numbers: ~**5 ms on macOS vs ~200 ms on Windows for 10 MB** binary (anecdotal). A maintainer says IPC "still uses the fetch api… with the currently used system webviews I believe that we can't improve it much further." https://github.com/orgs/tauri-apps/discussions/11915 (fetched). Third-party tauri-conduit: 64 KB round trip **2.27 ms via invoke vs 0.20 ms binary** (https://github.com/userFRM/tauri-conduit/blob/master/BENCHMARKS.md **[search summary]**).
- **Implication:** keep chatty data access on one side. Either run the DB/query layer in the webview (TS) and use Rust only for OS access, or run it in Rust and send coarse-grained results. Do not run per-row IPC.

### Sidecars (for example Node)
- Official guide: "Node.js as a sidecar" (https://v2.tauri.app/learn/sidecar-nodejs/ **[search summary]**). Sidecars must be self-contained binaries. Options are `bun build --compile` (~50–80 MB, fast) or `@yao-pkg/pkg`; the original `pkg` is archived (https://dev.to/riponcm/shipping-a-nodejs-server-as-a-native-desktop-app-with-tauri-and-bun-mok **[search summary]**). One report: ~28 MB .deb with a Node sidecar vs 7 MB pure Rust (https://www.threads.com/@codeforreal/post/C74cDXuS0ja **[search summary]**).
- Cons: per-architecture binaries, a second process lifecycle, localhost ports or stdio protocol, codesigning/notarization of the extra binary, and MAS sandbox inheritance. Above all, **it erases most of Tauri's size win and re-creates "Electron with extra steps."**

### Bundle size / memory
- Often-quoted: hello-world Tauri **~3 MB vs ~85 MB Electron**; idle **~42 MB vs ~168 MB** (https://tech-insider.org/tauri-vs-electron-2026/ **[search summary, Sep 2026]**). Hoppscotch: **165 MB → 8 MB** bundle and "70% less memory" (same source and openalternative.co **[search summary]**). See the memory caveat above.

### Mac App Store
- Supported, with an official guide (https://v2.tauri.app/distribute/app-store/ **[search summary]**). It needs App Sandbox plus `network.client`/`network.server` entitlements. Private APIs (devtools in release, `macos-private-api` transparency) must be off. Tauri 3 removes the private-API transparency path entirely (GitHub releases, fetched).

### Production apps
- GitButler (Svelte + Rust), Hoppscotch, Cap (screen recorder), Spacedrive, Clash Verge, Gitify: https://openalternative.co/stacks/tauri, https://tech-insider.org/tauri-vs-electron-2026/ **[search summary]**. Note: search summaries also listed **AppFlowy** and **Sourcegraph Cody** as Tauri apps. AppFlowy's shipping clients are Flutter (see §5), and the Tauri Cody app was discontinued years ago. **Treat those two claims as wrong.**

### Failure modes / complaints (2025–2026)
1. WebKit-only bugs and drift from the Chromium web build (above).
2. Linux WebKitGTK performance.
3. Hard e2e testing on macOS (no CDP).
4. Rust learning curve and compile times for a TS-only solo dev.
5. Mobile plugin gaps.
6. Memory claims that do not hold.
7. IPC-channel security bug fixed in 2.12.

---

## 2. Electron

### Version and cadence
- **Current stable: Electron 44** (released 25 Aug 2026; Chromium M152, Node 24). Latest patch is **44.5.1**, npm 2026-09-30. **A major every 8 weeks; the 3 latest majors are supported**: 44 until 2 Mar 2027, 43 until 5 Jan 2027, 42 until 20 Oct 2026. https://releases.electronjs.org/, https://endoflife.date/electron **[search summary]**; npm (accessed).
- 2026 majors: 40 (15 Jan), 41 (10 Mar), 42 (5 May), 43 (30 Jun), 44 (25 Aug). **About 6 forced majors a year** is the main maintenance tax.

### Security model
- `contextIsolation` on by default since **Electron 12**. Renderer **sandbox** on by default since **Electron 20** (https://www.electronjs.org/blog/electron-20-0, https://www.electronjs.org/docs/latest/tutorial/context-isolation **[search summary]**).
- **Fuses** are package-time toggles: `RunAsNode`, `NodeCliInspect`, `EnableEmbeddedAsarIntegrityValidation`, `OnlyLoadAppFromAsar`. Together, ASAR-integrity and only-load-from-asar make it "impossible to load non-validated code", with codesigning protecting the fuse bits (https://www.electronjs.org/docs/latest/tutorial/fuses, https://www.electronjs.org/docs/latest/tutorial/asar-integrity **[search summary]**). `@electron/fuses` 2.1.3 (npm 2026-06-29).
- Past bypass advisories exist, for example GHSA-7m48-wc93-9g85 (filetype confusion, Windows) (https://github.com/electron/electron/security/advisories/GHSA-7m48-wc93-9g85 **[search summary]**).
- The model is "secure if you follow the checklist". Tauri's is "deny unless granted".

### Size / memory
- Hello world ~85 MB; real apps 120–200 MB installers; idle 150–250 MB (https://tech-insider.org/tauri-vs-electron-2026/, https://www.buildmvpfast.com/blog/tauri-v2-vs-electron-desktop-apps-2026 **[search summary]**). See the Tauri memory caveat: the real-app RAM gap is smaller than these numbers suggest.

### Mac App Store
- Supported via the "MAS build" of Electron, which runs under App Sandbox. **crashReporter and autoUpdater are disabled in the MAS build**, and some accessibility, DNS-change and video-capture limitations apply (https://www.electronjs.org/docs/latest/tutorial/mac-app-store-submission-guide **[search summary]**). `@electron/osx-sign` now fails fast when the `app-sandbox` entitlement is missing (https://github.com/electron/osx-sign/pull/432 **[search summary]**).

### Production apps
- VS Code, Slack, Discord, Notion, 1Password 8, Signal, ChatGPT (Windows), Codex Desktop, **Claude Desktop**, Obsidian, Linear desktop. Leaving: Teams (WebView2), Zed (native) (https://codenote.net/en/posts/famous-electron-apps-2026-research/, https://www.dbreunig.com/2026/02/21/why-is-claude-an-electron-app.html **[search summary, 2026]**). "Linear desktop is Electron" is widely reported but **[unverified directly]**.

### utilityProcess and native modules (better-sqlite3)
- `better-sqlite3` 13.0.3 (npm 2026-08-05) is synchronous. In the main process a slow query blocks tray, menus and all window IPC: one report measured p50 1.9 ms → 159 ms freezes (https://github.com/yinxulai/one-switch/issues/9 **[search summary]**).
- **Trilium spike, 18 Sep 2026:** moving SQLite and the backend into a `utilityProcess` with direct renderer `MessagePort`s cut **main-process stalls from 2993 ms to 24 ms** and per-request latency from **0.56 to 0.17 ms** on a 22k-note DB. The PR was closed as exploratory, with crash-handling and packaging issues left open (https://github.com/TriliumNext/Trilium/pull/11572, fetched).
- **Pattern:** DB in a `utilityProcess` (or worker_thread), with renderers talking to it through MessagePorts. Native modules must be rebuilt per Electron ABI (`@electron/rebuild`) at every major upgrade, which happens 6 times a year. Node's built-in `node:sqlite` avoids the native-module rebuild but is also synchronous.

---

## 3. Native Swift/SwiftUI (+AppKit)

- **No mature, open-source, Notion-style native block editor exists.** Open-source native options are Markdown/TextKit 2 editors: LapermEditor (TextKit 2, Swift 6) and swift-markdown-engine (AppKit, the editor inside "Nodes") (https://github.com/k-ymmt/LapermEditor, https://github.com/nodes-app/swift-markdown-engine **[search summary]**). SwiftUI `TextEditor` gained `AttributedString` rich text and `AttributedTextSelection` in iOS/macOS 26 (https://www.hackingwithswift.com/quick-start/swiftui/how-to-use-rich-text-editing-with-textview-and-attributedstring **[search summary]**). That is fine for a single rich-text field, but it is not a block editor with slash menu, drag handles, nested blocks, embeds or collaborative CRDT binding.
- **What the native apps do:**
  - **Craft**: Mac Catalyst plus a custom in-house sync protocol, one codebase for iOS/iPadOS/macOS/visionOS, and its own block editor (https://www.craft.do/blog/create-first-class-visionos-experience **[search summary]**).
  - **Bear 2**: its own editor "Panda", with a **C++ AST core** plus ObjC/Swift. It was spun out in June 2026 as **Lettera** (https://blog.bear.app/2026/06/introducing-lettera-a-native-markdown-editor-for-mac-now-in-beta/ **[search summary, 2026-06]**; Panda history: https://blog.bear.app/2021/06/checking-in-on-panda-the-next-editor-for-bear/). Bear took years to build Panda.
  - **Things 3**: fully native Apple-only app; 3.23/3.24 shipped Aug–Sep 2026 (https://www.macrumors.com/2026/09/14/things-3-updated-for-ios-27/ **[search summary]**).
  - **Apple Notes**: Apple-internal TextKit stack **[unverified: no public engineering source found]**.
- **Cost of sharing logic with a web app:** a native UI cannot reuse BlockNote/React. Options are to (a) duplicate domain logic in Swift, (b) run shared TS logic in JavaScriptCore (`JSContext`) with a bridge, or (c) move core logic to Rust and use UniFFI for both Swift and WASM. Every option means a second UI codebase, plus a block editor written from scratch. A block-document format compatible with BlockNote JSON would need its own native renderer and editor. **For a solo dev this is a multi-year commitment (see Bear/Panda). Rule it out for the editor; native is reasonable only for small satellites (widgets, Share extension, Shortcuts, menu-bar capture).**

---

## 4. Other options

| Option | Version / status (dated) | Notes | Fit |
|---|---|---|---|
| **Electrobun** | **2.0.2** (npm/GitHub 2026-09-29); 1.x earlier in 2026. 12.9k stars, 112 open issues (README, fetched) | 2.0 moved the core to Zig, added its own build CLI "Hutch" and its **own JS runtime "Cottontail"** (JavaScriptCore + Zig). It decoupled from Bun after Anthropic's Bun→Rust rewrite; the author cited the rewrite's rollout practices (https://x.com/YoavCodes/status/2058064720553222567; https://blackboard.sh/blog/electrobun-2-0/ **[search summary, 2026]**). System webview or optional bundled CEF; ~14 MB apps, ~14 KB diff updates. macOS 14+, Win 11, Ubuntu 24.04. **No mobile.** | Interesting, but it has had a runtime swap within a year and has a single maintainer. Too risky as the foundation. |
| **Wails 3** | **Beta** since 2 Aug 2026; latest 3.0.0-beta.25/27 (npm `@wailsio/runtime` 3.0.0-beta.27, 2026-10-01). GA tracker: 27 blocking gates, none passed (https://github.com/wailsapp/wails/issues/5844, fetched) | Go backend, system webview (the same WebKit issues as Tauri). Mobile "experimental, does not block GA". | No: Go backend, same webview trade-off, not GA. |
| **Dioxus / Rust-native** | Dioxus **0.7** released 23 Jan 2026 with "Dioxus Native" (Blitz renderer on WGPU) (https://dioxuslabs.com/blog/release-070/ **[search summary]**). Blitz is alpha, "not ready for production" (https://blitz.is/about **[search summary]**) | Rewrites the UI in Rust, with no BlockNote/ProseMirror. | No. |
| **Flutter desktop** | Mature on macOS; used by AppFlowy (§5). Rich-text options are super_editor ("early stage"; long without a standard release) and flutter_quill (https://pub.dev/documentation/super_editor/latest/ **[search summary]**) | Dart rewrite. AppFlowy shows it is possible, but they wrote their own editor (appflowy_editor). | No: throws away the TS/React investment. |
| **React Native macOS / Expo** | RN 0.87.1 (npm 2026-08-26); `react-native-macos` 0.83.0 (npm 2026-09-30), trailing upstream. **Expo Desktop** announced at App.js Conf, 29 May 2026 (https://expo.dev/blog/expo-highlights-new-products-and-plans-for-the-future **[search summary]**); earlier, Expo said the macOS fork lag made it "prohibitively expensive" (https://platform.uno/articles/expo-windows-macos-desktop-app/ **[search summary]**) | Native UI with shared React, but **no web block editor**. You would host BlockNote in a WebView anyway (a "DOM component"). | Only if the phone app must be native-feeling. The editor still runs in a webview. |
| **Capacitor** | **8.5.2** (npm 2026-09-11). SQLite plugins are maintained (`@capacitor-community/sqlite` 8.1.1, Aug 2026) (https://capgo.app/blog/capacitor-community-sqlite-alternative/ **[search summary]**) | The web app in WKWebView on iOS/Android. **Obsidian's mobile apps use it** (https://forum.obsidian.md/t/what-technology-obsidian-mobile-is-developed-with/40125 **[search summary]**). | Strong candidate for the **phone app**, whichever desktop shell is chosen. |

---

## 5. Real-world local-first apps

| App | Shell | Local DB | Sync | Source |
|---|---|---|---|---|
| **Linear** | Web + Electron desktop **[Electron: unverified directly]** | **IndexedDB** with an in-memory MobX object graph | Own sync engine: local mutation → transaction queue persisted in IDB → server → WebSocket deltas | https://github.com/wzhudev/reverse-linear-sync-engine, https://www.fujimon.com/blog/linear-sync-engine **[search summary]** |
| **Obsidian** | **Electron** (desktop), **Capacitor** (mobile); custom UI, no React | Plain Markdown files on disk | Obsidian Sync (proprietary) or 3rd-party file sync | https://forum.obsidian.md/t/what-technology-obsidian-mobile-is-developed-with/40125 **[search summary]** |
| **Anytype** | **Electron** + React + MobX (`anytype-ts`) | Go middleware **anytype-heart** over gRPC | **any-sync** (Go): encrypted P2P/CRDT spaces, self-hostable nodes | https://github.com/anyproto/anytype-ts, https://tech.anytype.io/any-sync/overview **[search summary]** |
| **AFFiNE** | **Electron** (v39 per DeepWiki) + BlockSuite editor | SQLite via Rust (NAPI-RS); OctoBase | **y-octo**, a Rust Yjs-compatible CRDT | https://github.com/toeverything/AFFiNE, https://github.com/toeverything/OctoBase, https://deepwiki.com/toeverything/AFFiNE/1.1-architecture-overview **[search summary]** |
| **AppFlowy** | **Flutter** (all platforms) | Rust core (FlowySDK via Dart FFI); SQLite plus collab docs | **yrs** CRDT (AppFlowy-Collab) | https://appflowy.com/blog/tech-design-flutter-rust, https://github.com/AppFlowy-IO/AppFlowy-Collab **[search summary]** |
| **Logseq DB** | **Electron** (pinned 39.8.8 in spring 2026) + web | **SQLite** (official `@sqlite.org/sqlite-wasm`, OPFS in web) plus DataScript | RTC sync server (DB version) **[RTC details unverified]**. Logseq 2.0 beta announced 13 Jul 2026 | https://discuss.logseq.com/t/logseq-db-changelog/30013?page=2, https://news.ycombinator.com/item?id=48896229 **[search summary]** |
| **Tana** | Desktop app (offline mode added); framework **[unverified]** | Local graph download and local search indexes | Cloud; read-only offline for shared workspaces | https://outliner.tana.inc/blog/tana-desktop-now-works-offline-your-knowledge-graph-anywhere **[search summary]** |
| **Capacities** | Desktop/mobile/web; framework **[unverified]** | Own local DB (not files) | Offline-first, syncs when online | https://docs.capacities.io/misc/offline-support **[search summary]** |
| **Heptabase** | **Electron** | Local DB | AWS sync when multi-device is on | https://wiki.heptabase.com/version-one **[search summary]** |
| **Reflect** | Desktop + iOS; framework **[unverified]** | Local, offline mode | **E2EE**: XChaCha20-Poly1305 with Argon2id key | https://reflect.academy/security-and-encryption **[search summary]** |
| **Notion** | **Electron** desktop; web | Desktop: native SQLite cache for years. Web: **WASM SQLite on OPFS** in a **SharedWorker** (only the active tab writes; 20% faster navigation) | Offline mode (Aug 2025): the SQLite cache became a persistent store, and offline-marked pages are **migrated to a CRDT model**. DB properties are last-writer-wins; the first 50 rows of a DB view are downloaded | https://www.notion.com/blog/how-we-sped-up-notion-in-the-browser-with-wasm-sqlite, https://www.notion.com/blog/how-we-made-notion-available-offline, https://alternativeto.net/news/2025/8/notion-rolls-out-offline-mode-edit-pages-without-an-internet-connection **[search summary]** |
| **Craft** | **Native (Mac Catalyst)** | Native **[DB unverified]** | Custom in-house sync protocol | https://www.craft.do/blog/create-first-class-visionos-experience **[search summary]** |
| **Excalidraw** | Web/PWA | localStorage (scene) + IndexedDB (files) | E2EE collaboration rooms | https://deepwiki.com/zsviczian/excalidraw/7.3-local-data-persistence **[search summary]** |
| **tldraw** | Web SDK | IndexedDB via `persistenceKey` | `@tldraw/sync`: one Cloudflare Durable Object with SQLite per board, over WebSockets | https://tldraw.dev/docs/persistence **[search summary]** |

**Pattern:** every Notion-class block editor that ships to many users on desktop is **web tech (mostly Electron)**. AppFlowy is the exception (Flutter, with its own editor) and Craft is the native exception (with a custom editor). Tauri hosts no notable Notion-class app among those checked.

---

## 6. UI framework and app framework

### React 19 vs Svelte 5 / Solid
- Versions (npm, accessed): React **19.3.0** (2026-09-09; the repo uses 19.2.8), Svelte **5.57.1**, Solid **1.9.15**. **React Compiler 1.0** has been stable since 2025-10-07 (npm `babel-plugin-react-compiler`), which removes much of React's manual-memo cost.
- **Editor ecosystem (npm peer deps, accessed):** `@blocknote/react`/`shadcn`/`mantine` 0.55.0 require React 18/19. BlockNote's own docs say the vanilla `@blocknote/core` path means "writing your own UI elements, and is not recommended" (https://www.blocknotejs.org/docs/getting-started/vanilla-js **[search summary]**). Tiptap 3.31 has first-party React and Vue bindings; Svelte is community-only (`svelte-tiptap` 3.0.1). Lexical 0.52 has `@lexical/react`; Svelte is community (`svelte-lexical` 0.6.5). ProseMirror is framework-free.
- **What a local desktop app gains by leaving React:** smaller runtime and finer-grained reactivity (Svelte/Solid). That matters little: in a local app the bottleneck is data access and editor size, not VDOM diffing, and bundle bytes are free on desktop. **What it loses:** BlockNote (the current editor and the BlockNote JSON in `objects.body`), shadcn/ui, cmdk, the existing components, and the shared code with the web app. **Verdict: keep React 19 (with the Compiler).**

### Next.js inside a desktop shell
- Tauri's official Next.js guide requires `output: 'export'` (static) with `images.unoptimized` (https://v2.tauri.app/start/frontend/nextjs/ **[search summary]**). Static export **cannot use Server Actions** (build error "Server Actions are not supported with static export"), RSC data fetching at request time, route handlers, middleware/proxy, cookies, rewrites/redirects, ISR, or dynamic routes without `generateStaticParams` (https://nextjs.org/docs/app/guides/static-exports, https://github.com/vercel/next.js/discussions/67503 **[search summary]**).
- **The current repo (read-only check, 2026-10-02):** 29 files with `"use server"`, 78 files use `server-only`/`getCurrentAccount`, 7 route handlers, 25 `page.tsx`. Almost all data flow is server-side, so a static export would keep the components but **replace the whole data path**. A local-first app needs that rewrite anyway: queries move to a local DB.
- Electron *could* run a full Next server (Node) in-process, but that keeps a localhost server inside the app. It is unusual, slow to start and awkward for MAS. Not recommended.
- **Alternative: Vite 8.3 + React 19 + TanStack Router 1.170** (npm, accessed) as a pure SPA that runs identically in Tauri/Electron/Capacitor/browser. TanStack Router has type-safe routes and search params, which suits the existing `?link=`/`?project=` URL-state pattern. The community consensus for Tauri is SPA, not Next (https://github.com/orgs/tauri-apps/discussions/6083 **[search summary]**). Next.js can stay as the hosted web shell (or be replaced by the same SPA plus a thin API) during transition.

---

## 7. Scored comparison (1 = poor, 5 = best; for this project: solo TS dev, Mac first, phone later, web stays, block editor, AI)

| Criterion | **Tauri 2** (+Vite/React SPA) | **Electron 44** (+Vite/React SPA) | Native Swift/SwiftUI | Electrobun 2 | Wails 3 | Flutter | RN macOS/Expo |
|---|---|---|---|---|---|---|---|
| Performance (startup, size, RAM) | 4 (tiny bundle; RAM ≈ parity) | 3 | 5 | 4 | 4 | 4 | 4 |
| Security (defaults) | 5 (default-deny ACL) | 4 (good with fuses + checklist) | 5 | 3 | 3 | 4 | 4 |
| Stability / maturity | 4 (desktop) / 3 (mobile) | 5 | 5 (platform); 1 (block editor) | 2 | 2 (beta) | 4 | 3 |
| Dev speed (for this codebase) | 4 (Rust for the native layer; WebKit testing) | 5 (all TS, Chromium = web, Playwright) | 1 | 3 | 3 | 1 | 2 |
| Domain fit (block editor, graph, AI, offline) | 4 (Safari drift risk) | 5 (what Notion/AFFiNE/Anytype/Logseq ship) | 2 | 3 | 3 | 2 | 2 |
| Cost (≈$0; Apple dev $99/yr for all) | 5 | 5 | 5 | 5 | 5 | 5 | 5 |
| Lock-in (low = 5) | 4 (web UI portable; Rust glue) | 4 (web UI portable; Node glue) | 1 | 3 | 3 | 1 | 2 |
| Mobile path | 3 (same shell, young) | 2 (needs Capacitor, as Obsidian does) | 5 (but a second UI) | 1 (none) | 1 (experimental) | 5 | 4 |
| **Total (/40)** | **33** | **33** | 29 | 24 | 24 | 26 | 26 |

---

## 8. Recommendation

1. **UI: keep React 19 + BlockNote, and move from Next.js to Vite + React + TanStack Router as a pure SPA** that runs unchanged in a browser, a desktop shell and a mobile webview. Domain logic stays in `packages/core`, and the data path moves from Server Actions to a local-DB/repository layer. This is the same work whichever shell is picked.
2. **Shell: Tauri 2 and Electron tie on score, and they win on different axes.**
   - **Choose Electron 44** if the priority is *the Mac app shipping fast and behaving exactly like the web build*: same Chromium as Playwright today, all-TypeScript main process (Anthropic SDK and better-sqlite3/`node:sqlite` in a `utilityProcess`), and the stack Notion, AFFiNE, Anytype, Logseq, Heptabase, Obsidian and Claude Desktop ship. The phone app is then a **Capacitor** build of the same SPA (the Obsidian model). Costs: ~100+ MB bundle and six majors a year.
   - **Choose Tauri 2** if *one shell for Mac and phone* and a small, default-deny app matter more, and you accept Safari/WebKit testing plus some Rust. Run SQLite natively (plugin-sql/rusqlite) behind coarse IPC. Tauri 3's CEF runtime (alpha, Oct 2026) may later remove the WebKit risk on desktop.
   - **My lean is Electron for desktop plus Capacitor for the phone, with the shell layer kept thin** (a `platform` adapter for DB, files, notifications and deep links) so a later switch to Tauri, once its CEF runtime is stable, is cheap. The reason is that the dominant risk for a solo dev is time lost to WebKit-only editor bugs: ProseMirror/BlockNote in Safari, contenteditable quirks, and drag-and-drop. Electron removes that risk on the platform used most (Mac), and the phone build has to live with WKWebView whatever the choice.
3. **Do not** go native Swift for the main UI (no native block editor; Bear needed years for Panda). Use Swift only for small extensions (widgets, Share sheet, Shortcuts).
4. **Do not** pick Electrobun (runtime churn, single maintainer, no mobile), Wails 3 (beta, Go) or Dioxus/Flutter (UI rewrite).

### Open items to verify by hand (blocked in this session)
- Read the full Tauri 2.12 post and the Tauri 3/CEF roadmap; check whether a stable ETA exists.
- Check OPFS/IndexedDB durability inside WKWebView under a Tauri custom protocol and inside a Capacitor iOS app.
- Check WebGPU availability in WKWebView (not needed today).
- Confirm the desktop frameworks of Linear, Tana, Capacities and Reflect.
- Prototype BlockNote 0.55 in WKWebView (Safari 26) to see how bad WebKit-only editor bugs actually are. This is the deciding experiment between Tauri and Electron.
