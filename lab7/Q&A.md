# Lab 07 — Q&A: Resilience, Routing, Versioning, MCP, vLLM/llm-d & FinOps

Stack referenced throughout: **kagent** (Kubernetes-native agent framework — the `Agent` CRD = system prompt + tools + LLM config), **kgateway** (Kubernetes Gateway-API control plane) + **agentgateway** (Rust data-plane proxy for LLM/MCP/A2A traffic), **MCP** (Model Context Protocol), **FastMCP** (Python MCP framework), **vLLM** and **llm-d** (model serving / inference scheduling).

> Snippets are illustrative — field names follow the kgateway/agentgateway and kagent CRD shapes at time of writing; check your installed CRD version (`kubectl explain <kind>`) before applying.

---

## 1. How could we handle "agent got stuck" scenarios?

No single switch — you compose layers:

1. **Gateway-enforced timeouts at every hop.** A hung LLM or tool call is killed after N seconds and surfaces as an error the agent loop can handle, instead of hanging forever. Most reliable layer — doesn't depend on the agent's code behaving.
2. **Max-iteration / max-tool-call limits in the agent loop** so a "thinking in circles" agent terminates with a partial answer.
3. **Kubernetes liveness/readiness probes + restartPolicy** to restart a deadlocked pod.
4. **A2A / orchestrator-level deadline + cancellation** so a stuck child agent doesn't stall the parent.
5. **Observability to *detect* stuck:** per-step traces (step count, time-since-last-token); alert + auto-restart on "no new spans for X seconds."

(We hit exactly this in Lab 2: `qwen3.6` through the gateway timed out on model cold-load → empty A2A response, fixed by the timeout layer.)

**Layer 1 — timeout on the route to the model/MCP backend:**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: llm-route
  namespace: kagent
spec:
  parentRefs:
    - name: agentgateway
  rules:
    - backendRefs:
        - name: openai-backend
          group: gateway.kgateway.dev
          kind: Backend
      timeouts:
        request: 30s          # kill a hung LLM call after 30s
        backendRequest: 30s
```

**Layer 2 — bound the agent loop (kagent Agent CRD):**

```yaml
apiVersion: kagent.dev/v1alpha1
kind: Agent
metadata:
  name: ollama-agent
  namespace: kagent
spec:
  modelConfig: qwen3-config
  systemMessage: "You are an SRE assistant."
  maxIterations: 8            # stop reason ↔ tool looping after 8 cycles
  tools:
    - mcpServer: { name: everything-mcp }
```

**Layer 3 — liveness probe + restart on the agent pod:**

```yaml
livenessProbe:
  httpGet: { path: /healthz, port: 8080 }
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3        # watchdog fails /healthz when loop is stalled → kubelet restarts
restartPolicy: Always
```

**Layer 4 — caller-side deadline + cancellation (A2A / tool call, Python):**

```python
import asyncio

async def call_child_agent(client, task):
    try:
        return await asyncio.wait_for(client.run(task), timeout=60)  # hard deadline
    except asyncio.TimeoutError:
        await client.cancel(task.id)   # don't leak a stuck child
        return {"status": "timeout", "partial": None}
```

---

## 2. Any automatic timeout / circuit breaker patterns coming out of this framework?

Yes — primarily from the **gateway**, declaratively. kagent adds agent-loop guards + k8s patterns.

**Retries with a budget + circuit breaking + outlier detection on the Backend:**

```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: BackendConfigPolicy
metadata:
  name: llm-resilience
  namespace: kagent
spec:
  targetRefs:
    - group: gateway.kgateway.dev
      kind: Backend
      name: openai-backend

  # --- circuit breaking: reject fast when overloaded ---
  connectionPool:
    http:
      maxRequestsPerConnection: 100
    tcp:
      maxConnections: 50

  # --- outlier detection: eject an unhealthy/throttled endpoint ---
  outlierDetection:
    consecutive5xx: 5
    interval: 10s
    baseEjectionTime: 30s
    maxEjectionPercent: 50
```

**Retry policy on the route (capped so you don't amplify load):**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: llm-route, namespace: kagent }
spec:
  rules:
    - backendRefs: [{ name: openai-backend, group: gateway.kgateway.dev, kind: Backend }]
      retry:
        codes: [502, 503, 504]
        attempts: 2
        backoff: 200ms
      timeouts: { request: 30s }
```

---

## 3. How does kgateway handle model failover?

Through **priority groups** on the `Backend`: groups are ordered (failover priority = list order); within a group, models are load-balanced with **Power of Two Choices (P2C)** on health/latency/load; **outlier detection** ejects unhealthy endpoints; a single OpenAI-compatible front makes cross-provider failover possible.

