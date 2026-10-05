# Shared preamble on a four-replica Gemma service

`arc_easy` does not carry a long shared prefix. A TrustyAI `LMEvalJob` with `taskNames: [arc_easy]` sends `Question: {{question}}\nAnswer:`. Each question is different, and that wrapper is shorter than one picker block, so a block-size change cannot pin the job onto one pod. Leave that job unchanged and use it as the control.

The pin measurement is a separate `/v1/completions` load. Every prompt starts with the same preamble. A unique line follows it. Post `prompt` to `/v1/completions` so a chat template is not inserted in front of that text.

Run this before changing the scheduler. vLLM stays at `--block-size=32` with prefix caching on. Four replicas have to be Ready.

Set the namespace and service name for the Gemma deployment under test:

```bash
export NS=<namespace>
export SVC=<LLMInferenceService name>
```

## 1. Confirm the service

```bash
oc get llminferenceservice "$SVC" -n "$NS" \
  -o jsonpath='ready={.status.conditions[?(@.type=="Ready")].status}{"\n"}replicas={.spec.replicas}{"\n"}url={.status.url}{"\n"}model={.spec.model.name}{"\n"}auth={.metadata.annotations.security\.opendatahub\.io/enable-auth}{"\n"}args={.spec.template.containers[?(@.name=="main")].env[?(@.name=="VLLM_ADDITIONAL_ARGS")].value}{"\n"}'

oc get pods -n "$NS" -l "app.kubernetes.io/name=$SVC" \
  --field-selector=status.phase=Running \
  -o custom-columns=NAME:.metadata.name,READY:.status.containerStatuses[0].ready,NODE:.spec.nodeName
```

`VLLM_ADDITIONAL_ARGS` has to contain `--block-size=32` and `--enable-prefix-caching`. If `replicas` is under 4, raise it and wait until four pods are Ready:

```bash
oc patch llminferenceservice "$SVC" -n "$NS" --type=merge -p '{"spec":{"replicas":4}}'
```

```bash
export URL=$(oc get llminferenceservice "$SVC" -n "$NS" -o jsonpath='{.status.url}')
export MODEL=$(oc get llminferenceservice "$SVC" -n "$NS" -o jsonpath='{.spec.model.name}')
echo "$URL"
echo "$MODEL"
```

The model pod listens on HTTPS port 8000. When `VLLM_API_KEY` is set in that container, `/tokenize` and `/metrics` need it. The gateway client below does not send a bearer token when `security.opendatahub.io/enable-auth` is `false`.

## 2. Build one shared preamble

Repeat one paragraph until the file is at least 1,860 characters. That is the shortest long preamble that stuck to one pod in the lab. The unique suffix is added later, so the prompts are not identical.

```bash
python3 << 'PY'
from pathlib import Path
sentence = (
    "Shared preamble for this routing check. "
    "This text is identical on every request. "
    "The line after it is the only text that changes. "
)
text = ""
while len(text) < 1860:
    text += sentence
Path("preamble.txt").write_text(text, encoding="utf-8")
print("characters", len(text))
print("bytes", len(text.encode()))
PY
```

## 3. Count tokens on the served model

A character count from another tokenizer does not transfer. Ask a Ready pod of this service. Grow the file until `count` is at least 32. vLLM stores one KV block per `--block-size` tokens, and a partial block is not stored.

```bash
pod() {
  oc get pod -n "$NS" -l "app.kubernetes.io/name=$SVC" \
    --field-selector=status.phase=Running \
    -o jsonpath='{.items[0].metadata.name}'
}

tokenize() {
  python3 - "$MODEL" preamble.txt << 'PY'
import json, sys
from pathlib import Path
model, path = sys.argv[1], sys.argv[2]
Path("/tmp/preamble-tokenize.json").write_text(json.dumps({
    "model": model,
    "prompt": Path(path).read_text(encoding="utf-8"),
}))
PY
  local name
  name=$(pod)
  oc cp /tmp/preamble-tokenize.json "$NS/$name:/tmp/preamble-tokenize.json" -c main
  oc exec -n "$NS" "$name" -c main -- bash -c \
    'hdr=(); if [ -n "${VLLM_API_KEY:-}" ]; then hdr=(-H "Authorization: Bearer $VLLM_API_KEY"); fi; curl -sk https://127.0.0.1:8000/tokenize -H "Content-Type: application/json" "${hdr[@]}" --data-binary @/tmp/preamble-tokenize.json'
}
```

```bash
tokenize
```

