# AGENTS.md — Enterprise Baseline (Project-Agnostic)

This document defines default engineering constraints for AI agents and human contributors.
These rules are intentionally strict to preserve reliability, security, maintainability, and performance.

## Authority and Scope

Machine-optimized RDF compact version: `project-agnostic-AGENTS-rdf.md`

- This file is the canonical source for contributor behavior.
- Any shorter instruction file (for IDE assistants, bots, etc.) MUST NOT weaken or contradict this file.
- If guidance is condensed, it must still explicitly retain:
  1. architecture and separation-of-concerns boundaries,
  2. event-loop and latency safety rules,
  3. async and deterministic error-handling contracts,
  4. resilience rules (circuit breaker, retry, error differentiation),
  5. security and data-handling constraints,
  6. mandatory rationale-focused code comments,
  7. validation/test execution expectations before merge.

## Architecture and Separation of Concerns

- Keep transport layers thin (controllers/routes/handlers): parse input, call service, map response, return.
- Put business rules in services/domain modules, not controllers, templates, or data-access plumbing.
- Keep data access behind explicit repository/data modules.
- Keep external integrations (queues, HTTP clients, storage) behind adapter interfaces.
- Prefer composition and factory-style construction over deep inheritance.
- WHY: strict layering improves testability, change isolation, and incident debugging speed.

## Performance and Concurrency Guardrails

### Non-negotiable hot-path rule

In code that executes per request/job/event under traffic:

- Do NOT use blocking sync APIs for filesystem, subprocess, compression, or crypto.
- Do NOT run unbounded or size-unaware loops.
- Do NOT perform expensive serialization, traversal, or template compilation repeatedly per request.

### Required alternatives

- Use async APIs and bounded work units.
- Cache immutable/rarely-changing artifacts when safe.
- Use streaming for large payloads.
- Move expensive CPU work to workers/queues/precompute stages.
- Add input-size guardrails and timeouts.

## Async and Error Contracts

- Public APIs should be consistently async when async work is possible.
- Never mix callback and Promise completion paths in ways that can double-complete.
- Avoid “sometimes sync, sometimes async” callback timing.
- Surface errors through one deterministic path (throw/reject or callback(err), not multiple).
- Centralize error-to-HTTP/protocol mapping and keep status code semantics consistent.

### Error Differentiation

Classify every error at the point it is caught or created:

- **Transient** (network timeout, 502/503, rate-limit): safe to retry.
- **Permanent** (400 validation, 404 not-found, schema mismatch): fail immediately; retrying will not help.
- **Upstream / dependency** (third-party outage, queue unavailable): route through circuit breaker; may need fallback.
- **Internal / bug** (null-ref, assertion, invariant violation): fail fast, alert, do not retry.

Propagate the classification through the error object or a wrapper so callers can branch on category, not on string matching or status-code guessing.
- WHY: undifferentiated “catch-all → retry” masks permanent failures and wastes resources on hopeless retries.

## Resilience and Failure Handling

### Circuit Breaker

When calling any external dependency (HTTP service, database, queue, third-party API):

- Track consecutive failure counts or error-rate windows.
- **Open** the circuit (reject calls immediately with a descriptive error) when the failure threshold is breached.
- **Half-open** after a cooldown period: allow a single probe request to test recovery.
- **Close** the circuit when the probe succeeds.
- Log every state transition with dependency name, threshold, and window size.
- WHY: without circuit breakers a single downstream outage cascades into thread/connection exhaustion across the entire system.

### Retry Behavior

Retries are only appropriate for **transient** errors (see Error Differentiation above).

- Use exponential backoff with jitter; never use fixed-interval or immediate retries under load.
- Cap retries with a hard maximum (recommend ≤ 3 for synchronous paths, configurable for async/worker paths).
- Ensure the operation is **idempotent** before enabling retries; if it is not, do not retry.
- Set a per-attempt timeout that is shorter than the overall request budget.
- Emit a metric or structured log on every retry attempt (attempt number, delay, error category).
- After final retry exhaustion, surface a clear terminal error—do not silently swallow.
- WHY: uncontrolled retries amplify load during outages (“retry storm”) and violate latency budgets.

### Fallback and Graceful Degradation

- When a dependency is unavailable and a circuit is open, prefer returning a degraded response (cached data, default value, feature toggle off) over a hard failure when the business context allows it.
- Document which paths support degradation and what the degraded behavior is.
- Never silently degrade without logging; operators must know the system is running in reduced mode.
- WHY: users tolerate stale or partial data better than complete outages.

## Security Baseline

- Validate and normalize all untrusted input at boundaries.
- Enforce allowlists for enum-like fields and strict schema validation for payloads.
- Use parameterized DB operations / approved query layers only.
- Never leak internals in client-facing errors.
- Keep secrets out of logs and source; use environment/secret manager integration.
- Apply least privilege for service credentials and external access.
- WHY: most severe incidents are input-validation and data-exposure failures.

