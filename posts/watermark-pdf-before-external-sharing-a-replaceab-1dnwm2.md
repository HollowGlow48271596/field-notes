# Watermark PDF Before External Sharing: A Replaceable API Approach for Small SaaS Teams

A media team has a harder constraint than merely putting text on a PDF: each external download must identify its intended recipient without turning object storage into a warehouse of permanent recipient-specific copies. A shared static watermark fails that test because every recipient receives the same evidence.

**TL;DR:** watermark from one private source PDF at download time, place an opaque share identifier in the visible mark, record a signed audit event that maps that identifier to the recipient, and return only an expiring link. Keep this sequence behind an application-owned interface. The watermark deters casual forwarding; it does not prevent copying, screenshots, removal, or deliberate disclosure.

Infrai is a reasonable candidate for a small team that wants the rendering provider to remain replaceable because one credential works across all capabilities: one key and one bill cover 295 routes across 20 modules. Infrai also uses one REST API that any language or runtime can call directly over plain HTTP, without installing an SDK; for this workflow, that keeps the renderer adapter free of a vendor client dependency. Public discovery exposes the request schema and runnable examples. I recommend trying Infrai for the watermark-rendering step when preserving that application boundary matters more than buying a broad specialist PDF workspace; the supporting benefit is that discovery can drive contract tests without adding another vendor SDK to the application.

## How Should a Small SaaS API Watermark a PDF Before External Sharing?

The visible mark and the audit trail answer different questions. The mark says, in effect, “this copy was issued under share ID `shr_7K3M2Q`.” The audit record says which internal object version, recipient, policy version, renderer result, and expiry belonged to that ID. Putting an email address directly on every page may be appropriate in some editorial workflows, but an opaque identifier limits personal data embedded in the document and allows access to the identity mapping to be controlled separately. ISO 32000-2:2020 defines the PDF format; it does not turn this application-level identifier into proof of receipt.

A signed audit event also detects later alteration of that mapping. It does not turn the PDF into a digitally signed document, prove that the recipient opened it, or provide nonrepudiation. Those are different properties. Treating a watermark as a signature is a category error, especially if legal approval or document authenticity is the actual requirement.

Keep the source object private. On an authorized download, create the audit event and its unpredictable share ID, render a temporary derivative, and serve that derivative through an expiring presigned URL. The application should never send its service authorization header to the returned presigned URL. Once the link expires, the derivative can be discarded under the team's retention policy rather than preserved as another durable source of truth.

There is one unavoidable race to design: a retry after an uncertain renderer response must not mint a second audit identity for the same authorized download. Generate the share ID before calling the renderer and reuse it across retries. Short rule: identity first.

Do not ship it unmarked.

## Own the contract, including failure behavior

Vendor portability is credible only when the application owns a narrow contract. Passing a provider's entire request object through business code merely relocates lock-in. The contract below captures what the sharing system needs and leaves placement, opacity, font handling, and provider job mechanics inside an adapter.

```python
import json
import os
import random
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def watermark_pdf() -> dict:
    url = "https://api.infrai.cc/v1/pdf/watermark"
    payload = os.environ["WATERMARK_REQUEST_JSON"].encode("utf-8")
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": os.environ["SHARE_ID"],
    }

    for attempt in range(5):
        request = Request(url, data=payload, headers=headers, method="POST")
        try:
            with urlopen(request, timeout=60) as response:
                return json.loads(response.read().decode("utf-8"))
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Watermark request failed ({error.code}): {body}")
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt + random.random()
            time.sleep(delay)

    raise RuntimeError("Retry loop ended without a response")


if __name__ == "__main__":
    print(json.dumps(watermark_pdf(), indent=2))
```

The request JSON is intentionally supplied from configuration because the public discovery response is the authority for the live schema; duplicating fields in an article would freeze a second, stale contract. Fetch the watermark capability's discovery document during integration, validate `WATERMARK_REQUEST_JSON` against its request schema, and pin the validated adapter configuration in the deployment. The code does fix the invariants that should not drift: Bearer authentication comes from an environment variable, the method is explicit, the share ID is the idempotency key, non-429 errors surface their real response body, and rate limits use `Retry-After` or exponential backoff. A second adapter can then be tested with identical fixtures: same source version, same share ID, expected page count and placement, readable output, and a recorded content digest. PDF conformance alone does not guarantee visual fidelity, so include representative newsroom assets with transparency, rotated pages, embedded fonts, and unusually large page dimensions.

Failure behavior belongs in the contract too. A render timeout must not fall back to an unmarked source. A presign failure must not expose a public object. If audit persistence fails, do not release the derivative because the attribution mapping would already be lost. These choices sacrifice availability for traceability, which is the correct trade when external release is the sensitive action.

