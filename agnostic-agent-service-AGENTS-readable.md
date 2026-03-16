# AGENTS.md — AI Agent Service

This document defines engineering constraints for AI agents and human contributors working on
Node.js-based AI agent services. It extends the project-agnostic enterprise baseline with
agent-specific rules covering inference calls, tool execution, loop safety, grounding, and
session lifecycle.

The project-agnostic baseline governs anything not addressed here. Where this file conflicts
with the baseline, this file wins.

## Authority and Scope

- This file is the canonical source for contributor behavior on this service.
- Any shorter instruction file (for IDE assistants, bots, etc.) MUST NOT weaken or contradict this file.
- If guidance is condensed, it must still explicitly retain:
  1. Agent loop safety rules (maxSteps, timeout, runaway prevention),
  2. Tool contract and grounding rules (no state invented by the model),
  3. Inference call resilience (timeout, retry classification, circuit breaker),
  4. Session lifecycle and audit trail requirements,
  5. Prompt and system prompt management rules,
  6. Error differentiation and deterministic error handling,
  7. Security and secret-handling constraints,
  8. Rationale-focused code comment standard.

---

## Architecture and Separation of Concerns

The agent service is an **inference orchestration process**, not an HTTP server.
Keep its layers clean:

- **Entry point** (`index.js`): accept user input, start session, call agent, return result. No business logic.
- **Agent loop** (`agent.js`): owns `generateText` call, tool registry injection, `maxSteps` policy. No tool implementation.
- **Tools** (`tools/*.js`): one file per domain (nodes, vms, storage). Each tool calls the API client. No direct HTTP construction.
- **API client** (`client/api.js`): owns all HTTP to downstream services. Applies session ID header, handles envelope unwrapping, classifies errors. No inference logic.
- **Config** (`config.js`): all environment-backed config. No defaults that are unsafe for production.

WHY: the agent loop, tool execution, and downstream HTTP are three distinct failure domains.
Keeping them separate means a tool call failure does not require changes to the inference layer,
and a model change does not require changes to tool implementations.

---

## Agent Loop Safety

### maxSteps is mandatory

Every `generateText` / `streamText` call MUST set `maxSteps` explicitly.
Never omit it or set it to `Infinity`.

```js
// WRONG
await generateText({ model, tools, prompt });

// CORRECT
await generateText({ model, tools, maxSteps: config.agent.maxSteps, prompt });
```

Default: `10`. Reduce for simple single-tool queries. Never exceed `25` without explicit justification.

WHY: without a step cap, a model that repeatedly selects the wrong tool or misreads a tool
result will loop indefinitely, hammering downstream APIs and exhausting VRAM.

### Per-session timeout

Every agent invocation MUST be wrapped in a timeout. If inference + tool calls do not complete
within the budget, abort and surface a clear timeout error to the caller.

```js
const AGENT_TIMEOUT_MS = config.agent.timeoutMs ?? 30_000;

const result = await Promise.race([
  runAgentLoop(prompt, sessionId),
  new Promise((_, reject) =>
    setTimeout(() => reject(new Error('AGENT_TIMEOUT')), AGENT_TIMEOUT_MS)
  ),
]);
```

WHY: a slow or hung Ollama inference pass will otherwise block the process indefinitely.

### No mutable shared state in the agent loop

The agent loop MUST be stateless across invocations. Session ID, tool instances, and conversation
history are created fresh per invocation.

- Do NOT store tool state, conversation history, or model context in module-level variables.
- Do NOT cache inference results across sessions.

WHY: shared mutable state causes one session's failures to corrupt another's, and makes
runaway loop detection unreliable.

---

## Tool Contract

### One action per tool

Each tool MUST represent exactly one, clearly bounded action against one endpoint.
Do NOT build multi-step or conditional logic into a tool's `execute` function.

