# Evaluation

Checked 2026-10-01 on OpenShift AI 3.4.4. No TrustyAI `LMEvalJob` was created for this document. The check is a completions load against `qwen-l4`, four replicas, model `Qwen/Qwen3-0.6B`. Traffic tables are in [results.md](results.md).

Queue weight 2, prefix weight 3. vLLM stayed at `--block-size=32`. The model pods were not restarted. The new scheduler logged `blockSizeTokens: 96` and `autoTune: false`.

| | |
|---|---|
| API | `POST /v1/completions` |
| Requests per family | 24 |
| Concurrency | 4 |
| `max_tokens` | 1 |
| temperature | 0 |
| Hits | increase in `vllm:prefix_cache_hits_total` |
| Client time | end-to-end p50 and p95 through the load balancer |
| Server time | vLLM mean time to first token over the same 24 requests |

First pass families: `silver`, `ivory`, `cinnamon`, `castle`.

Second pass, after the same document was applied again: unique prompts, `willow` (257 characters, 73 tokens), and `peacock` (389 characters, 97 tokens). The scheduler log for that apply again showed `blockSizeTokens: 96` and `autoTune: false`.
