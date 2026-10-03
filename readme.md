# Jev vs LLM Compare — Nemotron 3.5 Lightning on PhishNChips v5.2

Extension of the published Jev vs Claude Haiku 4.5 phishing benchmark with a third baseline:
**NVIDIA Nemotron 3.5 Lightning 30B A3B**, run locally on an H100 80GB via NIM, tested in a 2x2
grid (thinking ON/OFF x verdict / 5 signals).

Benchmark code: https://github.com/anisselbd/jev-phishing-bench

---

## Repo contents

| File | Description |
|---|---|
| `readme.md` | This file — full setup, protocol, results, and findings |
| `nemotron_report.md` | Per-fold breakdown tables and comparison report |
| `NEMOTRON_RUNBOOK.md` | Original planning runbook (pre-run) |
| `nemotron_results_2026-10-02.tar.gz` | All 12 raw JSONL fold files + 12 `.meta.json` sidecars (8.7 MB) |

---

## Experiment design

### The 2x2 grid

| | Verdict (1 question) | 5 signal questions |
|---|---|---|
| **Thinking OFF** | Run 1 | Run 2 |
| **Thinking ON** | Run 3 | Run 4 |

- **Verdict**: single phishing/click question, same prompt as the Haiku baseline
- **Signals**: 5 noul questions identical to what Jev is asked (domain mismatch, free hosting, lure,
  urgency, generic sender). Score = mean of 5 signal probabilities, threshold 0.5
- **Thinking ON**: `enable_thinking: true, reasoning_budget: 16384` — model produces ~1100 tokens
  of reasoning before the JSON answer
- **Thinking OFF**: must explicitly send `enable_thinking: false` — the model defaults to ON if omitted
- **3-fold CV**: 2000 emails split into 3 non-overlapping folds (666+667+667) for variance estimates
- **Local H100**: all runs against `http://localhost:8000/v1`, zero network overhead, $0 cost

### Dataset

PhishNChips v5.2 — 2000 emails (1000 phishing, 1000 legitimate), Hugging Face `AreLit/PhishNChips`.
Same dataset used for the published Jev and Haiku baselines.

---

## Final results

### Headline table (format of the original benchmark)

Verdict task, 2000 emails. Jev and Haiku columns are the published numbers; Nemotron columns are
pooled across 3 folds on a local H100.

| Metric | Jev | Haiku 4.5 | Nemotron OFF | Nemotron ON |
|---|---|---|---|---|
| Accuracy | 62.6% | 81.3% | 64.3% | 77.5% |
| Recall on phishing | 43.2% | 76.4% | 30.2% | 58.5% |
| False positive rate | 18.0% | 13.8% | 1.6% | 3.5% |
| AUROC | 0.689 | 0.837 | 0.712 | 0.899 |
| ECE | 0.154 | 0.097 | 0.286 | 0.226 |
| Latency p50 | 239 ms | 687 ms | 419 ms | 3574 ms |
| Latency p95 | 331 ms | 980 ms | 491 ms | 6575 ms |
| Wall-clock, 2k emails | ~14 min | ~67 min | 28 s | 986 s |
| Cost per 1k emails | $0.038 | $0.462 | $0.00 | $0.00 |

Jev and Haiku latencies (p50, p95) were measured over the network from France; Nemotron was measured
against `localhost`. Nemotron p95 is computed from the raw per-request latencies in the results
tarball. Jev/Haiku p95 come from `results/report.md` in the original repo, which publishes no p90.

### Verdict task (pooled across 3 folds)

| Model | AUROC | Accuracy | Recall | FPR | ECE | Lat p50 | Out tok/req | Wall-clock 2k | Cost/1k |
|---|---|---|---|---|---|---|---|---|---|
| Jev (published) | 0.689 | 62.6% | 43.2% | 18.0% | 0.154 | 239 ms | — | ~14 min | $0.038 |
| Haiku 4.5 (published) | 0.837 | 81.3% | 76.4% | 13.8% | 0.097 | 687 ms | ~17 | ~67 min | $0.462 |
| Nemotron OFF (c=32) | 0.712 | 64.3% | 30.2% | 1.6% | 0.286 | 419 ms | 18 | 28s | $0.00 |
| **Nemotron ON (c=8)** | **0.899** | **77.5%** | **58.5%** | **3.5%** | 0.226 | 3574 ms | 1108 | 986s | $0.00 |

### Verdict — per-fold breakdown

**Thinking OFF (c=32, warmup=3)**

