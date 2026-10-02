# Jev vs LLM Compare — Nemotron Extension Runbook

This repo documents the ongoing work to extend the published Jev vs Claude Haiku 4.5 phishing benchmark
with a third baseline: **NVIDIA Nemotron 3.5 Lightning 30B A3B**, tested in two modes (thinking ON/OFF)
and two question styles (verdict only / 5 signals).

The actual benchmark code lives at: https://github.com/anisselbd/jev-phishing-bench

---

## What we are doing

The published benchmark (`jev-phishing-bench`) compared Jev vs Claude Haiku 4.5 on 2000 phishing emails.
This extension adds Nemotron as a second LLM baseline to answer two questions:

1. Does chain-of-thought reasoning (`enable_thinking: true`) improve phishing detection?
2. Does decomposing into 5 signals (domain mismatch, free hosting, lure, urgency, generic sender) still
   add value when the base model is stronger than Jev?

### The 2x2 experiment grid

| | Verdict only (1 question) | 5 signal questions |
|---|---|---|
| **Thinking OFF** | Run 1 | Run 2 |
| **Thinking ON** | Run 3 | Run 4 |

- **Verdict**: ask "is this email phishing?" — same single question used for the Haiku baseline
- **Signals**: ask the same 5 signal questions asked to Jev (domain mismatch, free hosting, lure, urgency,
  generic sender)
- **Thinking ON**: `enable_thinking: true, reasoning_budget: 16384`
- **Thinking OFF**: must explicitly send `enable_thinking: false` — the model defaults to ON if omitted

### Published baselines to beat

| | Jev (pub.) | Haiku 4.5 (pub.) | Nemotron thinking OFF | Nemotron thinking ON |
|---|---|---|---|---|
| Accuracy | 62.6% | 81.3% | ? | ? |
| Recall on phishing | 43.2% | 76.4% | ? | ? |
| False positive rate | 18.0% | 13.8% | ? | ? |
| AUROC | 0.689 | 0.837 | ? | ? |
| ECE | 0.154 | 0.097 | ? | ? |
| Latency p50 (sequential) | 239 ms | 687 ms | ? | ? |
| Cost / 1k emails | $0.038 | $0.462 | $0 (local) | $0 (local) |

Signals comparison (half-B evaluation, same split as published):

| | Jev (pub.) | Haiku 4.5 (pub.) | Nemotron OFF | Nemotron ON |
|---|---|---|---|---|
| Best single signal B | 89.4% | 94.2% | ? | ? |
| Logistic B accuracy | 95.0% | 93.2% | ? | ? |
| Logistic AUROC B | 0.982 | 0.991 | ? | ? |

---

## Machine setup (start from scratch)

### Hardware

- **GPU**: H100 80GB HBM3 (on-machine, dedicated)
- **OS**: Linux 6.11.0 (nvidia kernel)
- All experiments run against a **local NIM container** on the H100 — no remote API calls, no rate
  limits, no shared bandwidth.

### Prerequisites

- Python 3.11+, `uv` package manager (`pip install uv` or `curl -LsSf https://astral.sh/uv/install.sh | sh`)
- Docker with NVIDIA Container Toolkit (`nvidia-docker2`)
- An NGC API key (from `build.nvidia.com`) — needed only once to pull the NIM image

### Clone and install

```bash
git clone https://github.com/anisselbd/jev-phishing-bench.git
cd jev-phishing-bench
uv sync
uv add openai  # needed for NIM calls
```

### Deploy NIM locally on H100

This is the critical step. All experiments must hit the local NIM, not the remote shared API.

**Step 1 — Authenticate to NVIDIA's container registry (once per machine):**

```bash
docker login nvcr.io --username '$oauthtoken' --password <your-ngc-api-key>
```

**Step 2 — Pull and start the NIM container:**

