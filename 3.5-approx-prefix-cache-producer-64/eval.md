# Evaluation

Checked 2026-10-01 on OpenShift AI 3.5.1. No TrustyAI `LMEvalJob` was created for this document. The check is a completions load against `qwen-l4`, two replicas, model `Qwen/Qwen3-0.6B`. Traffic tables are in [results.md](results.md).

The controller kept `blockSizeTokens: 64` and `autoTune: false` on `approx-prefix-cache-producer`. The process auto-created the estimate `token-producer`. vLLM stayed at `--block-size=32`. The model pods were not restarted. Queue weight 2, prefix weight 3, `kv-cache-utilization-scorer` weight 2, `no-hit-lru-scorer` weight 2.

| | |
|---|---|
| API | `POST /v1/completions` |
| Model | `Qwen/Qwen3-0.6B` |
| Hits | increase in `vllm:prefix_cache_hits_total` |

| Family | Requests observed | Send shape |
|---|---|---|
| `cinnamon`, 30 tokens, 266 bytes | 8 | one at a time |
| `cinnamon`, 23 tokens, 203 bytes | 24 | 12 and 12 across both pods |
| `red`, 20 tokens, 81 bytes | both pods | requests differ |
| `cat`, 45 tokens, 181 bytes | 24 | `(11) × 32` hits on each pod, so 12 requests per pod |
| `green`, 50 tokens, 299 bytes | 24 | one pod |
| `x`, 67 tokens, 135 bytes | both pods | requests differ, including one at a time |
| 66-token prefix plus a long unique tail, 265 bytes | 24 | one pod |

The service was restored to the picker document it had before this check. The model pods were not restarted.
