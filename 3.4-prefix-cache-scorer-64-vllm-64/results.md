# Prefix-cache scorer at 64, vLLM at 64

The completions evaluation is in [eval.md](eval.md).

OpenShift AI **3.4.4**. Four Qwen3-0.6B replicas. Queue weight 2, prefix weight 3. vLLM `--block-size=64`. The new pods logged `block_size: 64`, selected FlashAttention, and `vllm:cache_config_info` reported `block_size="64"` and `enable_prefix_caching="True"`.

This was Qwen3-0.6B on an L4. Routing still follows the 256-character picker window. The hit counter moves only in blocks of 64.

24 requests per family, concurrency 4, `max_tokens` 1, temperature 0.

## Short prefixes

Shared text under 256 characters. The family spreads. Forty shared tokens fill one block of 32 and fill none of a block of 64, so the hit counter stays 0.

| Family | Shared characters | Shared Qwen tokens | Pods | Requests per pod | Hit tokens |
|---|---|---|---|---|---|
| `silk` | 101 | 20 | 4 | 5, 6, 6, 7 | 0 |
| `zinc` | 201 | 40 | 4 | 7, 10, 3, 4 | 0 |
| `coral`, 24 identical copies | 239 | 40 | 4 | 7, 6, 3, 8 | 0 |

The first family after the rollout (`silk`) had client p95 0.516 s and mean time to first token 0.089 s while the new pods finished compile. Later families settled near the times in the long-prefix table.

## Long prefixes

| Family | Shared characters | Shared Qwen tokens | Pods | Hit tokens | Hit rate | Client p50 | Client p95 | Mean TTFT |
|---|---|---|---|---|---|---|---|---|
| `spinach` | 262 | 33 | 1 of 4 | 0 | 0% | — | — | — |
| `bronze` | 258 | 37 | 1 of 4 | 0 | 0% | — | — | — |
| `palace` | 461 | 66 | 1 of 4 | 1472 | 89.4% | 0.187 s | 0.215 s | 0.014 s |
| `crystal` | 774 | 97 | 1 of 4 | 1472 | 61.6% | 0.183 s | 0.218 s | 0.014 s |
| `diamond` | 1038 | 130 | 1 of 4 | 2944 | 92.5% | 0.191 s | 0.207 s | 0.015 s |
| `dolphin`, 66-token prefix plus a long unique tail | 526 | 66 | 1 of 4 | 1472 | — | — | — | — |

`bronze`: 23 of 24 requests reached the pod. One request failed in DNS before a pod. Hits stayed 0.

The 33-token and 37-token families are long enough for the 256-character window and short of a 64-token KV block. 1472 is one cached block of 64, `(23) × 64`. 2944 is two, `(23) × 128`. A 97-token prefix at vLLM 32 cached 2208 tokens (three blocks of 32) with client p50 0.187 s. Moving the KV block from 32 to 64 left client latency on this model in the same band and changed how much of a 97-token prefix is reusable: 64 tokens, once, until the prefix reaches 128.

A cold-then-warm check of a 97-token prefix (`library`, 1 cold request then 8 warm) stayed on one pod, recorded 512 hit tokens (`(8) × 64`), client p50 0.190 s cold and 0.182 s warm, mean TTFT 0.012 s.
