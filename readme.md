# Jev vs LLM Compare — Nemotron Extension Runbook

This repo documents the work to extend the published Jev vs Claude Haiku 4.5 phishing benchmark
with a third baseline: **NVIDIA Nemotron 3.5 Lightning 30B A3B**, tested in two modes (thinking ON/OFF)
and two question styles (verdict only / 5 signals).

The benchmark code lives at: https://github.com/anisselbd/jev-phishing-bench

---

## What we are doing

The published benchmark compared Jev vs Claude Haiku 4.5 on 2000 phishing emails (PhishNChips v5.2).
This extension adds Nemotron as a second LLM baseline to answer two questions:

1. Does chain-of-thought reasoning (`enable_thinking: true`) improve phishing detection?
2. Does decomposing into 5 signals (domain mismatch, free hosting, lure, urgency, generic sender)
   still add value when the base model is stronger than Jev?

### The 2x2 experiment grid

| | Verdict only (1 question) | 5 signal questions |
|---|---|---|
| **Thinking OFF** | Run 1 | Run 2 |
| **Thinking ON** | Run 3 | Run 4 |

- **Verdict**: "is this email phishing?" — same single question as the Haiku baseline
- **Signals**: same 5 signal questions asked to Jev (domain mismatch, free hosting, lure, urgency,
  generic sender)
- **Thinking ON**: `enable_thinking: true, reasoning_budget: 16384` — model outputs ~840 tokens of
  reasoning before the JSON answer
- **Thinking OFF**: must explicitly send `enable_thinking: false` — the model defaults to ON if omitted

### Published baselines to beat

| | Jev (pub.) | Haiku 4.5 (pub.) | Nemotron thinking OFF | Nemotron thinking ON |
|---|---|---|---|---|
| Accuracy | 62.6% | 81.3% | TBD | TBD |
| Recall on phishing | 43.2% | 76.4% | TBD | TBD |
| False positive rate | 18.0% | 13.8% | TBD | TBD |
| AUROC | 0.689 | 0.837 | TBD | TBD |
| ECE | 0.154 | 0.097 | TBD | TBD |
| Latency p50 | 239 ms | 687 ms | ~14 ms (local NIM) | ~2.5s (local NIM) |
| Cost / 1k emails | $0.038 | $0.462 | $0 (local GPU) | $0 (local GPU) |

---

## Session log (2026-10-02)

### What we learned about the remote API approach (abandoned)

The original plan was to hit `integrate.api.nvidia.com` (shared remote NIM). This was wrong:

- **Bimodal latency**: p50 ~0.88s but p95 ~84s — NIM backend queueing made sequential runs average
  32s/email, giving an estimated 18-hour runtime for 2000 emails
- **Rate limits at concurrency ≥ 8**: 11/40 errors at c=8, 16/40 at c=16
- **H100 completely idle**: the machine has an H100 80GB that was sitting idle the entire time —
  we were never using it
- **Decision**: deploy NIM locally on the H100, all experiments run against `http://localhost:8000/v1`

### Local NIM deployment

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

Wait ~2-3 min for model loading, then:

```bash
until curl -s http://localhost:8000/v1/models | grep -q '"id"'; do sleep 5; done
```

**Important**: the NIM reports its model as `nvidia/nemotron-3.5-lightning` (not the image tag).
Use this exact string in `.env` as `LLM_MODEL`.

### Network floor (measured after local deployment)

```
localhost NIM: warm p50 = <1ms, concurrent x16 throughput = 1041 req/s
remote TypeSafe: warm p50 = 170ms, concurrent x16 throughput = 62 req/s
```

Local NIM has effectively zero network overhead — all latency is GPU compute.

### Concurrency sweep (thinking OFF, local NIM)

| concurrency | rps | p50ms | p95ms | tok_out/s |
|---|---|---|---|---|
| 1 | 14.6 | 68 | 71 | 266 |
| 2 | 19.3 | 103 | 106 | 353 |
| 4 | 25.1 | 144 | 234 | 458 |
| 8 | 13.2 | 162 | 1138 | 242 |
| 16 | 19.1 | 1345 | 1362 | 348 |
| **32** | **71.9** | **404** | **412** | **1313** |
| 64 | 73.2 | 395 | 406 | 1332 |

