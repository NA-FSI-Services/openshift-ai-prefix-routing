# Prefix-cache scorer at 96, vLLM at 32

The completions evaluation is in [eval.md](eval.md).

OpenShift AI **3.4.4**. Four Qwen3-0.6B replicas. Queue weight 2, prefix weight 3. vLLM left at `--block-size=32`. The model pods were not restarted for this check.

`blockSizeTokens: 96` is a 384-character window (`96 × 4`). The new scheduler logged `blockSizeTokens: 96` and `autoTune: false`. On this Qwen tokenizer, 384 characters was 44 tokens, one vLLM block of 32, and the hit counter moved.

24 requests per family, concurrency 4, `max_tokens` 1, temperature 0.

## Benchmark

| Family | Shared characters | Shared tokens | Pods | Requests per pod | Hit tokens | Hit rate | Client p50 | Client p95 | Mean TTFT |
|---|---|---|---|---|---|---|---|---|---|
| `silver` | 279 | 40 | 4 | 4, 9, 2, 9 | 96, 256, 32, 256 | 62.6% | 0.187 s | 0.209 s | 0.012 s |
| `ivory` | 258 | 43 | 4 | 4, 8, 5, 7 | 96, 224, 128, 192 | 58.5% | 0.186 s | 0.218 s | 0.012 s |
| `cinnamon` | 393 | 44 | 1 | 24, 0, 0, 0 | 736 | 65.8% | 0.185 s | 0.213 s | 0.014 s |
| `castle` | 678 | 97 | 1 | 0, 24, 0, 0 | 2208 | 92.4% | 0.186 s | 0.214 s | 0.014 s |

`ivory` is 258 characters, the same window that pinned when the scorer was 64. At 96 it used all four pods. Pods that saw it twice cached one block of 32: `(requests − 1) × 32` matches every column. `cinnamon` at 393 characters pinned, and 736 is `(23) × 32`. Forty-four tokens is one block of 32, so 65.8% is that block against the whole prompt. `castle` at 97 tokens pinned, and 2208 is `(23) × 96`.

Client latency does not separate these rows. p50 stays between 0.185 s and 0.187 s. Qwen3-0.6B prefill is shorter than the round trip to the load balancer.

## Repeat on the same cluster

A second apply of the same document, same weights, vLLM still at 32.

| Family | Shared | Pods | Hit tokens |
|---|---|---|---|
| Unique prompts | none | 4 (7, 6, 8, 3) | 0 |
| `willow` | 257 characters, 73 tokens | 4 (6, 3, 12, 3) | 320, 128, 704, 128, which is `(requests − 1) × 64` |
| `peacock` | 389 characters, 97 tokens | 1 (24) | 2208, `(23) × 96` |

Unique prompts used the pool. A repeated prefix under 384 characters used all four pods. A repeated prefix over that window stuck to one pod, and the hit counter rose by 32 tokens per full vLLM block.