```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: Backend
metadata:
  name: llm-failover
  namespace: kagent
spec:
  type: AI
  ai:
    # The LIST ORDER is the failover priority.
    priorityGroups:
      - providers:                         # priority 0 — cheap tier, P2C-balanced
          - name: openai-cheap
            openai:
              model: gpt-3.5-turbo
              authToken: { secretRef: { name: openai-secret } }
          - name: anthropic-cheap
            anthropic:
              model: claude-3-5-haiku-latest
              authToken: { secretRef: { name: anthropic-secret } }
      - providers:                         # priority 1 — premium fallback
          - name: openai-premium
            openai:
              model: gpt-4.1
              authToken: { secretRef: { name: openai-secret } }
          - name: anthropic-premium
            anthropic:
              model: claude-opus-4-1
              authToken: { secretRef: { name: anthropic-secret } }
```

---

## 4. Can we automatically switch from OpenAI → Claude → local model?

Yes — three ordered priority groups in one `Backend`. Client always hits the gateway's single OpenAI-compatible endpoint; no app change to add/reorder a fallback.

```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: Backend
metadata: { name: tiered-llm, namespace: kagent }
spec:
  type: AI
  ai:
    priorityGroups:
      - providers:                         # group 0 — OpenAI
          - name: openai
            openai:
              model: gpt-4o
              authToken: { secretRef: { name: openai-secret } }
      - providers:                         # group 1 — Anthropic Claude
          - name: claude
            anthropic:
              model: claude-sonnet-4-5
              authToken: { secretRef: { name: anthropic-secret } }
      - providers:                         # group 2 — local (Ollama / vLLM, OpenAI-compatible)
          - name: local-qwen
            openai:
              model: qwen3.6:35b           # must support tool-calling for agentic flows
              customHost: { host: ollama-llm.kagent.svc, port: 11434 }
```

Caveats: failover preserves *availability*, not answer quality; a fallback model in an agentic flow must also emit clean tool calls (Lab 2: `llama3.2` unreliable → `qwen3.6:35b`); auth/keys live in the gateway, not the agent.

---

## 5. Could we seamlessly handle the response formats from these providers?

Mostly yes — the gateway normalizes everything to the **OpenAI-compatible** schema, so the client sends and receives one shape regardless of backend:

```bash
# Client always speaks OpenAI format — gateway translates to whichever provider served it
curl http://agentgateway.kagent.svc/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
        "model": "tiered-llm",
        "messages": [{"role": "user", "content": "list pods in default"}],
        "tools": [{"type": "function",
                   "function": {"name": "list_pods", "parameters": {"type":"object"}}}]
      }'
```

Response envelope is consistent (`choices[].message.tool_calls`, `usage.prompt_tokens`, …) whether OpenAI, Claude, or local served it. Where "seamless" leaks: streaming chunk shapes & finish reasons, tool-call edge cases (OpenAI `tool_calls` vs Anthropic `tool_use`, parallel calls, partial JSON), provider-specific fields (`thinking`, `logprobs`), and token-usage accounting (matters for FinOps).

---

## 6. Can we version the agents built from kagent?

Yes — kagent is declarative CRDs in Git → GitOps-native versioning. Pin everything; avoid `:latest`.

```yaml
apiVersion: kagent.dev/v1alpha1
kind: Agent
metadata:
  name: ollama-agent
  namespace: kagent
  labels:
    app.kubernetes.io/version: "1.4.0"      # semver on the agent itself
  annotations:
    kagent.dev/prompt-revision: "2026-05-31-a"
spec:
  modelConfig: qwen3-config
  systemMessage: "You are an SRE assistant. (v1.4.0)"
---
apiVersion: kagent.dev/v1alpha1
kind: ModelConfig
metadata: { name: qwen3-config, namespace: kagent }
spec:
  provider: OpenAI
  model: "qwen3.6:35b"                       # pinned model+tag, not a floating alias
```

