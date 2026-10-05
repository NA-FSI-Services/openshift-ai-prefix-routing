# Evaluation

Checked 2026-10-05 on OpenShift AI 3.4.4. No TrustyAI `LMEvalJob` was created. The check is a completions load against `qwen-l4` in `llm-d-repro`, four ready replicas, model `Qwen/Qwen3-0.6B`. Traffic tables are in [results.md](results.md). The service manifest for the locked window is [llminferenceservice.yaml](llminferenceservice.yaml).

The product profile is queue weight 2 and prefix weight 3. This run swapped those weights to queue **3** and prefix **2**. vLLM stayed at `--block-size=32`. The model pods were not restarted. Their args were `--max-model-len=4096` and `--block-size=32`. `vllm:cache_config_info` on a running pod reported `block_size="32"` and `enable_prefix_caching="True"`.

Pod order in [results.md](results.md) is `clwtr`, `jnqwp`, `m4ktj`, `qgbvj` (`qwen-l4-kserve-6685bfc95b-*`).

## Two picker documents

Both documents use `apiVersion: inference.networking.x-k8s.io/v1alpha1` and `max-score-picker`, which this binary injects. Queue weight 3, prefix weight 2.

| Pass | `prefix-cache-scorer` parameters | Scheduler pod | Startup log |
|---|---|---|---|
| Field omitted | none | `qwen-l4-kserve-router-scheduler-599b8f7c4d-zmjwh` | `2026-10-05T15:29:02Z`. `BlockSize is not positive, using default value` `default=16`. `autoTune: true`, `blockSizeTokens: 16`. |
| Window locked | `blockSizeTokens: 16`, `autoTune: false` | `qwen-l4-kserve-router-scheduler-68c959d4c9-6zwwk` | `2026-10-05T15:36:08Z`. `autoTune: false`, `blockSizeTokens: 16`. |

`parsed config` on both pods listed queue-scorer weight 3.00, prefix-cache-scorer weight 2.00, and `max-score-picker`.

On this binary, `blockSizeTokens` is a character window (`blockSizeTokens × 4`). Sixteen is 64 characters. An incomplete character block is dropped. vLLM stores one KV block per 32 tokens. A partial block is not stored.

## Load

`POST /v1/completions` to the `qwen-l4` status URL. `max_tokens` 1, temperature 0. Hits are the increase in `vllm:prefix_cache_hits_total`. Requests are the increase in `vllm:request_success_total`.

Each family starts from `Hello` and appends ` <word>` until the preamble crosses a character or token target. The suffix ` request N only` keeps the prompts from being identical. Token counts are Qwen3-0.6B token ids, with no special tokens added. A new word starts cold.

| Pass | Requests | Concurrency |
|---|---|---|
| Quiet | 24 | 4 |
| Busy | 96 | 32 |

After the check the service was put back to `blockSizeTokens: 32`, `autoTune: false`, queue weight 1, and prefix weight 10. Scheduler `qwen-l4-kserve-router-scheduler-89867ddf8-v64jn` logged that at `2026-10-05T15:38:20Z`.
