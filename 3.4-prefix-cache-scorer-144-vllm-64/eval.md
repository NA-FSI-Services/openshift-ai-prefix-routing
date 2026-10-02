# Evaluation

Checked 2026-10-01 on OpenShift AI 3.4.4. No TrustyAI `LMEvalJob` was created for this document. The check is a completions load against `qwen-l4`, four replicas, model `Qwen/Qwen3-0.6B`. Traffic tables are in [results.md](results.md).

Queue weight 2, prefix weight 3. vLLM `--block-size=64`. The scheduler log showed `blockSizeTokens: 144`.

| | |
|---|---|
| API | `POST /v1/completions` |
| Requests per family | 24 |
| Concurrency | 4 |
| `max_tokens` | 1 |
| temperature | 0 |
| Hits | increase in `vllm:prefix_cache_hits_total` |
| Client time | end-to-end p50 through the load balancer |
| Server time | vLLM mean time to first token over the same 24 requests |

Families: `trumpet` (526 characters, 66 tokens), `soldier` (558 characters, 70 tokens), and `vanilla` (582 characters, 73 tokens).
