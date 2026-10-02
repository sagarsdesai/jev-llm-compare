# Nemotron Extension Runbook

This document describes exactly how to pick up the Nemotron benchmarking work on a new machine. It covers setup, configuration, the 4 test runs, bug fixes already applied in the codebase, progress tracking, analysis, and how to read results.

---

## 1. What We Are Doing

The main benchmark (`README.md`) compared **Jev** vs **Claude Haiku 4.5**. This extension adds a second LLM baseline: **NVIDIA Nemotron 3.5 Lightning 30B A3B**, tested in two modes and across two question styles.

### The 2x2 comparison grid

| | Verdict only (1 question) | 5 signal questions |
|---|---|---|
| **Thinking OFF** | Run 1 | Run 2 |
| **Thinking ON** | Run 3 | Run 4 |

- **Verdict**: ask "is this email phishing?" — same single question used for the Haiku baseline
- **Signals**: ask the same 5 signal questions asked to Jev (domain mismatch, free hosting, lure, urgency, generic sender)
- **Thinking ON**: `enable_thinking: true, reasoning_budget: 16384` — model outputs a reasoning trace before answering (~37-67s per email, ~1000-1400 output tokens)
- **Thinking OFF**: must explicitly send `enable_thinking: false` — the model defaults to thinking ON when no flag is sent

**Why this matters**: we want to know whether reasoning helps on phishing detection, and whether the 5-signal decomposition still adds value when the base model is much stronger than Jev.

### Comparison to published baselines

After all 4 runs complete we fill in this table:

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

## 2. Machine Setup (from scratch)

### Prerequisites

- Python 3.11+, `uv` package manager
- Internet access to `https://integrate.api.nvidia.com`
- No GPU required (NIM is a remote API)

### Clone and install

```bash
git clone https://github.com/anisselbd/jev-phishing-bench.git
cd jev-phishing-bench
uv sync
```

### Download benchmark data

```bash
uv run prepare_data.py
```

This downloads PhishNChips v5.2 (2000 emails) into `data/`. Verify: `wc -l data/emails.jsonl` should print 2000.

### Create `.env`

Create the file `/home/ubuntu/jev-phishing-bench/.env` (never commit this):

```
# TypeSafe (Jev) - leave blank, not used for Nemotron runs
TYPESAFE_API_KEY=
TYPESAFE_BASE_URL=https://api.typesafe.ai
TYPESAFE_MODEL=jev-latest

# NVIDIA NIM endpoint
LLM_PROVIDER=openai
LLM_BASE_URL=https://integrate.api.nvidia.com/v1
LLM_API_KEY=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
LLM_MODEL=nvidia/nemotron-3.5-lightning-30b-a3b
LLM_PRICE_IN=0.20
LLM_PRICE_OUT=0.65
LLM_RPM=0
# leave these two as-is; individual run commands override them inline
LLM_EXTRA_BODY={"chat_template_kwargs":{"enable_thinking":true},"reasoning_budget":16384}
LLM_NO_RESPONSE_FORMAT=1
LLM_GRID_MODEL=nvidia/nemotron-3.5-lightning-30b-a3b
```

**API key note**: the key above is the NVIDIA NIM API key. It provides access to the `nvidia/nemotron-3.5-lightning-30b-a3b` model at `https://integrate.api.nvidia.com/v1`.

### Copy existing progress files (from the previous machine)

The previous machine already made progress on all 4 runs. If you have the files, copy them into place:

```
results/raw/llm_nvidia/nemotron-3.5-lightning-30b-a3b_pass1.jsonl           (verdict thinking OFF)
results/raw/llm_nvidia/nemotron-3.5-lightning-30b-a3b_signals_pass1.jsonl   (signals thinking OFF)
results/raw/llm_nvidia/nemotron-3.5-lightning-30b-a3b_thinking_pass1.jsonl  (verdict thinking ON)
results/raw/llm_nvidia/nemotron-3.5-lightning-30b-a3b_thinking_signals_pass1.jsonl (signals thinking ON)
```

Progress as of 2026-10-01:
- Verdict thinking OFF: ~99 emails done (out of 2000)
- Signals thinking OFF: ~5 emails done
- Verdict thinking ON: ~29 emails done
- Signals thinking ON: ~11 emails done

The scripts are resumable: they check which email IDs are already present and skip them. Just run the same command again.

