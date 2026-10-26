# Runbook

## Lifecycle

```bash
scripts/download-model.sh          # resume/verify 33 GiB of FP8 weights
scripts/start.sh                   # create or start the container, wait for /health
scripts/status.sh                  # container state, model card, headline metrics
scripts/stop.sh                    # stop, preserve the container
scripts/stop.sh --remove           # stop and delete the container
```

`scripts/start.sh` is idempotent: an existing running container is reused; a
stopped one is started. If the container exists with stale flags, remove it and
start again.

## What start does

1. Resolves Docker access (direct, then `sg docker`, then passwordless sudo).
2. Checks the NVIDIA runtime and that `models/Nanbeige4.3-33B-FP8` has weights.
3. Generates `secrets/api_key` if absent (mode 600).
4. Refuses to start if port 8012 is held by something else.
5. Runs `vllm serve` with the frozen flags and waits up to 40 minutes for
   `/health`.

Expected startup on a warm page cache is 3–6 minutes for 33 GiB of FP8 weights
plus CUDA graph capture. The first start after a download can be slower.

## Screens

```bash
export NANBEIGE_API_KEY="$(cat secrets/api_key)"
scripts/run-screens.sh --execute --confirm-idle
ONLY=retrieval scripts/run-screens.sh --execute --confirm-idle
```

The driver stops and restarts the container before every stage so each screen
starts with an empty prefix cache and a fresh KV pool. Set
`RESTART_PER_SCREEN=0` only for debugging; the resulting warm-cache
contamination is not a valid baseline.

## Direct benchmark calls

```bash
python3 -B tests/benchmark.py --label short-c1-cold-20261026 \
  --mode decode --cache cold --concurrency 1 \
  --prompt-tokens 1024 --output-tokens 256 --rounds 3 \
  --execute --confirm-idle

python3 -B tests/benchmark.py --label retrieval-200000-c1-cold-20261026 \
  --mode retrieval --cache cold --concurrency 1 \
  --prompt-tokens 200000 --output-tokens 512 --rounds 3 \
  --execute --confirm-idle
```

Both scripts refuse to send inference without `--execute --confirm-idle`.

## Logs and evidence

```bash
docker logs --tail 100 -f vllm-nanbeige43-33b
ls -lt evidence/ | head
python3 tests/summarize.py
```

Reports are created exclusively and never edited. `state/model.json` records
the downloaded file hashes.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Container exits during startup | Memory plan too large or weights missing | `docker logs`, lower `GPU_MEMORY_UTILIZATION`, re-run `download-model.sh --verify` |
| `ValueError: ... max_model_len` | Request longer than the served context | Keep prompt + output ≤ 262,144 tokens |
| Tool calls arrive as plain text | Parser mismatch | The profile uses `qwen3_coder`; the template must use `tool_call_format="xml"` |
| No `reasoning_content` | Thinking disabled or parser mismatch | Omit `chat_template_kwargs.enable_thinking=false`; profile uses `qwen3` |
| `cached_tokens` stays 0 | Prefix caching disabled or prompts differ | Check `--enable-prefix-caching` and use an identical prompt set |
| `permission denied` on Docker | Login session lacks the group | Log in again, or let the scripts use `sg docker` |
| Port 8012 already bound | Another server is running | `scripts/stop.sh` or change `PORT` |

## Security notes

- The API is bound to `127.0.0.1` by default. Set `BIND_ADDR=0.0.0.0` only on a
  trusted network.
- `secrets/api_key` is generated locally and gitignored. vLLM receives it as
  `--api-key`, so it is visible in `docker inspect` to local users with Docker
  access. That is acceptable on a single-user Spark; do not treat it as a
  secret from the host's own administrators.
- The checkpoint is an unsigned community FP8 conversion. Verify the recorded
  SHA-256 values in `state/model.json` before trusting a new download.
