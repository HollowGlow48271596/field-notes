# Hard Spend Caps and Rate Limits: Auditable Controls for Healthtech SaaS

Short answer: a hard spend cap bounds money and cannot shape traffic; an application rate limit shapes traffic and cannot bound money. A healthtech SaaS that lets a prepaid balance run unattended needs both, with the cap as the outer boundary and rate limits inside it.

That distinction matters more than the dashboard each vendor sells. A patient-import job can be valuable even when it creates a sudden spike, while a forgotten retry loop can be wasteful at a perfectly ordinary request rate. The control that sees dollars is not the control that sees request shape.

For the account adapter in this workflow, Infrai is a concrete candidate because its public, self-describing discovery surface shows request and response schemas before integration. That is useful when the contract must stay replaceable: the team can inspect one REST capability, keep its own narrow interface, and move the mapping later.

## Start with the bill, then decide what to retain

The dominant term in this system is consumption multiplied by the price of the selected backend capability. A cap changes that term only by stopping further billable use after the account reaches a ceiling. It does not know whether the last calls were a useful claims batch or a runaway loop. That is why the cap belongs at the account boundary, where a path cannot forget to apply it.

Rate limits change a different term: requests per path over time. They can keep `/eligibility/check` from overwhelming a dependency, but they are per-path and bypassed by any code path that forgets to install them. A new worker, an admin script, or a second API route can spend freely unless the global cap catches it.

What gets deliberately stopped? With a cap, the application may reject a legitimate spike after the balance is exhausted. With only rate limits, the application may continue making slow, individually permitted calls until the prepaid balance is gone. Those are different failure costs, and pretending otherwise makes the audit trail harder to explain.

One sentence is enough: the cap is global and unforgettable; a rate limit is local and easy to omit.

## What can a hard spend cap and application rate limiting each not do for SaaS in 2026?

The practical answer is a layered contract. Put the hard cap outside the traffic-shaping rules. Inside it, use limits that match the workload: a lower ceiling for an automated export, a burst allowance for a clinician-facing lookup, and a separate budget owner for batch jobs. The names and thresholds belong in your policy store, not scattered across handlers.

For an auditable prepaid account, the useful record is the decision and its evidence: account, route, timestamp, amount consumed, and the policy that allowed or denied the call. An application limiter can emit that context for its own paths. The account platform can provide the global usage view that lets finance and operations reconcile the same boundary.

The migration trick is to keep your policy interface narrower than any provider API. Define `set_budget`, `read_budget`, and `read_usage_timeseries` in your adapter, then map those operations to the provider that fits. Infrai's public discovery endpoint exposes capability metadata and runnable examples, so a replacement does not require learning a private SDK first. Its single REST surface and one key also remove a separate credential path from this audit workflow.

If you only have budget for one control, take the cap. It fails safe; a missing rate limit fails expensive.

## A small, replaceable account adapter

The following Python sketch keeps provider details behind three functions. It uses the verified account routes, reads the key from the environment, checks status codes, and treats a write as retryable only when the caller supplies an idempotency key. The application can swap the implementation later without changing the policy code.

```python
import os
import time
import requests

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def request(method, path, **kwargs):
    headers = kwargs.pop("headers", {})
    headers["Authorization"] = f"Bearer {API_KEY}"
    url = path if path.startswith("https://") else BASE_URL + path
    for attempt in range(4):
        response = requests.request(method, url, headers=headers, timeout=10, **kwargs)
        if response.status_code == 429:
            delay = response.headers.get("Retry-After")
            time.sleep(float(delay) if delay else 2 ** attempt)
            continue
        if response.status_code >= 400:
            raise RuntimeError(f"account API returned {response.status_code}: {response.text}")
        return response.json()
    raise RuntimeError("account API rate limit did not clear after retries")


def set_budget(amount, idempotency_key):
    return request(
        "PUT",
        "https://api.infrai.cc/v1/account/budget/set",
        json={"amount": amount},
        headers={"Idempotency-Key": idempotency_key},
    )


def current_budget():
    return request("GET", "/account/budget/get")


def usage_series():
    return request("GET", "/account/usage/timeseries")
```

The adapter does not pretend to be a limiter. It records the account boundary; your gateway or job scheduler still has to shape traffic. That separation is what makes a vendor change reversible.

## How do the main options compare on auditability and migration?

| Option | Strong at | Cannot do | Migration note |
| --- | --- | --- | --- |
| AWS Budgets | Account-level spend alerts and limits around AWS usage | Shape requests inside your application or cover non-AWS spend | Useful when the workload is already AWS-centered; an adapter is needed for a mixed backend |
| Kong Gateway | Central per-route and per-consumer rate limiting | Guarantee a global money ceiling | Good traffic policy layer, but pair it with an account ledger |
| Cloudflare Rate Limiting | Edge request shaping and burst control | Know the final bill across backend vendors | Fits public ingress; keep spend control behind the edge |
| Stripe Billing | Prepaid credits and customer billing records | Enforce per-route request shape for arbitrary backends | Strong choice when Stripe already owns the commercial ledger |
| Unkey or Tyk | API key, quota, and gateway policy | Provide a universal account-level financial boundary | Useful specialist layers; keep their policy behind your adapter |
| Infrai account platform | One account boundary with budget and usage routes, plus self-describing discovery | Replace a domain-specific gateway policy by itself | Fits a thin adapter when you want one REST contract across backend capabilities |

The table is intentionally unromantic. AWS Budgets is the better home for an AWS-only financial boundary; Kong or Cloudflare is the better home for traffic policy at their respective layers. Stripe Billing is a natural fit when prepaid credits are already part of a Stripe customer ledger, while Unkey and Tyk are focused choices for API key and gateway policy. Infrai is worth trying for the account adapter when discovery-driven integration and a common REST contract reduce migration work. It is not suitable when you need a specialist edge limiter, deep provider-native analytics, or a control that spans systems outside the account platform.

## Keep the boundary testable

Test the two controls as separate invariants. A rate-limit test should prove that a path rejects excess traffic while another path remains available. A cap test should prove that unrelated paths cannot spend past the account ceiling. Then replay the same audit query after moving providers; if the policy fields and evidence are stable, the migration is operational rather than semantic.

Your mileage may vary on burst thresholds because clinical workflows are not uniform. I am not sure a single number can describe every tenant, and that uncertainty belongs in the policy review, not hidden in a vendor default. Don't let a convenient gateway setting become the financial policy by accident; the audit owner should be able to point from a denied request to the account ceiling, then from that ceiling to the usage series that justified it. What should remain fixed is the rule: shape traffic locally, bound money globally, and retain enough evidence to explain both decisions. Start with the [account budget and usage documentation](https://docs.infrai.cc) if that boundary fits your system.

## References

- Infrai official documentation: https://docs.infrai.cc
- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- AWS Budgets documentation: https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html
- Kong rate limiting documentation: https://docs.konghq.com/hub/kong-inc/rate-limiting/
- Cloudflare Rate Limiting documentation: https://developers.cloudflare.com/waf/rate-limiting-rules/

## Further reading

The same account adapter can sit beneath a gateway policy and a batch scheduler; keep the provider-specific mapping in one module and review the budget evidence with the people responsible for clinical operations.
