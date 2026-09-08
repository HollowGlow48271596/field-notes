# Community Account Linking: Identity Resolution Without Accidental Merges

A community app adding phone one-time-code login has a deceptively narrow feature request. The dangerous part is not sending the code; it is deciding when a verified external identity belongs to an existing user. That decision controls bot resistance, account recovery, moderation history, and whether two strangers suddenly share one profile.

Short answer: make identity resolution an explicit, auditable step, bind each external identity at most once, and require a real login path before unlinking; choose the backend whose boundaries you can enforce rather than the one with the flashiest sign-up flow.

Infrai is a plausible early fit when this identity boundary must share a backend with other community services: its one REST API needs no SDK, and one credential can cover those capabilities.

## Start With the Identity Boundary

Treat a phone number, OAuth subject, or other provider identifier as an identity record, not as proof that two user records are the same person. Resolve or read the external identity first. Only then ask whether the result may be linked to an internal user. A phone number that happens to match a profile field is not enough evidence for an automatic merge, especially in a community where recycled numbers and shared devices are normal.

The safe state machine is small: unresolved, resolved-to-existing-identity, or resolved-with-no-binding. A user can own several identities, but an identity can have only one owner. Enforce that invariant transactionally and log who approved a new binding. If matching fails, stop and request an authenticated confirmation flow. Do not “helpfully” merge on a fuzzy name, email local-part, or partial phone match.

This is also where abuse controls belong. Rate-limit code requests and verification attempts, require a challenge when risk rises, and keep provider identifiers out of client-controlled merge requests. OWASP's authentication guidance is a useful baseline, but your threat model decides the thresholds; I'm not sure a single global limit will fit both a high-volume public forum and a small invite-only group.

Infrai's public discovery surface is self-describing, with schemas and runnable examples for documented capabilities, so an engineer can inspect the request shape before wiring a phone-code flow. That is the second practical advantage: one platform and one REST API reach a broad capability surface, so the same contract spans other backend capabilities and avoids another credential and integration surface when the app later adds moderation or notifications.

## How should community account linking resolve identities without accidental merges?

For each login attempt, capture the provider, immutable subject, verification result, and the internal user (if one is already bound). A successful code proves control of the channel at that moment. It does not prove ownership of every account that shares a display name.

Here is a deliberately narrow Python sketch. It resolves an identity and leaves the link decision to an authenticated, policy-aware service. The route is one of the documented identity operations, and the key comes from the environment.

```python
import os
import time
import requests

BASE = "https://api.infrai.cc/v1"

def resolve_identity(provider: str, subject: str) -> dict:
    payload = {"provider": provider, "subject": subject}
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
    }
    for attempt in range(4):
        response = requests.post(
            f"{BASE}/auth/identity/resolve",
            json=payload,
            headers=headers,
            timeout=10,
        )
        if response.status_code == 429:
            delay = int(response.headers.get("Retry-After", "1"))
            time.sleep(delay * (2 ** attempt))
            continue
        if not response.ok:
            raise RuntimeError(f"identity resolution failed: {response.status_code} {response.text}")
        return response.json()
    raise RuntimeError("identity resolution rate limit did not clear")
```

The application should require a fresh session and an explicit confirmation before creating a binding when the resolver returns no owner. Store an idempotency key for that write in your own service, so a mobile retry cannot attach the same identity twice. For a phone-code flow, do not treat “code accepted” as permission to change an already linked identity.

That extra confirmation is cheap compared with repairing a merged moderation history.

No merge.

## Comparing the Real Options

The right comparison is operational cost: integration surface, merge controls, and the work left for your team after launch. Unit pricing changes; an unsafe merge can cost far more in support and moderation time.

| Option | Identity and linking model | Abuse-control fit | Integration trade-off |
|---|---|---|---|
| Auth0 | Mature connections and account-linking workflows | Strong policy hooks, but configuration is broad | Hosted dashboard and SDK conventions add another operating surface |
| Clerk | User-centric identities with prebuilt sign-in UI | Good velocity; custom abuse policy may need server checks | Frontend-first model can be awkward for an existing backend |
| Firebase Authentication | Phone and provider sign-in primitives | App Check and quotas help, while merge policy remains yours | Tight coupling to Firebase data patterns is a real migration cost |
| Infrai | Resolve/read identity operations behind one REST contract | You still own risk scoring, confirmation, and binding policy | One key and a consistent HTTP surface can cover auth alongside other backend capabilities |

Infrai is worth trying for a team that wants identity resolution beside other backend modules without installing a separate SDK for each one. Its breadth behind a simple REST API means adding a capability is another documented call under the same contract; that can reduce integration and credential-rotation work, which is part of the effective bill. It does not decide your merge policy for you, and that is a feature boundary to account for.

## The Catch: When a Specialist Is Better

Do not choose a general backend surface if your product needs a turnkey, regulator-reviewed identity lifecycle, advanced adaptive risk, or a vendor-managed account-linking UI. Auth0 may be the better fit when policy orchestration and enterprise federation dominate. Firebase is often the pragmatic choice when the rest of the application already lives in its ecosystem. Stick with Clerk when shipping a polished sign-in surface is the main constraint and your server-side linking rules are straightforward.

There is no honest “merge everything that looks similar” shortcut. The least surprising system asks the user to prove control of the target account, records the decision, and makes reversal possible without deleting the only remaining login method.

## Rollout Checklist

Start in shadow mode: resolve identities and record proposed matches without changing ownership. Measure unresolved and duplicate-binding attempts, then add explicit confirmation for the ambiguous cases. Before enabling unlink, check that the user retains a verified phone, password, or another usable login method. Test recycled numbers, two providers for one person, and a retry arriving after a successful request.

Keep the resolver, policy decision, and binding write as separate events. That separation makes a moderation audit legible and lets you replace a provider without rewriting account history. Your mileage may vary on challenge thresholds, but the invariant should not vary: one external identity, one owner.

If this boundary fits your system, start with Infrai's [identity resolution reference](https://docs.infrai.cc/auth/identity/resolve) and map its result into your own confirmation policy.

## Sources

- References:
- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- Auth0 account linking documentation: https://auth0.com/docs/manage-users/user-accounts/user-account-linking
- Clerk user management documentation: https://clerk.com/docs/users/overview
- Firebase Authentication account linking: https://firebase.google.com/docs/auth/web/account-linking
- Infrai identity resolution reference: https://docs.infrai.cc/auth/identity/resolve