| Fold | n | Accuracy | Recall | FPR | AUROC | ECE | Lat p50 | Wall-clock |
|---|---|---|---|---|---|---|---|---|
| Fold 1 | 666 | 62.8% | 29.1% | 2.1% | 0.709 | 0.296 | 428 ms | 9s |
| Fold 2 | 667 | 66.1% | 32.7% | 1.8% | 0.715 | 0.268 | 419 ms | 10s |
| Fold 3 | 667 | 64.0% | 28.8% | 0.9% | 0.713 | 0.295 | 414 ms | 9s |
| **Mean ± std** | 2000 | 64.3% ±1.7 | 30.2% ±2.2 | 1.6% ±0.6 | 0.712 ±0.003 | 0.286 ±0.016 | 420 ms | 28s total |

**Thinking ON (c=8, warmup=2)**

| Fold | n | Accuracy | Recall | FPR | AUROC | ECE | Lat p50 | Wall-clock |
|---|---|---|---|---|---|---|---|---|
| Fold 1 | 666 | 77.6% | 60.9% | 4.9% | 0.888 | 0.227 | 3604 ms | 333s |
| Fold 2 | 667 | 78.6% | 58.7% | 2.4% | 0.908 | 0.216 | 3580 ms | 331s |
| Fold 3 | 667 | 76.3% | 55.9% | 3.3% | 0.902 | 0.235 | 3553 ms | 322s |
| **Mean ± std** | 2000 | 77.5% ±1.1 | 58.5% ±2.5 | 3.5% ±1.3 | 0.899 ±0.010 | 0.226 ±0.010 | 3579 ms | 986s total |

### Signals task (pooled across 3 folds)

*Jev and Haiku baselines use logistic regression on B-half 1000 emails — not directly comparable.*

| Model | AUROC | Accuracy | Recall | FPR | Lat p50 | Wall-clock 2k |
|---|---|---|---|---|---|---|
| Jev (published, logistic) | 0.982 | 95.0% | 97.0% | 7.0% | — | ~14 min |
| Haiku 4.5 (published, logistic) | 0.991 | 93.2% | 91.8% | 5.4% | — | ~67 min |
| Nemotron OFF (c=32) | 0.970 | 76.8% | 54.3% | 0.5% | 927 ms | 419s |
| Nemotron ON (c=8) | 0.929 | 73.0% | 48.5% | 2.6% | 10625 ms | 3691s |

### Signals — per-fold breakdown

**Thinking OFF (c=32, warmup=3)**

| Fold | n | Accuracy | Recall | FPR | AUROC | Lat p50 | Wall-clock |
|---|---|---|---|---|---|---|---|
| Fold 1 | 656 | 75.9% | 53.6% | 0.9% | 0.966 | 963 ms | 144s |
| Fold 2 | 665 | 78.2% | 56.3% | 0.6% | 0.970 | 904 ms | 134s |
| Fold 3 | 661 | 76.4% | 53.0% | 0.0% | 0.976 | 907 ms | 140s |
| **Mean ± std** | 1982 | 76.8% ±1.2 | 54.3% ±1.7 | 0.5% ±0.5 | 0.970 ±0.005 | 925 ms | 419s total |

**Thinking ON (c=8, warmup=2)**

| Fold | n | Accuracy | Recall | FPR | AUROC | Lat p50 | Wall-clock |
|---|---|---|---|---|---|---|---|
| Fold 1 | 665 | 73.1% | 49.3% | 2.1% | 0.933 | 10426 ms | 1225s |
| Fold 2 | 667 | 73.8% | 50.2% | 3.5% | 0.927 | 10688 ms | 1240s |
| Fold 3 | 666 | 72.1% | 46.2% | 2.1% | 0.928 | 10641 ms | 1227s |
| **Mean ± std** | 1998 | 73.0% ±0.9 | 48.5% ±2.0 | 2.6% ±0.8 | 0.929 ±0.003 | 10585 ms | 3691s total |

### Throughput (local H100)

| Run | Concurrency | Emails/s | Output tok/s | Wall-clock (2k emails) |
|---|---|---|---|---|
| Verdict OFF | 32 | 71.3 | 1301 | 28s (0.5 min) |
| Signals OFF | 32 | 4.8 | 238 | 419s (7.0 min) |
| Verdict ON | 8 | 2.0 | 2249 | 986s (16.4 min) |
| Signals ON | 8 | 0.5 | 2442 | 3691s (61.5 min) |

Signals OFF is slower than verdict OFF (4.8 vs 71 emails/s) because the 5-question prompt is ~5x
longer — more prefill tokens per request, less GPU batching efficiency. Output tok/s is similar
(5 floats vs 2 fields).

---

## Key findings

**Verdict task:**
- Thinking ON AUROC **0.899** beats Haiku 4.5 (0.837) — best of all models tested, at zero cost
- Thinking OFF AUROC 0.712 beats Jev (0.689) with near-zero FPR (1.6% vs 18.0%)
- Thinking improves AUROC by +0.187 and recall by +28pp at the cost of 8x latency and 61x output tokens
- Both modes poorly calibrated: ECE 0.286/0.226 vs Haiku 0.097 — probability scores less reliable than Haiku's

