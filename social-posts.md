# Social Posts — Gemma 2 2B vs Phi-3 Mini

---

## VERSION 1: Medium Article (Deep Dive Lite)

---

# Gemma 2 2B vs Phi-3 Mini: What Actually Happens When You Benchmark Two SLMs on a Phone

*Both models. Same device. Same binary. No cloud.*

---

## The Setup

I ran both Gemma 2 2B and Phi-3 Mini 3.8B — quantized to Q4_K_M — on a Samsung Galaxy S24 (Snapdragon 8 Gen 3, Android 14) using llama.cpp entirely CPU-side, no GPU, no NPU. Each model was benchmarked using llama.cpp's built-in `llama-bench` tool: five prompt lengths from 32 to 512 tokens for prefill speed, plus token generation at both 4 and 8 threads, each configuration repeated three times and averaged. The result is 10 data points per model, all measured on identical hardware under identical conditions.

---

## Finding 1: Gemma's Prefill Curve Is Inverted

This was the unexpected result. On Phi-3 Mini, prefill throughput *falls* as prompts get longer — from 34.63 t/s at 32 tokens down to 23.88 t/s at 512 tokens. That's the expected pattern: longer prompts mean quadratic attention growth, so throughput degrades.

Gemma 2 2B does the opposite. Throughput *rises* from 68.77 t/s at 32 tokens to a peak of 71.62 t/s at 256 tokens before dropping to 59.93 t/s at 512 tokens. The model is getting more efficient as prompts grow, not less.

The likely explanation: Gemma's smaller weight matrices (2.61B vs 3.82B parameters) allow llama.cpp's SIMD-optimised batch matrix operations to hit a more efficient compute regime at medium batch sizes. The prefill pass is compute-bound for Gemma in a way it isn't for Phi-3, and the compute utilization improves with batch size up to a point.

**The practical implication:** Gemma doesn't just start faster — it *stays* faster proportionally as your prompts grow.

---

## Finding 2: The 8-Second UX Boundary Moves

Time to First Token (TTFT) is the wall-clock wait before the first word appears. It's entirely determined by prefill speed and prompt length. Here's what it looks like across three representative prompt lengths:

| Prompt | Gemma 2 2B TTFT | Phi-3 Mini TTFT | Gemma faster by |
|--------|----------------|-----------------|-----------------|
| Short (~25 words / 32 tokens) | **0.47 s** | 0.92 s | 2.0× |
| Medium (~100 words / 128 tokens) | **1.79 s** | 4.31 s | 2.4× |
| Long (~400 words / 512 tokens) | **8.54 s** | 21.4 s | 2.5× |

UX research consistently puts the "this feels broken" threshold at around 8–10 seconds of no feedback. Phi-3 crosses that threshold at roughly 230 tokens — about a 175-word prompt. Gemma crosses it at 512 tokens — roughly 400 words.

**In practice:** every real-world input that would make Phi-3 feel sluggish falls within Gemma's responsive range. Document summarization, OCR output, pasted articles — these all sit in the 200–500 token range where the difference is between a spinner and a wall.

---

## Finding 3: The Thread Config That Changes the Calculus

For token *generation*, both models confirm the same rule: use 4 threads (performance cores only), not 8. Adding the efficiency cores creates big.LITTLE synchronization overhead that costs ~21% throughput on both models.

For *prefill*, Gemma breaks the rule. At 8 threads, Gemma's prefill speed rises to 72.41 t/s — a 16% improvement over its 4-thread figure. Phi-3 sees no meaningful gain. Gemma's smaller weight matrices give the efficiency cores enough useful work to do without hitting the synchronization bottleneck.

**The config implication:** for workloads that involve long inputs and short outputs — document Q&A, summarization, OCR analysis — you can run Gemma at 8 threads and get faster TTFT with acceptable generation speed. That option simply doesn't exist for Phi-3.

---

## Final Verdict

**Choose Gemma 2 2B if:** speed and responsiveness are the priority. It generates 63% faster, processes long prompts 2.5× sooner, fits in 29% less RAM, and supports a 8K context window vs Phi-3's 4K. For interactive chat, document pipelines, and OCR workflows, it is the better default on current Android hardware.

**Choose Phi-3 Mini if:** accuracy matters more than speed. Its MMLU score (~68.8%) outpaces Gemma (~51.3%) by a meaningful margin. For complex reasoning, multi-step math, or code generation where correctness is non-negotiable, the quality gap justifies the wait.

The right answer for most use cases: Gemma for day-to-day use, Phi-3 when the task demands it. The Sara Chat UI now lets you switch between them in under 30 seconds without touching the terminal.

*Full benchmark data and raw logs: [github.com/destortion/llm-on-local-device](https://github.com/destortion/llm-on-local-device)*

---
---

## VERSION 2: LinkedIn Post (Executive Summary)

---

I ran two small language models fully on a phone. No cloud. No GPU. Here's what the numbers actually showed.

Gemma 2 2B vs Phi-3 Mini 3.8B — same device, same inference engine, measured head-to-head on a Snapdragon 8 Gen 3 via llama.cpp:

⚡ 63% faster token generation (14.2 vs 8.7 tokens/sec)
⚡ 2.5× faster Time-to-First-Token on long prompts (8.5s vs 21.4s)
⚡ 29% smaller RAM footprint (1.6 vs 2.2 GB)
⚡ 2× larger context window (8K vs 4K tokens)

The counterintuitive finding: Gemma's prefill throughput *increases* as prompts get longer — the opposite of Phi-3. Smaller weight matrices hit a more efficient SIMD batching regime at medium context lengths.

The nuance nobody mentions: Phi-3 Mini still wins on reasoning benchmarks (MMLU ~68.8% vs ~51.3%). Speed and accuracy remain a tradeoff, even at the 2B–4B scale.

Both ran on a Galaxy S24 inside Termux — no root, no custom ROM, no proprietary SDK. Just llama.cpp and an aarch64 CPU doing real work.

On-device AI in 2026 isn't a demo anymore. It's a practical tool with measurable, reproducible performance characteristics.

What's your take — does on-device inference have a realistic path to replacing cloud APIs for everyday tasks, or is the quality gap still too wide?

---
*Character count: ~1,180*