```js
// WRONG — tool does two things
execute: async ({ nodeId }) => {
  const status = await apiGet(`/api/nodes/${nodeId}`);
  const vms = await apiGet(`/api/nodes/${nodeId}/vms`);
  return { status, vms };
}

// CORRECT — separate tools
listNodeVMs: tool({ ... execute: async ({ nodeId }) => apiGet(`/api/nodes/${nodeId}/vms`, sessionId) })
getNodeStatus: tool({ ... execute: async ({ nodeId }) => apiGet(`/api/nodes/${nodeId}`, sessionId) })
```

WHY: multi-step tools obscure which step failed, prevent the model from selecting granularly,
and make retry classification ambiguous.

### Tool descriptions are load-bearing

Tool `description` fields are not documentation — they are the model's only signal for tool
selection. Treat them with the same discipline as a public API contract.

Rules:
- Describe what data the tool returns, not just what it does.
- Include the natural language phrases a user might ask that should trigger this tool.
- If two tools are semantically adjacent, explicitly differentiate them in their descriptions.
- Never use vague verbs: "get", "fetch", "retrieve" are acceptable only with a precise object noun.

```js
// WEAK
description: 'Get node info.'

// CORRECT
description: 'Returns CPU usage, memory utilization, disk usage, uptime, and online/offline status for a single named Proxmox node. Use when asked about a specific node by name.'
```

WHY: Mistral 7B and similar small models rely heavily on description text for tool selection.
Vague descriptions cause wrong tool calls, which cause wrong answers.

### Tool parameters must be minimal and typed

- Only include parameters the tool genuinely requires.
- Every parameter MUST have a `z.describe()` annotation explaining valid values.
- Use `z.enum()` for fields with a bounded value set — never `z.string()` for status filters.

```js
parameters: z.object({
  status: z.enum(['running', 'stopped', 'paused'])
    .optional()
    .describe('Filter VMs by power state. Omit to return all VMs.'),
})
```

WHY: unbounded `z.string()` parameters allow the model to pass invalid values that
produce confusing downstream errors rather than clear validation failures.

### Tools must not invent or transform data

A tool's `execute` function MUST return the data it receives from the API client with minimal
transformation. It MUST NOT:

- Fill in missing fields with default values or assumptions.
- Merge data from multiple API calls (use separate tools for each call).
- Format, summarize, or interpret data for the model.

The model's job is interpretation. The tool's job is retrieval.

WHY: data transformation in tools makes it impossible to distinguish a model reasoning error
from a data pipeline error when an answer is wrong.

---

## Grounding Rules

These rules exist to prevent the model from inventing cluster state.

### Tool calls are the only source of facts

The system prompt MUST explicitly instruct the model not to answer questions about cluster
state from its training data or prior context. It must always call a tool.

Required language in every system prompt for infrastructure agents:

```
Never answer questions about the current state of the cluster, VMs, nodes, or storage
from memory or assumption. Always use the available tools to retrieve live data.
If no tool can answer the question, say so explicitly.
```

### Validate groundedness before surfacing answers

For queries where the model's answer references specific entities (VM names, node names,
IP addresses, counts), the agent SHOULD log the tool results alongside the final answer
for post-hoc verification. Do not validate inline — just preserve the evidence.

WHY: a 7B model can conflate a tool result with prior training knowledge. Logged tool
results allow you to detect and debug this pattern without instrumenting the model itself.

---

## Inference Call Resilience

Treat the Ollama inference endpoint as a fallible external dependency.

### Timeouts

Every inference call MUST have an explicit per-call timeout distinct from the session timeout.
Recommended: `20_000ms` for a single inference pass on Mistral 7B.

### Error classification

Apply the same classification from the baseline:

| Error | Class | Action |
|---|---|---|
| Ollama HTTP 5xx, connection refused | Upstream | Circuit breaker, no retry on sync path |
| Ollama HTTP 429 | Transient | Backoff retry, max 2 attempts |
| Malformed / unparseable model response | Internal | Fail fast, log raw response, do not retry |
| Model returns no tool call and no answer | Internal | Fail with `NO_MODEL_OUTPUT`, do not retry |
| `maxSteps` exceeded | Internal | Fail with `MAX_STEPS_EXCEEDED`, do not retry |

