# Prefix-cache scorer at 144, vLLM at 64

The completions evaluation is in [eval.md](eval.md).

OpenShift AI **3.4.4**. Four Qwen3-0.6B replicas. Queue weight 2, prefix weight 3. vLLM `--block-size=64`. The scheduler log showed `blockSizeTokens: 144`.

`144 × 4 = 576` characters. A 64-token prefix of the `cinnamon` family on this tokenizer is 572 characters, so that prefix does not complete a character block. This is the measured pairing when vLLM is already at 64. A longer token reaches 64 tokens in more than 572 characters. The character length of a 64-token prefix on the tokenizer in use is the number to divide by 4, then add one block.

24 requests per family, concurrency 4, `max_tokens` 1, temperature 0.

| Family | Shared characters | Shared tokens | Pods | Requests per pod | Hit tokens | Hit rate | Client p50 | Mean TTFT |
|---|---|---|---|---|---|---|---|---|
| `trumpet` | 526 | 66 | 4 | 7, 1, 7, 9 | 384, 0, 384, 512 | 77.8% | 0.189 s | 0.013 s |
| `soldier` | 558 | 70 | 4 | 6, 1, 9, 8 | 320, 0, 512, 448 | 73.5% | 0.190 s | 0.019 s |
| `vanilla` | 582 | 73 | 1 | 0, 0, 24, 0 | 1472 | 81.1% | 0.186 s | 0.014 s |

Under 576 characters the family spreads, including a 70-token prefix that already fills one KV block of 64. Hits on those pods are `(requests − 1) × 64`. The pod that received one request recorded 0. Over 576 characters the family pins, and 1472 is `(23) × 64`. Seventy-three tokens do not fill a second block.
