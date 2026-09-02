# Choosing Auth App Avatar Delivery: Private Access or Cacheable CDN Images?

**Short answer:** For most authenticated applications, keep the original avatar private, store its object key rather than a delivery URL in the user profile, and choose delivery per audience: issue a short-lived signed URL when viewing permission is sensitive, or publish a deliberately derived avatar through a CDN when broad visibility and cache reuse matter more.

This is an architecture decision, not a storage toggle. A profile image can be low sensitivity and still reveal an account's existence, survive an account-visibility change in a cache, or become an authorization side channel. Conversely, forcing every tiny avatar through per-view authorization can add machinery without protecting anything the product actually treats as private. The deciding constraint is who may possess the bytes after the URL has been copied, not whether the screen displaying them requires login.

## Decision and invariants

The default design has two layers. The private source object is the durable record; its opaque key is attached to the profile. A delivery layer then produces either an expiring signed URL or a public, cacheable rendition according to an explicit visibility policy. Don't persist a signed URL as profile data. It is a temporary bearer capability, and AWS documents that presigned URLs can be used until they expire and can be reused during that interval.

Keep these invariants regardless of provider:

1. The database stores a stable object key and a version, never a temporary signature.
2. Upload authorization and download authorization are separate decisions.
3. The object key is unguessable, but unguessability is not treated as access control.
4. A visibility change creates a new delivery decision; it does not depend on the old URL disappearing from somebody's clipboard.
5. Avatar replacement changes the versioned key or rendition name, so stale caches cannot silently become the current profile image.
6. Logs record policy outcomes and object versions, while credentials and signed query parameters are redacted.

That's the boundary.

The table captures the choice without pretending either path wins universally:

| Decision axis | Short-lived signed object URL | Public CDN rendition |
|---|---|---|
| Reader eligibility | Evaluated before each URL is issued | Anyone holding or discovering the URL can read it |
| Cache reuse | Depends on whether the delivery design normalizes or forwards signature parameters | High potential because viewers share one stable rendition URL |
| Revocation | Stop issuing new URLs; already issued URLs remain usable until their expiration | Purge or replace the public rendition, then account for downstream copies |
| Profile update | Issue against the new versioned object key | Publish a new versioned rendition URL |
| Operational load | Signature issuance, expiry handling, and authorization telemetry | Publication workflow, cache invalidation, and visibility-transition handling |
| Best fit | Private workspaces, moderated identities, regulated or sensitive membership | Public communities, author pages, and avatars already defined as public content |

## How should an auth app deliver private user profile images?

Start with a permission statement that a reviewer can falsify: “A viewer may read avatar version V only while viewer X may view user Y's profile.” If the real rule sounds like that, a signed URL is the cleaner representation because issuance follows an authorization check and the capability expires. AWS calls presigned URLs bearer tokens and notes that their effective lifetime is constrained by the underlying credential lifetime; a URL created with temporary credentials stops working when those credentials expire, even if the requested URL lifetime was longer.

Expiry is not revocation. That distinction catches designs that look secure on a diagram but have no bounded answer to “What happens after this member is removed?” If policy allows a five-minute exposure window, a five-minute application TTL may be reasonable; five minutes here is an application choice, not a universal recommendation. Tightening it to a few seconds can cause retries and slow page loads to cross the boundary, while stretching it to days weakens the point of checking authorization at issuance. I'm not sure there is one correct TTL without the product's threat model, observed page-load distribution, and incident response target. Your mileage may vary.

For a public profile, signing every request often models the wrong rule. If the avatar is intended to be visible to anonymous visitors and search crawlers, publish a separate rendition with a stable, versioned name and cache it at the edge. Keep the original private anyway: originals may contain dimensions, metadata, or quality that the public UI doesn't need, and a derived object gives the publication workflow a clear boundary. This does add a transformation and lifecycle step — there is no free architecture — but it avoids confusing “the account requires authentication” with “every byte on the account page is confidential.”

A mixed application should not collapse these cases into one global bucket policy. Use an explicit enum such as `private`, `members`, or `public`, and map each state to delivery behavior. A move from public to private must remove the published rendition as an operational action, yet the system should still assume that an earlier recipient may have retained a copy. No URL mechanism can claw bytes back from a client that already downloaded them.

## Failure boundaries worth naming

The first failure mode is storing a signed URL in the user row. It works during a happy-path demo, then expires and turns durable profile state into a clock-dependent value. A client may repeatedly render the dead value and receive an authorization-style failure such as `403`, even though the object still exists and the user is still allowed to view it. The fix in the architecture is straightforward: retain the object key, authorize the viewer, and mint delivery access at read time.

The second is treating a random public path as private. A long identifier reduces accidental guessing, but URLs leak through browser history, screenshots, application logs, analytics payloads, copied HTML, and human sharing. Once read permission is public, the path itself is the capability and has no meaningful expiry. Call it public in the threat model.