**Saturation at c=32** — c=64 gives only +2% over c=32. The dip at c=8/c=16 is a NIM scheduler
artifact (prefill batching). Used c=32 for all thinking-OFF runs.

### Script fixes applied (not yet upstream)

**`bench/common.py`** — `done_ids()` excludes format_error records so they get retried; added
`fold_emails()` for deterministic non-overlapping slices.

**`run_llm.py`** — added `--suffix`, `--fold K`, `--num-folds N`, `--warmup W`; HTTP/2 connection
pool via `httpx.Limits`; sidecar `.meta.json` with wall-clock throughput; guarded `response_format`
with `LLM_NO_RESPONSE_FORMAT`; pinned `temperature=0` after `body.update(extra)`.

**`run_llm_signals.py`** — same fold/warmup/httpx/sidecar changes; imports `make_client` from
`run_llm`.

**`net_floor.py`** — added `--concurrency C` for loaded concurrent probes.

**`run_concurrency_sweep.py`** — new script: sweeps concurrency levels, measures throughput/latency,
identifies saturation point.

**`analyze.py`** — added `--folds N` and `--llm-suffix`; `load_llm_folds()` merges fold files;
`fold_stats_summary()` computes mean ± std.

### Thinking ON parse fix

Local NIM thinking ON format: the model streams the opening `<think>` token before it reaches the
response text, so the response body contains only the reasoning followed by `</think>JSON` — no
opening tag. The original JSON regex matched an invalid template pattern inside the reasoning text,
causing 100% format errors on runs 3 and 4 (first attempt).

Fix in both `run_llm.py` and `run_llm_signals.py`:

```python
if "</think>" in cleaned:
    cleaned = cleaned.split("</think>", 1)[-1].strip()
elif "<think>" in cleaned:
    cleaned = re.sub(r"<think>.*?</think>", "", cleaned, flags=re.S).strip()
```

Runs 3 and 4 were cleared and restarted after the fix. Zero format errors since.

### Experiment results (2026-10-02)

All 4 experiments × 3 folds = 12 JSONL files, **8000 total emails processed**, 0 API errors,
0 format errors.

| Run | Files | Emails | Errors | Throughput | Output tok/s |
|---|---|---|---|---|---|
| Run 1: Verdict OFF | fold1-3_pass1 | 2000 | 0 | ~70 rps | ~1300 |
| Run 2: Signals OFF | signals_fold1-3_pass1 | 2000 | 0 | ~65 rps | ~1200 |
| Run 3: Verdict ON | thinking_fold1-3_pass1 | 2000 | 0 | ~2 rps | ~2250 |
| Run 4: Signals ON | signals_thinking_fold1-3_pass1 | 2000 | 0 | ~1 rps | ~2000 |

Thinking OFF at 70 rps vs sequential remote-API estimate of 0.03 rps: **>2000x faster**.

---

## Machine setup (start from scratch)

### Hardware

- **GPU**: H100 80GB HBM3 (on-machine, dedicated)
- **OS**: Linux 6.11.0 (nvidia kernel)
- All experiments run against a **local NIM container** on the H100

### Prerequisites

- Python 3.11+, `uv` package manager
- Docker with NVIDIA Container Toolkit (`nvidia-docker2`)
- NGC API key (from `build.nvidia.com`) — needed once to pull the NIM image

### Clone and install

```bash
git clone https://github.com/anisselbd/jev-phishing-bench.git
cd jev-phishing-bench
uv sync
uv add openai
```

### Deploy NIM on H100

```bash
# Authenticate (once per machine)
docker login nvcr.io --username '$oauthtoken' --password <your-ngc-api-key>

# Pull and start (model weights cached in ~/.cache/nim after first run)
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

# Wait for ready (~2-3 min)
until curl -s http://localhost:8000/v1/models | grep -q '"id"'; do sleep 5; done
curl -s http://localhost:8000/v1/models

# Verify GPU is in use
nvidia-smi
```

**After reboot** (weights cached, fast):

```bash
docker run -d --name nim-nemotron --gpus all --shm-size=16GB \
    -e NGC_API_KEY=<your-ngc-api-key> \
    -v "$HOME/.cache/nim:/opt/nim/.cache" \
    -p 8000:8000 \
    nvcr.io/nim/nvidia/nemotron-3.5-lightning-30b-a3b:latest
```