### Circuit breaker on Ollama

If Ollama fails consecutively (threshold: 3 failures within 60s), open the circuit.
Return a clear `INFERENCE_UNAVAILABLE` error to the caller. Do not queue or retry against
an open circuit on synchronous paths.

WHY: Ollama serving a large model under VRAM pressure can enter a degraded state where
it accepts connections but never responds. Without a circuit breaker this hangs every
agent session.

---

## API Client Contract

The API client (`client/api.js`) is the sole owner of all HTTP communication to downstream
services. No tool or agent module constructs fetch calls directly.

### Session ID header

Every outbound request MUST include `X-Agent-Session-Id`. This is set once per session
and injected into the client at session start — tools do not manage it themselves.

### Envelope unwrapping

The client MUST unwrap the `{ success, data, error }` envelope before returning to tools.
Tools receive `data` directly or a thrown error — they never inspect the envelope.

```js
// client/api.js
if (!envelope.success) {
  const err = new Error(envelope.error.message);
  err.code = envelope.error.code;
  err.class = classifyApiError(envelope.error.code); // transient | permanent | upstream
  throw err;
}
return envelope.data;
```

WHY: envelope inspection scattered across tool implementations creates inconsistent error
handling and makes error classification impossible to enforce uniformly.

### Error classification from API responses

The client MUST classify errors before throwing:

| API error code | Class |
|---|---|
| `NODE_NOT_FOUND`, `VM_NOT_FOUND` | Permanent |
| `INVALID_QUERY_PARAM` | Permanent |
| `RATE_LIMITED` | Transient (honor `Retry-After`) |
| `PROXMOX_UNREACHABLE`, `PROXMOX_TIMEOUT` | Upstream |
| `INTERNAL_ERROR` | Upstream |
| Network / connection failure | Upstream |

---

## Session Lifecycle

### Session ID

Every agent invocation generates a UUID v4 session ID using `crypto.randomUUID()`.
No external library is required or permitted for this.

The session ID MUST be:
- Generated before any tool call
- Attached to every outbound API request via `X-Agent-Session-Id`
- Included in all log entries for the session
- Returned to the caller alongside the agent's answer

### Session log entry

At session close (success or failure), emit a single structured summary log entry:

```json
{
  "ts": "2025-06-01T14:23:01.482Z",
  "sessionId": "a3f1c2d4-...",
  "prompt": "Which VMs are currently stopped?",
  "stepsUsed": 2,
  "toolsCalled": ["listVMs"],
  "durationMs": 4312,
  "outcome": "success",
  "errorCode": null
}
```

For failed sessions, include `errorCode` and `errorClass`.

WHY: per-session summary logs allow you to detect runaway behavior, wrong tool selection
patterns, and latency regressions without reading individual tool call logs.

---

## Prompt Management

### System prompt is config, not code

The system prompt MUST be loaded from environment config or a config file.
It MUST NOT be a hardcoded string literal in `agent.js`.

WHY: the system prompt is the primary lever for correcting model behavior. It will change
more frequently than the inference code. Keeping it in config allows iteration without
code deploys.

### Prompt change discipline

Changing the system prompt is a behavioral change, not a cosmetic one.
Every system prompt change MUST:
- Be committed to version control with a clear commit message describing the behavioral intent
- Be validated against the five golden path queries before merge
- Note any observed changes in tool selection or answer quality

### No prompt injection from user input

User input MUST be passed as the `prompt` parameter only — never interpolated into the
system prompt at runtime.

```js
// WRONG
system: `${config.agent.systemPrompt}\nUser context: ${userInput}`,

// CORRECT
system: config.agent.systemPrompt,
prompt: userInput,
```

