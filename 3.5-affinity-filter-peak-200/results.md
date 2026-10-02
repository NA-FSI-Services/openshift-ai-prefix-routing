# Affinity filter, peak 200

The completions evaluation is in [eval.md](eval.md).

OpenShift AI **3.5.1**. Four Qwen3-0.6B replicas. vLLM left at `--block-size=32`. The model pods were not restarted.

`peakPrefillThroughput: 200` keeps the 2000 ms budget and opens the pool at **400** estimated tokens (`2000 / 1000 × 200`). The scheduler loaded that peak with `blockSizeTokens: 96`, `autoTune: false`, and `weighted-random-picker`.

[Peak 400](../3.5-affinity-filter-peak-400/) opens at 800 estimated tokens. On this four-replica pool that left an 1860-character overlapping family on one pod and spread a 3680-character family across three pods. At about 4 bytes per estimated token, 1860 characters is about 465 tokens, over a 400-token gate and under an 800-token gate.

Counts are `POST /v1/completions` in one fixed pod order.

| Family | Requests | Pods |
|---|---|---|
| Unique prompts | 16 | 4, 4, 3, 5 |
| 248 bytes | 12 | 5, 5, 2, 0 |
| `willow`, 1860 characters, concurrency 4 | 16 | 11, 0, 0, 5 |
| `peacock`, 1960 characters, concurrency 4 | 16 | 8, 0, 0, 8 |
| `birch`, 3680 characters, concurrency 4 | 16 | 7, 5, 4, 0 |
| `willow`, back to back, concurrency 1 | 8 | 5, 0, 0, 3 |
| Idle, 3200 characters, 0.8 s apart | 12 | 12, 0, 0, 0 |

The 1860-character family used two pods, including when the copies were sent one after another with no extra pause. The 3680-character family used three pods. The idle run stayed on one pod. `vllm:prefix_cache_hits_total` rose on the pods that received each family.

For another model, set the peak to `prompt_tokens / 2` at a 2000 ms budget from the preamble that has to spread. A 3,000-token prompt is a peak of 1,500. The number 200 is the rate that opened a 400-token gate for this 1860-character Qwen prompt. An open gate on this burst used two or three of the four replicas.

| What you send | Measured here |
|---|---|
| Unique prompts | All four pods. The split was 4, 4, 3, 5. |
| 248-byte prefix | Three pods. |
| 1860-character prefix, overlapping or back to back | Two pods. |
| 3680-character prefix, overlapping | Three pods. |
| 3200-character prefix, 0.8 s idle between requests | One pod. |
