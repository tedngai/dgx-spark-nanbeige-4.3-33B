# DGX Spark — Nanbeige4.3-33B on vLLM

An evidence-first serving workspace for **Nanbeige4.3-33B** on one NVIDIA DGX
Spark (GB10, 121 GiB unified memory). vLLM serves an FP8 checkpoint behind an
OpenAI-compatible API; every measurement is written to `evidence/` and rendered
into [`docs/RESULTS.md`](docs/RESULTS.md).

This is a screening workspace, not a production deployment. `production_qualified`
is false everywhere.

## Why FP8

Nanbeige4.3-33B is a **dense** 33B model. On GB10 the decode loop is bound by
unified-memory bandwidth (~273 GB/s), not by compute:

| Weights | Bytes read per token | Ideal decode, batch 1 |
|---|---:|---:|
| BF16 | ~62 GiB | ~4 tok/s |
| FP8 (this profile) | ~33 GiB | ~8 tok/s |
| NVFP4 | ~20 GiB | ~13 tok/s |

FP8 keeps 8-bit weights *and* 8-bit activations with dynamic per-token scaling,
which is close to BF16 quality while roughly doubling the decode ceiling. The
frozen checkpoint is `primitive-ai/Nanbeige4.3-33B-FP8`, a compressed-tensors
W8A8 conversion of `Nanbeige/Nanbeige4.3-33B`.

## Production recipe (measured)

| Setting | Value |
|---|---|
| Model | `Nanbeige/Nanbeige4.3-33B` @ `c1093c5…` |
| Served weights | `primitive-ai/Nanbeige4.3-33B-FP8` @ `0140c15…` (33.1 GiB) |
| Architecture | `LlamaForCausalLM`, 52 layers, 56 heads, 8 KV heads |
| Engine | vLLM 0.32.0, `vllm/vllm-openai:latest` pinned by digest |
| Parallelism | TP1, one GB10 |
| Served context | 262,144 tokens (checkpoint supports 524,288) |
| Concurrency | 16 sequences, 8,192-token prefill chunks |
| KV cache | FP8, prefix caching and chunked prefill on |
| Memory plan | `--gpu-memory-utilization 0.85` |
| Parsers | `qwen3` reasoning, `qwen3_coder` tool calls, auto tool choice |
| API | `http://127.0.0.1:8012/v1`, `/health`, `/metrics`, `/tokenize` |
| Auth | Bearer key from `secrets/api_key` (gitignored) |

`configs/production.json` is the frozen contract; `scripts/start.sh` assembles
the flags. `configs/workloads.json` is the screen matrix.

## Host prerequisites

- DGX Spark / GB10 with a working NVIDIA container runtime (`docker run --gpus all`).
- Docker access for the current user, or `sg docker` membership.
- ~120 GiB free disk for the weights and image.
- Python 3.12 for the harness (standard library only).

## Quick start

```bash
cp .env.example .env                 # optional overrides
scripts/download-model.sh            # 33 GiB, resumes, verifies every SHA-256
scripts/start.sh                     # serve and wait for /health
scripts/status.sh

python3 tests/validate_setup.py --expect-running --hash-model
python3 -B -m unittest discover -s tests -v
```

Then, with an idle service:

```bash
export NANBEIGE_API_KEY="$(cat secrets/api_key)"
python3 -B tests/smoke_test.py --label smoke-interface --execute --confirm-idle
scripts/run-screens.sh --execute --confirm-idle     # restart per screen
python3 tests/summarize.py                          # Markdown tables
```

`scripts/stop.sh` stops and preserves the container; `scripts/stop.sh --remove`
deletes it. Port 8012 is used so a Flare or other server on 8011 can coexist.

## What the harness measures

- **Decode screens** — 1,024-token prompts, 256 forced output tokens, C1/2/4/8.
  Aggregate tok/s is completion tokens divided by request elapsed time.
- **Prefill screens** — 16K and 32K prompts, TTFT and end-to-end rate.
- **Retrieval** — 32K / 100K / 200K documents with three planted `ACCESS CODE`
  values; a run fails if any code is missing from the answer.
- **Warm reuse** — identical prompts, requiring ≥90% `cached_tokens` on every
  request, which exercises vLLM's prefix cache.
- **Telemetry** — Prometheus counters/gauges sampled every 0.2 s and host
  `MemAvailable` every second. GB10 does not report VRAM through NVML.

## Benchmark results

See [`docs/RESULTS.md`](docs/RESULTS.md). `tests/summarize.py` renders every
attempt in `evidence/`, including failures.

## Contents

```
configs/     frozen profile and screen matrix
docs/        method, runbook, results
scripts/     download, start, stop, status, screen driver
tests/       client, prompts, benchmark, smoke, validation, unit tests
evidence/    append-only JSON reports
models/      downloaded weights (gitignored)
secrets/     API key (gitignored)
state/       model verification record (gitignored)
```

## Qualification boundary

Three-round screens are screening evidence, not stable tails. No sustained load,
overload, fairness, coding-quality or agentic benchmark is claimed. The
checkpoint is a community FP8 conversion, not a publisher-signed artifact, and
the base model card's benchmark table is not reproduced here.
