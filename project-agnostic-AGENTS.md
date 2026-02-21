# AGENTS.md — Enterprise Baseline (Project-Agnostic)

This document defines default engineering constraints for AI agents and human contributors.
These rules are intentionally strict to preserve reliability, security, maintainability, and performance.

## 1) Authority and Scope

- This file is the canonical source for contributor behavior.
- Any shorter instruction file (for IDE assistants, bots, etc.) MUST NOT weaken or contradict this file.
- If guidance is condensed, it must still explicitly retain:
  1. architecture and separation-of-concerns boundaries,
  2. event-loop and latency safety rules,
  3. async and deterministic error-handling contracts,
  4. security and data-handling constraints,
  5. mandatory rationale-focused code comments,
  6. validation/test execution expectations before merge.

## 2) Architecture and Separation of Concerns

- Keep transport layers thin (controllers/routes/handlers): parse input, call service, map response, return.
- Put business rules in services/domain modules, not controllers, templates, or ORM models.
- Keep data access behind explicit repository/data modules.
- Keep external integrations (queues, HTTP clients, storage) behind adapter interfaces.
- Prefer composition and factory-style construction over deep inheritance.
- WHY: strict layering improves testability, change isolation, and incident debugging speed.

## 3) Performance and Concurrency Guardrails

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

## 4) Async and Error Contracts

- Public APIs should be consistently async when async work is possible.
- Never mix callback and Promise completion paths in ways that can double-complete.
- Avoid “sometimes sync, sometimes async” callback timing.
- Surface errors through one deterministic path (throw/reject or callback(err), not multiple).
- Centralize error-to-HTTP/protocol mapping and keep status code semantics consistent.

## 5) Security Baseline

- Validate and normalize all untrusted input at boundaries.
- Enforce allowlists for enum-like fields and strict schema validation for payloads.
- Use parameterized DB operations / approved query layers only.
- Never leak internals in client-facing errors.
- Keep secrets out of logs and source; use environment/secret manager integration.
- Apply least privilege for service credentials and external access.
- WHY: most severe incidents are input-validation and data-exposure failures.

## 6) Logging and Observability

- Use structured logs (JSON/fields), not ad-hoc free-form strings on hot paths.
- Gate verbosity by level and avoid expensive log payload construction when dropped.
- Include correlation/request IDs, latency, and status outcome where relevant.
- Emit metrics/events for throughput, error rate, latency, and saturation.

## 7) Code Commenting Standard (Mandatory)

Default to documenting rationale (why), not mechanics (what).

- Do NOT add comments that simply narrate obvious code.
- DO add comments for non-trivial decisions, fallback behavior, edge cases, security choices, and performance tradeoffs.
- For non-trivial blocks, include at least one WHY-comment.

Preferred format:

```text
WHY: <constraint/rationale>
TRADEOFF: <accepted downside>
VERIFY IF CHANGED: <what must be re-tested>
```

## 8) Documentation Sync Rules

When changing public behavior, interfaces, package boundaries, configuration, deployment, or operational runbooks:

1. Update user-facing README/API docs.
2. Update developer docs/architecture notes.
3. Update operational docs (runbooks, env vars, migration notes).
4. Update changelog/release notes where applicable.
5. Verify example snippets still execute against current code.

## 9) Testing and Quality Gates

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

## 10) Review Checklist

For every non-trivial change, reviewers/agents must verify:

1. No blocking sync APIs introduced in hot paths.
2. No async consistency regressions.
3. Error handling is deterministic and policy-aligned.
4. Security validation is present at boundaries.
5. Logging is useful, structured, and not throughput-dominant.
6. Docs/tests updated with behavior changes.
7. Rationale comments exist where decisions are non-obvious.

## 11) Exception Process

Exceptions are allowed only when explicitly documented in code review/PR notes:

- Why the exception is needed.
- Why safer alternatives were not used.
- Scope and blast radius (startup-only, low-frequency admin path, etc.).
- Verification performed and rollback plan.

## 12) Agent Delivery Contract

Before finalizing any contribution, agents should:

1. Summarize what changed and why.
2. Report exact validation commands executed and outcomes.
3. Call out known limitations or follow-up work.
4. Avoid hidden assumptions; state constraints clearly.

Quality and safety take priority over speed.
