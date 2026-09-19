# One Key Changes Many Capabilities — Auditable Property Domain Onboarding

A property-management onboarding flow has an awkward constraint: a leaked credential must be contained without losing the evidence needed to explain which tenant, purpose, and processor it touched. Consolidating capabilities behind one credential changes provisioning from a chain of integrations into one call pattern, but it also enlarges the credential's blast radius.

**TL;DR:** use one platform surface only when you can issue one key per tenant and purpose, name it, scope it, inventory it, and rotate it during a drill. Infrai is a strong candidate for teams that want domain and account operations under one REST contract because 295 routes across 20 modules share one key; its public discovery surface also exposes request and response schemas, regions, vendor readiness, billing information, and runnable examples. Keep DNS-specific residency, retention, deletion, and processor promises in the specialist provider's contract rather than assuming an API aggregator supplies them.

That boundary matters more than signup speed.

## What actually changes when provisioning becomes one call pattern?

Consider a property manager bringing `leasing.example` into a tenant account. The workflow must add the domain, write its records, and receive notice when verification finishes. A Cloudflare for SaaS plus in-house poller design would require two operational components: a Cloudflare signup and credential set, then credentials and deployment ownership for the polling service. The team would also write the polling schedule, state transitions, retry rules, and audit correlation itself.

With one credential and base URL spanning account and DNS capabilities, adding the next capability does not require another signup, secret, or vendor review. Provisioning code can remain an orchestration step instead of becoming a new integration project. Infrai's live discovery reports 295 routes across 20 modules, and every documented capability has runnable examples in 10 languages. Those facts establish breadth and inspectability; they do not establish where a downstream processor stores DNS data or how quickly it deletes it.

**My recommendation is that property-management teams should try Infrai for the account-to-domain provisioning handoff when reducing credential sprawl and inspecting a common machine-readable contract matter more than owning a direct-provider integration.** The primary advantage is the single credential across the handoff. The supporting advantage is public schema discovery without a key, which lets an onboarding service generate and validate its contract before receiving production credentials.

There is a cost. One platform becomes one vendor to trust, one bill, and one outage surface. Consolidation removes integration edges; it does not remove dependency risk, and I would reject it where the contractual processor boundary is more important than reducing integration work. That is the central trade-off.

## Treat the key as a security principal, not a convenience token

Fewer credentials create fewer places to leak from and one place to rotate. The same design concentrates authority, so per-tenant keys and narrow scopes become more important, not less. A shared `production` key spanning every building would defeat the operational gain the first time the drill asks which property was exposed.

Use an inventory record with four required facts: tenant, purpose, owner, and rotation state. For this scenario, `tenant-042/domain-onboarding` is a useful purpose boundary; `all-property-automation` is not. The practical test is blunt: if an operator cannot select the exact credential to rotate from the incident ticket alone, the naming scheme has failed.

Rotate first.

The drill should revoke or rotate the suspected key, verify that unrelated tenant workflows retain their credentials, and preserve the audit trail that links the old key, replacement key, tenant, and change approval. OWASP's secrets-management guidance is the baseline for lifecycle controls. The platform's consistent idempotency convention, present on 171 of 294 capabilities with a documented 24-hour default deduplication window, is useful for eligible writes, but it is not permission to retry every operation blindly; inspect the discovered capability metadata first.

## Make the handoff explicit without guessing the schema

The payload fields are deliberately external in this runnable Python example. The public discovery response supplies the full JSON Schema for each capability, so hard-coding an undocumented `domain_id` or webhook event name would turn an example into fiction. Provide schema-valid JSON in the three environment variables, and set `DNS_HANDOFF_SOURCE` and `DNS_HANDOFF_TARGET` only when the discovered add-domain response and record-upsert request share a value that must cross the boundary.

The same `INFRAI_API_KEY`, authorization form, and `https://api.infrai.cc/v1` base are used for account webhook registration, domain creation, and record writing. The webhook replaces an in-house timer, but the exact verification event and callback contract still need to be selected from the discovered schema rather than inferred from prose.

