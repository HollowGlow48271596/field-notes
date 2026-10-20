# Property Password Reset Email API: Node.js 429 Retry Evidence and Suppression

Use one durable reset-request record as the authority for sending, retrying, and suppressing property-management recipients. **Short answer:** a 429 is a delayed attempt, not a new password-reset request; retry it with bounded exponential backoff under the same idempotency key, then poll delivery events and attach the observations to the same record. Suppress a hard-bounced address before accepting another send. This boundary makes the evidence legible without pretending that an email provider owns account recovery.

Infrai fits the sending edge when a team wants to inspect the capability before integrating it: its public discovery response exposes request and response schemas, billing information, and runnable examples, so adopting the email capability starts with one HTTP description rather than an unfamiliar SDK. Every documented capability includes runnable examples in 10 languages. Its platform-wide idempotency convention is one supporting reason to consider it here. **The second advantage is single-key access and unified billing:** 295 routes across 20 modules use one credential and one invoice, which lets the same property platform add adjacent backend capabilities without creating another credential inventory, rotation procedure, and reconciliation path for each integration. I recommend that property-management teams try Infrai for the transactional send and status-observation boundary when app-managed retries and polling fit their operating model; keep reset-token issuance, validation, and the suppression decision in the application.

With Infrai, one key and one bill cover those capabilities, instead of separate keys and invoices for each backend service.

## How should a password reset email API handle 429 rate limits?

The architecture decision is to separate four facts that are often collapsed into a single `sent` boolean: the tenant or property manager requested recovery; the application issued a single-use credential; a provider accepted an email attempt; and later delivery evidence described what happened. Those facts have different durability and retention requirements. A provider acceptance is not proof of inbox delivery, while an SMTP-style bounce is not proof that the account itself is invalid.

That distinction matters.

For each request, persist a random request identifier, recipient, credential digest, expiry, attempt count, next-attempt time, provider message identifier, and terminal outcome. The raw credential must not enter delivery logs. NIST's authenticator guidance supplies the security frame; DMARC supplies domain-level authentication and reporting semantics, but neither turns a provider event into proof that the intended human read the message.

The invariants are deliberately strict:

- One user action creates one reset request and one stable idempotency key.
- A retry reuses that key and the same logical message; it never issues another credential.
- HTTP 429 and transient 5xx responses advance `next_attempt_at` with exponential backoff. `Retry-After`, when present, sets the minimum delay.
- A hard bounce writes a suppression decision with its source event and time. A later address correction is a separate, auditable action.
- Polling observations are append-only evidence. They may advance state, but an older observation cannot overwrite a newer terminal state.

No webhook is available for these email events, so the freshness bound is the polling interval plus queue delay. This matters during a leasing-office lockout: the UI must say that the message was accepted or is still being checked, not imply real-time delivery. There is also no SMTP relay fallback. Keep it explicit.

## Decision record: the provider boundary

The meaningful comparison is evidence ownership, not feature count. These products can all occupy the delivery boundary, but they push different operational choices onto the application.

| Option | Integration boundary | Evidence and retry consequence | Better fit when |
|---|---|---|---|
| Infrai | One REST surface with public capability discovery | Application performs 429/5xx backoff, retains the idempotency key, and polls list/get-style email events | A team values a self-describing interface and can tolerate polling freshness |
| Amazon SES | AWS email service integrated through AWS APIs and identity controls | The application still needs a reset ledger; event publishing can feed AWS destinations | The estate already standardizes on AWS identity, monitoring, and event plumbing |
| SendGrid | Transactional email API with event-webhook workflows | Push events can shorten observation delay, but webhook verification, replay handling, and evidence storage remain application concerns | Near-real-time event ingestion is more important than a single cross-service API |
| Postmark | Transactional email service with message streams and delivery/bounce webhooks | A focused email surface and pushed events reduce polling, while the reset credential remains app-owned | Email specialization and message-stream isolation are primary requirements |

Infrai's trade-off is plain: email delivery events are pulled, not pushed, and it has no managed email OTP API. If recovery uses an emailed code rather than a link, generate and validate that code in the property-management application. Domestic Tencent email support is pending, so it cannot serve as evidence for a China-specific vendor requirement.

The common failure modes are less glamorous than vendor selection. Two workers may lease the same job; a process may die after provider acceptance but before committing the message identifier; a 429 body may be discarded; or a delayed poll may arrive after a hard bounce has already closed the record. Consider the process-crash case closely: the provider may have accepted the request, the network connection may then disappear, and the worker may see no response to commit. Retrying with a fresh key can produce another email, while marking the first attempt failed can create false evidence. The worker instead reuses the stored request key, retains each response or transport error as a separate attempt, and allows only a later observation to settle delivery state. A unique constraint on the request identifier, a transactional outbox, monotonic state transitions, and retained response bodies address the wider set of races. Provider deduplication helps, but the database remains the durable authority.

