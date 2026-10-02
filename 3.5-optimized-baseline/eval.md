# Evaluation

Checked 2026-10-01. No TrustyAI `LMEvalJob` was created for this document. On 3.4.4 the scheduler did not become Ready, so there is no traffic result on that release. On 3.5.1 the check is a completions load against `qwen-l4`, two replicas, model `Qwen/Qwen3-0.6B`. Traffic tables are in [results.md](results.md).

## OpenShift AI 3.4.4

The inline document was stored and rendered into the scheduler Deployment `--config-text`. The scheduler pod entered `CrashLoopBackOff` at startup with `no kind "EndpointPickerConfig" is registered for version "llm-d.ai/v1alpha1"`. A throwaway pod of the same plugins rewritten to `apiVersion: inference.networking.x-k8s.io/v1alpha1` exited with `plugin type 'approx-prefix-cache-producer' is not registered`. The service was restored afterward. No completions were sent while the scheduler was down.

## OpenShift AI 3.5.1

Two ready pods. The scheduler loaded the four plugins, injected `max-score-picker`, set `token-load-scorer` weight to 1, and raised `blockSizeTokens` from 16 to 64 with `autoTune: true`. Counts are the increase in `POST /v1/completions` on each pod.

| | |
|---|---|
| API | `POST /v1/completions` |
| `max_tokens` | 1 |
| temperature | 0 |
| Model | `Qwen/Qwen3-0.6B` |

| Family | Requests | Concurrency | Client p50 |
|---|---|---|---|
| Unique prompts, before this document | 16 | 4 | 0.184 s |
| Shared prefix, 1860 characters, before this document | 16 | 4 | 0.195 s |
| Unique prompts | 16 | 4 | — |
| `willow`, 1860 characters | 16 | 4 | — |
| `peacock`, 1960 characters | 16 | 4 | — |
| `willow`, one request at a time | 8 | 1 | — |
| Short prefix, 72 bytes | 12 | 4 | — |
| Prefix under one 256-byte block, 248 bytes | 12 | 4 | — |
| Prefix over one 256-byte block, 336 bytes | 12 | 4 | — |
| `willow` plus a 3160-character unique tail | 16 | 4 | — |

Nothing waited in vLLM. Requests finished in about 0.2 s. The service was put back to queue weight 2 and prefix weight 3 after the check.

The same document was loaded again the same day with four replicas. The scheduler again injected `max-score-picker` and raised the producer block size from 16 to 64. The four-replica counts are in [results.md](results.md). The service was restored to queue weight 2 and prefix weight 3 afterward. Replicas were left at 4.
