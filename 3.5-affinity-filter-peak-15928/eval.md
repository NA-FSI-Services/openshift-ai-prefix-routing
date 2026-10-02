# Evaluation

Checked 2026-10-01 on OpenShift AI 3.5.1. No TrustyAI `LMEvalJob` was created for this document. The check is a completions load against `qwen-l4`, two replicas, model `Qwen/Qwen3-0.6B`. vLLM stayed at `--block-size=32`. The model pods were not restarted. Traffic tables are in [results.md](results.md).

The first scheduler pod logged `Picker: weighted-random-picker`, `blockSizeTokens: 96`, and `autoTune: false`. A second apply added `inFlightLoadProducerName: inflight-load-producer` and `prefixMatchInfoProducerName: approx-prefix-cache-producer`. Counts are the increase in `POST /v1/completions` on each pod.

| | |
|---|---|
| API | `POST /v1/completions` |
| Model | `Qwen/Qwen3-0.6B` |
| temperature | 0 |
| `max_tokens` | 1, except the 40-request `birch` row |

| Family | Requests | Concurrency | `max_tokens` | Notes |
|---|---|---|---|---|
| Unique prompts | 16 | 4 | 1 | |
| 72 bytes, 248 bytes, 301 bytes, 477 bytes | 12 each | 4 | 1 | |
| `willow`, 1860 characters | 16 | 4 | 1 | |
| `peacock`, 1960 characters | 16 | 4 | 1 | same pod as `willow` |
| `willow`, one at a time | 8 | 1 | 1 | same pod |
| `willow` plus a 3160-character tail | 16 | 4 | 1 | |
| `birch`, 3680 characters | 16 | 4 | 1 | producer names set |
| `birch` | 40 | 50 ms apart | 80 | producer names set |
| `aspen`, a second long preamble | 16 | 4 | 1 | producer names set |

The service was put back to queue weight 2 and prefix weight 3 after the check.

The same document, with the producer name fields omitted, was loaded again with four replicas. The scheduler logged `weighted-random-picker`, `blockSizeTokens: 96`, `autoTune: false`, and `peakPrefillThroughput: 15928`. Those counts are in [results.md](results.md).
