# Optimized baseline

The inline document is the `EndpointPickerConfig` from the upstream llm-d optimized-baseline router overlay (`guides/optimized-baseline/router/optimized-baseline.values.yaml` on `llm-d/llm-d`). Helm-only fields (`extraServicePorts`, `epp`, `modelServers`) are unknown on `spec.router` for both 3.4.4 and 3.5.1. A server-side dry-run left the object unchanged on both releases.

The filter keeps endpoints whose prefix-cache score is at least 0.80. The load gate reopens the pool when the best sticky endpoint's estimated time to first token exceeds the best other endpoint by more than `maxTTFTPenaltyMs`. Built-in defaults are `maxTTFTPenaltyMs: 18000` and `peakPrefillThroughput: 15928` tokens/s. Estimated TTFT in milliseconds is `inFlightTokens / peakPrefillThroughput × 1000`. The gate opens when the sticky pod is about **286,704** in-flight tokens ahead of an idle pod (`18000 / 1000 × 15928`). Those defaults are the Qwen3-32B, two-H100, TP=2 calibration named in the upstream file.

Plugin types in `/app/epp`:

| Type | 3.4.4 binary | 3.5.1 binary |
|---|---|---|
| `approx-prefix-cache-producer` | absent | present |
| `inflight-load-producer` | absent | present |
| `prefix-cache-affinity-filter` | absent | present |
| `token-load-scorer` | absent | present |

## OpenShift AI 3.4.4

The API stored the document and rendered it into the scheduler Deployment `--config-text`. The scheduler pod entered `CrashLoopBackOff`. Startup log:

```text
Failed to parse configuration
failed to decode configuration JSON/YAML: no kind "EndpointPickerConfig" is registered for version "llm-d.ai/v1alpha1"
```

The same plugins with `apiVersion: inference.networking.x-k8s.io/v1alpha1` were started as a throwaway pod on the same image and exited with:

```text
plugin instantiation failed: plugin type 'approx-prefix-cache-producer' is not registered
```

The 3.4.4 scheduler was restored afterward and returned to Ready.

## OpenShift AI 3.5.1

Two Qwen3-0.6B replicas. The controller kept the four plugins and added `metrics-data-source` with `scheme: https`. The scheduler loaded the config and the service returned to Ready.

```text
Filters: [prefix-cache-affinity-filter]
Scorers: [token-load-scorer: 1.000000]
Picker: max-score-picker
```

The YAML sets no parameters, so the process used built-in defaults and then adjusted two of them:

| Log | What it means |
|---|---|
| `Prefix DataProducer initialized` with `autoTune: true` and `blockSizeTokens: 16` | Producer default. The document does not set `blockSizeTokens` or `autoTune`. |
| `configured blockSizeTokens is below the recommended minimum, overriding it` `blockSizeTokens: 16` `minimum: 64` | The running window is 64. With the auto-created estimate tokenizer that is 256 bytes. |
| `auto-created default producer` `token-producer` | The matcher is the estimate backend. |
| `token-load-scorer` weight `1.00` | The profile listed the scorer with no weight. The process set 1. |
| `max-score-picker` | Injected. The YAML does not name `weighted-random-picker`. |

Counts are the increase in `POST /v1/completions` on the two pods. `max_tokens` 1, temperature 0. Client p50 on the first pair of families was 0.184 s (unique) and 0.195 s (shared). Nothing waited in vLLM.

### Before this document

Queue weight 2, prefix weight 3. The running producer was the 3.5.1 default (block size raised from 16 to 64, `autoTune: true`).

| Family | Requests | Concurrency | Pods |
|---|---|---|---|
| Unique prompts | 16 | 4 | 6, 10 |
| Shared prefix, 1860 characters | 16 | 4 | 16, 0 |

### With optimized baseline

| Family | Shared text | Requests | Concurrency | Pods |
|---|---|---|---|---|
| Unique prompts | none | 16 | 4 | 8, 8 |
| `willow` | 1860 characters | 16 | 4 | 16, 0 |
| `peacock` | 1960 characters, a different preamble | 16 | 4 | 0, 16 |
| `willow`, one request at a time | 1860 characters | 8 | 1 | 8, 0, the same pod as the concurrent `willow` wave |
| Short prefix | 72 bytes | 12 | 4 | 6, 6 |
| Prefix under one 256-byte block | 248 bytes | 12 | 4 | 8, 4 |
| Prefix over one 256-byte block | 336 bytes | 12 | 4 | 7, 5 |
| `willow` plus a unique tail | 1860-character prefix, 3160-character tail | 16 | 4 | 0, 16 |

Unique prompts used both pods. Each long prefix family stayed on one pod, and the two families used different pods. Prefixes shorter than one 256-byte block used both pods. The 336-byte prefix used both pods. The long `willow` prefix stayed on one pod, including when a longer unique tail was appended and when the requests were sent one at a time.

Requests finished in about a fifth of a second, so the in-flight token count stayed far under 286,704 and the 18-second load gate did not open.
