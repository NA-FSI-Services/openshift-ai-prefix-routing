# Evaluation

Two evaluations were run against this block size on 2026-10-01. The TrustyAI `LMEvalJob` objects are in [lmevaljob.yaml](lmevaljob.yaml). The routing tables are in [results.md](results.md).

The `LMEvalJob` runs used queue weight **1** and prefix-cache weight **10**. [llminferenceservice.yaml](llminferenceservice.yaml) is the product-weight document, queue **2** and prefix **3**. The completions rows in [results.md](results.md) are that product-weight check. The ARC-Easy scores below are the weight 10 / 1 check.

## LMEvalJob

OpenShift AI 3.4.4. Service `qwen-l4` in `llm-d-repro`, four replicas. vLLM `--block-size=32`, prefix caching on. TrustyAI was `Managed` on `DataScienceCluster` `default-dsc`, with `eval.lmeval.permitOnline` and `permitCodeExecution` set to `allow`.

Each job is `local-completions` against

`http://openshift-ai-inference-openshift-default.openshift-ingress.svc.cluster.local/llm-d-repro/qwen-l4/v1/completions`

Task `arc_easy`, `limit: 24`, `batchSize: 1`, `num_concurrent: 1`, 0-shot. ARC-Easy is multiple choice, so 24 questions became 96 API requests, one per answer choice. The prompt template is `Question: {{question}}\nAnswer:`. Each question is different, and the shared text is shorter than a 16-token block.

`qwen-block32` and `qwen-block32-r2` ran with `blockSizeTokens: 32` and `autoTune: false`. `qwen-block16` and `qwen-block16-r2` used the same job spec after the service field was set to `blockSizeTokens: 16`. That 16 is the size this binary logs when the field is omitted. vLLM stayed at `--block-size=32`. Each repeat started only after a new scheduler pod was `2/2` and its own log showed the block size for that run.

| Job | Picker | State | API time | Requests | `acc` | `acc_norm` | `acc_stderr` | `acc_norm_stderr` |
|---|---|---|---|---|---|---|---|---|
| `qwen-block32` | 32 | Complete, Succeeded | 5s | 96/96 | 0.625 | 0.4583333333333333 | 0.101 | 0.104 |
| `qwen-block16` | 16 | Complete, Succeeded | 2s | 96/96 | 0.625 | 0.458 | 0.101 | 0.104 |
| `qwen-block32-r2` | 32 | Complete, Succeeded | 2s | 96/96 | 0.625 | 0.458 | 0.101 | 0.104 |
| `qwen-block16-r2` | 16 | Complete, Succeeded | 2s | 96/96 | 0.625 | 0.458 | 0.101 | 0.104 |

`qwen-block32` status recorded `Requesting API` `96/96`, elapsed `0:00:05`, `reason: Succeeded`, `acc,none` 0.625, and `acc_norm,none` 0.4583333333333333. Accuracy did not move between block sizes. Qwen3-0.6B on 24 ARC-Easy items is a routing probe.

`vllm:request_success_total` increased by 96 on each run. `vllm:prefix_cache_hits_total` and `vllm:prefix_cache_queries_total` stayed 0 on every pod. Pod order is `qwen-l4-kserve-6685bfc95b-26q8f`, `k9ptn`, `z82c2`, `zg8rj`.

| Job | Requests per pod | Share | Pods used | Prefix-cache hits |
|---|---|---|---|---|
| `qwen-block32` | 23, 19, 22, 32 | 24%, 20%, 23%, 33% | 4 of 4 | 0 |
| `qwen-block16` | 23, 14, 29, 30 | 24%, 15%, 30%, 31% | 4 of 4 | 0 |
| `qwen-block32-r2` | 33, 11, 26, 26 | — | 4 of 4 | 0 |
| `qwen-block16-r2` | 34, 25, 21, 16 | — | 4 of 4 | 0 |

Every replica received ARC-Easy traffic under both settings. The shared prompt text is shorter than one picker block, so this task does not show the scorer pin a pod.

## 16-token prefix load

48 `/v1/completions` prompts. The Qwen3 tokenizer encodes every prompt with the same first 16 tokens (`Hello rack rack …`, ids `9707, 29659 × 15`). The suffix `request N only` makes the full prompt 20 or 21 tokens, so the prompts do not share a 32-token block. Concurrency 4. vLLM `--block-size=32`. Queue weight 1, prefix weight 10. The query counter moved on the pods that received prompts. The hit counter did not.

| Run | Picker | Requests per pod (`26q8f`, `k9ptn`, `z82c2`, `zg8rj`) | Pods used | Prefix-cache hits |
|---|---|---|---|---|
| First | 16 | 28, 20, 0, 0 | 2 of 4 | 0 |
| First | 32 | 16, 10, 12, 10 | 4 of 4 | 0 |
| Repeat | 16 | 0, 0, 20, 28 | 2 of 4 | 0 |
| Repeat | 32 | 12, 12, 9, 15 | 4 of 4 | 0 |

At 16, the shared prefix is one complete picker block and two replicas kept the dataset. At 32, the same prefix is not a complete picker block and all four replicas moved. A 20-token prompt does not fill a vLLM block of 32, so hits stay 0 in both cases.

## Product-weight completions load

Queue weight 2, prefix weight 3, `blockSizeTokens: 32`, `autoTune: false`, vLLM `--block-size=32`. 24 `POST /v1/completions` requests per family, concurrency 4, `max_tokens` 1, temperature 0, model `Qwen/Qwen3-0.6B`. Hits are the increase in `vllm:prefix_cache_hits_total`. Families and pod counts are in [results.md](results.md).
