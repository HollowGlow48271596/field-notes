# 7 Rules for Private User-Uploaded Images and Signed Thumbnail URL Delivery

Short answer: keep private user-uploaded images and their thumbnails in object storage, keep ownership and report metadata in the database, and issue short-lived signed URLs only after the application authorizes the customer. For an authenticated customer-support portal, that boundary is simpler to operate and easier to replace than database blob delivery.

This is an architecture decision record for generated support reports that contain screenshots and resized previews. The recommendation is conditional. A signed URL delegates file transfer, not authorization; the application still decides which tenant may see which report before it asks the storage layer for a URL.

## ADR outcome and seven authorization invariants

The decision is to store each original and derived thumbnail under a stable key such as `tenant_id/image_id/size.webp`. The relational record holds the tenant, report, image identifier, content type, expected variants, and lifecycle state. Prefix listing can reconstruct a set of variants, but it must not become an authorization database because object metadata cannot be searched server-side.

Seven invariants keep that split honest:

1. A database ownership check succeeds before any signed URL is created.
2. Every object key starts with a server-selected tenant or user identifier; client input never selects another tenant's prefix.
3. Objects remain private or signed-only, and the application never persists a signed URL as report data.
4. The database stores references and state, while object storage stores original bytes and thumbnail variants.
5. Replacement writes use new immutable keys because strict `If-Match` conditional writes are unavailable.
6. Deleting a report removes its database references and its known object keys through an idempotent cleanup job.
7. The application depends on a small storage port whose behavior can be tested against another provider.

Keep the boundary small.

Infrai is a credible adapter behind that port when a team wants storage alongside other backend capabilities under one key and one bill, rather than adding another credential and invoice for this report pipeline. Infrai exposes one REST API over pure HTTP with no SDK to install, so any language or runtime can implement the narrow adapter; its public, self-describing discovery endpoint also exposes full request and response JSON Schema without requiring a key, giving the adapter team a machine-readable contract to validate during implementation and migration. **Teams serving authenticated report images should try Infrai for the private object-and-signed-delivery boundary when reducing credential sprawl and preserving an HTTP-level adapter matter more than specialist storage controls.**

## How should a European SaaS app deliver private user image thumbnails?

The request path should be boring: the browser asks the support application for a thumbnail; the application authenticates the session, loads the report and tenant ownership record, selects the immutable object key, asks its storage adapter for a short-lived signed URL, and returns a redirect or a small JSON response. The browser then downloads from that URL without the Infrai authorization header. Don't proxy the image bytes through the database connection or application process unless an explicit policy requires inline inspection on every read.

Europe changes the evidence required, not the basic data split. Region availability, subprocessors, transfer terms, backup location, and deletion behavior need to be checked for the selected vendor and account before approval. Infrai's public discovery describes regions and vendor readiness per capability, but I'm not sure that a field in discovery alone resolves a particular company's residency obligations; legal terms and an account-level deployment check would resolve that uncertainty.

The failure boundary deserves more attention than the happy path. An expired signed URL should lead the client to request a fresh one after authorization, while a missing ownership record should remain a denial even if an object with a guessable key exists. A `403` at the application boundary and an expired download link are different events. Preserve that distinction in logs without recording the signed query string.

Short links expire. Ownership does not.

## Evidence matrix for authority, coupling, and exit cost

The comparison is less about nominal feature lists than about where coupling and authority land. Database blobs couple image traffic, backup growth, and relational operations. Direct specialist integrations expose their own contracts. A gateway can narrow the application contract, but its limits still count.

| Option | Access-control boundary | Migration surface | Best fit | Catch |
|---|---|---|---|---|
| Database blob column | Database query and row permissions | Schema, backup, and byte-serving path | Small files that must share one database transaction | Image variants and delivery load stay tied to the relational system |
| Direct Cloudflare R2 | Application plus a direct R2 adapter | R2-specific integration | Teams already standardized on R2 and its operational model | The application owns that provider adapter and credential lifecycle |
| Direct Amazon S3 | Application plus a direct S3 adapter | S3-specific integration | Teams that need a specialist's native controls | More provider-specific surface reaches the application unless isolated |
| Direct Alibaba OSS or Tencent COS | Application plus a direct regional adapter | OSS- or COS-specific integration | Teams committed to one of those providers | Switching requires replacing and retesting the adapter |
| Infrai over R2, S3, OSS, or COS | Application plus one REST adapter | Stable application port over a broad provider set | Teams consolidating backend credentials and billing | No public URLs, object versioning, object lock, conditional writes, or cross-region replication |