```bash
export LOCAL_NIM_CACHE=~/.cache/nim
mkdir -p "$LOCAL_NIM_CACHE"

docker run -d \
    --name nim-nemotron \
    --gpus all \
    --shm-size=16GB \
    -e NGC_API_KEY=<your-ngc-api-key> \
    -v "$LOCAL_NIM_CACHE:/opt/nim/.cache" \
    -p 8000:8000 \
    nvcr.io/nim/nvidia/nemotron-3.5-lightning-30b-a3b:latest
```

**Step 3 — Wait for NIM to be healthy (~2-3 minutes):**

```bash
until curl -s http://localhost:8000/v1/models | grep -q '"id"'; do sleep 5; done
curl -s http://localhost:8000/v1/models
```

Expected output: `{"object":"list","data":[{"id":"nvidia/nemotron-3.5-lightning",...}]}`

> **Important**: The NIM reports its model ID as `nvidia/nemotron-3.5-lightning` (not the image tag
> `nemotron-3.5-lightning-30b-a3b`). Use this exact string in `.env` as `LLM_MODEL`.

**Step 4 — Check GPU is being used:**

```bash
nvidia-smi
# Should show the nim-nemotron process consuming GPU memory
```

**Check NIM logs if it fails to start:**

```bash
docker logs nim-nemotron --tail 50
```

**Restart NIM after machine reboot:**

```bash
# NIM container does NOT auto-restart — re-run Step 2 after any reboot
# Model weights are cached in ~/.cache/nim so the second pull is instant
docker run -d --name nim-nemotron --gpus all --shm-size=16GB \
    -e NGC_API_KEY=<your-ngc-api-key> \
    -v "$HOME/.cache/nim:/opt/nim/.cache" \
    -p 8000:8000 \
    nvcr.io/nim/nvidia/nemotron-3.5-lightning-30b-a3b:latest
```

### Download benchmark data

```bash
uv run prepare_data.py
```

Verify: `wc -l data/emails.jsonl` should print 2000.

### Create `.env` (local NIM — no API key needed for inference)

```
TYPESAFE_API_KEY=
TYPESAFE_BASE_URL=https://api.typesafe.ai
TYPESAFE_MODEL=jev-latest

# Local NIM on H100
LLM_PROVIDER=openai
LLM_BASE_URL=http://localhost:8000/v1
LLM_API_KEY=local
LLM_MODEL=nvidia/nemotron-3.5-lightning
LLM_PRICE_IN=0.0
LLM_PRICE_OUT=0.0
LLM_RPM=0
LLM_EXTRA_BODY={"chat_template_kwargs":{"enable_thinking":true},"reasoning_budget":16384}
LLM_NO_RESPONSE_FORMAT=1
LLM_GRID_MODEL=nvidia/nemotron-3.5-lightning
```

> **Never** commit `.env` to git. It is gitignored. The NGC API key used to pull the image
> should not appear in any script or commit.

### Create results directory

```bash
mkdir -p results/raw/llm_nvidia
```

---

## Apply script fixes (critical — not yet merged upstream)

The upstream repo is missing several bug fixes and new flags required for Nemotron runs.
Apply all of them before running anything:

**Fix 1 — `bench/common.py`: `done_ids()` must exclude format errors and add `fold_emails()`**

```python
# bench/common.py line ~79
# BEFORE:
return {r["id"] for r in read_jsonl(path) if r.get("ok")}
# AFTER:
return {r["id"] for r in read_jsonl(path) if r.get("ok") and "format_error" not in r}

# Add new function:
def fold_emails(emails: list, fold: int, num_folds: int) -> list:
    """Return the fold-th (1-indexed) non-overlapping slice."""
    size = len(emails)
    start = (fold - 1) * size // num_folds
    end = fold * size // num_folds
    return emails[start:end]
```

**Fix 2 — `run_llm.py`: guard `response_format`, pin temperature, add fold/warmup/httpx flags**

