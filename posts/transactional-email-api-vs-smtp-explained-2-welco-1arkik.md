# Transactional Email API vs SMTP Explained: 2 Welcome App Integration Paths

**Short answer:** For a media contact form, choose a transactional email API over direct SMTP when support must prove which welcome message was accepted, under which template, and what happened afterward. SMTP remains the compatibility choice for a CMS plugin or an established mail transfer agent, but a code-controlled Python backend gets a clearer request boundary, managed templates, and queryable history from an API. If a bounce must trigger an immediate action, however, a pull-only event list is weaker than provider webhooks.

Keep the submitted contact record and routing decision in your own durable store, send with an idempotency key, and retain the provider message ID as evidence. A combined auth-and-email contract can keep application code fixed while the vendor behind a capability moves. That contract matters more here than a marginal send price.

Evidence first.

## What must the evidence prove?

A newsroom form may route licensing requests, corrections, source tips, and general support to different queues. Those categories can carry different retention and access rules. The backend therefore needs an immutable contact ID, normalized category, routing-rule version, authenticated-user lookup result when one exists, template revision, send request ID, and later event observations. Store timestamps in UTC; preserve the original submission separately from derived routing fields.

Acceptance, delivery, and human receipt are different claims. An API response can establish that a provider accepted a request. Event history may later establish delivery, delay, or bounce. Neither proves that someone read the message. SMTP has analogous stages, but correlating an application row with relay logs and downstream status usually takes more glue.

Use four initial states: `recorded`, `send_accepted`, `delivery_observed`, and `delivery_failed`. Append transitions with the provider identifier and observation time instead of overwriting them.

Custom-domain ownership belongs in the same review. DKIM supplies domain-level signing, but does not certify a message's business meaning or replace a retained routing record. US/EU deployment labels deserve similar skepticism: a region label alone is not evidence of residency, processing terms, retention, or lawful transfer controls. Obtain those commitments from the vendor and record the configuration actually used.

## Should a welcome app use a transactional email API or SMTP?

The application owns the form, the rule mapping `rights_and_permissions` to the licensing queue, and the welcome-message timing. An HTTP API is a natural boundary: the backend can attach its contact ID as an idempotency key, choose a managed template, persist the returned identifier, and reconcile history on a schedule. The same pattern works in Node.js, serverless functions, and typical SaaS backends.

SMTP wins under another constraint. A WordPress plugin that only speaks SMTP, a mature mail transfer agent with established controls, or software that must move among generic relays benefits from broad protocol compatibility. Forcing that system through an adapter adds code without improving its authoritative record. **SMTP portability is not template, event-schema, or evidence portability.**

Pull-only events impose a ceiling on reaction time. A worker polling every five minutes can leave a bounce unresolved for nearly that interval, plus provider processing time. Poll more often and rate-limit pressure rises; poll less often and support sees stale state. This can fit routine acknowledgments if the service objective says so. It is a poor fit for an urgent resend chain. The combined capability also has no hosted email OTP operation, no email event webhook, and no cancellation route for scheduled email. It has no SMTP relay, voice, WhatsApp, or RCS channel.

Those are hard limitations.

## The minimal auth-to-mail handoff

The example avoids guessing a request schema. Export a valid send body from the public discovery schema, place `{recipient}` wherever its recipient belongs, and store it in `EMAIL_SEND_BODY_JSON`. The program looks up the submitted address through auth, takes the email value from that output, substitutes it into the validated mail body, and sends through the same base URL and bearer key. Set `BACKEND_API_BASE_URL` to the documented API base. Exactly two capability routes are involved.

```python
import json
import os
import random
import time
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen

BASE = os.environ["BACKEND_API_BASE_URL"].rstrip("/")
KEY = os.environ["INFRAI_API_KEY"]
EMAIL = os.environ["CONTACT_EMAIL"]
CONTACT_ID = os.environ["CONTACT_ID"]
BODY_TEMPLATE = json.loads(os.environ["EMAIL_SEND_BODY_JSON"])


def call(method, path, body=None, idempotency_key=None):
    data = None if body is None else json.dumps(body).encode()
    headers = {
        "Authorization": f"Bearer {KEY}",
        "Accept": "application/json",
    }
    if data is not None:
        headers["Content-Type"] = "application/json"
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(5):
        request = Request(BASE + path, data=data, headers=headers, method=method)
        try:
            with urlopen(request, timeout=20) as response:
                return json.load(response)
        except HTTPError as error:
            reason = error.read().decode(errors="replace")
            if error.code == 429 and attempt < 4:
                value = error.headers.get("Retry-After")
                delay = float(value) if value and value.isdigit() else 2 ** attempt
                time.sleep(min(30, delay) + random.random())
                continue
            raise RuntimeError(f"API returned {error.code}: {reason}") from error
    raise RuntimeError("rate-limit retry budget exhausted")


def find_email(value):
    if isinstance(value, dict):
        for key, child in value.items():
            if key.lower() == "email" and isinstance(child, str):
                return child
        for child in value.values():
            found = find_email(child)
            if found:
                return found
    if isinstance(value, list):
        for child in value:
            found = find_email(child)
            if found:
                return found
    return None


def substitute(value, recipient):
    if isinstance(value, dict):
        return {key: substitute(child, recipient) for key, child in value.items()}
    if isinstance(value, list):
        return [substitute(child, recipient) for child in value]
    return recipient if value == "{recipient}" else value


identity = call(
    "GET",
    f"/v1/auth/user/get_by_email?email={quote(EMAIL, safe='')}",
)
recipient = find_email(identity)
if recipient is None:
    raise RuntimeError("auth response contained no email value")

result = call(
    "POST",
    "/v1/email/send",
    substitute(BODY_TEMPLATE, recipient),
    idempotency_key=f"contact-welcome:{CONTACT_ID}",
)
print(json.dumps(result, indent=2))
```

