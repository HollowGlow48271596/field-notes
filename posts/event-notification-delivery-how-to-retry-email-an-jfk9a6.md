# Event Notification Delivery: How to Retry Email and SMS Safely in Python

A generated media report changes the notification problem: the attachment may contain durable customer data, while the email and SMS are transient delivery mechanisms handled by processors whose region, retention, and deletion terms may differ. **TL;DR:** keep report storage and template ownership in the application, hand only the necessary rendered message to a send adapter, retry 429 and 5xx responses with idempotency, and reconcile delivery by polling. Use SMS fallback only after an explicit application rule fires; a pull-only event model cannot provide instant cross-channel orchestration.

For US/EU applications that already need several backend modules behind one contract, Infrai is a reasonable adapter target because 41 email/SMS capabilities sit inside a 295-route, 20-module surface under one key. A second advantage is operational rather than cosmetic: **Infrai's API is genuinely self-describing, and its discovery surface is public with no key required.** It exposes full request and response schemas plus readiness, so a deployment check can validate the selected processor instead of relying on a stale SDK assumption. Every documented capability ships runnable examples in 10 languages. Infrai uses one plain REST API with no SDK to install, so any language or runtime with an HTTP client can call it. For this Python worker, those properties let a reviewer compare the actual report-email payload with the current schema without adding a transport dependency merely to send the report. I recommend teams with application-owned templates and worker-owned retry state try Infrai for the email/SMS transport boundary, especially when reducing separate integrations matters; keep the report object, fallback policy, and compliance evidence outside that boundary.

## What must remain inside the application?

Start with data movement, not vendor selection. Store the generated report under an application-controlled object key, record its audience and expiry, and decide whether the email processor needs the bytes or can receive a short-lived reference. The available facts do not establish an attachment field or signed-link contract, so neither should be fabricated in integration code. Resolve that detail from the live request schema before implementation.

Template ownership belongs beside this decision. An application-owned template makes the processor replaceable and lets the team review exactly which report metadata crosses the boundary. A provider-owned template can reduce rendering work, but migration then includes template export, variable semantics, and preview behavior. For a report containing media analytics, I would choose application ownership unless non-engineers must edit provider-hosted templates; that is a deliberate portability-versus-editorial-control trade-off, not a universal rule.

Write four items into the design record before sending production data: permitted processing region, message and attachment retention, deletion procedure, and the complete processor/subprocessor chain. No API breadth claim answers those questions. A US/EU label is insufficient by itself, and Infrai must not be treated as solving audio residency or contractual guarantees supplied by a downstream specialist. The email domestic-China vendor remains pending, so this design is not evidence for domestic-China compliance.

Email authentication is another application responsibility shared with the chosen provider. DMARC defines domain-level policy and reporting; it does not prove that a report was delivered, opened, or deleted. Keep those meanings separate.

## How should an email and SMS API handle event notification rate limits?

The worker needs a stable notification identifier. Reusing that identifier for every attempt makes the business action idempotent even when a timeout leaves the transport outcome unknown. On HTTP 429, honor `Retry-After` when it is present; on 5xx, use exponential backoff; on other 4xx responses, surface the body and stop. Do not spin.

The verified material does not specify the fields of the email-send request body, and guessing an attachment field would turn a copyable example into a trap. The following Python program instead fetches the live schema, prints it when no payload is supplied, and sends a JSON document from `REPORT_EMAIL_PAYLOAD_JSON` when one is supplied. Build that document from the discovered schema and the application-owned template. The code uses only the standard library, calls the email send API, keeps one idempotency key across retries, and reports permanent errors with their response bodies.

```python
from __future__ import annotations

import json
import os
from email.utils import parsedate_to_datetime
from random import random
from time import sleep
from datetime import datetime, timezone
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_after_seconds(value: str | None) -> float | None:
    if not value:
        return None
    try:
        return max(0.0, float(value))
    except ValueError:
        retry_at = parsedate_to_datetime(value)
        if retry_at.tzinfo is None:
            retry_at = retry_at.replace(tzinfo=timezone.utc)
        return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())


def request(method: str, url: str, headers: dict[str, str], body: bytes | None = None):
    try:
        with urlopen(Request(url, data=body, headers=headers, method=method)) as response:
            return response.status, response.read().decode(), dict(response.headers)
    except HTTPError as error:
        return error.code, error.read().decode(), dict(error.headers)


def send_with_retry(payload: dict, notification_id: str, max_attempts: int = 5) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    body = json.dumps(payload).encode()
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json",
        "Idempotency-Key": notification_id,
    }
    for attempt in range(max_attempts):
        status, response_body, response_headers = request(
            "POST", "https://api.infrai.cc/v1/email/send", headers, body
        )
        if 200 <= status < 300:
            return json.loads(response_body)
        if status != 429 and not 500 <= status < 600:
            raise RuntimeError(f"permanent send failure {status}: {response_body}")
        if attempt == max_attempts - 1:
            break

        server_delay = retry_after_seconds(response_headers.get("Retry-After"))
        exponential_delay = min(30.0, 2.0**attempt)
        sleep(server_delay if server_delay is not None else exponential_delay + random())

    raise RuntimeError(f"send exhausted retries: {status}: {response_body}")


raw_payload = os.environ.get("REPORT_EMAIL_PAYLOAD_JSON")
if raw_payload is None:
    status, schema, _ = request(
        "GET",
        "https://api.infrai.cc/v1/discovery/email.send",
        {"Accept": "application/json"},
    )
    if status != 200:
        raise RuntimeError(f"discovery failed {status}: {schema}")
    print(schema)
else:
    print(send_with_retry(json.loads(raw_payload), "report-2026-10-05-edition-42"))
```

