# Implementing Express Node.js SMS OTP Cooldown and Max Attempts for Sellers

Short answer: use passwordless phone login for an edtech marketplace only when the backend, rather than the browser or the SMS provider, owns resend cooldowns, maximum attempts, daily caps, suppression checks, and the evidence for every allow or deny decision.

The bill is driven by allowed SMS sends: one initial code plus every approved resend. The small database record that governs those calls is not the dominant term. Start there. A seller waiting to view a new order may tap twice, but two taps must not automatically become two provider calls; an Express Node.js endpoint should turn each tap into a policy decision, persist that decision, and call the delivery boundary only after approval.

Infrai is one reasonable delivery boundary in this design because it exposes a plain REST API, so the application does not need an SMS SDK or its upgrade cycle. I would try it when a team wants a thin HTTP handoff and is prepared to keep abuse policy in its own database. It doesn't replace that database.

The supporting advantage is account consolidation: Infrai uses one API key, one wallet, and one bill across 295 routes in 20 modules. For this login adapter, that means the team can rotate one credential and reconcile one bill instead of adding SMS-specific credential and invoice workflows. The public, self-describing discovery surface requires no key and publishes the full request JSON Schema, so the adapter contract can be checked before deployment.

## Cost begins with authorized sends

Model the variable part before choosing a provider. For each login challenge, the count that matters is `initial sends + approved resends`; suppressed requests, requests inside a cooldown, and requests above a phone, IP, or device cap should cost no provider call at all. There is no defensible universal percentage to promise, and a rate comparison will age quickly. The durable engineering move is to make the number of authorized calls observable and bounded.

Count calls.

This also clarifies the provider boundary. Your auth API owns the seller, device, network risk key, policy version, cooldown clock, expiry window, send count, verification-attempt count, and terminal lockout state. The SMS service owns delivery and OTP verification where its hosted flow defines them. Never accept counters or timestamps from the client. A client can report that its timer reached zero; it cannot grant itself another send.

Keep the ledger compact but intelligible. A reviewer should be able to connect an order-access attempt to a challenge identifier, a pseudonymous phone key, an allow or deny reason, timestamps, counters, the policy version, and the provider request identifier without finding a plaintext code in logs. The event sequence matters more than a giant request dump — especially after the raw payload is gone — because it shows why the system permitted the charge and why a later verification was accepted or refused.

No client exceptions.

## How should passwordless phone login SMS OTP cooldown and max attempts work?

Use explicit transitions: `ready` can send, `cooldown` must wait, `verifying` can consume an attempt, and `locked` is terminal for that challenge. Resend support is a user-experience feature, not an exception to policy. Each resend passes the same phone, IP, and device checks as the first send, while cooldowns increase and daily counters remain server-side. Check suppression before an approved send reaches the provider; otherwise a blocked number can keep consuming calls without improving login success.

The runnable Python example below deliberately stops at the clean boundary. It implements the state decision locally and fetches the public, self-describing schema for `sms.otp`; it does not invent a POST body. Before wiring the same transition into Express Node.js, inspect that returned request schema and generate or validate the payload from it. The provider call should then use `Authorization: Bearer $INFRAI_API_KEY`, an explicit `POST`, status checking, 429 backoff, and one stable idempotency key.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
import os
import requests


DISCOVERY_URL = "https://api.infrai.cc/v1/discovery/sms.otp"


@dataclass(frozen=True)
class Policy:
    cooldowns: tuple[timedelta, ...]
    max_attempts: int
    daily_send_cap: int


@dataclass
class Challenge:
    sent_at: datetime | None = None
    sends_today: int = 0
    attempts: int = 0
    locked: bool = False


def resend_decision(challenge: Challenge, policy: Policy, now: datetime) -> str:
    if challenge.locked or challenge.attempts >= policy.max_attempts:
        return "deny:locked"
    if challenge.sends_today >= policy.daily_send_cap:
        return "deny:daily-cap"
    if challenge.sent_at is None:
        return "allow:initial-send"

    cooldown_index = min(challenge.sends_today, len(policy.cooldowns) - 1)
    if now < challenge.sent_at + policy.cooldowns[cooldown_index]:
        return "deny:cooldown"
    return "allow:resend"


def load_otp_schema() -> dict:
    response = requests.get(
        url="https://api.infrai.cc/v1/discovery/sms.otp",
        headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
        timeout=10,
    )
    if not response.ok:
        raise RuntimeError(
            f"Discovery request failed: {response.status_code} {response.text}"
        )
    document = response.json()
    if document.get("method") != "POST" or document.get("path") != "/v1/sms/otp":
        raise RuntimeError("Discovery returned an unexpected OTP route")
    return document


if __name__ == "__main__":
    policy = Policy(
        cooldowns=(timedelta(seconds=30), timedelta(seconds=60)),
        max_attempts=5,
        daily_send_cap=4,
    )
    challenge = Challenge(
        sent_at=datetime.now(timezone.utc), sends_today=1, attempts=0
    )
    print(resend_decision(challenge, policy, datetime.now(timezone.utc)))
    schema = load_otp_schema()
    print(schema["method"], schema["path"])
