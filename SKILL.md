---
name: systematic-debugging
description: Diagnose bugs, test failures, and unexpected technical behavior by finding the cause before fixing; use a short path for isolated defects and boundary tracing for complex failures.
---

# Systematic Debugging

## Core principle

Establish an evidence-supported cause before implementing a corrective fix. Distinguish a confirmed cause from a hypothesis; do not accumulate speculative patches. A reversible containment measure may reduce harm within existing authorization, but label it as mitigation, not resolution, and continue the investigation.

## Scope and parallel work

- Define the failing behavior, environment/version, expected outcome, and permitted changes. Start with read-only evidence.
- Read current file contents and relevant working-tree changes before editing. Preserve other contributors' work; limit edits to the identified cause. If the same lines changed since inspection, reread and reconcile rather than overwrite, reset, or stash their work.
- Skill use does not authorize deployment, production restarts, new access, payments, external messages, or new monitoring. Use already-granted authorization in its actual scope. Continue independent work when one action needs owner input.
- Isolate tests that write data or run cleanup. Do not reproduce a destructive or externally mutating failure in a live account merely to observe it again.

## Choose the shortest sufficient route

**Short route:** an isolated configuration, documentation, or code defect with a directly inspectable cause and no unresolved cross-component effect.

1. Inspect the error, current configuration/code, and recent relevant change.
2. Confirm cause with a focused read-only check or safe reproduction.
3. Apply one bounded correction within authorization.
4. Verify the actual affected behavior and relevant nearby checks. A direct configuration check can suffice; do not invent a test that repeats the edit.

**Full route:** intermittent failures, multiple components, unknown external outcome, repeated failed fixes, or local success that differs from the public system. Use the four phases below. Escalate from the short route when its evidence no longer explains the failure; a simple defect does not require a full-system audit.

## Evidence discipline

Keep three categories distinct in working notes and reports:

- **Fact:** observed result with source, environment/version and time where relevant.
- **Hypothesis:** proposed cause, why it fits, and an observation that would disprove it.
- **Unverified:** missing access, absent evidence, or a test not yet performed.

Readiness, authorization, installation, delivery, and acceptance are different states. A screenshot or old note can guide inspection but does not prove the current system's behavior.

## Phase 1 — Investigate the cause

1. Read the complete relevant error and trace, while protecting secrets. Record stage/error code rather than assuming a downstream service failed.
2. Reproduce safely if needed. For an externally mutating request with an unknown outcome, first reconcile its existing operation; do not resubmit it for reproduction.
3. Inspect relevant diffs, config propagation, runtime version and dependencies. Prefer existing logs and status evidence before adding instrumentation.
4. Map only the applicable chain, for example **UI → API → queue → external service → persisted result**. Some systems have no queue or external service; do not invent those layers.
5. At relevant boundaries compare expected versus observed input/output using safe metadata: local correlation identifier, status, attempt count, timestamp, version, and destination category. Find the last confirmed success and first missing or failed step, then inspect that boundary. Do not log whole requests, user records, or credentials.
6. If a value originated earlier, trace callers back to the producer. Read [root-cause-tracing.md](root-cause-tracing.md) for deep call chains or test pollution.

### Safe diagnostics

Never dump `env`, configuration files, Authorization headers, cookies, tokens, passwords, subscription links, or full provider/user payloads. Do not enable shell tracing around credentials. Inspect needed values internally and emit only allowlisted booleans/statuses, including on errors; sanitize exception text before sharing it.

Example: distinguish presence and valid URL shape without revealing the endpoint or a key. Replace these example variable names with those actually used by the affected component.

```python
import os
from urllib.parse import urlsplit

key = os.environ.get("SERVICE_API_KEY", "")
endpoint = os.environ.get("SERVICE_ENDPOINT", "")
try:
    parsed = urlsplit(endpoint)
    valid_endpoint = bool(parsed.scheme == "https" and parsed.hostname
                          and not parsed.username and not parsed.password)
    if parsed.port is not None:
        valid_endpoint = valid_endpoint and 1 <= parsed.port <= 65535
except ValueError:
    valid_endpoint = False
print({"key_present": bool(key.strip()),
       "endpoint_present": bool(endpoint.strip()),
       "endpoint_shape_valid": valid_endpoint})
```

These flags do not prove a valid key, DNS, TLS, reachability, or successful authorization. Check those layers separately only when relevant and authorized; report safe error categories rather than raw credential-bearing output.

## Phase 2 — Compare patterns

