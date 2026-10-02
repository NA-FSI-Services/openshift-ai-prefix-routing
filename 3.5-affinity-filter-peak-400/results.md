# Affinity filter, peak 400

The completions evaluation is in [eval.md](eval.md).

OpenShift AI **3.5.1**. Two Qwen3-0.6B replicas. vLLM left at `--block-size=32`. The model pods were not restarted.

`peakPrefillThroughput: 400` keeps the 2000 ms budget and opens the pool at **800** estimated tokens (`2000 / 1000 × 400`). The scheduler loaded that peak. A later pod loaded the same peak with the producer name fields omitted. Both spreads below are from that gate. The producer name is not what changed the result.

The number 400 is the Qwen3-0.6B rate that opened an 800-token gate on this pair. For another model, set `peakPrefillThroughput` to `prompt_tokens / (maxTTFTPenaltyMs / 1000)` from one real prompt, so the gate opens while a single request is in flight. At a 2000 ms budget that is `prompt_tokens / 2`. A 3,000-token prompt is a peak of 1,500.

On the scheduler pod, `parsed config` has to name `weighted-random-picker`, the producer log has to say `blockSizeTokens: 96` and `autoTune: false`, and the loaded filter parameters have to show the peak that was set. This document stays off the 3.4.4 scheduler. That binary does not contain these plugins.

Counts are `POST /v1/completions` on the two pods.

| Family | Requests | Pods |
|---|---|---|
| Unique prompts | 16 | 6, 10 |
| `cedar`, 3800 characters, concurrency 4 | 16 | 7, 9 |
| `cedar`, back to back, concurrency 1 | 12 | 5, 7 |
| `pine`, a second 3760-character preamble, concurrency 4 | 16 | 8, 8 |
| `IDLE400`, 3200 characters, 12 requests with 0.8 s of idle time between them | 12 | 0, 12 |
| Same shape, producer names omitted, 3030 characters, concurrency 4 | 16 | 9, 7 |

A long preamble sent while another copy is in flight uses both pods. Two different long preambles both use both pods. The idle run still uses one pod, so a quiet repeated prefix keeps its cache target.

## Four replicas

Same plugins and the same peak, producer names omitted. Four L4 replicas. vLLM stayed at `--block-size=32`. The scheduler logged `weighted-random-picker`, `blockSizeTokens: 96`, `autoTune: false`, and `peakPrefillThroughput: 400`. The two-replica rows above used preambles of 3030–3800 characters. This rerun also sent the 1860-character `willow` family.

| Family | Requests | Pods |
|---|---|---|
| Unique prompts | 16 | 4, 3, 6, 3 |
| 72 bytes | 12 | 5, 3, 3, 1 |
| 248 bytes | 12 | 3, 4, 2, 3 |
| 336 bytes | 12 | 2, 1, 4, 5 |
| 477 bytes | 12 | 3, 3, 4, 2 |
| `willow`, 1860 characters, concurrency 4 | 16 | 0, 16, 0, 0 |
| `peacock`, 1960 characters | 16 | 16, 0, 0, 0 |
| `birch`, 3680 characters, concurrency 4 | 16 | 0, 5, 7, 4 |
| `aspen`, 3200 characters, concurrency 4 | 16 | 0, 10, 0, 6 |
| `willow`, back to back, concurrency 1 | 8 | 0, 8, 0, 0 |
| `willow` plus a 3160-character tail | 16 | 4, 5, 7, 0 |
| `birch`, 40 requests, 50 ms apart, `max_tokens` 80 | 40 | 0, 11, 17, 12 |
| Idle, 3200 characters, 0.8 s apart | 12 | 0, 12, 0, 0 |

Unique prompts and prefixes through 477 bytes used all four pods. The 1860-character and 1960-character overlapping families stayed on one pod. At about 4 bytes per estimated token those prompts are under the 800-token gate. A 3200-character family used two pods, and a 3680-character family used three, including 40 overlapping requests with `max_tokens` 80. One replica stayed at 0 on that 40-request row. The idle run stayed on one pod.

The peak that spread the 1860-character family on this pool is 200, in [3.5-affinity-filter-peak-200](../3.5-affinity-filter-peak-200/). 400 remains the rate that opens an 800-token gate.

| What you send | Measured here |
|---|---|
| Unique prompts | Both pods. The split was 6 and 10. |
| Prefix under 384 bytes | Both pods. |
| Long prefix, requests overlapping | Both pods. `cedar` was 7 and 9. `pine` was 8 and 8. |
| Long prefix, 0.8 s idle between requests | One pod. `IDLE400` was 0 and 12. |
