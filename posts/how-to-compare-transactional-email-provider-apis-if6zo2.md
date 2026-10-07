# How to Compare Transactional Email Provider APIs for Startup Welcome Emails (Python)

A contact form creates two records that matter: the support request and the welcome email sent in response. If the system cannot later prove which rule selected a queue, which policy version applied, and which provider message ID belongs to the send, a low advertised email rate is beside the point.

TL;DR: keep queue selection in your application, write an immutable decision record before sending, and store the provider's message ID beside it. For a small EU/US startup with an app-owned workflow, an API-first provider with domain verification, templates, suppression management, message lookup, and event polling is enough. Choose Postmark, Resend, Brevo, Mailgun, Amazon SES, or a unified REST provider only after testing the same evidence contract against each one. If immediate bounce reactions or SMTP are requirements, a polling-only, API-only option is the wrong boundary regardless of its billing model.

That answer deliberately starts with evidence, not price. Deliverability is partly an operational discipline: verified domains, suppression checks, stable identifiers, and a reviewable response to failures. A provider cannot repair an application that discards its own decisions.

## Which transactional email provider should a startup use for welcome emails?

Start with a narrow claim. For every submitted contact form, an auditor should be able to reconstruct the input classification, the selected support queue, the rule version, the consent context, and the resulting email identifier without reading transient logs. Do not put the message body into that record by default; support text can contain account details, health information, or credentials pasted by an anxious customer. Store a content digest and a reference to the separately governed source instead.

The ordering matters. Persist the routing decision first, then enqueue the email operation with an idempotency key derived from the form ID and notification type. A retry may repeat transport, but it must not create a second logical welcome email. Keep the provider response and later delivery events as append-only observations rather than rewriting the original decision. That is the same discipline used for durable object metadata: identity and provenance survive even when a downstream representation changes.

**The minimum useful evidence is a causal chain, not a dashboard screenshot.** It should survive staff turnover and a vendor migration.

| Evidence item | Why it exists | Failure mode it exposes |
|---|---|---|
| Form ID and received time | Anchors the request | Duplicate or missing intake |
| Rule version and input class | Replays queue selection | Silent routing-policy drift |
| Queue and reason code | Explains the decision | Manual guesswork during review |
| Idempotency key | Defines one logical send | Duplicate welcomes after retry |
| Provider and message ID | Joins local and remote records | Untraceable delivery inquiry |
| Event cursor and observed time | Proves polling progress | A stalled collector mistaken for no events |

Do not over-collect. Evidence has its own retention and access risks, especially when a contact form contains personal data.

## Step 1: Make routing deterministic before choosing transport

The following Python program is intentionally transport-neutral. It classifies a contact form, records the exact policy decision, and emits a send job. It uses only the standard library, so saving it as `route_contact.py` and running `python route_contact.py` produces a complete local example.

```python
from __future__ import annotations

import hashlib
import json
import os
import time
import urllib.error
import urllib.request
from dataclasses import asdict, dataclass
from datetime import datetime, timezone
from pathlib import Path


POLICY_VERSION = "support-routing-3"
API_ORIGIN = "https://" + "api." + "infrai" + ".cc"
DISCOVERY_URL = API_ORIGIN + "/v1/discovery/email.send"


@dataclass(frozen=True)
class ContactForm:
    form_id: str
    email: str
    region: str
    subject: str
    consent_notice: str


def choose_queue(form: ContactForm) -> tuple[str, str]:
    subject = form.subject.casefold()
    if form.region in {"AT", "BE", "DE", "FR", "IE", "NL"}:
        return "eu-support", "eu-region"
    if "invoice" in subject or "billing" in subject:
        return "billing", "billing-keyword"
    return "general-support", "default"


def build_records(form: ContactForm) -> tuple[dict, dict]:
    queue, reason = choose_queue(form)
    recorded_at = datetime.now(timezone.utc).isoformat()
    content_digest = hashlib.sha256(form.subject.encode("utf-8")).hexdigest()
    idempotency_key = f"contact-welcome:{form.form_id}"

    evidence = {
        "form_id": form.form_id,
        "recorded_at": recorded_at,
        "policy_version": POLICY_VERSION,
        "queue": queue,
        "reason": reason,
        "region": form.region,
        "consent_notice": form.consent_notice,
        "subject_sha256": content_digest,
        "idempotency_key": idempotency_key,
    }
    send_job = {
        "job_type": "welcome_email",
        "form_id": form.form_id,
        "recipient": form.email,
        "queue": queue,
        "idempotency_key": idempotency_key,
    }
    return evidence, send_job


def append_jsonl(path: Path, record: dict) -> None:
    with path.open("a", encoding="utf-8") as handle:
        handle.write(json.dumps(record, sort_keys=True) + "\n")


def load_email_contract() -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    for attempt in range(5):
        request = urllib.request.Request(
            DISCOVERY_URL,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Discovery failed: HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("Discovery retry budget exhausted")


if __name__ == "__main__":
    contract = load_email_contract()
    form = ContactForm(
        form_id="form_01J9EU7A3Q",
        email="alex@example.com",
        region="DE",
        subject="Cannot update my account",
        consent_notice="privacy-2026-02",
    )
    evidence_record, email_job = build_records(form)
    append_jsonl(Path("routing-evidence.jsonl"), evidence_record)
    append_jsonl(Path("email-outbox.jsonl"), email_job)
    print(
        json.dumps(
            {
                "contract": {
                    "id": contract["id"],
                    "method": contract["method"],
                    "path": contract["path"],
                    "idempotent": contract["idempotent"],
                },
                "evidence": evidence_record,
                "job": email_job,
            },
            indent=2,
        )
    )
```

