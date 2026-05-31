# Lab2 — Connect Ollama to Kagent + Declarative MCP Tool Server + Agent

**Date:** 2026-05-31
**Status:** Approved (design)

## Goal

Complete the three-part Lab2 task on the running `abox` kind cluster:

1. **Connect a model** — wire the local Ollama provider into Kagent as a usable model.
2. **Create a declarative MCP tool server** — a Kagent `MCPServer` running the
   `@modelcontextprotocol/server-everything` reference server.
3. **Create an agent** in Kagent that uses the Ollama model and the MCP tools.

## Environment (verified)

- Cluster: `kind-abox`, 3 nodes, Kagent `0.7.23` installed in namespace `kagent`.
- Controllers running: `kagent-controller`, `kagent-kmcp-controller-manager` (reconciles
  `MCPServer` CRs), `kagent-tools`, plus the agentgateway controller
  (`agentgateway.dev/agentgateway`).
- CRDs available: `agents.kagent.dev`, `modelconfigs.kagent.dev`, `mcpservers.kagent.dev`
  (`v1alpha2`); `agentgatewaybackends.agentgateway.dev` (`v1alpha1`); standard Gateway API
  (`gateways`, `httproutes`).
- Existing Gateway `agentgateway-external` (gatewayClassName `agentgateway`, port 80). Its
  LoadBalancer EXTERNAL-IP is `<pending>` (no MetalLB in kind) — irrelevant here because
  the model traffic stays in-cluster via service DNS.
- Each Gateway provisions a Service named after it, selected by
  `gateway.networking.k8s.io/gateway-name=<name>`, reachable at
  `<gateway-name>.<namespace>.svc.cluster.local`.
- Ollama running on the host at `:11434` with `llama3.2:latest` pulled and reachable.
- `default-model-config` (OpenAI / `gpt-4.1-mini`) exists but is **not** used here.

## Architecture & data flow

```
┌─────────────── kind cluster (abox), namespace: lab2 ───────────────┐      host
│                                                                     │
│  Agent (ollama-agent, declarative)                                  │
│    │ modelConfig                          │ tools                   │
│    ▼                                       ▼                         │
│  ModelConfig (ollama-model-config)        MCPServer (everything-mcp)│
│    provider: OpenAI                         stdio: npx              │
│    model: llama3.2                          server-everything ──────┤ (pod)
│    openAI.baseUrl:                                                  │
│      http://ollama-llm.lab2.svc.cluster.local/v1                    │
│    │ (OpenAI-compatible HTTP)                                       │
│    ▼                                                                │
│  Gateway (ollama-llm, agentgateway class, :80)                      │
│    │ HTTPRoute (ollama-route): / → backend                          │
│    ▼                                                                │
│  AgentGatewayBackend (ollama-backend)                               │
│    ai.providers[]: openai, host: host.docker.internal:11434 ────────┼──► Ollama :11434
│                    path: /v1                                        │      (llama3.2)
└─────────────────────────────────────────────────────────────────────┘
```