**Signals task:**
- Signals decomposition is highly effective: AUROC jumps from 0.712 (verdict OFF) to 0.970 (signals OFF)
- Signals OFF AUROC 0.970 is within 0.012 of Jev (0.982) and near-zero FPR (0.5%)
- Thinking ON **hurts** signals: 0.970 → 0.929 — extended reasoning adds noise to URL/domain feature checks
- Accuracy gap vs Jev/Haiku on signals is misleading: they use logistic regression on top of signals; Nemotron uses mean threshold

**Infrastructure:**
- Local H100 vs remote shared API: 72 rps vs ~0.03 rps (thinking OFF) — **>2000x faster**
- Verdict OFF 2k emails: 28s on local H100 vs ~18 hours estimated on remote integrate.api.nvidia.com
- Cost: $0 on local GPU vs $0.462/1k for Haiku

---

## Artifacts

### Raw results (in this repo)

`nemotron_results_2026-10-02.tar.gz` (8.7 MB) — all 12 JSONL fold files and 12 `.meta.json` sidecars:

```
nemotron-3.5-lightning_fold{1,2,3}_pass1.jsonl             Run 1: Verdict OFF
nemotron-3.5-lightning_fold{1,2,3}_pass1.meta.json
nemotron-3.5-lightning_signals_fold{1,2,3}_pass1.jsonl     Run 2: Signals OFF
nemotron-3.5-lightning_signals_fold{1,2,3}_pass1.meta.json
nemotron-3.5-lightning_thinking_fold{1,2,3}_pass1.jsonl    Run 3: Verdict ON
nemotron-3.5-lightning_thinking_fold{1,2,3}_pass1.meta.json
nemotron-3.5-lightning_signals_thinking_fold{1,2,3}_pass1.jsonl   Run 4: Signals ON
nemotron-3.5-lightning_signals_thinking_fold{1,2,3}_pass1.meta.json
```

To extract:
```bash
tar -xzf nemotron_results_2026-10-02.tar.gz
# extracts to results/raw/llm_nvidia/
```

### JSONL record format

Verdict records:
```json
{
  "id": "phish_0001",
  "y": 1,
  "ok": true,
  "answer": {"click": 0, "phishing_probability": 0.95},
  "latency_s": 0.41,
  "usage": {"input_tokens": 349, "output_tokens": 18},
  "model": "nvidia/nemotron-3.5-lightning",
  "concurrency": 32,
  "fold": 1,
  "num_folds": 3
}
```

Signals records have `"signals"` in place of `"answer"`:
```json
{
  "signals": {
    "sig_domain_mismatch": 0.92,
    "sig_free_hosting": 0.10,
    "sig_lure": 0.85,
    "sig_urgency": 0.71,
    "sig_generic_sender": 0.44
  }
}
```

### Meta sidecar format

```json
{
  "wall_clock_s": 9.2,
  "emails_per_sec": 72.4,
  "tokens_out_per_sec": 1344,
  "concurrency": 32,
  "warmup": 3,
  "fold": 1,
  "num_folds": 3,
  "ok": 667,
  "api_errors": 0,
  "format_errors": 0,
  "tok_in": 232145,
  "tok_out": 12006
}
```

---

## Reproducibility

### Hardware

- GPU: H100 80GB HBM3, dedicated on-machine
- OS: Linux 6.11.0 (nvidia kernel)
- All inference: local NIM container on the H100, `http://localhost:8000/v1`

### Prerequisites

- Python 3.11+, `uv` package manager
- Docker with NVIDIA Container Toolkit (`nvidia-docker2`)
- NGC API key (from `build.nvidia.com`) — needed once to pull the NIM image

### Setup

```bash
git clone https://github.com/anisselbd/jev-phishing-bench.git
cd jev-phishing-bench
uv sync
uv add openai
uv run prepare_data.py   # downloads PhishNChips v5.2 to data/emails.jsonl
```

### Deploy NIM on H100

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

until curl -s http://localhost:8000/v1/models | grep -q '"id"'; do sleep 5; done
```

The NIM reports its model ID as `nvidia/nemotron-3.5-lightning` (not the image tag).

### `.env`

```
LLM_PROVIDER=openai
LLM_BASE_URL=http://localhost:8000/v1
LLM_API_KEY=local
LLM_MODEL=nvidia/nemotron-3.5-lightning
LLM_PRICE_IN=0.0
LLM_PRICE_OUT=0.0
LLM_RPM=0
LLM_EXTRA_BODY={"chat_template_kwargs":{"enable_thinking":true},"reasoning_budget":16384}
LLM_NO_RESPONSE_FORMAT=1
```

Never commit `.env`.

### Run all 4 experiments

```bash
C_OFF=32
C_ON=8