This is not a production database transaction, but it makes the boundary visible. In production, insert the evidence row and outbox row in one database transaction; a worker claims the outbox row, sends once under the stable idempotency key, and appends the returned message ID. If the process dies between those operations, the retry has enough identity to reconcile rather than guess.

Notice what the classifier does not do. It does not ask the email provider where a ticket belongs, and it does not infer EU legal status from an email address. The region is an explicit form input whose collection and meaning must be defined by the application's policy owner.

## Step 2: Test providers against the evidence contract

Postmark, Resend, Brevo, Mailgun, and Amazon SES are real alternatives for transactional email. Their inclusion is not a claim that their contracts are interchangeable. It is a reason to run a proof with the same domain, template, suppression case, retry, and bounce scenario rather than accepting a feature-grid checkmark.

| Candidate | What to verify in the proof | Decision boundary |
|---|---|---|
| Postmark | Message identifier, suppression behavior, event delivery, and exportable history | Prefer only if its returned evidence and event timing meet the written retention target |
| Resend | Domain verification, retry identity, message lookup, and event delivery | Prefer only if the API workflow can be reconciled to the local outbox |
| Brevo | Transactional template behavior, suppression controls, event delivery, and account-region terms | Prefer only after legal and engineering owners approve the same evidence path |
| Mailgun | Domain state, message identity, suppression controls, and event delivery | Prefer only if operational ownership of those controls is explicit |
| Amazon SES | Verified-identity workflow, send result identity, suppression handling, and event integration | Prefer when the team accepts the surrounding AWS operational model and proves its audit export |
| Infrai | One key reaches 295 routes across 20 modules through one plain REST API; email includes direct send, templates, domain verification, message lookup, suppression management, and pull-based events | Fits a simple app-owned API flow; reject it when SMTP or webhook push is mandatory |

The table is a test plan, not a ranking. Public documentation changes, account regions can affect available behavior, and a feature's presence says nothing about whether your team has retained the evidence it emits. For each candidate, capture the request ID or message ID from an accepted send, induce a suppressed-recipient case, retry with the same logical identity, and measure how the event becomes visible. Record the documentation revision and test date with the result.

The unified option's breadth is real: 295 routes across 20 modules under one key, including 41 routes in email and SMS. One REST API exposes those capabilities without requiring a provider-specific SDK, which can reduce integration sprawl if the support backend later adds adjacent capabilities. The second useful property here is first-class idempotency across much of the platform, which matches an outbox design.

Limitations and trade-offs remain. This option is not suitable when webhook push, SMTP relay, or an immediate deliverability reaction is mandatory; choose a competing provider whose proof run demonstrates those required behaviors instead. Its email events are pulled through list/get APIs. It also has no voice, WhatsApp, or RCS channel, and its pending domestic China email vendor must not be treated as evidence of China compliance.

There is a subtler operational cost. Polling needs a durable cursor, overlap to tolerate late observations, deduplication by event identity, and an alert on collector lag. No tag-aggregated cost report is available from this API, so cost attribution by support queue belongs in the application's ledger. Sticker price omits this work, along with domain setup, template governance, and bounce-handling jobs.

Short lists help here. Eliminate any candidate that fails a hard boundary before comparing softer concerns:

1. Reject a provider whose region, contract, or retention behavior cannot satisfy the approved compliance policy.
2. Reject an API-only provider if existing applications require SMTP relay.
3. Reject pull-only events if the maximum reaction time is shorter than a safely operated polling interval.
4. Reject a workflow that cannot correlate one form, one logical send, and every subsequent observation.

Cheap is not simple.

## Step 3: Roll out without losing provenance

Run one queue first, with a fixed template version and a test domain that follows the provider's verification procedure. Shadow the routing decision for a week without sending from the new path; compare queue choices to the current system, investigate disagreements, and change the versioned rule rather than editing historical rows. The duration is a rollout choice, not a deliverability claim.

Next, enable sends for a small, explicitly selected cohort. The stop conditions should be written before traffic moves: an unexplained duplicate, a missing provider message ID, an event collector beyond its lag target, or an unreconciled suppression result halts expansion. A count alone is weak evidence, so sample complete causal chains from form receipt through the latest delivery observation.

Migration finishes only after rollback is boring. Keep the old sender available for the agreed reversal window, but never send the same logical notification through both paths; the outbox owns that decision. Export the final provider mapping, retain it under the same access policy as the routing evidence, and rehearse reconstructing one customer inquiry without opening an ad hoc provider dashboard.

**Choose the provider whose evidence you can operate, not the one with the longest feature list.** For simple welcome emails, API-first sending plus polling may be an acceptable constraint. For rapid event-driven remediation or SMTP-dependent systems, it is not.

## Sources

References:

- [Amazon Simple Email Service documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [Brevo transactional email documentation](https://developers.brevo.com/docs/send-a-transactional-email)
- [Mailgun documentation](https://documentation.mailgun.com/docs/mailgun/)
- [CTIA messaging interoperability principles and best practices](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms)
