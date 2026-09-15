# One API Key, Many Capabilities — Node.js Changes in a Logistics Provisioning Call

TL;DR: A single provisioning call changes the unit of correctness. You are no longer checking whether one key was issued; you are proving that a bundle of capabilities, limits, and billing identity became effective together, without granting a capability to the wrong logistics customer. Keep a durable provisioning record, make retries idempotent, and meter usage against that record rather than against whatever token happens to arrive at a worker.

In a logistics platform, onboarding can create dispatch access, tracking events, label generation, and usage-metering permissions in one request. The convenience is real. The accounting risk is larger: if label jobs are attributed to the account that supplied a shared credential, invoices can be wrong even while every API call returns 200.

## What changes when provisioning is one call?

A multi-step flow exposes intermediate states. A client can have tracking permission before label permission, or a metering record can exist before the secret is available. A single call hides those transitions from the caller, but it does not remove them from the system. They still exist in the database, secret store, authorization cache, and message queues.

That means the API contract needs a state model, not a promise that everything happened atomically. I use a provisioning intent with a stable request identifier and an explicit status such as `pending`, `active`, or `failed`. The response can say that the intent is accepted; workers then reconcile each capability until the intent reaches a terminal state. If the product requires synchronous activation, the server still records the intent before touching external systems, so a timeout can be retried without creating a second billing identity. Set a concrete reconciliation budget, such as 30 seconds, and expose the age of the oldest pending intent; an unbounded spinner hides a billing defect from both operators and customers.

That boundary matters.

The important distinction is between identity and credential. One customer account may have several credentials, and one credential may authorize several capabilities. Metering must point to the customer account and capability grant recorded in the provisioning intent, never infer ownership from a key string or from the last service that touched the request.

## The attribution ledger is the real control plane

For each provisioning intent, persist an immutable event with these fields: account ID, capability set, policy version, effective time, request ID, and the credential fingerprint. Store the secret itself in a secrets manager; the ledger only needs a non-reversible fingerprint or reference. OWASP recommends separating secrets from source code and limiting access through controlled, auditable mechanisms, which fits this split between a ledger and a secret store.

Usage events should carry the intent ID (or a derived grant ID) from ingress to the metering sink. A shipment scan might produce 18 tracking events and two label renders; those are separate meter dimensions even if the same key authorizes both. The invoice job aggregates by account ID, capability, and the event's usage window.

A minimal event shape looks like this:

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Mapping

@dataclass(frozen=True)
class UsageEvent:
    event_id: str
    account_id: str
    grant_id: str
    capability: str
    quantity: int
    occurred_at: datetime
    attributes: Mapping[str, str]
```

The `event_id` is the deduplication key. The `grant_id` is the attribution anchor. If a queue retries a message, the same event is accepted once; if a worker receives a key with no matching active grant, it sends the event to a quarantine stream instead of guessing. Guessing is how a disputed invoice becomes impossible to explain.

## Which failure modes deserve a test before launch?

The first failure is partial success. Capability A is committed, capability B times out, and the client retries. An idempotency key on the provisioning request lets the server return the original intent and lets a reconciler finish B. It does not make every downstream dependency transactional, so the reconciler needs a bounded retry policy and a visible terminal error.

The second is stale authorization. A policy cache may continue to accept a revoked grant for a short interval. Record both the grant's effective and revoked timestamps, then have the meter apply the same policy used by authorization at event time. A late event can remain billable if its `occurred_at` precedes revocation; that rule must be explicit and tested with clock skew.

The third is capability drift. Someone edits a bundle definition and existing customers silently receive a new permission. Version the bundle and include the version in every usage event. A migration can then compare `bundle_v3` with `bundle_v4` and show exactly which accounts changed.

The fourth is replay. Webhooks and queue deliveries are commonly at-least-once. A unique constraint on `(account_id, event_id)` or an equivalent idempotent sink prevents duplicate quantity. Do not use a timestamp as a deduplication key; two legitimate scans can share the same second.

The fifth is ambiguous ownership in asynchronous work. Pass the grant ID in the job payload, alongside the account ID, and reject a payload where they disagree. This is a small check with a large effect on invoice integrity.

## How do common tools differ at this boundary?

The design should survive a provider change, so compare tools by the boundary they enforce rather than by feature count.

| Tool or pattern | Useful boundary | Attribution caveat for a metered account |
| --- | --- | --- |
| AWS IAM | Policy documents attach actions to principals and resources; access keys are credentials for those principals. | A key can authenticate a principal, but your invoice still needs an application-level account and grant ID for capability-level usage. |
| Stripe API keys | Keys can be restricted to a subset of API operations, which is useful for limiting a billing worker. | Restriction of operations does not define which logistics tenant generated a shipment event; retain tenant context in your own ledger. |
| HashiCorp Vault | Centralizes secret storage and access policies, with audit records for secret operations. | Secret access logs show who fetched a secret, not necessarily which shipment quantity should be invoiced. |
| OAuth 2.0 scopes | A standard vocabulary for delegated permissions carried in access tokens. | A scope is authorization data, not a durable billing dimension; persist the grant and event identity separately. |

These products solve different layers. IAM and OAuth describe authorization. Vault addresses secret custody. Stripe's restricted keys limit operations. None of them, by themselves, can reconstruct that customer `acme-west` generated 2,400 label renders during a particular invoice window. That reconstruction belongs in the application ledger and its tests.

## A rollout that keeps invoices explainable

Start with shadow metering. Emit grant IDs and capability names while invoices continue using the old account-level counter. Compare totals for a fixed window, investigate every mismatch, and only then switch the invoice calculation. Keep the old counter for one retention period so support can explain a transition without querying a deleted queue.

Next, exercise failure injection: timeout the secret store, delay policy propagation, duplicate queue deliveries, and submit the same provisioning request concurrently. In a 2026 rollout, I would record the bundle schema version and the reconciliation budget in every test fixture, then replay the fixtures after each migration. The acceptance criteria are concrete: one intent, one active grant per capability, no duplicate usage quantity, and a trace from invoice line to source event.

Operationally, alert on quarantined events, grant/account mismatches, and reconciliation age. Those signals are more useful than a generic error rate because they map directly to financial correctness. Access to the ledger should be read-only for most operators, with append-only writes from the metering path and audited corrections through a separate workflow.

The decision rule is narrow: use one provisioning call when the caller benefits from a bundle, but preserve independently versioned grants and an auditable usage path underneath. The call is an ergonomic boundary. It is not an accounting boundary.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://datatracker.ietf.org/doc/html/rfc6749
- https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html
- https://docs.stripe.com/keys
- https://developer.hashicorp.com/vault/docs/concepts
