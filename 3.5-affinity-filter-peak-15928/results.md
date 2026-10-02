# Affinity filter, peak 15,928

The completions evaluation is in [eval.md](eval.md).

OpenShift AI **3.5.1**. Two Qwen3-0.6B replicas. vLLM left at `--block-size=32`. The model pods were not restarted.

`maxTTFTPenaltyMs: 2000` and `peakPrefillThroughput: 15928` open the pool when the sticky pod is **31,856** estimated tokens ahead of an idle pod (`2000 / 1000 × 15928`).

The scheduler logged `Picker: weighted-random-picker`, `blockSizeTokens: 96`, and `autoTune: false`. It did not raise the block size to 64. A second apply set `inFlightLoadProducerName: inflight-load-producer` and `prefixMatchInfoProducerName: approx-prefix-cache-producer` on the filter. The long families still pinned. Naming the producers did not move them.

Counts are `POST /v1/completions` on the two pods.

| Family | Requests | Pods |
|---|---|---|
| Unique prompts | 16 | 9, 7 |
| 72 bytes | 12 | 5, 7 |
| 248 bytes | 12 | 7, 5 |
| 301 bytes | 12 | 5, 7 |
| 477 bytes | 12 | 7, 5 |
| `willow`, 1860 characters, concurrency 4 | 16 | 0, 16 |
| `peacock`, 1960 characters | 16 | 0, 16, the same pod as `willow` |
| `willow`, one request at a time | 8 | 0, 8, that same pod |
| `willow` plus a 3160-character tail | 16 | 16, 0 |
| `birch`, 3680 characters, concurrency 4, producer names set | 16 | 0, 16 |
| `birch`, 40 requests, 50 ms apart, `max_tokens` 80, producer names set | 40 | 0, 40 |
| `aspen`, a second long preamble, producer names set | 16 | 16, 0 |

Unique prompts and prefixes under 384 bytes used both pods. Every preamble of 1860 characters or more stayed on one pod, including 40 overlapping requests. With one pod over the 0.80 affinity line, `weighted-random-picker` has one pod to draw.

## Four replicas

Same document, producer names omitted. Four L4 replicas. vLLM stayed at `--block-size=32`. The scheduler logged `weighted-random-picker`, `blockSizeTokens: 96`, `autoTune: false`, and `peakPrefillThroughput: 15928`. Counts are in one fixed pod order.

| Family | Requests | Pods |
|---|---|---|
| Unique prompts | 16 | 5, 5, 0, 6 |
| 72 bytes | 12 | 2, 3, 1, 6 |
| 248 bytes | 12 | 3, 5, 2, 2 |
| 336 bytes | 12 | 3, 5, 2, 2 |
| 477 bytes | 12 | 3, 3, 5, 1 |
| `willow`, 1860 characters, concurrency 4 | 16 | 16, 0, 0, 0 |
| `peacock`, 1960 characters | 16 | 0, 0, 0, 15 |
| `birch`, 3680 characters | 16 | 0, 16, 0, 0 |
| `aspen`, 3200 characters | 16 | 0, 0, 16, 0 |
| `willow`, one request at a time | 8 | 8, 0, 0, 0, the same pod as `willow` |
| `willow` plus a 3160-character tail | 16 | 0, 0, 0, 16 |
| `birch`, 40 requests, 50 ms apart, `max_tokens` 80 | 40 | 0, 40, 0, 0 |
| Idle, 3200 characters, 0.8 s apart | 12 | 0, 0, 12, 0 |

Every preamble of 1860 characters or more stayed on one pod, including 40 overlapping requests. The four long families used four different pods. One `peacock` request failed in the client before it was sent. Unique prompts used three of the four pods in this 16-request sample. Prefixes through 477 bytes used all four.
