# Research: app shell, where the logic lives, UI framework, rich-text editor

- **For:** ADR-0001 (founding architecture)
- **Date:** 2026-10-02. All links below were accessed on this date unless a source date is given.
- **Status:** Research input, not a decision.

## TL;DR

1. **Shell: Tauri 2** (Rust core + system WKWebView) for the Mac app. Electron is the safe fallback; native SwiftUI is the best Mac app but costs a second UI for web and has no usable CRDT-aware block editor.
2. **Logic: hybrid, Rust-heavy.** Rust owns storage, sync, CRDT host, crypto, search, and the domain mutations, so the same rules run on the Mac, the hub, the phone (UniFFI or Tauri mobile) and the web (wasm). TypeScript owns views, view state and the editor.
3. **UI: React 19 + React Compiler 1.0.** Chosen for the editor and keyboard-UI ecosystem, not for raw speed.
4. **Editor engine: ProseMirror.** Kit: **BlockNote (MPL core)** if the CRDT is Yjs; **Tiptap 3 (MIT) + loro-prosemirror** with our own block UI if the CRDT is Loro. A two-week spike decides; the CRDT research decides first.
5. Biggest risks: WKWebView quirks (IME, shortcuts, the 16-minute background suspend), Yjs 14 still pre-release, loro-prosemirror is young (v0.4.x, one main maintainer team).

---

## 1. Shell

### 1.1 Facts per option

