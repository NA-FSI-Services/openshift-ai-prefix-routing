# Prefix-cache scorer at 32, vLLM at 32

Evaluation jobs, including the TrustyAI `LMEvalJob` manifests and ARC-Easy scores, are in [eval.md](eval.md).

OpenShift AI **3.4.4**. Four Qwen3-0.6B replicas. Queue weight 2, prefix weight 3. vLLM `--block-size=32`, prefix caching on.

`blockSizeTokens: 32` is a 128-character window (`32 × 4`). The scheduler log for this setting is `PrefixCachePlugin initialized` with `blockSizeTokens: 32` and `autoTune: false`. When the field is omitted, the same binary logs `blockSizeTokens: 16`.

## Traffic

24 `/v1/completions` requests per family, concurrency 4, `max_tokens` 1, temperature 0.

| Family | Shared characters | Shared Qwen tokens | Pods | Requests per pod | Hit tokens |
|---|---|---|---|---|---|
| `cinnamon` | 132 | 15 | 3 of 4 | 15, 3, 0, 6 | 0 |
| 97-token prefix | — | 97 | 1 of 4 | 24 on one pod | 2208, `(23) × 96` |

The 132-character prefix is one complete character block and 15 Qwen tokens, under one vLLM block of 32. The picker kept most of the family on one pod. `vllm:prefix_cache_hits_total` stayed 0. The 97-token prefix fills three vLLM blocks of 32, and 2208 is `(23) × 96`.

Matching the two integers does not make a picker block equal a KV block. On this tokenizer the picker starts pinning at 128 characters, which can be well under 32 tokens.

## Same window at prefix weight 10

An earlier 3.4.4 run on the same four replicas used prefix weight 10 and queue weight 1, with vLLM still at 32. That weight pair holds any scored prefix. The character boundary was the same.

| Shared characters | Shared Qwen tokens | Pods | Hit tokens |
|---|---|---|---|
| 118 | 18 | 4 of 4 | 0 |
| 132 | 20 | 2 of 4 (11, 0, 0, 13) | 0 |
| 97-token prefix (`maple`) | 97 | 1 of 4 | 2208 |

A separate 16-token shared prefix (80 characters, under `32 × 4` and over `16 × 4`) used all four pods at `blockSizeTokens: 32` and two of four pods at `blockSizeTokens: 16`. Hits stayed 0 in both cases: the prompts are 20 or 21 tokens and do not fill a vLLM block of 32.
