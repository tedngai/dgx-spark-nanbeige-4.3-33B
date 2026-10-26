# Methodology

## Principles

1. **Evidence first.** Every inference run writes an append-only JSON report
   under `evidence/` with a UTC timestamp. Failed and partial runs stay.
2. **Capacity first.** Work completion, service health, token accounting and
   telemetry are separate observations. Queueing and low memory are reported,
   not hidden, unless `--policy no-queue` is selected.
3. **No inference without consent.** Both `--execute` and `--confirm-idle` are
   required by `tests/smoke_test.py` and `tests/benchmark.py`.
4. **Do not weaken a profile to pass.** `configs/production.json` is the frozen
   contract. A mismatch is recorded and fixed at the cause.

## Workloads

### Decode

1,024-token synthetic documents, 256 output tokens with `ignore_eos`, so every
sample generates exactly 256 tokens. Concurrency 1, 2, 4 and 8. Each round uses
a fresh prompt namespace, so no prompt prefix is shared between rounds.

### Prefill

16,384- and 32,768-token documents with 256 forced output tokens at C1. TTFT
isolates prefill cost; the aggregate rate falls as prefill grows because the
same output budget is spread over a longer request.

### Retrieval

A synthetic document with three planted `ACCESS CODE` values at 20%, 55% and
85% of its length, followed by an instruction to list every code. The run fails
if any code is missing. This is an in-context recall check, not document
reasoning quality.

### Warm reuse

The same prompt set is sent once as warmup, then measured. A round passes only
when every request reports `cached_tokens >= 90%` of its prompt tokens. This
exercises vLLM's automatic prefix cache, which is enabled in the profile.

## Definitions

- **TTFT** — time from request start to the first non-empty text delta, not the
  first SSE event. Reasoning deltas count as text.
- **Aggregate tok/s** — completion tokens for all requests in a round divided by
  the round's wall-clock elapsed time.
- **Decode tok/s (estimate)** — tokens between the first and last text delta
  divided by that span. It is an estimate because vLLM may emit several tokens
  per SSE delta.
- **Prompt tokens** — the tokenizer's count of the raw document; the served
  chat template adds a small preamble, visible in `usage.prompt_tokens`.

## Telemetry

| Signal | Source | Cadence |
|---|---|---|
| Running / waiting requests | `vllm:num_requests_running`, `vllm:num_requests_waiting` | 0.2 s |
| KV cache usage | `vllm:kv_cache_usage_perc` (or `gpu_cache_usage_perc`) | 0.2 s |
| Prefix cache hits/queries | `vllm:prefix_cache_hits_total`, `vllm:prefix_cache_queries_total` | 0.2 s |
| Token counters | `vllm:prompt_tokens_total`, `vllm:generation_tokens_total` | 0.2 s |
| Host memory | `/proc/meminfo` `MemAvailable`, swap | 1 s |

NVML does not expose GB10 VRAM, so no `nvidia-smi` memory number is used.
`MemAvailable` is host-wide and cannot attribute memory to vLLM, the page cache
or another process.

## Screens and restarts

`scripts/run-screens.sh` restarts the container before each stage. vLLM keeps
its prefix cache and KV pool across requests, so a restart is the only clean
baseline. Startup reloads 33 GiB of weights; that cost is excluded from every
measurement because the benchmark starts only after `/health` is ready.

## Statistics

Three rounds per screen. p50 is reported; p95 is omitted below 30 samples and
is not a stable tail even above it. Treat the numbers as screening evidence.
