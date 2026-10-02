# Bench: local embedding models vs an API

Input to ADR-0001 (semantic index). Measured 2026-10-02. Scripts and raw output are in the session scratchpad (`bench/embed/`); the core script is at the end of this note.

## TL;DR

- **Local default: Qwen3-Embedding-0.6B**, 8-bit weights, Matryoshka-cut to **512 dims**, vectors stored as **float16** (1 KB per chunk). It has the best public scores of the small open models on everything we care about: all of MMTEB, the Polish slice, the English retrieval slice, and RTEB. It is Apache 2.0 and takes 32K tokens of context. Its cost is size: it is the slowest model here on a CPU.
- **Runner-up: EmbeddingGemma-300M.** It is 2–3 points lower on quality and about 2–3× faster on a CPU. Pick it if the index must be built on weak hardware (a small Linux hub, or the phone). The Gemma licence is not OSI.
- **API default (opt-in): voyage-4-lite**, at $0.02 per million tokens. On the 16 RTEB retrieval tasks that every model shares, it is **5.6 points above Qwen3-0.6B** (76.9 vs 71.3). There are no public Polish scores for it.
- Don't use multilingual-e5-small (weakest), arctic-embed-m-v2.0 (weak on Polish and on everything except RTEB) or nomic-v2-moe (512-token limit, middling scores). bge-m3 scores well on Polish but is as slow as Qwen and weaker overall.
- **Caveat:** Hugging Face is blocked from this machine, so **no real weights could be run.** Speed and RAM come from architecture-identical models with random weights. Speed and memory depend on shapes, not on weight values, so these numbers hold. The quality probe could only run its BM25 baseline. Quality figures therefore come from the official MTEB results repository, which I recomputed locally.

## 1. Machine and method

| | |
|---|---|
| Machine | Linux x86_64, 4 vCPU Xeon @ 2.8 GHz (AVX2, AVX-512, VNNI; no AMX), 16 GB RAM, no GPU |
| Contention | **Another agent's benchmark ran on the same VM the whole time** (load average 3–5 on 4 vCPU). At 4 threads, a 2048² matmul fell from 126 to 15 GFLOPS from oversubscription, so every run uses **2 threads**. The load average is logged per run. Treat the numbers as ±30% and a pessimistic floor. |
| Software | Python 3.11.15, torch 2.14.1 (CPU), transformers 5.18.0, onnxruntime 1.30.0, onnx 1.23.1, mteb 2.22.1 |
| Network | `huggingface.co`, its mirrors, ModelScope, Kaggle, Ollama, the qdrant-fastembed GCS bucket and most vendor sites are **blocked** by the egress policy (403). GitHub raw files and git clones of public repos work. |

**Speed and RAM.** Each model is rebuilt from its published architecture (layers, hidden size, heads, FFN, vocabulary, GQA, head dim) with random weights. Parameter counts match the model cards: e5-small 118M, bge-m3 568M, Qwen3 596M, EmbeddingGemma 303M (plus 5M for its two small output projections, which are left out). Each model is saved as fp32 safetensors. Then, in a **fresh process**, the script times the load plus the first forward pass ("cold start", with a warm page cache) and reads RSS. It then measures throughput for batch 1 and batch 32 at 60 and 400 tokens, using random token ids, without padding or tokenizer time. Runtimes tested: PyTorch fp32, PyTorch dynamic int8 (Linear layers only), and ONNX Runtime fp32 and int8 (dynamic quantisation of a plain `torch.onnx` export, without the transformer-fusion optimiser). The ONNX export failed for Gemma3 and ran out of disk for the two models above 2 GB, so those have PyTorch numbers only.

Two models are proxies, marked with \*:
- **arctic-m-v2**: its real body is gte-multilingual-base (12 × 768, RoPE). I used XLM-R-base shapes, which do about the same compute.
- **nomic-v2-moe**: it has 8 experts and routes each token to 2, in every other layer. I used a dense 12 × 768 model with FFN = 4608, which matches the active compute. Its RAM is understated: the real model has 475M parameters, so about 1.9 GB of weights in fp32 and about 0.5 GB in int8.