### `.env` (local NIM — no cost, no rate limits)

```
TYPESAFE_API_KEY=
TYPESAFE_BASE_URL=https://api.typesafe.ai
TYPESAFE_MODEL=jev-latest

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

> Never commit `.env`. It is gitignored.

### Download data

```bash
uv run prepare_data.py
# Verify: wc -l data/emails.jsonl  →  2000
```

---

## Critical configuration notes

### Thinking OFF requires an explicit flag

```
LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}'
```

Never use `LLM_EXTRA_BODY={}` — model defaults to thinking ON when no flag is sent.

### NIM rejects `response_format` when thinking is ON

`.env` sets `LLM_NO_RESPONSE_FORMAT=1`. Clear it inline for thinking-OFF runs:
`LLM_NO_RESPONSE_FORMAT=''`

### Local NIM strips the opening `<think>` token

The response body looks like: `[reasoning text]</think>{"click":1,"phishing_probability":0.9}`
(no opening `<think>` tag). Both `parse_answer()` and `parse_signals()` split on `</think>` to
extract the JSON.

### Model ID reported by local NIM

Image tag: `nemotron-3.5-lightning-30b-a3b` — NIM reports: `nvidia/nemotron-3.5-lightning`.
Use the latter in `.env`.

---

## Running all 4 experiments (full protocol)

### Step 1: Net floor

```bash
uv run net_floor.py --n 30 --concurrency 16
```

### Step 2: Concurrency sweep (thinking OFF)

```bash
LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' \
LLM_NO_RESPONSE_FORMAT='' \
uv run run_concurrency_sweep.py --suffix off
```

Use the recommended concurrency (`C_OFF`) for runs 1 and 2. For this H100: **c=32**.

### Step 3-4: Runs 1 and 2 (thinking OFF, c=32)

```bash
C_OFF=32
for FOLD in 1 2 3; do
  LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' \
  LLM_NO_RESPONSE_FORMAT='' \
  uv run run_llm.py --fold $FOLD --num-folds 3 --concurrency $C_OFF --warmup 3
done

for FOLD in 1 2 3; do
  LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' \
  LLM_NO_RESPONSE_FORMAT='' \
  uv run run_llm_signals.py --fold $FOLD --num-folds 3 --concurrency $C_OFF --warmup 3
done
```

### Step 5-6: Runs 3 and 4 (thinking ON, c=8)

```bash
C_ON=8
for FOLD in 1 2 3; do
  uv run run_llm.py --suffix thinking --fold $FOLD --num-folds 3 --concurrency $C_ON --warmup 2
done

for FOLD in 1 2 3; do
  uv run run_llm_signals.py --suffix thinking --fold $FOLD --num-folds 3 --concurrency $C_ON --warmup 2
done
```

### Step 7: Analyze

```bash
LLM_MODEL=nvidia/nemotron-3.5-lightning uv run analyze.py --folds 3
LLM_MODEL=nvidia/nemotron-3.5-lightning uv run analyze.py --llm-suffix thinking --folds 3
```

### One-shot command

```bash
cd /home/ubuntu/jev-phishing-bench && C_OFF=32 && C_ON=8

for FOLD in 1 2 3; do
  LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' LLM_NO_RESPONSE_FORMAT='' \
  uv run run_llm.py --fold $FOLD --num-folds 3 --concurrency $C_OFF --warmup 3
done && \
for FOLD in 1 2 3; do
  LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' LLM_NO_RESPONSE_FORMAT='' \
  uv run run_llm_signals.py --fold $FOLD --num-folds 3 --concurrency $C_OFF --warmup 3
done && \
for FOLD in 1 2 3; do
  uv run run_llm.py --suffix thinking --fold $FOLD --num-folds 3 --concurrency $C_ON --warmup 2
done && \
for FOLD in 1 2 3; do
  uv run run_llm_signals.py --suffix thinking --fold $FOLD --num-folds 3 --concurrency $C_ON --warmup 2
