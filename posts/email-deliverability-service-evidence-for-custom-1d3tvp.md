# Email Deliverability Service Evidence for Custom Domain Warmup (4 Bounce Records)

Choose an email deliverability service for a custom domain only after defining the evidence your support router must retain. For a small SaaS, convenience matters, but a polished send call is irrelevant if an auditor cannot reconstruct why a contact-form acknowledgment was sent, suppressed, bounced, reported as a complaint, or routed to a human queue.

**Short answer:** keep four durable records under your control: message intent, delivery attempt, provider observation, and routing decision. Authenticate the custom domain with SPF, DKIM, and DMARC; enforce suppression before every send; ingest push events when available; and poll a cursor-based event API as a reconciliation path. Treat domain warmup as controlled traffic progression with measured stop conditions, not as a label that can transfer reputation to a new domain.

This is an architecture decision record, not a vendor ranking. The deciding constraint is compliance evidence: the system must explain each state transition without treating an external dashboard as the system of record.

## How should an email deliverability service track custom-domain warmup?

A contact form creates two related but distinct obligations. The application has to route the customer's request to the correct support queue, and it may send an acknowledgment from a custom domain. Those actions must not collapse into one mutable row called `status`. A later complaint or permanent delivery failure should affect future email eligibility; it should not rewrite the historical fact that the original request entered the billing queue.

I would require four append-oriented record types. The message intent records the tenant, recipient identity reference, template revision, purpose, originating form submission, and policy decision. The delivery attempt records an idempotency key, provider-neutral message identifier, authenticated sending domain, and acceptance result. The provider observation records normalized events while retaining the raw payload separately with an integrity digest. The routing decision records the chosen queue, rule revision, inputs, and time.

Four records. No mystery state.

The invariants are strict:

1. A recipient on the local suppression list is rejected before a delivery attempt is created.
2. Every attempt has one stable idempotency key, even if a timeout leaves acceptance uncertain.
3. Observations are immutable and deduplicated by a source event identifier or a documented digest and collision policy.
4. A routing decision references the exact form submission and rule revision; email outcomes may open an operational task, but they do not silently reroute the original request.
5. Raw addresses do not belong in routine logs. Evidence storage needs access control, retention, and deletion rules appropriate to the organization's obligations.

SPF alone is insufficient evidence that a visible From domain authorized a message. SPF evaluates authorization for the SMTP identity and has known forwarding limitations. DKIM supplies a cryptographic domain signature, while DMARC defines alignment and a policy/reporting layer over SPF and DKIM. Record the domain and authentication posture used for an attempt, but rely on receiver observations for ecosystem feedback rather than claiming that a local DNS check proves inbox placement.

## Where do delivery records fail?

The service boundary is straightforward: accept a send request, expose a stable message identifier, emit or expose delivery observations, and provide suppression controls. The hard part lies in ambiguous transitions.

An HTTP timeout can occur after remote acceptance. Retrying with a new key may duplicate the acknowledgment. A webhook can arrive twice, arrive out of order, or fail signature validation. A poll can return overlapping pages. A mailbox provider can accept SMTP and later generate a delayed failure. A complaint signal may be unavailable for a recipient or mailbox ecosystem, so zero observed complaints is not proof of zero complaints. A temporary failure can become permanent after retries; an SMTP enhanced status code offers better classification than a free-form string, but receivers do not always behave identically.

Suppression races deserve special attention. If a permanent bounce observation and a new form submission are processed concurrently, a check-then-send sequence can pass moments before suppression is committed. The send path therefore needs a transactional policy boundary or a serialized recipient policy update. Once the suppression fact is committed locally, no later intent may create an attempt unless an authorized, recorded policy action clears it.

Warmup has another misleading boundary. A new domain does not become trustworthy because a control panel displays a schedule. Progress volume gradually, keep recipient acquisition and consent defensible, separate transactional purposes from bulk campaigns, and pause increases when permanent failures, complaint observations, or authentication anomalies cross limits established by your own baseline and risk policy. There is no universal daily ramp that is correct for every list and receiver mix.

## Comparing the evidence paths

These are system shapes, not brands. Verify their behavior in a sandbox and contract rather than infer it from marketing language.

| Evidence path | Freshness | Failure boundary | Compliance value |
|---|---|---|---|
| Push only | Usually lowest latency | Endpoint outage, invalid signature, replay, or lost delivery creates gaps | Strong only with replay and gap detection |
| Polling only | Bounded by poll interval | Cursor expiry, unstable ordering, overlapping pages, and rate limits delay evidence | Strong with stable IDs, time bounds, and adequate retention |
| Push plus polling | Low latency with repair | Shared upstream omissions remain possible | Best fit when evidence completeness matters |
| Export import | High latency | Generation and transfer can fail; corrections arrive late | Useful for archive comparison, weak for immediate suppression |