Managed by Flux (Lab 4's abox cluster already does this) so the whole release is versioned + rollback-able:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata: { name: kagent, namespace: kagent }
spec:
  chart:
    spec:
      chart: kagent
      version: "0.3.2"                        # pin chart version → git revert = rollback
```

---

## 7. Any blue/green or canary deployment patterns for agents?

Yes — weight `HTTPRoute` backends (canary/blue-green), or use Argo Rollouts/Flagger for automated analysis.

**Canary via Gateway-API weights (95/5):**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: agent-canary, namespace: kagent }
spec:
  parentRefs: [{ name: agentgateway }]
  rules:
    - backendRefs:
        - name: agent-v1
          port: 8080
          weight: 95
        - name: agent-v2          # new version — start at 5%, ramp up
          port: 8080
          weight: 5
```

**Automated canary analysis (Flagger) judged on agent quality, not just HTTP 200:**

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata: { name: ollama-agent, namespace: kagent }
spec:
  targetRef: { apiVersion: apps/v1, kind: Deployment, name: ollama-agent }
  analysis:
    interval: 1m
    stepWeight: 10
    threshold: 5
    metrics:
      - name: tool-call-error-rate          # agent-specific signal
        thresholdRange: { max: 2 }
        interval: 1m
      - name: request-success-rate
        thresholdRange: { min: 99 }
        interval: 1m
```

---

## 8. What's the fastmcp-python framework mentioned?

**FastMCP** — a high-level Python framework to build/consume MCP servers. Decorate a typed function → compliant tool with auto-generated JSON-Schema; supports stdio + HTTP/SSE.

```python
from fastmcp import FastMCP

mcp = FastMCP("sre-tools")

@mcp.tool()
def restart_deployment(namespace: str, name: str) -> str:
    """Restart a Kubernetes Deployment by name."""
    # ... call k8s API ...
    return f"restarted {namespace}/{name}"

@mcp.resource("logs://{pod}")
def pod_logs(pod: str) -> str:
    """Expose pod logs as an MCP resource."""
    return read_logs(pod)

if __name__ == "__main__":
    mcp.run(transport="http", host="0.0.0.0", port=8080)   # or transport="stdio"
```

Wired into a kagent agent as an MCP tool server:

```yaml
apiVersion: kagent.dev/v1alpha1
kind: MCPServer
metadata: { name: sre-tools, namespace: kagent }
spec:
  transport: http
  url: http://sre-tools.kagent.svc:8080
```

---

## 9. Is it the easiest path to MCP?

For **Python authoring**, effectively yes (basis of the official SDK's high-level API). Sometimes easier: reuse an existing server, use the TS SDK for Node, or **generate from an OpenAPI spec** — FastMCP can do the last one:

```python
import httpx
from fastmcp import FastMCP

# Turn an existing REST API into an MCP server — no hand-written tools
openapi_spec = httpx.get("https://api.example.com/openapi.json").json()
mcp = FastMCP.from_openapi(
    openapi_spec=openapi_spec,
    client=httpx.AsyncClient(base_url="https://api.example.com"),
)

if __name__ == "__main__":
    mcp.run(transport="http", port=8080)
```

---

## 10. About FinOps: how much control can I have?

A lot — the gateway is the chokepoint for every token, so it both **meters** and **enforces**. agentgateway ships budget/spend controls.

**Token-based rate limit + spend cap as gateway policy:**

```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
metadata: { name: llm-budget, namespace: kagent }
spec:
  targetRefs: [{ group: gateway.networking.k8s.io, kind: HTTPRoute, name: llm-route }]
  ai:
    # meter on tokens, not request count → aligns the limit with real cost
    rateLimit:
      tokens:
        requestsPerUnit: 100000     # 100k tokens
        unit: Minute
```

Plus Kubernetes FinOps for self-hosted models (HPA / scale-to-zero on the vLLM/Ollama Deployment).

---

## 11. Token-level / per-agent-level control

Yes — limits in token units, scoped by **caller identity** (per agent / API key / route). Different agents get different policies:

```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
metadata: { name: agent-b-tight, namespace: kagent }
spec:
  targetRefs: [{ group: gateway.networking.k8s.io, kind: HTTPRoute, name: agent-b-route }]
  ai:
    rateLimit:
      tokens: { requestsPerUnit: 10000, unit: Minute }   # batch agent → cheap, tight
    # match on identity injected upstream (e.g. header set per agent / API key)
    descriptors:
      - entries:
          - genericKey: { value: "agent-b" }
```

```yaml
# Premium customer-facing agent gets a larger budget + premium model route
metadata: { name: agent-a-premium }
spec:
  ai:
    rateLimit:
      tokens: { requestsPerUnit: 1000000, unit: Hour }
```

---

## 12. Can I implement custom cost controls?

Yes — gateway ext-proc/filters, Prometheus alerting, OPA admission, app-level caps, all as policy-as-code.

**Custom budget check via external processing (gateway calls your service before forwarding):**

```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
metadata: { name: budget-gate, namespace: kagent }
spec:
  targetRefs: [{ group: gateway.networking.k8s.io, kind: HTTPRoute, name: llm-route }]
  extProc:
    grpcService:
      backendRef: { name: budget-service, port: 9000 }   # your code: 200=ok, 429=over budget
```

**OPA/Gatekeeper: reject agents that use premium models without a budget annotation:**

```rego
package kagent.budget

violation[{"msg": msg}] {
  input.review.object.kind == "ModelConfig"
  startswith(input.review.object.spec.model, "gpt-4")
  not input.review.object.metadata.annotations["finops.dev/budget"]
  msg := "gpt-4-class model requires a finops.dev/budget annotation"
}
```

**Prometheus alert on token spend spike:**

```yaml
groups:
  - name: llm-finops
    rules:
      - alert: AgentTokenSpend
        expr: sum(rate(ai_gateway_tokens_total[5m])) by (agent) > 5000
        for: 10m
        labels: { severity: warning }
        annotations: { summary: "Agent {{ $labels.agent }} burning tokens" }
```

---

## 13. Per-agent budgets or depth / token limits

Two complementary axes: **budget over time** (gateway, Q11) and **depth per run** (agent loop).

**Depth caps inside the kagent Agent (bounds one task):**

```yaml
apiVersion: kagent.dev/v1alpha1
kind: Agent
metadata: { name: ollama-agent, namespace: kagent }
spec:
  modelConfig: qwen3-config
  maxIterations: 8            # cap reason→act→observe cycles
  maxToolCalls: 20           # cap total tool invocations per task
  a2aConfig:
    maxRecursionDepth: 3     # agent-calls-agent tree can't explode
---
apiVersion: kagent.dev/v1alpha1
kind: ModelConfig
metadata: { name: qwen3-config, namespace: kagent }
spec:
  model: "qwen3.6:35b"
  maxTokens: 2048            # cap output tokens per request
```

Budget (time) bounds aggregate cost; depth bounds a single task. Depth caps are the most effective guard against runaway agentic cost.

---

## 14. Is vLLM suitable for agents with many back-and-forth tool calls, or better for single-shot inference?

Well-suited to agentic/multi-turn — arguably more than single-shot — thanks to **continuous batching** + **prefix caching** (re-sent growing context isn't recomputed). Serves an OpenAI-compatible tool-calling API.

```bash
vllm serve Qwen/Qwen3-32B \
  --enable-prefix-caching \          # reuse KV for the repeated agent-loop prefix
  --enable-auto-tool-choice \        # tool/function calling for agents
  --tool-call-parser hermes \
  --max-num-seqs 256                 # continuous batching across concurrent requests
```

Plugs into the gateway as an OpenAI-compatible Backend:

```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: Backend
metadata: { name: vllm-backend, namespace: kagent }
spec:
  type: AI
  ai:
    priorityGroups:
      - providers:
          - name: vllm
            openai:
              model: Qwen/Qwen3-32B
              customHost: { host: vllm.kagent.svc, port: 8000 }
```

Caveats: per-step latency matters (each step waits on a tool); many distinct, non-overlapping contexts reduce prefix-cache benefit. The "vLLM = batch, so single-shot" intuition is backwards — its features shine when many requests share a long, repeated prefix (i.e. agents).

---

## 15. Does llm-d's scheduler help when an agent makes 15 LLM calls?

Yes — close to its sweet spot. llm-d adds a **KV-cache-aware scheduler** at the Gateway-API layer that routes all 15 (shared-prefix) calls to the *same* vLLM replica already holding the cache → no prompt recompute per call; plus load-aware balancing and prefill/decode disaggregation.

```yaml
# llm-d InferencePool with cache-aware scheduling (Gateway API Inference Extension)
apiVersion: inference.networking.x-k8s.io/v1alpha2
kind: InferencePool
metadata: { name: qwen-pool, namespace: llm-d }
spec:
  targetPortNumber: 8000
  selector: { app: vllm-qwen }
  extensionRef:
    name: llm-d-scheduler          # EPP: scores replicas by KV-cache hit + load
---
apiVersion: inference.networking.x-k8s.io/v1alpha2
kind: InferenceModel
metadata: { name: qwen3, namespace: llm-d }
spec:
  modelName: Qwen/Qwen3-32B
  poolRef: { name: qwen-pool }
  criticality: Standard
```

Caveats: benefit scales with **shared-prefix size** (15 calls, big common context → large win; 15 distinct prompts → mostly just smarter LB) and with **scale** (many sessions/replicas). For a single-replica setup (Lab 2 Ollama) it's overkill — plain vLLM (Q14) already gives the prefix-cache benefit.

---
