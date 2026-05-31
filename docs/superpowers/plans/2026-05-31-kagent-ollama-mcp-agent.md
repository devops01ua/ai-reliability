# Lab2: Kagent + Ollama + Declarative MCP Server + Agent — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire the host's local Ollama (`llama3.2`) into Kagent as a usable model via an in-cluster AgentGateway, deploy a declarative `server-everything` MCP tool server, and create a Kagent agent that uses both — all in a new `lab2` namespace on the running `abox` kind cluster.

**Architecture:** Agent → ModelConfig (`openAI.baseUrl` → in-cluster `ollama-llm` Gateway) → AgentGatewayBackend (OpenAI provider, `host.docker.internal:11434`) → host Ollama. Tools flow Agent → `everything-mcp` MCPServer pod (stdio/npx). Each layer is a declarative manifest applied in dependency order and verified by its `Accepted`/`Ready` status condition.

**Tech Stack:** Kubernetes (kind), Kagent `0.7.23` (`agents`, `modelconfigs`, `mcpservers` CRDs), agentgateway (`gateways`, `httproutes`, `agentgatewaybackends`), Ollama, `@modelcontextprotocol/server-everything` (npx).

**Verified field facts (do not guess — these were checked against the live CRDs):**
- `MCPServer` is `kagent.dev/v1alpha1`. Fields: `spec.deployment.{cmd,args,port,image}`, `spec.transportType: stdio`, `spec.stdioTransport: {}`.
- `Agent`/`ModelConfig` are `kagent.dev/v1alpha2`.
- `Agent.spec`: `description` (top-level), `declarative.{modelConfig, systemMessage, stream, tools[], a2aConfig, deployment}`.
- `Agent.spec.declarative.tools[]`: `type: McpServer`, `mcpServer.{name, kind, apiGroup, toolNames[]}`.
- `ModelConfig.spec`: `provider: OpenAI`, `model`, `apiKeySecret`, `apiKeySecretKey`, `openAI.baseUrl`.
- `AgentgatewayBackend` is `agentgateway.dev/v1alpha1`: `spec.ai.groups[].providers[]` with `name`, `openai: {}`, `host`, `path`.
- A Gateway provisions a Service named after it; reachable at `<gw-name>.<ns>.svc.cluster.local`.
- Working dir for all commands: `/Users/ahrechanychenko/Documents/DevOps01/ai-sre/lab2`.
- Cluster context is `kind-abox`.

**Note on "tests":** This is declarative infra, not unit-tested code. Each task's "failing test" is applying the manifest and reading back the resource's status condition (and, for the model path, an end-to-end curl). The rhythm is: write manifest → apply → verify status/behavior → commit.

---

## Task 0: Preconditions check

**Files:** none (verification only)

- [ ] **Step 1: Confirm context, namespace-absence, and Ollama reachability**

Run:
```bash
kubectl config current-context
kubectl get ns lab2 2>&1 | grep -q NotFound && echo "lab2 absent (good)" || echo "lab2 exists"
curl -sf http://localhost:11434/api/tags >/dev/null && echo "ollama OK" || echo "ollama UNREACHABLE"
ollama list | grep -q 'llama3.2' && echo "llama3.2 present" || echo "MISSING llama3.2"
```
Expected: context `kind-abox`; `lab2 absent (good)`; `ollama OK`; `llama3.2 present`.

If `lab2 exists`, stop and reconcile with the user before continuing.

---

## Task 1: Namespace

**Files:**
- Create: `lab2/00-namespace.yaml`

- [ ] **Step 1: Write the manifest**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lab2
  labels:
    app.kubernetes.io/part-of: ai-sre-lab2
```

- [ ] **Step 2: Apply**

Run: `kubectl apply -f lab2/00-namespace.yaml`
Expected: `namespace/lab2 created`.

- [ ] **Step 3: Verify**

Run: `kubectl get ns lab2`
Expected: `lab2   Active`.

- [ ] **Step 4: Commit**

```bash
git add lab2/00-namespace.yaml
git commit -m "Lab2: add lab2 namespace"
```

---

## Task 2: Ollama Gateway + AgentGatewayBackend + HTTPRoute

**Files:**
- Create: `lab2/01-ollama-gateway-backend.yaml`

This is the in-cluster proxy that turns OpenAI-compatible requests into calls to the host's Ollama. Three resources in one file.

- [ ] **Step 1: Write the manifest**

```yaml
# Dedicated LLM gateway for the Ollama OpenAI-compatible proxy.
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ollama-llm
  namespace: lab2
