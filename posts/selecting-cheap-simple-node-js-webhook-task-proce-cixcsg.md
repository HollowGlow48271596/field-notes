# Selecting Cheap, Simple Node.js Webhook Task Processing with Queues and Retries

Put incoming webhook work on a durable queue, acknowledge the request only after the task is accepted, and let a separate worker enforce rate limits, retries, and exponential backoff. For an e-commerce cleanup job, delivery guarantees matter more than finding the cheapest timer: a scheduler decides *when* to ask for work, while the queue and handler decide whether that work is eventually completed without corrupting inventory or deleting the wrong object.

Short answer: select the smallest hosted or self-managed system that provides durable task acceptance, explicit retry policy, concurrency control, and observable terminal failures; a cron-only service is sufficient only when the triggered operation is already idempotent and durable elsewhere.

This is an architecture decision record for delayed webhook tasks initiated by a Node.js SaaS, though the critical worker example is deliberately Python because the protocol boundary should be ordinary HTTP rather than an SDK assumption. “Cheap” belongs in the constraint set, but a low timer price cannot compensate for lost acknowledgements or duplicate cleanup.

## What should a Node.js SaaS require from a webhook task queue with retries and backoff?

The invariants come first. A received task needs a stable task ID, a durable payload or durable reference to one, a recorded acceptance boundary, and a handler whose result can be applied more than once. The system must also distinguish a retryable attempt from a terminal failure. If a platform's documentation doesn't make those states observable, I'm not sure a feature checklist can resolve the risk; a failure-injection test can.

For the concrete case, imagine a marketplace that marks abandoned product-image uploads for deletion after 24 hours. The webhook request must not remain open until then. Its payload can contain `task_id`, `shop_id`, `object_key`, `not_before`, and an expected object version. The object version matters — without it, a late cleanup attempt could remove a newer upload that reused the same key.

The acceptance boundary is narrow: validate and authenticate the webhook, persist or enqueue the task, then return success. Don't acknowledge before durable acceptance. Don't perform the remote deletion inside the web request either, because a slow downstream call converts routine backpressure into request timeouts and ambiguous retries from the sender.

Delivery labels need skeptical reading. “At least once” means duplicates are allowed, not that completion is magically guaranteed; finite retry schedules can still end in a dead-letter or failed state. “Exactly once” is usually the wrong application-level target because the queue cannot make an external object store and the commerce database participate in one atomic commit. The useful invariant is an idempotent effect: repeated delivery of `task_id=cleanup_7f2a` produces the same final state.

Small detail, big boundary.

## Decision record: compare the failure boundaries

The options differ less by their timer syntax than by where they place durable state and who must operate it. This table is the selection record, not a vendor ranking.

| Option | Durable acceptance | Rate-limit control | Retry and backoff ownership | Operational burden | Not suitable when |
|---|---|---|---|---|---|
| Cron trigger calling an HTTP endpoint | Outside the timer; the endpoint must persist work | In the application or a separate queue | In the application | Low for timing, higher for delivery logic | Each scheduled item needs independent retry state |
| Hosted event or task workflow | Usually part of the service contract; verify the documented boundary | Service concurrency plus application limits | Mostly configured, with handler idempotency still owned by the application | Lower, with platform coupling | Policy requires self-hosting or the workflow semantics cannot be verified |
| Self-managed durable queue plus scheduler | In the queue after a confirmed publish | Worker concurrency and token-bucket policy | Worker and queue configuration | Highest: upgrades, storage, alerting, and recovery stay with the team | The team cannot staff queue operations |
| Database-backed task table | In the application's transactional database | Claim query and worker concurrency | Application code and task columns | Moderate, but polling and table maintenance need care | Task volume or retention creates unacceptable pressure on the primary database |

A cron trigger is a clock, not a per-task delivery ledger. Cloudflare documents Cron Triggers as scheduled Worker invocations and notes that configuration changes can take several minutes to propagate. That makes the mechanism plausible for periodic scanning, but the scanner still needs durable task state and idempotent effects. Inngest documents event-driven functions, retries, concurrency, throttling, rate limiting, and scheduled execution; those are relevant capabilities to verify against the workload, not a reason to skip an acceptance-boundary test.

The database-table option deserves more respect than it usually gets. If creating an order-cancellation record and its cleanup task must be atomic, inserting both in one database transaction avoids a gap between business state and queue publication. Workers can claim due rows with locking semantics, increment attempts, and move exhausted work to a terminal state. The catch is that the primary database then carries polling, indexes, retention, and contention. Your mileage may vary with workload shape; queue depth, row churn, and the database's measured lock behavior settle this choice better than an abstract throughput claim.

Cost should be modeled as triggered executions, retained state, egress, and on-call time together. A self-managed queue can have a small invoice and a large ownership burden. A hosted workflow can reverse that profile. Neither is inherently “cheap” until expected task volume, retention, retry amplification, and operator time are put into the same estimate.

## Put the critical path in the worker

The worker below shows the contract that survives a change of scheduler or queue. It uses a stable idempotency key, refuses work before `not_before`, checks the expected object version, applies a local rate limiter, and classifies outcomes. The queue adapter is intentionally generic; production code must connect `ack`, `retry`, and `fail` to durable queue operations.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from enum import Enum
from typing import Protocol


