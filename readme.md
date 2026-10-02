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
| Cost / 1k emails | $0.038 | $0.462 | ? | ? |

Signals comparison (half-B evaluation, same split as published):

| | Jev (pub.) | Haiku 4.5 (pub.) | Nemotron OFF | Nemotron ON |
|---|---|---|---|---|
| Best single signal B | 89.4% | 94.2% | ? | ? |
| Logistic B accuracy | 95.0% | 93.2% | ? | ? |
| Logistic AUROC B | 0.982 | 0.991 | ? | ? |

---

## Machine setup (start from scratch)

### Prerequisites

- Python 3.11+, `uv` package manager (`pip install uv` or `curl -LsSf https://astral.sh/uv/install.sh | sh`)
- Internet access to `https://integrate.api.nvidia.com`
- No GPU required (NIM is a remote API)

### Clone and install

```bash
git clone https://github.com/anisselbd/jev-phishing-bench.git
cd jev-phishing-bench
uv sync
uv add openai  # needed for NIM calls
```

### Apply script fixes (critical — not yet merged upstream)

The upstream repo is missing several bug fixes and new flags required for Nemotron runs.
Apply all of them before running anything:

**Fix 1 — `bench/common.py`: `done_ids()` must exclude format errors**

```python
# bench/common.py line ~79
# BEFORE:
return {r["id"] for r in read_jsonl(path) if r.get("ok")}
# AFTER:
return {r["id"] for r in read_jsonl(path) if r.get("ok") and "format_error" not in r}
```

**Fix 2 — `run_llm.py`: guard `response_format`, pin temperature, add `--suffix`**

In `build_body()`, replace the unconditional `response_format` line:

```python
# In the openai branch of build_body(), replace:
#   "response_format": {"type": "json_object"},
# with:
if not env("LLM_NO_RESPONSE_FORMAT"):
    body["response_format"] = {"type": "json_object"}

# After body.update(extra), add:
body["temperature"] = 0  # prevent LLM_EXTRA_BODY from overriding temperature
```

In `main()`, add after the existing `--concurrency` argument:

```python
parser.add_argument("--suffix", type=str, default="", help="tag appended to output filename")
```

Update the output path:

```python
suffix_part = f"_{args.suffix}" if args.suffix else ""
out = RAW_DIR / f"llm_{model}{suffix_part}_pass{args.pass_no}.jsonl"
```

**Fix 3 — `run_llm_signals.py`: add `--concurrency`, `--suffix`, fix `LLM_NO_RESPONSE_FORMAT`,
fix temperature, fix consecutive_failures counter, add threading**

Add imports at the top:

```python
import threading
from concurrent.futures import ThreadPoolExecutor
```

In `build_body()`:

```python
# Same as run_llm.py: replace unconditional response_format with:
if not env("LLM_NO_RESPONSE_FORMAT"):
    body["response_format"] = {"type": "json_object"}
# And after body.update(extra):
body["temperature"] = 0
```

In `main()`, add arguments:

```python
parser.add_argument("--concurrency", type=int, default=1)
parser.add_argument("--suffix", type=str, default="")
```

Update the output path:

```python
suffix_part = f"_{args.suffix}" if args.suffix else ""
out = RAW_DIR / f"llm_{model}_signals{suffix_part}_pass{args.pass_no}.jsonl"
```

Replace the sequential loop with a threaded worker pattern mirroring `run_llm.py`
(use `threading.Lock()`, `threading.Event()`, `ThreadPoolExecutor`).
The key bug: `consecutive_failures` was not incremented in the parse-exception branch — fix that too.

> All three fixes are already applied in the code on the current machine at `/home/ubuntu/jev-phishing-bench`.

### Download benchmark data

```bash
uv run prepare_data.py
```

Verify: `wc -l data/emails.jsonl` should print 2000.

### Create `.env`

```
TYPESAFE_API_KEY=
TYPESAFE_BASE_URL=https://api.typesafe.ai
TYPESAFE_MODEL=jev-latest

LLM_PROVIDER=openai
LLM_BASE_URL=https://integrate.api.nvidia.com/v1
LLM_API_KEY=<your-nvidia-nim-api-key>
LLM_MODEL=nvidia/nemotron-3.5-lightning-30b-a3b
LLM_PRICE_IN=0.20
LLM_PRICE_OUT=0.65
LLM_RPM=0
LLM_EXTRA_BODY={"chat_template_kwargs":{"enable_thinking":true},"reasoning_budget":16384}
LLM_NO_RESPONSE_FORMAT=1
LLM_GRID_MODEL=nvidia/nemotron-3.5-lightning-30b-a3b
```

### Create results directory

```bash
mkdir -p results/raw/llm_nvidia
```

---

## Running the 4 experiments

### Why --concurrency 8 is required

The NIM endpoint has a **bimodal latency distribution**: most requests return in ~1s but a tail of
requests takes 100+ seconds (NIM backend queueing). Running sequentially means one slow request
blocks 8+ fast ones. With `--concurrency 8`, slow requests don't stall the queue and total wall-clock
time drops from ~18 hours to ~1-2 hours for thinking-OFF runs. Per-email `latency_s` stored in JSONL
is unaffected by concurrency — it measures each request independently.

### The 4 run commands (chain in sequence)