spec:
  gatewayClassName: agentgateway
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same
---
# AI backend: forwards OpenAI-compatible chat requests to host Ollama.
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: ollama-backend
  namespace: lab2
spec:
  ai:
    groups:
    - providers:
      - name: ollama
        openai: {}              # model name passed through from the request
        host: host.docker.internal:11434
        path: /v1
---
# Route all traffic on the gateway to the Ollama backend.
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: ollama-route
  namespace: lab2
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: ollama-llm
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - group: agentgateway.dev
      kind: AgentgatewayBackend
      name: ollama-backend
```

- [ ] **Step 2: Apply**

Run: `kubectl apply -f lab2/01-ollama-gateway-backend.yaml`
Expected: gateway, agentgatewaybackend, and httproute all `created`.

- [ ] **Step 3: Verify the Gateway is programmed**

Run: `kubectl get gateway ollama-llm -n lab2 -o jsonpath='{.status.conditions[?(@.type=="Programmed")].status}{"\n"}'`
Expected: `True` (may take ~15s; re-run if empty).

- [ ] **Step 4: Verify the Gateway provisioned a Service**

Run: `kubectl get svc -n lab2 -l gateway.networking.k8s.io/gateway-name=ollama-llm`
Expected: a Service (ClusterIP) exposing port 80. This confirms `ollama-llm.lab2.svc.cluster.local:80` will resolve.

- [ ] **Step 5: Verify the HTTPRoute is accepted**

Run: `kubectl get httproute ollama-route -n lab2 -o jsonpath='{.status.parents[0].conditions[?(@.type=="Accepted")].status}{"\n"}'`
Expected: `True`.

- [ ] **Step 6: Commit**

```bash
git add lab2/01-ollama-gateway-backend.yaml
git commit -m "Lab2: add Ollama LLM gateway, backend, and route"
```

---

## Task 3: End-to-end model reachability test (Ollama through the gateway)

**Files:** none (behavioral verification)

This proves the entire model path works *before* Kagent is involved — isolating gateway/Ollama issues from Kagent config issues.

- [ ] **Step 1: Curl the gateway's OpenAI endpoint from inside the cluster**

Run:
```bash
kubectl run llm-probe -n lab2 --rm -i --restart=Never --image=curlimages/curl:8.10.1 -- \
  -sS -X POST http://ollama-llm.lab2.svc.cluster.local/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"llama3.2","messages":[{"role":"user","content":"Reply with the single word: pong"}],"stream":false}'
```
Expected: a JSON chat-completion response whose `choices[0].message.content` contains text from the model (e.g. "pong"). The pod auto-deletes (`--rm`).

- [ ] **Step 2: If it fails — diagnose host.docker.internal**

If the curl errors with a DNS/connection failure to `host.docker.internal`, run the fallback probe to confirm it's the host-resolution issue, then apply the documented mitigation:
```bash
# Find the docker bridge gateway IP usable from kind nodes:
docker network inspect kind -f '{{(index .IPAM.Config 0).Gateway}}'
```
Mitigation: edit `lab2/01-ollama-gateway-backend.yaml`, set the backend `host:` to `<that-ip>:11434`, re-apply, and re-run Step 1. Record the change in the README (Task 6, gotchas). Do NOT proceed to Task 4 until Step 1 returns a model completion.

- [ ] **Step 3: No commit** (verification only; any manifest change committed under Task 2's file).

---

## Task 4: Dummy Secret + Ollama ModelConfig

**Files:**
- Create: `lab2/02-ollama-modelconfig.yaml`

- [ ] **Step 1: Write the manifest**

`stringData` keeps the throwaway key readable; Ollama ignores auth but Kagent requires the secret reference.

```yaml
# Throwaway API key — Ollama ignores auth, but Kagent's ModelConfig requires it.
apiVersion: v1
kind: Secret
metadata:
  name: ollama-dummy-key
  namespace: lab2