**Quality.** I sparse-cloned [`embeddings-benchmark/results`](https://github.com/embeddings-benchmark/results) (commit `ecd91ce`, 2026-10-01) and computed the scores with `mteb` 2.22.1. My MMTEB means match the published figures (Qwen3-0.6B 64.34 vs 64.33 on its card; EmbeddingGemma 61.15; bge-m3 59.56), so the pipeline is sound.

## 2. Speed (2 threads, contended Xeon; higher is better except latency)

Latency is for one text (batch 1). Throughput is texts per second at batch 32. "Load" is the time from process start to the first embedding.

| Model | Runtime | Load s | RSS MB | Latency 60 tok, ms | Throughput 60 tok, /s | Latency 400 tok, ms | Throughput 400 tok, /s |
|---|---|---|---|---|---|---|---|
| multilingual-e5-small (118M) | torch fp32 | 0.5 | 909 | 53 | 42 | 206 | 7.2 |
| | torch int8 | 0.9 | 1303 | 26 | 70 | 132 | 7.2 |
| | ORT fp32 | 3.8 | 1052 | 26 | 40 | 276 | 4.2 |
| | **ORT int8** | 0.9 | **666** | **15** | **72** | 192 | 5.6 |
| arctic-embed-m-v2.0\* (305M) | torch fp32 | 0.8 | 1157 | 102 | 17 | 442 | 2.2 |
| | ORT int8 | 0.9 | 897 | 31 | 36 | 302 | 3.3 |
| nomic-v2-moe\* (475M / 305M active) | torch fp32 | 0.9 | 1263\* | 138 | 12 | 649 | 1.5 |
| | ORT int8 | 1.7 | 944\* | 34 | 33 | 339 | 3.0 |
| EmbeddingGemma-300M | torch fp32 | 0.8 | 1215 | 166 | 11 | 689 | 1.2 |
| | torch int8 | 3.7 | 2110 | 110 | 16 | 473 | 1.3 |
| bge-m3 (568M) | torch fp32 | 1.7 | 1991 | 358 | 5.1 | 1521 | 0.7 |
| | torch int8 | 9.2 | 3328 | 142 | 9.5 | 837 | 1.0 |
| Qwen3-Embedding-0.6B (596M) | torch fp32 | 1.9 | 2509 | 556 | 3.0 | 2356 | 0.3 |
| | torch int8 | 6.6 | 3527 | 225 | 5.4 | 1290 | 0.5 |

How to read it:
- **RSS for "torch int8" is inflated.** The fp32 copy stays resident and the embedding table stays fp32. Real int8 footprints are roughly: weights ≈ parameter count × 1 byte, plus 200–400 MB of runtime. See the ORT int8 rows, and note that Q8 GGUF/MLX files for Qwen3-0.6B are about 0.6 GB. Peak RSS during ONNX load was 2–3 GB, because ORT copies the initialisers.
- Most of the parameters in the 250K-vocabulary XLM-R models (e5, arctic, bge-m3) are the embedding table, which costs RAM but no compute. Qwen3 and Gemma spend their parameters on layers. That is why they are slower on CPU per parameter.
- **int8 speeds things up 1.5–2.5× at short lengths**, and less at 400 tokens, where attention dominates.
- **This box is far slower than a Mac.** The published Apple figure for Qwen3-0.6B on MLX is about 44K tokens/s ([below](#4-apple-silicon-numbers-published)). Here it does 0.5 × 400 = **~200 tokens/s**. A Linux CPU hub is a poor place to build a Qwen3 index: 5M tokens would take about 7 h here (int8, 2 contended threads), against about 2 min on an M2 Max. A personal corpus of about 5M tokens is an estimate.

## 3. Quality

### 3a. Tiny personal probe (60 items, 20 queries, PL+EN)

`probe.py` holds 60 realistic items: 30 English and 30 Polish tasks, notes and links. There are 20 queries: 10 English and 10 Polish. Seven are fully cross-lingual (all relevant items in the other language) and ten are mixed. **Only the BM25 baseline could run**, because the model weights are unreachable:

| Method | recall@5 all | MRR@10 all | Same-language (3 q) | Mixed (10 q) | Cross-lingual (7 q) |
|---|---|---|---|---|---|
| BM25 (no stemming) | 0.32 | 0.57 | 0.83 / 0.78 | 0.33 / 0.80 | **0.07 / 0.14** |

What this shows: plain keyword search almost never finds a Polish note for an English query, or the reverse, and Polish inflection hurts even same-language matches. That is the gap the semantic index must close. The probe runs with one command once weights are reachable: `python probe.py st Qwen/Qwen3-Embedding-0.6B --dims 512`. **Run it on the Mac before the ADR is accepted.** It takes about a minute per model.

### 3b. Public scores (MTEB results repo, recomputed 2026-10-02)

| Model | MMTEB v2 mean (131 tasks) | **Polish slice** (12 tasks) | English retrieval slice (15 tasks) | RTEB, 16 common tasks | MTEB(Europe) (65 common) |
|---|---|---|---|---|---|
| multilingual-e5-small | 56.30 | 64.10 | 43.53 | 55.17 | 52.46 |
| EmbeddingGemma-300M | 61.15 | 70.19 | 55.49 | – (3/30 run) | 59.25 |
| **Qwen3-Embedding-0.6B** | **64.34** | **71.81** | **59.35** | 71.30 | **60.39** |
| bge-m3 | 59.56 | 71.14 | 48.22 | 61.74 | 57.07 |
| nomic-embed-text-v2-moe | 57.64 | 67.11 | 49.25 | 60.16 | 54.37 |
| arctic-embed-m-v2.0 | 53.70 | 62.76 | 50.88 | 62.21 | 51.05 |
| OpenAI text-embedding-3-small (ref.) | 53.44 (129 tasks) | 63.20 | 46.89 | 58.40 | 52.99 |
| voyage-4-lite (API) | not run | not run | not run | **76.94** | not run |
| voyage-4 (API) | not run | not run | not run | 79.95 | not run |
| voyage-4-large (API) | not run | not run | not run | 81.56 | not run |
| voyage-4-nano (open) | 9 tasks only | 2 tasks only | – | 1 task only | – |

How these were computed:
- The Polish slice is the mean over the `pol` subsets of the 12 MMTEB v2 tasks that every model above has run. These are mostly classification, STS and bitext; the only retrieval task among them is BelebeleRetrieval, where Gemma scores 92.95, bge-m3 92.90 and Qwen3 89.43.
- The English retrieval slice covers the 15 retrieval and reranking tasks in MMTEB v2 that have English subsets.
- RTEB (beta) is a retrieval benchmark that mixes public and private sets. Its 16 common tasks are the ones that Qwen3, the Voyage models, bge-m3, e5, nomic, arctic and OpenAI all have. **The Voyage results in that repository were submitted under the `mongodb/` (Voyage's owner) namespace, so they are vendor-run.**
- MTEB versions differ between sources, so compare within one column only.

### 3c. Facts per model

| Model | Params | Dims (Matryoshka) | Max tokens | Licence | Released | Price |
|---|---|---|---|---|---|---|
| multilingual-e5-small | 118M | 384 (no MRL) | 512 | MIT | 2024-02 | free |
| EmbeddingGemma-300M | 308M | 768 → 512/256/128 | 2,048 | Gemma Terms (not OSI; gated download) | 2025-09-04 | free |
| Qwen3-Embedding-0.6B | 596M | 1024 → any 32–1024 | 32K | Apache 2.0 | 2025-06-05 | free |
| bge-m3 | 568M | 1024 (+ sparse, multi-vector) | 8,192 | MIT | 2024 | free |
| nomic-embed-text-v2-moe | 475M (305M active) | 768 → 256 | 512 | Apache 2.0 | 2025-02-07 | free |
| arctic-embed-m-v2.0 | 305M | 768 → 256 | 8,192 | Apache 2.0 | 2024-12-04 | free |
| voyage-4-lite | n/a | 1024 default; 2048/512/256; float, int8, uint8, binary | 32K | API | 2026-01-15 | **$0.02 / M tokens** |
| voyage-4 | n/a | as above | 32K | API | 2026-01-15 | $0.06 / M |
| voyage-4-large | n/a | as above | 32K | API | 2026-01-15 | $0.12 / M |
| voyage-4-nano | 346M | 2048 → 256 | 32K | Apache 2.0, open weights | 2026-01-15 | free |
| OpenAI text-embedding-3-small | n/a | 1536 → any | 8,191 | API | 2024-01-25 | $0.02 / M |

Sources:
- Parameters, dims, max tokens, licences and dates come from `mteb` 2.22.1 `ModelMeta` and the results repo's `model_meta.json` (Voyage output dtypes as well).
- Matryoshka ranges come from the [Qwen3-Embedding README](https://github.com/QwenLM/Qwen3-Embedding) (accessed 2026-10-02) and the earlier sprintOS note [research-ai-jobs.md §2](research-ai-jobs.md) for Gemma, nomic and arctic.
- Prices come from LiteLLM's [`model_prices_and_context_window.json`](https://github.com/BerriAI/litellm/blob/main/model_prices_and_context_window.json) (commit of 2026-10-02).
- **Unverified here:** Voyage's free allowance of 200M tokens per account (from research-ai-jobs.md, citing OpenRouter and embeddingcost.com) and the claim that the voyage-4 family shares one embedding space (the [Voyage blog, 2026-01-15](https://blog.voyageai.com/2026/01/15/voyage-4/), referenced by the results repo). Both vendor sites are blocked from this machine.

**Cost check.** About 5M tokens of personal data costs **$0.10** to embed once with voyage-4-lite, and re-embedding on every edit stays at cents per month. So $0 versus nearly $0 does not decide this. Offline use, privacy and the network round-trip do.

## 4. Apple Silicon numbers (published)

| Model | Runtime / chip | Number | Source |
|---|---|---|---|
| Qwen3-Embedding-0.6B | MLX, M2 Max 32 GB | **44K tokens/s** at batch 32; **1–3 ms** for one embedding | [jakedahn/qwen3-embeddings-mlx README](https://github.com/jakedahn/qwen3-embeddings-mlx) (accessed 2026-10-02) |
| EmbeddingGemma-300M | MLX 8-bit, M5 Max | ~87K tokens/s, 3.2 ms per query, 1.6 GB peak; MLX about 1.8× GGUF | [HF card aufklarer/EmbeddingGemma-300M-MLX-8bit](https://huggingface.co/aufklarer/EmbeddingGemma-300M-MLX-8bit), via research-ai-jobs.md; **unverified here** (HF blocked) |
| Runtime support | fastembed-rs (ONNX/candle) ships e5-small, bge-m3, EmbeddingGemma (also 4-bit), nomic-v2-moe and Qwen3-Embedding (candle); mlx-embeddings supports Qwen3 | [fastembed-rs README](https://github.com/Anush008/fastembed-rs), [mlx-embeddings README](https://github.com/Blaizzy/mlx-embeddings) (accessed 2026-10-02) |

I found no published Core ML or llama.cpp Metal numbers for these exact models from a source I could reach. Treat the Mac speed of Qwen3-0.6B Q8 under llama.cpp Metal as **to be measured**. As a rough guide, MLX vs this box is more than 100× faster on bulk work.

## 5. Recommendation

1. **The local model is the default, and the index is built on the Mac:** Qwen3-Embedding-0.6B, 8-bit weights, via MLX on the Mac and llama.cpp/GGUF wherever it must run off-Mac.
   - Use its instruction prompt for queries, and no instruction for documents.
   - Cut to **512 dims** and store **float16**: 1 KB per chunk, so 20K chunks is about 20 MB.
   - Brute-force cosine is fine at that size.
   - Keep `model` and `dims` on every row, as Sprint's ADR-0004 already does.
2. **Don't build the index on a CPU-only hub with Qwen3.** At ~200 tokens/s here, a first backfill takes hours. If the hub must embed while the Mac sleeps, it should embed only new or changed items (seconds each), or the workspace should use EmbeddingGemma.
3. **Phone:** don't embed queries on the phone at first. Ask the Mac or hub, or fall back to FTS5. A 0.6B model is too heavy for that one job.
4. **API as an opt-in, not the default:** voyage-4-lite, 1024 or 512 dims, int8 output. It is the best quality per dollar, but data leaves the device and it needs a network. **One model per workspace:** switching is a background re-embed.
5. **Weights int8, vectors float16.** int8 weights cost little quality on published Q8 builds (unverified for Polish, so measure it) and double CPU speed. int8 or binary vectors are not worth the complexity at 20 MB.

## 6. What would change this

- **The real-weights probe** (`probe.py`, plus a larger set of about 200 queries taken from the user's own data):
  - If EmbeddingGemma ties Qwen3 on Polish and cross-lingual recall, pick Gemma: half the compute, and fine on a hub or phone.
  - If both trail voyage-4-lite by more than 5 points recall@10, make the API the default when a key exists, with the local model as the offline fallback.
- **voyage-4-nano** gets full MMTEB/Polish scores. If it is near Qwen3 and really shares a space with voyage-4-lite, it becomes the best local pick: local and API vectors would be interchangeable.
- **A hub-heavy topology.** If most embedding has to happen on a cheap Linux VPS, use a smaller model (EmbeddingGemma int8, or e5-small if quality can drop by about 7 points on Polish).
- **Licence.** If the multi-tenant product must avoid non-OSI terms, rule out EmbeddingGemma.
- **Mac measurements.** If the MLX or llama.cpp Metal throughput of Qwen3-0.6B Q8 on a base M1/M2 falls below about 2K tokens/s (unlikely given 44K on an M2 Max), drop to Gemma.

## 7. Core script (speed and RAM; trimmed)

```python
# bench.py — run each mode in a fresh process; THREADS=2 python bench.py torch|torchq|ort NAME [fp32|int8]
SHAPES = [(1, 60), (32, 60), (1, 400), (32, 400)]
def run(fwd, vocab):
    out = {}
    for b, n in SHAPES:
        x = rng.integers(1000, min(vocab, 100000), size=(b, n)); fwd(x)          # warm-up
        t = time.perf_counter()
        for _ in range(REPS[(b, n)]): fwd(x)
        dt = (time.perf_counter() - t) / REPS[(b, n)]
        out[f"b{b}x{n}"] = b / dt                                                   # texts/sec
    return out
# cold start: t0 -> from_pretrained(dir) [+ quantize_dynamic for int8] -> first forward; RSS from /proc/self/status
m = cls.from_pretrained(mdir).eval()
if mode == "torchq": m = torch.ao.quantization.quantize_dynamic(m, {torch.nn.Linear}, dtype=torch.qint8)
fwd = lambda x: m(input_ids=torch.from_numpy(x), attention_mask=torch.ones(x.shape, dtype=torch.long)).last_hidden_state
# ORT: torch.onnx.export(...) then onnxruntime.quantization.quantize_dynamic(..., QuantType.QInt8)
```

Architectures (`archs.py`):
- e5-small = XLM-R 12×384, FFN 1536, vocabulary 250K
- bge-m3 = XLM-R 24×1024, FFN 4096
- Qwen3-0.6B = Qwen3 28×1024, 16/8 heads, head dim 128, FFN 3072, vocabulary 151,669
- EmbeddingGemma = Gemma3 24×768, 3/1 heads, head dim 256, FFN 1152, vocabulary 262,144, bidirectional

Quality tables: `mteb_scores.py` and `mteb_pl.py`, which use `mteb.cache.ResultCache` over the cloned results repo.
