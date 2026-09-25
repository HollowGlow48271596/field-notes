# Daily Changelog Monitoring: Content-Addressed Diffs Before Semantic Reindexing

Short answer: fetch the changelog daily with conditional HTTP requests, normalize only known presentation noise, hash stable content blocks, and notify plus reindex only the blocks whose hashes changed. For healthtech product-content search, this boundary controls index cost: rebuilding embeddings for an unchanged page wastes capacity, while treating a rearranged page as new creates duplicate retrieval evidence.

This architecture decision chooses content-addressed block diffs, a durable snapshot, and an outbox notification. Its invariants are strict. One successful observation produces one immutable snapshot; a failed or ambiguous fetch never replaces the last good snapshot; notifications identify added, removed, and modified blocks; indexing consumes committed events rather than scraping independently. Product content also stays outside clinical decision support.

## How should Node.js scrape a changelog page daily and notify safely?

The first failure boundary is HTTP. A 304 response means the selected representation was unchanged for that conditional request. A timeout, 429, 5xx, TLS error, or unexpected content type is an observation failure, not an empty changelog. Keep the prior snapshot and alert on repeated failures.

Never convert uncertainty into deletion.

Extraction is the next boundary. Cookie banners, timestamps, navigation, and reordered markup can change raw HTML without changing a release note. Normalization must be narrow and fixture-tested. Stripping every date or number is dangerous in healthtech content because a release version, dosage-unit label, compatibility statement, or regulatory identifier may be exactly what a search user needs. Consider a page that moves an unchanged entry beneath a new month heading while also correcting one sentence in an older entry. A whole-page hash reports one opaque change; a position-based list reports every shifted entry; a heading-only key misses the correction if two releases reuse a heading such as “Maintenance update.” Stable entry links solve much of this when the publisher supplies them. Without those links, the extractor needs a documented composite key and collision tests. This is the unglamorous part of the design, but it decides whether one correction causes one embedding write or a rewrite of the entire history.

Identity needs equal care. Prefer a publisher-supplied entry ID or permalink. Otherwise derive a key from a normalized heading and stable ancestor, then store the full normalized block and a SHA-256 digest. The digest detects equality; it does not explain a modification, so retain old and new text long enough to produce a reviewable diff. Collision resistance cannot repair unstable extraction.

Finally, notification and indexing should not share a fragile inline transaction with fetching. Commit the snapshot and an outbox row together; separate workers can deliver notifications and update the index idempotently. Exactly-once delivery is not the useful promise. Stable event IDs plus idempotent consumers are.

| Option | Index-cost behavior | Main failure mode | Valid use |
|---|---|---|---|
| Re-embed whole page daily | Work grows with total content | Layout churn creates duplicate chunks | Tiny, stable pages |
| Diff raw HTML | Skips byte-identical pages | Templates look substantive | Contractually stable markup |
| Diff normalized blocks | Re-embeds changed blocks | Bad selectors hide or invent changes | Entry-based changelogs |
| Consume Atom or RSS | Small parsing surface | A truncated feed omits detail | Verified complete feeds |

Choose normalized block diffs, preferring a feed only after checking its completeness against the page. This adds schema ownership, fixtures, and migrations. Those costs buy a bounded reindexing unit: one changed entry rather than an accumulated history.

Retrieval-augmented generation combines generation with retrieved non-parametric memory. Stale and duplicated memory can therefore alter the evidence reaching generation, although the original RAG paper does not prescribe scraping or chunking policy. Preserve source URL, observation time, content hash, and lifecycle state so every result can be traced to the snapshot that supplied it.

## Critical path in code

The critical path is runtime-independent. The Python below keeps page-specific extraction behind a function; a Node.js implementation should preserve conditional validators, bounded reads, deterministic blocks, atomic persistence, and stable event IDs. The explicit limits are a 2,000,000-byte body, a 20-second request timeout, and SHA-256 content digests. Those are policy inputs, not universal defaults: operators should lower the byte cap for known-small sources and set the timeout within the scheduler's retry budget.