- Add `--suffix`, `--fold K`, `--num-folds N`, `--warmup W` arguments
- Add `make_client(concurrency)` using `httpx.Limits` with HTTP/2
- Guard `response_format` with `LLM_NO_RESPONSE_FORMAT` env var
- Pin `temperature=0` after `body.update(extra)` to prevent override
- Implement warmup: fire W real requests before `wall_clock_start` (not recorded)
- Write sidecar `.meta.json` at end with throughput stats

**Fix 3 — `run_llm_signals.py`: same fold/warmup/httpx/sidecar changes as `run_llm.py`**

> All fixes are already applied in `/home/ubuntu/jev-phishing-bench` (current machine).
> The new scripts `run_concurrency_sweep.py` and updated `net_floor.py` are also in place.

---

## Fair benchmark protocol

### Why local NIM changes everything

The previous approach hit `integrate.api.nvidia.com` (shared remote NIM):
- Bimodal latency: p50 ~0.88s but p95 ~84s from backend queueing
- Rate limits at concurrency ≥ 8: 11/40 errors at c=8, 16/40 at c=16
- Sequential runs averaged 32s/email — effectively 18 hours for 2000 emails
- H100 on-machine was sitting **completely idle**

With local NIM on H100:
- No rate limits, no shared bandwidth, no tail-latency spikes
- GPU is saturated by the benchmark workload directly
- Throughput limited only by model compute, not network
- Cost = $0 (electricity only)

### Three non-overlapping folds for statistical validity

Each experiment runs 3 non-overlapping folds of ~667 emails each (total = 2000). This gives:
- 3 independent accuracy estimates per experiment
- Mean ± std across folds (not just a single point estimate)
- Zero email overlap between folds (verified by seeded order index)

### Concurrency sweep to find GPU saturation

Before the main runs, sweep concurrency levels to find where throughput plateaus:

```bash
# Find where doubling concurrency gives <20% more throughput
uv run run_concurrency_sweep.py
```

Use the recommended concurrency for all main benchmark runs.

### Warmup requests

Fire a small number of real requests before timing starts to:
- Establish HTTP/2 keep-alive connections
- Prime the NIM JIT compilation cache on first load

Warmup requests use real API calls but results are discarded and not recorded.

---

## Running the full protocol (all 4 experiments × 3 folds)

### Step 1: Network floor (run once, ~2 min)

```bash
cd /home/ubuntu/jev-phishing-bench
uv run net_floor.py --n 50 --concurrency 32
```

### Step 2: Concurrency sweep — thinking OFF (run once, ~5-10 min)

```bash
LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' \
LLM_NO_RESPONSE_FORMAT='' \
uv run run_concurrency_sweep.py --suffix off
```

Check recommended concurrency in the output table. For local NIM, expect saturation at 16-64 depending
on GPU utilization. Use the recommended value (call it `C_OFF`) for runs 1 and 2.

### Step 3: Run 1 — Verdict, thinking OFF (3 folds)

```bash
for FOLD in 1 2 3; do
  LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' \
  LLM_NO_RESPONSE_FORMAT='' \
  uv run run_llm.py --fold $FOLD --num-folds 3 --concurrency $C_OFF --warmup 3
done
```

### Step 4: Run 2 — Signals, thinking OFF (3 folds)

```bash
for FOLD in 1 2 3; do
  LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' \
  LLM_NO_RESPONSE_FORMAT='' \
  uv run run_llm_signals.py --fold $FOLD --num-folds 3 --concurrency $C_OFF --warmup 3
done
```

### Step 5: Concurrency sweep — thinking ON (optional, may differ from OFF)

```bash
uv run run_concurrency_sweep.py --suffix on
# Uses .env defaults: enable_thinking=true, LLM_NO_RESPONSE_FORMAT=1
```

Use recommended concurrency `C_ON` for runs 3 and 4 (thinking is slower, so saturation point is lower).

