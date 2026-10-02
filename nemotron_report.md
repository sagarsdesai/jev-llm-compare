# Nemotron 3.5 Lightning vs Jev vs Claude Haiku 4.5
## PhishNChips v5.2 — 2000 emails, 3-fold cross-validation on local H100 80GB

Generated 2026-10-02 | Model: `nvidia/nemotron-3.5-lightning` (NIM container, local H100)
Jev and Haiku baselines from the original published benchmark (same dataset, same prompt).

---

## Summary table (pooled across 3 folds)

| Model | AUROC | Accuracy | Recall | FPR | ECE | Lat p50 | Wall-clock 2k | Cost/1k |
|---|---|---|---|---|---|---|---|---|
| Jev | 0.689 | 62.6% | 43.2% | 18.0% | 0.154 | 239 ms | ~14 min | $0.038 |
| Haiku 4.5 | 0.837 | 81.3% | 76.4% | 13.8% | 0.097 | 687 ms | ~67 min | $0.462 |
| Nemotron OFF | 0.712 | 64.3% | 30.2% | 1.6% | 0.286 | 419 ms | 28s (0.5 min) | $0.00 |
| Nemotron ON | 0.899 | 77.5% | 58.5% | 3.5% | 0.226 | 3574 ms | 986s (16.4 min) | $0.00 |

Signals task (verdict equivalent for signal-based scoring):

| Model | AUROC | Accuracy | Recall | FPR | Wall-clock 2k |
|---|---|---|---|---|---|
| Jev (B, logistic) | 0.982 | 95.0% | 97.0% | 7.0% | ~14 min |
| Haiku (B, logistic) | 0.991 | 93.2% | 91.8% | 5.4% | ~67 min |
| Nemotron OFF signals | 0.970 | 76.8% | 54.3% | 0.5% | 419s (7.0 min) |
| Nemotron ON signals | 0.929 | 73.0% | 48.5% | 2.6% | 3691s (61.5 min) |

---

## Verdict task — per-fold breakdown

### Nemotron thinking OFF (c=32, warmup=3)

| Fold | Accuracy | Recall | FPR | AUROC | ECE | Lat p50 | Wall-clock |
|---|---|---|---|---|---|---|---|
| Fold 1 (666 emails) | 62.8% | 29.1% | 2.1% | 0.709 | 0.296 | 428 ms | 9.0s |
| Fold 2 (667 emails) | 66.1% | 32.7% | 1.8% | 0.715 | 0.268 | 419 ms | 10.5s |
| Fold 3 (667 emails) | 64.0% | 28.8% | 0.9% | 0.713 | 0.295 | 414 ms | 8.7s |
| **Mean ± std** | 64.3% ±1.7 | 30.2% ±2.2 | 1.6% ±0.6 | 0.712 ±0.003 | 0.286 ±0.016 | 420 ms | 28s total |

### Nemotron thinking ON (c=8, warmup=2)

| Fold | Accuracy | Recall | FPR | AUROC | ECE | Lat p50 | Wall-clock |
|---|---|---|---|---|---|---|---|
| Fold 1 (666 emails) | 77.6% | 60.9% | 4.9% | 0.888 | 0.227 | 3604 ms | 332.9s |
| Fold 2 (667 emails) | 78.6% | 58.7% | 2.4% | 0.908 | 0.216 | 3580 ms | 331.1s |
| Fold 3 (667 emails) | 76.3% | 55.9% | 3.3% | 0.902 | 0.235 | 3553 ms | 321.6s |
| **Mean ± std** | 77.5% ±1.1 | 58.5% ±2.5 | 3.5% ±1.3 | 0.899 ±0.010 | 0.226 ±0.010 | 3579 ms | 986s total |

---

## Signals task — per-fold breakdown

### Nemotron thinking OFF signals (c=32, warmup=3)

| Fold | Accuracy | Recall | FPR | AUROC | Lat p50 | Wall-clock |
|---|---|---|---|---|---|---|
| Fold 1 (656 emails) | 75.9% | 53.6% | 0.9% | 0.966 | 963 ms | 144.1s |
| Fold 2 (665 emails) | 78.2% | 56.3% | 0.6% | 0.970 | 904 ms | 134.1s |
| Fold 3 (661 emails) | 76.4% | 53.0% | 0.0% | 0.976 | 907 ms | 140.4s |
| **Mean ± std** | 76.8% ±1.2 | 54.3% ±1.7 | 0.5% ±0.5 | 0.970 ±0.005 | 925 ms | 419s total |

### Nemotron thinking ON signals (c=8, warmup=2)

| Fold | Accuracy | Recall | FPR | AUROC | Lat p50 | Wall-clock |
|---|---|---|---|---|---|---|
| Fold 1 (665 emails) | 73.1% | 49.3% | 2.1% | 0.933 | 10426 ms | 1224.8s |
| Fold 2 (667 emails) | 73.8% | 50.2% | 3.5% | 0.927 | 10688 ms | 1239.9s |
| **Mean ± std** | 73.0% ±0.9 | 48.5% ±2.0 | 2.6% ±0.8 | 0.929 ±0.003 | 10585 ms | 3691s total |

---

## Key findings

**Verdict task:**
- Thinking ON AUROC **0.899** is the best of the four models, beating Haiku (0.837). But recall 58.5% still trails Haiku (76.4%) — the model is cautious.
- Thinking OFF barely beats Jev on AUROC (0.712 vs 0.689) with near-zero FPR (1.6%) — rarely flags anything.
- Thinking improves AUROC by 0.187 and recall by 28.3pp at the cost of 8x latency and 60x more output tokens.
- Both modes poorly calibrated (ECE 0.286/0.226 vs Haiku 0.097).

**Signals task:**
- Signals decomposition is highly effective: Nemotron OFF AUROC jumps from 0.712 (verdict) to 0.970 (signals) — within 0.012 of Jev (0.982).
- Near-zero FPR on signals: 0.5% OFF — the 5 signal questions are far more precise than a direct verdict.
- Thinking ON **hurts** signals (0.929 vs 0.970 OFF). Reasoning adds noise to URL/domain feature checks.

**Infrastructure:**
- Local H100 vs remote API: 72 rps vs 0.03 rps (thinking OFF) — **2400x faster**. 2000 emails in 28s vs ~18 hours.
- Wall-clock for thinking ON: 986s (16.4 min) (verdict), ~70 min (signals) — dominated by ~1100 reasoning tokens/email.
- Cost: $0 on local GPU vs $0.462/1k for Haiku.

---

## Throughput detail (local H100)

| Run | Concurrency | Emails/s | Output tok/s | Wall-clock (2k emails) |
|---|---|---|---|---|
| Verdict OFF | 32 | 71.3 | 1301 | 28s (0.5 min) |
| Signals OFF | 32 | 4.8 | 238 | 419s (7.0 min) |
| Verdict ON | 8 | 2.0 | 2249 | 986s (16.4 min) |
| Signals ON | 8 | 0.5 | 2442 | 3691s (61.5 min) |

Signals OFF is slower than verdict OFF (4.8 vs 71 emails/s) because the prompt is 5x longer —
more tokens to prefill per request, less batching efficiency. Output tokens/s similar (signals
returns 5 floats vs 2 for verdict).