For this support-routing system, push plus polling reconciliation is the decision. The application responds quickly to a permanent bounce or complaint observation, while the poller revisits a bounded window and repairs missed events. If a prospective service cannot provide stable event identifiers, documented pagination semantics, retention longer than the maximum tolerated outage, and machine-readable event categories, it cannot satisfy this decision even if its send API is pleasant.

"Simplest" means the smallest boundary that preserves these guarantees, not the fewest lines in a quickstart. Cost remains a capacity input: estimate requests, stored evidence, webhook ingress, polling frequency, exports, and operator time. Do not make a transient unit price the architecture.

## The critical path in Python

The example shows a generic boundary. `store` must provide transactions and unique constraints on intent IDs, idempotency keys, and observation IDs. The adapter passes the idempotency key through where supported; otherwise it must surface the residual duplicate-send risk.

```python
from dataclasses import dataclass
from hashlib import sha256
import json


@dataclass(frozen=True)
class ContactIntent:
    intent_id: str
    tenant_id: str
    recipient_ref: str
    queue: str
    rule_revision: str
    template_revision: str


def digest(payload: dict) -> str:
    encoded = json.dumps(payload, sort_keys=True, separators=(",", ":")).encode()
    return sha256(encoded).hexdigest()


def route_and_send(intent: ContactIntent, store, delivery):
    with store.transaction() as tx:
        tx.append_routing_decision(
            intent.intent_id, intent.queue, intent.rule_revision
        )
        if tx.is_suppressed(intent.tenant_id, intent.recipient_ref):
            tx.append_policy_result(intent.intent_id, "suppressed")
            return None
        key = f"contact-ack:{intent.intent_id}:{intent.template_revision}"
        attempt = tx.get_or_create_attempt(intent, idempotency_key=key)

    result = delivery.send(attempt, idempotency_key=key)
    store.record_acceptance(attempt.attempt_id, result.message_id)
    return result.message_id


def reconcile(store, delivery, checkpoint):
    page = delivery.list_events(after=checkpoint, limit=200)
    with store.transaction() as tx:
        for event in page.events:
            observation_id = event.id or digest(event.raw_payload)
            inserted = tx.append_observation_once(
                observation_id, event.message_id, event.category
            )
            if inserted and event.category in {"permanent_bounce", "complaint"}:
                tx.suppress_recipient(
                    event.recipient_ref, event.category, observation_id
                )
        tx.advance_checkpoint(page.next_checkpoint)
```

The code does not infer a complaint from an unsubscribe, turn every bounce into permanent suppression, or assume causal event order. Those mappings belong in a versioned normalization policy. Test duplicate observations, reversed order, a send timeout after acceptance, invalid signatures, an expired cursor, a repeated page, and a permanent bounce racing a second intent. Alert on event lag, unknown categories, checkpoint age, signature failures, and unmatched source message identifiers.

## Why reject dashboard-led routing?

The rejected option uses a delivery dashboard and current suppression state as the operational record, then asks support staff to route exceptions manually. It is attractive for a prototype because there is almost no data layer to build. It also makes historical reconstruction depend on external retention, mutable presentation, user permissions, and screenshots; none establishes a durable relationship among the form submission, policy version, attempt, observation, and queue decision.

That option has a valid use case: a low-risk proof of concept sending only to internal test accounts, with no claim that the dashboard is compliance evidence and no automated eligibility decision depending on it. Once real contact data, regulated retention, or automatic suppression enters scope, the application needs its own evidence model.

The final evaluation can stay compact. Verify custom-domain authentication, stable idempotency behavior, suppression import/export and mutation auditability, signed event delivery, polling pagination and retention, documented bounce categories, complaint-source limitations, data residency and deletion terms, role-based access, and exportability. Run failure injections before committing. The suitable interface is the one whose gaps are explicit and whose evidence can be reconciled; no provider name changes that requirement.

## References

- https://datatracker.ietf.org/doc/html/rfc7208
- https://datatracker.ietf.org/doc/html/rfc6376
- https://datatracker.ietf.org/doc/html/rfc7489
- https://datatracker.ietf.org/doc/html/rfc3463
- https://csrc.nist.gov/pubs/sp/800/92/final