### Step 6: Run 3 — Verdict, thinking ON (3 folds)

```bash
for FOLD in 1 2 3; do
  uv run run_llm.py --suffix thinking --fold $FOLD --num-folds 3 --concurrency $C_ON --warmup 2
done
```

### Step 7: Run 4 — Signals, thinking ON (3 folds)

```bash
for FOLD in 1 2 3; do
  uv run run_llm_signals.py --suffix thinking --fold $FOLD --num-folds 3 --concurrency $C_ON --warmup 2
done
```

### Step 8: Analyze

```bash
# Thinking OFF
LLM_MODEL=nvidia/nemotron-3.5-lightning uv run analyze.py --folds 3

# Thinking ON
LLM_MODEL=nvidia/nemotron-3.5-lightning uv run analyze.py --llm-suffix thinking --folds 3
```

### One-shot command (chain all 4 runs sequentially at C=32 placeholder — update C after sweep)

```bash
cd /home/ubuntu/jev-phishing-bench

C_OFF=32   # update after concurrency sweep
C_ON=16    # update after sweep with thinking ON

for FOLD in 1 2 3; do
  echo "=== RUN 1 fold $FOLD ===" && \
  LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' LLM_NO_RESPONSE_FORMAT='' \
  uv run run_llm.py --fold $FOLD --num-folds 3 --concurrency $C_OFF --warmup 3
done && \
for FOLD in 1 2 3; do
  echo "=== RUN 2 fold $FOLD ===" && \
  LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' LLM_NO_RESPONSE_FORMAT='' \
  uv run run_llm_signals.py --fold $FOLD --num-folds 3 --concurrency $C_OFF --warmup 3
done && \
for FOLD in 1 2 3; do
  echo "=== RUN 3 fold $FOLD ===" && \
  uv run run_llm.py --suffix thinking --fold $FOLD --num-folds 3 --concurrency $C_ON --warmup 2
done && \
for FOLD in 1 2 3; do
  echo "=== RUN 4 fold $FOLD ===" && \
  uv run run_llm_signals.py --suffix thinking --fold $FOLD --num-folds 3 --concurrency $C_ON --warmup 2
done && \
echo "=== ALL DONE ==="
```

Scripts are resumable. If interrupted, re-run the same command — already-completed emails are skipped.

---

## Critical configuration notes

### Thinking OFF requires an explicit flag

Never use `LLM_EXTRA_BODY={}` to disable thinking. The model defaults to thinking ON when no flag
is sent. Runs 1 and 2 must use:

```
LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}'
```

### NIM rejects `response_format: json_object` when thinking is ON

When thinking is enabled the NIM endpoint errors if `response_format` is sent. The `.env` already sets
`LLM_NO_RESPONSE_FORMAT=1`. For thinking-OFF runs, clear it inline (`LLM_NO_RESPONSE_FORMAT=''`).

### Model ID reported by local NIM

The image tag is `nemotron-3.5-lightning-30b-a3b` but the NIM reports its model as
`nvidia/nemotron-3.5-lightning`. Use the latter in `.env` as `LLM_MODEL` — it is what the API
accepts and what gets stored in JSONL output.

---

## Output file structure

All raw output in `results/raw/llm_nvidia/`:

| File | Run |
|---|---|
| `llm_nvidia/nemotron-3.5-lightning_fold1_pass1.jsonl` | Run 1 fold 1: Verdict, thinking OFF |
| `llm_nvidia/nemotron-3.5-lightning_fold2_pass1.jsonl` | Run 1 fold 2: Verdict, thinking OFF |
| `llm_nvidia/nemotron-3.5-lightning_fold3_pass1.jsonl` | Run 1 fold 3: Verdict, thinking OFF |
| `llm_nvidia/nemotron-3.5-lightning_signals_fold1_pass1.jsonl` | Run 2 fold 1: Signals, thinking OFF |
| ... (same pattern) | ... |
| `llm_nvidia/nemotron-3.5-lightning_thinking_fold1_pass1.jsonl` | Run 3 fold 1: Verdict, thinking ON |
| `llm_nvidia/nemotron-3.5-lightning_thinking_signals_fold1_pass1.jsonl` | Run 4 fold 1: Signals, ON |

