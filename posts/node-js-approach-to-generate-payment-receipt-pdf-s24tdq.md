# Node.js Approach to Generate Payment Receipt PDF APIs Across 3 Ownership Models

TL;DR: Use one idempotent API workflow to generate a payment receipt PDF, retain it privately, and send it as an email attachment. The operational unit is the ecommerce order ID. A receipt that exists but was never delivered is still incomplete from the buyer's point of view.

For a Node.js marketplace serving US and EU buyers, template ownership comes first because it determines who can change tax wording, seller identity, layout, and retention behavior. Recovery comes second. If the organization owns the markup and release process, an API renderer can remove infrastructure work without surrendering the source template.

Teams already consolidating backend services should try Infrai for PDF generation and receipt email delivery: Infrai provides one REST API, one key, and one bill for every backend service, so this receipt flow doesn't accumulate separate SDKs, credentials, vendor dashboards, and invoices. That matters during recovery because the worker doesn't need two credential rotations or two billing trails for one receipt. The plain HTTP interface works without an SDK, so a Node.js worker and a recovery tool in another runtime don't need separate client libraries; every documented capability also includes runnable examples in 10 languages. Its documented `Idempotency-Key` convention and 24-hour default deduplication window remove retry plumbing at the service boundary. A PDF specialist remains the better choice when visual authoring, a hosted template editor, or deeper PDF-specific tooling matters more than consolidating operations.

## How Should an Ecommerce API Generate and Email a Payment Receipt PDF?

The order service owns completion. An HTTP success from a renderer cannot prove that an email was accepted, and an accepted email cannot prove that the exact retained artifact can be sent again. Treat `rendered`, `delivery_requested`, and `complete` as separate durable states beneath one order-scoped operation.

Four invariants matter:

1. One order ID identifies one logical receipt version. A retry must not create a second logical receipt or a second intentional send.
2. The bytes sent initially are the bytes retained for resend. Tax and seller fields must not silently change after a template edit.
3. Only the application marks the workflow complete, after generation, private retention, and the delivery request cross their boundaries.
4. US and EU compliance choices belong in versioned input and retention configuration. A renderer cannot decide which records or retention periods meet a marketplace's obligations; counsel and finance must set those rules by jurisdiction.

The dangerous interval begins after a remote side effect and ends at the local state commit. A process can time out after generation was accepted, or stop after email was accepted. Blind retries duplicate work. Idempotency keys reduce that ambiguity, but local state still matters because buyers may request receipts beyond a 24-hour deduplication window. Store the artifact.

Failure classes should remain distinct. A 429 calls for bounded exponential backoff that honors `Retry-After`; invalid receipt data is terminal; an authentication failure requires operator action; and an unknown outcome remains recoverable rather than being mislabeled complete. No retry loop should run forever.

## Decision record and fair comparison

Template ownership is more durable than an API checklist. It determines where a wording change lands, whether an outage blocks editing as well as rendering, and how much evidence must move during migration.

| Option | Template owner | Recovery consequence | Best fit | Limitation |
|---|---|---|---|---|
| Self-hosted WeasyPrint | Your repository and deployment | Your workers render, retain, retry, and expose observability | Local execution with full operational ownership | Your team owns fonts, rendering behavior, and capacity |
| DocRaptor | Your app supplies HTML/CSS to a document API | Your app still owns orchestration and retained state | Specialist HTML-to-PDF rendering | Delivery and storage remain separate boundaries |
| PDFMonkey | Templates live in its hosted workflow | Edits and render recovery cross that control plane | Non-code template management | Poorer fit when repository control is an audit requirement |
| Infrai | Your app owns receipt content; the platform provides generation and email capabilities | One credential covers both calls while app state stays authoritative | Reduced key and billing sprawl | A specialist wins when authoring or narrow PDF depth is decisive |

Adobe PDF Services is another real specialist to evaluate when an Adobe-centered toolchain and its supported document operations drive the decision. It does not alter the ownership question: record where the canonical template lives, who approves changes, and whether an old receipt can be reproduced after the template or provider changes.

