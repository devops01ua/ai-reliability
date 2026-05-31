# Lab 4 — RightSizer SRE Agent

Custom declarative **kagent Agent** that finds **underutilized nodes** and **over-requested workloads**, serves an **Agent Card** at the A2A Well-Known URI, and is registered in the **agentregistry** Inventory via the `DiscoveryConfig` CRD (the controller auto-creates the linked `AgentCatalog` entry that shows the agent as Running/Deployed).

> Runs in namespace **`kagent`** (the agent references the `kagent-tool-server` RemoteMCPServer, whose tool refs are namespace-local — so the agent and a copy of the Ollama `ModelConfig` live alongside it in `kagent`).

## Components

| File | Resource | Purpose |
|------|----------|---------|
| `00-modelconfig.yaml` | `Secret` + `kagent.dev/v1alpha2 ModelConfig` | Ollama (qwen3.6) model config in ns `kagent` |
| `10-agent.yaml` | `kagent.dev/v1alpha2 Agent` | The agent: model, read-only k8s tools, a2aConfig skills |
| `30-discoveryconfig.yaml` | `agentregistry.dev/v1alpha1 DiscoveryConfig` | Auto-discovers + registers the agent in the Inventory (the "upload via CRD") |

## Apply

```bash
kubectl --context kind-abox apply -f 00-modelconfig.yaml
kubectl --context kind-abox apply -f 10-agent.yaml
kubectl --context kind-abox get agent rightsizer-agent -n kagent      # READY/ACCEPTED True
kubectl --context kind-abox apply -f 30-discoveryconfig.yaml
kubectl --context kind-abox get agentcatalog kagent-rightsizer-agent -n agentregistry \
  -o custom-columns='NAME:.metadata.name,PUBLISHED:.status.published,READY:.status.deployment.ready,STATUS:.status.status'
# kagent-rightsizer-agent   true   true   active   (Running / Deployed)
```

## Fetch the Agent Card (Well-Known URI)

```bash
kubectl --context kind-abox port-forward -n kagent svc/kagent-controller 8083:8083 &
curl -s http://localhost:8083/api/a2a/kagent/rightsizer-agent/.well-known/agent.json | python3 -m json.tool
```

Both `/.well-known/agent.json` and `/.well-known/agent-card.json` return the card (name, description, url, capabilities, and the two skills).

## Run it (A2A message/send)

```bash
kubectl --context kind-abox port-forward -n kagent svc/kagent-controller 8083:8083 &
curl -s -X POST http://localhost:8083/api/a2a/kagent/rightsizer-agent/ \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":"1","method":"message/send","params":{"message":{"role":"user","parts":[{"kind":"text","text":"Analyze kind-abox: list underutilized nodes and over-requested workloads."}],"messageId":"m1"}}}' \
  | python3 -m json.tool
```

## Model & tools

- Model: `ollama-model-config` (qwen3.6, local Ollama) — reused from Lab 2, copied into ns `kagent`.
- Tools: `kagent-tool-server` RemoteMCPServer, read-only subset (`k8s_get_resources`, `k8s_describe_resource`, `k8s_get_events`, `k8s_get_cluster_configuration`, `k8s_get_pod_logs`).
- No metrics-server on kind-abox → analysis is **requests-vs-allocatable**.

## Skills (Agent Card)

- **detect-underutilized-nodes** — low CPU/memory utilization vs capacity; consolidation candidates.
- **detect-overrequested-workloads** — requests greatly exceed real usage; right-sizing recommendations.

## Evidence

See `evidence/`:

- `agent-status.txt` — Agent `READY=True, ACCEPTED=True` + full YAML.
- `agent-card.json` — Agent Card fetched from the Well-Known URI.
- `rightsizer-run.txt` — live A2A run: the agent's right-sizing report on kind-abox.

The Inventory listing and the discovered `kagent-rightsizer-agent` entry
(Running/Deployed) are captured under `../screenshots/02-inventory/`.
