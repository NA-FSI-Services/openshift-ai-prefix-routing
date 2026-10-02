# Evaluation

Checked 2026-10-01 on OpenShift AI 3.5.1. No TrustyAI `LMEvalJob` was created for this document. The check is a completions load against `qwen-l4`, four replicas, model `Qwen/Qwen3-0.6B`. vLLM stayed at `--block-size=32`. The model pods were not restarted. Traffic tables are in [results.md](results.md).

The scheduler logged `weighted-random-picker`, `blockSizeTokens: 96`, `autoTune: false`, and `peakPrefillThroughput: 200`. Counts are the increase in `POST /v1/completions` on each pod.

| | |
|---|---|
| API | `POST /v1/completions` |
| Model | `Qwen/Qwen3-0.6B` |
| temperature | 0 |
| `max_tokens` | 1 |

| Family | Requests | Concurrency | Notes |
|---|---|---|---|
| Unique prompts | 16 | 4 | 4, 4, 3, 5 |
| 248 bytes | 12 | 4 | 5, 5, 2, 0 |
| `willow`, 1860 characters | 16 | 4 | 11, 0, 0, 5 |
| `peacock`, 1960 characters | 16 | 4 | 8, 0, 0, 8 |
| `birch`, 3680 characters | 16 | 4 | 7, 5, 4, 0 |
| `willow`, back to back | 8 | 1 | 5, 0, 0, 3 |
| Idle, 3200 characters | 12 | 0.8 s between requests | 12, 0, 0, 0 |

The service was put back to queue weight 2 and prefix weight 3 after the check. Replicas were left at 4.
