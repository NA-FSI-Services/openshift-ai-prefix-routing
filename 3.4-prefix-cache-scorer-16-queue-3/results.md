# Prefix window 16, queue weight 3, prefix weight 2

The completions load is in [eval.md](eval.md). The locked document is [llminferenceservice.yaml](llminferenceservice.yaml).

OpenShift AI **3.4.4**. Four Qwen3-0.6B replicas. Queue weight **3**, prefix weight **2**. vLLM left at `--block-size=32`. The model pods were not restarted.

`blockSizeTokens: 16` is a 64-character window (`16 × 4`). The product weights are queue 2 and prefix 3. This run swaps them. A full prefix match still adds to the queue score of the pod that already holds the prefix, so a higher queue weight does not by itself send the next request to an idle pod while that queue is empty.

Client p50 on the first pass stayed between 0.19 s and 0.22 s. Qwen3-0.6B answers faster than a queue forms at these depths.

Pod order is `clwtr`, `jnqwp`, `m4ktj`, `qgbvj`.

## Window locked at 16

Scheduler `qwen-l4-kserve-router-scheduler-68c959d4c9-6zwwk`, `2026-10-05T15:36:08Z`: `blockSizeTokens: 16`, `autoTune: false`, queue weight 3, prefix weight 2.

| Family | Shared text | Load | Requests per pod | Hits | Result |
|---|---|---|---|---|---|
| `linen` | 59 characters, 10 tokens | 24 requests, concurrency 4 | 6, 7, 3, 8 | 0 | Spread. Under 64 characters. |
| `pearl` | 95 characters, 16 tokens | 24, concurrency 4 | 0, 0, 24, 0 | 0 | Pinned. One picker block, no KV block. |
| `topaz` | 167 characters, 55 tokens | 24, concurrency 4 | 0, 0, 24, 0 | 736 | Pinned. `736 = 23 × 32`. |
| `crimson` | 765 characters, 96 tokens | 24, concurrency 4 | 0, 0, 0, 24 | 2208 | Pinned. `2208 = 23 × 96`. |
| `indigo` | 96 characters, 27 tokens | 96, concurrency 32 | 0, 96, 0, 0 | 0 | Pinned. One picker block, no KV block. |
| `maroon` | 341 characters, 97 tokens | 96, concurrency 32 | 0, 0, 0, 96 | 9120 | Pinned. `9120 = 95 × 96`. |

`pearl` is the gap. Ninety-five characters complete one 64-character picker block and stay at 16 Qwen tokens, under one vLLM block of 32. All 24 requests landed on `m4ktj` and `vllm:prefix_cache_hits_total` stayed 0. `indigo` is the same shape at 32 concurrent users: all 96 requests on `jnqwp`, hit counter 0.

`linen` at 59 characters does not complete a picker block and used all four pods.

## Field omitted

The first pass removed the `prefix-cache-scorer` parameters, which is the 3.4 default. Scheduler `qwen-l4-kserve-router-scheduler-599b8f7c4d-zmjwh` at `2026-10-05T15:29:02Z` logged `BlockSize is not positive, using default value` `default=16`, then `autoTune: true` and `blockSizeTokens: 16`. The same log did not print a later block size. The requests followed a 128-character window, which is `32 × 4`, the vLLM block size with autotune left on.

| Family | Shared text | Load | Requests per pod | Hits | Result |
|---|---|---|---|---|---|
| Unique prompts | none | 24, concurrency 4 | 4, 5, 9, 6 | 0 | Spread |
| `marble` | 61 characters, 9 tokens | 24, concurrency 4 | 3, 7, 4, 10 | 0 | Spread |
| `amber` | 95 characters, 16 tokens | 24, concurrency 4 | 7, 5, 7, 5 | 0 | Spread |
| `jasper` | 96 characters, 27 tokens | 24, concurrency 4 | 6, 4, 6, 8 | 0 | Spread |
| `onyx` | 105 characters, 41 tokens | 24, concurrency 4 | 7, 4, 3, 10 | 192, 96, 64, 288 | Spread, hits on every pod |
| `velvet` | 96 characters, 14 tokens | 96, concurrency 32 | 18, 24, 27, 27 | 0 | Spread |
| Unique, busy | none | 96, concurrency 32 | 21, 25, 27, 23 | 0 | Spread |
| `cobalt` | 159 characters, 45 tokens | 24, concurrency 4 | 0, 24, 0, 0 | 736 | Pinned. `736 = 23 × 32` |
| `nickel` | 229 characters, 33 tokens | 24, concurrency 4 | 0, 0, 0, 24 | 736 | Pinned. `736 = 23 × 32` |
| `copper` | 278 characters, 40 tokens | 96, concurrency 32 | 96, 0, 0, 0 | 3040 | Pinned. `3040 = 95 × 32` |
| `granite` | 317 characters, 40 tokens | 24, concurrency 4 | 24, 0, 0, 0 | 736 | Pinned. `736 = 23 × 32` |
| `basalt` | 341 characters, 97 tokens | 24, concurrency 4 | 0, 0, 24, 0 | 2208 | Pinned. `2208 = 23 × 96` |
| `saffron` | 261 characters, 97 tokens | 96, concurrency 32 | 96, 0, 0, 0 | 9120 | Pinned. `9120 = 95 × 96` |

`onyx` is under 128 characters and over 32 tokens. The picker did not keep the family together. Each pod that saw the prompt more than once stored one KV block: the hit columns are `(requests − 1) × 32`.

`amber` at 95 characters spread on this pass. The same length, `pearl`, pinned after `autoTune` was turned off and `blockSizeTokens` was set to 16.

## What the swap changed

A prefix the picker scores still stays on one pod when the queues are empty. Queue weight 3 and prefix weight 2 did not move `pearl`, `indigo`, `topaz`, `crimson`, or `maroon`. Concurrency 32 did not move `indigo` or `maroon`.

The swap shows up when the picker scores no block. With the field omitted and autotune on, that line was 128 characters: `amber`, `jasper`, and `velvet` used all four pods, and `cobalt` at 159 characters pinned. With the window locked at 16, that line is 64 characters: `linen` spread, and `pearl` at 95 characters pinned with a hit counter of 0.
