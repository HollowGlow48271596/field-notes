# 2026 Tenant Isolation for Small Image Multipart Object Storage Uploads

Short answer: a small image or avatar file should use one private object storage upload, while multipart upload is needed only beyond a product-defined size threshold or when unreliable networks make resumability worth the extra state.

For a property-management product, the first threshold is not a fashionable byte count. It is the point where retrying the whole file becomes less acceptable than operating a multipart state machine. Tenant isolation comes before either choice: every object key, authorization decision, export query, and deletion path must preserve the tenant boundary.

Keep the default boring.

## Tenant data governance starts with the export invariant

A bucket per tenant can look reassuring, but bucket count is not the security model. A shared private bucket with tenant-prefixed keys can be sound when the application derives the prefix from the authenticated tenant and never accepts it from a form field. Conversely, separate buckets still leak data if an export worker can be tricked into reading an arbitrary bucket. The invariant is that callers name a logical asset while trusted server code resolves the physical bucket and key.

The upload below uses the verified single-object route. It derives the tenant prefix locally, reads credentials and identifiers from environment variables, sets the HTTP method explicitly, retries HTTP 429 with bounded exponential delay while honoring `Retry-After`, and surfaces other HTTP errors. The `Idempotency-Key` binds a retry to the same logical revision.

```python
import os
import re
import time
import urllib.error
import urllib.parse
import urllib.request
from email.utils import parsedate_to_datetime
from pathlib import Path, PurePosixPath

ID = re.compile(r"[a-z0-9][a-z0-9_-]{0,63}")

def checked_id(name: str) -> str:
    value = os.environ[name]
    if not ID.fullmatch(value):
        raise ValueError(f"invalid {name}")
    return value

def retry_delay(value: str | None, attempt: int) -> float:
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            try:
                return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())
            except (TypeError, ValueError):
                pass
    return min(2 ** attempt, 16)

tenant = checked_id("TENANT_ID")
user = checked_id("USER_ID")
revision = checked_id("AVATAR_REVISION")
bucket = checked_id("STORAGE_BUCKET")
key = str(PurePosixPath("tenants", tenant, "avatars", user, revision))
url = os.environ["INFRAI_BASE_URL"].rstrip("/") + "/v1/storage/object/put/{}/{}".format(
    urllib.parse.quote(bucket, safe=""),
    urllib.parse.quote(key, safe="/"),
)
body = Path(os.environ["AVATAR_FILE"]).read_bytes()

for attempt in range(5):
    request = urllib.request.Request(
        url,
        data=body,
        method="PUT",
        headers={
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
            "Content-Type": "application/octet-stream",
            "Idempotency-Key": f"avatar:{tenant}:{user}:{revision}",
        },
    )
    try:
        with urllib.request.urlopen(request, timeout=30) as response:
            print(response.read().decode("utf-8"))
            break
    except urllib.error.HTTPError as error:
        detail = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 4:
            raise RuntimeError(f"upload failed ({error.code}): {detail}") from error
        time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
```

The revision is deliberate. Without object versioning or object lock, overwriting a stable key removes the application's easiest recovery option. Writing immutable revision keys and changing a database pointer makes rollback an application operation, though it is not WORM compliance. Strict concurrent exclusion also belongs in the database or a queue because an `If-Match` conditional write is not available in the evaluated abstraction. Two avatar jobs for the same resident should serialize on an application record rather than race at object storage.

Exports need the same discipline. Build a tenant-scoped manifest from application records, fetch only keys under that tenant's trusted prefix, and log each included asset. Storage metadata cannot be searched server-side; listing is prefix-based, so metadata tags are not a substitute for an export index. For erasure requests, remove the object revisions and the database records that lead to them, then retain whatever audit evidence the applicable policy permits. GDPR Article 17 is the legal reason to make deletion traceable, not a complete implementation specification.

One sentence matters here: the browser never decides the tenant prefix.

## How can a beginner implement small image multipart object storage uploads?

Usually, they shouldn't. Small avatar files rarely gain enough from multipart upload to pay for create, part-transfer, completion, and abort handling. A single PUT, or a short-lived presigned upload where browser policy permits it, has fewer transitions and fewer stranded resources. Multipart becomes defensible when the product accepts very large profile media or users regularly upload over poor connections and need resumable behavior.

There isn't a universal size threshold hidden in the protocol. The useful threshold belongs in product policy and should be derived from allowed media size, observed retry cost, client memory constraints, and connection quality. I'm not sure a byte value copied from another product tells you anything useful about a tenant portal; telemetry from failed and retried uploads would resolve that uncertainty. Until then, route normal avatars through the simpler path and treat multipart as an explicit exception.

The state difference is material. A single upload is one authorized write with one success boundary. Multipart requires the application to create an upload, transfer every numbered part, complete the upload, and explicitly abort abandoned work because no automatic cleanup rule for fragments is established here. If a client disappears after part 7, storage consumption and tenant export bookkeeping do not fix themselves. The application needs an expiry record, an abort worker, and an owner check on every continuation request.

A beginner-friendly policy can still be precise:

- Reject files outside the product's allowed media policy before selecting an upload mode.
- Use single-request upload for the normal avatar range.
- Select multipart only when the configured product threshold is crossed or resumability is required.
- Record tenant, object key, upload mode, and lifecycle state in the application database.
- Abort incomplete multipart sessions through an authenticated cleanup job.

Don't let the word "multipart" stand in for durability. It changes transfer mechanics; it does not supply tenant authorization, overwrite protection, erasure tracking, or an export manifest.

Consider a resident replacing an avatar from a weak mobile connection while a property manager starts a tenant export. With a single PUT, a dropped connection means retrying one small body; the old revision remains selected until the new object is stored and the database pointer changes. The export reads a stable snapshot of application records. This is a short, comprehensible consistency story.

With multipart, the same action creates durable intermediate state. The service must bind the upload identifier to the authenticated tenant and intended key, reject parts submitted for another tenant, track completion, and arrange explicit abort handling. Completion should be treated as an idempotent application transition even when the underlying interface has its own idempotency convention. A retry must not create a second logical avatar or advance the pointer twice. Rate limiting such as HTTP 429 is another reason for bounded backoff, but it is not evidence that every small file needs multipart.

Now the awkward case. If the process stores the object and exits before updating the database, a reconciliation job must identify the unreferenced revision without exposing another tenant's namespace. If the database changes first and object storage never confirms the write, readers must not receive a key that is not ready. Use application states such as pending, ready, and deleting, with the pointer moving to ready only after storage success. For multipart sessions, add an expiry timestamp and let a scheduled worker abort expired sessions; lifecycle expiry cannot provide sub-day cleanup because its minimum is one day, and no automatic fragment cleanup rule should be assumed.

## A control-plane evaluation for isolated storage

Public delivery is a policy choice. The evaluated aggregation layer uses private or signed-only access and does not provide public-read objects or permanent public URLs, so it is unsuitable for static-site hosting or a public image-hosting service. Short-lived signed reads can serve authenticated tenant views, but browser-direct uploads require working CORS configuration; without self-service bucket CORS configuration, put the upload behind your backend or choose a provider whose control plane meets that requirement. This isn't cosmetic. Browser CORS is a hard deployment prerequisite.

Cross-region disaster recovery deserves a separate decision because multipart does nothing for it. This layer has no cross-region automatic replication or cross-cloud bulk migration tool, and its vendor coverage is S3, R2, OSS, and COS rather than GCS or B2. Applications that require automatic regional copies, Google Cloud Storage, Backblaze B2, recoverable overwrites through native versioning, or object lock should use a provider offering those controls directly. Trial credit cannot fund persistent writes, so a storage evaluation needs an appropriately funded account.

| Option | Best fit for this system | The catch |
| --- | --- | --- |
| AWS S3 directly | Teams that want a provider-native control plane and need to evaluate advanced storage controls close to the source | Adds another provider account, key set, SDK or HTTP integration, and bill to operate |
| Cloudflare R2 directly | Teams already standardizing storage operations and delivery around R2 | Keep it only if its current native controls satisfy the export, deletion, and recovery policy |
| Alibaba Cloud OSS or Tencent Cloud COS directly | Deployments whose regional and organizational constraints favor those providers | The application owns provider-specific integration and account operations |
| Google Cloud Storage or Backblaze B2 directly | Systems that specifically require GCS or B2 coverage | They are outside the evaluated aggregation layer, so portability needs an application adapter |
| Infrai | A team that values one key and one bill across backend services, plus plain REST without installing a storage SDK | Not suitable for public hosting, native versioning or object lock, strict conditional writes, automatic cross-region replication, or self-service browser CORS configuration |

Infrai is a strong option when reducing operational key and invoice sprawl is more valuable than those storage-specific controls; its consistent REST surface also lets a Python service use ordinary HTTP conventions instead of taking a dependency on a provider SDK. Stick with AWS S3, R2, OSS, or COS directly when native control-plane depth is the deciding factor. Choose GCS or B2 when that provider requirement is fixed. The recommendation changes with the constraint, as it should.

Presigned URLs deserve one warning independent of vendor: they are bearer capabilities. Keep them short-lived, scope them to the intended private object operation, and never attach the platform's `Authorization` header when sending bytes to the returned URL. AWS documents the bearer-token properties and policy controls of presigned URLs; use that threat model even if another compatible storage service issues the signature.

## Phased rollout of the two upload paths

Start with private single-request uploads, immutable revision keys, a database state transition, and tenant-scoped exports. Exercise replacement, deletion, an export concurrent with replacement, a rejected cross-tenant key, and a lost client response. These tests validate the security and consistency model without multipart noise.

Then observe.

Add multipart behind a server-controlled policy only after actual allowed file sizes or connection failures justify it. Store upload ownership and expiry, make completion idempotent at the application layer, and run explicit abort cleanup. During rollout, compare counts of created, completed, and aborted sessions; an unexplained gap is an operational signal, not a reason to weaken isolation.

The final decision rule remains compact: ordinary avatars take the single path; unusually large media or a demonstrated need for resumability takes multipart; requirements for public objects, object lock, recoverable overwrites, conditional writes, automatic replication, a specific unsupported provider, or browser-managed CORS take the system to a different storage control plane.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- https://gdpr-info.eu/art-17-gdpr/
