# Prefix-cache scorer at 64, vLLM at 32

The completions evaluation is in [eval.md](eval.md).

OpenShift AI **3.4.4**. Four Qwen3-0.6B replicas. Queue weight 2, prefix weight 3. vLLM `--block-size=32`, prefix caching on. `vllm:cache_config_info` reported `block_size="32"` and `enable_prefix_caching="True"`.

`blockSizeTokens: 64` is a 256-character window (`64 × 4`). The scheduler log showed `blockSizeTokens: 64` and `autoTune: false`. The 3.4.4 image has no `approx-prefix-cache-producer` type. The field is set on `prefix-cache-scorer`.

24 requests per family, concurrency 4, `max_tokens` 1, temperature 0.

## Short prefixes

Shared text under 256 characters. The incomplete character block is dropped, so the prefix score stays 0, including for identical copies.

| Family | Shared characters | Shared Qwen tokens | Pods | Requests per pod | Hit tokens | Hit rate |
|---|---|---|---|---|---|---|
| `oak` | 82 | 20 | 4 | 9, 7, 4, 4 | 0 | 0% |
| `sage` | 201 | 40 | 4 | 6, 9, 7, 2 | 160, 256, 192, 32 | 62.6% |
| `amber`, 24 identical copies | 239 | 40 | 4 | 3, 6, 8, 7 | 64, 160, 224, 192 | 66.7% |

The 40-token rows record `(requests on that pod − 1) × 32` wherever a pod received the prompt twice. Routing did not keep the family together. The 20-token row records nothing, because it does not fill a vLLM block of 32.

## Long prefixes

Shared text of 256 characters or more. The picker pins the family. vLLM records a hit only when the shared prefix fills 32 tokens.

| Family | Shared characters | Shared Qwen tokens | Pods | Hit tokens | What that count is |
|---|---|---|---|---|---|
| `cinnamon` | 258 | 29 | 1 of 4 | 0 | Under one block of 32. Pinned, and nothing is reused. |
| `quartz` | 258 | 37 | 1 of 4 | 736 | `(23) × 32`. One block. |
| `violet` | 461 | 66 | 1 of 4 | 1472 | `(23) × 64`. Two blocks of 32. |
| `walnut` | 678 | 97 | 1 of 4 | 2208 | `(23) × 96`. Three blocks of 32. |
| `copper` | 909 | 130 | 1 of 4 | 2976 | Four blocks of 32 would be `(23) × 128 = 2944`. This counter was 32 higher. |
| `marble`, 66-token prefix plus a long unique tail | 461 | 66 | 2 of 4 | 512 and 896 | `(requests − 1) × 64` on each pod that received it. |

`cinnamon` and `quartz` are the same character window and different token counts. The picker pinned both. vLLM could reuse only `quartz`.

## Character boundary at prefix weight 10

An earlier run on the same cluster used prefix weight 10 and queue weight 1. The pin started at 256 characters, independent of the Qwen token count.

| Shared characters | Shared Qwen tokens | Pods | Hit tokens |
|---|---|---|---|
| 252 | 43 | 4 | 32 on pods that took more than one request |
| 258 | 44 | 1 | that pod |
| 251 | 51 | 4 | 32 per pod that took more than one request |
| 256 | 52 | 1 | that pod |
| 45–61 tokens, over 256 characters | 45–61 | 1 | `(requests − 1) × 32` = 736 |
| 64–66 tokens | 64–66 | 1 | `(requests − 1) × 64` = 1472 |
| 97 tokens | 97 | 1 or 2 | `(requests − 1) × 96` |

With the scorer removed and only `queue-scorer` left at weight 1, a 97-token family used all four pods (`4, 3, 6, 11`). Hits on each pod were still `(requests − 1) × 96`. The scorer is what collapsed the long family onto one or two pods.