Read `count` in the JSON. If it is under 32, append the same sentence and tokenize again:

```bash
python3 << 'PY'
from pathlib import Path
sentence = (
    "Shared preamble for this routing check. "
    "This text is identical on every request. "
    "The line after it is the only text that changes. "
)
Path("preamble.txt").write_text(Path("preamble.txt").read_text(encoding="utf-8") + sentence, encoding="utf-8")
text = Path("preamble.txt").read_text(encoding="utf-8")
print("characters", len(text))
print("bytes", len(text.encode()))
PY
tokenize
```

Stop at the first `count` of 32 or more. Record characters, bytes, and `count`.

For a second, longer load, keep appending until `count` is a few thousand. A service started with `--max-model-len=100000` can hold that. Send the 32-token file first. Raise the client timeout to 600 seconds for the longer file. Prefill of a few thousand tokens is slow, and `max_tokens` stays 1 so generation does not add to that wait.

## 4. Make 24 prompts that share that prefix

```bash
python3 - "$MODEL" << 'PY'
import json, sys
from pathlib import Path
model = sys.argv[1]
preamble = Path("preamble.txt").read_text(encoding="utf-8")
prompts = [
    {"id": i, "prompt": preamble + f"\nRequest {i} only. Reply with one word."}
    for i in range(24)
]
Path("preamble-load.json").write_text(json.dumps({
    "model": model,
    "prompts": prompts,
}, indent=2))
print("prompts", len(prompts), "shared_characters", len(preamble))
PY
```

Confirm the shared token prefix. Tokenize prompt 0 and prompt 1 with the same `/tokenize` call, substituting each `prompt` into the JSON body. The leading token ids match through the preamble. They diverge at `Request N only`.

## 5. Record counters, then send the load

The counters are cumulative. Save this output before the run.

```bash
oc get pods -n "$NS" -l "app.kubernetes.io/name=$SVC" \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' \
| while read -r name; do
    echo "== $name"
    oc exec -n "$NS" "$name" -c main -- bash -c \
      'hdr=(); if [ -n "${VLLM_API_KEY:-}" ]; then hdr=(-H "Authorization: Bearer $VLLM_API_KEY"); fi; curl -sk https://127.0.0.1:8000/metrics "${hdr[@]}"' \
      | grep -E '^vllm:request_success_total|^vllm:prefix_cache_hits_total|^vllm:prefix_cache_queries_total'
  done
```

Send 24 completions at concurrency 4, `max_tokens` 1, temperature 0:

```bash
python3 - "$URL" preamble-load.json << 'PY'
import json, ssl, sys
from concurrent.futures import ThreadPoolExecutor, as_completed
from pathlib import Path
from urllib import request, error

base, dataset_path = sys.argv[1], sys.argv[2]
endpoint = base.rstrip("/") + "/v1/completions"
dataset = json.loads(Path(dataset_path).read_text())
ctx = ssl._create_unverified_context()

def post(prompt):
    body = json.dumps({
        "model": dataset["model"],
        "prompt": prompt,
        "max_tokens": 1,
        "temperature": 0,
    }).encode()
    req = request.Request(
        endpoint,
        data=body,
        headers={"Content-Type": "application/json"},
        method="POST",
    )
    with request.urlopen(req, timeout=600, context=ctx) as resp:
        resp.read()
        return resp.status

ok = failed = 0
with ThreadPoolExecutor(max_workers=4) as pool:
    futures = [pool.submit(post, item["prompt"]) for item in dataset["prompts"]]
    for fut in as_completed(futures):
        try:
            fut.result()
            ok += 1
        except (error.URLError, TimeoutError) as exc:
            failed += 1
            print("request failed:", exc, file=sys.stderr)
print(f"completed={ok} failed={failed} total={ok + failed}")
sys.exit(0 if failed == 0 else 1)
PY
```

## 6. Subtract the counters

Run the metrics command from step 5 again. One pod taking all 24, or two pods when the first concurrent wave lands on two before the index updates, is the pin. The other pods stay at their previous totals.

`vllm:prefix_cache_hits_total` rising means vLLM stored the preamble. The counter staying at 0 means the shared prefix is still under 32 tokens on this tokenizer.

`POST /v1/completions` lines in the model pod log also show which replica accepted the request. They do not show whether the prefix was stored. Keep the hit counter either way.

Run the existing `arc_easy` job as it is. All four pods should move, and the hit counter should stay flat. That job is not edited.

Leave the scheduler as it is until this pin shows up.