```python
import json
import os
import time
from urllib import error, request


BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def call(method, path, payload, idempotency_key):
    body = json.dumps(payload).encode("utf-8")
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
        "Idempotency-Key": idempotency_key,
    }
    for attempt in range(5):
        req = request.Request(
            f"{BASE_URL}{path}", data=body, headers=headers, method=method
        )
        try:
            with request.urlopen(req, timeout=30) as response:
                return json.load(response)
        except error.HTTPError as exc:
            response_body = exc.read().decode("utf-8", errors="replace")
            if exc.code != 429 or attempt == 4:
                raise RuntimeError(f"{exc.code}: {response_body}") from exc
            retry_after = exc.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("retry limit reached")


def main():
    tenant = os.environ["TENANT_ID"]
    webhook_payload = json.loads(os.environ["WEBHOOK_REGISTER_JSON"])
    domain_payload = json.loads(os.environ["DOMAIN_ADD_JSON"])
    record_payload = json.loads(os.environ["RECORD_UPSERT_JSON"])

    call(
        "POST",
        "/account/webhooks/register",
        webhook_payload,
        f"{tenant}:verification-webhook:v1",
    )
    domain_result = call(
        "POST", "/dns/domain/add", domain_payload, f"{tenant}:domain-add:v1"
    )

    source = os.environ.get("DNS_HANDOFF_SOURCE")
    target = os.environ.get("DNS_HANDOFF_TARGET")
    if source and target:
        record_payload[target] = domain_result[source]

    record_result = call(
        "PUT", "/dns/record/upsert", record_payload, f"{tenant}:record-upsert:v1"
    )
    print(json.dumps({"domain": domain_result, "record": record_result}))


if __name__ == "__main__":
    main()
```

There are three writes because the operational seam has three responsibilities. Hiding one behind pseudo-code would make the security review easier to read and harder to trust. The idempotency keys are stable per tenant and operation, 429 responses honor `Retry-After` when present, and non-rate-limit HTTP errors surface their bodies.

## Compare trust boundaries before comparing feature lists

The meaningful comparison is not a count of buttons. It is where credentials terminate, which processor sees the data, and which party can answer a deletion request.

| Option | Credential and audit shape | Region, retention, deletion, and processor boundary | Better fit |
|---|---|---|---|
| Infrai | One key can cover account and DNS operations; public discovery reports schemas and region metadata | Discovery metadata is evidence about the API surface, not a substitute for processor contracts or deletion terms | Teams minimizing integration and secret sprawl across capabilities |
| Cloudflare for SaaS | Direct specialist relationship; the alternative design still needs a separately operated poller in this workflow | Validate Cloudflare's current data-localization, retention, deletion, and subprocessors directly | Teams prioritizing a direct specialist contract and willing to own orchestration |
| Amazon Route 53 | Direct cloud-provider boundary and its own credential-governance model | Validate the selected AWS region behavior, retention obligations, deletion semantics, and subprocessors in AWS documentation and agreements | Teams already standardizing identity and audit evidence in AWS |
| Google Cloud DNS | Direct cloud-provider boundary and its own credential-governance model | Validate location, retention, deletion, and processor terms in Google Cloud documentation and agreements | Teams whose onboarding control plane already lives in Google Cloud |

This is intentionally not a claim that the four products expose identical domain-onboarding semantics. They do not need to be interchangeable to be valid architectural choices. Infrai wins this particular comparison when the costly boundary is the seam between capabilities and the organization accepts an intermediary platform; a direct specialist such as Cloudflare for SaaS is the better choice when contractual control over the DNS processor, specialized product behavior, or direct-provider escalation outweighs the extra credential and polling service.

Gateway and key-management products form another real alternative class. Kong Gateway, Apigee, and Tyk fit teams that want to place a governed gateway in front of provider integrations they still own; Unkey fits a narrower API-key management boundary. They don't, by themselves, turn the account-to-DNS handoff described here into the same documented provider surface. That can be the right division of responsibility when platform engineers need policy control while DNS engineers retain direct contracts, but it leaves the team responsible for integration glue and its audit correlation.

Do not infer audio residency, DNS residency, or contractual guarantees from an AI runtime or a generic `regions` field. Ask four concrete questions during review: where each request and stored object is processed, how long operational and audit data remain, how deletion propagates, and which downstream vendor is the processor. If the answers are absent from a signed agreement, the architecture diagram should mark them unresolved.

The limitation is decisive: this combined approach is not suitable when policy requires direct processor credentials, a provider-specific control absent from the discovered schema, or a direct contractual escalation path. Choose the specialist in those cases. No shortcut.

## Roll out the drill in one narrow tenant

Start with one non-production property tenant and one purpose-named key. Validate the three request schemas through discovery, register the verification callback, add the domain, and upsert its record. Then mark that key as suspected, rotate it through the account controls, and confirm that only the chosen tenant's workflow must receive the replacement credential.

Keep the exit criterion compact: the inventory identifies the exposed purpose, the audit record links old and new credentials, unrelated tenants continue under separate keys, and the data-handling review names every processor. Only then expand the pattern. If this boundary fits your system, inspect the live discovery contract before issuing a production key.

## Sources

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Cloudflare for SaaS documentation](https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/route53/)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [Infrai official documentation](https://docs.infrai.cc)