```bash
cd /home/ubuntu/jev-phishing-bench

# RUN 1: Verdict, thinking OFF
LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' \
LLM_NO_RESPONSE_FORMAT='' \
uv run run_llm.py --concurrency 8

# RUN 2: Signals (5 questions), thinking OFF
LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' \
LLM_NO_RESPONSE_FORMAT='' \
uv run run_llm_signals.py --concurrency 8

# RUN 3: Verdict, thinking ON (.env already has enable_thinking:true and LLM_NO_RESPONSE_FORMAT=1)
uv run run_llm.py --suffix thinking --concurrency 8

# RUN 4: Signals, thinking ON
uv run run_llm_signals.py --suffix thinking --concurrency 8
```

### Single background command (recommended)

```bash
cd /home/ubuntu/jev-phishing-bench && \
echo "=== RUN 1: Verdict thinking OFF ===" && \
LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' LLM_NO_RESPONSE_FORMAT='' uv run run_llm.py --concurrency 8 && \
echo "=== RUN 2: Signals thinking OFF ===" && \
LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' LLM_NO_RESPONSE_FORMAT='' uv run run_llm_signals.py --concurrency 8 && \
echo "=== RUN 3: Verdict thinking ON ===" && \
uv run run_llm.py --suffix thinking --concurrency 8 && \
echo "=== RUN 4: Signals thinking ON ===" && \
uv run run_llm_signals.py --suffix thinking --concurrency 8 && \
echo "=== ALL DONE ==="
```

Scripts are resumable. If interrupted, re-run the same command — already-completed emails are skipped.

### Time estimates (at concurrency 8)

| Run | Per-email latency | 2000 emails wall-clock |
|---|---|---|
| Verdict thinking OFF | ~1-3s p50 (tail: 100s) | ~30-60 min |
| Signals thinking OFF | ~1-3s p50 | ~30-60 min |
| Verdict thinking ON | ~40-70s | ~2.5-4h |
| Signals thinking ON | ~40-70s | ~2.5-4h |

### Checking progress

```bash
wc -l results/raw/llm_nvidia/*.jsonl
```

Each line = one email processed. Need 2000 in each file for a complete run.

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

### Contaminated data backup

An early run used `LLM_EXTRA_BODY={}` to try to disable thinking — this does not work and collected
90 emails with thinking implicitly ON labeled as "thinking OFF". These are preserved as:

```
results/raw/llm_nvidia/nemotron-3.5-lightning-30b-a3b_pass1.contaminated_backup.jsonl
```

The main `_pass1.jsonl` was cleared and restarted with the correct flag.

---

## Output file structure

All raw output in `results/raw/llm_nvidia/`:

| File | Run |
|---|---|
| `nemotron-3.5-lightning-30b-a3b_pass1.jsonl` | Run 1: Verdict, thinking OFF |
| `nemotron-3.5-lightning-30b-a3b_signals_pass1.jsonl` | Run 2: Signals, thinking OFF |
| `nemotron-3.5-lightning-30b-a3b_thinking_pass1.jsonl` | Run 3: Verdict, thinking ON |
| `nemotron-3.5-lightning-30b-a3b_thinking_signals_pass1.jsonl` | Run 4: Signals, thinking ON |

Each JSONL line:

```json
{
  "id": "phish_0001",
  "y": 1,
  "ok": true,
  "click": 1,
  "phishing_probability": 0.95,
  "latency_s": 2.91,
  "usage": {"input_tokens": 349, "output_tokens": 17},
  "model": "nvidia/nemotron-3.5-lightning-30b-a3b"
}
```

Signals runs also include `sig_domain_mismatch`, `sig_free_hosting`, `sig_lure`, `sig_urgency`,
`sig_generic_sender` (each 0-1).

---

## Running analysis

After all 4 runs reach 2000 lines:

```bash
cd /home/ubuntu/jev-phishing-bench

# Thinking OFF
LLM_MODEL=nvidia/nemotron-3.5-lightning-30b-a3b uv run analyze.py

# Thinking ON
LLM_MODEL=nvidia/nemotron-3.5-lightning-30b-a3b uv run analyze.py --llm-suffix thinking
```

Outputs: `results/metrics.json`, `results/report.md`.

---

## Progress as of 2026-10-02

| Run | Status | Records done |
|---|---|---|
| Run 1: Verdict thinking OFF | In progress | ~780 / 2000 |
| Run 2: Signals thinking OFF | Not started | 0 / 2000 |
| Run 3: Verdict thinking ON | Not started | 0 / 2000 |
| Run 4: Signals thinking ON | Not started | 0 / 2000 |

Currently running on `/home/ubuntu/jev-phishing-bench` with concurrency 8 in a background process.

### What we learned today

- **NIM latency is bimodal**: p50 ~0.88s but p95 ~84s. A tail of requests takes 100+ seconds due
  to NIM backend queueing. Sequential runs are therefore ~32s/email average despite p50 being <1s.
  Always use `--concurrency 8`.
- **Three script bugs were present in the upstream repo** and are now fixed locally (see setup section).
- **`--suffix` and `--concurrency` flags for `run_llm_signals.py`** did not exist upstream — added locally.

---

## Quick reference

```bash
# Check progress
wc -l results/raw/llm_nvidia/*.jsonl

# Tail live output
tail -f results/raw/llm_nvidia/nemotron-3.5-lightning-30b-a3b_pass1.jsonl | \
  python3 -c "import sys,json; [print(r['id'], r.get('phishing_probability'), f\"{r.get('latency_s',0):.2f}s\") for r in map(json.loads, sys.stdin)]"

# Dry-run test (no output written)
LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' \
LLM_NO_RESPONSE_FORMAT='' \
uv run run_llm.py --limit 1 --dry-run
```