class Result(Enum):
    ACK = "ack"
    RETRY = "retry"
    FAIL = "fail"


@dataclass(frozen=True)
class CleanupTask:
    task_id: str
    shop_id: str
    object_key: str
    expected_version: str
    not_before: datetime
    attempt: int


class CleanupStore(Protocol):
    def already_completed(self, task_id: str) -> bool: ...
    def current_version(self, shop_id: str, object_key: str) -> str | None: ...
    def delete_and_record(self, task: CleanupTask) -> None: ...


class RateLimiter(Protocol):
    def allow(self, shop_id: str) -> bool: ...


def retry_delay_seconds(attempt: int) -> int:
    # Cap exponential backoff; the queue should add jitter when scheduling.
    return min(300, 2 ** min(attempt, 8))


def process_cleanup(
    task: CleanupTask,
    store: CleanupStore,
    limiter: RateLimiter,
) -> tuple[Result, int | None]:
    now = datetime.now(timezone.utc)

    if store.already_completed(task.task_id):
        return Result.ACK, None
    if now < task.not_before:
        return Result.RETRY, max(1, int((task.not_before - now).total_seconds()))
    if not limiter.allow(task.shop_id):
        return Result.RETRY, retry_delay_seconds(task.attempt)

    current_version = store.current_version(task.shop_id, task.object_key)
    if current_version is None:
        return Result.ACK, None
    if current_version != task.expected_version:
        return Result.FAIL, None

    store.delete_and_record(task)
    return Result.ACK, None
```

One caveat is deliberately visible: `delete_and_record` names an application invariant, not an assertion that a remote delete and a local database write are atomic. Walk the failure sequence rather than trusting the method name. Attempt 1 reads object version `v17`, confirms that it matches the task, sends the delete, and then the worker process stops before its queue acknowledgement and local completion record. The queue later delivers attempt 2. That attempt must treat an absent object as the already-achieved target state, record completion for the same stable task ID, and acknowledge; issuing a second delete must also be harmless if the storage API exposes absence that way. Now change one detail: a merchant uploads a replacement under the same key before attempt 2, producing version `v18`. The worker must refuse to delete it because the task authorized deletion of `v17`, not “whatever currently occupies this path.” This is why the payload carries an expected version and why a bare object key is insufficient. A real implementation should retain the task ID as a completion record, define how long that record lives, and ensure its retention exceeds the queue's redelivery horizon. The remote delete and local write still are not atomic, but redelivery converges instead of turning the ambiguous interval into data loss.

Duplicates are normal.

Backoff should include jitter so a shared outage or rate-limit window doesn't release every delayed task at the same instant. Per-shop concurrency prevents one large merchant from starving smaller shops, while a global ceiling protects the downstream storage service. Use separate limits: concurrency bounds simultaneous work; a token bucket bounds starts over time. They solve different overload modes.

Retries also need a budget. Authentication failures, malformed payloads, and version mismatches are terminal because waiting does not change them. Timeouts and explicit throttling signals can be retryable, up to the documented attempt or age limit. Record the decision, next attempt time, and last error class without placing secrets or full webhook bodies in logs.

## Test delivery guarantees before deployment

Start with a contract test that submits one task and proves the web request returns after durable acceptance, not after cleanup. Then deliver the same task twice concurrently and verify one final effect. Stop a worker between the external effect and acknowledgement; restart it and verify convergence. Hold the rate limiter closed, check that the attempt is deferred rather than lost, and release it. Finally, exhaust the retry budget and confirm that the terminal task is searchable and replay requires an explicit operator action.

Those tests should run against the actual queue adapter. Mocks won't reveal acknowledgement timing.

Test the stop point.

Operationally, alert on the age of the oldest ready task, terminal-failure count, retry rate, claim-to-completion latency, and rate-limit deferrals. Queue depth alone is weak: a depth of 10,000 could be healthy during a planned import, while a single cleanup task stuck beyond its deletion deadline can violate the business rule. Deployment should drain or version workers so an older process does not misread a newly shaped payload; additive payload changes and explicit schema versions make rolling upgrades less exciting.

## Rejected option and the case where it wins

For this e-commerce cleanup, reject a bare cron callback that scans and deletes everything within one invocation. It combines discovery, rate limiting, external side effects, and retry state in one execution window. A timeout near the end leaves the next invocation guessing which objects were handled, and one large shop can consume the whole run.

Stick with cron-only when the job is a bounded, idempotent sweep over authoritative state, missed runs can be repaired by the next sweep, and per-item retry history has no business value. Cache warming or recomputing a derived daily aggregate can fit that boundary. Cloudflare Cron Triggers are one documented implementation of scheduled invocation; a conventional system scheduler can satisfy the same architecture when the runtime and operations model fit.

Choose a hosted workflow when the team values managed retry state and concurrency controls more than portability. Choose a self-managed queue when policy, workload control, or existing operations make ownership acceptable. Choose a database-backed queue when transactionally coupling business state and task creation is the dominant invariant and measured database load remains within limits.

The decision is therefore conditional: use the queue or workflow that can prove durable acceptance and observable terminal outcomes with the least operational surface your team can actually support. Keep the cleanup effect idempotent regardless. Schedulers can move; correctness should not move with them.

## References

- https://developers.cloudflare.com/workers/configuration/cron-triggers/
- https://www.inngest.com/docs
