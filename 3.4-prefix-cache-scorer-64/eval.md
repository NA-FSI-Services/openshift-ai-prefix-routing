# Evaluation

Checked 2026-10-01 on OpenShift AI 3.4.4. No TrustyAI `LMEvalJob` was created for this document. The check is a completions load against `qwen-l4`, four replicas, model `Qwen/Qwen3-0.6B`. Traffic tables are in [results.md](results.md).

## Product-weight load

Queue weight 2, prefix weight 3. vLLM `--block-size=32`. `vllm:cache_config_info` reported `block_size="32"` and `enable_prefix_caching="True"`. The scheduler log showed `blockSizeTokens: 64` and `autoTune: false`.

| | |
|---|---|
| API | `POST /v1/completions` |
| Requests per family | 24 |
| Concurrency | 4 |
| `max_tokens` | 1 |
| temperature | 0 |
| Hits | increase in `vllm:prefix_cache_hits_total` |
| Queries | increase in `vllm:prefix_cache_queries_total` |

Families: `oak`, `sage`, `amber` (24 identical copies), `cinnamon`, `quartz`, `violet`, `walnut`, `copper`, and `marble` (66-token prefix plus a long unique tail). A new word started cold.

## Earlier load at prefix weight 10

Same cluster and four replicas, before the product-weight pass. Queue weight 1, prefix-cache weight 10. vLLM stayed at `--block-size=32`. The model pods were not restarted. The scheduler pod logged `autoTune=false`, `blockSizeTokens=64`, `maxPrefixBlocksToMatch=256`, `lruCapacityPerServer=31250`, and `parsed config` with queue-scorer weight 1, prefix-cache-scorer weight 10, and `max-score-picker`.

The same 24-request, concurrency-4, `max_tokens` 1, temperature 0 shape was used. Token counts were read from the model pod `/tokenize`. A separate pass removed `prefix-cache-scorer` and left `queue-scorer` at weight 1. That pass sent a 97-token family and a 49-token family. Pod counts for both passes are in [results.md](results.md).