**Tauri 2** (Rust + system webview: WKWebView on macOS/iOS, WebView2 on Windows, WebKitGTK on Linux, Android System WebView)
- Version: 2.12.0 released 2026-09-26, "the biggest update so far in 2.x" (permission-request handler, faster mobile dev loading, Android SDK 37) — [Tauri blog](https://v2.tauri.app/blog/tauri-2.12/), [releases](https://v2.tauri.app/release/). 2.0 stable was 2024-10-02.
- Size / RAM / start (blog measurements, thin methodology, treat as indicative): hello-world 3.2 MB vs 85 MB Electron; idle 42 MB vs 168 MB; cold start 380 ms vs 1,420 ms — [tech-insider.org, 2026](https://tech-insider.org/tauri-vs-electron-2026/). Hoppscotch reported 165 MB → 8 MB and ~70% less memory after moving from Electron — [tech-insider.org](https://tech-insider.org/tauri-vs-electron-2026/) (secondary source; unverified at Hoppscotch).
- Caveat on RAM: on macOS the page runs in a separate WebKit WebContent process ("tauri://localhost" in Activity Monitor), which naive measurements miss. Real apps report 900 MB+ in that process when the page leaks — [terax-ai #238](https://github.com/crynta/terax-ai/issues/238), [tolaria #1208](https://github.com/refactoringhq/tolaria/issues/1208). The webview's memory is as good as our JS; Tauri does not save us from that.
- Security model: the webview has zero native access by default; access comes only through **capabilities** (which windows/webviews get which **permissions**, narrowed by **scopes**). Optional **isolation pattern**: a sandboxed iframe intercepts and encrypts every IPC call with a per-run key — [Isolation](https://v2.tauri.app/concept/inter-process-communication/isolation/), [Capabilities](https://v2.tauri.app/security/capabilities/). Independent audit of 2.0 by Radically Open Security (2024-08-07): 11 High, 2 Elevated, 3 Moderate, 5 Low (e.g. IPC callable from any origin, dev server exposing disk), fixed before 2.0; isolation can be defeated on Windows and Android — [report PDF](https://fossies.org/linux/tauri/audits/Radically_Open_Security-v2-report.pdf), [summary](https://readoss.com/en/tauri-apps/tauri/tauri-permission-system-capabilities-acl-security-boundary). Mac App Store with App Sandbox is documented and done in practice — [Tauri App Store guide](https://v2.tauri.app/distribute/app-store/).
- IPC: v2 rewrote IPC to support raw binary payloads and `Channel` streaming (fetch-based above ~1 KB), so CRDT update bytes need not go through JSON — [Tauri 2.0 release](https://v2.tauri.app/blog/tauri-20/), [DeepWiki IPC](https://deepwiki.com/tauri-apps/tauri/3-ipc-and-communication).
- Mobile: iOS/Android ship and real apps are in the App Store (e.g. flow-like, a Rust-heavy app, April 2026 — [write-up](https://medium.com/@trivajay259/what-happens-when-you-ship-a-rust-heavy-app-to-the-ios-app-store-with-tauri-2-ddea0584bae7)). Consensus: works, but mobile plugins lag desktop, signing and webview quirks are rough; for mobile-first, people still pick React Native or Flutter — [tech-insider tutorial 2026](https://tech-insider.org/tauri-tutorial-cross-platform-rust-app-2026/). Rate: usable, not mature.
- Web: the UI is a normal web app, so it runs in a browser if the backend calls go through an interface (Tauri IPC on desktop, wasm/worker or HTTP to hub on web).
- Chromium option: a CEF runtime exists only on the unreleased `feat/cef` branch and a community extraction ("macOS/Windows compile-ported blind") — [tauri-runtime-cef](https://github.com/SableClient/tauri-runtime-cef). Do not plan on it.
- Production: GitButler (Svelte + Rust), Spacedrive v3, Hoppscotch, AppFlowy (Flutter UI, Tauri for web/desktop parts — unverified), Padloc — [awesome-tauri](https://github.com/tauri-apps/awesome-tauri), [GitButler](https://github.com/gitbutlerapp/gitbutler).
- Known failure modes on macOS (WKWebView):
  - **Background suspend:** WebKit suspends the WebContent process after ~16 min in background; under memory pressure it can fail to resume and the window becomes a dead shell — [openworker #665, 2026-09-16](https://github.com/andrewyng/openworker/issues/665). Must handle "renderer terminated → reload" ourselves ([example](https://github.com/xnmp/tauri-explorer/issues/942)).
  - **IME and keys:** Enter that confirms an IME conversion also triggers app shortcuts; dropped keystrokes reported with xterm.js; Cmd+C/V/X/A quirks in Monaco; Cmd+F not native; key events missing until first click; "unsupported key" beep — [monocode #115](https://github.com/hardbeat920/monocode/issues/115), [tauri #9385](https://github.com/tauri-apps/tauri/issues/9385), [tauri #5464](https://github.com/tauri-apps/tauri/issues/5464), [tauri #2626](https://github.com/tauri-apps/tauri/issues/2626), [wry #1177](https://github.com/tauri-apps/wry/issues/1177). Most are fixable in our code (check `isComposing`, build an app menu), but each costs time.
  - **Storage:** don't keep primary data in IndexedDB/OPFS inside the webview. WebKit quotas and eviction differ for embedded webviews and OPFS has different restrictions in WKWebView — [web.dev](https://web.dev/articles/storage-for-the-web), [opfs-checker](https://github.com/wendylabsinc/opfs-checker). Under Tauri, data lives in Rust (SQLite files), so this is avoided.
  - **Engine follows the OS:** the app gets the Safari version of the user's macOS. New CSS/JS features need a minimum macOS version; bugs differ across versions. Set a floor (e.g. macOS 14+) and test on it.

**Electron**
- Version: 44.5.1 (Chromium M152, Node 24), 2026-09-30; a new major every 8 weeks, only the last three supported — [releases](https://releases.electronjs.org/), [schedule](https://releases.electronjs.org/schedule). That means a forced upgrade at least every ~6 months.
- Size/RAM/start: ~80–200 MB bundles, ~3–4x Tauri's idle RAM and start time in the blog measurements above. Same Chromium everywhere: no engine drift, best DevTools, best IME/keyboard parity with Chrome.
- Security: contextIsolation (default since 12), renderer sandbox (default since 20), and **fuses** to turn off `runAsNode`, `NODE_OPTIONS`, inspector args, and to enforce ASAR integrity — [Security](https://www.electronjs.org/docs/latest/tutorial/security), [Fuses](https://www.electronjs.org/docs/latest/tutorial/fuses). Defaults still leave `runAsNode` on, so a hardened config is our job. Recurring CVEs in context isolation — [cvedetails](https://www.cvedetails.com/vulnerability-list/vendor_id-17824/product_id-44696/Electronjs-Electron.html), [2026 write-up](https://securityonline.info/electron-security-vulnerabilities-sandbox-escape-context-isolation/).
- Mobile: none. Web: same UI runs in a browser.
- Production: VS Code, Slack, Notion, Linear, Obsidian, Figma desktop.

**Native Swift/SwiftUI + AppKit**
- Best Mac fit: smallest RAM (tens of MB; unverified for our app), real App Sandbox, Spotlight/Shortcuts/Services, perfect IME and keys.
- But: no web story (a second UI in TS is needed anyway), and the **editor** is the blocker. SwiftUI got `TextEditor` with `AttributedString` rich text in macOS/iOS 26 — [Hacking with Swift](https://www.hackingwithswift.com/quick-start/swiftui/how-to-use-rich-text-editing-with-textview-and-attributedstring) — but a Notion-style block editor with drag handles, slash menu, inline mentions and a CRDT binding does not exist; it would be built on TextKit 2 (e.g. [STTextView](https://github.com/krzyzanowskim/STTextView)) and that is months of work by itself — [Cindori on building a rich text editor](https://cindori.com/developer/building-rich-text-editor).

**Others**
- **Electrobun** v1 (Feb 2026): Bun main process + system webview, ~12–14 MB bundles, 14 KB bsdiff updates; optional CEF (~100 MB) — [Electrobun v1](https://blackboard.sh/blog/electrobun-v1/), [InfoWorld](https://www.infoworld.com/article/4137964/first-look-electrobun-for-typescript-powered-desktop-apps.html). Same WKWebView issues as Tauri, small community, no mobile. Interesting only if the core is TypeScript.
- **Wails** v3 is still beta (beta.26, 2026-09-25); Go core — [Wails v3 beta](https://v3.wails.io/blog/wails-v3-beta/). Go has no first-class CRDT/editor story we need.
- **Dioxus** 0.7 (Rust UI, webview or its own wgpu renderer "Blitz", still WIP) — [release](https://dioxuslabs.com/blog/release-070/). No mature rich-text editor; would cut us off from ProseMirror.
- **Flutter**: one UI on all platforms, but rich text is its weak spot (AppFlowy's editor fights macOS IME/delta bugs — [flutter #101013](https://github.com/flutter/flutter/issues/101013), [appflowy-editor #1204](https://github.com/AppFlowy-IO/appflowy-editor/issues/1204)); web build is canvas-based, weak for a text-heavy app.
- **Compose Multiplatform** 1.12.1 (2026-09-22), iOS stable since 1.8 — [JetBrains](https://blog.jetbrains.com/kotlin/2025/05/compose-multiplatform-1-8-0-released-compose-multiplatform-for-ios-is-stable-and-production-ready/). Same editor gap as Flutter; desktop runs on the JVM.
- **React Native macOS**: 0.83 while core RN is at 0.87 — [npm](https://www.npmjs.com/package/react-native-macos), [RN releases](https://reactnative.dev/docs/releases). Lags core; rich text needs a webview anyway (e.g. Tiptap in a WebView).

### 1.2 Scoring (1–5, higher is better)

| | Perf | Security | Stability | Dev speed | Domain fit | Cost | Lock-in | Mobile | Web reuse | **Total** |
|---|---|---|---|---|---|---|---|---|---|---|
| **Tauri 2** | 4 | 5 | 4 | 4 | 5 | 5 | 4 | 3 | 4 | **38** |
| Electron | 2 | 3 | 5 | 5 | 4 | 5 | 4 | 1 | 5 | 34 |
| Native Swift | 5 | 5 | 5 | 2 | 2 | 5 | 2 | 4 (iOS) | 1 | 31 |
| Electrobun | 4 | 3 | 2 | 4 | 4 | 5 | 3 | 1 | 4 | 30 |
| Flutter | 4 | 4 | 4 | 3 | 2 | 5 | 2 | 5 | 2 | 31 |
| Compose MP | 3 | 4 | 4 | 3 | 2 | 5 | 2 | 4 | 2 | 29 |
| Dioxus | 4 | 4 | 2 | 3 | 2 | 5 | 3 | 2 | 3 | 28 |

"Domain fit" here is mostly: does a top-quality CRDT block editor exist for it.

**Recommendation:** Tauri 2. Mac uses WKWebView, logic in Rust, same UI in the browser. Keep the UI free of Tauri imports behind one `CoreClient` interface, so moving to Electron (or a native shell hosting the same webview) is a contained change.

**What would change it:**
- The spike finds WKWebView IME/shortcut/suspend problems we can't work around in the editor → Electron (same UI, same Rust core as a Node addon via napi-rs or as a sidecar).
- The user decides the Mac app must feel 100% native and accepts a separate web UI → SwiftUI shell + Rust core via UniFFI, with the editor still as a ProseMirror webview island.
- Tauri ships an official CEF runtime → that removes engine drift; re-evaluate.

---

## 2. Where domain logic lives

The question: one core in **Rust** (Tauri commands on desktop, UniFFI to Swift/Kotlin, wasm on web, native on the hub) or in **TypeScript** (webview, Node/Bun/Deno on hub, Hermes in React Native)?

| | Rust core | TS core | Hybrid (Rust data + TS rules) |
|---|---|---|---|
| Correctness | 5: enums, exhaustive match, no null, no `any`; refactors are safe | 3: strict TS + zod helps, but runtime gaps stay | 3: rules split across a boundary |
| AI-assisted speed for one person | 3: slower compile/iterate; AI writes good Rust but borrow-checker loops cost time | 5: fastest loop, hot reload | 4 |
| Sharing with hub | 5: same binary | 4: needs a JS runtime on the hub | 3: hub needs both |
| Sharing with phone | 4: UniFFI (used by Mozilla in Firefox — [uniffi-rs](https://github.com/mozilla/uniffi-rs)) or Tauri mobile | 3: React Native or webview | 3 |
| Sharing with web | 4: wasm (size and startup cost; threads limited) | 5: native | 4 |
| Perf (search, sync, crypto, indexing) | 5 | 3 | 5 |
| CRDT fit | 5 with Loro (one Rust core, JS is its wasm) or Automerge; 4 with Yjs (yrs is a separate port, Yjs 14 compat still in progress — [y-crdt](https://github.com/y-crdt), [yrs on lib.rs](https://lib.rs/crates/yrs)) | 5 with Yjs | as Rust |

Key points:
- The editor's CRDT binding runs in JS no matter what. So the page document lives in **two places**: in the webview (bound to ProseMirror) and in the core (storage, sync, search, AI). They exchange binary updates. With **Loro** or **Automerge** both sides run the same Rust code (JS is wasm), which removes a whole class of compat bugs. With **Yjs**, the JS side is Yjs and the Rust side is yrs; compatible for v13, unclear for v14 today. This ties the editor choice to the CRDT choice (see the sync/CRDT research).
- AI agents and rituals run on the hub when the Mac sleeps. If rules live in TS and the hub is Rust, the hub must embed a JS runtime. Avoid that.
- Rust → TS types: generate them (e.g. specta/tauri-specta or ts-rs; versions unverified) so the UI never hand-writes the API shape.

**Recommendation:** Rust core owns storage, sync, CRDT host, crypto, search/index, ontology validation and every domain mutation ("command" API: `create_task`, `close_sprint`, ...). TS owns views, view state, optimistic UI and the editor. The rule: if the hub or another device needs it, it goes in Rust.

**What would change it:** if the spike shows Rust iteration is too slow for the user's pace, move the domain rules (not storage/sync/crypto) to TS and run them on the hub in Bun/Deno — the "hybrid" column. If Electrobun or Electron were chosen, a TS core gets cheaper.

---

## 3. UI framework inside the webview

| | Perf | Editor ecosystem | Keyboard/dnd libs | AI codegen quality | Mobile path | Stability | **Total** |
|---|---|---|---|---|---|---|---|
| **React 19 + Compiler** | 3 | 5 | 5 | 5 | 4 | 5 | **27** |
| Svelte 5 | 5 | 3 | 3 | 4 | 2 | 4 | 21 |
| Solid | 5 | 2 | 2 | 3 | 1 | 3 | 16 |
| Vue 3 | 4 | 4 (Tiptap has Vue) | 3 | 4 | 2 | 5 | 22 |

- **React Compiler 1.0** is stable since 2025-10-07; auto-memoization, Meta reports up to 12% faster initial loads and >2.5x faster interactions in one app; lint rules in `eslint-plugin-react-hooks` — [react.dev](https://react.dev/blog/2025/10/07/react-compiler-1). This narrows the perf gap to signals frameworks.
- Editor kits are React-first: BlockNote (React only), Tiptap UI components and templates (React), Lexical (React plugins), Plate (React). Svelte/Solid get ProseMirror via adapters ([prosemirror-adapter](https://github.com/Saul-Mirone/prosemirror-adapter)) but no Notion-like kit. GitButler shows Svelte works for a big Tauri app, but it has no block editor.
- Keyboard-first: `cmdk` (command palette), `dnd-kit` or `@atlaskit/pragmatic-drag-and-drop`, `tinykeys`-style key maps, Radix/Base UI primitives — all React-first (versions not checked here).

**Recommendation:** React 19 + React Compiler, Vite, strict TS. **What would change it:** choosing Tiptap with a fully custom block UI makes Svelte 5 viable; only worth it if profiling shows React is the bottleneck.

---

## 4. Editor, judged on CRDT fit

### 4.1 Facts

**BlockNote** (on ProseMirror + Tiptap)
- v0.55.0 (2026-09-22): mobile formatting toolbar now supported; v0.54 math and Mermaid blocks; **v0.52 (2026-07-20) decoupled Yjs from core** (`withCollaboration` from `@blocknote/core/yjs`) — [releases](https://github.com/TypeCellOS/BlockNote/releases). Still pre-1.0, breaking changes between minors.
- CRDT: Yjs via y-prosemirror, first-class. BlockNote team is building Yjs 14 attributed history / track changes with y-prosemirror — [FOSDEM 2026](https://fosdem.org/2026/schedule/event/8VKQXR-blocknote-yjs-prosemirror/). No official Loro or Automerge support; since it is ProseMirror underneath, loro-prosemirror could in theory be plugged in, but comments/versioning features assume Yjs (unverified; spike item).
- Block IDs: every block has a stable `id` (built in). Mentions: custom inline content types. Notion UX out of the box: slash menu, side menu, drag handles, nesting, tables.
- Licence: core MPL-2.0 (fine for closed source; changes to BlockNote files stay open). `@blocknote/xl-*` (PDF/DOCX/ODT export, multi-column, AI) are **GPL-3.0 or commercial** — [pricing](https://www.blocknotejs.org/pricing), [XL licence](https://www.blocknotejs.org/legal/blocknote-xl-commercial-license). Avoid xl-* unless we buy a licence or open-source the app.
- Size: heaviest of the group, ~180–390 KB gzip depending on UI kit — [eddyter, 2026](https://eddyter.com/blogs/rich-text-editor-bundle-size-comparison-2026), [EditorStack](https://www.editorstack.cc/libraries/blocknote) (methods vary). In a desktop app this matters less than on the web.
- Sprint already uses it, so we know its limits.

**Tiptap 3** (ProseMirror)
- 3.0 stable July 2025; core MIT; 10 former Pro extensions open-sourced (UniqueID, DragHandle, Details, Mathematics, FileHandler, TableOfContents, Emoji...) — [Tiptap 3 stable](https://tiptap.dev/blog/release-notes/tiptap-3-0-is-stable), [open-sourcing](https://tiptap.dev/blog/release-notes/were-open-sourcing-more-of-tiptap).
- The **Notion-like template is paid** (Start plan, ~$49/month annual); only the "Simple" template is free MIT — [template docs](https://tiptap.dev/docs/ui-components/templates/notion-like-editor), [pricing](https://tiptap.dev/pricing). So with MIT only, we build the block UI (slash menu, side menu, drag) ourselves.
- CRDT: Yjs (Collaboration extension, Hocuspocus server, MIT); y-prosemirror v2 demo uses Tiptap 3 — [y-prosemirror](https://github.com/yjs/y-prosemirror). Loro via loro-prosemirror plugins (ProseKit examples use it — [prosekit/examples](https://github.com/prosekit/examples/pull/1864)).
- Block IDs via UniqueID (MIT now). Mentions: official Mention extension. ~34 KB gzip core + ~105 KB starter kit (source above).

**Raw ProseMirror**: same CRDT bindings as Tiptap, total control, most work. Tiptap is a thin enough layer that "raw" buys little.

**CRDT bindings for ProseMirror (the core of this section)**
- **y-prosemirror** v1 (Yjs 13): stable, used by Tiptap, BlockNote and many products; collaborative undo, cursors. **@y/prosemirror v2** for Yjs 14 adds suggestion mode, attribution, version diffs; v2.0.0-14 published 2026-10-01; Yjs 14 at rc.26 (2026-09-07). "Most users should continue to use y-prosemirror with Yjs v13 for now" — [repo](https://github.com/yjs/y-prosemirror), [releases](https://github.com/yjs/y-prosemirror/releases). Attribution is directly useful for "AI edited this paragraph" provenance.
- **loro-prosemirror** v0.4.4 (2026-08-22): sync, undo/redo, cursors via EphemeralStore, multiple editors per Loro doc; ~144 stars, MIT — [repo](https://github.com/loro-dev/loro-prosemirror), [releases](https://github.com/loro-dev/loro-prosemirror/releases). Recent fixes are about mark ranges, out-of-bounds positions and destroyed views: young but active. Loro itself 1.16.4 (2026-09-30) — [loro releases](https://github.com/loro-dev/loro/releases). Rate: beta-to-early-stable.
- **@automerge/prosemirror** v0.2.0 (2026-02-25): self-described "beta quality... there are bugs" — [repo](https://github.com/automerge/automerge-prosemirror). Not ready for the main editor.

**Lexical** (Meta)
- @lexical/yjs 0.49 — [npm](https://www.npmjs.com/package/@lexical/yjs); Yjs binding is mature and used at Meta scale. Loro binding exists only as third-party (datalayer/lexical-loro) — [repo](https://github.com/datalayer/lexical-loro).
- Node keys are runtime-only, so stable block IDs are ours to add. No Notion-like kit out of the box. Small core (~50–90 KB gzip, sources disagree). Pre-1.0 after years.

**Plate** (Slate): rich Notion-like components (shadcn-based), Yjs via @platejs/yjs / slate-yjs, v54 beta in June 2026 — [releases](https://platejs.org/docs/releases). Slate's history of IME and Android input bugs is the main worry (well known, not re-verified here). No Loro binding.

**BlockSuite / AFFiNE**: MIT, web components, its own block model on Yjs; strongest "Notion + whiteboard" feature set — [repo](https://github.com/toeverything/blocksuite). But it is developed inside AFFiNE and published as an extract (recent forks re-import it from AFFiNE v0.27.4 — [PR](https://github.com/Cloaked-Workspace/blocksuite/pull/4)); its store wants to own the document model. High coupling, poor fit for our own ontology.

**Milkdown**: Markdown-first ProseMirror kit, Yjs plugin. Good for Markdown files on disk, weaker block UX.

**Native (TextKit 2)**: see 1.1. No CRDT binding; Loro has Swift bindings (loro-swift, maturity unverified) but the editor would be all ours.

### 4.2 Scoring (1–5)

| | CRDT fit (Yjs) | CRDT fit (Loro) | Block IDs | Mentions | Notion UX built in | Licence | IME/mobile | Size | Maturity | **Total (Yjs path)** | **Total (Loro path)** |
|---|---|---|---|---|---|---|---|---|---|---|---|
| **BlockNote** | 5 | 2 (unproven) | 5 | 4 | 5 | 4 (MPL; avoid xl-*) | 4 | 2 | 3 | **32** | 29 |
| **Tiptap 3 (MIT)** | 5 | 4 | 4 | 5 | 2 | 5 | 4 | 4 | 5 | 34* | **33** |
| Lexical | 4 | 2 | 2 | 4 | 2 | 5 | 4 | 5 | 4 | 30 | 28 |
| Plate | 4 | 1 | 4 | 4 | 4 | 5 | 2 | 3 | 3 | 29 | 26 |
| BlockSuite | 3 (own model) | 1 | 5 | 4 | 5 | 5 | 3 | 1 | 3 | 30 | 28 |
| Milkdown | 4 | 3 | 2 | 3 | 2 | 5 | 4 | 4 | 3 | 30 | 29 |
| Native TextKit 2 | 1 | 2 | 1 | 2 | 1 | 5 | 5 | 5 | 2 | 22 | 23 |

\* Tiptap scores high on paper but "Notion UX built in = 2" hides ~3–6 weeks of building slash/side menus, drag handles and block nesting ourselves. BlockNote gives those today.

**IME/mobile note:** ProseMirror-based editors handle `beforeinput`/composition better than Slate in practice; BlockNote 0.55 explicitly targets iOS/Android keyboards. All of them inherit WKWebView behaviour on Mac/iOS, which is why IME is a spike item, not a desk conclusion.

### 4.3 Recommendation

- Lock the **engine: ProseMirror**. It is the only engine with mature bindings to all three CRDTs (Yjs stable, Loro active, Automerge beta) and it keeps both kits open.
- **If the CRDT is Yjs:** BlockNote (MPL core, no xl-*) for pages and journal; plain Tiptap for small fields (task description, comments). Plan the move to Yjs 14 / @y/prosemirror when it goes stable, for attribution of AI edits.
- **If the CRDT is Loro:** Tiptap 3 MIT + loro-prosemirror, with our own block UI (slash menu, side menu, drag handle using MIT DragHandle + UniqueID). First try BlockNote + loro-prosemirror in the spike; if it works without forking, keep BlockNote's UI.
- Store page content in the CRDT as the source of truth; derive plain text, links, mentions and block IDs into SQLite in the Rust core for search and the context graph.

**What would change it:** loro-prosemirror stalls or fails the spike → Yjs path. Yjs 14 slips well into 2027 and v13 attribution is needed → custom provenance outside the editor. BlockNote relicenses more of core or stays unstable through 1.0 → Tiptap path. Our UI needs canvas/whiteboard soon → look again at BlockSuite.

---

## 5. Risks and unknowns

1. **WKWebView**: IME + shortcut conflicts, background suspend and dead renderer, engine version tied to macOS. Mitigation: macOS floor, renderer-crash reload, composition-aware key handling, app menu for all shortcuts.
2. **Two copies of the document** (webview + core). Bugs in update exchange lose edits. Mitigation: core is the durable copy; webview sends updates over a binary Channel; fuzz tests that replay random edits on both sides and compare.
3. **Yjs 14 / yrs**: if the CRDT is Yjs, the Rust side (yrs 0.26) may lag Yjs 14 format changes.
4. **loro-prosemirror maturity**: small team, v0.x; we may need to fix bugs upstream.
5. **BlockNote pre-1.0 churn**: breaking changes between minor versions (e.g. v0.52 collaboration API).
6. **Tauri mobile maturity**: phone may still end up React Native or native Swift with the Rust core via UniFFI; keep the core free of Tauri types.
7. **Rust iteration speed** for one person; measure in the spike, do not assume.
8. Most benchmark numbers above come from blogs with weak methods; we must measure our own.

## 6. Spike (about 2 weeks) — what it must prove

Build: Tauri 2.12 app, React 19 + Compiler, Rust core with SQLite, one page editor (BlockNote, then Tiptap) bound to the chosen CRDT, with the doc mirrored in the Rust core over a binary channel. Same UI also served in a browser against the core compiled to wasm (or a hub over HTTP).

Pass criteria (on an Apple Silicon Mac, oldest supported macOS):
- Cold start to interactive editor < 600 ms; idle RAM (app + WebContent process, from Activity Monitor) < 150 MB with one 1,000-block page open.
- Typing latency p95 < 16 ms in a 10,000-block page; paste of 500 blocks < 300 ms.
- IME: Japanese, Chinese pinyin, Ukrainian and Polish layouts, macOS dead keys and emoji picker: no double Enter, no dropped or duplicated chars, undo groups sane.
- All app shortcuts (Cmd+K palette, Cmd+Z/Shift+Z, Cmd+F, block moves) work and don't fire during composition.
- Drag and drop: files from Finder into a page; block drag within and between pages.
- Leave the app in background 20+ minutes under memory pressure: it recovers without data loss.
- Two devices (Mac + browser) edit the same page offline, reconnect, converge; undo is per-user; block IDs and mentions survive merges.
- Kill the app mid-typing: no lost edits beyond the last ~1 s.
- Record dev-time: how many hours each kit took for slash menu, mentions, block IDs, drag.
