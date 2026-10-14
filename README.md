# kagent POC configuration

Config files for a read-only kagent proof of concept. Written and schema-checked against
**kagent v0.10.3** (API `kagent.dev/v1alpha2`) and **kagent-tools 0.3.0**.

> kagent v1.0 pre-releases move to `api.kagent.dev/v1alpha3` and change the Agent layout.
> If you install a 1.0 pre-release, these Agent/ModelConfig files need updating.

| File | What it does |
|---|---|
| `01-namespaces.yaml` | `kagent` (platform) and `demo-apps` (broken demo workloads) |
| `02-llm-secret.yaml` | LLM API key Secret (template; prefer the `kubectl create secret` command) |
| `values-kagent.yaml` | Helm values: model from Secret, read-only tool server, trimmed agent set |
| `03-modelconfig-openai.yaml` | ModelConfig for an external API (Option A) |
| `04-modelconfig-vllm.yaml` | Secret + ModelConfig for self-hosted vLLM (Option B, optional) |
| `05-agent-poc-troubleshooter.yaml` | Read-only troubleshooting agent with an evidence-first prompt |
| `06-demo-workloads.yaml` | OOM crash loop, bad image tag and CPU hog for the demo |

## Prerequisites

- `kubectl` access with cluster-admin (for the install only), Helm 3.8+ (OCI support)
- Prometheus reachable in-cluster (e.g. kube-prometheus-stack). Update the URL in
  `values-kagent.yaml` **and** in the agent's `systemMessage` in `05-…yaml`
- metrics-server, if you want `kubectl top`-style CPU/memory answers
- An OpenAI API key (Option A) or a vLLM endpoint serving a tool-calling model (Option B)

## Install order

```bash
# 1. Namespaces
kubectl apply -f 01-namespaces.yaml

# 2. LLM credentials (keeps the key out of files and Git)
kubectl create secret generic kagent-openai -n kagent \
  --from-literal=OPENAI_API_KEY="$OPENAI_API_KEY"

# 3. CRDs, then kagent. Keep the release name "kagent": the agent file references
#    the tool server as "kagent-tool-server", which is derived from it.
helm install kagent-crds oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds -n kagent
helm install kagent oci://ghcr.io/kagent-dev/kagent/helm/kagent \
  -n kagent -f values-kagent.yaml
#    To pin a version, check the published chart version first:
#    helm show chart oci://ghcr.io/kagent-dev/kagent/helm/kagent | grep '^version'

# 4. Model + agent
kubectl apply -f 03-modelconfig-openai.yaml
kubectl apply -f 05-agent-poc-troubleshooter.yaml

# 5. Demo workloads (only when you are ready to demo)
kubectl apply -f 06-demo-workloads.yaml
```

## Verify

```bash
kubectl get pods -n kagent                       # controller, ui, tools, agents Running
kubectl get modelconfigs,agents,remotemcpservers -n kagent
kubectl describe agent poc-troubleshooter -n kagent   # look for Ready / Accepted conditions

# Tool server really is read-only:
kubectl auth can-i delete pods -n demo-apps \
  --as=system:serviceaccount:kagent:kagent-tools          # expect: no
kubectl auth can-i list pods -n demo-apps \
  --as=system:serviceaccount:kagent:kagent-tools          # expect: yes
kubectl auth can-i get secrets -n kagent \
  --as=system:serviceaccount:kagent:kagent-tools          # expect: no

# Open the UI
kubectl port-forward -n kagent svc/kagent-ui 8080:8080   # then http://localhost:8080
```

If the tool server's ServiceAccount has a different name, find it with
`kubectl get sa -n kagent`. Likewise check `kubectl get svc -n kagent` if the UI
service name differs.

## Demo prompts (select poc-troubleshooter in the UI)

1. "Why is the oom-demo pod in demo-apps restarting?" — expect: exit code 137, OOMKilled,
   128Mi limit, and a proposed memory change as a YAML diff
2. "Why is the bad-image deployment in demo-apps not rolling out?" — expect: ImagePullBackOff,
   tag not found
3. "Which pods in demo-apps are using the most CPU, and are any being throttled?" — expect:
   cpu-hog, near its 500m limit
4. "Give me a health summary of the cluster." — expect: nodes, non-Running pods, warning events

## Switching to self-hosted vLLM

```bash
kubectl apply -f 04-modelconfig-vllm.yaml
kubectl patch agent poc-troubleshooter -n kagent --type merge \
  -p '{"spec":{"declarative":{"modelConfig":"vllm-llama"}}}'
```

The served model must support tool calling, and vLLM must be started with
`--enable-auto-tool-choice --tool-call-parser <parser>`. Without that the agent can
chat but cannot call any tools.

## Security choices in these files

- **Tool server is read-only at two layers.** The kagent-tools chart defaults to a
  cluster-admin role (`*` on everything). `rbac.readOnly: true` swaps that for
  get/list/watch, and `--read-only` stops write tools from being registered at all.
- **No Secret access.** `allowSecrets: false` and an explicit instruction in the prompt.
- **Stock k8s-agent disabled.** It ships `k8s_patch_resource`, `k8s_delete_resource`,
  `k8s_execute_command` and `k8s_apply_manifest`. The POC agent lists only read tools.
- **No `utils` tools.** That provider includes a `shell` tool.
- **Keys never in values or Git.** Helm only references the Secret.

## Known gotchas

- **Prometheus URL:** in kagent-tools 0.3.0 the Prometheus tools take the URL as a
  per-call argument and default to `localhost:9090`. That is why the agent prompt tells
  it which URL to pass. If metric queries fail, check this first.
- **Nodes:** the chart's read-only role omits nodes, so `values-kagent.yaml` adds them
  back for cluster health checks. That rule stops working if you switch the tool server
  to namespace-scoped RBAC (`rbac.namespaces`).
- **Phase 6 (remediation):** keep this agent read-only. For controlled fixes, prefer a
  GitOps flow (agent proposes a diff, a human merges, Argo CD applies). kagent also
  supports `requireApproval` on individual tools, which pauses for human sign-off,
  but that needs a separate, write-capable, namespace-scoped tool server.

## Clean up

```bash
kubectl delete -f 06-demo-workloads.yaml
helm uninstall kagent -n kagent && helm uninstall kagent-crds -n kagent
kubectl delete namespace demo-apps kagent
```
