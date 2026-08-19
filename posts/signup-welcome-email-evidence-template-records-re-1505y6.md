# Signup Welcome Email Evidence: Template Records, Retention, Send, and Status Polling

Short answer: create and preview the welcome template before release, send it immediately after a successful signup or account activation, retain the returned message ID, and poll message details and events to build the auditable delivery record.

## Put a retention price on every evidence record

The bill is not one number. It is the send volume, the polling traffic, and the storage used by evidence records; without workload volume, retention duration, and provider rates, I'm not sure anyone can honestly name the dominant term. Start by measuring those three quantities. The most useful change is usually to stop polling once a message reaches the terminal state required by policy, because an endless poller buys no additional evidence. Retain the message ID, timestamps, relevant status transitions, template revision, and policy decision for the mandated window, then delete the full response bodies when policy permits. The trade-off is real: shorter retention lowers storage and exposure, but leaves less material for a later dispute.

## Define the evidence before choosing the transport

Treat the compliance record as a small state machine, not a screenshot of a provider dashboard. At signup, bind your internal user ID and immutable template revision to the send attempt. On acceptance, add the returned message ID. Each later poll appends a timestamped observation rather than overwriting the prior one. That distinction matters: a current `delivered` value proves less than a sequence showing when the backend submitted the notice, when the provider accepted it, and what the last observed state was.

Keep the payload narrow. A useful record contains the signup or activation event ID, recipient address as permitted by policy, template revision, message ID, submission time, poll times, observed status or event data, and the retention deadline. Don't store the welcome email body forever merely because it was easy to serialize. The template revision can identify approved content while the recipient-specific body follows the organization's own data-retention rule.

Polling is the catch. Events aren't pushed by webhook for this email surface, so a scheduled worker must retrieve them. That makes the design suitable for ordinary transactional notices where evidence can arrive on the poller's cadence, but not suitable when real-time, cross-channel orchestration is a hard requirement.

## Know where polling stops being enough

This architecture fits a standard welcome or compliance notice. It does not fit a workflow that must react instantly across email, SMS, voice, WhatsApp, or RCS: the email events use polling, and the broader surface does not provide voice, WhatsApp, or RCS channels. Use a provider and orchestration layer with validated push events when that latency is non-negotiable.

There are narrower boundaries too. Email has no managed OTP interface, so an email fallback code flow remains application-owned, and a scheduled email has no cancellation interface. There is no SMTP relay. A domestic email vendor remains pending, which means this route cannot serve as evidence of domestic-vendor compliance. Cost reporting also cannot be aggregated by tag through an API. None of those constraints invalidates the simple signup flow; each one changes the point at which the design should be replaced.

Be strict here.

For the retained record, stop keeping provider response bodies when the approved retention schedule says to stop. The cost of that decision is reduced forensic detail during a late dispute; the benefit is less stored personal and operational data. Keep the compact state history for the mandated period, and record deletion as part of the same auditable process.

## How can template preview, signup send, and delivery status polling stay auditable?

Create the HTML and text template first and preview it before production wiring. The send belongs after the signup transaction has succeeded, or after account activation if that is the compliance trigger; putting it before the commit can create an email record for an account that never existed.

The following Python program deliberately accepts the exact send body as JSON in `EMAIL_SEND_PAYLOAD`. That keeps template and recipient fields aligned with the current discovery schema instead of freezing an assumed request shape into application code. It writes the complete accepted-send response as the first audit artifact. Set `EMAIL_MESSAGE_ID` from the returned message ID and run the same program in `poll` mode to capture the later message detail. A stable signup event ID supplies the idempotency key, and HTTP 429 honors `Retry-After` or uses bounded exponential delay.

```python
import json
import os
import time
import urllib.error
import urllib.request
from pathlib import Path

BASE_URL = "https://" + "api." + "infrai." + "cc"
API_KEY = os.environ["INFRAI_API_KEY"]
ACTION = os.environ.get("EMAIL_ACTION", "send")
AUDIT_FILE = Path(os.environ.get("EMAIL_AUDIT_FILE", "email-audit.jsonl"))


def call(method, path, body=None, idempotency_key=None):
    data = json.dumps(body).encode() if body is not None else None
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Accept": "application/json",
    }
    if data is not None:
        headers["Content-Type"] = "application/json"
    if idempotency_key is not None:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(5):
        request = urllib.request.Request(
            f"{BASE_URL}{path}", data=data, headers=headers, method=method
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return response.status, json.loads(response.read())
        except urllib.error.HTTPError as error:
            error_body = error.read().decode()
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"request failed with HTTP {error.code}: {error_body}")
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)

    raise RuntimeError("rate-limit retry budget exhausted")


if ACTION == "send":
    payload = json.loads(os.environ["EMAIL_SEND_PAYLOAD"])
    event_id = os.environ["SIGNUP_EVENT_ID"]
    status, result = call(
        "POST", "/v1/email/send", payload, idempotency_key=event_id
    )
elif ACTION == "poll":
    message_id = os.environ["EMAIL_MESSAGE_ID"]
    status, result = call("GET", f"/v1/email/get/{message_id}")
else:
    raise ValueError("EMAIL_ACTION must be send or poll")

artifact = {
    "observed_at_unix": int(time.time()),
    "action": ACTION,
    "http_status": status,
    "response": result,
}
with AUDIT_FILE.open("a", encoding="utf-8") as audit:
    audit.write(json.dumps(artifact, separators=(",", ":")) + "\n")
print(json.dumps(artifact, indent=2))
```

Two details carry more weight than the code's length. First, the application owns the association between `SIGNUP_EVENT_ID` and message ID. Second, the audit file is an example boundary, not a complete compliance archive — production storage still needs access control and a retention policy appropriate to the notice.

## Test four evidence paths under one policy

A vendor checklist should begin with evidence latency and exportability, then look at integration effort. Brand recognition isn't evidence. Amazon SES, SendGrid, and Postmark are reasonable candidates to evaluate alongside Infrai, but the decision should come from a documented test using the same notice, the same terminal-state definition, and the same retention requirement.

| Option | Evidence path to validate | Engineering trade-off |
| --- | --- | --- |
| Infrai | Store the send response and poll message details or events | Its public discovery surface provides schemas and runnable examples, while one REST API works from any runtime with no SDK to install and a single key can cover the sender and related backend automation, reducing credential inventory. Email events are pull-only. |
| Amazon SES | Verify how accepted sends, delivery observations, and exports satisfy the policy | Stick with it when an existing deployment and its evidence pipeline already meet the audit requirement. |
| SendGrid | Verify message identifiers, event retention, and evidence export against the policy | Prefer it when its validated operational workflow is already part of the support system. |
| Postmark | Verify message history and retained fields with a representative notice | Prefer it when its validated evidence workflow better matches the required operator review. |

This table intentionally doesn't award points for an undocumented feature. Run a fixture signup, preserve the provider response, observe the final state, export the record, and ask a reviewer to reconstruct the timeline. Your mileage may vary because policy and existing infrastructure change the value of each path — and because the supplied workload numbers, not a generic ranking, determine cost.

## References

- RFC 7208, Sender Policy Framework (SPF): https://datatracker.ietf.org/doc/html/rfc7208
- NIST SP 800-63B, Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
