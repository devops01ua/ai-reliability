# AI-SRE — Reliability Engineering with AI Agents on Kubernetes

A progressive, hands-on lab series for running, gatewaying, governing, and operating **AI agents** as production infrastructure. Each lab builds on the previous one, moving from a standalone LLM gateway to a full in-cluster AI stack with discovery, governance, and a vector database — and finishing with the design patterns that keep agentic workloads reliable and cost-controlled.

## Stack at a glance

| Layer | Technologies |
|-------|--------------|
| Agent framework | [kagent](https://kagent.dev) (`Agent` CRD = system prompt + tools + LLM config) |
| Gateways | **agentgateway** (Rust data-plane proxy for LLM / MCP / A2A) + **kgateway** (Kubernetes Gateway API control plane) |
| Inference | **Ollama** (local, `qwen3.6`), **vLLM**, **llm-d** (KV-cache-aware scheduling) |
| Protocols | **MCP** (Model Context Protocol), **A2A** (Agent-to-Agent + Agent Cards / Well-Known URI), OpenAI-compatible LLM API, Gateway API |
| Platform | Kubernetes (**kind**), **Flux** (GitOps) |
| Data & governance | **Qdrant** (vector DB), **agentregistry** Inventory, **MCP Security Governance** (MCPG) |

Most cluster labs run on a local **`kind` "abox"** cluster with kagent + Flux installed.

## Labs

### [Lab 1 — AgentGateway LLM Gateway with Local Ollama](./lab1)
Install AgentGateway locally, configure it as an **LLM gateway** in front of a local Ollama provider, explore the dashboard UI, and use the same config to expose an **MCP gateway** on the same port. Covers Backends, Policies (CORS/routing), and invocation logs.
*Standalone — no cluster required.*

### [Lab 2 — Connecting Ollama to Kagent](./lab2)
Connect a local Ollama model to **Kagent** on a kind cluster, deploy a declarative **MCP tool server**, and create an **Agent** that uses both. Demonstrates in-cluster model routing (`Agent → Gateway → AgentgatewayBackend → host Ollama`) via the Kubernetes Gateway API, the `MCPServer` CRD (stdio transport), and the `Agent` CRD. Six manifests applied in order, verified with live A2A `message/send` calls.

### [Lab 4 — Agent + Agent Card, AI Inventory, MCP Governance, Qdrant](./lab4)
A full AI-infrastructure slice on `kind-abox`:
- **`rightsizer-agent`** — a custom kagent Agent that finds underutilized nodes and over-requested workloads, serving an **Agent Card** at the A2A **Well-Known URI**.
- **agentregistry Inventory** — auto-discovers all AI resources via the `DiscoveryConfig` CRD (8 agents, 1 MCP server, 3 models; the custom agent shows as **Running/Deployed**).
- **MCP Governance (MCPG)** — evaluates MCP security posture (**score 100, Compliant**).
- **Qdrant** — vector database deployed via the official Helm chart, with a demo collection.

### [Lab 7 — Design Q&A: Resilience, Routing, Versioning, MCP, vLLM/llm-d & FinOps](./lab7)
A reference document (no executable code) covering 15 design questions: composing timeout layers, circuit breakers, model/provider failover, response normalization, agent versioning (GitOps), blue/green & canary rollouts, FastMCP, FinOps token budgets and per-agent cost controls, and the advantages of vLLM and the llm-d scheduler for agentic flows.

## Repository layout

```
ai-sre/
├── README.md                  # This file
├── lab1/                      # AgentGateway + Ollama (standalone)
│   ├── README.md
│   ├── config.yaml            # LLM + MCP gateway config
│   ├── *.png                  # Dashboard / invocation-log screenshots
│   └── *.cast                 # asciinema demo recording
├── lab2/                      # Ollama → Kagent on kind
│   ├── README.md
│   ├── kind-config.yaml
│   ├── 00-namespace.yaml … 04-ollama-agent.yaml   # 6 ordered manifests
│   └── screenshots/
├── lab4/                      # Agent Card + Inventory + MCPG + Qdrant
│   ├── README.md
│   ├── rightsizer-agent/      # Manifests + evidence/ (agent card, run logs)
│   └── screenshots/           # 01-agent-card, 02-inventory, 03-mcpg, 04-qdrant
└── lab7/                      # Design patterns Q&A
    └── Q&A.md
```

Each lab has its own `README.md` with full setup, apply order, and evidence (screenshots).

## Learning outcomes

- Build, gateway, and orchestrate AI agents on Kubernetes.
- Route LLM and MCP traffic through a gateway with policies and failover.
- Register agents for discovery via **Agent Cards** and the **agentregistry** Inventory.
- Govern MCP security posture and enforce **FinOps** cost controls at the gateway.
- Compose reliability patterns — timeouts, retries, circuit breakers, canary rollouts.
- Integrate MCP tools declaratively and serve scalable inference with vLLM / llm-d.