- Find a working case and compare relevant code, inputs, configuration and runtime versions. Do not read an entire unrelated implementation to diagnose one boundary.
- Identify differences that could explain the observation and their dependencies. A mock, different account, local process, or older build is not equivalent to the failing public case.
- Use [defense-in-depth.md](defense-in-depth.md) if alternate paths can bypass the proposed validation; additional layers need a demonstrated reason.

## Phase 3 — Test a hypothesis

- State one hypothesis and a discriminating check. Prefer read-only comparison; otherwise make the smallest reversible change needed within scope.
- Change one causal variable at a time. If evidence contradicts the hypothesis, record the result and revise it; do not layer another guessed fix on top.
- After repeated failed attempts, stop accumulating patches and revisit evidence, environment, test validity and possible shared state/coupling. No fixed attempt count proves an architectural defect.
- Ask the owner only when the next step requires a scope/product decision or missing permission. Uncertainty alone is a reason to investigate, not to manufacture certainty or stop all independent work.

## Phase 4 — Correct and verify

1. Establish the failing observation before correction. Use the project's tools; optional test-writing or verification skills are not prerequisites.
2. Correct the demonstrated cause with a bounded change. Avoid unrelated refactoring. Add regression coverage when it protects a meaningful behavior; direct inspection may suffice for a small reversible configuration/documentation defect.
3. Run required and relevant checks, then verify the affected outcome. Distinguish these evidence levels:
   - **Local:** build/unit tests; mocks prove only the simulated contract.
   - **Installed:** expected version/config is running in the target environment; this alone does not prove the feature works.
   - **User outcome:** the original flow reaches the expected persisted and visible result in that environment. Obtain authorization for any required live data/device action.
4. For a public failure, a passing local suite or HTTP200 is insufficient. Trace what the response proves: accepted request, queued operation, provider effect, saved result, or actual UI/media behavior.
5. Stop repeating passed checks unless code changed, a check failed, or specific unresolved evidence justifies another run. Report partial validation honestly when a live gate remains.

## External blockers and unknown outcomes

A timeout after a mutation means **unknown**, not **failed**. Keep the existing operation/reference and idempotency key in the appropriate protected project record; do not expose identifiers that grant access.

- Reconcile via the existing queue/database state and provider status/result using authorized read-only tools. A read-only status lookup is different from retrying creation or sending.
- Retry mutation only when reconciliation establishes non-execution with no outstanding processing, or the provider's confirmed idempotency contract makes the same operation safe to repeat within its validity period. Preserve the same identity; do not invent a new key to bypass uncertainty.
- If reconciliation is unavailable, stop further mutation. Continue independent diagnosis and preserve a checkpoint: observed stage, safe evidence, unresolved outcome, prior attempts, exact missing access/action, and next discriminating check.
- If the cause is external or timing-dependent, record which boundaries were verified and what remains unknown. Timeouts, retry handling and monitoring are separate changes requiring justification and scope; they do not establish a cause or grant access.

## Red flags and partner signals

Pause and reassess when tempted to say "probably X" as fact, add several fixes at once, treat a green build as public success, dump credentials for diagnosis, repeat an unknown mutation, or overwrite another developer's edits. Manual verification is valid when it observes the real outcome; skipping all verification is not.

If the owner says "stop guessing", "is that actually happening?", or disputes the result, identify the contradicted claim and seek concrete evidence. Do not restart every completed phase without a relevant reason.

## Report the result

Keep the report proportional: confirmed cause (or unresolved hypothesis), bounded changes and files, checks/environment, actual outcome, and any blocker with a precise continuation step. Call mitigation, local validation, deployment, and user acceptance by their actual names. Do not invent success rates, time savings, or universal quality guarantees.

## Supporting techniques

Read only what the task needs:
- [root-cause-tracing.md](root-cause-tracing.md): backward tracing and test pollution; its test runner requires an isolated disposable checkout.
- [defense-in-depth.md](defense-in-depth.md): justified guards at bypassable boundaries.
- [condition-based-waiting.md](condition-based-waiting.md): bounded waits for async test conditions, not retries of unknown mutations.
- [test-scenarios.md](test-scenarios.md): three fictional requests for behavioral evaluation of this local revision.

## Provenance

Based on `obra/superpowers`, `skills/systematic-debugging`. Original creation history and examples remain in [CREATION-LOG.md](CREATION-LOG.md); its historical claims are not validation of this local revision. Locally adapted for scoped diagnostics, secret protection and explicit validation boundaries (2026-10-07). No upstream authorship is claimed.