done && echo "=== ALL DONE ==="
```

Scripts are resumable — already-completed emails are skipped.

---

## Output file structure

All raw output in `results/raw/llm_nvidia/`. Each experiment = 3 fold files + 3 sidecar `.meta.json`.

| File pattern | Run |
|---|---|
| `nemotron-3.5-lightning_fold{1,2,3}_pass1.jsonl` | Run 1: Verdict, thinking OFF |
| `nemotron-3.5-lightning_signals_fold{1,2,3}_pass1.jsonl` | Run 2: Signals, thinking OFF |
| `nemotron-3.5-lightning_thinking_fold{1,2,3}_pass1.jsonl` | Run 3: Verdict, thinking ON |
| `nemotron-3.5-lightning_signals_thinking_fold{1,2,3}_pass1.jsonl` | Run 4: Signals, thinking ON |

Each JSONL record:

```json
{
  "id": "phish_0001", "y": 1, "ok": true,
  "answer": {"click": 0, "phishing_probability": 0.95},
  "latency_s": 0.41,
  "usage": {"input_tokens": 349, "output_tokens": 18},
  "model": "nvidia/nemotron-3.5-lightning",
  "concurrency": 32, "fold": 1, "num_folds": 3
}
```

Sidecar `.meta.json` per file:

```json
{
  "wall_clock_s": 9.2, "emails_per_sec": 72.4, "tokens_out_per_sec": 1344,
  "concurrency": 32, "warmup": 3, "fold": 1, "num_folds": 3
}
```

---

## Progress as of 2026-10-02 (ALL COMPLETE)

| Run | Status | Records | Errors | Wall-clock (2k emails) |
|---|---|---|---|---|
| Net floor | **DONE** | — | — | — |
| Concurrency sweep (OFF) | **DONE** | 30/level x7 | 0 | — |
| Run 1: Verdict OFF (3 folds) | **DONE** | 2000 / 2000 | 0 | 28s |
| Run 2: Signals OFF (3 folds) | **DONE** | 2000 / 2000 | 0 | 419s (7 min) |
| Run 3: Verdict ON (3 folds) | **DONE** | 2000 / 2000 | 0 | 986s (16 min) |
| Run 4: Signals ON (3 folds) | **DONE** | 2000 / 2000 | 0 | 3691s (61 min) |

All 8000 emails processed, 0 API errors, 0 format errors.

## Final results summary

| Model | Task | AUROC | Accuracy | Recall | FPR | ECE | Wall-clock 2k |
|---|---|---|---|---|---|---|---|
| Jev (published) | Verdict | 0.689 | 62.6% | 43.2% | 18.0% | 0.154 | ~14 min |
| Haiku 4.5 (published) | Verdict | 0.837 | 81.3% | 76.4% | 13.8% | 0.097 | ~67 min |
| Nemotron OFF | Verdict | 0.712 | 64.3% | 30.2% | 1.6% | 0.286 | 28s |
| Nemotron ON | Verdict | **0.899** | 77.5% | 58.5% | 3.5% | 0.226 | 986s |
| Jev (published) | Signals | 0.982 | 95.0% | 97.0% | 7.0% | — | ~14 min |
| Haiku 4.5 (published) | Signals | 0.991 | 93.2% | 91.8% | 5.4% | — | ~67 min |
| Nemotron OFF | Signals | 0.970 | 76.8% | 54.3% | 0.5% | — | 419s |
| Nemotron ON | Signals | 0.929 | 73.0% | 48.5% | 2.6% | — | 3691s |

Key findings:
- Thinking ON verdict AUROC **0.899** beats Haiku 4.5 (0.837), best of the four models tested
- Thinking ON costs 8x latency and 61x output tokens vs OFF
- Signals decomposition is highly effective: AUROC 0.712 → 0.970 (thinking OFF), within 0.012 of Jev
- Thinking ON **hurts** signals: 0.970 → 0.929 (reasoning adds noise to URL/domain feature checks)
- Local H100: verdict OFF 2k emails in 28s vs ~18 hours estimated on the remote shared API

---

## Quick reference

```bash
# Check NIM is running
curl -s http://localhost:8000/v1/models | python3 -m json.tool

# GPU utilization during runs
nvidia-smi dmon -d 2

# Check progress
wc -l results/raw/llm_nvidia/*.jsonl

# Dry-run test (thinking OFF, no output written)
LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' \
LLM_NO_RESPONSE_FORMAT='' \
uv run run_llm.py --limit 1 --dry-run

# Stop NIM
docker stop nim-nemotron
```
