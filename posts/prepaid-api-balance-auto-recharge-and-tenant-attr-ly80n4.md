# Prepaid API Balance Auto Recharge and Tenant Attribution (One Shared Ceiling)

The bill starts with chargeable API usage, plus whatever operational and evidence-storage costs the service incurs; transferring money into a prepaid balance is a funding event, not evidence that a particular property-management tenant consumed it. For a small SaaS issuing and revoking scoped keys per tenant, that distinction should determine the design of auto recharge, manual top-ups, and the per-day ceiling.

TL;DR: attribute usage to an immutable tenant and credential-version record before deciding how to replenish the shared balance. Set the auto-recharge trigger from expected consumption during replenishment delay, apply one daily funding allowance to both automatic and manual additions, and enforce tenant usage budgets separately. Keep reconciliation evidence longer than diagnostic request content. A funded account proves nothing about who should receive an invoice.

## Start with the chargeable quantity

Consider an explicitly hypothetical planning worksheet: 12 property-management tenants, each making 1,000 equally chargeable requests per day. That is 12,000 chargeable requests. Assume for this worksheet that every request costs one internal credit, with no other usage charges; these are arithmetic inputs, not a provider's billing rules or measured traffic. Under those assumptions, usage contributes 12,000 credits per day to the bill. Changing three funding transfers into one changes none of that consumption. Any transfer fees or different charging units would need their own verified terms before this worksheet could represent a real invoice.

Count usage first.

The change that moves the assumed dominant term is reducing chargeable work, such as removing 100 redundant requests per tenant per day where the application can preserve its required behavior. In this example, that removes 1,200 requests and leaves 10,800 credits of daily usage. A recharge threshold merely changes when those credits become available. Do not turn a funding setting into an efficiency claim.

There is another bill worth examining: the cost of preserving enough evidence to allocate that consumption correctly. A property manager disputing a charge needs a stable connection between a metered operation, the credential used, and the tenant responsible at that time. Keeping every prompt, response, and retry payload is a separate storage decision, with separate access and deletion consequences. Start with the allocation record; justify additional content retention independently.

## Who owns usage after a tenant key is revoked?

A scoped key should resolve to a tenant identifier, credential version, permitted operations, and lifecycle state. These are proposed application records, not assumptions about a commercial account API. The OWASP Secrets Management Cheat Sheet discusses secret lifecycle management, rotation, revocation, access controls, and auditing. Use those concerns as acceptance criteria for the credential layer, while defining the billing behavior explicitly in the application.

Revocation changes authorization going forward. Historical attribution must remain interpretable.

Suppose tenant `property_042` replaces credential version `key_v3` with `key_v4`. A usage receipt recorded against `key_v3` must continue to point to `property_042`, even after that credential stops authorizing new requests. Resolve ownership when admitting the request and preserve the versioned association with its eventual metering result. Looking up the current key owner during invoice generation creates an avoidable ambiguity if records have since been deleted or reassigned. Never reuse a credential identifier for a different tenant.

The awkward boundary is a request admitted before revocation but completed afterward. Define that policy before issuing keys: the proposed ledger charges its metered result to the tenant recorded at admission, while the authorization layer rejects subsequent admissions using the revoked credential. If cancellation is supported, record its outcome separately; a revocation event alone is insufficient evidence that already admitted work consumed nothing. This is a design contract to test, not a promise about external cancellation behavior.

Raw keys do not belong in usage receipts. Retain an opaque credential reference and protect the secret through its own lifecycle.

## How should auto recharge and manual top-ups share a prepaid API balance ceiling?

For the hypothetical account, suppose the planning burn rate is 30 credits per minute, the assumed replenishment delay is 4 minutes, and the chosen reserve is 80 credits. A starting threshold is therefore 200 credits: 30 multiplied by 4, plus 80. These inputs require validation against the actual workload and payment process. They provide no assurance against a burst larger than the reserve, a delayed balance observation, or a failed replenishment.

The threshold answers when to attempt funding. It doesn't answer how much a tenant may consume.

| Control | Proposed scope | What it constrains | Trade-off or failure mode |
| --- | --- | --- | --- |
| Auto-recharge threshold | Shared prepaid balance | Point at which replenishment is requested | Delayed balance observations can trigger overlapping attempts |
| Manual top-up | Operator funding action | Explicitly approved addition | Human response time can leave an otherwise healthy workload unfunded |
| Daily funding ceiling | Automatic and manual additions together | New funding reserved for one defined day | Existing balance remains available unless separately restricted |
| Tenant usage budget | Each tenant's admitted work | Reserved usage exposure | Conservative reservations may temporarily reject work that would have fit |
| Credential revocation | One credential version | Future admissions | Previously admitted work needs an explicit completion policy |

