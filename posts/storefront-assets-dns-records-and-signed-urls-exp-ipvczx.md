# Storefront Assets: DNS Records and Signed URLs Explained Through 3 Invariants

Use a versioned desired-state ledger for the storefront's asset hostname, and make publication depend on separate DNS, transport, and access checks. The deciding constraint is drift: an internal admin console can record what an operator wanted while the authoritative records, delivery endpoint, and signed URL policy describe different configurations. A successful write is insufficient evidence to label that hostname ready.

TL;DR: DNS records route a customer hostname; TLS authenticates the serving endpoint; signed URLs authorize requests. Reconcile those responsibilities independently, then publish an explicit readiness decision for the same configuration revision. For an e-commerce console, the useful question is whether the customer domain serves the intended assets under the intended access policy, with evidence that still belongs to the current revision.

## Decision: treat publication as a consistency problem

This note chooses a reconciliation ledger as the organizing mechanism for customer asset publication. Its editorial lens is change verification: the console must distinguish stored intent from observed behavior. Consider the illustrative hostname `assets.shop.example`, whose desired CNAME target is `edge.delivery.example`. Both names are examples, and a desired TTL of 300 seconds below is a configuration choice, not a measured propagation guarantee.

The console should retain the intended record, its configuration revision, the observed authoritative answer, and the results of HTTPS and authorization checks. Keep those observations attributable to the revision that triggered them. Otherwise, a delayed check can certify yesterday's target after an operator has already changed today's target, creating the same stale-read problem that makes a storage control plane difficult to trust.

Require fresh evidence.

The ledger is an application design, not a DNS transaction mechanism. RFC 1034 describes DNS caching and distributed resolution; writing a row in an application database does not synchronize those caches. Equally, a DNS answer is no evidence that an asset request passed authorization. The admin console needs vocabulary for those boundaries before it needs another green badge.

## How should DNS records and signed URLs serve assets on a customer domain?

Define 3 invariants. First, the authoritative DNS record matches the desired owner, type, target, and configured TTL. Second, an HTTPS check for the customer hostname reaches the intended serving configuration and validates its certificate. Third, the access check demonstrates the intended authorization behavior for that configuration, including rejection of an invalid or expired signed request. These are proposed acceptance conditions; they are deliberately stricter than merely receiving a successful record-write response.

For a CNAME-based design, compare DNS names case-insensitively and normalize a trailing root dot before comparing targets. RFC 1035 defines case-insensitive DNS name comparison. Do not apply that normalization to object paths or signatures. They belong to a different protocol boundary. Also inspect the existing record set: RFC 2181 restricts other data at a CNAME owner, with exceptions associated with DNS security. Adding an alias without considering the owner's existing data is a record-model error.

Configured TTL and remaining cached TTL are different observations.

Don't conflate them.

Use an authoritative record read to compare the configured TTL; a recursive resolver can return a countdown reflecting time spent in cache. A matching authoritative answer therefore supports the publication decision without proving that every customer resolver has discarded a previous answer. Label that distinction in the console. The failure mode is stale resolver state, and the practical consequence is that an old delivery target may still receive traffic during a change.

TLS has its own boundary. RFC 8446 describes the TLS handshake and certificate authentication; readiness should exercise the actual customer hostname, because a working delivery hostname alone does not establish that the customer-facing HTTPS endpoint is usable. Domain enrollment and tenant ownership verification belong before publication too. A record pointing at shared infrastructure does not, by itself, identify which tenant should receive the requests.

Keep mail outside this change's record scope. DMARC, specified in RFC 7489, concerns mail authentication policy; it does not authorize access to storefront images or downloads. An asset-domain workflow should not overwrite unrelated mail records while reconciling its own intended changes.

## Compare the publication strategies

The important comparison is how each strategy handles a disagreement between intent and observation.

| Strategy | Evidence available to the console | Failure boundary and trade-off |
| --- | --- | --- |
| Mark ready after a record write | The write operation reported success | Lowest orchestration effort, but no evidence about HTTPS, request authorization, or subsequent drift |
| Read back DNS before marking ready | Authoritative record state matches intent | Detects record disagreement; still leaves delivery and access behavior untested |
| Reconcile a revision with separate probes | DNS, HTTPS, and authorization observations associated with one revision | More state and probe work; supports a readiness decision whose evidence can be inspected and invalidated |

Choose the third strategy when the internal console owns the publishing decision. Its cost is operational complexity: someone must maintain probe identities, observation freshness, and transitions after a failed check. That is a team ownership decision as much as an implementation decision. There is no useful unit-price comparison until the team knows how many hostnames it will observe and how often it requires new evidence.

## Critical path: reject stale evidence before publishing

The following Python example implements only the publication gate. DNS reads, certificate checks, tenant enrollment, and authorization probes are adapter responsibilities. The example assumes a CNAME record, one expected target, and a configured TTL read from the authoritative record configuration. It does not resolve names or implement signed URLs.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Intent:
    revision: int
    owner: str
    target: str
    ttl: int