---

## 3. The 4 Run Commands

All 4 runs can be chained in sequence with a single background command. `--concurrency 8` fires 8 parallel API calls at a time for throughput; each email still gets its own isolated request.

```bash
cd /home/ubuntu/jev-phishing-bench

# RUN 1: Verdict, thinking OFF (2000 emails)
LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' \
LLM_NO_RESPONSE_FORMAT='' \
uv run run_llm.py --concurrency 8

# RUN 2: Signals (5 questions), thinking OFF (2000 emails)
LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' \
LLM_NO_RESPONSE_FORMAT='' \
uv run run_llm_signals.py --concurrency 8

# RUN 3: Verdict, thinking ON (2000 emails)
# .env already has enable_thinking:true and LLM_NO_RESPONSE_FORMAT=1
uv run run_llm.py --suffix thinking --concurrency 8

# RUN 4: Signals (5 questions), thinking ON (2000 emails)
uv run run_llm_signals.py --suffix thinking --concurrency 8
```

**As a single chained background command:**

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

### Time estimates (at concurrency 8)

| Run | Per-email latency | 2000 emails wall-clock |
|---|---|---|
| Verdict thinking OFF | ~2-4 s | ~10-20 min |
| Signals thinking OFF | ~3-5 s | ~12-20 min |
| Verdict thinking ON | ~40-70 s | ~2.5-4 h |
| Signals thinking ON | ~40-70 s | ~2.5-4 h |

Total: roughly 6-9 hours for all 4 runs.

### Checking progress

```bash
# count completed emails in each file
wc -l results/raw/llm_nvidia/nemotron-3.5-lightning-30b-a3b_pass1.jsonl
wc -l results/raw/llm_nvidia/nemotron-3.5-lightning-30b-a3b_signals_pass1.jsonl
wc -l results/raw/llm_nvidia/nemotron-3.5-lightning-30b-a3b_thinking_pass1.jsonl
wc -l results/raw/llm_nvidia/nemotron-3.5-lightning-30b-a3b_thinking_signals_pass1.jsonl
```

Each line = one email processed. You need 2000 in each file for the full run.

---

## 4. Critical Configuration Notes

### Thinking OFF requires an explicit flag

**Do not** use `LLM_EXTRA_BODY={}` to disable thinking. The Nemotron model on NIM defaults to thinking ON when no thinking parameter is sent. You must explicitly pass:

```
LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}'
```

This is what Runs 1 and 2 do (passed as inline env vars to override `.env`).

### NIM rejects `response_format: json_object` when thinking is ON

When thinking is enabled, the NIM endpoint returns an error if the request includes `"response_format": {"type": "json_object"}`. The fix is `LLM_NO_RESPONSE_FORMAT=1` in `.env`, which is already set. For thinking-OFF runs we clear it (`LLM_NO_RESPONSE_FORMAT=''`) because it is safe and helps enforce JSON output.

### Model on NIM

NIM serves `nvidia/nemotron-3.5-lightning-30b-a3b` with:
- NVFP4 weights (the checkpoint is NVFP4, not BF16)
- FP8 KV cache by default

There is no separate FP8 model ID. This is the only model ID available on NIM for this checkpoint.

---

## 5. Output File Structure

All raw output lives in `results/raw/llm_nvidia/`:

| File | Run | Description |
|---|---|---|
| `nemotron-3.5-lightning-30b-a3b_pass1.jsonl` | Run 1 | Verdict, thinking OFF |
| `nemotron-3.5-lightning-30b-a3b_signals_pass1.jsonl` | Run 2 | Signals, thinking OFF |
| `nemotron-3.5-lightning-30b-a3b_thinking_pass1.jsonl` | Run 3 | Verdict, thinking ON |
| `nemotron-3.5-lightning-30b-a3b_thinking_signals_pass1.jsonl` | Run 4 | Signals, thinking ON |
| `nemotron-3.5-lightning-30b-a3b_pass1.contaminated_backup.jsonl` | backup | 90 emails run while model was implicitly thinking-ON, kept for reference |

Each line in the JSONL files is a JSON record:

```json
{
  "id": "phish_0001",
  "y": 1,
  "ok": true,
  "click": 1,
  "phishing_probability": 0.95,
  "latency_s": 2.91,
  "usage": {"input_tokens": 349, "output_tokens": 17},
  "model": "nvidia/nemotron-3.5-lightning-30b-a3b",
  "timestamp": "2026-10-01T00:00:00Z"
}
```

