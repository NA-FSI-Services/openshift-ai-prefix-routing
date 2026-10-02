# Affinity filter, peak 400

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

| What you send | Measured here |
|---|---|
| Unique prompts | Both pods. The split was 6 and 10. |
| Prefix under 384 bytes | Both pods. |
| Long prefix, requests overlapping | Both pods. `cedar` was 7 and 9. `pine` was 8 and 8. |
| Long prefix, 0.8 s idle between requests | One pod. `IDLE400` was 0 and 12. |