Model traffic flows **Agent → in-cluster AgentGateway → Ollama** (the chosen "via
AgentGateway in-cluster" path). MCP tool traffic flows **Agent → MCPServer pod**.

## Decisions

| Decision | Choice | Rationale |
| --- | --- | --- |
| Ollama reachability | Via in-cluster AgentGateway | Exercises the gateway/Backend pattern; the interesting part of the lab. |
| Model | `llama3.2` | Pulled locally; referenced in Lab1. |
| MCP server | `server-everything` (reference) | Known-good declarative tool set; mirrors Lab1. |
| Namespace | `lab2` (new) | Isolation; all resources self-contained → no ReferenceGrants. |
| Gateway | Dedicated `ollama-llm` | Clean separation of the LLM proxy from the kagent UI/API gateway. |

## Components & file layout

All manifests in `lab2/`, numbered for `kubectl apply` dependency order:

```
lab2/
├── kind-config.yaml                  # existing — unchanged
├── 00-namespace.yaml                 # Namespace: lab2
├── 01-ollama-gateway-backend.yaml    # Gateway (ollama-llm) + AgentGatewayBackend + HTTPRoute
├── 02-ollama-modelconfig.yaml        # Secret (dummy OPENAI_API_KEY) + ModelConfig
├── 03-everything-mcpserver.yaml      # MCPServer (server-everything, stdio)
├── 04-ollama-agent.yaml              # Agent → modelConfig + MCP tools
└── README.md                         # Lab2 writeup (objectives, apply order, testing)
```

### 00 — Namespace
`Namespace/lab2`. All other resources are created here.

### 01 — Gateway + AgentGatewayBackend + HTTPRoute
- `Gateway/ollama-llm`: `gatewayClassName: agentgateway`, one HTTP listener on port 80,
  `allowedRoutes.namespaces.from: Same`.
- `AgentgatewayBackend/ollama-backend` (`agentgateway.dev/v1alpha1`):
  `spec.ai.groups[].providers[]` with `name: ollama`, `openai: {}` (model taken from
  request), `host: host.docker.internal:11434`, `path: /v1`.
- `HTTPRoute/ollama-route`: `parentRef` → `ollama-llm`, match `PathPrefix: /`,
  `backendRef` → `AgentgatewayBackend ollama-backend`.

### 02 — Secret + ModelConfig
- `Secret/ollama-dummy-key`: holds `OPENAI_API_KEY` with a throwaway value. Ollama ignores
  auth, but Kagent's `ModelConfig` requires `apiKeySecret` + `apiKeySecretKey`.
- `ModelConfig/ollama-model-config` (`kagent.dev/v1alpha2`): `provider: OpenAI`,
  `model: llama3.2`, `apiKeySecret: ollama-dummy-key`, `apiKeySecretKey: OPENAI_API_KEY`,
  `openAI.baseUrl: http://ollama-llm.lab2.svc.cluster.local/v1`.

### 03 — MCPServer
- `MCPServer/everything-mcp` (`kagent.dev/v1alpha2`): stdio transport,
  `cmd: npx`, `args: [-y, @modelcontextprotocol/server-everything]`. The KMCP controller
  reconciles it into a pod and registers its tools.

### 04 — Agent
- `Agent/ollama-agent` (`kagent.dev/v1alpha2`, declarative): `spec.declarative.modelConfig:
  ollama-model-config`, a short `systemMessage`, `tools` referencing `everything-mcp`
  (selecting a couple of safe tools, e.g. `echo`, `add`), and an `a2aConfig.skills` block
  mirroring the `k8s-agent` pattern. Standard `deployment.resources` requests/limits.

## Error handling & known risks

- **`host.docker.internal` resolution from kind pods.** On Docker Desktop this resolves to
  the host. If the agentgateway pod cannot resolve it, fall back to the Docker bridge host
  IP (or add a `--add-host` / hostAliases). Verified mitigation step in testing.
- **`server-everything` first-start latency.** `npx` installs the package on first run; the
  MCPServer pod may take a moment to become Ready. Wait and re-check.
- **Dummy secret requirement.** Kagent rejects a `ModelConfig` without `apiKeySecret`; the
  throwaway value is intentional and documented.
- **Reconciliation status.** Each layer exposes an `Accepted`/`Ready` condition — verify
  every one before declaring success.

## Testing

1. **End-to-end model reachability** — from inside a cluster pod, curl
   `http://ollama-llm.lab2.svc.cluster.local/v1/chat/completions` with a `llama3.2`
   chat request; expect a completion from Ollama.
2. **Resource health** — `kubectl get gateway,httproute,agentgatewaybackend,modelconfig,
   mcpserver,agent -n lab2` all show `Accepted/Ready: True`.
3. **Agent in UI** — open the Kagent UI, chat with `ollama-agent`; confirm (a) it responds
   (model connection works) and (b) it can invoke an MCP tool such as `echo` (tool server
   works).

## Out of scope

- Modifying the existing `kagent`-namespace agents or `default-model-config`.
- Host port mappings / external LoadBalancer exposure (traffic stays in-cluster).
- Building a custom MCP server (using the reference `server-everything`).
