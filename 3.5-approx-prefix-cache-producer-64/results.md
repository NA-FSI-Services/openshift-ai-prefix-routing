# Approximate prefix-cache producer at 64

The completions evaluation is in [eval.md](eval.md).

OpenShift AI **3.5.1**. Two Qwen3-0.6B replicas. Queue weight 2, prefix weight 3, plus `kv-cache-utilization-scorer` and `no-hit-lru-scorer` at weight 2. vLLM `--block-size=32`. The model pods were not restarted.

The controller kept `blockSizeTokens: 64` and `autoTune: false` on `approx-prefix-cache-producer`. The process auto-created `token-producer` with the estimate backend, 4 bytes per pseudo-token. On that backend `blockSizeTokens: 64` is a **256-byte** window. The producer log reporting 64 with `autoTune: false` means that 256-byte window is live. A value below 64 on this binary is raised to 64, so copying 32 onto this field makes the same 256-byte matcher.

The document does not name a model tokenizer. The rows below are the estimate backend. They are not a token-id match.

`vllm:cache_config_info` has to say `block_size="32"` and `enable_prefix_caching="True"`.

| Family | Qwen tokens | Bytes | Where it went | Hit tokens |
|---|---|---|---|---|
| `cinnamon`, one at a time | 30 | 266 | one pod, 8 of 8 | 0 |
| `cinnamon` | 23 | 203 | both pods, 12 and 12 | 0 |
| `red`, requests differ | 20 | 81 | both pods | 0 |
| `cat`, requests differ | 45 | 181 | both pods | `(11) × 32` on each |
| `green`, requests differ inside the first 64 Qwen tokens | 50 | 299 | one pod, 24 | 736, `(23) × 32` |
| `x`, requests differ | 67 | 135 | both pods | `(requests − 1) × 64` |
| 66-token prefix plus a long unique tail | 66 | 265 | one pod, 24 | 1472, `(23) × 64` |

`green` is over 256 bytes and under 64 Qwen tokens, and it pinned. `x` is over 64 Qwen tokens and under 256 bytes, and it used both pods, including when the requests were sent one at a time. The matcher followed the byte window.

A 266-byte prefix of 30 Qwen tokens pinned with a hit counter of 0. Identical copies of 20 Qwen tokens (43 bytes) can pin with 0 hits on this backend. Unique prompts in the same run still have to spread.

With both extra scorers at weight 2, the 66-token prefix with a long unique tail stayed on one of the two pods once the text was past 256 bytes. Prefix weight 3 held it.

Producer `blockSizeTokens: 96` (384 bytes on this estimate backend) was not run on 3.5.1. On 3.4.4 the same number 96, set on `prefix-cache-scorer`, is the configuration in [3.4-prefix-cache-scorer-96](../3.4-prefix-cache-scorer-96/).
