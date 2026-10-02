# Research: AI, the always-on agent, MCP, background work, automations, crash reporting

- **Status:** Research input for ADR-0001. Not a decision.
- **Date:** 2026-10-02. All web sources accessed 2026-10-02 unless a source date is given.
- **Inputs:** Sprint's `docs/specs/smarter.md` (R4) and `docs/decisions/0004-semantic-index.md`; the shared brief (Mac first, optional hub, never require a server, about $0 running cost, multi-tenant later).

Scores are 1–5, higher is better (for cost and lock-in, 5 = cheapest / least lock-in).

## Summary

1. **One AI gateway module** in the shared core: every model call goes through it, with per-feature config, a usage ledger built from the API's `usage` fields, budgets checked before each call, prompt caching on by default and the Batch API for anything that can wait. The user's own Anthropic key sits in the Keychain (Mac) or the hub's secret store. No key proxy now.
2. **Embeddings: local by default**, with the API as an opt-in. Shortlist for a spike on a Polish + English test set: Qwen3-Embedding-0.6B (best small multilingual score, Apache 2.0), voyage-4-nano (Apache 2.0, shares an embedding space with the Voyage API models) and EmbeddingGemma-300M (smallest; Gemma licence). Vectors live in the same SQLite file, brute-force search at first.
3. **Local LLMs are not good enough to carry the rituals.** Apple's on-device model is 8K context, and Apple Intelligence does not support Polish. Use code plus embeddings for cheap tasks such as tag suggestions (as Sprint already does), and Claude for rituals and Ask.
4. **The agent is mostly deterministic jobs.** The model only drafts text and proposals through structured outputs. Ask is one bounded, read-only tool loop on the plain Messages API. No Claude Agent SDK and no Managed Agents. Every write is a suggestion the user accepts. Untrusted content never drives a tool call.
5. **MCP server:** stdio on the Mac (Claude Desktop, Claude Code) and Streamable HTTP plus OAuth on the hub. Read scopes plus one write tool, `create_suggestion`. Target spec 2026-07-28 (stateless).
6. **Telegram** for chat and capture, using long polling from the hub or Mac, with an allowlist of user IDs. WhatsApp, Signal and iMessage are rejected for now.
7. **Background work:** a durable job table in the same SQLite database (leases, retries, idempotency keys) on both the Mac and the hub. Sprint boundaries are applied lazily and idempotently. No Redis and no Inngest.
8. **Automations:** no rules engine in v1. Signed outbound webhooks and MCP cover power users. n8n stays outside the product because of its licence.
9. **Crash reports:** off by default. Apple's own crash reports plus MetricKit on the Mac and phone, and an opt-in, Sentry-protocol endpoint for the hub that can point at Bugsink or GlitchTip. No product analytics.

---

## 1. Model access and the AI gateway

### Current Anthropic API facts (as of 2026-10-02)