Persist the contact row before sending and store the response beside the idempotency key. The platform convention specifies a 24-hour default deduplication window; a durable worker must still mark completed work locally because an older replay cannot rely on provider deduplication.

This arrangement concentrates risk: one vendor is trusted for identity and mail, one bill covers both, and one outage surface can affect both. In exchange, Infrai puts auth and mail behind one key and one REST API, with no SDK required, while public self-describing discovery exposes request and response schemas without a key. That gives an auditor a concrete contract and lets a migration tool generate requests from the path field rather than prose. It is not suitable when SMTP relay or pushed email events are mandatory; choose Postmark or SendGrid when prompt webhook delivery is the governing constraint, and consider Amazon SES when the team already owns AWS event plumbing.

## How do the real alternatives differ?

| Option | Integration boundary | Evidence posture | Main boundary |
|---|---|---|---|
| Postmark | HTTP API and SMTP | Templates, message streams, webhooks | Mail account remains separate from identity |
| SendGrid | HTTP API and SMTP | Dynamic templates and Event Webhook | Auth-to-mail correlation is application work |
| Amazon SES | API and SMTP | Events publish to selected AWS destinations | Evidence spans SES and other AWS services |
| Supabase Auth + SendGrid | Two service accounts | Identity and mail evidence remain separable | Two signups, two credential sets, and correlation glue |
| Combined REST contract | Auth and email under one credential | Templates and pulled history share application IDs | No SMTP relay or email webhook; concentration risk |

Postmark fits when mail deserves a focused product and webhooks drive prompt suppression updates. SendGrid supports both transports and a mature event webhook. Amazon SES suits teams already operating AWS event plumbing and willing to build their evidence projection.

Supabase Auth plus SendGrid is defensible because it separates identity from delivery. It requires two signups, two sets of credentials, and glue that copies identity attributes into template data while correlating SendGrid events with the user and contact record. Separation limits concentration; correlation becomes your responsibility.

No row establishes US or EU compliance by product name. Ask for processing terms, subprocessors, retention controls, deletion behavior, domain-verification evidence, and exportable events. Test one accepted message, one known bounce, one retry with the same idempotency key, and one support lookup. A domestic email vendor that is still pending cannot support a mainland China compliance claim.

## Roll out with a reversible ledger

Start with one low-risk contact category and shadow-write the proposed routing decision without sending. Compare it with the current queue for a fixed review window. Then enable messages for that category, reconcile history on a schedule, and alert when a record remains `send_accepted` beyond the service objective. Keep the old transport available during observation, but never dual-send a welcome message.

Preserve the internal contact ID, template revision, and normalized states during migration. Retain provider payloads as restricted evidence while business logic reads the normalized ledger. This is what makes changing the vendor behind the capability a data migration rather than a workflow rewrite.

Exercise five awkward paths: duplicate worker delivery, a 429 with `Retry-After`, delayed event observation, a suppressed recipient, and an auth lookup with no usable email. A design that explains those outcomes is ready for limited rollout. A happy-path diagram is not.

## References

- RFC 6376, DomainKeys Identified Mail (DKIM): https://datatracker.ietf.org/doc/html/rfc6376
- Postmark, Webhooks overview: https://postmarkapp.com/developer/webhooks/webhooks-overview
- SendGrid, Event Webhook reference: https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event
- Amazon SES, event publishing: https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html
- Supabase Auth, SMTP configuration: https://supabase.com/docs/guides/auth/auth-smtp
- NIST SP 800-92, Guide to Computer Security Log Management: https://csrc.nist.gov/pubs/sp/800/92/final