# Run 1: Verdict OFF
for FOLD in 1 2 3; do
  LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' LLM_NO_RESPONSE_FORMAT='' \
  uv run run_llm.py --fold $FOLD --num-folds 3 --concurrency $C_OFF --warmup 3
done

# Run 2: Signals OFF
for FOLD in 1 2 3; do
  LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' LLM_NO_RESPONSE_FORMAT='' \
  uv run run_llm_signals.py --fold $FOLD --num-folds 3 --concurrency $C_OFF --warmup 3
done

# Run 3: Verdict ON
for FOLD in 1 2 3; do
  uv run run_llm.py --suffix thinking --fold $FOLD --num-folds 3 --concurrency $C_ON --warmup 2
done

# Run 4: Signals ON
for FOLD in 1 2 3; do
  uv run run_llm_signals.py --suffix thinking --fold $FOLD --num-folds 3 --concurrency $C_ON --warmup 2
done
```

Scripts are resumable — already-completed emails are skipped automatically.

---

## Critical configuration notes

**Thinking OFF requires an explicit flag** — the model defaults to ON if `enable_thinking` is omitted:
```
LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}'
```

**NIM rejects `response_format` when thinking is ON** — set `LLM_NO_RESPONSE_FORMAT=1` in `.env`
for thinking ON runs, clear it inline for thinking OFF:
```
LLM_NO_RESPONSE_FORMAT='' uv run run_llm.py ...
```

**Local NIM strips the opening `<think>` token** — response body is `[reasoning]</think>JSON` with
no opening tag. Fix applied in both `run_llm.py` and `run_llm_signals.py`:
```python
if "</think>" in cleaned:
    cleaned = cleaned.split("</think>", 1)[-1].strip()
elif "<think>" in cleaned:
    cleaned = re.sub(r"<think>.*?</think>", "", cleaned, flags=re.S).strip()
```
Without this fix, the JSON regex matches the template text inside the reasoning block and causes
100% format errors on thinking ON runs.

---

## Script changes vs upstream

All changes are in `jev-phishing-bench`, not yet upstreamed:

| File | Change |
|---|---|
| `bench/common.py` | `done_ids()` excludes format_error records (so they get retried); added `fold_emails()` for deterministic non-overlapping slices |
| `run_llm.py` | `--suffix`, `--fold K`, `--num-folds N`, `--warmup W`; HTTP/2 connection pool via `httpx.Limits`; sidecar `.meta.json`; `LLM_NO_RESPONSE_FORMAT` guard; `temperature=0` pinned after `body.update(extra)`; `</think>` parse fix |
| `run_llm_signals.py` | Same fold/warmup/httpx/sidecar/parse changes; imports `make_client` from `run_llm` |
| `net_floor.py` | `--concurrency C` for loaded concurrent probes |
| `run_concurrency_sweep.py` | New script: sweeps concurrency levels, measures throughput/latency, identifies GPU saturation point |
| `analyze.py` | `--folds N`, `--llm-suffix`; `load_llm_folds()` merges fold files; `fold_stats_summary()` computes mean ± std |

---

## Why remote API was abandoned

Original plan was `integrate.api.nvidia.com` (shared remote NIM):
- p50 ~0.88s but p95 ~84s — bimodal latency from NIM backend queueing
- 11/40 errors at c=8, 16/40 errors at c=16 — rate limits under concurrency
- Estimated 18-hour runtime for 2000 emails sequential
- H100 on the machine was idle the entire time

Decision: deploy NIM locally. Verdict OFF at c=32 ran 2000 emails in 28s — >2000x faster.

### Concurrency sweep result (thinking OFF, local NIM)

| Concurrency | RPS | p50 ms | p95 ms | tok_out/s |
|---|---|---|---|---|
| 1 | 14.6 | 68 | 71 | 266 |
| 2 | 19.3 | 103 | 106 | 353 |
| 4 | 25.1 | 144 | 234 | 458 |
| 8 | 13.2 | 162 | 1138 | 242 |
| 16 | 19.1 | 1345 | 1362 | 348 |
| **32** | **71.9** | **404** | **412** | **1313** |
| 64 | 73.2 | 395 | 406 | 1332 |

Saturation at c=32 — c=64 gives only +2%. The dip at c=8/c=16 is a NIM scheduler artifact
(prefill batching switching strategy). Used c=32 for all thinking OFF runs.

---

## Quick reference

```bash
# Check NIM is running
curl -s http://localhost:8000/v1/models | python3 -m json.tool

# GPU utilization during runs
nvidia-smi dmon -d 2

# Check record counts
wc -l results/raw/llm_nvidia/*.jsonl

# Dry-run (no output written)
LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' \
LLM_NO_RESPONSE_FORMAT='' \
uv run run_llm.py --limit 1 --dry-run

# Stop NIM
docker stop nim-nemotron
```