Prices per million tokens, from the [pricing page](https://platform.claude.com/docs/en/about-claude/pricing):

| Model | Input | Output | Cache read | 5-min cache write | Batch (in/out) | Context |
|---|---|---|---|---|---|---|
| Largest Claude model | $4 | $20 | $0.20 (0.05×) | $5 | $2 / $10 | 1M |
| Mid-size Claude model | $2 | $10 | $0.20 | $2.50 | $1 / $5 | 1M |
| Smallest Claude model | $1 | $5 | $0.10 | $1.25 | $0.50 / $2.50 | 200K |
| Claude Fable 5.1 (top tier) | $10 | $50 | $0.25 | $12.50 | $5 / $25 | 1M |

- **Batch API: 50% off input and output.** It stacks with caching ([pricing](https://platform.claude.com/docs/en/about-claude/pricing)). Results come back asynchronously, in any order.
- **Prompt caching:** a 5-minute write costs 1.25× and a 1-hour write 2×. A read costs 0.1× (0.05× on the largest model). The cache is a prefix match in the order tools → system → messages, and it is isolated per workspace ([pricing](https://platform.claude.com/docs/en/about-claude/pricing); [prompt caching docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)).
- **Tokenizer:** Recent Claude models produce about 30% more tokens for the same text, so cost estimates must use `count_tokens`, not a characters ÷ 4 rule ([pricing](https://platform.claude.com/docs/en/about-claude/pricing)).
- **Structured outputs:** `output_config.format` (JSON schema) and `strict: true` on tools guarantee schema-valid output. On the current largest and mid-size models, forced `tool_choice` (`any`/`tool`) returns a 400, so use structured outputs instead. the largest Claude model cannot turn thinking off: `effort` (low to max, default `medium`) is the only control ([migration guide](https://platform.claude.com/docs/en/about-claude/models/migration-guide)).
- **Safety refusals:** the newest models can stop with `stop_reason: "refusal"`. The gateway must handle that, and a server-side `fallbacks` parameter (beta) can reroute the request.
- **Usage fields** on every response: `input_tokens`, `output_tokens`, `cache_creation_input_tokens` (with a 5m/1h split), `cache_read_input_tokens` and `server_tool_use`. The ledger is built from these, not from estimates.
- **Agent options:** the plain Messages API with your own loop, the SDK's Tool Runner (a thin loop over your tools), the **Claude Agent SDK** (the Claude Code harness as a library: built-in file and bash tools, hooks and permissions; the TypeScript package bundles a native Claude Code binary, the Python one needs the CLI; [Agent SDK docs](https://code.claude.com/docs/en/agent-sdk/typescript)), and **Managed Agents**, which run in Anthropic-hosted sessions at $0.08 per session-hour plus tokens, with no batch discount ([pricing](https://platform.claude.com/docs/en/about-claude/pricing)).

### Design: one gateway module

```
features ──▶ gateway.call(feature, input, {urgency})
               │ 1 look up feature config (model, effort, max_tokens, cache plan, batchable, budget class)
               │ 2 estimate (count_tokens) → check budget (month, feature, run) → refuse or degrade
               │ 3 build request: frozen system + tools first (cached), volatile tail last
               │ 4 send now, or enqueue into a batch job when urgency = "can wait"
               │ 5 parse: stop_reason (refusal, max_tokens), validate structured output
               │ 6 write ledger row from response.usage × versioned price table
               ▼
           result | typed error
```

- **Key storage.** Mac: a Keychain item (`kSecClassGenericPassword`, `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`, not synced to iCloud). Hub: a file readable only by the service user, or the OS secret store. The key never enters the synced database and never goes to other devices. Each device that runs AI holds its own key copy, entered by the user.
- **Per-feature config** is data in the database, not code. Each feature records its model, effort, max_tokens, whether it can be batched, and a monthly cap. Suggested defaults, to be tuned in the spike:

| Feature | Default model | Effort | Mode | Why |
|---|---|---|---|---|
| Sprint close summary, retro draft | the largest Claude model | medium | Batch (results within the hour are fine) | quality matters; batch halves cost |
| Rollover / weekly plan proposal | the largest Claude model | medium | Batch, or now when the user taps **Suggest again** | reasoning over a structured plan |
| Triage of captured items | the smallest or mid-size Claude model | low | Batch per hour | high volume, simple |
| Link summary | the mid-size Claude model | low | now (user-initiated) | user is waiting |
| Ask your OS | the mid-size model (largest as an opt-in) | low/medium | now, streamed | the main cost driver |
| Tag and kind suggestions | none (embedding vote) | – | – | already works without an LLM in Sprint R4.4a |

- **Rough cost for one person** (estimates only; measure in the spike). Weekly rituals: about 40K input and 4K output tokens on the largest model in batch ≈ $0.12 a week. Ask on the mid-size Claude model, retrieval first and 1–3 tool turns: about 20–40K input (mostly cached) and 1K output ≈ $0.03–0.08 a question, so 10 questions a day ≈ $10–25 a month. **Ask is what the budget is for**, as Sprint already found ($7–10 a month with Ask in `smarter.md`).
- **Budgets:** a monthly cap that the user sets. At 80% the app shows a notice. At 100% interactive features pause and scheduled rituals still run, up to their own small cap. A per-run cap uses `task_budget` (beta) or a turn limit in our loop. The ledger is append-only and covered by the event log's tenancy rules, like `app.ai_usage` in Sprint.
- **Caching plan:** the system prompt and tool definitions are frozen per release and sorted deterministically. Workspace context (ontology, active sprint) goes into a second cached block. The question goes last. A check that `cache_read_input_tokens > 0` on the second call of a loop goes in CI against a fake server.
- **Provider abstraction:** a thin interface (`complete`, `stream`, `batch`, `embed`) so that Apple's models or a local model can serve a feature. Do not build a general multi-provider layer: one real provider and one fake for tests.

### Should a hosted product proxy keys later?

- **Not now.** Bring-your-own-key keeps our cost at $0 and keeps the user's data flowing straight from their device to Anthropic.
- **Later, for a paid hosted tier:** use one organisation key held by the hosted service, with per-tenant metering in the same ledger shape and budgets enforced server-side. **Never proxy a user's own key through our servers.** Holding other people's keys is liability with no gain. A proxy must strip nothing, log only usage, and pin `inference_geo` if residency is promised (×1.1 price).

| Option | Perf | Security | Maturity | Dev speed | Fit | Cost | Lock-in |
|---|---|---|---|---|---|---|---|
| BYOK, direct from the device or hub (rec.) | 5 | 4 | 5 | 5 | 5 | 5 | 4 |
| Our proxy with user keys | 4 | 2 | 4 | 3 | 2 | 3 | 3 |
| Our org key, metered (hosted tier later) | 4 | 4 | 4 | 3 | 4 | 2 | 3 |

**What would change it:** a hosted tier launches (then use our org key for that tier only), or Apple's Foundation Models `LanguageModel` protocol with Anthropic's Swift package becomes the easier path on Apple devices (see §3).

---

## 2. Embeddings

### Contenders

| Model | Params / size | Dims (Matryoshka) | Context | Multilingual quality | Licence | Notes |
|---|---|---|---|---|---|---|
| Voyage **voyage-4-lite** (API) | – | 2048/1024/512/256 | 32K | no MTEB published; Voyage reports RTEB | commercial API | $0.02/M tokens, first 200M tokens free ([OpenRouter](https://openrouter.ai/voyageai/voyage-4-lite), [embeddingcost](https://embeddingcost.com/voyage)); voyage-4 family shares one embedding space ([MongoDB](https://www.mongodb.com/products/updates/the-voyage-4-series-now-available/), 2026-01-15) |
| Voyage **voyage-4-nano** (open) | ~340M | 2048–256 | 32K | unverified | **Apache 2.0** | Qwen3 backbone; same space as voyage-4-lite/large, so local and API vectors are compatible ([Voyage blog](https://blog.voyageai.com/2026/01/15/voyage-4/), [Fireworks](https://fireworks.ai/models/fireworks/voyage-4-nano)) |
| **Qwen3-Embedding-0.6B** | 0.6B, ~639 MB | 32–1024 | 32K | **MMTEB 64.33** | Apache 2.0 | 100+ languages ([Qwen blog](https://qwenlm.github.io/blog/qwen3-embedding/), [GitHub](https://github.com/QwenLM/Qwen3-Embedding)) |
| **EmbeddingGemma-300M** | 308M, <200 MB quantized | 768–128 | 2K | MTEB multilingual v2 **61.15** | Gemma Terms (not OSI) | 100+ languages ([Google blog](https://developers.googleblog.com/introducing-embeddinggemma/), [paper](https://arxiv.org/pdf/2509.20354)) |
| nomic-embed-text-v2-moe | 475M total / 305M active | 768–256 | 512 | strong on MIRACL (~0.66) | Apache 2.0 | ~100 languages ([Docker Hub card](https://hub.docker.com/r/ai/nomic-embed-text-v2-moe)) |
| Snowflake arctic-embed-l-v2.0 | 568M | 1024 (MRL to 256) | 8K | MIRACL 0.649, CLEF 0.541 | Apache 2.0 | 74 languages; best on out-of-domain CLEF ([paper](https://arxiv.org/html/2412.04506v2)) |
| BGE-M3 | 568M | 1024 | 8K | MIRACL 0.678, CLEF 0.410 | MIT | dense + sparse + multi-vector |
| multilingual-e5-large | 560M | 1024 | 512 | MIRACL 0.651, CLEF 0.431 | MIT | older; short context |

The MIRACL and CLEF numbers for the last three are from the Arctic-Embed 2.0 paper. MTEB versions differ between sources, so compare within one column only. Polish is in MMTEB and MIRACL is multilingual, but **no source gives Polish-only scores for these models**, so the spike must measure them.

**Throughput on Apple Silicon** (published, not ours):
- EmbeddingGemma, MLX 8-bit on an M5 Max: ~87K tokens/s, 253 docs/s, 3.2 ms per query, 1.6 GB peak RAM. MLX is about 1.8× faster than GGUF for bulk work ([HF card, aufklarer](https://huggingface.co/aufklarer/EmbeddingGemma-300M-MLX-8bit)).
- Qwen3-Embedding-0.6B on MLX: ~44K tokens/s ([qwen3-embeddings-mlx](https://github.com/jakedahn/qwen3-embeddings-mlx); chip not stated).
- A personal corpus of about 5M tokens is therefore a few minutes of one-time work, and a single note embeds in milliseconds.

**Runtimes:** MLX (fastest on Mac, Swift and Python bindings, Mac and iOS only); llama.cpp/GGUF (portable to the hub on Linux, Metal on Mac); ONNX Runtime with the CoreML execution provider (portable, partly on the ANE); Core ML (best on iPhone, needs conversion); candle and fastembed-rs (Rust, good for a Rust hub binary; check which models they support). The runtime follows the app language chosen in ADR-0001; GGUF and ONNX are the portable fallbacks.

### Recommendation: local by default, API as an opt-in

| Option | Perf | Security/privacy | Maturity | Dev speed | Fit (PL+EN, offline, phone) | Cost | Lock-in |
|---|---|---|---|---|---|---|---|
| Local Qwen3-Embedding-0.6B | 4 | 5 | 4 | 3 | 4 | 5 | 5 |
| Local voyage-4-nano (+ optional API in the same space) | 4 | 5 | 3 | 3 | 4 | 5 | 4 |
| Local EmbeddingGemma-300M | 5 | 5 | 4 | 3 | 4 (2K context means more chunking) | 5 | 4 (licence) |
| API voyage-4-lite | 4 (network) | 3 | 5 | 5 | 3 (no offline, data leaves the device) | 5 (free tier) | 3 |

- Why local: search by meaning, related items and duplicates work offline, on the plane, with no key and no third party seeing every note. The cost is really $0. The API is almost free too (200M free tokens), so **privacy and offline use decide it, not price**.
- **One model per workspace, pinned.** Vectors from different models cannot be compared. Store `model` and `dims` on every row (as ADR-0004 does), and re-embed in the background when either changes.
- **Where vectors are computed:** on the Mac or hub, which has the model. Vectors are derived data. Either sync them (tagged with model and dims, about 1 KB per chunk at 512 dims in fp16 or int8) or have each device rebuild them. The phone needs the model only to embed queries offline. Otherwise it asks the hub or Mac, or falls back to full-text search.
- **Spike decides between Qwen3-0.6B and voyage-4-nano** (and EmbeddingGemma if phone size dominates). Test set: 200 real queries in Polish and English over a seeded personal corpus, with recall@10, plus Mac and iPhone latency and memory.

### Where the index lives

- In the **same SQLite file** as the objects, in a side table: `object_chunks(object_id, chunk, content_hash, model, dims, embedding BLOB)`. It keeps the tenancy columns. It is not in the event log, and derived data is recomputed, as ADR-0004 does.
- **sqlite-vec**: stable v0.1.9 (2026-03-31). Brute-force `vec0` with float, int8 and bit vectors is stable. ANN indexes (DiskANN, IVF, rescore) are still in alpha (v0.1.10-alpha.4) ([releases](https://github.com/asg017/sqlite-vec/releases)). Brute force over 50K chunks of 512 dims is tens of milliseconds on M-series (to be measured).
- **SQLite's own `vec1`** (IVFADC + OPQ, version 0.7, "no longer preview") is an alternative when ANN is needed ([sqlite.org/vec1](https://sqlite.org/vec1)). Watch it; do not depend on it now.
- **Re-embedding on a model change:** a background job walks rows where `model != current`, at a low priority, in batches. Search keeps using the old vectors until the new set for that object is complete; then the job swaps them in one transaction. Hybrid search (FTS5 + vectors, reciprocal rank fusion with k = 60, as in Sprint R4.2) covers the gap.

**What would change it:** the spike shows local quality on Polish clearly below voyage-4-lite (then make the API the default when a key exists, with local nano as the offline fallback in the same space); or iPhone memory limits rule out every local model above 300M.

---

## 3. Local LLMs on Apple Silicon

- **Apple Foundation Models** (macOS/iOS 26+): about a 3B on-device model with guided generation (typed structured output), tool calling and streaming. It is free and offline, and needs an Apple Intelligence-capable device. At WWDC26: **context 8,192 tokens**, image input on iOS 27, a `LanguageModel` protocol that lets Claude (through Anthropic's Swift package) or MLX models stand in, and a **Private Cloud Compute model with 32K context**. PCC is free for developers under 2M first-time downloads but needs an entitlement and has daily per-user limits ([WWDC26 session 241](https://developer.apple.com/videos/play/wwdc2026/241/)).
- **Language gap:** Apple Intelligence supports 16 languages, and **Polish is not one of them** ([MacObserver](https://www.macobserver.com/news/apple-intelligence-english-only-features-16-languages/), [Apple](https://www.apple.com/ios/feature-availability/)). For a user who writes in Polish, the on-device model is unreliable for anything that reads their text.
- **MLX with small open models** (Qwen3 4–8B, Gemma 3): workable on a Mac with 16 GB or more (tens of tokens per second), but it uses 3–6 GB of RAM and battery. Quality on retro or plan drafting is well below Claude, and it doesn't help the phone or a small hub.

| Use | Best tool | Why |
|---|---|---|
| Tag and kind suggestions, duplicates | embedding neighbours plus a vote (no LLM) | proven in Sprint R4.4a; follows the user's own habits; free; any language |
| Triage hint (task, idea or note?) | embedding classifier first; a smaller Claude model batch if needed | cheap and deterministic |
| Sprint close, retro draft, weekly plan, Ask | Claude | needs long context, reasoning and Polish |
| English-only micro text (titles, one-line summaries) | Apple on-device model, opportunistically | free, but behind the gateway interface so it can be switched off |

**What would change it:** Apple adds Polish, or a local 4–8B model passes our ritual eval at an acceptable level. Then route the cheap features locally through the same gateway.

---

## 4. The always-on agent

### Where it runs

On the hub when one exists; otherwise on the Mac while it is awake. The phone never runs rituals. The rule for which runner wins is a lease in the job table (§7): one runner per workspace holds the lease for scheduled AI work, and the others skip.

### Deterministic jobs, not agent loops

| Work | Shape | Model's role |
|---|---|---|
| Sprint close, rollover | pure code: compute metrics, move items as the rules say (as proposals) | write the summary text from computed facts (structured output) |
| Retro draft | code gathers the evidence | draft sections; every claim cites object IDs; validated before storing |
| Weekly plan | code builds candidates, capacity and constraints | choose and rank moves from a closed list (`accept_carry_over`, `pull_idea`…); code validates each, as in Sprint R4.6 |
| Triage | code finds candidates | classify into fixed labels |
| Ask your OS | **one bounded tool loop**: read-only tools, at most N turns, a token budget | look things up and answer with citations |

- **Use the plain Messages API with our own small loop or the SDK Tool Runner**, not the Claude Agent SDK. The Agent SDK brings the Claude Code harness (file, bash and web tools; a bundled CLI binary). That is the wrong shape and the wrong attack surface for a personal data server, and it adds a Node or Python dependency to the hub. Managed Agents would move the loop and session state to Anthropic and cost runtime; our tools would also have to be reachable from their cloud. Revisit the Agent SDK only if we later want a "do things on my Mac" agent.
- **AI proposes, the user confirms** (Sprint's rule). The agent has **no mutating tools** apart from `create_suggestion(kind, payload, evidence)`. Accepting a suggestion runs the same command a manual edit would, so the event log records the user's action. Destructive moves (drop, archive, delete) always need an explicit accept, even under automations.
- **Audit log:** each run writes one row: feature, trigger, model, prompt version hash, tool calls (name, arguments hash, result size), suggestions created, tokens and cost, outcome. It is kept with the ledger, append-only and visible in Settings → AI.

### Prompt injection defence

Saved links, page summaries, Telegram messages, imported files and MCP inputs are **untrusted**. The guidance in force is: once an agent has read untrusted input, that input must not be able to trigger consequential actions ([Beurer-Kellner et al., "Design Patterns for Securing LLM Agents against Prompt Injections", 2025](https://arxiv.org/pdf/2506.08837)). CaMeL shows that it can be enforced: a privileged LLM plans from the trusted query, a quarantined LLM with no tools reads untrusted data, and an interpreter tracks provenance capabilities on values. It solves 77% of AgentDojo tasks with security guarantees, against 84% undefended ([Debenedetti et al., arXiv 2503.18813](https://arxiv.org/pdf/2503.18813); SaTML 2026).

What we do (cheaper than full CaMeL and enough for read-only tools):
1. **Taint by provenance.** Every object and field carries `source` (user, ai, web, message, import). Text from non-user sources is wrapped as data in the prompt (with delimiters defused, as in Sprint R4.4b), and a mid-conversation `system` message, not user text, carries operator instructions.
2. **Quarantine pattern for ingest.** Summarising or triaging a saved page uses a call with **no tools**, whose structured output (summary, suggested tags) is stored as `source = ai` + `derived_from = web`.
3. **Ask is read-only**, and its answers are rendered without auto-loading remote images or links, which closes the markdown-image exfiltration channel. The agent has no web-fetch tool and no "send message" tool. A Telegram reply goes only to the requesting allowlisted user, and the model cannot choose the recipient. This removes one leg of the "private data + untrusted content + exfiltration" trifecta.
4. **No tool call whose arguments come from untrusted text runs without confirmation.** Suggestions derived from tainted inputs show their provenance in the accept dialog.
5. **Plan-then-execute for rituals:** the tool sequence is fixed in code; the model only fills slots.

**Risks:** Ask could still be misled into a wrong answer by a poisoned note (integrity, not exfiltration). Mitigations are citations on every claim, plus showing which sources were web content.

Scoring for the agent runtime:

| Option | Perf | Security | Maturity | Dev speed | Fit | Cost | Lock-in |
|---|---|---|---|---|---|---|---|
| Deterministic jobs + Messages API + own bounded loop (rec.) | 5 | 5 | 5 | 4 | 5 | 5 | 4 |
| Claude Agent SDK | 3 | 3 | 4 | 4 | 2 | 4 | 3 |
| Managed Agents | 4 | 3 | 3 | 4 | 2 | 3 | 2 |

**What would change it:** we add agent features that do open-ended work on files or the web (then the Agent SDK, sandboxed and with hooks, is worth it), or a vendor-neutral need appears.

---

## 5. MCP server

### Spec facts

- **Current version: 2026-07-28**, which supersedes 2025-11-25. It makes the protocol stateless: no protocol sessions and no `Mcp-Session-Id`. Streamable HTTP requests carry `MCP-Protocol-Version`, `Mcp-Method` and `Mcp-Name` headers. On auth, it adds RFC 9207 issuer validation and tokens bound to their authorization server; **Client ID Metadata Documents are preferred and Dynamic Client Registration is deprecated** ([Stacktree](https://stacktr.ee/blog/mcp-2026-spec-changes), [Equixly, 2026-08-05](https://equixly.com/blog/2026/08/05/stateless-mcp/), [AAIF](https://aaif.io/blog/mcp-is-growing-up)). The primary spec site was blocked from this environment, so these points come from secondary sources and must be **verified against modelcontextprotocol.io** before we build.
- 2025-11-25 had already made MCP servers OAuth 2.1 resource servers with RFC 9728 Protected Resource Metadata and RFC 8707 resource indicators, made CIMD the default and added async Tasks ([Aaron Parecki, 2025-11-25](https://aaronparecki.com/2025/11/25/1/mcp-authorization-spec-update), [Auth0](https://auth0.com/blog/mcp-november-2025-specification-update/)).
- **Clients:** Claude Desktop runs local servers over stdio, packaged as `.mcpb` desktop extensions, and remote servers as custom connectors over Streamable HTTP with OAuth. **Custom-connector traffic comes from Anthropic's cloud**, so a remote server must be publicly reachable ([Claude docs](https://claude.com/docs/connectors/custom/remote-mcp), [mcpverdict](https://mcpverdict.com/mcp/clients/claude-desktop/)). Claude Code supports both, and on the local machine it can reach a LAN hub directly.

### Design

- **Mac:** the app ships a small `sprintos-mcp` binary, launched over **stdio** by Claude Desktop (as an MCPB) or Claude Code. It talks to the running app or opens the database read-only. Its trust is the OS user, plus a per-client grant stored in the app ("Claude Desktop: read, suggest").
- **Hub:** **Streamable HTTP** on the same server binary, with OAuth 2.1. The hub is its own authorization server, uses CIMD, issues short-lived tokens bound to the resource, and has scopes `graph:read`, `graph:read:private` (journal) and `suggestions:write`. A claude.ai connector works only if the user exposes the hub publicly (for example through Tailscale Funnel or a tunnel). This is optional and documented, never required.
- **Tools** (small, typed, paginated, all `readOnlyHint` except the last): `search(query, types?, since?)` (hybrid), `get(id)`, `neighbours(id, edge_types?, depth≤2)`, `list_sprint(week?)`, `timeline(from, to, types?)`, `create_suggestion(kind, target, payload, rationale)`. The last writes a `source = mcp`, `status = suggested` row that the user accepts in the app. No delete, no direct edit, no raw SQL.
- Tool results that contain user content are returned as data with provenance, because the calling client's model is also exposed to injection from notes the user saved from the web.

### Incidents to learn from (2025–2026)

- Prompt injection through tool data: GitHub MCP issue → private-repo leak (Invariant Labs, May 2025), and the Asana MCP cross-tenant exposure (June 2025) ([AuthZed timeline](https://authzed.com/blog/timeline-mcp-breaches)).
- RCE in the stdio launch pattern across SDKs ("Mother of all AI supply chains", OX Security, 2026): 10+ CVEs. Also LiteLLM MCP RCE (CVE-2026-30623) and Windsurf config hijack (CVE-2026-30615) ([OX Security](https://www.ox.security/blog/mcp-supply-chain-advisory-rce-vulnerabilities-across-the-ai-ecosystem/), [Vulnerable MCP](https://vulnerablemcp.info/)).
- Tool-metadata poisoning, including hidden Unicode TAG characters in descriptions ([arXiv 2607.05744](https://arxiv.org/pdf/2607.05744)); rug-pull servers that change behaviour after install; Context7 prompt injection, CVSS 9 (CVE-2026-75130).
- NSA guidance on MCP security design, June 2026 ([CSI](https://media.defense.gov/2026/Jun/02/2003943289/-1/-1/0/CSI_MCP_SECURITY.PDF)).

Lessons we apply: static tool descriptions with an ASCII-only check in CI; no shell, no file paths and no config the server reads from the model; least-privilege scopes; bind to localhost or authenticate; no token passthrough.

| Option | Perf | Security | Maturity | Dev speed | Fit | Cost | Lock-in |
|---|---|---|---|---|---|---|---|
| stdio (Mac) + Streamable HTTP/OAuth (hub), suggestion-only writes (rec.) | 5 | 4 | 4 | 3 | 5 | 5 | 5 |
| Read-only only | 5 | 5 | 4 | 4 | 3 | 5 | 5 |
| Full write tools | 5 | 2 | 4 | 3 | 3 | 5 | 5 |

---

## 6. Messaging capture and chat

| Channel | Cost | Works without a public IP | Effort | Security notes | Verdict |
|---|---|---|---|---|---|
| **Telegram bot** | free | **yes, long polling** (`getUpdates`, 30–60 s timeout); a webhook needs public HTTPS on 443/80/88/8443 ([Bot API](https://core.telegram.org/bots/api)) | low | bot chats are not end-to-end encrypted (Telegram servers see them); allowlist `from.id`; token as a secret | **recommended** |
| WhatsApp Business Platform | per message; from 2026-10-01 service replies are billed after 1,000 free a month per number ([Engagelab](https://www.engagelab.com/blog/whatsapp-pricing-2026-service-message-cost), [Meta](https://whatsappbusiness.com/products/platform-pricing/)) | no (webhook only) | high (business verification) | Meta-hosted Cloud API | reject |
| Signal via signal-cli | free | yes | medium | unofficial; acts as a linked device; the daemon has no auth of its own; reports of CPU blow-ups after months ([signal-cli](https://github.com/AsamK/signal-cli), [issue #1585](https://github.com/AsamK/signal-cli/issues/1585)) | later, opt-in |
| iMessage | free | needs a Mac | high | no API; Apple bans automated accounts with no published limits ([Lindy](https://www.lindy.ai/blog/imessage-api-three-rewrites-one-apple-ban-and-what-actually-works)) | reject |

- **Design:** the hub (or the Mac when awake) runs one long-poll loop. An inbound message from an allowlisted user ID becomes a capture (`source = message`, tainted), or a read-only Ask if it starts with `?`. Replies go only to that chat. A pairing flow puts a one-time code in the app, which the user sends to the bot, and that binds their user ID. Unknown senders are ignored and counted. The Telegram long-poll offset is stored in the database, so a restart never replays or drops updates.
- **Mac-only users:** polling works while the Mac is awake. Telegram keeps undelivered updates for 24 hours, so short sleeps lose nothing; longer sleeps lose updates. This is the plain trade-off of "no server".

---

## 7. Background work (replacing Inngest)

### Mac

- **In-app scheduler** for work while the app runs, plus **`NSBackgroundActivityScheduler`** for deferrable maintenance (embedding sweep, re-embed, batch result polling). It respects power and thermal state and App Nap.
- **Optional `SMAppService.agent`** (macOS 13+): a bundled, signed LaunchAgent that keeps a small helper alive when the window is closed, so the Mac can be the "hub" (Telegram polling, rituals, MCP). It shows up in System Settings → Login Items, so the user can see it and turn it off ([theevilbit](https://theevilbit.github.io/posts/smappservice/)). It is off by default and asked for at first run.
- A sleeping Mac runs nothing, so jobs must be safe to run late (see the lazy boundary below).

### Hub (and the Mac): a durable job table in the same SQLite file

```
jobs(id, workspace_id, kind, idempotency_key UNIQUE, payload, run_at, priority,
     status queued|running|done|failed|dead, attempts, max_attempts,
     lease_owner, lease_until, last_error, created_at, finished_at)
```
- Claim: `BEGIN IMMEDIATE; UPDATE jobs SET status='running', lease_owner=?, lease_until=now+60s WHERE id = (SELECT id … WHERE status='queued' AND run_at<=now ORDER BY priority, run_at LIMIT 1) RETURNING *`. SQLite has a single writer, so this is race-free in one process. Leases cover crashes. Workers heartbeat the lease on long jobs.
- Retries use exponential backoff with jitter. `dead` after max_attempts appears in Diagnostics. Schedules (cron-like) are rows that enqueue their next run when they finish, keyed by `kind:period` so they never double-fire.
- Debounce (the "embed one minute after the last edit") is an upsert on `idempotency_key = embed:<object_id>` that pushes `run_at` forward.
- **Libraries:** if the core is Rust, `apalis` with `apalis-sqlite` (heartbeats, orphan re-enqueue; active in 2026; [lib.rs](https://lib.rs/crates/apalis-sqlite)) is an option, but the table above is about 300 lines and we control it. **BullMQ needs Redis: avoid.** Graphile Worker is Postgres-only.
- **Jobs are not synced as data.** Each runner has its own queue. Their *effects* are synced objects and events.

### Idempotent, lazy sprint boundary

- The boundary is a fact of time, not an event to catch. On every read and every runner tick: if `now ≥ sprint.end` and the sprint is not closed, apply `close(sprint_id)`. Its operations have **deterministic IDs** derived from `(sprint_id, "close")`, so two devices doing it offline produce the same event, and the merge deduplicates it. The AI summary is a separate job keyed `summary:<sprint_id>` that only the lease holder runs.

### iOS

- `BGAppRefreshTask` gets about 30 seconds, opportunistically. `BGProcessingTask` gets minutes, mainly when the phone is idle and charging. iOS 26 adds `BGContinuedProcessingTask` for user-started work, with a progress UI ([WWDC25 227](https://developer.apple.com/videos/play/wwdc2025/227/), [guide](https://swiftcrafted.dev/article/swiftui-background-tasks-ios-26-bgapprefreshtask-bgprocessingtask-bgcontinuedprocessingtask)). Nothing important can depend on these: the phone syncs and applies lazy boundaries when opened, and never runs rituals.

| Option | Perf | Security | Maturity | Dev speed | Fit | Cost | Lock-in |
|---|---|---|---|---|---|---|---|
| Own SQLite job table (rec.) | 5 | 5 | 4 | 4 | 5 | 5 | 5 |
| apalis-sqlite (if Rust) | 5 | 5 | 3 | 4 | 4 | 5 | 4 |
| Inngest / hosted queue | 4 | 3 | 5 | 4 | 2 (needs a server, cost) | 2 | 2 |
| BullMQ + Redis | 5 | 4 | 5 | 3 | 1 | 3 | 3 |

---

## 8. Automations

- **n8n** is under the Sustainable Use License, which is fair-code and not OSI-approved. A user may self-host it for their own use, including business use. **We may not host it for others for money, white-label it, or embed it in a paid hosted product** without an n8n commercial licence ([SSD Nodes](https://www.ssdnodes.com/learn/n8n-sustainable-use-license-explained), [Wikipedia](https://en.wikipedia.org/wiki/N8n)). So we integrate with it from outside and never bundle it.
- **Recommendation for v1:** no general rules engine. Ship:
  1. **Ritual schedules** (built in, configurable times).
  2. **Signed outbound webhooks** on event types (`task.completed`, `sprint.closed`…). They use an HMAC-SHA256 signature and a timestamp, retry through the job table, and send content only if the user ticks "include content".
  3. **An inbound capture endpoint** on the hub (token-scoped, capture only).
  4. **MCP** (§5) for agent-driven automation.
  Users who want more connect n8n, Zapier or Shortcuts to these.
- **Later**, if real use asks for it, a small built-in rules engine: `when <event + filter> then <one of a fixed action list>`. Actions are suggestion-only for anything destructive. Rules are stored as data in the ontology, so they sync and stay multi-tenant safe.

| Option | Perf | Security | Maturity | Dev speed | Fit | Cost | Lock-in |
|---|---|---|---|---|---|---|---|
| Webhooks + inbound capture + MCP (rec.) | 5 | 4 | 5 | 4 | 4 | 5 | 5 |
| Built-in rules engine now | 5 | 4 | 3 | 2 | 4 | 5 | 4 |
| Bundle n8n | 3 | 3 | 5 | 3 | 2 | 4 | 2 (licence) |

---

## 9. Crash and error reporting

| Option | Weight | Licence | Free tier | Privacy | Fit |
|---|---|---|---|---|---|
| Apple crash reports (Xcode Organizer / App Store Connect) | none | – | free | only from users who share analytics with developers; symbolicated | Mac and iOS, App Store or TestFlight builds |
| MetricKit (`MXCrashDiagnostic`, hangs, CPU, disk) | none | – | free | payload delivered **to the app** on the device; we decide what to send | Mac and iOS, any distribution |
| Sentry SaaS | – | FSL/BSL | 5K errors a month, 1 user, 30 days ([sentrypricing](https://sentrypricing.com/free-plan)) | `beforeSend` scrubbing; PII off by default | good SDKs, third party |
| Sentry self-hosted | 40+ containers, 16 GB RAM minimum, 24–32 GB in practice ([develop.sentry.dev](https://develop.sentry.dev/self-hosted/), [issue 3467](https://github.com/getsentry/self-hosted/issues/3467)) | FSL | – | full control | far too heavy |
| GlitchTip | ~4 containers (Django, Celery, Postgres, Redis), ~512 MB | MIT | – | full control | Sentry-SDK compatible ([Dash0](https://www.dash0.com/comparisons/best-glitchtip-alternatives)) |
| Bugsink | 1 container, SQLite, ~512 MB | PolyForm Shield (source-available) | – | full control | Sentry-SDK compatible ([Bugsink vs GlitchTip](https://www.bugsink.com/blog/bugsink-vs-glitchtip/)) |

**Recommendation and telemetry policy:**
- **Nothing leaves a device unless the user opts in**, per category: crash reports, and separately "include log tail". There is no product analytics or usage telemetry at all. AI usage stays in the local ledger.
- **Mac and iOS:** collect MetricKit payloads and our own panic/crash logs locally. After a crash, show "Send report?" with the exact payload visible and editable. Also rely on Apple's opt-in crash reports.
- **Hub and server:** use a Sentry-protocol SDK with `sendDefaultPii = false` and a `beforeSend` that drops message bodies, object titles and paths. The DSN is empty by default. Our own collector, when we need one, is **Bugsink** (one container, SQLite, cheap VPS) or **GlitchTip** if a fully OSI licence matters. A hosted tier later can use the same.
- Scrubbing rule: reports carry stack traces, versions, OS and hardware, and an anonymous install ID that rotates monthly. They never carry user content, keys or tokens. A test feeds a seeded secret through a crash and asserts it is absent.

---

## Risks and unknowns

- **Polish quality** of local embedding models is unmeasured. This is the biggest open question for §2.
- **MCP spec details** come from secondary sources (the primary site was blocked here). Verify before implementing. The ecosystem's security is still poor, so treat every MCP client as untrusted too.
- **Ask cost** could exceed the user's expectation. Budgets must be on from day one, with a clear spend view.
- **Model churn:** prices and parameters changed three times in 2026 (thinking modes, forced tool choice, refusals). The gateway must keep model specifics in config and in one adapter.
- **Mac-only users** get rituals and Telegram only while the Mac is awake. That is acceptable by design but must be explained in the product.
- **Sync of derived vectors** versus recomputing them per device is undecided. It depends on the sync design in ADR-0001.

## Spikes (each with a pass bar)

1. **Embedding bake-off** (3 days): Qwen3-Embedding-0.6B, voyage-4-nano and EmbeddingGemma locally, and voyage-4-lite by API, on 200 PL/EN queries over a seeded corpus. Pass: the chosen local model reaches recall@10 within 5 points of voyage-4-lite; Mac M1 bulk embedding is at least 2K tokens/s; iPhone query embedding takes ≤ 50 ms and ≤ 300 MB RAM (or we accept no offline meaning search on the phone).
2. **sqlite-vec brute force** (1 day): 50K chunks × 512 dims, int8 and fp32. Pass: p95 under 50 ms on an M1 and on a Raspberry Pi 5 or a small VPS.
3. **Gateway + ledger** (2 days) against a fake server and then the real API: caching hit on the second call of a loop; batch round-trip; refusal handling; ledger matches the Console usage report within 1%.
4. **Ask loop** (3 days): five read-only tools and 30 real questions. Pass: ≥ 80% answered with correct citations; median cost ≤ $0.05 on the mid-size Claude model; an injection test set (20 poisoned saved pages) produces zero tool calls driven by injected text and zero rendered external URLs.
5. **Job table** (1 day): kill -9 during 1,000 jobs; pass if every job runs at least once and side effects run exactly once through idempotency keys. Lazy close on two offline replicas merges to one close event.
6. **MCP** (2 days): a stdio server in Claude Desktop and Claude Code, and a Streamable HTTP + OAuth (CIMD) server on the hub with Claude Code. Pass: scopes are enforced, `create_suggestion` shows up in the app, and the official conformance or inspector tool shows no errors.
7. **Telegram** (½ day): long polling from behind NAT, pairing, allowlist, and restart without replay.