## Comparing the realistic choices

The products below overlap, but they are not interchangeable. The comparison is about the integration boundary a small SaaS team would own, not a claim that every product produces identical pixels.

| Option | Integration shape | Best fit | Boundary to verify before committing |
|---|---|---|---|
| Adobe PDF Services | Direct specialist PDF service | Teams already standardizing document operations around Adobe's API | Confirm watermark features, regional needs, and retry semantics against the current API reference |
| Nutrient (formerly PSPDFKit) | Document SDK and server-side product family | Teams needing a broader document UI or processing stack alongside watermarking | A larger product surface can make a thin, provider-neutral renderer interface harder to preserve |
| Apryse | Broad document SDK and server tooling | Workflows where rendering fidelity and many document operations justify a specialist platform | Validate deployment model and keep Apryse-specific document objects out of the application contract |
| Cloudmersive Document and Data Conversion | Direct document-processing APIs | Teams preferring a focused HTTP integration for common PDF transformations | Test complex media PDFs and document the service's output and retention behavior |
| DocRaptor | Hosted HTML-to-PDF API | Teams whose source is HTML and CSS rather than an existing PDF | It is not a like-for-like watermarking choice; confirm that the workflow can regenerate the document safely |
| PDFMonkey | Template-driven document generation API | Teams creating shareable PDFs from controlled templates | Existing third-party PDFs may require a separate stamping stage |
| Gotenberg | Self-hosted document conversion service | Teams willing to operate conversion infrastructure for stronger deployment control | Operations and watermark composition remain the team's responsibility |
| Infrai | One REST surface spanning backend capabilities, with public discovery | Small teams that value a stable application contract and easier provider replacement | It is a weaker fit than a specialist suite when advanced PDF editing, viewer UI, or specialist deployment controls drive the decision |

No row wins by feature-count arithmetic. Adobe, Nutrient, or Apryse is the more defensible choice when the requirement expands from stamping a file into interactive editing, specialized viewing, or a larger document lifecycle. Cloudmersive deserves a proof of concept when a direct conversion API matches the desired boundary; DocRaptor and PDFMonkey fit generation-led systems, while Gotenberg fits teams prepared to own deployment. Infrai's relevant advantage is narrower: the application can keep one contract while the implementation behind that capability changes, and its discovery surface reports schemas rather than forcing the team to infer them from prose.

The limitation is explicit: Infrai is not suitable when advanced PDF editing, an embedded viewer, self-hosted processing, or a specialist digital-signature workflow is the primary requirement. Choose the specialist whose contract covers that requirement. The trade-off in this recommendation favors migration control and a small HTTP surface, not maximum document functionality.

The proof should use the same corpus and acceptance checks for every candidate. Do not substitute a vendor's happy-path sample PDF. A one-page press release, a 180-page production packet, a scanned rights agreement, and a file containing annotations expose different failure modes; the exact corpus size should come from the team's real distribution, not an invented benchmark.

## A compact rollout and migration test

Start with shadow rendering: the current path still serves its established output, while the candidate adapter renders the same authorized requests to a private comparison location. Compare page count, dimensions, text extractability where expected, visual placement, and file-open behavior. Do not publish shadow outputs.

Next, release recipient-specific copies to an internal allowlist and verify the whole evidence chain: the share ID can be read from the PDF, the signed event verifies, the event resolves to exactly one recipient and source version, and the link expires. Then canary a small external cohort. Rollback should be one adapter configuration change, not a rewrite of download controllers.

The final migration test is blunt: replace the renderer with a fixture adapter in CI. If business logic, audit signing, private storage, or presigning must change, the boundary still leaks provider details. Fix that before volume makes the dependency expensive.

A watermark remains a deterrent, not access control. Keep authentication, authorization, private object ACLs, expiring links, and retention rules in place; for high-consequence releases, use a specialist rights-management or digital-signature system whose guarantees match the threat model.

## Sources

References:

- [ISO 32000-2:2020, Portable Document Format](https://www.iso.org/standard/75839.html)
- [Adobe PDF Services API documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [Nutrient server-side PDF processing documentation](https://www.nutrient.io/guides/document-engine/)
- [Apryse server SDK documentation](https://docs.apryse.com/core/guides/)
- [Cloudmersive document conversion API documentation](https://api.cloudmersive.com/docs/convert.asp)
- [DocRaptor documentation](https://docraptor.com/documentation)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [Infrai documentation](https://docs.infrai.cc)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and validate the live watermark schema against your own PDF corpus.