## Logging and Observability

- Use structured logs (JSON/fields), not ad-hoc free-form strings on hot paths.
- Gate verbosity by level and avoid expensive log payload construction when dropped.
- Include correlation/request IDs, latency, and status outcome where relevant.
- Emit metrics/events for throughput, error rate, latency, and saturation.

## Code Commenting Standard (Mandatory)

Comments are the design record, not a transcript of the code. Assume the reader is a competent developer who can read the code but was not in the room when it was written. They can see *what* the code does; they cannot see *why it does it that way* or *what else was considered*. A comment earns its place when removing it would force the next person to re-derive a decision by trial and error.

### Rules

- Default to documenting rationale (why), not mechanics (what).
- Do NOT add comments that simply narrate obvious code. If a comment could be regenerated by reading the line below it, delete it.
- DO add comments for non-trivial decisions, fallback behavior, edge cases, security choices, and performance tradeoffs.
- DO record the obvious-looking alternative that was rejected, so the next reader (or agent) does not "fix" the code back to a broken state.
- DO mark load-bearing code: a line or ordering that looks incidental but isn't. State what breaks if it changes.
- For non-trivial blocks, include at least one WHY-comment.
- Write the comment you would want if you were paged at 3am to change this code and had never seen it before.

### Rationale describes *why*, never *current values*

Comments that duplicate a value or behavior owned elsewhere (a magic number's meaning, a copy of a config default) will drift out of date. Describe the reasoning and point to the source of truth — do not restate the value.

### Tie rationale to its stability

- A decision tied to a *temporary* constraint must say so and name the trigger that retires it (e.g. "remove once the v2 endpoint ships"). Otherwise the constraint is preserved forever after it stops being real.
- A decision tied to a *permanent* invariant must say so, so it is not later "cleaned up."

### Comment scope

- **Module header** — why this module exists, its boundary, and what it deliberately does *not* do.
- **Function doc** — the contract: what it guarantees, what it assumes the caller has already done, and its failure modes.
- **Inline** — reserved for the load-bearing or counterintuitive line. Inline comments are for surprises only.

### Stored-procedure boundary (required)

Any place the JS layer compensates for stored-procedure behavior — deadlock/retry handling, result-shape quirks, an SP owning a transaction the JS layer cannot see — MUST carry a rationale comment. This seam is exactly where a junior or agent will otherwise "simplify" something load-bearing.

### Preferred format

```text
WHY: <constraint/rationale; include the rejected alternative if one looks obvious>
TRADEOFF: <accepted downside>
STABILITY: <permanent invariant | temporary — retire when X>
VERIFY IF CHANGED: <what must be re-tested>
```

`STABILITY` may be omitted only when the rationale is self-evidently permanent. `WHY` and `VERIFY IF CHANGED` are always required for non-trivial blocks.

## Documentation Sync Rules

When changing public behavior, interfaces, package boundaries, configuration, deployment, or operational runbooks:

1. Update user-facing README/API docs.
2. Update developer docs/architecture notes.
3. Update operational docs (runbooks, env vars, migration notes).
4. Update changelog/release notes where applicable.
5. Verify example snippets still execute against current code.

## Testing and Quality Gates

Minimum expectation before merge:

- Lint/format checks pass.
- Unit tests cover core logic and error paths.
- Integration tests cover boundary behavior and critical workflows.
- Any concurrency-sensitive changes include stress/race/ordering validation where practical.
- New public behaviors include tests and docs.

If a check is intentionally skipped, document:

- why it was skipped,
- impact and risk,
- follow-up owner/date.

## Review Checklist

For every non-trivial change, reviewers/agents must verify:

1. No blocking sync APIs introduced in hot paths.
2. No async consistency regressions.
3. Error handling is deterministic and policy-aligned.
4. Errors are classified (transient/permanent/upstream/internal) and routed accordingly.
5. External calls use circuit breakers or document why they are exempt.
6. Retries are idempotent, bounded, and use backoff with jitter.
7. Security validation is present at boundaries.
8. Logging is useful, structured, and not throughput-dominant.
9. Docs/tests updated with behavior changes.
10. Rationale comments exist where decisions are non-obvious.

## Exception Process

Exceptions are allowed only when explicitly documented in code review/PR notes:

- Why the exception is needed.
- Why safer alternatives were not used.
- Scope and blast radius (startup-only, low-frequency admin path, etc.).
- Verification performed and rollback plan.

## Agent Delivery Contract

Before finalizing any contribution, agents should:

1. Summarize what changed and why.
2. Report exact validation commands executed and outcomes.
3. Call out known limitations or follow-up work.
4. Avoid hidden assumptions; state constraints clearly.

Quality and safety take priority over speed.