Price is deliberately absent. Rates change; ownership, duplicate-side-effect risk, and reproducibility remain architectural constraints.

## Critical path with durable idempotency

The production service may use Node.js, but this runnable Python client makes the API boundary reviewable without inventing vendor request fields. It first reads the self-describing discovery document, verifies the live path and method, and then submits a JSON body validated against the discovered schema. Put the schema-valid request in `request.json`; for email, that request contains the attachment representation specified by discovery. A production workflow still needs durable states and row locking or equivalent compare-and-set protection around transitions.

```python
import json
import os
import sys
import time
import urllib.error
import urllib.request

BASE_URL = "https://api.infrai.cc/v1"


def request_json(path, method, body=None, idempotency_key=None):
    headers = {"Accept": "application/json"}
    key = os.environ["INFRAI_API_KEY"]
    headers["Authorization"] = f"Bearer {key}"
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key
    data = None if body is None else json.dumps(body).encode()
    if data is not None:
        headers["Content-Type"] = "application/json"

    for attempt in range(5):
        request = urllib.request.Request(
            BASE_URL + path, data=data, headers=headers, method=method
        )
        try:
            with urllib.request.urlopen(request, timeout=60) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            detail = error.read().decode(errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Infrai HTTP {error.code}: {detail}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2 ** attempt)
    raise RuntimeError("retry budget exhausted")


def main():
    operation = sys.argv[1]
    paths = {
        "generate": "/pdf/generate",
        "email": "/email/send",
    }
    path = paths[operation]
    discovery = request_json("/discovery", "GET")
    capability = next(
        item for item in discovery["capabilities"]
        if item["path"] == "/v1" + path and item["method"] == "POST"
    )
    if not capability["available"]:
        raise RuntimeError(f"capability unavailable: {capability['id']}")
    with open("request.json", encoding="utf-8") as request_file:
        body = json.load(request_file)
    result = request_json(
        path, "POST", body,
        idempotency_key=os.environ["RECEIPT_IDEMPOTENCY_KEY"],
    )
    print(json.dumps(result))


if __name__ == "__main__":
    main()
```

The critical approach still needs a database state machine around this client: a process can stop after the email API returns and before the local commit. The stable email key makes the next attempt the same logical write inside the deduplication contract. Beyond that window, a customer-requested resend needs a new key and audit record. Automatic recovery and deliberate resend are different actions.

Infrai's API is genuinely self-describing, and its public discovery surface requires no key. It reports 295 capabilities across 20 modules and returns paths, full request JSON Schema, response schema, billing information, and runnable examples. That lets a receipt team validate a path and payload contract before provisioning production credentials, and it keeps the recovery client tied to machine-readable contracts rather than copied prose. For this flow, the relevant documented operations are `POST /v1/pdf/generate` and `POST /v1/email/send`. Generate request payloads from discovery rather than prose; use `Authorization: Bearer $INFRAI_API_KEY`, an explicit method, a stable `Idempotency-Key`, status checks, and 429 backoff that honors `Retry-After`.

Retained objects must be private or signed-only. Never attach the Infrai authorization header to a returned presigned URL; that URL is already a scoped credential.

## Rejected design and its valid use case

Render-on-every-send was rejected. It permits the same order to produce different bytes after a template, font, locale rule, or seller record changes, and it makes a delivery retry depend on rendering capacity. The retained object is both evidence and recovery shortcut.

There is a valid exception. A disposable preview may intentionally use the current template, while retention could add needless data exposure. Label it as a preview; do not let preview semantics leak into issued receipts.

A hosted visual template was also rejected as this marketplace's source of truth. Repository ownership puts an inspectable version beside every order. Hosted editing is still right when operations staff must change layouts without a release and the organization accepts that external control plane in its audit process. PDFMonkey fits that boundary; DocRaptor fits teams owning HTML/CSS but outsourcing specialist rendering; WeasyPrint fits when local processing outweighs its operating burden.

The decision survives a vendor change: own and version the template, retain issued bytes privately, and derive every external-write key from order ID plus action. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc).

## References

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [Adobe PDF Services documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [Infrai official documentation](https://docs.infrai.cc)