type: Opaque
stringData:
  OPENAI_API_KEY: "ollama-no-auth-required"
---
apiVersion: kagent.dev/v1alpha2
kind: ModelConfig
metadata:
  name: ollama-model-config
  namespace: lab2
spec:
  provider: OpenAI
  model: llama3.2
  apiKeySecret: ollama-dummy-key
  apiKeySecretKey: OPENAI_API_KEY
  openAI:
    baseUrl: http://ollama-llm.lab2.svc.cluster.local/v1
```

- [ ] **Step 2: Apply**

Run: `kubectl apply -f lab2/02-ollama-modelconfig.yaml`
Expected: secret and modelconfig `created`.

- [ ] **Step 3: Verify the ModelConfig is accepted**

Run: `kubectl get modelconfig ollama-model-config -n lab2 -o jsonpath='{.status.conditions[?(@.type=="Accepted")].status}{" "}{.status.conditions[?(@.type=="Accepted")].message}{"\n"}'`
Expected: `True` with a "Model configuration accepted"-style message.

- [ ] **Step 4: Commit**

```bash
git add lab2/02-ollama-modelconfig.yaml
git commit -m "Lab2: add Ollama ModelConfig and dummy key secret"
```

---

## Task 5: Declarative MCP tool server (server-everything)

**Files:**
- Create: `lab2/03-everything-mcpserver.yaml`

- [ ] **Step 1: Write the manifest**

`server-everything` is a Node package launched with `npx`; `image` provides the Node runtime the stdio sidecar runs the command in. `port` is where the KMCP transport adapter exposes the MCP HTTP endpoint.

```yaml
apiVersion: kagent.dev/v1alpha1
kind: MCPServer
metadata:
  name: everything-mcp
  namespace: lab2
spec:
  transportType: stdio
  stdioTransport: {}
  deployment:
    image: node:22-alpine
    port: 3000
    cmd: npx
    args:
    - "-y"
    - "@modelcontextprotocol/server-everything"
```

- [ ] **Step 2: Apply**

Run: `kubectl apply -f lab2/03-everything-mcpserver.yaml`
Expected: `mcpserver.kagent.dev/everything-mcp created`.

- [ ] **Step 3: Wait for the pod (npx installs on first start)**

Run:
```bash
kubectl rollout status deploy -n lab2 -l kagent.dev/mcpserver=everything-mcp --timeout=180s 2>/dev/null \
  || kubectl get pods -n lab2 -w
```
Expected: the MCPServer pod reaches `Running`/`Ready`. If the label selector finds nothing, list pods (`kubectl get pods -n lab2`) to find the everything-mcp pod and wait on it by name. First start can take a minute while `npx` fetches the package.

- [ ] **Step 4: Verify the MCPServer is accepted and tools are discovered**

Run:
```bash
kubectl get mcpserver everything-mcp -n lab2 -o jsonpath='{.status.conditions[?(@.type=="Accepted")].status}{"\n"}'
kubectl get mcpserver everything-mcp -n lab2 -o jsonpath='{.status}{"\n"}' | tr ',' '\n' | grep -iE 'tool|echo|add' || true
```
Expected: first command prints `True`. The second surfaces discovered tool names if the status exposes them (informational — `echo` and `add` should appear among them).

- [ ] **Step 5: If the pod crashloops on `npx`/network**

`npx -y` needs outbound network on first run. If the pod fails to pull the package, check logs: `kubectl logs -n lab2 -l kagent.dev/mcpserver=everything-mcp --tail=50`. Record the failure mode; if the cluster has no egress, the fallback is to switch `image` to a prebuilt MCP everything image — note this in the README rather than guessing an image here. Do not proceed until the MCPServer is `Accepted: True`.

- [ ] **Step 6: Commit**

```bash
git add lab2/03-everything-mcpserver.yaml
git commit -m "Lab2: add declarative server-everything MCP tool server"
```

---

## Task 6: The Agent

**Files:**
- Create: `lab2/04-ollama-agent.yaml`

- [ ] **Step 1: Write the manifest**

Mirrors the `k8s-agent` declarative pattern (a2aConfig.skills, deployment resources), points at the Ollama ModelConfig, and selects two safe `server-everything` tools (`echo`, `add`).

```yaml
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: ollama-agent
  namespace: lab2