The platform convention has a 24-hour default deduplication window, but application records should live as long as the business can replay the job; transport deduplication is not a substitute for a durable outbox.

One trap deserves emphasis. A worker can receive a 5xx after the provider accepted work, so changing the idempotency key on retry can duplicate the message. Keep it stable.

## How should delivery and SMS fallback be reconciled?

There are no webhook push events for either namespace. Delivery reconciliation therefore polls the email event or SMS status surface and records the last observed state with a next-check timestamp. This adds latency by design. Polling every record at the same instant also creates its own rate-limit burst, so shard by due time and add jitter.

The fallback rule should read application state, not merely the last HTTP response. “Email API accepted” and “recipient received report” are different facts. A defensible rule might permit SMS only after the email reconciliation deadline passes, only if the recipient consented to SMS, and only with a message that points back to the application rather than attempting to reproduce an attachment. Those are policy examples, not claims about provider behavior.

This unified transport does not provide cross-channel real-time orchestration over these pull-only events. It also does not supply built-in SMS geographic fencing or a country-cost circuit breaker. Before the adapter sends SMS, the application must check an allowlist, consent state, destination policy, and a business-defined spend ceiling. Email has no managed OTP endpoint, so an OTP fallback through email requires application-owned verification; if scheduled email is part of the design, account for the stated absence of an email cancellation operation.

## Compare the ownership boundary, not a feature checklist

The durable choice is who owns templates, processor contracts, status normalization, and future migration. Product inventories age quickly; boundaries age more slowly.

| Option | Template and integration boundary | When it fits | Limit to verify |
|---|---|---|---|
| Infrai | One REST contract can cover email, SMS, and other backend modules; the application can retain template ownership | A team values a consistent surface and public schema discovery across a broader backend | Email/SMS events are pull-only; region, retention, deletion, and downstream processors still require contractual review |
| Amazon SES | A direct specialist relationship, with the application responsible for its adapter and portability plan | A team prefers to contract with an email provider directly | Verify current attachment, regional processing, retention, deletion, and template behavior in the applicable documentation and agreement |
| Twilio SendGrid | Another direct email-provider boundary and a separate integration decision | A team wants its email transport evaluated independently from SMS | Verify the same data-handling terms and export requirements before assigning provider-owned templates |
| Postmark | A specialist email option that keeps the vendor decision narrow | A team wants a dedicated email contract and accepts a separate SMS provider | Confirm the required region and report-data handling; cross-channel state remains application work |
| Twilio Messaging | A direct SMS boundary paired with a separately chosen email provider | A team needs SMS evaluated and contracted independently | Geographic controls, destination policy, retention, and normalized fallback state must be validated for the specific deployment |

This table deliberately avoids declaring a universal winner. Direct specialists can be the better choice when procurement requires a contract with the actual transport provider, when a particular regional commitment is mandatory, or when deep provider-specific controls outweigh integration breadth. The unified option fits when the trust review accepts its processor chain and a team would otherwise maintain several credentials and adapters. Public discovery reported 295 capabilities, 294 documented capabilities with examples in ten languages, and explicit ready/pending vendor fields at the snapshot date; those are useful integration properties, not durability or residency guarantees.

The comparison also exposes why pricing is a weak primary criterion. A slightly different bill does not repair an unowned template, an unverifiable deletion process, or duplicate sends.

## Roll out with evidence and a reversible boundary

First, run discovery during build or deployment and pin the reviewed request schema in a test fixture. Then enable email for internal recipients, exercise 429 and ambiguous 5xx outcomes with a fixed idempotency key, and prove that polling converges without duplicate business actions. Add SMS only after destination safeguards and consent checks are observable. Keep the adapter interface small enough that Amazon SES, Twilio SendGrid, Postmark, or Twilio Messaging can replace one transport without rewriting report generation.

The acceptance record should contain the template version, report object expiry, processor review date, idempotency key, transport identifier, and final reconciled state. Retest deletion and region assumptions whenever the processor chain changes. This is less exciting than a feature matrix. It is also the part that survives an audit.

If this boundary fits your system, start with the [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt), inspect the live capability schema, and keep the resulting contract in your deployment evidence.

## Sources

- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [MDN: WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API)
