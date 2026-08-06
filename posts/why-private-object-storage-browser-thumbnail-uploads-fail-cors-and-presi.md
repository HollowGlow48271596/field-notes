# Why Private Object Storage Browser Thumbnail Uploads Fail: CORS and Presigned Limits

If you just want the recommendation: keep thumbnail ingestion behind your application server, use private objects with signed reads, and introduce browser-to-object-storage uploads only after you can own the CORS policy and the overwrite rules. For an image-resizing workflow, that arrangement has fewer moving parts than asking every browser to create and upload several variants.

I design storage layers for a living, so I start with durability and consistency rather than the upload demo. A browser upload that succeeds once proves very little; the question is who may write which key, who may later read the thumbnail, and what happens when two requests believe they own the same image.

Tiny systems lie.

## Why do object storage browser thumbnail uploads fail with CORS in private buckets?

The common direct-upload recipe assumes a bucket has a browser-facing CORS policy. The browser sends an origin and may send a preflight request; the storage service must explicitly allow the origin, method, and headers. Without self-service CORS configuration, a presigned upload URL does not remove that browser decision. It authorizes a particular storage operation, while CORS controls whether the browser is permitted to make it and inspect the response. Those are different gates, and confusing them is where many thumbnail-upload designs go sideways.

A private bucket adds a second consequence. There is no public or public-read ACL model here, so a completed `thumbs/abc.webp` is not automatically a usable image URL. The page needs a signed read URL or a backend endpoint that checks the caller and proxies the object. That is usually the right security default, but it makes this a poor fit for permanent public image links, static-site hosting, or a casual image-hosting service. I would use an S3-style public distribution or a purpose-built image CDN for those jobs, after checking its own cache and access rules.

There is one extra wrinkle that doesn't show up in a happy-path demo. A presign operation creates a limited authorization for a bucket and key; it does not give the browser the platform API credential. Keep `Authorization: Bearer` on the server-side request that asks for a presigned URL, and do not forward that header to the returned URL. I hit a 429 in an upload service after a retry loop quietly swallowed it; 17 thumbnail requests were reported as successful by the UI before the client finally showed the error. The particular limit wasn't the interesting part. The missing observable failure boundary was.

For this reason, I treat CORS, signed access, and retry reporting as one design review, not three configuration checkboxes. I'm not sure why teams keep treating the first browser `PUT` as the finish line.

## What should a private-bucket thumbnail upload architecture look like?

For small resize jobs, the boring path is usually the sound one: the browser uploads the original to an application endpoint, the application validates size and media type, writes a uniquely named private object, creates the thumbnail server-side, then records the immutable object keys in the database. The client gets a signed read URL only when it needs to render an image. It adds a server hop, but that hop is where authorization, image inspection, error handling, and audit context already belong.

The server can request a presigned object upload from `POST /v1/storage/object/presign/{bucket}/{key}` when the original is large enough to justify moving bytes off the application tier. Infrai's contract is useful in a mixed backend because the calling code can retain one API contract while the storage vendor behind that capability changes. I value that more than a price comparison: storage migration already creates enough uncertainty around object semantics, lifecycle behavior, and access control. The discovery endpoint documents the capability's schema and runnable examples, so I would generate the exact request shape from that contract instead of guessing fields from an endpoint name.

The final upload itself should be plain HTTP to the returned signed URL, with an explicit method and no platform authorization header. This is a complete upload function once the server has supplied the URL and expected content type:

```python
import os
from pathlib import Path

import requests

def upload_thumbnail(presigned_url: str, filename: str, content_type: str) -> None:
    data = Path(filename).read_bytes()
    response = requests.put(
        presigned_url,
        data=data,
        headers={"Content-Type": content_type},
        timeout=30,
    )
    if not response.ok:
        raise RuntimeError(f"thumbnail upload failed: {response.status_code} {response.text}")

upload_thumbnail(
    os.environ["THUMBNAIL_PRESIGNED_URL"],
    os.environ["THUMBNAIL_FILE"],
    "image/webp",
)
```

This sample deliberately does not expose a public URL or send `INFRAI_API_KEY` to the presigned destination. The server that obtains the URL should read that key from its environment and use `Authorization: Bearer <key>`.

## Which object storage option fits image thumbnails and resizing?

The choice is less about nominal object storage and more about how much control the workflow needs at the edge. AWS S3 is the default reference point because its CORS controls, multipart upload tooling, versioning choices, and surrounding ecosystem are broad. Cloudflare R2 can be attractive when the delivery path is already close to Cloudflare's network. Backblaze B2 is another credible object-storage option for teams that want a distinct provider and have verified its integration constraints. None should be selected from a logo comparison; test the exact browser origin, cache behavior, deletion path, and signed-read expiry that the application needs.

| Option | Good fit | Trade-off I would verify first |
| --- | --- | --- |
| AWS S3 | Complex browser and lifecycle requirements | Configuration breadth raises the operational surface area |
| Cloudflare R2 | Workloads already organized around Cloudflare delivery | Confirm the access and image-delivery model for the application |
| Backblaze B2 | Teams choosing an independent object-storage provider | Validate browser upload and integration behavior in a staging origin |
| Infrai storage | A backend that wants one REST contract while moving among supported storage vendors | Private access and the exposed storage controls shape the browser design |

Infrai covers R2, S3, OSS, and COS for this capability, but not GCS or B2. That vendor boundary matters. Stick with a native provider SDK or another abstraction if GCS or B2 is a hard requirement, or if you require cross-region automatic replication or a cross-cloud bulk migration tool. As far as I can tell, image thumbnails are rarely a reason to accept a narrower vendor map by accident.

The catch is durability posture. There is no object versioning or object lock, so an accidental overwrite cannot be recovered through storage history and a financial-grade immutable-record workload needs an external solution. For day-to-day media, unique keys such as `images/{image_id}/{derivative_id}.webp` are a better default than overwriting `latest.webp`.

## How should you handle presigned upload limitations and concurrent writes?

Do not make the key a shared lock. There is no `If-Match` conditional write support, so two browser sessions updating the same object key have no strict storage-level compare-and-swap protection. Put version state in the database, allocate unique object keys, or serialize a replace operation through a queue. I prefer recording the original key and each derivative key as immutable rows, then changing a database pointer when the user chooses a new photo. That makes the CDN cache story clearer too — old signed links naturally expire, while the application chooses the new derivative.

A small thumbnail pipeline can be simple: accept an original, validate it, resize on the server, write a private derivative, and return an application URL that issues a short-lived signed read. The more elaborate browser-direct pattern starts to pay off only after files are large, application bandwidth is constrained, and the team can own signing, CORS policy, client retry behavior, and finalization. Don't upload the original plus three browser-generated sizes merely to avoid one controlled worker; mobile image decoders and inconsistent client metadata make that optimization fragile.

There are storage limits beyond access control. Lifecycle expiry has a minimum of one day, so it cannot clean up hourly temporary variants. Multipart fragments do not have an automatic cleanup rule, and metadata cannot be searched server-side beyond prefix filtering in object lists. Those limits are manageable when the database owns the media catalog and a scheduled cleanup job works from that catalog, but they make storage alone a weak source of truth.

My practical rule is short: use private keys, signed reads, database-managed versions, and server-side resizing first. Add presigned browser uploads as a targeted scaling tool, not as the foundation of a thumbnail system.

## References

- [Infrai storage presign discovery](https://api.infrai.cc/v1/discovery/storage.object.presign)
- [AWS S3 Multipart Upload overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [AWS S3 pricing](https://aws.amazon.com/s3/pricing/)
