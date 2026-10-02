# Evaluation

Checked 2026-10-01 on OpenShift AI 3.4.4. No TrustyAI `LMEvalJob` was created for this document. The check is a completions load against `qwen-l4`, four replicas, model `Qwen/Qwen3-0.6B`. Traffic tables are in [results.md](results.md).

The model pods were rolled to `--block-size=64`. They logged `block_size: 64`, selected FlashAttention, and `vllm:cache_config_info` reported `block_size="64"` and `enable_prefix_caching="True"`. Queue weight 2, prefix weight 3. The scorer stayed at `blockSizeTokens: 64` and `autoTune: false`.

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

Families: `silk`, `zinc`, `coral` (24 identical copies), `spinach`, `bronze`, `palace`, `crystal`, `diamond`, and `dolphin` (66-token prefix plus a long unique tail).

`bronze` delivered 23 of 24 requests. One request failed in DNS before a pod.

`silk`, the first family after the rollout, had client p95 0.516 s and mean time to first token 0.089 s while the new pods finished compile.

A separate cold-then-warm check, `library`, sent 1 cold request and then 8 warm requests of a 97-token prefix. It stayed on one pod, recorded 512 hit tokens (`(8) × 64`), client p50 0.190 s cold and 0.182 s warm, and mean time to first token 0.012 s.