@dataclass(frozen=True)
class Evidence:
    revision: int
    owner: str
    record_type: str
    target: str
    configured_ttl: int
    https_ok: bool
    access_ok: bool


def dns_name(value: str) -> str:
    return value.rstrip(".").lower()


def publication_gate(wanted: Intent, seen: Evidence) -> tuple[str, ...]:
    if wanted.revision != seen.revision:
        return ("stale evidence",)
    failures = []
    if (dns_name(seen.owner), seen.record_type.upper(),
        dns_name(seen.target), seen.configured_ttl) != (
        dns_name(wanted.owner), "CNAME",
        dns_name(wanted.target), wanted.ttl
    ):
        failures.append("DNS drift")
    if not seen.https_ok:
        failures.append("HTTPS check failed")
    if not seen.access_ok:
        failures.append("authorization check failed")
    return tuple(failures)


wanted = Intent(12, "assets.shop.example", "edge.delivery.example", 300)
observed = Evidence(12, "assets.shop.example.", "CNAME",
                    "edge.delivery.example.", 300, True, True)
failures = publication_gate(wanted, observed)
ready = not failures
```

This function is intentionally small enough to review against the invariants. The harder production requirement is at the commit boundary: persist readiness only if the desired revision is still 12, using the database's conditional-update or transaction mechanism. If revision 13 arrives between the comparison and the write, discard the publication result and schedule checks for the new intent. Otherwise the code has a time-of-check/time-of-use race despite its revision comparison. In a test fixture, let revision 12 point at the original delivery target, pause its successful probe before the readiness write, and replace the desired target under revision 13. Then release the old probe. The expected result is no readiness update for revision 13 and a new check scheduled for that revision; a green status inherited from revision 12 would demonstrate that the persistence boundary is missing its condition.

Bind each probe to the tenant, hostname, and serving configuration as well as the revision; retain its observation time and a configured freshness limit. Those details are omitted from the example rather than disguised as implemented guarantees. Authorization evidence should cover a valid request, an expired request, a changed object path, and a request made through any alternate origin that could bypass the intended entry point. A Boolean named `access_ok` is useful only after those semantics are defined.

## Signed requests need a separate cache and origin contract

DNS cannot make a public origin private.

The architecture must specify where signed requests are checked, which resource and tenant the signature authorizes, how expiry is enforced, and how alternate routes to the object are controlled. RFC 2104 specifies HMAC, a possible cryptographic building block; it does not specify a complete signed URL format, key rotation policy, or request canonicalization scheme. Use the serving system's documented verification contract and test its boundaries.

Cache behavior deserves a specific acceptance test. Warm an object with an authorized request, then request the same object with missing, altered, and expired authorization. The protected response must still obey the chosen access policy on cache hits. RFC 9111 describes HTTP cache behavior, including restrictions around requests carrying an Authorization header, but a credential in a query string requires explicit application policy; those header rules alone do not establish the safety of a signed URL design.

A query credential also changes what request logging can expose. Configure redaction for signing parameters, record enough metadata to diagnose rejection reasons, and avoid retaining complete bearer URLs as routine probe evidence. For a private storefront export, the object might be durable and correctly addressed while access is wrong. Storage durability does not answer the authorization question.

Roll out the reconciliation worker with repeatable tests for stale revisions, record mismatches, and failed negative access checks. Bound retries and preserve the last observation instead of turning an exhausted retry into readiness. For observability, report the age of evidence and the reason intent differs from observation; counts of successful API writes cannot answer either question. If an external operator changes the authoritative record later, a subsequent reconciliation should revoke readiness and expose the new discrepancy.

## Rejected option: unconditional direct writes

An unconditional write from the admin console is a valid choice for a tool whose entire responsibility is submitting DNS changes, provided its status means submitted and the operator owns the remaining checks. It is rejected here because this console is responsible for deciding whether customer asset publication is ready, and a submitted record does not supply the necessary delivery or access evidence.

The resulting decision rule is narrow: publish only when current, attributable observations satisfy the DNS, transport, and authorization invariants, and keep checking for drift afterward.

A customer hostname is a routing identity. Readiness is a decision supported by evidence.

## References

- RFC 1034, Domain Names: Concepts and Facilities: https://datatracker.ietf.org/doc/html/rfc1034
- RFC 1035, Domain Names: Implementation and Specification: https://datatracker.ietf.org/doc/html/rfc1035
- RFC 2181, Clarifications to the DNS Specification: https://datatracker.ietf.org/doc/html/rfc2181
- RFC 8446, The Transport Layer Security Protocol Version 1.3: https://datatracker.ietf.org/doc/html/rfc8446
- RFC 2104, HMAC: Keyed-Hashing for Message Authentication: https://datatracker.ietf.org/doc/html/rfc2104
- RFC 9111, HTTP Caching: https://datatracker.ietf.org/doc/html/rfc9111
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
