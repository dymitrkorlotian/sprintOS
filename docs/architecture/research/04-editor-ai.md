# Research 04: the editor and AI for a local-first Mac (later phone) app

Research date: 2026-10-02. All "accessed" dates are 2026-10-02 unless noted.
Method: web search plus direct reads of GitHub repos. Several primary sites were **blocked by the session's egress proxy**
(blocknotejs.org, tiptap.dev, blog.voyageai.com, docs.voyageai.com, huggingface.co, arxiv.org, ai.google.dev, apple.com,
simonwillison.net, x.com). For those, facts come from search-engine snippets of the page (URL given) or from the project's
GitHub source. Items marked **[unverified]** are from snippets of secondary sites or from my own background knowledge and
should be checked before the ADR cites them as fact.

---

## PART A: the editor

### A1. BlockNote

| Topic | Finding | Source |
|---|---|---|
| Current version | **0.55.0, 22 Sep 2026** (mobile formatting toolbar promoted from experimental to supported). Before that: 0.54.1 (9 Sep 2026, Typst-based accessible PDF/UA-1 export), 0.54.0 (13 Aug 2026, KaTeX math blocks, Mermaid diagram blocks), 0.53.0 (6 Aug 2026, **breaking**: UI moved from Radix to Base UI, which affects `@blocknote/shadcn`), 0.52.0 (20 Jul 2026, **breaking**: Yjs decoupled from `@blocknote/core`; collaboration now via `withCollaboration` from `@blocknote/core/yjs`). | https://github.com/TypeCellOS/BlockNote/releases |
| Cadence | About one minor every 2–4 weeks; 0.47.0 was 23 Feb 2026 and 0.55.0 was 22 Sep 2026, so 8 minors in 7 months. Breaking changes land in minors (0.x semver). | same; https://github.com/TypeCellOS/BlockNote/blob/main/CHANGELOG.md |
| Licence | Core packages **MPL-2.0** (file-level copyleft: you must publish changes you make to BlockNote's own files; closed-source apps are allowed). `@blocknote/xl-*` packages (AI, multi-column, PDF/DOCX/ODT/email/Typst exporters) are **GPL-3.0 OR a commercial licence**. | https://github.com/TypeCellOS/BlockNote (README); https://github.com/suitenumerique/docs/issues/2748 (XL moved from AGPL to GPL-3.0) |
| Commercial licence price | Business tier: **$390/month billed monthly, or $2,340/year** (= $195/month). It includes the commercial XL licence (AI, multi-column, exports), logo on site, SLA. Enterprise: custom. Free tier: all blocks/UI, drag and drop, slash menu, real-time collaboration, comments. | Read from the pricing page source in the repo: `docs/app/pricing/page.tsx` (repo HEAD 2 Oct 2026, https://github.com/TypeCellOS/BlockNote/tree/main/docs/app/pricing). Snippet of https://www.blocknotejs.org/pricing gives "$195/month, $2,340 billed yearly". |
| When the licence matters | Only if an `xl-*` package ships in a closed-source product. A personal, unsold app is not "distributing", and an open-source GPL-3.0 app is fine. Selling a closed-source Mac app that uses `xl-ai` or `xl-multi-column` → needs Business. | pricing page snippet ("a commercial license is needed only when you use any of the XL packages and you cannot comply with the GPL-3.0") |
| Collaboration | **Yjs only** in practice. Providers documented: Liveblocks, PartyKit, Y-Sweet, Hocuspocus, plain y-websocket/WebRTC. Since 0.52 Yjs is an optional entrypoint, which in principle opens the door to other CRDTs, but **no Loro or Automerge adapter exists**; the repo docs contain no mention of either (grep of `docs/` on 2 Oct 2026). Comments/threads also depend on Yjs. | https://www.blocknotejs.org/docs/features/collaboration (snippet); release notes v0.52.0; own grep of repo |
| Yjs 14 work | BlockNote team and Kevin Jahns (Yjs) are building attributed version history and track changes (suggestions) on Yjs 14 / `@y/prosemirror`, funded by ZenDiS (openDesk) and DINUM (La Suite Docs). FOSDEM 2026 talk. | https://fosdem.org/2026/schedule/event/8VKQXR-blocknote-yjs-prosemirror/ |
| 1.0 status | **No 1.0 date found.** Still 0.x after ~4 years; no public 1.0 roadmap located. **[unverified: absence of evidence]** | releases page; search found nothing |
| Long-document performance | No authoritative benchmark found. BlockNote renders every block as React-managed node views on top of ProseMirror, so very long pages (thousands of blocks) are heavier than plain ProseMirror; 0.54.1 notes "performance improvements to block change resolution". Treat as **[unverified]**; worth a spike with a 2–5k-block page in the target WebView. | release notes v0.54.1; https://velt.dev/blog/blocknote-collaborative-editor-guide (generic remarks only) |
| Team / funding | Built by TypeCell (Yousef El-Dardiry, Matthew Lipski and team, Netherlands). Sponsors on README: **NLnet Foundation** (TypeCell is "championed by NLnet"), Vercel, BrowserStack. Feature work repeatedly sponsored by **DINUM (France) and ZenDiS (Germany)** for La Suite Docs / openDesk, which are built on BlockNote. Revenue also from Business subscriptions. 10.3k GitHub stars. | https://github.com/TypeCellOS/BlockNote ; https://github.com/TypeCellOS/BlockNote/releases/tag/v0.20.0 (DINUM/ZenDiS sponsorship notes); https://docs.la-suite.eu/ |

Takeaway: BlockNote is actively developed, funded by two governments plus paying customers, and is the only option that gives a
Notion-like block UX out of the box. Its risks are 0.x churn (two breaking minors in summer 2026) and a Yjs-only CRDT story.

### A2. Alternatives

| Editor | State in 2026 | Licence | CRDT | Notes / sources |
|---|---|---|---|---|
| **Tiptap 3** | 3.0 stable 12 Jul 2025; ~9M npm downloads/month. 10 formerly Pro extensions (DragHandle, UniqueID, Mathematics, TableOfContents, Details, Emoji, FileHandler…) open-sourced under MIT. | Core MIT. Paid: Tiptap Cloud (documents), comments, AI, conversion etc. Pricing (snippet): Start $49/mo (500 docs), Team $149/mo, Growth $999/mo; free plan removed June 2025 → 30-day trial. **[pricing unverified: tiptap.dev blocked]** | Yjs via Collaboration extension + **Hocuspocus** (MIT; v4 runs on Bun, Deno, Cloudflare Workers **[unverified]**). Being ProseMirror, loro-prosemirror and automerge-prosemirror can also be wired in. | https://tiptap.dev/blog/release-notes/tiptap-3-0-is-stable ; https://news.ycombinator.com/item?id=44202103 ; https://eddyter.com/blogs/tiptap-pricing-explained-2026 |
| **Lexical** (Meta) | v0.51.0 (late Sep 2026); maintainers say recent releases are "a big step" toward 1.0 (move to Extension / `$config` APIs, remove deprecated APIs). Powers Facebook/Messenger composers. Has **lexical-ios** (native Swift port, separate project). | MIT | `@lexical/yjs` only **[binding name from memory]**. No Loro/Automerge binding found. | https://www.npmjs.com/package/lexical ; https://github.com/facebook/lexical/releases ; https://swiftpackageindex.com/facebook/lexical-ios |
| **ProseMirror** directly | Mature, stable, Marijn Haverbeke; the base of BlockNote, Tiptap, Milkdown, Novel. | MIT | Best binding coverage: y-prosemirror, loro-prosemirror, @automerge/prosemirror all target it. | (background knowledge) |
| **Plate** (Slate) | v48 → v49 (Jun–Sep 2025) → v50 (Sep–Oct 2025); shadcn-based UI kit, AI plugins. | MIT **[from memory]** | slate-yjs; v48 added multi-provider (Hocuspocus + WebRTC on one Y.Doc). Slate has no Loro/Automerge binding. | https://platejs.org/docs/releases ; https://platejs.org/docs/migration/v48 |
| **Milkdown** | 7.x, v7.22.1 (12 Aug 2026), actively maintained; Markdown-first, ProseMirror-based. | MIT **[from memory]** | y-prosemirror plugin (`@milkdown/plugin-collab`). | search snippet (hysenlabs/github Milkdown), https://github.com/Milkdown |
| **Novel** (Steven Tey) | Tiptap wrapper with AI autocomplete; little release activity since 2024 **[unverified]**. | Apache-2.0 **[from memory]** | Whatever Tiptap gives you; none built in. | https://github.com/steven-tey/novel/releases |
| **Yoopta** | v6.0.0, 23 Feb 2026: headless, theme presets, new plugin architecture, "real-time collaboration". Slate-based. | MIT **[from memory]** | Collaboration mechanism not verified (likely Yjs via slate-yjs) **[unverified]**. | https://github.com/yoopta-editor/Yoopta-Editor/releases ; https://yoopta.dev/ |
| **Editor.js** | Block-JSON editor (not ProseMirror). Snippet claims last release Aug 2023 **[unverified, likely stale]**. | Apache-2.0 **[from memory]** | No real CRDT story. | search snippet |
| **Native Swift** | No production-grade Notion-like block editor exists. Building blocks: SwiftUI `TextEditor` + `AttributedString` rich text (iOS/macOS 26+, WWDC25 session 280); TextKit 2. Hobby/POC projects: Grimoire (TextKit 2 block model), ProseKit (mymind POC, Tiptap-like model in Swift), swift-markdown-engine. Lexical-iOS exists but is a separate, iOS-first port. CRDT side: automerge-swift (0.5.x, maintained), loro-swift (MIT, "experimental", 1.16.x), YSwift/y-uniffi ("Work In Progress"). None ships a TextKit binding. | n/a | Would need a custom CRDT↔TextKit binding. | https://developer.apple.com/videos/play/wwdc2025/280/ ; https://github.com/mymindcorp/ProseKit ; https://github.com/loro-dev/loro-swift ; https://github.com/automerge/automerge-swift ; https://github.com/y-crdt/y-uniffi |

Notion-likes built on them (context, background knowledge **[unverified details]**): La Suite Docs and openDesk (BlockNote),
AFFiNE (BlockSuite, its own editor on Yjs), AppFlowy (Flutter, own editor, Yjs-like CRDT via yrs), Outline (ProseMirror +
Yjs), Docmost (Tiptap + Hocuspocus), Anytype (own any-sync CRDT, native).

### A3. CRDT bindings for ProseMirror

| Binding | Version / date | Maturity | Features | Source |
|---|---|---|---|---|
| **y-prosemirror** (Yjs 13) | stable **1.3.7**; `@y/prosemirror` 2.0.0-14 (1 Oct 2026) for Yjs 14 is the unstable dev line | Production: used by BlockNote, Tiptap, Outline, Milkdown, La Suite Docs | Sync, per-client undo/redo, cursors via Awareness, schema recovery on invalid concurrent edits; Yjs 14 adds attributed version history and suggestions | https://github.com/yjs/y-prosemirror ; https://github.com/yjs/y-prosemirror/releases/tag/v2.0.0-14 |
| **loro-prosemirror** | **0.4.4, 22 Aug 2026** (0.4.0 Nov 2025 added CursorEphemeralStore) | Pre-1.0, actively maintained, smaller user base (ProseKit, Atomic Server patch it) | Sync, undo/redo as PM commands, cursors | https://github.com/loro-dev/loro-prosemirror/releases ; https://github.com/ontola/atomic-server/issues/1494 |
| **@automerge/prosemirror** | 0.2.0 (Feb 2026) per snippet **[version history unclear]** | README: "beta quality software… there are bugs"; requires Automerge's rich-text schema subset; PM doc must be initialised from the Automerge doc | Sync of blocks and marks | https://github.com/automerge/automerge-prosemirror ; Automerge 3 (10x less memory): https://automerge.org/blog/automerge-3/ |

**Best CRDT story: Yjs via y-prosemirror**, clearly. It is the only binding in production at scale, it is what BlockNote uses
natively, and it has a funded roadmap (version history, track changes). Loro is the most interesting newcomer (fast Rust core,
good history/time-travel, Swift bindings) but its ProseMirror binding is 0.4 and nobody has wired it into BlockNote. Automerge's
binding is beta.

Native-side note for a Mac/phone app: Yjs documents can be read and merged natively through **yrs** (Rust) or YSwift, so a
Swift or Rust sync layer can hold page bodies as Y.Docs while the editor itself runs in a WebView. Loro and Automerge have the
same property (Rust cores with Swift bindings).

### A4. Editor scorecard (1 = poor, 5 = best)

| Editor | CRDT fit | Licence | Maturity | Notion UX out of the box | Desktop/phone reuse (WebView) | Total /25 |
|---|---|---|---|---|---|---|
| **BlockNote** | 4 (Yjs first-class, Y14 history coming; no Loro/Automerge) | 4 (MPL core; XL = GPL or $2,340/yr) | 3 (0.x, breaking minors) | **5** | 5 | **21** |
| Tiptap 3 | 4 (Yjs + Hocuspocus; any PM binding) | 4 (MIT core; paid cloud/pro) | 5 | 2 (headless; templates help) | 5 | 20 |
| ProseMirror direct | **5** (all three bindings) | 5 | 5 | 1 | 5 | 21 (but months of UI work) |
| Lexical | 3 (Yjs only) | 5 | 3 (0.51, 1.0 near) | 2 | 5 (plus lexical-ios) | 18 |
| Plate | 3 (slate-yjs) | 5 | 3 | 4 | 5 | 20 |
| Milkdown | 3 | 5 | 3 | 2 | 5 | 18 |
| Yoopta 6 | 2–3 **[unverified]** | 5 | 2 | 4 | 5 | 18 |
| Novel | 2 | 5 | 2 | 3 | 5 | 17 |
| Editor.js | 1 | 5 | 3 | 3 | 5 | 17 |
| Native Swift (TextKit 2 / SwiftUI) | 2 (no binding) | 5 | 1 | 1 | 2 (no web reuse) | 11 |

### A5. Recommendation: keep BlockNote

- It already holds the data (`objects.body` as BlockNote JSON), it is the best Notion-like UX available, and its CRDT is the most
  mature one (Yjs). The local-first app can run the same React editor in a WebView (Tauri/Electron, or WKWebView inside a Swift
  shell) on Mac and later phone, and the web app keeps working unchanged.
- Choose **Yjs** as the CRDT for page bodies if BlockNote stays. A Loro- or Automerge-based sync engine would force a custom
  BlockNote adapter (possible after 0.52's decoupling, but nobody has built one, and comments/versioning features assume Yjs).
- Avoid `xl-*` packages, or budget the Business licence if the app is ever sold closed-source. `xl-ai` is not needed: AI calls
  go through our own `complete()`.
- Pin exact versions and upgrade deliberately (0.52 and 0.53 were breaking).
- Spike before committing: a 2–5k-block page in Safari/WKWebView, and an offline-merge test of two Y.Docs edited separately.

What would change this:
- BlockNote loses funding or stalls (DINUM/ZenDiS stop, no releases for 6+ months) → move to **Tiptap 3** (same ProseMirror/Yjs
  base; BlockNote is itself built on Tiptap, so the document model maps).
- The sync-engine ADR picks Loro or Automerge for everything → either write and own a BlockNote↔loro-prosemirror adapter, or
  store page bodies as Yjs updates inside the other engine as opaque blobs.
- A hard requirement for a fully native (no WebView) editor → no ready option; expect to build one (ProseKit/Grimoire-style)
  and lose the web editor parity. Not recommended for a solo project.

---

## PART B: AI locally vs API

### B1. Local embedding models (multilingual, short texts, Apple Silicon)

Two benchmark views are used because no single table covers every model:
- **MMTEB Retr.** = the Retrieval column of MTEB (Multilingual) as published in the Qwen3-Embedding README (scores pulled from
  the MTEB leaderboard on 6 Jun 2025). https://raw.githubusercontent.com/QwenLM/Qwen3-Embedding/main/README.md
- **Granite-18** = "MTEB Multilingual Retrieval (18 tasks)" from IBM's Granite R2 release table (29 Apr 2026), which also gives
  encode throughput on one H100 (spans/s), useful only as a relative speed indicator.
  https://github.com/ibm-granite/granite-embedding-models/

| Model | Params | Dims (Matryoshka?) | MMTEB mean (task) | MMTEB Retr. | Granite-18 retr. | Throughput (H100, rel.) | Licence | Size on disk (approx.) | Apple Silicon notes |
|---|---|---|---|---|---|---|---|---|---|
| **Qwen3-Embedding-0.6B** | 0.6B | 1024 (MRL, any dim → 512 OK), 32K ctx, instruction-aware | **64.33** | **64.64** | – | – (decoder, slower than BERT-size) | **Apache-2.0** | GGUF Q8 639 MB, Q4_K_M 378 MB | GGUF/llama.cpp, MLX community ports **[speed unverified]** |
| **EmbeddingGemma-300M** | 308M | 768 (MRL 512/256/128), 2K ctx | 61.15 | – | 62.5 | 1,277 | Gemma Terms of Use **[from memory; not Apache]** | ~0.6 GB bf16, ~0.3 GB int8 | Core ML ANE build: **~5.8 ms/embedding on M4**, 8-bit, 98.8% of ops on ANE (https://huggingface.co/erjigit17/embeddinggemma-300m-ane-coreml, snippet) |
| **harrier-oss-v1-270m** (Microsoft) | 270M | 640, 32K ctx, 94+ languages | – | – | **66.4** | 2,055 | **MIT** | ~0.55 GB fp16 (computed) | ONNX build exists (onnx-community) |
| **granite-embedding-311m-multilingual-r2** (IBM) | 311M | 768 (MRL), 32K ctx, 52 languages enhanced | – | – | 64.0 | **3,075** | **Apache-2.0** | ~0.6 GB fp16 (computed) | ModernBERT; fast |
| granite-embedding-97m-multilingual-r2 | 97M | 384 | – | – | 59.6 | 3,379 | Apache-2.0 | ~0.2 GB | best sub-100M |
| jina-embeddings-v5-text-nano | 239M | 768 (MRL to 32), 8K ctx | – | – | 63.3 | 1,081 | **CC BY-NC 4.0** (commercial needs licence) | ~0.5 GB | official MLX and GGUF builds |
| multilingual-e5-small | 118M (table says 96M) | 384 (no MRL) | – | – | 50.9 | 2,290 | MIT **[from memory]** | ~0.24 GB | everywhere (ONNX, transformers.js) |
| multilingual-e5-base | 278M | 768 | – | – | 52.7 | 1,800 | MIT **[from memory]** | ~0.56 GB | |
| multilingual-e5-large-instruct (ref.) | 560M | 1024 | 63.22 | 57.12 | – | – | MIT | ~1.1 GB | |
| bge-m3 | 568M | 1024 dense + sparse + ColBERT | 59.56 | 54.60 | – | – | MIT **[from memory]** | ~1.1 GB fp16 | Ollama/GGUF |
| nomic-embed-text-v2-moe | 475M (305M active) | 768 (MRL to 256), ~100 languages | – | – | – | – | **Apache-2.0** | ~0.95 GB fp16 | GGUF/Ollama |
| snowflake-arctic-embed-m-v2.0 | 305M (113M non-embedding) | 768 (MRL to 256), 8K ctx, 74 languages | – | – | 54.8 | 2,754 | **Apache-2.0** | ~0.6 GB | |
| jina-embeddings-v3 | 570M | 1024 (MRL) | – | – | – | – | **CC BY-NC 4.0** | ~1.1 GB | |
| jina-embeddings-v4 | 3.8B | 2048 (MRL) multimodal | – | – | – | – | Qwen Research Licence (non-commercial) | ~7.5 GB | too big for this use |
| **voyage-4-nano** (open weights) | ~340M (180M non-emb. + 160M emb.) | 2048/1024/**512**/256 (MRL), 32K ctx | – | – | – | – | **Apache-2.0** | ~0.7 GB fp16; community MLX 8-bit and ONNX ports | **shares the embedding space of voyage-4-lite/-4/-4-large** |

Sources for rows: Qwen README (above); EmbeddingGemma 61.15 from https://arxiv.org/pdf/2509.20354 (snippet) and
https://ai.google.dev/gemma/docs/embeddinggemma (snippet); harrier licence/dims from
https://huggingface.co/microsoft/harrier-oss-v1-270m (snippet) and https://the-decoder.com/microsofts-bing-team-open-sources-harrier-embedding-model/ ;
Granite from https://huggingface.co/blog/ibm-granite/granite-embedding-multilingual-r2 (snippet) and the GitHub table; jina v5 from
https://jina.ai/models/jina-embeddings-v5-text-nano/ (snippet, released 18 Feb 2026); jina v3/v4 from https://jina.ai/models/jina-embeddings-v4/
(snippet); nomic from https://simonwillison.net/2025/Feb/12/nomic-embed-text-v2/ (snippet); arctic from
https://huggingface.co/Snowflake/snowflake-arctic-embed-m-v2.0 (snippet); voyage-4-nano from https://huggingface.co/voyageai/voyage-4-nano
and https://huggingface.co/blog/mongodb-community/hugging-face-mongodb-voyage-4-nano (snippets).

Caveats:
- No source gave **Polish-specific** retrieval numbers for these models. MMTEB averages cover many languages; Polish is in the
  training mix of all the "100+ language" models but measured quality on short Polish notes is **unverified**. A 50-query
  bilingual eval on the user's own data would settle it.
- Apple Silicon speed numbers are scarce. Only the EmbeddingGemma Core ML figure (5.8 ms per embedding on M4) was found. A
  BERT-size (~300M) encoder typically embeds a short note in single-digit to tens of ms on M1–M4 via Core ML/ONNX/MLX, so
  embedding a few thousand personal notes is a matter of seconds to a minute **[estimate]**. Qwen3-0.6B, a decoder, is a few
  times slower per item **[estimate]**.
- transformers.js/WebGPU runs these in a WebView/browser too; speeds **[unverified]**.

**Voyage 4 family (API)** (blog post 15 Jan 2026; blog itself blocked, details from snippets):
- voyage-4-large $0.12, voyage-4 $0.06, **voyage-4-lite $0.02 per 1M tokens**; first **200M tokens free** per account on the
  voyage-4 generation; 33% batch discount. https://embeddingcost.com/voyage ; https://markaicode.com/pricing/voyage-ai-pricing/ ;
  https://openrouter.ai/voyageai/voyage-4-lite
- voyage-4-lite: 32K context, dims 2048/1024/**512**/256 (MRL), float/int8/uint8/binary output.
  https://cloudprice.net/models/voyage-4-lite
- Shared embedding space across voyage-4-large, -4, -4-lite and open-weight -4-nano, enabling "asymmetric retrieval" (index with
  one, query with another). https://blog.voyageai.com/2026/01/15/voyage-4/ (snippet)
- RTEB (Voyage's own retrieval benchmark): voyage-4-lite 68.10; voyage-4-nano "TBD". Voyage-4-large claimed +14% over OpenAI
  text-embedding-3-large. Not independently verified on the public MTEB leaderboard. https://docs.vaicli.com/docs/models/voyage-4-family ;
  https://aimultiple.com/embedding-models (snippets)
- Third-party multilingual test: voyage-4-lite nDCG@10 0.793 vs text-embedding-3-large 0.704. https://aimultiple.com/embedding-models (snippet)
- Cost reality for this app: a personal OS with ~10k objects × ~300 tokens = 3M tokens per full re-index = $0.06 with lite,
  and inside the free 200M anyway.

**Embedding recommendation**
1. **First try voyage-4-nano locally.** Apache-2.0, 512-dim MRL output, and it shares the embedding space with the
   voyage-4-lite vectors already stored as `halfvec(512)`. If the shared-space claim holds at 512 dims, the Mac app can embed
   queries and new notes offline without re-indexing, and the web app can keep calling voyage-4-lite. Must verify: (a) nano↔lite
   cross-model recall on our data, (b) nano quality on Polish, (c) whether MRL truncation to 512 keeps the shared space.
2. Fallback, fully local with a re-index: **Qwen3-Embedding-0.6B** (best MMTEB retrieval among small open models, Apache) or
   **harrier-oss-v1-270m** (MIT, top Granite-18 score, smaller) or **granite-311m-r2** (Apache, fastest). Keep 512 dims via MRL
   where supported (harrier is 640 fixed → re-check).
3. Avoid jina v3/v4/v5 (non-commercial) and EmbeddingGemma if the Gemma terms are a problem for a future sale.

### B2. Apple on-device options

**Foundation Models framework** (macOS/iOS 26+, updated at WWDC26 for OS 27):
- On-device "System Language Model", ~3B parameters (2025 tech report https://machinelearning.apple.com/research/apple-foundation-models-2025-updates ;
  https://arxiv.org/html/2507.13575v3). Guided generation (`@Generable`), tool calling, streaming.
- Context: **4,096 tokens** in OS 26 (https://www.infoq.com/news/2026/03/apple-foundation-models-context/); **8,192** in OS 27,
  with token-counting and context-size APIs; model "rebuilt from the ground up" and accepts images
  (https://developer.apple.com/videos/play/wwdc2026/241/ , fetched).
- **Private Cloud Compute** for third-party apps in OS 27: 32K context, reasoning levels, no API key; free for developers with
  under 2 million first-time downloads; users get daily quota, more with iCloud+ (same WWDC26 session).
- New `LanguageModel` protocol lets other models plug in: `MLXLanguageModel`, `CoreAILanguageModel` (local), and Anthropic/Google
  Swift packages (https://developer.apple.com/videos/play/wwdc2026/339/). Framework going open source.
- **Languages: Polish is not supported** by Apple Intelligence, so the on-device and PCC models should not be relied on for Polish.
  16 languages as of iOS 27 (https://www.macobserver.com/news/apple-intelligence-english-only-features-16-languages-2/ snippet);
  Polish press (Sep 2026) says Apple Intelligence still has no Polish, though a text-only Polish Siri appeared in iOS 27
  (https://www.benchmark.pl/apple-prezentuje-ios-27-nowa-siri-to-siri-ai-i-nadal-nie-mowi-po-polsku-7294720442882528a ;
  https://thinkapple.pl/2026/09/21/siri-po-polsku-w-ios-27-na-iphone-dostepna-siri-pl/ ; snippets). Whether the FM framework
  refuses Polish input or merely is untuned for it: **[unverified]**.
- Fit: fine for English-only micro-tasks (titles, tags, classification) offline; not a replacement for Claude for summaries,
  retro or triage across English and Polish.

**NaturalLanguage embeddings**:
- `NLEmbedding.sentenceEmbedding` exists only for ~6 Western European languages (en, es, de, fr, it, pt), so **no Polish**.
- `NLContextualEmbedding` (macOS 14+/iOS 17+) has script-family models; the Latin-script model covers ~20 languages
  **including Polish** (snippet citing https://github.com/turantekin/Parrot/pull/63). It returns **token-level** vectors that you
  must pool yourself and is not trained for retrieval; no MTEB numbers published. Expect clearly worse retrieval than any model
  in B1 **[judgement, unverified]**. Advantage: zero download, ships with the OS.

### B3. Local LLMs for summaries

Measured/reported decode speeds:
- M4 Max, MLX-Swift: Qwen 3.5 2B **292 tok/s**, Gemma 4 E2B 185 tok/s; llama.cpp about half that; Core ML/ANE much slower but
  3–8× less memory (~230 MB vs 1.2 GB at 2B). iPhone 17 Pro, Gemma 4 E2B: 39–61 tok/s. (https://github.com/john-rocky/apple-silicon-llm-bench , Sep 2026, fetched)
- Qwen3 8B 4-bit on M4 Max via MLX ~38–62 tok/s; Qwen 3.5 9B 4-bit on M4 Pro 24 GB ~32–42 tok/s
  (https://markaicode.com/benchmarks/hugging-face-qwen-3-m4-max-throughput-benchmark/ ; https://willitrunai.com/blog/qwen-3-5-mlx-apple-silicon-guide , snippets).
- llama.cpp, Llama-2 7B Q4_0: base M3 ~26 tok/s, M2 Max ~60, M3 Ultra ~116 (https://github.com/ggml-org/llama.cpp/discussions/4167 , snippet).
- RAM: a 7–9B model at 4-bit needs ~5–6 GB plus KV cache, so it is comfortable on 16 GB and painful on 8 GB Macs; 2–4B models
  fit in 2–3 GB **[estimate]**.
- New open models: Gemma 4 (2 Apr 2026, **Apache-2.0**, E2B/E4B/26B-MoE/31B, up to 256K ctx; https://opensource.googleblog.com/2026/03/gemma-4-expanding-the-gemmaverse-with-apache-20.html ,
  https://www.infoq.com/news/2026/04/google-gemm4/); Qwen 3.5 family (snippets above).

Quality vs Claude for summaries: no rigorous head-to-head found **[unverified]**. Practical judgement: 2–4B local models write
acceptable short English summaries but are noticeably weaker at faithful multi-document synthesis (weekly retro, sprint close),
instruction following with structured output, and Polish; 8–9B models narrow the gap but still trail Claude Sonnet/Opus clearly
on long-context synthesis.

Is shipping one in-app sensible? **Not now.** It adds a 1.5–5 GB download, RAM pressure, thermal/battery cost and a second
prompt-tuning target, for weaker results in Polish. Better: keep Claude as the default; make the model pluggable (the OS 27
Foundation Models `LanguageModel` protocol, or an OpenAI-compatible local endpoint such as Ollama / LM Studio / mlx-serve) so a
user can opt into a local model; use Apple's on-device model only for English micro-tasks.

### B4. Bring your own key, sign-in, and terms

- **Anthropic terms allow BYOK.** Anthropic's own legal page: customers "may not pay for, resell, or intermediate Claude usage on
  their end users' behalf. Each end user must authenticate with their own Anthropic API key, Claude subscription plan
  credentials, or 3P inference provider credential." (This text is in the Claude Code section but states the general position.)
  https://code.claude.com/docs/en/legal-and-compliance (fetched)
- **No "Sign in with Claude" for third-party apps.** Same page: "Anthropic does not permit third-party developers to offer
  Claude.ai login into their own applications, or to route requests through Free, Pro, or Max plan credentials on behalf of their
  users. Moreover, developers may not collect, store, or intermediate Claude.ai credentials or session tokens." Reported as a
  formal policy change in Feb 2026 (https://winbuzzer.com/2026/02/19/anthropic-bans-claude-subscription-oauth-in-third-party-apps-xcxwbn/).
  So a user's Claude Pro/Max subscription cannot power this app; only API keys (or Bedrock/Vertex/Foundry credentials).
- **New option: App Attest.** Claude for Foundation Models (Swift package, Apache-2.0, v0.1.0, beta, needs OS 27) supports
  `.appAttest(clientID:)`: the app ships no key; Anthropic issues 1-hour device tokens after Apple App Attest, scoped to Messages
  API, **billed to the developer's workspace**, revocable in Console. Also `.proxied` (your backend adds the key) and `.apiKey`
  (development only: "A key bundled into an app is extractable"). https://platform.claude.com/docs/en/cli-sdks-libraries/libraries/apple-foundation-models (fetched);
  https://claude.com/blog/claude-for-foundation-models
- Key-handling best practice for a desktop app (synthesis of the above plus Apple's WWDC26 guidance "Never store private keys in
  your app binary… store them securely using Keychain", https://developer.apple.com/videos/play/wwdc2026/241/):
  - user-entered key goes in the **macOS Keychain** (iCloud Keychain sync optional), never in the SQLite file, logs, crash
    reports (Sentry scrubbing) or exports;
  - call the API from the native/Rust side, not from the WebView's JS, so page content and editor plugins cannot read the key;
  - validate on entry with a cheap call, show spend, support rotation/removal; tell users to create a dedicated key with a spend
    limit in Console; Anthropic's own advice: a key "is a digital key to your account—much like a credit card number"
    (https://support.anthropic.com/en/articles/9767949-api-key-best-practices-keeping-your-keys-safe-and-secure , snippet).
  - for the owner's own use, the web app keeps the server-side key; the Mac app can use the same user's key from Keychain.
- **Voyage:** no OAuth found. ToS: if you use the Service on behalf of a third party you must have all necessary authorisations
  (https://www.voyageai.com/tos , snippet); billing goes to the key's account. No explicit BYOK prohibition found **[unverified:
  full ToS not readable]**. An open-weight local model (voyage-4-nano) removes the question for the desktop app.

### B5. Prompt injection from saved web pages

The app saves web pages (Library links), has private data (journal, tasks, pages) and plans "Ask your OS" (R4.7) with tools.
That is two legs of the lethal trifecta by design.

- **Simon Willison, "lethal trifecta" (16 Jun 2025)**: private data + untrusted content + ability to communicate externally =
  data theft; the reliable fix is to remove a leg, and the easiest leg to cut is exfiltration.
  https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/ (snippet). Follow-up (2 Nov 2025) covers Meta's **"Agents Rule of
  Two"** (an agent should have at most two of: untrusted input, access to sensitive data/systems, ability to change state or
  communicate externally) and "The Attacker Moves Second" (adaptive attacks broke published defenses at high rates)
  https://simonwillison.net/2025/Nov/2/new-prompt-injection-papers/ **[details from memory; page blocked]**.
- **OWASP Top 10 for LLM Applications 2025, LLM01 Prompt Injection**: covers indirect injection via websites/documents;
  mitigations are defense in depth: constrain behaviour in the system prompt, define output formats, segregate and mark external
  content, least privilege (scoped credentials, allow-listed tools), human approval for sensitive actions, output filtering,
  audit logs. Related: LLM06 Excessive Agency **[number from memory]**. https://www.mend.io/blog/2025-owasp-top-10-for-llm-applications-a-quick-guide/ ;
  https://www.promptfoo.dev/docs/red-team/owasp-llm-top-10/ (snippets)
- **Anthropic**: "Mitigating the risk of prompt injections in browser use" (24 Nov 2025): RL training against injections,
  classifiers, Opus 4.5 at ~1% attack success, and explicitly "no browser agent is immune"; 1% is still "meaningful risk".
  Guidance: never put untrusted content in the system prompt; keep it in user/tool-result content clearly delimited.
  (https://www.pymnts.com/news/artificial-intelligence/2025/anthropic-pushes-back-hackers-press-ai-weak-spots/ ;
  https://venturebeat.com/security/anthropic-browser-agent-hijacked-31-percent-before-safeguards-engaged ; snippets; Anthropic page itself not fetched.)
  A July 2026 report says Opus 5 with auto mode reached zero successes in internal browser tests (secondary, **[unverified]**:
  https://creati.ai/ai-news/2026-07-25/anthropic-says-claude-opus-5-with-auto-mode-drove-browser-prompt-injection-success-to-zero-in-in/).

Concrete rules for this app:
1. **Cut the exfiltration leg.** Tools available while saved-web content is in context must not reach the network: no web
   fetch/search, no email/HTTP tools, no auto-loaded remote images or links in rendered model output (markdown image
   exfiltration), no writing to shared/public pages.
2. **State-changing tools need confirmation** (create/edit/delete objects, sprint moves) when any untrusted content is in the
   context; show a diff.
3. **Quarantine pattern**: summarise/extract web pages in a separate, tool-less call whose output is stored as data (with
   provenance `source=web`) and is never treated as instructions by later calls.
4. Put page text in tool results or document blocks with clear delimiters, never in the system prompt; strip hidden text
   (CSS-hidden, `aria-hidden`, zero-width chars) at save time.
5. Least privilege per feature: triage and summaries get read-only tools; "Ask your OS" gets read tools plus confirm-gated
   writes; log every tool call in the `ai_usage` ledger.
6. Local models are **more** injectable than Claude, so a local LLM must get even fewer tools.

---

## Summary of recommendations and triggers to revisit

| Decision | Recommendation | Revisit if |
|---|---|---|
| Editor | Keep BlockNote, in a WebView on Mac/phone; Yjs for page bodies; pin versions; avoid `xl-*` | BlockNote stalls or loses funding (→ Tiptap 3); sync ADR picks Loro/Automerge (→ adapter or opaque blob); native editor becomes a must |
| CRDT for text | Yjs (y-prosemirror, Yjs 14 history coming) | loro-prosemirror reaches 1.0 and someone ships a BlockNote adapter |
| Embeddings | Spike voyage-4-nano locally (shared space with stored voyage-4-lite 512-d vectors); fallback Qwen3-Embedding-0.6B / harrier-270m / granite-311m-r2 with re-index | nano cross-model recall or Polish quality is poor; Voyage changes licence/pricing |
| LLM | Claude via API; pluggable model interface; Apple FM only for English micro-tasks | Apple adds Polish; a ≤4B open model matches Claude on our retro/summary evals |
| Keys | BYOK in Keychain, API calls from native side; consider App Attest if distributing to others on OS 27 | Anthropic launches a third-party OAuth |
| Injection | Remove exfiltration leg; confirm writes; quarantine web content | Never relax just because models improve |

## Could not verify
- BlockNote 1.0 plans; BlockNote long-document benchmarks.
- Tiptap 2026 price list and Hocuspocus 4 details (tiptap.dev blocked).
- Voyage blog/docs directly (blocked); voyage-4-nano benchmark scores (published as "TBD"); whether nano↔lite shared space holds at 512 dims.
- EmbeddingGemma licence wording; multilingual-e5/bge-m3/Plate/Milkdown/Yoopta/Editor.js/Novel licences (from memory).
- Polish-specific retrieval scores for any model; Apple Silicon speeds for most embedding models.
- Whether Foundation Models refuses or merely degrades on Polish input.
- Exact text of Anthropic's browser prompt-injection post and of Simon Willison's Nov 2025 post (blocked).