spec:
  description: A demo agent backed by local Ollama (llama3.2) with the server-everything MCP tools.
  declarative:
    modelConfig: ollama-model-config
    stream: true
    systemMessage: |
      You are a helpful assistant running on a local Ollama model.
      You have access to MCP tools from the server-everything reference server.
      Use the `echo` tool to repeat text back, and the `add` tool to sum two numbers.
      Prefer using a tool when the user's request maps directly to one.
    tools:
    - type: McpServer
      mcpServer:
        kind: MCPServer
        apiGroup: kagent.dev
        name: everything-mcp
        toolNames:
        - echo
        - add
    a2aConfig:
      skills:
      - id: echo-text
        name: Echo Text
        description: Repeat a provided message back to the user using the MCP echo tool.
        tags:
        - echo
        - demo
        examples:
        - "Echo: hello world"
        - "Repeat this back to me: testing 123"
      - id: add-numbers
        name: Add Numbers
        description: Add two numbers using the MCP add tool.
        tags:
        - math
        - demo
        examples:
        - "What is 2 + 2?"
        - "Add 17 and 25."
    deployment:
      resources:
        requests:
          cpu: 100m
          memory: 256Mi
        limits:
          cpu: 1000m
          memory: 1Gi
```

- [ ] **Step 2: Apply**

Run: `kubectl apply -f lab2/04-ollama-agent.yaml`
Expected: `agent.kagent.dev/ollama-agent created`.

- [ ] **Step 3: Verify the Agent is accepted and ready**

Run:
```bash
kubectl get agent ollama-agent -n lab2
kubectl get agent ollama-agent -n lab2 -o jsonpath='{.status.conditions[?(@.type=="Accepted")].status}{" / Ready="}{.status.conditions[?(@.type=="Ready")].status}{"\n"}'
```
Expected: the `get` shows `READY True` / `ACCEPTED True`; the jsonpath prints `True / Ready=True`.

- [ ] **Step 4: Verify the agent pod is running**

Run: `kubectl get pods -n lab2 -l kagent.dev/agent=ollama-agent`
Expected: an `ollama-agent-*` pod `Running` and `1/1`. (If the selector label differs, find it via `kubectl get pods -n lab2`.)

- [ ] **Step 5: Commit**

```bash
git add lab2/04-ollama-agent.yaml
git commit -m "Lab2: add ollama-agent using Ollama model + MCP tools"
```

---

## Task 7: End-to-end agent verification + README

**Files:**
- Create: `lab2/README.md`

- [ ] **Step 1: Exercise the agent over A2A (model + tool path)**

The Kagent controller serves each agent over A2A at `:8083/api/a2a/<ns>/<name>`. Port-forward and send a message that forces a tool call.

Run (in one shell, backgrounded):
```bash
kubectl -n kagent port-forward svc/kagent-controller 8083:8083 >/tmp/pf.log 2>&1 &
sleep 3
```
Then probe the agent card to confirm it's served:
```bash
curl -sf http://localhost:8083/api/a2a/lab2/ollama-agent/.well-known/agent-card.json | head -c 400; echo
```
Expected: a JSON agent card naming `ollama-agent` and its skills. (If the path differs by Kagent version, fall back to verifying via the Kagent UI in Step 2.)

- [ ] **Step 2: Verify in the Kagent UI (authoritative manual check)**

Run:
```bash
kubectl -n kagent port-forward svc/kagent-ui 8080:8080 >/tmp/pf-ui.log 2>&1 &
sleep 3
echo "Open http://localhost:8080 and select agent 'ollama-agent' (namespace lab2)."
```
Manual check: chat with `ollama-agent`. Confirm (a) it replies — proving the Ollama model connection through the gateway — and (b) ask "Add 17 and 25" / "Echo: hello" and confirm it invokes the MCP tool and returns the result. Stop the port-forwards afterward (`kill %1 %2` or `pkill -f port-forward`).

- [ ] **Step 3: Write the README**

```markdown
# Lab 2 — Connecting Ollama to Kagent: Model, Declarative MCP Server, and Agent

