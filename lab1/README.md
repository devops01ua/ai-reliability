# Lab 1 — AgentGateway LLM Gateway with Local Ollama

This lab walks through installing [AgentGateway](https://agentgateway.dev) locally,
configuring it as an **LLM gateway** in front of a local
[Ollama](https://ollama.com) provider, accessing the bundled **dashboard UI**, and
exploring its fundamental **Backends** and **Policy** capabilities. The same config
also exposes an **MCP gateway** on the same port.

## Lab objectives

1. Install AgentGateway locally —
   [binary deployment docs](https://agentgateway.dev/docs/standalone/latest/deployment/binary/).
2. Choose an LLM provider —
   [LLM providers docs](https://agentgateway.dev/docs/standalone/latest/llm/providers/)
   (this lab uses Ollama via the OpenAI-compatible provider).
3. Configure `config.yaml` —
   [LLM gateway tutorial](https://agentgateway.dev/docs/standalone/latest/tutorials/llm-gateway/).
4. Run the gateway and access the UI at <http://localhost:15000/ui/>.
5. Verify LLM access and explore the fundamental **Backends** and **Policy** features.

## What's in this folder

| File | Description |
| --- | --- |
| `config.yaml` | AgentGateway configuration — LLM proxy + MCP server on port `3000`. |
| `agentgateway-ui-test.png` | Screenshot of the bundled AgentGateway dashboard UI. |
| `agent-gateway-invocation-logs-testing-olama.png` | Screenshot of invocation logs while proxying to Ollama. |
| `testing-agentgateway-with-olama.cast` | [asciinema](https://asciinema.org) recording of the end-to-end test session. |

## Configuration overview

`config.yaml` sets up a single bind on port `3000` with two capabilities:

1. **LLM proxy** (`llm:` block) — an OpenAI-compatible endpoint that forwards
   chat completion requests to a local Ollama server at `localhost:11434`.
   The `name: "*"` model entry accepts any model name and passes it through to
   Ollama.
2. **MCP gateway** (`backends.mcp`) — exposes the
   [`@modelcontextprotocol/server-everything`](https://www.npmjs.com/package/@modelcontextprotocol/server-everything)
   reference server over MCP via a stdio target launched with `npx`.

A **CORS policy** is attached to the route (`allowOrigins: "*"`) with the headers
required for MCP (`mcp-protocol-version`, `Mcp-Session-Id`) and standard content
negotiation — this is the **Policy** fundamental being explored. The **Backends**
fundamental is demonstrated by the two backend types wired into the route: an LLM
backend (Ollama) and an MCP backend (`server-everything`).

## Prerequisites

- [AgentGateway](https://agentgateway.dev) binary (see step 1)
- [Ollama](https://ollama.com) running locally on port `11434`
- [Node.js / npx](https://nodejs.org) (for the MCP `server-everything` target)
- A pulled Ollama model (e.g. `ollama pull llama3.2`)

## Running

1. **Start Ollama** and pull a model:

   ```bash
   ollama serve
   ollama pull llama3.2
   ```

2. **Start AgentGateway** with this config:

   ```bash
   agentgateway -f config.yaml
   ```

   The LLM listener comes up on port `3000`. The bundled dashboard UI is served
   on the admin port at <http://localhost:15000/ui/>, where you can inspect the
   configured Backends, Policies, and live invocation logs.

## Testing

### LLM (OpenAI-compatible) endpoint

```bash
curl http://localhost:3000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3.2",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

### MCP endpoint

MCP requests must include the SSE `Accept` header:

```bash
curl http://localhost:3000/ \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{...}}'
```

### Replaying the recorded session

```bash
asciinema play testing-agentgateway-with-olama.cast
```

## Notes & gotchas

- **`llm:` shorthand vs. explicit `binds`.** This config intentionally uses the
  top-level `llm:` shorthand for the LLM proxy. The explicit `binds`-form AI
  backend requires a concrete model name — `model: "*"` is rejected with an
  *"invalid model name"* error — so the shorthand is used to keep the wildcard
  pass-through to Ollama working.
- **Dashboard UI `forEach` bug.** A known AgentGateway bug
  ([#1958](https://github.com/agentgateway/agentgateway/issues/1958)) causes a
  `forEach` TypeError in the dashboard UI when the `llm:` shorthand is used,
  because the `/config` API response omits the `binds` key. The proxy itself
  functions correctly; the UI error is cosmetic and tracked for fix in PR #2013.
- The `llm:` shorthand is deprecated upstream; the `migrate` subcommand can
  convert it to the newer `frontendPolicies` form.