For signals runs, there are also `sig_domain_mismatch`, `sig_free_hosting`, `sig_lure`, `sig_urgency`, `sig_generic_sender` keys (each a float 0-1).

---

## 6. Bug Fixes Applied to the Codebase

These bugs were found and fixed during this project. They are already in the code — do not reapply.

### Fix 1 (Critical): `run_llm_signals.py` ignored `LLM_NO_RESPONSE_FORMAT`

`run_llm.py` had a guard `if not env("LLM_NO_RESPONSE_FORMAT"): body["response_format"] = ...` but `run_llm_signals.py` always sent `response_format: json_object`. On NIM with thinking ON, this caused 100% API failures. Fixed by adding the same guard.

### Fix 2: Temperature silently overridden by `LLM_EXTRA_BODY`

`body.update(extra)` runs after setting `temperature: 0`, so anything in `LLM_EXTRA_BODY` could silently override temperature. Fixed by adding `body["temperature"] = 0` after `body.update(extra)` in both scripts.

### Fix 3: Format-error emails permanently abandoned on resume

`bench/common.py:done_ids()` returned email IDs even if they had a `format_error`. Those emails would never be retried. Fixed to exclude emails where `format_error` is present, so they are re-attempted on every resume:

```python
return {r["id"] for r in read_jsonl(path) if r.get("ok") and "format_error" not in r}
```

### Fix 4: Signals script did not trigger cascade-failure guard on parse errors

The consecutive-failure counter `failures` was not incremented in the parse-exception branch of `run_llm_signals.py`. Fixed to mirror `run_llm.py`.

### New: `--suffix` flag

Both `run_llm.py` and `run_llm_signals.py` now accept `--suffix <tag>`. This appends a tag to the output filename:

- `--suffix thinking` → `llm_<model>_thinking_pass1.jsonl`
- (no suffix) → `llm_<model>_pass1.jsonl`

### New: `--llm-suffix` flag in `analyze.py`

`analyze.py` now accepts `--llm-suffix <tag>` to load the suffixed output files:

```bash
LLM_MODEL=nvidia/nemotron-3.5-lightning-30b-a3b uv run analyze.py --llm-suffix thinking
```

### New: `--concurrency` flag in `run_llm_signals.py`

Added to match the existing flag in `run_llm.py`. Use `--concurrency 8` for throughput.

---

## 7. Running the Analysis

After all 4 runs reach 2000 lines each, run `analyze.py` twice:

```bash
cd /home/ubuntu/jev-phishing-bench

# Analyze thinking OFF runs (verdict + signals)
LLM_MODEL=nvidia/nemotron-3.5-lightning-30b-a3b uv run analyze.py

# Analyze thinking ON runs (verdict + signals)
LLM_MODEL=nvidia/nemotron-3.5-lightning-30b-a3b uv run analyze.py --llm-suffix thinking
```

Each invocation reads:
- `results/raw/llm_nvidia/nemotron-3.5-lightning-30b-a3b_pass1.jsonl` (or `_thinking_pass1.jsonl`)
- `results/raw/llm_nvidia/nemotron-3.5-lightning-30b-a3b_signals_pass1.jsonl` (or `_thinking_signals_pass1.jsonl`)

Outputs: `results/metrics.json`, `results/report.md`.

---

## 8. Latency Reporting Methodology

**Per-email `latency_s`** stored in each record measures the wall-clock time of that single HTTP request. This is unaffected by concurrency settings — each request is independent.

**Total wall-clock time** for 2000 emails = (2000 / concurrency) × avg_latency_per_email. Running with `--concurrency 8` is ~8x faster in total, but does not change the per-email latency.

**What to report:**

| Metric | Source | Concurrency effect |
|---|---|---|
| Accuracy, Recall, AUROC, ECE | `latency_s` values in JSONL | None — correct regardless |
| Cost | `usage` token counts | None |
| **Latency p50/p95** | `latency_s` distribution | Slight: 8 parallel requests may compete on NIM backend |
| Total run time | wall-clock | Divided by concurrency |

For a fair latency comparison with the published Haiku 4.5 numbers (p50 687 ms, measured sequentially), run a small sequential latency test after the main run:

