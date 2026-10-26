# Results and experiment history

All numbers below come from append-only reports in `evidence/`. Each screen was
run on a freshly restarted container (empty prefix cache, fresh KV pool) with a
fresh prompt namespace per round. Nothing here is extrapolated.

## 2026-10-26 — first complete screen

- **Model:** `primitive-ai/Nanbeige4.3-33B-FP8` @ `0140c157…`, 15 files,
  33.11 GiB, 9 LFS SHA-256 values verified against the pinned revision.
- **Base checkpoint:** `Nanbeige/Nanbeige4.3-33B` @ `c1093c50…`.
- **Engine:** vLLM 0.32.0, `torch 2.13.0+cu130`, image
  `vllm/vllm-openai:latest` @ `sha256:befccf0e…`, capability `(12, 1)`.
- **Contract:** TP1, 262,144-token context, 16 sequences, 8,192-token prefill
  chunks, FP8 KV cache, prefix caching and chunked prefill on.
- **Result:** 10/10 screens passed, 30/30 benchmark rounds completed, no
  preemptions and no scheduler queueing observed.

## Startup and capacity

| Observation | Measured |
|---|---|
| Weights loaded into memory | 33.44 GiB |
| Weight load time | 186–230 s (3.1–3.8 min) |
| Engine init after weights | ~39 s |
| Time from `start.sh` to `/health` | ~4.5–5 min |
| KV cache, torch.compile cold | 650,704 tokens / 64.54 GiB |
| KV cache, AOT compile cache warm | 679,872 tokens / 67.43 GiB |
| Concurrency at 262,144 tokens | 2.48× cold / 2.59× warm |
| Host `MemAvailable` while serving | 10.0–12.2 GiB |
| Peak swap used | 2.47 GiB |

The two KV figures are the same recipe on two starts. `torch.compile` artifacts
live in the container's `/tmp` cache; when the first start after a container
recreation has to compile, its graph capture consumes part of the memory plan
and the KV pool is ~29K tokens smaller. Report both rather than one.

## Short decode (1,024 input / 256 forced output)

| Label | Cache | C | Median aggregate tok/s | Decode tok/s p50 (est.) | TTFT p50 | Latency p50 |
|---|---|---:|---:|---:|---:|---:|
| `short-c1-cold` | cold | 1 | **6.59** | 6.65 | 0.471 s | 38.8 s |
| `short-c2-cold` | cold | 2 | **12.97** | 6.59 | 0.727 s | 39.4 s |
| `short-c4-cold` | cold | 4 | **24.82** | 6.56 | 2.415 s | 41.2 s |
| `short-c8-cold` | cold | 8 | **42.11** | 6.43 | 8.991 s | 48.6 s |
| `short-c1-warm` | warm | 1 | **6.64** | 6.64 | 0.167 s | 38.5 s |

Aggregate throughput scales **6.39×** from C1 to C8 while per-request decode
stays within 3% (6.65 → 6.43 tok/s). This is the expected signature of a
bandwidth-bound dense model: batching amortizes one weight read across more
sequences, but each stream still waits ~150 ms per token. TTFT rises from
0.47 s to 8.99 s because eight 1K prefills share the 8,192-token prefill budget.

The measured 6.65 tok/s at batch 1 is **82% of the 8.1 tok/s bandwidth ceiling**
for 33 GiB of weights at ~273 GB/s. There is no realistic engine change that
turns this model into a fast single-stream chat model on one GB10.

Warm reuse works: the second request reported `cached_tokens = 1,040` for a
1,021-token prompt (the chat template is included in the count) and TTFT fell
from 0.47 s to 0.17 s. Every warm round passed the ≥90% reuse gate.

## Prefill-heavy decode

| Label | Prompt | TTFT p50 | Prefill rate | Median aggregate tok/s | Latency p50 |
|---|---:|---:|---:|---:|---:|
| `prefill-16384-c1-cold` | 16,384 | 22.82 s | ~718 tok/s | 4.04 | 63.3 s |
| `prefill-32768-c1-cold` | 32,768 | 50.63 s | ~647 tok/s | 2.75 | 93.0 s |

Prefill is compute-bound and much slower per token than decode is per step:
a 32K prompt costs ~50 s before the first output token. The aggregate rate
falls because the same 256 output tokens are spread over a longer request;
TTFT is the number that matters for interactive use.

## Long-context retrieval

Three planted `ACCESS CODE` values at 20%, 55% and 85% of a synthetic document;
a round fails if any code is missing from the answer.

| Label | Prompt | TTFT p50 | Prefill rate | Output tokens | Codes found | Latency p50 |
|---|---:|---:|---:|---:|---:|---:|
| `retrieval-32768-c1-cold` | 32,768 | 50.51 s | ~649 tok/s | 19–29 | 3/3 in all rounds | 54.2 s |
| `retrieval-100000-c1-cold` | 100,000 | 214.47 s | ~466 tok/s | 28–32 | 3/3 in all rounds | 219.9 s |
| `retrieval-200000-c1-cold` | 200,000 | 625.21 s | ~320 tok/s | 27–33 | 3/3 in all rounds | 632.0 s |