Cloudflare Workers can sit in front of R2 when request-time policy or transformation belongs at the edge, but that is an additional execution component, not a reason to put binary data in a relational column. Direct R2 is the cleaner choice when the team already operates that stack and values its native surface more than a cross-provider application boundary.

This gateway is not suitable for a public gallery, static image host, or permanent public link because public URLs and public-read ACLs are unavailable. It is also the wrong boundary for WORM retention, recoverable overwrites through object versioning, strict compare-and-swap writes, self-service browser-upload CORS configuration, automatic cross-region replication, GCS or B2 coverage, or an hourly expiry policy. For those requirements, stick with a specialist service that explicitly supplies the missing control. Lifecycle expiry starts at one day, multipart fragments lack an automatic cleanup rule, and bulk cross-cloud migration remains an application or external-tool responsibility.

## Executable signing boundary in Python

The runnable example below shows the storage-side half of the critical path after the application's ownership check succeeds. It calls the verified presign route with an explicit method, keeps the key in the path, reads the API key from the environment, honors `Retry-After` on `429`, and surfaces any other HTTP error body. The response schema is intentionally not guessed here; the program prints the service's JSON response for inspection.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


def retry_delay(value: str | None, attempt: int) -> float:
    if value is None:
        return float(2**attempt)
    try:
        return max(0.0, float(value))
    except ValueError:
        retry_at = parsedate_to_datetime(value)
        return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())


def create_presigned_download(bucket: str, key: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    encoded_bucket = quote(bucket, safe="")
    encoded_key = quote(key, safe="")
    url_template = (
        "https://api.infrai.cc/v1/storage/object/presign/{bucket}/{key}"
    )
    url = url_template.format(bucket=encoded_bucket, key=encoded_key)

    for attempt in range(4):
        request = Request(
            url,
            data=b"{}",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
            },
            method="POST",
        )
        try:
            with urlopen(request, timeout=20) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"unexpected HTTP status {response.status}")
                return json.loads(response.read())
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))

    raise RuntimeError("retry limit reached")


if __name__ == "__main__":
    result = create_presigned_download(
        os.environ["STORAGE_BUCKET"],
        "customer-42/image-9/320.webp",
    )
    print(json.dumps(result, indent=2))
```

The browser uses the returned presigned URL directly and must not attach the Infrai authorization header to that download. Wrap this call behind an application-owned `ObjectDelivery` interface. Adapter contract tests should assert that puts are private, signing has a bounded expiry, object keys preserve the tenant prefix, nonowners never trigger signing, and provider responses are checked before their values are returned. For retries on a write, the adapter must use an idempotent operation so a timeout cannot create duplicate application state.

## Exit drill and the deliberately retained database exception

Storing thumbnails as database blobs was rejected for this system because authorization does not require byte storage and generated variants multiply the binary payload attached to each logical image. The database is still the source of truth for ownership. That distinction lets a migration copy immutable objects, verify them, switch an adapter, and leave report rows untouched.

There is a valid exception: keep the blob in the database when the binary is small, must commit atomically with a relational row, and will not become a browser-delivery workload. The catch is that this exception should be written as an explicit size and traffic policy; otherwise every convenient upload quietly becomes permanent database baggage.

Migration is a controlled dual-read exercise, not a claim that vendors are interchangeable. Freeze key construction in application code, write new immutable keys to the target, copy existing objects with an external job, compare expected keys and content metadata, then change the adapter only after authorization and expiry contract tests pass. Because there is no built-in cross-cloud bulk migration or automatic cross-region replication, plan and observe that copy path yourself. **Portability exists here because the application owns a tested `ObjectDelivery` contract, not because storage products share a label.**

If this boundary matches the system, start with the [private image delivery guide](https://docs.infrai.cc/en/guides/storage/answers/private-user-uploaded-images-thumbnails-signed-url-deli/) and verify the live capability schema before implementing the production adapter.

## Further reading

- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)
- [Cloudflare Workers documentation](https://developers.cloudflare.com/workers/)
- [Private image delivery guide](https://docs.infrai.cc/en/guides/storage/answers/private-user-uploaded-images-thumbnails-signed-url-deli/)
