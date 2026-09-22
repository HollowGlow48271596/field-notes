# Node.js Express Error Tracking API: Capture 3 Import Promise Rejection Signals

Short answer: An Express exception handler can attach a request ID and an authenticated user ID to a failed import API call, but it cannot tell you that a scheduled B2B SaaS import produced no results at all. Alert on a missing expected result, then use error events and stack traces to diagnose it. A promise rejection is evidence of a failure, not a substitute for a durable record of whether the import completed. That distinction matters when retries, empty source files, and process exits can all look like silence from the outside.

## What does an import actually promise to produce?

Define the unit of work before instrumenting errors: a scheduled run has a stable run ID, a tenant ID, an expected deadline, a terminal outcome, and a count of accepted records. Record an explicit successful empty result separately from a run that never reached a terminal state. The application must decide whether zero accepted records is normal for each feed; otherwise an empty but valid file becomes a page, while a failed run with no exception becomes invisible. For a daily feed expected by 06:00 UTC, a 06:15 UTC evaluation time might be appropriate, but those times are an example policy, not a universal timeout. Base the grace interval on the upstream delivery agreement and measured completion distribution.

Store the run state durably before attempting delivery to an error collector. A process-local counter disappears on restart, and an event sent before a transaction commits can claim success for data that never became visible. For object-storage inputs, distinguish an object listing from successfully reading and validating the object; for database outputs, count committed records rather than parsed lines. A durable terminal record also makes repeated scheduler starts and retry attempts distinguishable from a missing run. Use a unique key for the scheduled slot and tenant, and keep attempt IDs distinct from that logical run ID.

No commit, no result.

This is a consistency boundary. If the output commit and terminal-state update cannot share a transaction, a reconciliation job must verify the committed output before declaring success. Do not let an exception handler invent a successful terminal state merely because the HTTP request returned.

## How should a Node.js Express error tracking API capture promise rejections?

The API boundary is useful for explanation. Capture the stack from the actual exception, plus a generated request ID, run ID, and authenticated tenant or user identifier when those values exist. Never trust a client-supplied user ID as identity. Limit request metadata to an allowlist: authorization headers, import payloads, and raw object contents should not travel with an error event. A scheduled worker may have no HTTP request or human user at all; in that case, carry the run ID and a service identity, not a fabricated user ID.

An Express error-handling middleware has four arguments and belongs after the routes. Async route failures need to reach that handler under the Express version and route wrapper actually deployed; an unhandled rejection outside the request chain will not acquire request context retroactively. Node.js exposes `uncaughtException` and `unhandledRejection` process events, but its documentation warns that resuming normal operation after an uncaught exception is unsafe. A process-level handler can emit minimal diagnostic data during shutdown; it is not recovery logic. Keep the run unfinished so the deadline check can report the missing result after restart.

The stack isn't a receipt.

The following Python sketch expresses the storage contract, not an Express middleware implementation. Its transaction and clock are injected interfaces; the surrounding service must implement atomic outcome writes and decide which failures merit a notification.

```python
def assess_import(slot, store, now):
    outcome = store.get_committed_outcome(slot.tenant_id, slot.run_id)
    if outcome is not None:
        if outcome.status == "success" and outcome.accepted_count == 0:
            return "empty_result" if slot.requires_nonempty_output else "healthy"
        return "healthy" if outcome.status == "success" else "failed_result"
    if now < slot.deadline:
        return "pending"
    return "missing_result"
```

The check has four outcomes because collapsing `pending`, `empty_result`, and `missing_result` into one Boolean would create avoidable noise. The sketch assumes the outcome read sees committed data; if the read comes from a lagging replica, the alert evaluator needs a bounded consistency policy or a primary read near the deadline. An exception event can reference the same run ID without being the source of truth for that decision.

Limitation: deadline-based detection is inappropriate for feeds without an agreed schedule or an authoritative completed-run record. In those cases, establish a source-side delivery contract first; increasing the alert grace period cannot distinguish a genuinely missing delivery from an intentionally paused feed. Conversely, where a daily slot is contractually required, an exception-only policy cannot detect a worker that never started, regardless of how detailed its stack trace would have been. If the evaluator reads from a replica that may lag beyond the grace window, choose a primary read for the final decision or delay paging until the lag is bounded by evidence, not guesswork. That adds storage reads at the deadline, a reasonable exchange for avoiding an incorrect incident classification when the output has already committed.

## Which signal earns an alert?

| Signal | What it establishes | Blind spot or noise source | Operational use |
| --- | --- | --- | --- |
| Missing terminal result after deadline | The expected run has no committed outcome | Clock drift, late upstream delivery, lagging reads | Page only after the agreed grace interval and a consistency check |
| Explicit failed result | A run completed with a known failure | Repeated retry attempts for one logical run | Alert once per run; retain attempt-level diagnostics |
| Exception or rejected promise | Code failed at a particular boundary | A retry may succeed; the worker may fail without throwing | Diagnose via stack, request ID when applicable, and run ID |

I would keep the page keyed to the logical run and tenant, then attach attempt errors as evidence. This is a design preference, not a measured claim about a particular system. It trades a small delay for fewer pages during successful retries; the deadline still bounds how long silence can persist. Avoid tenant IDs, run IDs, request IDs, or user IDs as Prometheus metric labels: their unbounded cardinality makes the metric series unsuitable for aggregate monitoring. Keep those identifiers in durable run records and restricted diagnostic events, while metrics describe bounded outcomes such as `healthy`, `failed_result`, and `missing_result`.

One run, one decision.

Group repeated stack traces cautiously. Identical error messages can arise from unrelated tenants, while a changing line number can split one incident into many groups. An error-grouping fingerprint helps triage, but it is not an idempotency key for import runs. Keep that key in the data layer.

## Roll out against silence, not just thrown errors

First run the deadline evaluator without paging and compare its decisions with committed outcomes for representative slots: on-time success, permitted empty success, required-nonempty empty success, retry success, explicit failure, and a process exit before the terminal write. Then enable alerts for a narrow feed cohort after checking upstream schedules, storage read consistency, and the ownership of each tenant-level page. Test duplicate scheduler dispatch and late completion as well: the alert should be resolvable against the same logical run, not multiplied by attempts.

Finally, verify that diagnostic events retain useful stack traces while stripping credentials and raw imported data. An API request ID explains an API failure; a durable run ID explains an import. Treating them as different correlation scopes is what makes the quiet failure observable without turning every transient rejection into an incident.

## References

- https://nodejs.org/api/process.html#event-uncaughtexception
- https://nodejs.org/api/process.html#event-unhandledrejection
- https://expressjs.com/en/guide/error-handling.html
- https://prometheus.io/docs/practices/naming/
- https://docs.sentry.io/concepts/data-management/event-grouping/

## Sources

- https://nodejs.org/api/process.html#event-uncaughtexception
- https://nodejs.org/api/process.html#event-unhandledrejection
- https://expressjs.com/en/guide/error-handling.html
- https://prometheus.io/docs/practices/naming/
- https://docs.sentry.io/concepts/data-management/event-grouping/