```python
import hashlib
import json
import os
import tempfile
import urllib.error
import urllib.request
from dataclasses import asdict, dataclass
from pathlib import Path

MAX_BYTES = 2_000_000


@dataclass(frozen=True)
class Block:
    key: str
    text: str
    digest: str


def block(key: str, text: str) -> Block:
    normalized = " ".join(text.split())
    digest = hashlib.sha256(normalized.encode("utf-8")).hexdigest()
    return Block(key, normalized, digest)


def fetch(url: str, etag: str = "") -> tuple[bytes | None, str]:
    headers = {"User-Agent": "ChangelogMonitor/1.0 (+ops@example.invalid)"}
    if etag:
        headers["If-None-Match"] = etag
    request = urllib.request.Request(url, headers=headers)
    try:
        with urllib.request.urlopen(request, timeout=20) as response:
            if response.headers.get_content_type() != "text/html":
                raise ValueError("unexpected content type")
            body = response.read(MAX_BYTES + 1)
            if len(body) > MAX_BYTES:
                raise ValueError("response exceeds configured limit")
            return body, response.headers.get("ETag", "")
    except urllib.error.HTTPError as error:
        if error.code == 304:
            return None, etag
        raise


def changes(old: list[Block], new: list[Block]) -> dict[str, list[str]]:
    before = {item.key: item for item in old}
    after = {item.key: item for item in new}
    return {
        "added": sorted(after.keys() - before.keys()),
        "removed": sorted(before.keys() - after.keys()),
        "modified": sorted(
            key for key in before.keys() & after.keys()
            if before[key].digest != after[key].digest
        ),
    }


def atomic_write(path: Path, payload: dict) -> None:
    descriptor, temporary = tempfile.mkstemp(dir=path.parent)
    try:
        with os.fdopen(descriptor, "w", encoding="utf-8") as handle:
            json.dump(payload, handle, sort_keys=True)
            handle.flush()
            os.fsync(handle.fileno())
        os.replace(temporary, path)
    except BaseException:
        try:
            os.unlink(temporary)
        except FileNotFoundError:
            pass
        raise


def commit(state_path: Path, old: list[Block], new: list[Block]) -> dict:
    diff = changes(old, new)
    event_id = hashlib.sha256(
        json.dumps(diff, sort_keys=True).encode("utf-8")
    ).hexdigest()
    atomic_write(state_path, {
        "blocks": [asdict(item) for item in new],
        "pending_event": {"id": event_id, "changes": diff},
    })
    return diff
```

The omitted extractor is the principal risk and must be implemented against saved fixtures for the actual source. The notifier consumes pending events after commit and marks the exact event ID delivered. This JSON-file example is suitable for one process on one host; concurrent workers require storage with transactional or compare-and-set semantics.

Use the platform scheduler and add jitter when many sources share an hour. Respect published crawling policy and terms, identify the client, cap bytes and redirects, and allowlist destinations. User-supplied URLs create a server-side request forgery boundary, so production code must reject private, loopback, link-local, and disallowed addresses after DNS resolution and again after redirects.

## How do we know a notification is trustworthy?

Test the pipeline as a state machine. Fixtures should cover a new entry, edited text, deletion, reordering, chrome-only changes, malformed HTML, an unexpected login page, and a response cut off at the byte limit. Replaying one fixture twice must create no second event. Replaying an event after a notifier crash must be harmless at the receiver or suppressed by its ID.

Record attempted time, successful observation time, response status, block count, changed-block count, bytes fetched, and delivery outcome. Alert when the last successful observation exceeds the agreed freshness window, when block count falls sharply, or when extraction returns zero after previously returning content.

Silence has several meanings.

For the semantic index, map each block key to its document ID. Embed added and modified blocks; tombstone or delete removed blocks according to the index consistency contract. Evaluation should include a superseded product limitation and verify that citations resolve to the exact embedded snapshot. Recall alone will not expose stale evidence.

## Rejected option, and where it still fits

The rejected design replaces the entire page daily. It has a valid use: a small internal page with a few kilobytes of stable text, one owner, and no entry-level notifications may have fewer operational failure modes without block identity.

It stops fitting as a changelog grows or users need to distinguish a corrected entry from a new one. Full replacement couples writes to corpus size and can expose mixed generations unless the index supports an atomic generation swap. Content-addressed blocks keep work proportional to substantive change, but only when extraction stability is maintained as a contract.

Do not reindex because the clock fired. Reindex because a validated source block changed, and preserve enough state to show why the system believed it changed.

## References

- https://www.rfc-editor.org/rfc/rfc9110
- https://www.rfc-editor.org/rfc/rfc9309
- https://docs.python.org/3/library/hashlib.html
- https://docs.python.org/3/library/os.html#os.replace
- https://owasp.org/www-community/attacks/Server_Side_Request_Forgery
- https://arxiv.org/abs/2005.11401