For this design, a manual top-up consumes the same daily funding allowance as an automatic recharge. Otherwise an operator path can defeat the very ceiling the automation observes. Define the day boundary, its time zone, and which pending amounts count before deploying either path. A daily recharge ceiling is not a daily usage ceiling: an opening balance can fund work without any new transfer that day.

Choose automatic funding when continuity requires replenishment without waiting for an operator and the shared allowance can be enforced. Choose manual funding when explicit review is worth the interruption risk. Neither choice repairs missing tenant attribution. For the property-management application, the deciding criterion is whether every admitted charge can still be assigned correctly after rotation, retries, and reconciliation.

## Reserve once, reconcile against durable receipts

The following Python example is deliberately limited to funding eligibility. It uses integer internal credits, includes pending reservations, and makes the arithmetic visible without implying that an in-memory function provides concurrency control or payment execution.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class FundingDay:
    settled: int
    pending: int
    ceiling: int


def eligible_top_up(
    balance: int, threshold: int, requested: int, day: FundingDay
) -> int:
    values = (balance, threshold, requested,
              day.settled, day.pending, day.ceiling)
    if any(value < 0 for value in values):
        raise ValueError("Credit amounts must be nonnegative")
    if balance >= threshold:
        return 0
    remaining = max(0, day.ceiling - day.settled - day.pending)
    return min(requested, remaining)


day = FundingDay(settled=600, pending=200, ceiling=1000)
assert eligible_top_up(150, 200, 300, day) == 200
assert eligible_top_up(200, 200, 300, day) == 0
```

Here, the requested 300 credits become an eligible 200 because the pending reservation already occupies part of the allowance. The surrounding service must read the allowance and persist that new reservation atomically; two workers evaluating the same snapshot can otherwise each approve the same remaining capacity. Manual funding should use that reservation mechanism too, with a different admission condition for operator approval. It should not call this threshold check and pretend a human request was automatic.

Persistence matters more than the function. Give each funding intent a stable identifier and record transitions from reservation to confirmed settlement or confirmed failure. A timeout leaves an uncertain outcome. Reconcile it before releasing capacity or issuing another transfer, using whatever lookup and duplicate-prevention semantics the actual payment interface documents. A local identifier alone cannot guarantee that an external transfer executes once.

Maintain a separate usage ledger with tenant ID, credential version, operation ID, metered quantity, attribution timestamp, and reconciliation state. A single operation may produce several transport attempts; the rule for billable work must follow the actual metering contract rather than an assumption that retries are free or always charged. Preserve external receipt references when available. Keep tenant accounting and funding accounting connected through reconciliation, without treating a top-up as a tenant usage event.

Before rollout, test concurrent recharge triggers, a manual request racing an automatic request, an uncertain transfer result, a day-boundary transition with funding still pending, and a key revoked while work is running. Each test should verify persisted ledger state after restart, not merely an HTTP response. A small team's operational interface should expose pending funding, unallocated usage, and reconciliation age so an operator can explain a balance discrepancy without reconstructing it from raw application logs. Those are proposed checks and metrics, not reported test results.

## References

The OWASP Secrets Management Cheat Sheet supports the credential-lifecycle and audit considerations in this note. The funding arithmetic, reservation policy, and retention choices are proposed application design, not requirements attributed to that source. Its scope does not establish a payment provider's recharge delay, ceiling semantics, billing units, or retention obligations; those must be verified for the deployment.

The resulting storage policy should retain compact usage receipts, credential-to-tenant history, funding transitions, and reconciliation outcomes for the period required to settle and dispute charges. Choose that period with the actual contracts and legal obligations in view. Protect those records and constrain access; deleting raw keys must not erase attribution.

Deliberately stop keeping full request and response bodies once a justified, bounded diagnostic window expires, and avoid recording secret values in diagnostic logs. The trade-off is reduced diagnostic detail in exchange for less retained content to protect. The cost appears during a later dispute: the ledger may establish which tenant incurred a charge while no longer containing enough content to replay the application decision. Accept that loss explicitly. If replay is required, budget for a separately governed evidence store instead of silently making permanent payload retention a condition of accurate billing.

## Further reading

- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