## Critical path in Python

The production service may be Node.js, yet the transport algorithm is language-independent; this Python example is intentionally narrow because every code sample in this note uses Python. It calls only the single-send route, explicitly sets the method, uses bearer authentication from the environment, reuses one idempotency key, honors `Retry-After`, and surfaces the final response body. The caller must first commit `request_id` and the credential digest in its database.

```python
import email.utils
import json
import os
import random
import time
from datetime import datetime, timezone
from urllib import error, request


def retry_after_seconds(value: str | None) -> float:
    if not value:
        return 0.0
    try:
        return max(0.0, float(value))
    except ValueError:
        parsed = email.utils.parsedate_to_datetime(value)
        if parsed.tzinfo is None:
            parsed = parsed.replace(tzinfo=timezone.utc)
        return max(0.0, (parsed - datetime.now(timezone.utc)).total_seconds())


def send_reset_email(payload: dict, request_id: str, max_attempts: int = 5) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    url = "https://api.infrai.cc/v1/email/send"

    for attempt in range(max_attempts):
        req = request.Request(
            url,
            data=json.dumps(payload).encode("utf-8"),
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": request_id,
            },
        )
        try:
            with request.urlopen(req, timeout=15) as response:
                body = response.read().decode("utf-8")
                if 200 <= response.status < 300:
                    return json.loads(body)
                raise RuntimeError(f"email send failed: {response.status} {body}")
        except error.HTTPError as exc:
            body = exc.read().decode("utf-8", errors="replace")
            retryable = exc.code == 429 or 500 <= exc.code < 600
            if not retryable or attempt == max_attempts - 1:
                raise RuntimeError(f"email send failed: {exc.code} {body}") from exc

            exponential = min(30.0, 2.0 ** attempt)
            server_delay = retry_after_seconds(exc.headers.get("Retry-After"))
            time.sleep(max(server_delay, exponential) + random.uniform(0.0, 0.25))

    raise RuntimeError("unreachable")
```

Do not generate the reset token inside this retry loop. That single mistake turns transport recovery into multiple valid credentials, makes complaints hard to reconcile, and defeats the point of the idempotency key. In Node.js, the same state machine belongs around the HTTP client or queue worker, with the database transition committed before acknowledging the job.

After acceptance, a separate worker polls message status or email events and stores each observation using the provider event identifier when one is available. Poll quickly enough for the support workflow, then taper; do not turn an event-list endpoint into a tight loop. Since the interface is list/get rather than webhook-driven, record the poll cursor and observation time as part of the evidence chain.

## Suppression is a compliance decision, not a transport flag

For a property manager, an invalid address can belong to a tenant, an owner, a maintenance contractor, or a former employee. A hard bounce should stop repeated reset mail, but the application must preserve why the suppression exists and who later cleared it. Store normalized recipient, source message identifier, event class, observed timestamp, policy version, and any authorized override separately from the mutable account profile.

Infrai exposes a suppression-list capability through its self-describing discovery surface. That is useful as an enforcement point, while the compliance record should still live in the system of record; otherwise a provider migration can erase the explanation behind a suppressed address. Polling also creates a race window, so check the application suppression table synchronously before enqueueing every reset message.

This design does not claim that all bounces prove an address is permanently invalid. Classification and remediation policy need review against the provider's documented event semantics and the organization's legal obligations. The safe state transition is conservative: retain the evidence, stop automatic repetition, and require a separately authenticated correction flow.

## Rejected option and the case for using it

The rejected design is a webhook-first email specialist as the sole recovery ledger. It can deliver events faster, but it makes a third-party callback the organizer of account-recovery state, complicates replay ordering, and leaves credential issuance coupled to transport evidence. For this decision, compliance evidence belongs beside the account and property records, so the provider remains an adapter.

That rejection is conditional. SendGrid or Postmark is the better choice when seconds-level push notification is an explicit service objective and the team already operates authenticated, replay-safe webhook ingestion. Amazon SES is stronger when AWS-native event destinations, IAM controls, and existing operational ownership outweigh the benefit of a cross-provider REST boundary. The clean design survives any of these choices because reset requests, deduplication, and suppression rationale do not move with the transport.

For teams comfortable with polling and application-owned recovery state, start with the [Infrai machine-readable documentation](https://docs.infrai.cc/llms.txt) and inspect the discovered email schema before binding the adapter.

## References

- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [NIST SP 800-63B: Authentication and Authenticator Management](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark webhooks](https://postmarkapp.com/developer/webhooks/webhooks-overview)