This lab connects a local [Ollama](https://ollama.com) model to
[Kagent](https://kagent.dev) running on a kind cluster, deploys a **declarative
MCP tool server**, and creates an **agent** that uses both.

## Architecture

```
Agent (ollama-agent)
  ├── modelConfig → ModelConfig (ollama-model-config)
  │     openAI.baseUrl → http://ollama-llm.lab2.svc.cluster.local/v1
  │        → Gateway (ollama-llm) → HTTPRoute → AgentgatewayBackend (ollama-backend)
  │           → host.docker.internal:11434  (host Ollama, llama3.2)
  └── tools → MCPServer (everything-mcp)  [stdio: npx @modelcontextprotocol/server-everything]
```

Model traffic flows Agent → in-cluster AgentGateway → host Ollama. Tool traffic
flows Agent → the `everything-mcp` pod.

## Files (apply in order)

| File | Resources |
| --- | --- |
| `00-namespace.yaml` | `Namespace/lab2` |
| `01-ollama-gateway-backend.yaml` | `Gateway/ollama-llm`, `AgentgatewayBackend/ollama-backend`, `HTTPRoute/ollama-route` |
| `02-ollama-modelconfig.yaml` | `Secret/ollama-dummy-key`, `ModelConfig/ollama-model-config` |
| `03-everything-mcpserver.yaml` | `MCPServer/everything-mcp` |
| `04-ollama-agent.yaml` | `Agent/ollama-agent` |

## Prerequisites

- The `abox` kind cluster running with Kagent `0.7.23` installed.
- Ollama running on the host at `:11434` with `llama3.2` pulled (`ollama pull llama3.2`).

## Apply

```bash
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-ollama-gateway-backend.yaml
kubectl apply -f 02-ollama-modelconfig.yaml
kubectl apply -f 03-everything-mcpserver.yaml
kubectl apply -f 04-ollama-agent.yaml
# Or, after the namespace exists: kubectl apply -f .
```

## Verify

```bash
kubectl get gateway,httproute,agentgatewaybackend,modelconfig,mcpserver,agent -n lab2
```
All should report `Accepted`/`Ready: True`.

End-to-end model path (from inside the cluster):
```bash
kubectl run llm-probe -n lab2 --rm -i --restart=Never --image=curlimages/curl:8.10.1 -- \
  -sS -X POST http://ollama-llm.lab2.svc.cluster.local/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"llama3.2","messages":[{"role":"user","content":"say pong"}],"stream":false}'
```

Agent (UI):
```bash
kubectl -n kagent port-forward svc/kagent-ui 8080:8080
# open http://localhost:8080, pick ollama-agent (lab2), ask "Add 17 and 25" and "Echo: hello"
```

## Notes & gotchas

- **`host.docker.internal`** resolves to the host on Docker Desktop. If the
  agentgateway pod cannot resolve it, set the backend `host:` to the kind docker
  bridge gateway IP from `docker network inspect kind -f '{{(index .IPAM.Config 0).Gateway}}'`
  and re-apply.
- **Dummy API key** — Ollama needs no auth, but Kagent's `ModelConfig` requires an
  `apiKeySecret`; `ollama-dummy-key` holds a throwaway value.
- **MCP first start** — `npx -y` fetches `server-everything` on first run, so the
  `everything-mcp` pod may take ~1 min to become Ready and needs cluster egress.
- **CRD versions** — `MCPServer` is `kagent.dev/v1alpha1`; `Agent`/`ModelConfig`
  are `kagent.dev/v1alpha2`; `AgentgatewayBackend` is `agentgateway.dev/v1alpha1`.
```

If the host.docker.internal fallback was applied in Task 3, update the architecture diagram and the first gotcha to reflect the actual `host:` value used.

- [ ] **Step 4: Commit**

```bash
git add lab2/README.md
git commit -m "Lab2: add README documenting Ollama+MCP+agent setup"
```

---

## Done criteria

- `kubectl get gateway,httproute,agentgatewaybackend,modelconfig,mcpserver,agent -n lab2` → every resource `Accepted`/`Ready: True`.
- The in-cluster curl (Task 3) returns a model completion from Ollama.
- In the Kagent UI, `ollama-agent` responds to chat (model works) and invokes an MCP tool for "add"/"echo" prompts (tool server works).
- All five manifests + README committed to `lab2/`.
