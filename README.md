# OpenShift AI prefix-cache configurations

Lab measurements of `LLMInferenceService` scheduler documents on OpenShift AI **3.4.4** and **3.5.1**. The workload was Qwen3-0.6B on NVIDIA L4. Each directory is one configuration: the service manifest, the evaluation that produced the numbers, and the traffic result from the release it was run on.

`3.4-prefix-cache-scorer-32` is the configuration that has TrustyAI `LMEvalJob` objects (`arc_easy`, 24 examples, 96 completions). Those jobs ran at queue weight 1 and prefix weight 10. The other directories were checked with a `POST /v1/completions` load. Each directory's `eval.md` records the job parameters, and `results.md` records where the requests landed.

Checked 2026-10-01.

| | OpenShift AI 3.4.4 | OpenShift AI 3.5.1 |
|---|---|---|
| OpenShift | 4.20.39 | 4.20.39 |
| Operator | `rhods-operator.3.4.4` (`stable-3.4`) | `rhods-operator.3.5.1` (`stable-3.5`) |
| Accelerators | 4× NVIDIA L4 | 2× NVIDIA L4, then 4× NVIDIA L4 |
| Service | `qwen-l4`, 4 replicas | `qwen-l4`, 2 replicas, then 4 |
| vLLM | `registry.redhat.io/rhaii-early-access/vllm-cuda-rhel9:3.5.0-ea.2` (`0.21.0+rhaiv.10`) | `registry.redhat.io/rhaii/vllm-cuda-rhel9@sha256:c056e61672b6aea489ad5dde0bd2f8497230f5333e87f7cf6c494eba3bfdc808` (`0.24.0+rhaiv.13`) |
| Scheduler | `registry.redhat.io/rhoai/odh-llm-d-inference-scheduler-rhel9@sha256:8df63692a5d42f3c1e0b29ad59f303f1405faf553132aeca33471d2c0d195025` | `registry.redhat.io/rhoai/odh-llm-d-router-endpoint-picker-rhel9@sha256:14e57b536479e7d8b7d495000d8402ca780997994096277d131b56e6a9bcf27f` |

On 3.4.4, `blockSizeTokens` on `prefix-cache-scorer` is a character window (`blockSizeTokens × 4`). An incomplete character block is dropped. The 3.4.4 binary has `prefix-cache-scorer` and `queue-scorer`. It does not have `approx-prefix-cache-producer`, `prefix-cache-affinity-filter`, or the other 3.5 plugin types.

On 3.5.1, `blockSizeTokens` belongs on `approx-prefix-cache-producer`. With the auto-created estimate tokenizer (4 bytes per pseudo-token) that value is a byte window of the same width. A value below 64 is raised to 64. The 3.5.1 controller keeps the parameter object on the producer and drops it when it is left only on `prefix-cache-scorer`.

vLLM stores one KV block per `--block-size` tokens. A partial block is not stored. Hits below are the increase in `vllm:prefix_cache_hits_total`. Requests are `POST /v1/completions`, `max_tokens` 1, temperature 0, unless a row says otherwise. CPU and memory requests in the manifests are the L4 lab values. Replica count, image, `--block-size`, and the scheduler document are the values from that run.

## Configurations

| Directory | Release | Scheduler document | Measured result |
|---|---|---|---|
| [3.4-prefix-cache-scorer-32](3.4-prefix-cache-scorer-32/) | 3.4.4 | Scorer `blockSizeTokens: 32`, vLLM 32 | A 132-character prefix pins with a hit counter of 0. |
| [3.4-prefix-cache-scorer-64](3.4-prefix-cache-scorer-64/) | 3.4.4 | Scorer `blockSizeTokens: 64`, vLLM 32 | A 258-character, 29-token prefix pins with a hit counter of 0. |
| [3.4-prefix-cache-scorer-64-vllm-64](3.4-prefix-cache-scorer-64-vllm-64/) | 3.4.4 | Scorer 64, vLLM `--block-size=64` | 33-token and 37-token prefixes pin with a hit counter of 0. |
| [3.4-prefix-cache-scorer-96](3.4-prefix-cache-scorer-96/) | 3.4.4 | Scorer `blockSizeTokens: 96`, vLLM 32 | Under 384 characters the family uses all four pods. Over that window it sticks, and the hit counter moves. |
| [3.4-prefix-cache-scorer-144-vllm-64](3.4-prefix-cache-scorer-144-vllm-64/) | 3.4.4 | Scorer 144, vLLM 64 | Under 576 characters the family spreads, including a 70-token prefix. Over that window it sticks with real hits. |
| [3.5-optimized-baseline](3.5-optimized-baseline/) | 3.5.1 | Upstream optimized-baseline plugins, built-in defaults | A long repeated prefix stays on one pod, including a 40-request overlap on four replicas. One 1960-character family used two of those four pods. The same document crash-loops the 3.4.4 scheduler. |
| [3.5-affinity-filter-peak-15928](3.5-affinity-filter-peak-15928/) | 3.5.1 | Affinity filter, producer 96, peak 15928 | Unique prompts spread. A long preamble stays on one pod through 40 overlapping requests. The same pin held on four replicas. |
| [3.5-affinity-filter-peak-400](3.5-affinity-filter-peak-400/) | 3.5.1 | Same plugins, `peakPrefillThroughput: 400` | On two replicas, overlapping copies of a long preamble use both pods. On four replicas that peak still spreads a preamble of about 3200 characters or more, across two or three pods, and leaves an 1860-character preamble on one pod. A quiet repeated prefix stays on one pod. |
| [3.5-affinity-filter-peak-200](3.5-affinity-filter-peak-200/) | 3.5.1 | Same plugins, `peakPrefillThroughput: 200` | On four replicas, an 1860-character overlapping preamble uses two pods. A quiet 3200-character prefix stays on one pod. |
| [3.5-approx-prefix-cache-producer-64](3.5-approx-prefix-cache-producer-64/) | 3.5.1 | Producer `blockSizeTokens: 64`, estimate tokenizer, vLLM 32 | A 266-byte, 30-token prefix pins with a hit counter of 0. |

## Reading a run

- Unique prompts use the ready replicas.
- A repeated prefix shorter than the picker window uses the pool.
- A repeated prefix longer than the window sticks to one pod, or to two when the first concurrent wave lands on two, and the hit counter rises by one vLLM block for each completed block of the shared prefix.
- A stick whose hit counter stays at 0 is a picker match vLLM did not store.