WHY: interpolating user input into the system prompt enables prompt injection attacks
that can override grounding rules and tool restrictions.

---

## Security Baseline

In addition to the baseline security rules:

- Never log the full user prompt at `info` level in production — it may contain sensitive
  hostnames, credentials, or infrastructure details. Log a truncated hash or length only.
- Never include internal network topology, hostnames, or credentials in tool descriptions
  or parameter annotations — these are visible to the model and may appear in answers.
- The API client MUST NOT expose raw Proxmox API tokens or credentials. Config values
  are injected via environment only.
- The system prompt MUST NOT contain secrets, internal IPs, or credentials even as examples.

---

## Logging and Observability

In addition to the baseline logging rules:

- Log at `debug` level: full tool call parameters and raw tool results.
- Log at `info` level: session start, session end summary, tool call names (not params).
- Log at `warn` level: step count approaching `maxSteps`, `Retry-After` honored, circuit half-open.
- Log at `error` level: session failure, circuit open, inference timeout, unhandled tool error.

Never log at `debug` in production by default. Gate with `LOG_LEVEL=debug` explicitly.

WHY: tool call parameters can contain full VM inventories or node configs. Logging these
at `info` in production produces excessive volume and potential data exposure.

---

## Code Commenting Standard

Inherit the mandatory WHY / TRADEOFF / VERIFY IF CHANGED format from the baseline.

Additional required comment sites for agent services:

- Every tool `description` field change — note why the wording changed and what behavior it corrects.
- Every `maxSteps` value — note the reasoning for the chosen limit.
- Any place a tool result is transformed before being returned — explain why and what is lost.
- Any place the system prompt is referenced — note the version or last-validated date.

---

## Testing and Quality Gates

In addition to the baseline gates:

- **Golden path queries**: the five canonical queries defined in the executive summary MUST
  pass against a live (or stubbed) Proxmox API before merge on any change to agent loop,
  tools, system prompt, or API client.
- **Tool selection tests**: for each tool, include at least one test asserting the model
  selects it given a representative natural language prompt. Use a stub Ollama response.
- **Runaway loop test**: assert that an agent configured with `maxSteps: 3` against a
  tool that always errors terminates with `MAX_STEPS_EXCEEDED` and does not hang.
- **Envelope error test**: assert that a `success: false` API response causes the tool
  to throw a classified error, not return null or silently swallow.

---

## Review Checklist

For every non-trivial change, reviewers and agents must verify:

1. `maxSteps` is set and within documented limits.
2. Session timeout is present and covers the full agent invocation.
3. No mutable shared state introduced in agent loop or tools.
4. Each tool does exactly one thing against one endpoint.
5. Tool descriptions are specific enough to differentiate from adjacent tools.
6. Tool parameters use `z.enum()` for bounded value sets.
7. Tools do not transform, fill in, or invent data.
8. System prompt contains grounding instruction.
9. User input is not interpolated into the system prompt.
10. API client classifies all error codes before throwing.
11. Session ID is generated once and present on all outbound requests.
12. Session summary log entry is emitted on both success and failure.
13. No secrets, credentials, or internal topology in tool descriptions or prompts.
14. Golden path queries validated against target environment.

---

## Exception Process

Inherit from baseline. Additionally, exceptions to the following rules require explicit
sign-off with written justification:

- `maxSteps` above `25`
- Bypassing the API client to call downstream services directly from a tool
- Interpolating runtime data into the system prompt
- Disabling the session timeout

---

## Agent Delivery Contract

Before finalizing any contribution:

1. State what changed in the agent loop, tools, system prompt, or API client — and why.
2. Report which golden path queries were tested and their outcomes.
3. Report `maxSteps` consumed on each golden path query (indicates efficiency).
4. Call out any change in tool selection behavior observed during testing.
5. Call out known prompt limitations or model behavior edge cases discovered.
6. List any follow-up work deferred and why.

Quality and grounding take priority over answer speed.