Then there is cache identity. Two signed URLs for the same object can contain different query parameters. If an edge cache includes those parameters in its key, viewer-to-viewer reuse falls; if it ignores security parameters without a carefully designed trusted authorization layer, it may serve bytes across a permission boundary. Don't “fix” the hit rate by casually dropping signature fields. Decide which component authenticates, which component keys the cache, and which component is allowed to fetch the private origin, then test that exact composition.

Updates fail differently. Overwriting `avatar.jpg` asks every cache and client to agree about freshness at once. A versioned key such as an opaque object identifier plus a profile version lets the database switch atomically to a new identity while old cache entries age out. Deletion is more demanding: remove the source according to retention policy, stop signing it, remove any public rendition, and ensure backups or audit retention follow the application's stated policy rather than an improvised request handler.

Finally, cost can reverse a technically correct choice at scale. Model storage bytes, transformation work, origin reads, signing requests, CDN requests, and data transfer separately. AWS's S3 pricing page, for example, separates storage, requests and data retrieval, data transfer, management and analytics, replication, and transformation categories; the applicable entries depend on region and storage class. Use your own traffic distribution, especially cache misses and replacement frequency. A single average avatar size is not a cost model.

## Critical path in Python

The useful abstraction returns a delivery result, not “the avatar URL” as if all URLs had the same authority. This Python sketch keeps product policy visible and leaves provider-specific signing and CDN publication behind narrow interfaces:

```python
from dataclasses import dataclass
from enum import Enum
from typing import Protocol


class Visibility(Enum):
    PRIVATE = "private"
    MEMBERS = "members"
    PUBLIC = "public"


@dataclass(frozen=True)
class Avatar:
    owner_id: str
    object_key: str
    version: int
    visibility: Visibility


@dataclass(frozen=True)
class Delivery:
    url: str
    expires_at_epoch: int | None
    cache_scope: str


class Authorizer(Protocol):
    def can_view(self, viewer_id: str | None, avatar: Avatar) -> bool: ...


class PrivateStore(Protocol):
    def sign_read(self, object_key: str, ttl_seconds: int) -> Delivery: ...


class PublicRenditions(Protocol):
    def url_for(self, object_key: str, version: int) -> str: ...


def deliver_avatar(
    viewer_id: str | None,
    avatar: Avatar,
    authorizer: Authorizer,
    private_store: PrivateStore,
    public_renditions: PublicRenditions,
) -> Delivery:
    if avatar.visibility is Visibility.PUBLIC:
        return Delivery(
            url=public_renditions.url_for(avatar.object_key, avatar.version),
            expires_at_epoch=None,
            cache_scope="public",
        )

    if not authorizer.can_view(viewer_id, avatar):
        raise PermissionError("viewer cannot read this avatar version")

    return private_store.sign_read(
        object_key=avatar.object_key,
        ttl_seconds=300,
    )
```

The `300`-second TTL is a declared local policy. Production code should derive it from configuration, cap it within the signer and credential limits, and return an absolute expiry so the client can refresh before the boundary. More important, tests should freeze time and verify four transitions: allowed to denied, denied to allowed, private to public, and public to private. Add a concurrency test for an avatar update occurring between authorization and signing; the issued URL must refer to the authorized version, not whichever object happens to occupy a mutable name later.

Observe decisions rather than secrets. Useful counters include authorization denials, signing latency, rendition publication latency, CDN hit ratio by visibility class, refreshes before expiry, and reads requested for superseded versions. Never place a complete signed URL in structured logs — query parameters are credentials for as long as the capability remains valid.

## Rejected option and its valid use case

The rejected default is a single public bucket or public CDN path for every account avatar. It is not suitable when membership itself is private, when a blocked user must stop receiving newly authorized access, or when profile visibility can narrow after publication. Stick with signed delivery in those cases, accepting the extra authorization and refresh path because it represents the actual rule.

Public delivery remains the better option when the product explicitly defines avatars as public, anonymous traffic is expected, and the team can operate publication, versioning, and removal. The catch is that changing a profile to private cannot erase copies already downloaded while it was public. State that limitation in product behavior rather than hiding it behind an “invalidate cache” button.

There is also a valid middle design: authenticate at a trusted edge or image proxy and keep the origin private. It can combine stable client-facing paths with centralized policy, but it moves correctness into cache-key construction, edge authorization, and origin isolation. Choose it only when the team can test those boundaries and operate that component; otherwise, the apparently tidy URL masks a larger security surface.

The final record is conditional: private or membership-scoped avatars use short-lived signed access; intentionally public avatars use versioned CDN renditions; originals remain private; the profile stores identity and version, not a delivery credential. Revisit the decision when visibility semantics, cache architecture, credential lifetime, or traffic shape changes.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- https://aws.amazon.com/s3/pricing/