```bash
# 200-email sequential subset for clean latency measurement
LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' \
LLM_NO_RESPONSE_FORMAT='' \
uv run run_llm.py --pass 2 --sample 200 --concurrency 1
```

The `--pass 2` flag writes to `_pass2.jsonl`, separate from the main `_pass1.jsonl` accuracy data. Report:
- Accuracy numbers: from `_pass1.jsonl` (concurrency 8, 2000 emails)
- Latency p50/p95: from `_pass2.jsonl` (concurrency 1, 200 emails)

In the comparison table, note: "Latency measured at concurrency=1, 200 emails. Throughput run used concurrency=8."

---

## 9. Contaminated Data Note

On the first run attempt, `LLM_EXTRA_BODY={}` was used to try to disable thinking. This does NOT work — the model defaults to thinking ON when no flag is present. 90 emails were collected with thinking implicitly ON but labeled as "thinking OFF". These are preserved as:

```
results/raw/llm_nvidia/nemotron-3.5-lightning-30b-a3b_pass1.contaminated_backup.jsonl
```

The main `_pass1.jsonl` was cleared and restarted from scratch with the correct `enable_thinking: false` flag.

---

## 10. Repository Structure

```
jev-phishing-bench/
├── .env                          # API keys (never commit)
├── README.md                     # Published benchmark results (Jev vs Haiku)
├── NEMOTRON_RUNBOOK.md           # This file
├── CLAUDE.md                     # Project constraints and design decisions (French)
├── run_llm.py                    # LLM verdict runner (1 question per email)
├── run_llm_signals.py            # LLM signals runner (5 questions per email)
├── run_jev.py                    # Jev runner
├── analyze.py                    # Metrics computation, report generation
├── prepare_data.py               # Download and verify benchmark data
├── charts.py / charts_simple.py  # Chart generation
├── net_floor.py                  # Network latency measurement
├── bench/
│   ├── common.py                 # Shared utilities (done_ids, load_emails, etc.)
│   ├── heuristics.py             # Non-AI regex baseline
│   └── protocol.py               # Half-A/Half-B split logic
├── data/                         # Downloaded emails (not in git)
│   └── emails.jsonl              # 2000 emails, seeded order
└── results/
    ├── raw/
    │   └── llm_nvidia/           # All Nemotron output files
    └── report.md                 # Generated after analyze.py
```

---

## 11. Quick Reference Commands

```bash
# Check run progress
wc -l results/raw/llm_nvidia/*.jsonl

# Tail live output of a background run
tail -f results/raw/llm_nvidia/nemotron-3.5-lightning-30b-a3b_pass1.jsonl | python3 -c "
import sys, json
for line in sys.stdin:
    r = json.loads(line)
    print(r['id'], r.get('click'), r.get('phishing_probability'), r.get('latency_s'))
"

# Test a single email (dry run, no output written)
LLM_EXTRA_BODY='{"chat_template_kwargs":{"enable_thinking":false}}' \
LLM_NO_RESPONSE_FORMAT='' \
uv run run_llm.py --limit 1 --dry-run

# Analyze thinking OFF
LLM_MODEL=nvidia/nemotron-3.5-lightning-30b-a3b uv run analyze.py

# Analyze thinking ON
LLM_MODEL=nvidia/nemotron-3.5-lightning-30b-a3b uv run analyze.py --llm-suffix thinking
```

---

## 12. Known Issues and Open Questions

1. **Format errors** (~1-2% of emails): the thinking-ON model occasionally produces malformed JSON. The `done_ids` fix (Fix 3 above) ensures these are retried on resume. After all runs complete, check the format error count in the analyze output.

2. **Latency comparison fairness**: the concurrency-8 run gives accurate accuracy numbers but slightly inflated latency numbers (NIM backend handles 8 requests at once). See section 8 for the separate sequential latency test.

3. **FP8 weights**: NIM serves this model with NVFP4 weights + FP8 KV cache. True FP8 weights on an H100 would require local vLLM (which was explicitly rejected in favor of NIM). The NIM results represent what customers actually get from the hosted service.

4. **Thinking ON sample size**: originally planned as 300 emails for thinking ON (to save time/cost), but the actual runs use all 2000. The thinking-ON runs take ~3-4 hours each with concurrency 8. Reduce with `--limit 300` if that is too long.