Each JSONL line:

```json
{
  "id": "phish_0001",
  "y": 1,
  "ok": true,
  "click": 1,
  "phishing_probability": 0.95,
  "latency_s": 0.31,
  "usage": {"input_tokens": 349, "output_tokens": 17},
  "model": "nvidia/nemotron-3.5-lightning",
  "concurrency": 32,
  "fold": 1,
  "num_folds": 3
}
```

Each JSONL file has a sidecar `.meta.json` with wall-clock throughput:

```json
{
  "wall_clock_s": 42.1,
  "emails_per_sec": 15.85,
  "tokens_out_per_sec": 1240.3,
  "concurrency": 32,
  "warmup": 3,
  "fold": 1,
  "num_folds": 3
}
```

---

## Running analysis

After all runs complete:

```bash
cd /home/ubuntu/jev-phishing-bench

# Thinking OFF (merges fold1+fold2+fold3, computes mean ± std)
LLM_MODEL=nvidia/nemotron-3.5-lightning uv run analyze.py --folds 3

# Thinking ON
LLM_MODEL=nvidia/nemotron-3.5-lightning uv run analyze.py --llm-suffix thinking --folds 3
```

Outputs: `results/metrics.json`, `results/report.md`.

---

## Progress as of 2026-10-02

| Run | Status | Records done |
|---|---|---|
| Setup: local NIM on H100 | **DONE** | Container healthy at localhost:8000 |
| Step 1: Net floor | Not started | — |
| Step 2: Concurrency sweep (thinking OFF) | Not started | — |
| Run 1: Verdict thinking OFF (3 folds) | Not started | 0 / 2000 |
| Run 2: Signals thinking OFF (3 folds) | Not started | 0 / 2000 |
| Step 5: Concurrency sweep (thinking ON) | Not started | — |
| Run 3: Verdict thinking ON (3 folds) | Not started | 0 / 2000 |
| Run 4: Signals thinking ON (3 folds) | Not started | 0 / 2000 |

**All previous runs against the remote API were cleared.** Starting fresh on local NIM.

### What changed from the previous approach

| Before | After |
|---|---|
| Remote shared API (`integrate.api.nvidia.com`) | Local NIM on H100 80GB |
| Bimodal latency (p50 0.88s, p95 84s) | GPU-bound latency only |
| Rate limits at concurrency ≥ 8 | No rate limits |
| Sequential ~32s/email average | Expected <1s/email at saturation |
| H100 completely idle | H100 fully utilized |
| Single pass, no variance estimate | 3 folds, mean ± std |
| No warmup | Warmup + HTTP/2 connection pool |
| No GPU saturation measurement | Concurrency sweep before main runs |

---

## Quick reference

```bash
# Check NIM is running
curl -s http://localhost:8000/v1/models | python3 -m json.tool

# Check GPU utilization during runs
nvidia-smi dmon -d 2

# Check progress
wc -l results/raw/llm_nvidia/*.jsonl

# Restart NIM after reboot (weights cached, fast)
docker run -d --name nim-nemotron --gpus all --shm-size=16GB \
    -e NGC_API_KEY=<your-ngc-api-key> \
    -v "$HOME/.cache/nim:/opt/nim/.cache" \
    -p 8000:8000 \
    nvcr.io/nim/nvidia/nemotron-3.5-lightning-30b-a3b:latest

# Stop NIM
docker stop nim-nemotron

# Dry-run test (no output written)
LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' \
LLM_NO_RESPONSE_FORMAT='' \
uv run run_llm.py --limit 1 --dry-run
```
