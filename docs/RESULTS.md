# Results and experiment history

## 2026-10-26 — workspace prepared

- Created this workspace for **Nanbeige4.3-33B** on one DGX Spark (GB10, SM121,
  121 GiB unified memory) with vLLM 0.32.0 (`torch 2.13.0+cu130`, capability
  `(12, 1)`).
- Froze the serving contract in `configs/production.json`: FP8 W8A8 weights,
  TP1, 262,144-token context, 16 sequences, 8,192-token prefill chunks, FP8 KV
  cache, prefix caching and chunked prefill on.
- Built the download/verify script, lifecycle scripts, harness, smoke test,
  setup validator and summarizer.
- **23 offline unit tests pass** (`python3 -B -m unittest discover -s tests -v`).
  They cover stream accounting, Prometheus parsing, token targeting, budget
  gates, evidence immutability and secret scrubbing; they send no inference.

No benchmark has been run at this point. Tables below are filled from
`evidence/` only after a screen completes.

## Measured screens

_Not yet measured._

## Telemetry and caveats

_Not yet measured._

## Known limits

- **Bandwidth-bound decode.** A 33B dense model at FP8 reads ~33 GiB per token,
  so batch-1 decode is bounded near 8 tok/s on GB10 regardless of engine.
  Batching raises aggregate throughput, not per-request latency.
- **262K served context, not 512K.** The checkpoint supports 524,288 tokens,
  but one GB10 cannot hold a 512K KV cache plus weights with this memory plan.
- **Community FP8 conversion.** Quality is assumed close to BF16, not measured
  against it in this workspace.
- **Three-round screens.** Screening evidence only; no sustained-load,
  overload, fairness or task-quality claim.
