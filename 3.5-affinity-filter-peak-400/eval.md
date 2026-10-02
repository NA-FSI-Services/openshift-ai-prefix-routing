# Evaluation

Checked 2026-10-01 on OpenShift AI 3.5.1. No TrustyAI `LMEvalJob` was created for this document. The check is a completions load against `qwen-l4`, two replicas, model `Qwen/Qwen3-0.6B`. vLLM stayed at `--block-size=32`. The model pods were not restarted. Traffic tables are in [results.md](results.md).

The scheduler loaded `peakPrefillThroughput: 400` with the producer names set. A later pod loaded the same peak with those names omitted. Counts are the increase in `POST /v1/completions` on each pod.

| | |
|---|---|
| API | `POST /v1/completions` |
| Model | `Qwen/Qwen3-0.6B` |
| `max_tokens` | 1 |
| temperature | 0 |

| Family | Requests | Concurrency | Notes |
|---|---|---|---|
| Unique prompts | 16 | 4 | split 6 and 10 |
| `cedar`, 3800 characters | 16 | 4 | split 7 and 9 |
| `cedar`, back to back | 12 | 1 | split 5 and 7 |
| `pine`, 3760 characters | 16 | 4 | split 8 and 8 |
| `IDLE400`, 3200 characters | 12 | 0.8 s idle between requests | 0 and 12 |
| 3030 characters, producer names omitted | 16 | 4 | split 9 and 7 |

The service was put back to queue weight 2 and prefix weight 3 after the check.

The same peak, with the producer name fields omitted, was loaded again with four replicas. The scheduler logged `peakPrefillThroughput: 400`, `blockSizeTokens: 96`, and `autoTune: false`. Those counts, including the 1860-character family this peak left on one pod, are in [results.md](results.md).