All nine rounds retrieved all three codes. Achieved prompt lengths were
32,766–32,768, 99,996–100,000 and 199,997–200,000 tokens.

TTFT grows **12.4×** from 32K to 200K while prompt length grows 6.1×, which is
the expected superlinear attention cost. A 200K prompt takes **10.4 minutes**
before the first token; prefill, not decode, is the practical limit for
long-context use on this host. At 200K the KV cache peaked at 29.4% of the pool,
consistent with one 200K sequence in a 679,872-token cache.

## Telemetry

From each report's `metrics.summary` and `memory` blocks. Gauges are sampled at
0.2 s, host memory at 1 s; short spikes can be missed.

| Label | Peak running | Peak waiting | Max KV usage | Min `MemAvailable` | Max swap |
|---|---:|---:|---:|---:|---:|
| `short-c1-cold` | 1 | 0 | 0.20% | 11.6 GiB | 1.49 GiB |
| `short-c2-cold` | 2 | 0 | 0.38% | 12.2 GiB | 2.38 GiB |
| `short-c4-cold` | 4 | 0 | 0.74% | 10.3 GiB | 0.93 GiB |
| `short-c8-cold` | 8 | 0 | 1.48% | 10.7 GiB | 1.37 GiB |
| `short-c1-warm` | 1 | 0 | 0.19% | 11.1 GiB | 1.57 GiB |
| `prefill-16384-c1` | 1 | 0 | 2.45% | 11.0 GiB | 2.47 GiB |
| `prefill-32768-c1` | 1 | 0 | 4.86% | 10.5 GiB | 2.36 GiB |
| `retrieval-32768-c1` | 1 | 0 | 4.83% | 10.5 GiB | 2.46 GiB |
| `retrieval-100000-c1` | 1 | 0 | 14.71% | 10.1 GiB | 1.74 GiB |
| `retrieval-200000-c1` | 1 | 0 | 29.41% | 10.0 GiB | 1.13 GiB |

No stage queued a request (`peak_waiting = 0`), no preemption was recorded, and
no OOM kill occurred. Host memory is the tight resource, not KV: the kernel
keeps ~10 GiB available and uses up to 2.5 GiB of swap while the weights and
page cache are resident.

## Interface smoke

`smoke-interface-2` passes chat, streaming, reasoning split, tool calls,
forced-output accounting, tokenization and prefix reuse. The Qwen-style parsers
work with this checkpoint's template: `--reasoning-parser qwen3` splits thinking
into `delta.reasoning`, and `--tool-call-parser qwen3_coder` produces structured
`tool_calls` when the request asks for `tool_call_format="xml"`.

## Excluded or failed runs

- `smoke-interface` (first attempt) failed two checks and is kept in `evidence/`.
  vLLM 0.32 returns reasoning in `delta.reasoning`, not `reasoning_content`, and
  only reports `usage.prompt_tokens_details` with `--enable-prompt-tokens-details`.
  Both were fixed and the clean rerun is `smoke-interface-2`.
- The first container start failed because vLLM 0.32 removed
  `--disable-log-requests`; the supported flag is `--no-enable-log-requests`.
  No benchmark ran against the failed container.

## Comparison with the previous workspace

The prior project served `nvidia/Qwen3.8-Flash-Next-FP4` (MoE, FP4, MTP) on the
same host. It reported 37.35 tok/s at C1 and 102.94 tok/s at C8 for the same
1K/256 workload. Nanbeige4.3-33B is a **dense** FP8 model and measures 6.59 tok/s
at C1 and 42.11 tok/s at C8: 5.7× slower per stream and 2.4× lower aggregate at
C8. That gap is architectural (all 33B parameters are read every token), not a
tuning defect. Nanbeige's advantage, if any, has to come from answer quality,
which this workspace does not measure.

## Known limits

- **Bandwidth-bound decode.** 33 GiB of FP8 weights at ~273 GB/s bounds batch-1
  decode near 8 tok/s. Measured 6.65 tok/s is 82% of that ceiling.
- **Prefill is the weak axis.** ~650 tok/s at 32K and ~320 tok/s at 200K. A 200K
  prompt costs >10 minutes to first token.
- **262K served context, not 512K.** The checkpoint supports 524,288 tokens, but
  one GB10 cannot hold a 512K KV cache plus weights under this memory plan.
- **Community FP8 conversion.** Quality is assumed close to BF16; it is not
  measured against BF16 here. Retrieval is synthetic in-context recall only.
- **Three-round screens.** Screening evidence, not stable tails; p95 is omitted
  below 30 samples. No sustained load, overload, fairness or coding-quality
  claim. `production_qualified` is false everywhere.