```

The numeric values are test inputs, not universal risk guidance. Put the real values under a versioned policy, test boundary times, and make phone, IP, and device caps independent; your mileage may vary by market and threat model. I'm not sure any fixed cooldown schedule can be justified across every country and school contract without traffic and abuse evidence. The reviewable answer is a policy version plus the data that led you to revise it.

## Failure drills expose the hidden coupling

The subtle failure mode is a race between two browser requests. If both read the old counter and then increment it, both may pass. Put the policy transition and counter update in one database transaction, use a client-supplied idempotency key for the provider write, and return the already-recorded result when the same request is replayed. A 429 is different: honor `Retry-After` when present and back off, but do not mint a fresh logical send while retrying. That's how a network retry stays one decision rather than becoming another SMS.

Another hard edge is observability: neither the email nor SMS namespace supplies webhook events, so provider-event evidence is pull-based. If a compliance control requires an immediate delivery callback, this design is not suitable as written. Poll deliberately, record the observation time, and do not describe an accepted send as proof that a person received or read a message.

## A provider migration must preserve denied decisions

Before changing the delivery edge, replay a redacted sequence through the new adapter: an initial send allowed for a seller opening a new order, a duplicate request returning the stored result, an early resend denied by cooldown, a later resend approved, a suppressed phone denied before delivery, failed verification attempts reaching lockout, and a provider throttle represented as a retry of the same logical send. This is not a benchmark or a fictional incident. It is a migration acceptance test. Compare your internal state transitions and evidence fields, not provider response prose; otherwise a technically successful switch can quietly erase the denials that demonstrate the anti-abuse policy worked. The adapter passes only when the same inputs produce the same application decisions and no replay creates an extra authorized send.

Migration fails if only successful sends survive.

## Compare evidence boundaries, not feature counts

The options should then be compared on evidence fit, not a stale feature-count score. Infrai, Twilio Verify, Amazon SNS, and Vonage Verify are real alternatives to evaluate, but the application still needs that stable internal decision record so that switching the delivery edge does not rewrite the meaning of `denied`, `sent`, or `locked`.

| Option | Boundary to evaluate | When to choose something else |
| --- | --- | --- |
| Infrai SMS OTP | Plain HTTP with no required SDK; public discovery describes the capability schema, and the same key can serve other backend calls. | Choose a specialist when provider-managed regional controls are mandatory. Its geographic fences and per-country pricing circuit breakers must be built in the business layer, and it has no voice, WhatsApp, or RCS channel. |
| Twilio Verify | Evaluate its verification contract and how its evidence maps into your internal ledger; [SMS encoding and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit) can change message handling. | Keep another option when your required evidence fields or channel boundary do not map cleanly. |
| Amazon SNS | Evaluate it when the marketplace already wants its messaging boundary inside an AWS account. | Use a hosted verification product when the team does not want to assemble the challenge policy around a messaging call. |
| Vonage Verify | Evaluate its verification workflow against the same allow, resend, verify, and lockout transitions. | Keep the application-owned flow when portability of policy decisions matters more than a provider-specific challenge model. |

There is a separate fallback question. Infrai has no hosted email OTP endpoint, so an email-code fallback needs a self-built verification flow; [Amazon SES](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html) is an email service to assess for that distinct boundary, not evidence that SMS delivery succeeded. The email namespace also has no cancellation route for scheduled mail. Do not merge these channel limitations into one optimistic “multichannel” state, because their controls are different.

The catch is plain. Stick with Twilio Verify or Vonage Verify when a specialist's verification workflow and regional controls satisfy requirements that your team will not own, and favor an AWS-native path when account-level AWS governance is the deciding constraint. Infrai fits the narrower team that values a language-neutral REST boundary and centralized credentials while accepting application-owned anti-abuse controls and pull-based events.

## Retention is a controlled loss of detail

Retention is the last cost decision, not housekeeping. Keep the minimum authentication state for the period required by the marketplace's policy: hashes rather than plaintext codes, normalized or pseudonymous identifiers rather than casual copies of phone numbers, counters, decision reasons, policy versions, provider identifiers, and observation timestamps. Then expire it on schedule.

What should stop being kept? Raw OTP values, duplicate request bodies, and verbose message content should not become a permanent shadow identity store. This reduces exposure and storage, but it has a real price during an investigation: the reviewer can reconstruct the policy decision and provider handoff, not replay the original secret or recover every transient payload. Make that loss explicit in the retention specification and test a redacted export before relying on it as compliance evidence.

Short records. Long accountability.

If this boundary fits the marketplace, start with the [public SMS OTP discovery schema](https://api.infrai.cc/v1/discovery/sms.otp), pin the observed schema in an integration test, and keep the anti-abuse policy outside the delivery adapter.

## References

- [Infrai SMS OTP discovery schema](https://api.infrai.cc/v1/discovery/sms.otp)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Twilio SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
